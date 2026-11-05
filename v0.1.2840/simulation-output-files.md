---
title: "Simulation output files"
description: "Every simulation writes record files that you list and download with the results API. This page names the files the platform documents, says what writes..."
---

Every simulation writes record files that you list and download with the results API. This page names the files the platform documents, says what writes each one, and describes the columns of the files that several models share.

Files written by a single model, such as the Chakra workload or the Scala UET NIC, are described on that model's records page; the [file types overview](#file-types-overview) links to each one.

## How to access output files

After a simulation ends, the platform makes every file the simulator wrote available through three endpoints. In the examples, replace `{sim_id}` with the simulation's ID and `{file_name}` with a file name from the list.

### List the files

```bash
curl -X GET "https://api.scalacomputing.com/api/v1/simulations/{sim_id}/results?type=csv" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

```json
{
  "simulationId": "sim_3K9VG9B0TZrUrvmv9lfDopnDAMF",
  "duration": "00:12:41",
  "files": [
    {
      "type": "csv",
      "name": "sim_3K9VG9B0TZrUrvmv9lfDopnDAMF-nd-stats-0.csv",
      "size": "9510000",
      "downloadUrl": "https://..."
    }
  ],
  "pagination": {
    "count": 100,
    "hasMore": true,
    "nextCursor": "eyJrIjoi..."
  }
}
```

- Each entry in `files` carries a `downloadUrl` that is valid for 15 minutes. Download the file from it directly.
- `size` is the file's size in bytes, as a string. `duration` is the run's elapsed time.
- `type` comes from the file's extension: `csv`, `json`, `log`, `pcap`, `state`, or `unknown`. Use the `type` query parameter to list one type only.
- A page holds at most 100 files. When `pagination.hasMore` is `true`, pass `pagination.nextCursor` as the `next` query parameter to get the next page. A simulation run on several simulator processes writes one file per process for each record, so expect several pages.
- A file in a subdirectory is listed with its path, for example `subdir/file.csv`.

### Download one file

```bash
curl -X GET "https://api.scalacomputing.com/api/v1/simulations/{sim_id}/results/download-url?fileName={file_name}" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

This endpoint issues a new 15-minute `downloadUrl` for the file named by `fileName`, for example when the link from the list has expired. Its response also gives `contentType`, `sizeBytes`, `expiresAt`, and, when one is available, a `checksum`.

### Get computed results

`GET /api/v1/simulations/{sim_id}/results/data` computes results on the platform from the record files, so you do not have to download and combine every shard yourself. The response does not hold the results: its `file.downloadUrl` links to a zstd-compressed JSON document that does.

```bash
URL=$(curl -s "https://api.scalacomputing.com/api/v1/simulations/{sim_id}/results/data?metric=nd-stats&tier=all" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" | jq -r '.file.downloadUrl')
curl -s "$URL" | zstd -d > nd-stats-summary.json
```

`metric` names the record to compute from, and `mode` the kind of result. `mode` defaults to `summary`.

| `metric` | Computed from | `mode` values |
|----------|---------------|---------------|
| `nd-stats` | [`nd-stats`](#network-device-statistics-nd-stats) | `summary`, `timeSeries`, `utilizationBins`, `uplinkFairness` |
| `pfc` | [`ecn-pfc-stats`](#ecn-and-pfc-statistics-ecn-pfc-stats) | `summary`, `timeSeries` |
| `perf` | [`perf`](#performance-metrics-perf) | `summary`, `completionTimeBins` |
| `roce-transport-stats` | [`roce-transport-stats`](#roce-transport-statistics-roce-transport-stats) | `summary`, `timeSeries` |
| `uet-transport-stats` | [`uet-transport-stats`](./models/scala-uet-nic/records.md#uet-transport-stats) | `summary`, `timeSeries` |
| `ranks` | [`chakra-completion-stats`](./models/chakra-workload/records.md#rank-completion) | `summary` |
| `chakra-perf` | [`chakra-perf`](./models/chakra-workload/records.md#transfers) | `summary` |
| `collectives` | [`chakra-comm-collective-percentiles-stats`](./models/chakra-workload/records.md#collectives) | `summary` |
| `cdf` | [`chakra-cdf`](./models/chakra-workload/records.md#rank-completion) | `summary` |

`tier` limits the result to one network tier: `ssw`, `fsw`, `rsw`, or `nic` (see [Network tiers](#network-tiers)). The default, `all`, covers every tier. The [Simulations API reference](./api-reference/simulations.md) lists every parameter of the endpoint, including its filters.

## File naming convention

Output files follow the pattern:

```
{simulation}-{record}-{shard-index}.csv
```

For example, a simulation with the ID `sim_3K9VG9B0TZrUrvmv9lfDopnDAMF` produces files like:
- `sim_3K9VG9B0TZrUrvmv9lfDopnDAMF-ecn-pfc-stats-0.csv`
- `sim_3K9VG9B0TZrUrvmv9lfDopnDAMF-nd-stats-0.csv`
- `sim_3K9VG9B0TZrUrvmv9lfDopnDAMF-nd-stats-1.csv`

`{simulation}` is the same in every file of one simulation. It is usually the simulation's ID, as in the example above, but some simulations use the name you gave the simulation instead. To find a file, match the record part of its name rather than the prefix.

`{shard-index}` numbers the simulator processes. A simulation runs as one or more simulator processes, and each process writes its own file for each record, covering the nodes that process ran. A simulation that runs as one process writes only shard `0`. To cover the whole simulation, combine the files of every shard.

A few files differ from the pattern:
- `global-sim-report` is a JSON file: `{simulation}-global-sim-report-{shard-index}.json`.
- `chakra-mapping` is written in shard `0` only, and `chakra-comm-collective-percentiles-stats` is one file for the whole run ([Chakra workload records](./models/chakra-workload/records.md#records-a-simulation-writes)).

## File types overview

### Files described on this page

| File type | Name pattern | Written | Description |
|-----------|-------------|---------|-------------|
| [Network device statistics](#network-device-statistics-nd-stats) | `*-nd-stats-*.csv` | By default | Per-interface packet/byte counters, throughput, utilization |
| [Performance metrics](#performance-metrics-perf) | `*-perf-*.csv` | By default | Application-level latency and transfer measurements |
| [ECN and PFC statistics](#ecn-and-pfc-statistics-ecn-pfc-stats) | `*-ecn-pfc-stats-*.csv` | By default | Priority Flow Control and ECN marking counters |
| [Application report](#application-report-application-report) | `*-application-report-*.csv` | When turned on | Socket and application binding information |
| [Beacon](#beacon-beacon) | `*-beacon-*.csv` | By default | Periodic heartbeat counters for client/server traffic |
| [RoCE transport statistics](#roce-transport-statistics-roce-transport-stats) | `*-roce-transport-stats-*.csv` | By default, for each Scala RoCE NIC | RDMA over Converged Ethernet transport-layer counters |
| [Tier hop](#tier-hop-tier-hop) | `*-tier-hop-*.csv` | By default | Packet counts received in each switch tier |
| [Topology report](#topology-report-topology-report) | `*-topology-report-*.csv` | By default | Static topology describing device interconnections |
| [Switch buffer statistics](#switch-buffer-statistics-scala-switch-agg-buff-stats) | `*-scala-switch-agg-buff-stats-*.csv` | By default, for each Scala Switch | Aggregated switch buffer occupancy and utilization |
| [Packet trimming statistics](#packet-trimming-statistics-packet-trimming-stats) | `*-packet-trimming-stats-*.csv` | When turned on | Packets trimmed, and trimmable and trimmed packets handled, by each switch |
| [Global simulation report](#global-simulation-report-global-sim-report) | `*-global-sim-report-*.json` | By default | Run totals of packets, bytes, drops, and PFC frames, per tier |

### Files described on a model page

| File type | Written by | Written | Described in |
|-----------|-----------|---------|--------------|
| `chakra-capable-hosts`, `chakra-mapping`, `chakra-apps` | Chakra workload | By default | [Placement records](./models/chakra-workload/records.md#placement-records) |
| `chakra-perf` | Chakra workload | By default | [Transfers](./models/chakra-workload/records.md#transfers) |
| `chakra-completion-stats`, `chakra-cdf` | Chakra workload | By default | [Rank completion](./models/chakra-workload/records.md#rank-completion) |
| `chakra-comm-collective-stats`, `chakra-comm-collective-percentiles-stats` | Chakra workload | By default | [Collectives](./models/chakra-workload/records.md#collectives) |
| `chakra-profiling` (text) | Chakra workload | When turned on | [Profiling](./models/chakra-workload/records.md#profiling) |
| `uet-transport-stats` | Scala UET NIC | By default | [uet-transport-stats](./models/scala-uet-nic/records.md#uet-transport-stats) |
| `uet-events`, `uet-monitor`, `net-buff-stats`, `pcie-stats` | Scala UET NIC | When turned on | [Records turned on by attributes](./models/scala-uet-nic/records.md#records-turned-on-by-attributes) |
| `mrc-transport-stats` | Scala MRC NIC | By default | [mrc-transport-stats](./models/scala-mrc-nic/records.md#mrc-transport-stats) |
| `mrc-events`, `mrc-monitor` | Scala MRC NIC | When turned on | [Records turned on by attributes](./models/scala-mrc-nic/records.md#records-turned-on-by-attributes) |
| `net-buff-stats` | Scala RoCE NIC | When turned on | [Records the buffers and PFC write](./models/scala-roce-nic/buffers-and-pfc.md#records-the-buffers-and-pfc-write) |
| `scala-switch-load-balancing-stats` | Scala Switch | By default | [What is recorded](./models/scala-switch/packet-handling.md#what-is-recorded) |
| `rdma-perf` | Scala RDMA application | By default | [rdma-perf](./models/scala-rdma-application/traffic-generation.md#rdma-perf) |
| `tcp-stats` | Scala Traffic Generator | When turned on | [tcp-stats](./models/scala-traffic-generator/traffic-generation.md#tcp-stats) |

A simulation also writes [diagnostic files](#diagnostic-files) for the simulator's own use.

## Detailed file schemas

### Network device statistics (nd-stats)

Network Device Statistics files contain per-interface, per-time-step counters for every network device in the simulated topology. Each row captures packet counts, byte volumes, throughput rates, link utilization percentages, and drop counters. The global `EnableNetDeviceStatLogging` (default `true`) turns the file on, and `metric=nd-stats` computes results from it ([Get computed results](#get-computed-results)).

**Filename pattern:** `{simulation}-nd-stats-{shard-index}.csv`

#### Columns

| Column | Type | Description |
|--------|------|-------------|
| `time_utc` | String | Wall-clock timestamp (ISO 8601) |
| `sim_time_sec` | Float64 | Simulated time in seconds |
| `component_name` | String | Name of the network component (e.g., `ScalaFabricSwitch-0-0-0-0-1-0`) |
| `nd_name` | String | Network device class name |
| `component_type` | String | Type of component (e.g., `Switch`) |
| `ifid` | Int64 | Interface ID on the device |
| `uldl` | String | Direction: `ul` (uplink) or `dl` (downlink) |
| `node_id` | Int64 | Unique node identifier |
| `tier` | Integer | Network tier of the device (see [Network tiers](#network-tiers)) |
| `subtier` | Int64 | Sub-tier identifier; `4294967295` when the device has none |
| `total_packets_tx` | Int64 | Cumulative packets transmitted |
| `total_packets_rx` | Int64 | Cumulative packets received |
| `total_bytes_tx_GB` | Float64 | Cumulative bytes transmitted (GB) |
| `total_bytes_rx_GB` | Float64 | Cumulative bytes received (GB) |
| `interval_bytes_tx_MB` | Float64 | Bytes transmitted in the current interval (MB) |
| `interval_bytes_rx_MB` | Float64 | Bytes received in the current interval (MB) |
| `tx_rate_Gbps` | Float64 | Transmit rate (Gbps) |
| `rx_rate_Gbps` | Float64 | Receive rate (Gbps) |
| `tx_link_percent_util` | Float64 | Transmit link utilization (percent) |
| `rx_link_percent_util` | Float64 | Receive link utilization (percent) |
| `tx_drops` | Int64 | Cumulative transmit-side drops |
| `rx_drops` | Int64 | Cumulative receive-side drops |

#### Network tiers

| Tier value | Abbreviation | `tier` filter | Description |
|------------|-------------|---------------|-------------|
| 0 | SSW | `ssw` | Spine Switch |
| 1 | FSW | `fsw` | Fabric Switch |
| 2 | RSW | `rsw` | Rack Switch |
| 3 | NIC | `nic` | Network Interface Card |
| 4 | Host | None | Host |
| 5 | None | None | In `nd-stats`, a device with no tier. In `topology-report`, a scale-up switch ([Scale-up groups](#scale-up-groups)) |

#### Example row

```csv
time_utc,sim_time_sec,component_name,nd_name,component_type,ifid,uldl,node_id,tier,subtier,total_packets_tx,total_packets_rx,total_bytes_tx_GB,total_bytes_rx_GB,interval_bytes_tx_MB,interval_bytes_rx_MB,tx_rate_Gbps,rx_rate_Gbps,tx_link_percent_util,rx_link_percent_util,tx_drops,rx_drops
2026-01-13T19:11:38Z,0.003,ScalaFabricSwitch-0-0-0-0-1-0,scala_switch::ScalaSwitchEthNetDevice,Switch,0,dl,0,1,4294967295,115,112,7.3062e-05,7.4146e-05,0.073062,0.074146,0.584496,0.593168,0.073062,0.074146,0,0
```

---

### Performance metrics (perf)

Performance Metrics files capture application-level latency measurements. Two applications write rows:

- The Scala Traffic Generator writes a row for each transaction whose reply completes in the recording window ([perf](./models/scala-traffic-generator/traffic-generation.md#perf)).
- The Scala UDP Echo application's client writes a row for each echoed packet.

In other simulations the file holds only its header. `metric=perf` computes results from it ([Get computed results](#get-computed-results)).

**Filename pattern:** `{simulation}-perf-{shard-index}.csv`

The name pattern `*-perf-*.csv` also matches the `chakra-perf` and `rdma-perf` files, which are other records.

#### Columns

| Column | Type | Description |
|--------|------|-------------|
| `job_id` | String | Identifier for the simulation job |
| `time_utc` | String | Wall-clock timestamp (ISO 8601) |
| `sim_time_sec` | Float64 | Simulated time in seconds |
| `metric_name` | String | Name of the performance metric (e.g., `ScalaApp`) |
| `perf_metric_usec` | Float64 | Measured latency in microseconds |
| `metric_path` | String | `NAN` from both writers |
| `metric_aggregation_id` | String | `NAN` from both writers |
| `node_id` | Int64 | Node that recorded the metric |
| `transaction_number` | Int64 | Sequential transaction number. `NAN` from the UDP Echo client |
| `start_time_sec` | Float64 | Simulation time when the transaction started (seconds) |
| `transfer_size_b` | Float64 | Size of the echoed packet in bytes. `NAN` from the Scala Traffic Generator |
| `application_name` | String | Name of the application that generated the transaction |

#### Example row

```csv
job_id,time_utc,sim_time_sec,metric_name,perf_metric_usec,metric_path,metric_aggregation_id,node_id,transaction_number,start_time_sec,transfer_size_b,application_name
example,2026-01-13T19:11:40Z,0.00500019,ScalaApp,2.83117,NAN,NAN,478,4,0.00499736,NAN,TrafficGeneratorClient-P:0-R:3-S:24-R:0-C:0-1554
```

---

### ECN and PFC statistics (ecn-pfc-stats)

ECN and PFC statistics files record Priority Flow Control (PFC) frame counters and Explicit Congestion Notification (ECN) marking counts per interface, per priority level. Use this data to analyze congestion events and flow control behavior. The global `EnablePFCECNStatsLogging` (default `true`) turns the file on, and `metric=pfc` computes results from it ([Get computed results](#get-computed-results)).

An interface and priority get rows only once one of their counters is non-zero. From then on they get a row every interval.

**Filename pattern:** `{simulation}-ecn-pfc-stats-{shard-index}.csv`

#### Columns

| Column | Type | Description |
|--------|------|-------------|
| `job_id` | String | Identifier for the simulation job |
| `time_utc` | String | Wall-clock timestamp (ISO 8601) |
| `sim_time_sec` | Float64 | Simulated time in seconds |
| `node_id` | Int64 | Unique node identifier |
| `component_name` | String | Name of the network component |
| `component_type` | String | Type of component (e.g., `Switch`) |
| `ifid` | Int32 | Interface ID |
| `uldl` | String | Direction: `ul` (uplink) or `dl` (downlink) |
| `tier` | Int32 | Network tier (see [Network tiers](#network-tiers)) |
| `subtier` | Int64 | Sub-tier identifier |
| `priority` | Int64 | Traffic priority class |
| `pfcxoff_tx` | Int64 | PFC XOFF frames transmitted |
| `pfcxoff_rx` | Int64 | PFC XOFF frames received |
| `pfcxon_tx` | Int64 | PFC XON frames transmitted |
| `pfcxon_rx` | Int64 | PFC XON frames received |
| `pfc_time_paused_usec` | Float64 | Cumulative time paused due to PFC (microseconds) |
| `interval_total_time_xoff_usec` | Float64 | Total XOFF time in the current interval (microseconds) |
| `interval_max_time_xoff_usec` | Float64 | Maximum single XOFF duration in the interval (microseconds) |
| `interval_total_time_xoff_egr_occ_usec` | Float64 | Total XOFF time due to egress occupancy in the interval (microseconds) |
| `interval_max_time_xoff_egr_occ_usec` | Float64 | Maximum single XOFF duration due to egress occupancy (microseconds) |
| `total_xoff_time_egr_occ_usec` | Float64 | Cumulative XOFF time due to egress occupancy (microseconds) |
| `percent_xon` | Float64 | Fraction of the time since data traffic started that the priority was not paused, from 0 to 1 |
| `xoff_egr_occ_ratio` | Float64 | Time paused due to egress occupancy, as a fraction of the time since data traffic started |
| `pkts_ecn_marked` | Int64 | Number of packets marked with ECN |

> **Note:** The microsecond-duration columns are fractional (`Float64`); the simulator derives them from nanosecond timers divided by 1000. Server-side PFC summary values (e.g. `totalTimePausedUsec`) may therefore be non-integer.

#### Example header

```csv
job_id,time_utc,sim_time_sec,node_id,component_name,component_type,ifid,uldl,tier,subtier,priority,pfcxoff_tx,pfcxoff_rx,pfcxon_tx,pfcxon_rx,pfc_time_paused_usec,interval_total_time_xoff_usec,interval_max_time_xoff_usec,interval_total_time_xoff_egr_occ_usec,interval_max_time_xoff_egr_occ_usec,total_xoff_time_egr_occ_usec,percent_xon,xoff_egr_occ_ratio,pkts_ecn_marked
```

---

### Application report (application-report)

Application Report files describe the socket bindings established at simulation startup. Each row maps a host component and node to its application type, socket role, DSCP/ECT settings, and peer node. The file is written only when the global `EnableApplicationReport` is `true` (default `false`).

**Filename pattern:** `{simulation}-application-report-{shard-index}.csv`

#### Columns

| Column | Type | Description |
|--------|------|-------------|
| `job_id` | String | Identifier for the simulation job |
| `time_utc` | String (ISO 8601) | Wall-clock timestamp |
| `component_name` | String | Host component name (e.g., `ScalaHost-0-0-0-0`) |
| `node_id` | Integer | Unique node identifier |
| `socket_type` | String | Socket role: `CLIENT` or `SERVER` |
| `app_typeid` | String | NS3 application type (e.g., `ns3::ScalaTrafficGeneratorApplication`) |
| `app_name` | String | Application instance name |
| `dscp` | Integer | Differentiated Services Code Point value |
| `ect` | Integer | ECN-Capable Transport flag |
| `peer_node_id` | Integer | Node ID of the communication peer |

#### Example row

```csv
job_id,time_utc,component_name,node_id,socket_type,app_typeid,app_name,dscp,ect,peer_node_id
example,2026-01-13T19:11:37Z,ScalaHost-0-0-0-0,262,CLIENT,ns3::ScalaTrafficGeneratorApplication,TrafficGeneratorClient-P:0-R:0-S:0-R:0-C:0-1,0,0,386
```

---

### Beacon (beacon)

Beacon files record periodic heartbeat counters tracking client and server activity and data goodput. What the client and server columns count depends on the application: requests and replies for the Scala Traffic Generator ([Beacon](./models/scala-traffic-generator/traffic-generation.md#beacon)), work requests for the Scala RDMA application ([Beacon](./models/scala-rdma-application/traffic-generation.md#beacon)).

**Filename pattern:** `{simulation}-beacon-{shard-index}.csv`

#### Columns

| Column | Type | Description |
|--------|------|-------------|
| `job_id` | String | Identifier for the simulation job |
| `time_utc` | String (ISO 8601) | Wall-clock timestamp |
| `sim_time_sec` | Float | Simulated time in seconds |
| `interval_duration_sec` | Integer | Wall-clock seconds since the previous row |
| `tx_client` | Integer | Cumulative units sent by clients |
| `rx_client` | Integer | Cumulative units received by clients |
| `tx_server` | Integer | Cumulative units sent by servers |
| `rx_server` | Integer | Cumulative units received by servers |
| `rx_bytes_reader` | Integer | Cumulative bytes received by reader applications |
| `rx_bytes_writer` | Integer | Bytes of the requests the servers received in full. The Scala Traffic Generator counts only those received in the recording window ([Beacon](./models/scala-traffic-generator/traffic-generation.md#beacon)) |
| `goodput_reader_MB_per_sec` | Float | Reader goodput per server, in MB/s (1 MB = 10^6 bytes), averaged over the intervals recorded so far |
| `goodput_writer_MB_per_sec` | Float | Writer goodput per server, in MB/s (1 MB = 10^6 bytes), averaged over the intervals recorded so far |

#### Example row

```csv
job_id,time_utc,sim_time_sec,interval_duration_sec,tx_client,rx_client,tx_server,rx_server,rx_bytes_reader,rx_bytes_writer,goodput_reader_MB_per_sec,goodput_writer_MB_per_sec
example,2026-01-13T19:11:42Z,0.006,1,5027,5027,5027,5027,0,1314708,0,52.5883
```

---

### RoCE transport statistics (roce-transport-stats)

RoCE Transport Statistics files record detailed RDMA over Converged Ethernet (RoCEv2) transport-layer counters per network device. This includes packet counts by verb type, acknowledgment/NACK tracking, reordering buffer statistics, work queue entries, retransmissions, and round-trip time measurements.

The file has a row for each Scala RoCE NIC every `NDStatsReportInterval`. The global `EnableRoceTransportStats` (default `true`) turns it on, and `metric=roce-transport-stats` computes results from it ([Get computed results](#get-computed-results)). [Records the transport writes](./models/scala-roce-nic/transport.md#records-the-transport-writes) explains what the transport counts.

**Filename pattern:** `{simulation}-roce-transport-stats-{shard-index}.csv`

#### Columns

| Column | Type | Description |
|--------|------|-------------|
| `time_utc` | String (ISO 8601) | Wall-clock timestamp |
| `sim_time_sec` | Float | Simulated time in seconds |
| `component_name` | String | Host component name |
| `node_id` | Integer | Unique node identifier |
| `num_pkts_sent` | Integer | Cumulative packets sent |
| `ivl_num_pkts_sent` | Integer | Packets sent in the current interval |
| `num_verb_pkts_sent` | Integer | Cumulative verb (data) packets sent |
| `ivl_num_verb_pkts_sent` | Integer | Verb packets sent in the interval |
| `num_non_verb_pkts_sent` | Integer | Cumulative non-verb (control) packets sent |
| `ivl_num_non_verb_pkts_sent` | Integer | Non-verb packets sent in the interval |
| `num_conn_pkts_sent` | Integer | Cumulative connection packets sent |
| `ivl_num_conn_pkts_sent` | Integer | Connection packets sent in the interval |
| `num_conn_pkts_recvd` | Integer | Cumulative connection packets received |
| `ivl_num_conn_pkts_recvd` | Integer | Connection packets received in the interval |
| `rdma_write_cnt` | Integer | Cumulative RDMA WRITE operations |
| `ivl_rdma_write_cnt` | Integer | RDMA WRITE operations in the interval |
| `rdma_send_cnt` | Integer | Cumulative RDMA SEND operations |
| `ivl_rdma_send_cnt` | Integer | RDMA SEND operations in the interval |
| `rdma_atomic_cnt` | Integer | Cumulative RDMA ATOMIC operations |
| `ivl_rdma_atomic_cnt` | Integer | RDMA ATOMIC operations in the interval |
| `num_acks_sent` | Integer | Cumulative ACKs sent |
| `ivl_num_acks_sent` | Integer | ACKs sent in the interval |
| `num_acks_recvd` | Integer | Cumulative ACKs received |
| `ivl_num_acks_recvd` | Integer | ACKs received in the interval |
| `num_sacks_sent` | Integer | Cumulative selective ACKs sent |
| `ivl_num_sacks_sent` | Integer | Selective ACKs sent in the interval |
| `num_sacks_recvd` | Integer | Cumulative selective ACKs received |
| `ivl_num_sacks_recvd` | Integer | Selective ACKs received in the interval |
| `num_nacks_sent` | Integer | Cumulative NACKs sent |
| `ivl_num_nacks_sent` | Integer | NACKs sent in the interval |
| `num_nacks_recvd` | Integer | Cumulative NACKs received |
| `ivl_num_nacks_recvd` | Integer | NACKs received in the interval |
| `num_writes_recvd` | Integer | Cumulative RDMA WRITE operations received |
| `ivl_num_writes_recvd` | Integer | RDMA WRITE operations received in the interval |
| `num_sends_recvd` | Integer | Cumulative RDMA SEND operations received |
| `ivl_num_sends_recvd` | Integer | RDMA SEND operations received in the interval |
| `num_cnp_sent` | Integer | Cumulative Congestion Notification Packets sent |
| `ivl_num_cnp_sent` | Integer | CNPs sent in the interval |
| `num_cnp_recvd` | Integer | Cumulative CNPs received |
| `ivl_num_cnp_recvd` | Integer | CNPs received in the interval |
| `num_seq_out_of_order` | Integer | Cumulative out-of-order sequence events |
| `ivl_num_seq_out_of_order` | Integer | Out-of-order events in the interval |
| `num_duplicate` | Integer | Cumulative duplicate packets received |
| `ivl_num_duplicate` | Integer | Duplicate packets received in the interval |
| `ivl_min_size_reorder_buf_psn` | Integer | Minimum reorder buffer size in the interval (PSN entries) |
| `max_size_reorder_buf_psn` | Integer | Maximum reorder buffer size observed (PSN entries) |
| `ivl_max_size_reorder_buf_psn` | Integer | Maximum reorder buffer size in the interval |
| `avg_size_reorder_buf_psn` | Float | Average reorder buffer size (PSN entries) |
| `ivl_avg_size_reorder_buf_psn` | Float | Average reorder buffer size in the interval |
| `num_wqe_created` | Integer | Cumulative Work Queue Entries created |
| `ivl_num_wqe_created` | Integer | WQEs created in the interval |
| `num_wqe_completed` | Integer | Cumulative WQEs completed |
| `ivl_num_wqe_completed` | Integer | WQEs completed in the interval |
| `num_rtx` | Integer | Cumulative retransmissions |
| `ivl_num_rtx` | Integer | Retransmissions in the interval |
| `num_rto` | Integer | Cumulative retransmission timeouts |
| `ivl_num_rto` | Integer | RTOs in the interval |
| `ivl_min_calc_rtt_usec` | Float | Minimum calculated RTT in the interval (microseconds) |
| `max_calc_rtt_usec` | Float | Maximum calculated RTT observed (microseconds) |
| `ivl_max_calc_rtt_usec` | Float | Maximum calculated RTT in the interval (microseconds) |
| `avg_calc_rtt_usec` | Float | Average calculated RTT (microseconds) |
| `ivl_avg_calc_rtt_usec` | Float | Average calculated RTT in the interval (microseconds) |
| `num_rx_drops` | Integer | Cumulative transport receive drops, summed over the four receive reasons below. Excludes NIC ingress-admission drops and transmit payload discards. None of the four reasons is counted in `nd-stats` `rx_drops`: on `ScalaRoceNic`, `nd-stats` `rx_drops` is the NIC ingress-admission drops alone |
| `ivl_num_rx_drops` | Integer | Receive-path discards in the interval |
| `num_rx_drops_no_qp_data` | Integer | Data packets discarded because no QP context resolved for the flow id, typically a late retransmit after its QP was torn down. Excluded from `nd-stats` `rx_drops` |
| `ivl_num_rx_drops_no_qp_data` | Integer | No-QP data discards in the interval |
| `num_rx_drops_no_qp_ctrl` | Integer | Control packets (ACK/SACK/NAK/CNP) discarded because no QP context resolved. Excluded from `nd-stats` `rx_drops` |
| `ivl_num_rx_drops_no_qp_ctrl` | Integer | No-QP control discards in the interval |
| `num_rx_drops_after_nak` | Integer | Packets discarded because the QP has an unfilled hole: the NAK-triggering packet and everything behind it, until the hole fills. Excluded from `nd-stats` `rx_drops` |
| `ivl_num_rx_drops_after_nak` | Integer | After-NAK discards in the interval |
| `num_rx_drops_ghost_ack` | Integer | Cumulative ACK or NAK packets discarded because their PSN exceeds the highest PSN transmitted by the QP. Excluded from `nd-stats` `rx_drops` |
| `ivl_num_rx_drops_ghost_ack` | Integer | Ghost ACK or NAK discards in the interval. Excluded from `nd-stats` `rx_drops` |
| `num_tx_payload_discards` | Integer | Cumulative fetched payload segments discarded before PSN assignment because their generation or QP instance is stale. Excluded from `num_rx_drops` and `nd-stats` `rx_drops` |
| `ivl_num_tx_payload_discards` | Integer | Transmit payload discards in the interval. Excluded from `num_rx_drops` and `nd-stats` `rx_drops` |

#### Example header

```csv
time_utc,sim_time_sec,component_name,node_id,num_pkts_sent,ivl_num_pkts_sent,num_verb_pkts_sent,ivl_num_verb_pkts_sent,num_non_verb_pkts_sent,ivl_num_non_verb_pkts_sent,num_conn_pkts_sent,ivl_num_conn_pkts_sent,num_conn_pkts_recvd,ivl_num_conn_pkts_recvd,rdma_write_cnt,ivl_rdma_write_cnt,rdma_send_cnt,ivl_rdma_send_cnt,rdma_atomic_cnt,ivl_rdma_atomic_cnt,num_acks_sent,ivl_num_acks_sent,num_acks_recvd,ivl_num_acks_recvd,num_sacks_sent,ivl_num_sacks_sent,num_sacks_recvd,ivl_num_sacks_recvd,num_nacks_sent,ivl_num_nacks_sent,num_nacks_recvd,ivl_num_nacks_recvd,num_writes_recvd,ivl_num_writes_recvd,num_sends_recvd,ivl_num_sends_recvd,num_cnp_sent,ivl_num_cnp_sent,num_cnp_recvd,ivl_num_cnp_recvd,num_seq_out_of_order,ivl_num_seq_out_of_order,num_duplicate,ivl_num_duplicate,ivl_min_size_reorder_buf_psn,max_size_reorder_buf_psn,ivl_max_size_reorder_buf_psn,avg_size_reorder_buf_psn,ivl_avg_size_reorder_buf_psn,num_wqe_created,ivl_num_wqe_created,num_wqe_completed,ivl_num_wqe_completed,num_rtx,ivl_num_rtx,num_rto,ivl_num_rto,ivl_min_calc_rtt_usec,max_calc_rtt_usec,ivl_max_calc_rtt_usec,avg_calc_rtt_usec,ivl_avg_calc_rtt_usec,num_rx_drops,ivl_num_rx_drops,num_rx_drops_no_qp_data,ivl_num_rx_drops_no_qp_data,num_rx_drops_no_qp_ctrl,ivl_num_rx_drops_no_qp_ctrl,num_rx_drops_after_nak,ivl_num_rx_drops_after_nak,num_rx_drops_ghost_ack,ivl_num_rx_drops_ghost_ack,num_tx_payload_discards,ivl_num_tx_payload_discards
```

---

### Tier hop (tier-hop)

Tier Hop files record the cumulative and interval packet counts received on the switch ports of each tier (spine, fabric, rack). Use this to understand traffic distribution across the network hierarchy. The file is written with `nd-stats`, and each shard counts the switches it runs.

**Filename pattern:** `{simulation}-tier-hop-{shard-index}.csv`

#### Columns

| Column | Type | Description |
|--------|------|-------------|
| `job_id` | String | Identifier for the simulation job |
| `time_utc` | String (ISO 8601) | Wall-clock timestamp |
| `sim_time_sec` | Float | Simulated time in seconds |
| `spine` | Integer | Cumulative packets received on spine switch ports |
| `fabric` | Integer | Cumulative packets received on fabric switch ports |
| `rack` | Integer | Cumulative packets received on rack switch ports |
| `spineInt` | Integer | Packets received on spine switch ports in the current interval |
| `fabricInt` | Integer | Packets received on fabric switch ports in the current interval |
| `rackInt` | Integer | Packets received on rack switch ports in the current interval |

#### Example row

```csv
job_id,time_utc,sim_time_sec,spine,fabric,rack,spineInt,fabricInt,rackInt
example,2026-01-13T19:11:39Z,0.003,0,18137,42672,0,18137,42672
```

---

### Topology report (topology-report)

Topology Report files describe the static physical topology of the simulated network. Each row represents one link between two devices, including tier placement, device type, link speed, and propagation delay. The file is written once, when the topology is built; each shard writes the links of the nodes it runs. The global `EnableTopologyReport` (default `true`) turns it on.

**Filename pattern:** `{simulation}-topology-report-{shard-index}.csv`

#### Columns

| Column | Type | Description |
|--------|------|-------------|
| `component_name` | String | Name of the source component |
| `component_type` | String | Type of source component (e.g., `Switch`) |
| `tier` | Integer | Network tier of the source component |
| `subtier` | Integer | Sub-tier of the source component |
| `node_id` | Integer | Node ID of the source component |
| `net_device` | String | Network device class on the source side |
| `dev_id` | Integer | Device/interface ID on the source side |
| `peer_component_name` | String | Name of the peer component |
| `peer_component_type` | String | Type of peer component |
| `peer_tier` | Integer | Network tier of the peer component |
| `peer_subtier` | Integer | Sub-tier of the peer component |
| `peer_node_id` | Integer | Node ID of the peer component |
| `peer_net_device` | String | Network device class on the peer side |
| `peer_dev_id` | Integer | Device/interface ID on the peer side |
| `linkspeed_Gbps` | Float | Link speed in Gbps |
| `prop_delay_ns` | Float | Propagation delay in nanoseconds |
| `pod` | Integer or empty | The POD that contains the source component, counted from 0 |
| `rack` | Integer or empty | The rack that contains the source component, counted from 0 across the whole topology |
| `pod_config_name` | String or empty | The name of the configuration of the POD that contains the source component. The name is written as given and is not quoted, so a comma or a line break in it breaks the row's columns |
| `peer_pod` | Integer or empty | The POD that contains the peer component |
| `peer_rack` | Integer or empty | The rack that contains the peer component |
| `peer_pod_config_name` | String or empty | The name of the configuration of the POD that contains the peer component, written as `pod_config_name` is |
| `plane` | Integer or empty | The plane of a link between a spine switch and a fabric switch, or of a scale-up link |
| `scaleup_group` | Integer or empty | The scale-up group of a scale-up link |

An empty field means the component or link has no such value. It is never written as 0:

- Rack switches, NICs and servers have a POD and a rack. Fabric switches have a POD and no rack. Spine switches and scale-up switches have neither.
- The FAP and FE nodes of a chassis switch have the POD and rack of their chassis.
- A link between a spine switch and a fabric switch has the plane of the spine. Spine switch `s` of a data center is in plane `s / N`, rounded down, where `N` is the number of spines in each plane. Other network links have no plane.
- Scale-up links have a plane and a scale-up group (see [Scale-Up Groups](#scale-up-groups)). Other links have no scale-up group.

#### Example row

```csv
component_name,component_type,tier,subtier,node_id,net_device,dev_id,peer_component_name,peer_component_type,peer_tier,peer_subtier,peer_node_id,peer_net_device,peer_dev_id,linkspeed_Gbps,prop_delay_ns,pod,rack,pod_config_name,peer_pod,peer_rack,peer_pod_config_name,plane,scaleup_group
ScalaFabricSwitch-0-0-0-0-1-0,Switch,1,4294967295,0,scala_switch::ScalaSwitchEthNetDevice,0,ScalaRackSwitch-2-2-0-0-2-0,Switch,2,4294967295,2,scala_switch::ScalaSwitchEthNetDevice,0,800,500,0,,POD,0,0,POD,,
```

#### Scale-up groups

A run that has scale-up groups (Section II of the Chakra mapping file) also reports each group in this file. Each link has two rows, one from each end. Both rows of a scale-up link carry its group in `scaleup_group` and its plane in `plane`. An untyped group has the one plane 0. The switch, spine and crossbar ends have an empty `pod`, `rack` and `pod_config_name`, and the member end has those of its server.

A group that names neither a fabric type nor a number of planes in Section II is an untyped group. The report shows an untyped group as a two-tier switched tree:

- Each subgroup has one switch component, named `scaleup-g<group>-sg<subgroup>`. Its component type is `ScaleUpSwitch`, its tier is 5, and its subtier is 1.
- A group with more than one subgroup also has one spine component, named `scaleup-g<group>-spine`. Its component type is `ScaleUpSwitch`, its tier is 5, and its subtier is 2.
- The `net_device` of every end that is a switch or a spine, and the `peer_net_device` of every member row, is `ScaleUpSwitchPort`.

A group that names a fabric type and a number of planes in Section II is a typed group. A typed group reports each plane separately.

- A typed group of fabric type EPS has the same tree for each plane `<k>`, counted from 0. The switch of a subgroup is named `scaleup-g<group>-p<k>-sg<subgroup>`, and the spine is named `scaleup-g<group>-p<k>-spine`. Their component type, tier, subtier and `net_device` are the same as above.
- A typed group of fabric type `OCS-XBar` has one crossbar for each plane `<k>`, named `scaleup-g<group>-p<k>-ocs`. Its component type is `ScaleUpOpticalSwitch`, its tier is 5, and its subtier is 1. The `net_device` of the crossbar end of a link is `ScaleUpOpticalSwitchPort`. Each member has one link to the crossbar of each plane. A group of this type has no spine. The `peer_net_device` of every member row is `ScaleUpOpticalSwitchPort`. The OCS controller writes no rows.

| Link | `linkspeed_Gbps` | `prop_delay_ns` |
|------|------------------|-----------------|
| Member to subgroup switch | The group bandwidth. For a typed group, the group bandwidth divided by the number of planes | Cable latency + half the switch latency |
| Subgroup switch to spine | The group bandwidth multiplied by the number of members in the subgroup. For a typed group, the group bandwidth divided by the number of planes, multiplied by the number of members in the subgroup | Cable latency + switch latency |
| Member to crossbar | The group bandwidth divided by the number of planes | Cable latency + half the switch latency |

On the member side, the row has the component name, type, tier, subtier and node ID of the member's server. Its `dev_id` is the number of devices on the server's node, plus the plane for a typed group, so it does not match the `dev_id` of any other link of that server. Its `net_device` is the class name of the scale-up port. On the crossbar side, the `dev_id` of a member is the position of the member in its group, in ascending order of Chakra ID. On the side of a subgroup switch, the `dev_id` of a member is the position of the member in its subgroup, in ascending order of Chakra ID. The link to the spine has the next number after the last member. On the spine, the `dev_id` of a subgroup is the position of the subgroup in the group, in ascending order of subgroup ID.

The delay of a scale-up row includes the switch latency, so the delays along a path add up to the crossing time of the group. In an untyped group and on an EPS plane, between two members of one subgroup, the delays add up to 1 switch latency and 2 cable latencies. Between members of different subgroups, they add up to 3 switch latencies and 4 cable latencies. On an `OCS-XBar` plane, between any two members, they add up to 1 switch latency and 2 cable latencies, whatever their subgroups. A scale-up row writes `prop_delay_ns` with four digits after the decimal point, so a half picosecond is exact. A switch latency of 1 ps with a cable latency of 0 gives 0.0005 ns for a member link.

When the run uses more than one core, the file of each shard holds the rows that the shard owns. The shard of a member holds both rows of the link of that member. The shard of the lowest Chakra ID of a group holds the rows of the links between that group's switches and spine. The files of all shards together describe each link once from each end.

---

### Switch buffer statistics (scala-switch-agg-buff-stats)

Switch Buffer Statistics files capture aggregated buffer occupancy metrics for each switch in the network. Each row reports buffer utilization for a specific buffer type (egress queue, ingress usage, or shared buffer), direction, and pool on a given switch at a given time step. Every Scala Switch writes rows, every `BufferStatsReportInterval` (a switch attribute, default 1 ms).

**Filename pattern:** `{simulation}-scala-switch-agg-buff-stats-{shard-index}.csv`

#### Columns

| Column | Type | Description |
|--------|------|-------------|
| `time_utc` | String (ISO 8601) | Wall-clock timestamp |
| `sim_time_sec` | Float | Simulated time in seconds |
| `component_name` | String | Switch component name |
| `tier` | Integer | Network tier of the switch |
| `node_id` | Integer | Unique node identifier |
| `buff_id` | String | Buffer type: `egressQueue`, `ingressUsage`, or `sharedBuffer` |
| `uldl` | String | Direction: `ul` (uplink), `dl` (downlink), or `-` (shared) |
| `pool_num` | Integer | Buffer pool number |
| `total_bytes` | Integer | Sum of the occupancy samples in the interval (bytes): `ivl_mean_bytes` × `ivl_count` |
| `buffer_limit` | Integer | Size of the shared pool in bytes |
| `ivl_count` | Integer | Number of samples in the interval |
| `ivl_min_bytes` | Integer | Minimum buffer occupancy in the interval (bytes) |
| `ivl_max_bytes` | Integer | Maximum buffer occupancy in the interval (bytes) |
| `ivl_mean_bytes` | Float | Mean buffer occupancy in the interval (bytes) |
| `ivl_max_pct_util` | Float | Maximum occupancy in the interval as a percentage of `buffer_limit`, clamped at 100 |
| `ivl_mean_pct_util` | Float | Mean occupancy in the interval as a percentage of `buffer_limit`, clamped at 100 |

#### Example row

```csv
time_utc,sim_time_sec,component_name,tier,node_id,buff_id,uldl,pool_num,total_bytes,buffer_limit,ivl_count,ivl_min_bytes,ivl_max_bytes,ivl_mean_bytes,ivl_max_pct_util,ivl_mean_pct_util
2026-01-13T19:11:39Z,0.003,ScalaFabricSwitch-0-0-0-0-1-0,1,0,egressQueue,dl,0,8856181,213308112,18114,0,5530,488.914,0.00259249,0.000229205
```

---

### Packet trimming statistics (packet-trimming-stats)

Packet Trimming Statistics files count, for each switch, the packets it trimmed and the trimmable and trimmed packets it handled. The file is written only when a switch has both `TrimmingEnabled` and `TrimmingStatsEnabled` set to `true` (both default `false`). It has a row for each such switch every `NDStatsReportInterval`. [Egress admission](./models/scala-switch/packet-handling.md#egress-admission) describes which packets a switch trims.

A packet is counted as received when the switch admits it at ingress, and as sent when the switch admits it to an egress queue. Each count has a cumulative column, prefixed `total_`, and an interval column, prefixed `ivl_`.

**Filename pattern:** `{simulation}-packet-trimming-stats-{shard-index}.csv`

#### Columns

| Column | Type | Description |
|--------|------|-------------|
| `time_utc` | String (ISO 8601) | Wall-clock timestamp |
| `sim_time_sec` | Float | Simulated time in seconds |
| `switch_name` | String | Switch component name |
| `switch_id` | Integer | Node ID of the switch |
| `total_trim_events`, `ivl_trim_events` | Integer | Packets the switch trimmed |
| `total_trimmable_pkts_sent`, `ivl_trimmable_pkts_sent` | Integer | Packets marked trimmable sent |
| `total_trimmable_pkts_rcvd`, `ivl_trimmable_pkts_rcvd` | Integer | Packets marked trimmable received |
| `total_trimmable_rtx_pkts_sent`, `ivl_trimmable_rtx_pkts_sent` | Integer | Retransmitted packets marked trimmable sent |
| `total_trimmable_rtx_pkts_rcvd`, `ivl_trimmable_rtx_pkts_rcvd` | Integer | Retransmitted packets marked trimmable received |
| `total_trimmed_pkts_sent`, `ivl_trimmed_pkts_sent` | Integer | Packets marked trimmed sent |
| `total_trimmed_pkts_rcvd`, `ivl_trimmed_pkts_rcvd` | Integer | Packets marked trimmed received |
| `total_trimmed_last_hop_pkts_sent`, `ivl_trimmed_last_hop_pkts_sent` | Integer | Packets marked trimmed on the last hop sent |

#### Example header

```csv
time_utc,sim_time_sec,switch_name,switch_id,total_trim_events,ivl_trim_events,total_trimmable_pkts_sent,ivl_trimmable_pkts_sent,total_trimmable_pkts_rcvd,ivl_trimmable_pkts_rcvd,total_trimmable_rtx_pkts_sent,ivl_trimmable_rtx_pkts_sent,total_trimmable_rtx_pkts_rcvd,ivl_trimmable_rtx_pkts_rcvd,total_trimmed_pkts_sent,ivl_trimmed_pkts_sent,total_trimmed_pkts_rcvd,ivl_trimmed_pkts_rcvd,total_trimmed_last_hop_pkts_sent,ivl_trimmed_last_hop_pkts_sent
```

---

### Global simulation report (global-sim-report)

The Global Simulation Report is a JSON file with run totals for the devices of one shard: packets, bytes, and drops from the same counters as `nd-stats`, and PFC frames and pause time from the same counters as `ecn-pfc-stats`. It is written when `nd-stats` or `ecn-pfc-stats` is on. The simulator rewrites it during the run, so the file holds the totals at its last update.

**Filename pattern:** `{simulation}-global-sim-report-{shard-index}.json`

The file has two objects, `net-device` and `pfc`. Each holds one `global` entry, which sums every device of the shard, and one entry for each tier and direction: `ssw-ul`, `ssw-dl`, `fsw-ul`, `fsw-dl`, `rsw-ul`, `rsw-dl`, `nic-ul`, and `nic-dl`. The file can hold more entries; this page describes only these.

| Object | Key | Description |
|--------|-----|-------------|
| `net-device` | `packets-tx`, `packets-rx` | Packets transmitted and received |
| `net-device` | `bytes-tx-gb`, `bytes-rx-gb` | Bytes transmitted and received (GB) |
| `net-device` | `egress-drops-tx`, `ingress-drops-rx` | Transmit-side and receive-side drops |
| `pfc` | `total-xoff-tx`, `total-xoff-rx`, `total-xon-tx`, `total-xon-rx` | PFC XOFF and XON frames transmitted and received |
| `pfc` | `total-time-paused-usec` (`global`), `time-paused-usec` (tier entries) | Time paused due to PFC (microseconds) |
| both | `direction` (tier entries) | `ul` or `dl` |

---

## Diagnostic files

The results list also holds files that you can ignore. Most of them report on the simulator's own run rather than on the simulated network:

- `scala-remote-trace-log`: a trace of the messages between simulator processes. It is written only when the global `EnableRemoteTraceLogging` is `true` (default `false`). Each message adds a row on the sending process and on each receiving process, so on a large run the file can reach hundreds of gigabytes, enough to fill the run's disk and stop the simulation.
- `cpu-mem-stats`: CPU and memory use of the machine running the simulator.
- `scala-switch-buff-stats`: per-port switch buffer occupancy.
- `.log` files: the simulator's logs.
- `generic-workload-config.csv`: the settings the simulator ran with.
- Objects under `summary/` and `parquet/`: the platform's caches for [Get computed results](#get-computed-results).

This page does not describe any other file in the results list.
