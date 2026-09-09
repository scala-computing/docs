---
title: "Simulation Output Files"
description: "The Scala NS3 network simulator produces CSV output files during each simulation run. These files capture detailed metrics about MPI communication, network..."
---

The Scala NS3 network simulator produces CSV output files during each simulation run. These files capture detailed metrics about MPI communication, network device behavior, switch buffer utilization, application performance, and more. Each file type is identified by a naming pattern embedded in its filename.

All output files are stored in S3 and can be accessed through the results API after a simulation completes.

## How to Access Output Files

Use the simulation results endpoints to list and download output files:

**List result files:**

```bash
curl -X GET "https://api.scalacomputing.com/api/v1/simulations/{sim_id}/results" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

**Download a specific file via presigned URL:**

```bash
curl -X GET "https://api.scalacomputing.com/api/v1/simulations/{sim_id}/results/download-url?fileName={file_name}" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

**Retrieve computed summary statistics (nd-stats, perf, pfc only):**

```bash
curl -X GET "https://api.scalacomputing.com/api/v1/simulations/{sim_id}/results/data?metric=nd-stats&tier=all" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

See the [Simulations API Reference](./api-reference/simulations.md) for full endpoint documentation.

## File Naming Convention

Output files follow the pattern:

```
{simulation-name}-{file-type}-{shard-index}.csv
```

For example, a simulation named `my-sim-2026-03-17` produces files like:
- `my-sim-2026-03-17-mpi-stats-0.csv`
- `my-sim-2026-03-17-nd-stats-0.csv`
- `my-sim-2026-03-17-nd-stats-1.csv`

The shard index (`-0`, `-1`, ...) allows the simulator to split large output across multiple files.

## File Types Overview

| File Type | Name Pattern | Description |
|-----------|-------------|-------------|
| [MPI Statistics](#mpi-statistics-mpi-stats) | `*-mpi-stats-*.csv` | MPI packet and transaction counters per time step |
| [MPI Profiling](#mpi-profiling-mpi-profiling) | `*-mpi-profiling-*.csv` | Per-core MPI engine profiling counters and timing |
| [Remote Trace Log](#remote-trace-log-scala-remote-trace-log) | `*-scala-remote-trace-log-*.csv` | Trace-level log of remote MPI operations |
| [Network Device Statistics](#network-device-statistics-nd-stats) | `*-nd-stats-*.csv` | Per-interface packet/byte counters, throughput, utilization |
| [Performance Metrics](#performance-metrics-perf) | `*-perf-*.csv` | Application-level latency and transfer measurements |
| [ECN/PFC Statistics](#ecnpfc-statistics-ecn-pfc-stats) | `*-ecn-pfc-stats-*.csv` | Priority Flow Control and ECN marking counters |
| [Application Report](#application-report-application-report) | `*-application-report-*.csv` | Socket and application binding information |
| [Beacon](#beacon-beacon) | `*-beacon-*.csv` | Periodic heartbeat counters for client/server traffic |
| [CPU/Memory Statistics](#cpumemory-statistics-cpu-mem-stats) | `*-cpu-mem-stats-*.csv` | Host-level CPU and memory utilization of the simulator process |
| [RoCE Transport Statistics](#roce-transport-statistics-roce-transport-stats) | `*-roce-transport-stats-*.csv` | RDMA over Converged Ethernet transport-layer counters |
| [Tier Hop](#tier-hop-tier-hop) | `*-tier-hop-*.csv` | Packet counts traversing each network tier |
| [Topology Report](#topology-report-topology-report) | `*-topology-report-*.csv` | Static topology describing device interconnections |
| [Switch Buffer Statistics](#switch-buffer-statistics-scala-switch-agg-buff-stats) | `*-scala-switch-agg-buff-stats-*.csv` | Aggregated switch buffer occupancy and utilization |

## Detailed File Schemas

### MPI Statistics (mpi-stats)

MPI Statistics files record per-time-step counters for MPI packet and transaction activity across the simulated cluster. Use this file to analyze MPI communication volume over time.

**Filename pattern:** `{simulation}-mpi-stats-{shard}.csv`

#### Columns

| Column | Type | Description |
|--------|------|-------------|
| `job_id` | String | Identifier for the simulation job |
| `time_utc` | String (ISO 8601) | Wall-clock timestamp when the row was emitted |
| `sim_time_sec` | Float | Simulated time in seconds |
| `mpi_pkt_tx` | Integer | Cumulative MPI packets transmitted |
| `mpi_pkt_rx` | Integer | Cumulative MPI packets received |
| `mpi_tr_tx` | Integer | Cumulative MPI transactions (operations) transmitted |
| `mpi_tr_rx` | Integer | Cumulative MPI transactions (operations) received |
| `mpi_app_tag_tx` | Integer | Cumulative MPI application-tagged messages transmitted |
| `mpi_app_tag_rx` | Integer | Cumulative MPI application-tagged messages received |
| `mpi_null` | Integer | Count of MPI null/idle events |

#### Example Row

```csv
job_id,time_utc,sim_time_sec,mpi_pkt_tx,mpi_pkt_rx,mpi_tr_tx,mpi_tr_rx,mpi_app_tag_tx,mpi_app_tag_rx,mpi_null
example,2026-01-13T19:11:39Z,0.003,0,0,0,0,0,0,0
```

---

### MPI Profiling (mpi-profiling)

MPI Profiling files capture per-core performance counters and timing breakdowns for the MPI simulation engine. Each row represents one core's profiling snapshot at a given simulation time step. Use this file to identify bottlenecks in the parallel simulation engine.

**Filename pattern:** `{simulation}-mpi-profiling-{shard}.csv`

#### Columns

| Column | Type | Description |
|--------|------|-------------|
| `job_id` | String | Identifier for the simulation job |
| `time_utc` | String (ISO 8601) | Wall-clock timestamp when the row was emitted |
| `sim_time_sec` | Float | Simulated time in seconds |
| `core` | Integer | Core/rank index within the MPI engine |
| `cntWhile` | Integer | Number of main-loop iterations |
| `cntCalcGrantedTime` | Integer | Count of granted-time calculations |
| `cntWhileNextGreater` | Integer | Iterations where next event time exceeded current window |
| `cntWhileIsLocalFinished` | Integer | Iterations where the local partition finished early |
| `cntWhileBoth` | Integer | Iterations matching both NextGreater and IsLocalFinished |
| `cntRxEqTx` | Integer | Iterations where receive count equaled transmit count |
| `cntRxEqTxLookAheadEqMax` | Integer | RxEqTx iterations where lookahead equaled max |
| `cntRxEqTxLookAheadNeqMax` | Integer | RxEqTx iterations where lookahead did not equal max |
| `cntRxNeqTx` | Integer | Iterations where receive count did not equal transmit count |
| `num_processed_events` | Integer | Cumulative number of discrete events processed |
| `testTime_sec` | Float | Wall-clock time spent in the test phase (seconds) |
| `mtestTime_sec` | Float | Wall-clock time for the MPI test phase (seconds) |
| `timeReceive_sec` | Float | Wall-clock time spent in MPI receive calls (seconds) |
| `mtimeReceive_sec` | Float | Wall-clock time for MPI-level receive (seconds) |
| `timeSend_sec` | Float | Wall-clock time spent in MPI send calls (seconds) |
| `mtimeSend_sec` | Float | Wall-clock time for MPI-level send (seconds) |
| `timeGather_sec` | Float | Wall-clock time spent in MPI gather calls (seconds) |
| `mtimeGather_sec` | Float | Wall-clock time for MPI-level gather (seconds) |
| `timeEvents_sec` | Float | Wall-clock time spent processing events (seconds) |
| `mtimeEvents_sec` | Float | Wall-clock time for MPI-level event processing (seconds) |

#### Example Row

```csv
job_id,time_utc,sim_time_sec,core,cntWhile,cntCalcGrantedTime,cntWhileNextGreater,cntWhileIsLocalFinished,cntWhileBoth,cntRxEqTx,cntRxEqTxLookAheadEqMax,cntRxEqTxLookAheadNeqMax,cntRxNeqTx,num_processed_events,testTime_sec,mtestTime_sec,timeReceive_sec,mtimeReceive_sec,timeSend_sec,mtimeSend_sec,timeGather_sec,mtimeGather_sec,timeEvents_sec,mtimeEvents_sec
example,2026-01-13T19:11:39Z,0.003,0,1498198,0,0,0,0,0,0,0,0,1498198,5.84884437,9.62542344,0,0,0,0,0,0,0,0
```

---

### Remote Trace Log (scala-remote-trace-log)

Remote Trace Log files record individual remote MPI operations at trace level, capturing the source and destination ranks, buffer contents, and channel timing for each remote call. This file is useful for debugging specific MPI communication patterns and diagnosing latency in remote operations.

**Filename pattern:** `{simulation}-scala-remote-trace-log-{shard}.csv`

#### Columns

| Column | Type | Description |
|--------|------|-------------|
| `time_utc` | String (ISO 8601) | Wall-clock timestamp when the event was logged |
| `sim_time` | Float | Simulated time at which the remote operation occurred |
| `source` | String | Source identifier for the trace event |
| `remote_trace_client_id` | String | Unique identifier for the remote trace client |
| `src_rank` | Integer | Source MPI rank initiating the operation |
| `dest_rank` | Integer | Destination MPI rank receiving the operation |
| `string_buf` | String | String representation of the buffer contents |
| `buf_size` | Integer | Size of the buffer in bytes |
| `status` | String | Status of the remote operation |
| `channel_delay` | Float | Channel delay for the remote operation (units vary by configuration) |

> **Note:** This file may be empty if no remote MPI trace events occurred during the simulation.

#### Example Header

```csv
time_utc,sim_time,source,remote_trace_client_id,src_rank,dest_rank,string_buf,buf_size,status,channel_delay
```

---

### Network Device Statistics (nd-stats)

Network Device Statistics files contain per-interface, per-time-step counters for every network device in the simulated topology. Each row captures packet counts, byte volumes, throughput rates, link utilization percentages, and drop counters. This is one of the three metric types that supports server-side summary aggregation via the results data endpoint.

**Filename pattern:** `{simulation}-nd-stats-{shard}.csv`

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
| `tier` | Int32 | Network tier (0=Spine, 1=Fabric, 2=Rack, 3=NIC) |
| `subtier` | Int64 | Sub-tier identifier |
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

#### Network Tiers

| Tier Value | Abbreviation | Description |
|------------|-------------|-------------|
| 0 | SSW | Spine Switch |
| 1 | FSW | Fabric Switch |
| 2 | RSW | Rack Switch |
| 3 | NIC | Network Interface Card |

#### Example Row

```csv
time_utc,sim_time_sec,component_name,nd_name,component_type,ifid,uldl,node_id,tier,subtier,total_packets_tx,total_packets_rx,total_bytes_tx_GB,total_bytes_rx_GB,interval_bytes_tx_MB,interval_bytes_rx_MB,tx_rate_Gbps,rx_rate_Gbps,tx_link_percent_util,rx_link_percent_util,tx_drops,rx_drops
2026-01-13T19:11:38Z,0.003,ScalaFabricSwitch-0-0-0-0-1-0,scala_switch::ScalaSwitchEthNetDevice,Switch,0,dl,0,1,4294967295,115,112,7.3062e-05,7.4146e-05,0.073062,0.074146,0.584496,0.593168,0.073062,0.074146,0,0
```

---

### Performance Metrics (perf)

Performance Metrics files capture application-level latency and transfer size measurements for each completed transaction. Each row represents one transaction observed by a traffic generator. This is one of the three metric types that supports server-side summary aggregation.

**Filename pattern:** `{simulation}-perf-{shard}.csv`

#### Columns

| Column | Type | Description |
|--------|------|-------------|
| `job_id` | String | Identifier for the simulation job |
| `time_utc` | String | Wall-clock timestamp (ISO 8601) |
| `sim_time_sec` | Float64 | Simulated time in seconds |
| `metric_name` | String | Name of the performance metric (e.g., `ScalaApp`) |
| `perf_metric_usec` | Float64 | Measured latency in microseconds |
| `metric_path` | String | Hierarchical path for the metric |
| `metric_aggregation_id` | String | Aggregation group identifier |
| `node_id` | Int64 | Node that recorded the metric |
| `transaction_number` | Int64 | Sequential transaction number |
| `start_time_sec` | Float64 | Simulation time when the transaction started (seconds) |
| `transfer_size_b` | Float64 | Transfer size in bytes |
| `application_name` | String | Name of the application that generated the transaction |

#### Example Row

```csv
job_id,time_utc,sim_time_sec,metric_name,perf_metric_usec,metric_path,metric_aggregation_id,node_id,transaction_number,start_time_sec,transfer_size_b,application_name
example,2026-01-13T19:11:40Z,0.00500019,ScalaApp,2.83117,NAN,NAN,478,4,0.00499736,NAN,TrafficGeneratorClient-P:0-R:3-S:24-R:0-C:0-1554
```

---

### ECN/PFC Statistics (ecn-pfc-stats)

ECN/PFC Statistics files record Priority Flow Control (PFC) frame counters and Explicit Congestion Notification (ECN) marking counts per interface, per priority level. Use this data to analyze congestion events and flow control behavior. This is one of the three metric types that supports server-side summary aggregation.

**Filename pattern:** `{simulation}-ecn-pfc-stats-{shard}.csv`

#### Columns

| Column | Type | Description |
|--------|------|-------------|
| `job_id` | String | Identifier for the simulation job |
| `time_utc` | String | Wall-clock timestamp (ISO 8601) |
| `sim_time_sec` | Float64 | Simulated time in seconds |
| `node_id` | Int64 | Unique node identifier |
| `component_name` | String | Name of the network component |
| `component_type` | String | Type of component (e.g., `Switch`) |
| `ifid` | Int64 | Interface ID |
| `uldl` | String | Direction: `ul` (uplink) or `dl` (downlink) |
| `tier` | Int32 | Network tier (0=Spine, 1=Fabric, 2=Rack, 3=NIC) |
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
| `percent_xon` | Float64 | Percentage of time the interface was in XON state |
| `xoff_egr_occ_ratio` | Float64 | Ratio of XOFF time attributed to egress occupancy |
| `pkts_ecn_marked` | Int64 | Number of packets marked with ECN |

> **Note:** The microsecond-duration columns are fractional (`Float64`); the simulator derives them from nanosecond timers divided by 1000. Server-side PFC summary values (e.g. `totalTimePausedUsec`) may therefore be non-integer.

#### Example Header

```csv
job_id,time_utc,sim_time_sec,node_id,component_name,component_type,ifid,uldl,tier,subtier,priority,pfcxoff_tx,pfcxoff_rx,pfcxon_tx,pfcxon_rx,pfc_time_paused_usec,interval_total_time_xoff_usec,interval_max_time_xoff_usec,interval_total_time_xoff_egr_occ_usec,interval_max_time_xoff_egr_occ_usec,total_xoff_time_egr_occ_usec,percent_xon,xoff_egr_occ_ratio,pkts_ecn_marked
```

---

### Application Report (application-report)

Application Report files describe the socket bindings established at simulation startup. Each row maps a host component and node to its application type, socket role, DSCP/ECT settings, and peer node.

**Filename pattern:** `{simulation}-application-report-{shard}.csv`

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

#### Example Row

```csv
job_id,time_utc,component_name,node_id,socket_type,app_typeid,app_name,dscp,ect,peer_node_id
example,2026-01-13T19:11:37Z,ScalaHost-0-0-0-0,262,CLIENT,ns3::ScalaTrafficGeneratorApplication,TrafficGeneratorClient-P:0-R:0-S:0-R:0-C:0-1,0,0,386
```

---

### Beacon (beacon)

Beacon files record periodic heartbeat counters tracking client and server packet activity and data goodput over each reporting interval.

**Filename pattern:** `{simulation}-beacon-{shard}.csv`

#### Columns

| Column | Type | Description |
|--------|------|-------------|
| `job_id` | String | Identifier for the simulation job |
| `time_utc` | String (ISO 8601) | Wall-clock timestamp |
| `sim_time_sec` | Float | Simulated time in seconds |
| `interval_duration_sec` | Float | Duration of the reporting interval (seconds) |
| `tx_client` | Integer | Cumulative packets transmitted by clients |
| `rx_client` | Integer | Cumulative packets received by clients |
| `tx_server` | Integer | Cumulative packets transmitted by servers |
| `rx_server` | Integer | Cumulative packets received by servers |
| `rx_bytes_reader` | Integer | Cumulative bytes received by reader applications |
| `rx_bytes_writer` | Integer | Cumulative bytes received by writer applications |
| `goodput_reader_MB_per_sec` | Float | Reader goodput for the interval (MB/s) |
| `goodput_writer_MB_per_sec` | Float | Writer goodput for the interval (MB/s) |

#### Example Row

```csv
job_id,time_utc,sim_time_sec,interval_duration_sec,tx_client,rx_client,tx_server,rx_server,rx_bytes_reader,rx_bytes_writer,goodput_reader_MB_per_sec,goodput_writer_MB_per_sec
example,2026-01-13T19:11:42Z,0.006,1,5027,5027,5027,5027,0,1314708,0,52.5883
```

---

### CPU/Memory Statistics (cpu-mem-stats)

CPU/Memory Statistics files capture host-level resource utilization of the simulator process itself, including virtual memory, RSS, system memory, swap, and CPU time breakdowns.

**Filename pattern:** `{simulation}-cpu-mem-stats-{shard}.csv`

#### Columns

| Column | Type | Description |
|--------|------|-------------|
| `job_id` | String | Identifier for the simulation job |
| `time_utc` | String (ISO 8601) | Wall-clock timestamp |
| `sim_time_sec` | Float | Simulated time in seconds |
| `host_name` | String | Hostname of the machine running the simulator |
| `pid` | Integer | Process ID of the simulator |
| `core_vm_usage_M` | Float | Virtual memory used by the process (MB) |
| `core_rss_M` | Float | Resident set size of the process (MB) |
| `core_vm_usage_pct` | Float | Virtual memory usage as a percentage of system total |
| `core_rss_usage_pct` | Float | RSS usage as a percentage of system total |
| `sys_total_mem_M` | Float | Total system memory (MB) |
| `sys_free_mem_M` | Float | Free system memory (MB) |
| `sys_used_mem_M` | Float | Used system memory (MB) |
| `sys_mem_usage_pct` | Float | System memory usage percentage |
| `sys_swap_usage_pct` | Float | System swap usage percentage |
| `user_time_ticks` | Integer | CPU time in user mode (ticks) |
| `kernel_time_ticks` | Integer | CPU time in kernel mode (ticks) |
| `total_time_ticks` | Integer | Total CPU time (ticks) |
| `elapse_time_ticks` | Integer | Elapsed wall-clock time (ticks) |
| `user_time_pct` | Float | User CPU time as a percentage |
| `system_time_pct` | Float | System/kernel CPU time as a percentage |
| `total_time_pct` | Float | Total CPU time as a percentage |

#### Example Header

```csv
job_id,time_utc,sim_time_sec,host_name,pid,core_vm_usage_M,core_rss_M,core_vm_usage_pct,core_rss_usage_pct,sys_total_mem_M,sys_free_mem_M,sys_used_mem_M,sys_mem_usage_pct,sys_swap_usage_pct,user_time_ticks,kernel_time_ticks,total_time_ticks,elapse_time_ticks,user_time_pct,system_time_pct,total_time_pct
```

---

### RoCE Transport Statistics (roce-transport-stats)

RoCE Transport Statistics files record detailed RDMA over Converged Ethernet (RoCEv2) transport-layer counters per network device. This includes packet counts by verb type, acknowledgment/NACK tracking, reordering buffer statistics, work queue entries, retransmissions, and round-trip time measurements.

**Filename pattern:** `{simulation}-roce-transport-stats-{shard}.csv`

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

#### Example Header

```csv
time_utc,sim_time_sec,component_name,node_id,num_pkts_sent,ivl_num_pkts_sent,num_verb_pkts_sent,ivl_num_verb_pkts_sent,num_non_verb_pkts_sent,ivl_num_non_verb_pkts_sent,num_conn_pkts_sent,ivl_num_conn_pkts_sent,num_conn_pkts_recvd,ivl_num_conn_pkts_recvd,rdma_write_cnt,ivl_rdma_write_cnt,rdma_send_cnt,ivl_rdma_send_cnt,rdma_atomic_cnt,ivl_rdma_atomic_cnt,num_acks_sent,ivl_num_acks_sent,num_acks_recvd,ivl_num_acks_recvd,num_sacks_sent,ivl_num_sacks_sent,num_sacks_recvd,ivl_num_sacks_recvd,num_nacks_sent,ivl_num_nacks_sent,num_nacks_recvd,ivl_num_nacks_recvd,num_writes_recvd,ivl_num_writes_recvd,num_sends_recvd,ivl_num_sends_recvd,num_cnp_sent,ivl_num_cnp_sent,num_cnp_recvd,ivl_num_cnp_recvd,num_seq_out_of_order,ivl_num_seq_out_of_order,ivl_min_size_reorder_buf_psn,max_size_reorder_buf_psn,ivl_max_size_reorder_buf_psn,avg_size_reorder_buf_psn,ivl_avg_size_reorder_buf_psn,num_wqe_created,ivl_num_wqe_created,num_wqe_completed,ivl_num_wqe_completed,num_rtx,ivl_num_rtx,num_rto,ivl_num_rto,ivl_min_calc_rtt_usec,max_calc_rtt_usec,ivl_max_calc_rtt_usec,avg_calc_rtt_usec,ivl_avg_calc_rtt_usec
```

---

### Tier Hop (tier-hop)

Tier Hop files record the cumulative and interval packet counts traversing each network tier (spine, fabric, rack). Use this to understand traffic distribution across the network hierarchy.

**Filename pattern:** `{simulation}-tier-hop-{shard}.csv`

#### Columns

| Column | Type | Description |
|--------|------|-------------|
| `job_id` | String | Identifier for the simulation job |
| `time_utc` | String (ISO 8601) | Wall-clock timestamp |
| `sim_time_sec` | Float | Simulated time in seconds |
| `spine` | Integer | Cumulative packets through the spine tier |
| `fabric` | Integer | Cumulative packets through the fabric tier |
| `rack` | Integer | Cumulative packets through the rack tier |
| `spineInt` | Integer | Packets through the spine tier in the current interval |
| `fabricInt` | Integer | Packets through the fabric tier in the current interval |
| `rackInt` | Integer | Packets through the rack tier in the current interval |

#### Example Row

```csv
job_id,time_utc,sim_time_sec,spine,fabric,rack,spineInt,fabricInt,rackInt
example,2026-01-13T19:11:39Z,0.003,0,18137,42672,0,18137,42672
```

---

### Topology Report (topology-report)

Topology Report files describe the static physical topology of the simulated network. Each row represents one link between two devices, including tier placement, device type, link speed, and propagation delay. This file is emitted once per simulation and does not change over time.

**Filename pattern:** `{simulation}-topology-report-{shard}.csv`

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

#### Example Row

```csv
component_name,component_type,tier,subtier,node_id,net_device,dev_id,peer_component_name,peer_component_type,peer_tier,peer_subtier,peer_node_id,peer_net_device,peer_dev_id,linkspeed_Gbps,prop_delay_ns
ScalaFabricSwitch-0-0-0-0-1-0,Switch,1,4294967295,0,scala_switch::ScalaSwitchEthNetDevice,0,ScalaRackSwitch-2-2-0-0-2-0,Switch,2,4294967295,2,scala_switch::ScalaSwitchEthNetDevice,0,800,500
```

---

### Switch Buffer Statistics (scala-switch-agg-buff-stats)

Switch Buffer Statistics files capture aggregated buffer occupancy metrics for each switch in the network. Each row reports buffer utilization for a specific buffer type (egress queue, ingress usage, or shared buffer), direction, and pool on a given switch at a given time step.

**Filename pattern:** `{simulation}-scala-switch-agg-buff-stats-{shard}.csv`

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
| `total_bytes` | Integer | Total bytes that have passed through the buffer |
| `buffer_limit` | Integer | Maximum buffer capacity in bytes |
| `ivl_count` | Integer | Number of samples in the interval |
| `ivl_min_bytes` | Integer | Minimum buffer occupancy in the interval (bytes) |
| `ivl_max_bytes` | Integer | Maximum buffer occupancy in the interval (bytes) |
| `ivl_mean_bytes` | Float | Mean buffer occupancy in the interval (bytes) |
| `ivl_max_pct_util` | Float | Maximum buffer utilization in the interval (fraction) |
| `ivl_mean_pct_util` | Float | Mean buffer utilization in the interval (fraction) |

#### Example Row

```csv
time_utc,sim_time_sec,component_name,tier,node_id,buff_id,uldl,pool_num,total_bytes,buffer_limit,ivl_count,ivl_min_bytes,ivl_max_bytes,ivl_mean_bytes,ivl_max_pct_util,ivl_mean_pct_util
2026-01-13T19:11:39Z,0.003,ScalaFabricSwitch-0-0-0-0-1-0,1,0,egressQueue,dl,0,8856181,213308112,18114,0,5530,488.914,0.00259249,0.000229205
```

---

### Per-Port Switch Buffer Statistics (scala-switch-buff-stats)

Per-Port Switch Buffer Statistics files carry the same measurements as the aggregate file above,
but broken out per interface rather than summed across the switch. This is the only view of an
individual queue's occupancy against that queue's own limit.

Emitted only when the switch's `ReportPerPortBufferStats` attribute is enabled; it is off by
default, so most runs will not produce this file.

**Filename pattern:** `{simulation}-scala-switch-buff-stats-{shard}.csv`

#### Columns

| Column | Type | Description |
|--------|------|-------------|
| `time_utc` | String (ISO 8601) | Wall-clock timestamp when the interval was reported |
| `sim_time_sec` | Float | Simulated time at the end of the reporting interval |
| `component_name` | String | Switch component name |
| `tier` | Integer | Network tier of the switch |
| `node_id` | Integer | Unique node identifier |
| `buff_id` | String | Buffer type: `egressQueue`, `ingressUsage`, or `hdrm` |
| `ifid` | Integer | Interface identifier — the port this row describes |
| `uldl` | String | Direction: `ul` (uplink) or `dl` (downlink) |
| `pool_num` | Integer | Buffer pool number |
| `buff_limit` | Integer | This port and pool's own limit in bytes |
| `ivl_count` | Integer | Number of samples in the interval |
| `ivl_min_bytes` | Integer | Minimum occupancy in the interval (bytes) |
| `ivl_max_bytes` | Integer | Maximum occupancy in the interval (bytes) |
| `ivl_mean_bytes` | Float | Mean occupancy in the interval (bytes) |
| `ivl_max_pct_util` | Float | Maximum utilization in the interval, clamped at 100 |
| `ivl_mean_pct_util` | Float | Mean utilization in the interval, clamped at 100 |

> **Note:** Both percentage columns are clamped at 100, so an occupancy that exceeds its
> configured limit is not visible in them. Compute the ratio from `ivl_max_bytes` and
> `buff_limit` if you need the true figure.

#### Example Row

```csv
time_utc,sim_time_sec,component_name,tier,node_id,buff_id,ifid,uldl,pool_num,buff_limit,ivl_count,ivl_min_bytes,ivl_max_bytes,ivl_mean_bytes,ivl_max_pct_util,ivl_mean_pct_util
2026-08-18T20:09:09Z,0.001,Pod-1-0.Rack-1.ScalaSwitchRack-2-0-0,2,0,egressQueue,0,dl,0,40960000,43764,4136,127066192,7.37875e+07,100,100
```
