---
title: "Records"
description: "This page lists the records a simulation writes for the Scala UET NIC, when each is written, and what turns it on."
---

This page lists the records a simulation writes for the Scala UET NIC, when each is written, and what turns it on.

For every record but `pcie-stats`, it also describes what one row holds. The records show the behavior the other pages describe: [Transport](./transport.md) for packets, acknowledgements, and retransmissions; [Multipath](./multipath.md) for the entropy values; [Congestion control](./congestion-control.md) for the congestion window and round-trip times; and [Buffers and PFC](./buffers-and-pfc.md) for the buffer pools and pause frames. [Configuration](./configuration.md) lists the attributes that turn the optional records on.

## Records a simulation writes

With the defaults, a simulation writes these records for every Scala UET NIC. Each file is named after the record, in the form `{simulation-name}-{record}-{shard-index}.csv`, as [Simulation output files](../../simulation-output-files.md) describes. When a simulation runs as several simulator processes, each process writes its own shard.

| Record | One row for | Written | Turned on by |
| --- | --- | --- | --- |
| `uet-transport-stats` | Each UET NIC, combining all its PDCs | Every `NDStatsReportInterval` | `EnableTransportStats` on the NIC (default `true`) |
| `nd-stats` | The NIC's network port | Every `NDStatsReportInterval` | The global `EnableNetDeviceStatLogging` (default `true`) |
| `ecn-pfc-stats` | Each traffic class with a receive pool, once the NIC has sent or received a PFC frame for it | Every `NDStatsReportInterval` | The global `EnablePFCECNStatsLogging` (default `true`) |

These records are written only when turned on ([Records turned on by attributes](#records-turned-on-by-attributes)):

| Record | One row for | Written | Turned on by |
| --- | --- | --- | --- |
| `net-buff-stats` | Each of the NIC's receive and transmit pools | Every `NetworkBufferStatsReportingInterval` | `EnableBufferStats` on the NIC and the global `EnableNetworkBufferStatsReporting` (both default `false`) |
| `uet-events` | Each PDC or work-request event | As each event happens | `EnableEventLogging` on the NIC (default `false`) |
| `uet-monitor` | Each UET packet that a selected PDC sends or receives | As each packet is sent or processed | `EnableUETMonitor`, `TriggerLoggingOnRttEnable`, or `TriggerLoggingOnCWindPen` (all default `false`) |
| `pcie-stats` | Each end of the NIC's PCIe link | Every `NDStatsReportInterval` | The global `EnablePCIeStatsLogging` (default `false`) |

A periodic record writes its first row at `StartTime` + `ClientStartOffset` + its interval of simulation time, and a row every interval after that. With the platform's defaults the first row is at 0.001 + 0.001 + 0.001 = 0.003 s.

`NDStatsReportInterval`, `NetworkBufferStatsReportingInterval`, `StartTime`, `ClientStartOffset`, and the switches marked global in the tables are global simulation parameters, not attributes of this model. They sit in the configuration's `SimulationParameters` and are changed with `PATCH /api/v1/configurations/{config_id}/parameters`; the two intervals are `double`s in seconds (default `0.001`).

## Getting the records

`GET /api/v1/simulations/{sim_id}/results` lists a simulation's result files, among them every record on this page; its `type` query parameter filters by file type, for example `csv`. `GET /api/v1/simulations/{sim_id}/results/download-url`, with the `fileName` query parameter set to a file's name, returns a link to download that file.

`GET /api/v1/simulations/{sim_id}/results/data` computes results on the platform from the record files and returns a link to download them. With `metric=uet-transport-stats` and `mode=timeSeries`, it computes a time series from the simulation's `uet-transport-stats` files; `timeSeries` is the only mode it serves for that metric. [Simulation output files](../../simulation-output-files.md) shows the same endpoint for `nd-stats`, and the [Simulations API reference](../../api-reference/simulations.md) lists its parameters.

Set `SIM_ID` to the simulation's ID and `FILE_NAME` to a name from the list:

```bash
curl -X GET "https://api.scalacomputing.com/api/v1/simulations/$SIM_ID/results?type=csv" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"

curl -X GET "https://api.scalacomputing.com/api/v1/simulations/$SIM_ID/results/download-url?fileName=$FILE_NAME" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"

curl -X GET "https://api.scalacomputing.com/api/v1/simulations/$SIM_ID/results/data?metric=uet-transport-stats&mode=timeSeries" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

## Identifying a NIC and a PDC

**A NIC** is identified by its FEP id, which is the NIC's node id in the simulation. It appears as `fep_id` in `uet-transport-stats`, as `FEP_ID` in `uet-events` and `uet-monitor`, and as `node_id` in `nd-stats`, `ecn-pfc-stats`, and `net-buff-stats`; `FilterOnFepId` takes the same number. In a Chakra workload simulation, the `NicNodeID` column of `chakra-mapping` gives the node id of each rank's NIC ([Placement records](../chakra-workload/records.md#placement-records)).

**A PDC** (packet delivery context) carries one direction of a connection between two NICs, with an end on each: the sending end and the receiving end ([Messages and packet delivery contexts](./transport.md#messages-and-packet-delivery-contexts)). Each end has its own PDC id, written `<IPv4 of that end>:<PDC id>`, for example `32.0.0.2:4500`, where the PDC id is a 16-bit number. `uet-events` and `uet-monitor` show the end on the NIC that wrote the row as `local_PDC_ID` and the other end as `remote_PDC_ID`.

- The receiving NIC assigns its PDC ids from 32,769 to 65,535.
- The Chakra workload assigns the sending end's ids from 1 to 32,767, gives each message its own PDC, and reuses an id after its message is done. An id therefore names a connection only for the time that connection is open.
- On the sending end, the id part of `remote_PDC_ID` reads `0` until the sender learns the receiving end's id from its first ACK.

## uet-transport-stats

`uet-transport-stats` has one row for each UET NIC whose `EnableTransportStats` is `true`, every `NDStatsReportInterval`. A row combines every PDC and every congestion window on the NIC: the record has no row for a single PDC or destination. A row has 69 columns. Columns that start with `ivl_` cover the interval since the previous row; the matching columns without the prefix cover the simulation so far, so the first row's `ivl_` columns cover everything before it.

The counters:

| Columns | What they hold |
| --- | --- |
| `time_utc`, `sim_time_sec` | When the row was written: wall-clock time, and simulation time in seconds |
| `component_name`, `fep_id` | The NIC: its component name and its FEP id |
| `total_pkts_sent`, `total_pkts_rcvd` | UET packets sent and received. Base-RTT probes are counted only in the probe columns, so `total_pkts_sent` is the sum of the write, ack, and nack columns for packets sent |
| `write_pkts_sent`, `write_pkts_rcvd` | Data packets sent, retransmissions included, and data packets received, trimmed ones included |
| `ack_pkts_sent`, `ack_pkts_rcvd` | ACKs sent and received, the ACKs of base-RTT probes included |
| `nack_pkts_sent`, `nack_pkts_rcvd` | NACKs sent and received. The NIC sends a NACK only for a trimmed data packet ([Trimmed packets and NACKs](./transport.md#trimmed-packets-and-nacks)) |
| `wqes_created`, `wqes_completed` | Work requests (messages) the host posted to the NIC, and work requests whose every byte has been acknowledged |
| `num_out_of_order_events_on_nic` | Data packets received ahead of the next expected PSN ([Receiving a packet](./transport.md#receiving-a-packet)) |
| `max_out_of_order_on_flow` | The most out-of-order packets one PDC held at once: over the simulation so far, and in the interval |
| `num_dup_events_on_nic` | Data packets received that had already been received ([Receiving a packet](./transport.md#receiving-a-packet)) |
| `probe_sent`, `probe_ack_rcvd` | Base-RTT probes sent, about one per message on each PDC, and the probe ACKs received ([Acknowledgements](./transport.md#acknowledgements)) |
| `num_rto`, `num_rtx`, `max_tx_count` | Retransmission. `num_rto` counts data packets whose retransmission timer expired while they were in flight, and `num_rtx` the data packets retransmitted, after a timer expiry or a NACK ([Retransmission timer](./transport.md#retransmission-timer)) |
| `m_marked_pkts_sent` | As a receiver: ACKs sent with the ECN echo set, probe ACKs included ([How the window responds to each ACK](./congestion-control.md#how-the-window-responds-to-each-ack)) |
| `m_marked_pkts_rcvd` | As a sender: data ACKs received with the ECN echo set that gave an RTT sample ([How the window responds to each ACK](./congestion-control.md#how-the-window-responds-to-each-ack)) |
| `num_quick_adapt` | Times Quick Adapt cut a congestion window ([Quick Adapt](./congestion-control.md#quick-adapt)) |

Every counter has an `ivl_` twin, for example `ivl_num_rtx`. Sent packets are counted when the transport hands them down to be sent, before they wait in the egress buffer. A received packet is counted after the receive buffer admits it; a packet the receive buffer drops is not counted here.

The samples come in blocks. In each column's name, whichever of `min`, `mean` (or `avg`), and `max` comes first says whether the column holds the least, the mean, or the greatest sample, over the interval for an `ivl_` column and over the simulation so far for the others: `ivl_min_calc_avg_q_delay_usec` is the least average queueing delay sampled in the interval.

| Columns | Sample |
| --- | --- |
| `ivl_calc_min_rcvr_cwnd_pen`, `calc_mean_rcvr_cwnd_pen`, `ivl_calc_mean_rcvr_cwnd_pen`, `ivl_calc_max_rcvr_cwnd_pen` | The receiver penalty, 0 to 127, carried in each data ACK that gives an RTT sample ([Receiver penalty](./congestion-control.md#receiver-penalty)) |
| `calc_min_cwnd_bytes`, `ivl_calc_min_cwnd_bytes`, `calc_mean_cwnd_bytes`, `ivl_calc_mean_cwnd_bytes`, `ivl_calc_max_cwnd_bytes` | The congestion window, in bytes, at each first transmission of a data packet ([The congestion window](./congestion-control.md#the-congestion-window)) |
| `ivl_min_calc_avg_q_delay_usec`, `avg_calc_avg_q_delay_usec`, `ivl_avg_calc_avg_q_delay_usec`, `max_calc_avg_q_delay_usec`, `ivl_max_calc_avg_q_delay_usec` | The average queueing delay congestion control keeps, in µs, at each data ACK that gives an RTT sample ([How the window responds to each ACK](./congestion-control.md#how-the-window-responds-to-each-ack)) |
| `ivl_min_calc_rtt_usec`, `max_calc_rtt_usec`, `ivl_max_calc_rtt_usec`, `avg_calc_rtt_usec`, `ivl_avg_calc_rtt_usec` | The RTT of each data ACK that gives an RTT sample, in µs: the time from the end of the data packet's serialization to the ACK's arrival, less the time the receiver reports holding the ACK ([Base RTT](./congestion-control.md#base-rtt)) |

The record also has the block `ivl_calc_min_bytes_in_flight`, `calc_mean_bytes_in_flight`, `ivl_calc_mean_bytes_in_flight`, and `ivl_calc_max_bytes_in_flight`.

A mean is taken over every sample from every congestion window on the NIC, not per PDC. An interval with no samples writes 0. On this NIC:

- With `PassThrough` set to `true`, the congestion window columns keep moving although the window holds back no packet ([Turning congestion control off](./congestion-control.md#turning-congestion-control-off)).
- With `NSCCEnable` set to `false`, the average queueing delay columns and `num_quick_adapt` stay 0; the RTT columns are still written.
- At the default PCIe settings the receiver-penalty columns stay 0: the host interface drains received data faster than the link delivers it.

`GET /api/v1/simulations/{sim_id}/results/data` serves this record as a time series ([Getting the records](#getting-the-records)).

## nd-stats

With the defaults, `nd-stats` has a row for the NIC's network port every `NDStatsReportInterval`, with `tier` `3`. It counts every frame on the port, PFC frames included, and its byte counts include the Ethernet header. `rx_drops` counts the received packets the receive buffer could not hold ([Receive buffer](./buffers-and-pfc.md#receive-buffer)). [Network Device Statistics](../../simulation-output-files.md#network-device-statistics-nd-stats) lists the columns.

## ecn-pfc-stats

With the defaults, `ecn-pfc-stats` has a row for each traffic class with a receive pool, classes 0 and 1 at the defaults, every `NDStatsReportInterval`; `priority` is the class. A row is written only once one of its counters is non-zero, and the counters count from the start of the simulation, so a class's rows start with the first PFC frame the NIC sends or receives for that class and continue every interval after. With the default `PFCEnableVector` the NIC sends no PFC frames, so its rows appear only after the rack switch pauses it.

`pfcxoff_tx` and `pfcxon_tx` count the pause and XON frames the NIC sends for the class; `pfcxoff_rx` and `pfcxon_rx` the frames it receives from the switch ([PFC](./buffers-and-pfc.md#pfc)). [ECN/PFC Statistics](../../simulation-output-files.md#ecnpfc-statistics-ecn-pfc-stats) lists every column.

## Records turned on by attributes

### net-buff-stats

With `EnableBufferStats` set to `true` on the NIC and the global `EnableNetworkBufferStatsReporting` set to `true`, `net-buff-stats` has one row for each of the NIC's buffer pools every `NetworkBufferStatsReportingInterval`. With only one of the two set, the NIC writes no rows. `buffer_id` names the pool: `RX_TC<n>_BUFFER` for the receive pool of class n and `TX_TC<n>_BUFFER` for its transmit pool, one for each class with a pool. At the defaults there are four ([Buffer pools](./buffers-and-pfc.md#buffer-pools)):

| `buffer_id` | `buffer_max_size_B` at the defaults |
| --- | --- |
| `RX_TC0_BUFFER` | 0.9 × 4,000,000 = 3,600,000 |
| `RX_TC1_BUFFER` | 0.1 × 4,000,000 = 400,000 |
| `TX_TC0_BUFFER` | 0.9 × 256,000 = 230,400 |
| `TX_TC1_BUFFER` | 0.1 × 256,000 = 25,600 |

The columns are `job_id`, `time_utc`, `sim_time_sec`, `component_name`, `node_id`, `buffer_id`, `tc`, `occupancy_B`, `max_occupancy_B`, `buffer_max_size_B`, `num_enq`, `num_deq`, `interval_num_enq`, `interval_num_deq`, `interval_enq_B`, `interval_deq_B`, `interval_max_occupancy_B`, `enq_Gbps`, `deq_Gbps`, `overflow`, and `percent_occupancy`. `node_id` is the NIC's FEP id, `buffer_max_size_B` the pool's size, and the occupancy columns mean:

- **A receive pool's occupancy** is the received packets it holds, counted at their IP size, 4,184 bytes for a full data packet: a data packet from its arrival until the PCIe link to the host accepts its write, and an ACK, NACK, probe, or trimmed packet until the NIC has processed it ([Receive buffer](./buffers-and-pfc.md#receive-buffer)).
- **A transmit pool's occupancy** is the space reserved for packets waiting to be sent. A full data packet reserves its 4,202 bytes on the wire from the start of its payload fetch, or with `PrefetchBufferEnable` `true` from its commit to the wire, until it has been serialized ([Egress buffer](./buffers-and-pfc.md#egress-buffer)).

PFC frames are never charged to a pool.

### uet-events

With `EnableEventLogging` set to `true` on the NIC, `uet-events` has one row for each PDC or work-request event on that NIC, written as the event happens. Each simulator process writes one file for all its NICs.

| Column | Holds |
| --- | --- |
| `time_utc` | Wall-clock time, updated once per second |
| `sim_time_ns` | Simulation time of the event, in ns |
| `component_name`, `FEP_ID` | The NIC |
| `local_PDC_ID`, `remote_PDC_ID` | The PDC's end on this NIC and on the other NIC ([Identifying a NIC and a PDC](#identifying-a-nic-and-a-pdc)) |
| `wqe_ID`, `wqe_size_bytes` | The work request and its size in bytes, on work-request events; empty on the others |
| `ev_type` | The event, one of the six below |

| `ev_type` | When |
| --- | --- |
| `PDC_CREATED` | A PDC is created: on the sending NIC when the host posts a message to a peer on a new PDC, on the receiving NIC when the first data packet of a new connection arrives |
| `WQE_CREATED` | The host posts a work request on the PDC |
| `SEND_COMPLETE` | The NIC has issued the fetch of the work request's last segment; its data may not yet be sent or acknowledged |
| `WQE_COMPLETE` | Every byte of the work request has been acknowledged |
| `RECV_COMPLETE` | On the receiving NIC, the message's last packet has arrived and no gap remains. This row has no work request |
| `PDC_TERMINATED` | The host closes the PDC |

In a Chakra workload simulation each message has its own PDC, so `PDC_CREATED` and `PDC_TERMINATED` come once per message on each NIC.

### uet-monitor

`uet-monitor` logs individual UET packets: data packets, ACKs, NACKs, and base-RTT probes. PFC frames are not logged. Each simulator process writes one file, opened when `EnableUETMonitor` or either trigger is on; with only a trigger on, it holds only the connections the triggers select.

The monitor's settings take effect once per simulator process, from one UET NIC: give every UET NIC in a configuration the same monitor settings ([ScalaUETMonitor](./configuration.md#scalauetmonitor)).

**Filters.** With `EnableUETMonitor` set to `true`, the filters select the PDCs whose packets are logged. `All` matches everything; any other value must match exactly, and a packet is selected when it matches every filter that is not `All`. Its connection, both of its PDC ends, is then selected for the rest of the run, as with a trigger, so the monitor also logs the connection's packets at the other NIC when that NIC runs in the same simulator process:

- `FilterOnFepId`: the FEP id of the NIC the PDC end is on, as a decimal number.
- `FilterOnLocalIp` and `FilterOnRemoteIp`: the address in the local or the remote PDC id, a dotted IPv4 address such as `32.0.0.2`.
- `FilterOnPdc`: a whole PDC id, `<IPv4>:<PDC id>`, for example `32.0.0.2:4500`, matched against both the local and the remote id. The number after the colon is a PDC id, not a port, and PDC ids are reused ([Identifying a NIC and a PDC](#identifying-a-nic-and-a-pdc)).

**Triggers.** A trigger selects a connection, both of its PDC ends, from the packet that meets it, and the connection stays selected for the rest of the run:

- `TriggerLoggingOnRttEnable` fires on a received ACK whose RTT sample is at or above `TriggerLoggingOnRttThreshold`. Only rows for received ACKs carry an RTT.
- `TriggerLoggingOnCWindPen` fires on a receiver penalty at or above `TriggerLoggingOnCWindPenThreshold`: the penalty an ACK carries, or, for a data packet the sender sends, the latest penalty it has received. At the default PCIe settings the penalty stays 0, so with a threshold above 0 this trigger does not fire in a default run.

The two triggers share one budget for each simulator process, `TriggeredLoggingPDCLimit` connections. Once it is spent, the triggers select no more connections, and packets of other connections are logged only if the filters select them, with `EnableUETMonitor` `true`.

**When rows are written.** A data packet, ACK, or probe the NIC sends is logged when it has finished going out on the wire, so a packet that never finishes sending is never logged. A NACK the NIC sends, and every packet it receives, is logged when the NIC processes it.

**Columns.** A row has 27 columns. A column that does not apply to a row's packet is empty, except `job_ID`, which reads `0`, and `ACK_bitmap_PSNs`, which reads `[]`.

| Columns | What they hold |
| --- | --- |
| `sim_time_ns` | Simulation time of the row, in ns |
| `component_name`, `FEP_ID` | The NIC that logged the packet |
| `job_ID` | The work request the packet belongs to; `0` where none applies |
| `packet_UID` | The packet's id in the simulation |
| `packet_size` | A size in bytes |
| `local_PDC_ID`, `remote_PDC_ID` | The PDC's end on this NIC and on the other NIC |
| `entropy_value` | For a data packet, its entropy value, the UDP source port ([Multipath](./multipath.md)); for an ACK, the entropy value it carries back to the sender; for a probe or a NACK, a PDC id |
| `RTT_ns` | For a received ACK that gave an RTT sample, that RTT in ns |
| `direction` | `Tx` for a packet sent, `ReTx` for a retransmitted data packet, `Rx` for a packet received |
| `packet_type` | `RUD_REQUEST` (data), `ACK_CC` (ACK), `NACK`, or `CTRL` (base-RTT probe) |
| `nack_code` | For a NACK, `UET_TRIMMED`, or `UET_TRIMMED_LASTHOP` for a packet trimmed on the last hop |
| `AR_bit` | Whether the data packet requests an ACK ([Acknowledgements](./transport.md#acknowledgements)) |
| `ECN_marked` | Whether the received packet arrived ECN-marked |
| `M_Flag` | The ECN echo the ACK or NACK carries |
| `NIC_prefetched_bytes`, `PDC_prefetched_bytes` | Occupancy of the prefetch buffer, for the whole NIC and for this PDC ([Prefetch buffer](./transport.md#prefetch-buffer)) |
| `bytes_fetched`, `bytes_in_flight`, `congestion_window` | On the sender, the PDC group's bytes fetched from the host, its bytes in flight, and its congestion window, in bytes |
| `receiver_congestion_window_penalty` | The receiver penalty, 0 to 127, that the ACK carries |
| `DSCP_type` | The packet's marking: `NO_TRIM`, `TRIMMABLE`, `TRIMMABLE_RTX`, `TRIMMED`, `TRIMMED_LAST_HOP`, `CONTROL`, or `UNRECOGNIZED` ([Packet marking](./transport.md#packet-marking)) |
| `PSN` | The data packet's PSN; for an ACK, the PSN it acknowledges; for a NACK, the PSN it reports; for a probe, the probe's tag |
| `CACK_PSN`, `SACK_base_PSN`, `ACK_bitmap_PSNs` | For an ACK, its cumulative PSN, the base of its selective-acknowledgement window, and the PSNs its bitmap acknowledges, written `[p1:p2:...]` ([Selective acknowledgements](./transport.md#selective-acknowledgements)) |

Which columns each kind of row fills, besides `sim_time_ns`, `component_name`, `FEP_ID`, `packet_UID`, `packet_size`, `local_PDC_ID`, `remote_PDC_ID`, `direction`, `packet_type`, and `DSCP_type`, which every row fills:

| Row | Columns filled |
| --- | --- |
| Data packet sent or retransmitted | `job_ID`, `entropy_value`, `AR_bit`, `PSN`, the two prefetch columns, `bytes_fetched`, `bytes_in_flight`, `congestion_window` |
| Data packet received, trimmed ones included | `entropy_value`, `AR_bit`, `ECN_marked`, `PSN` |
| ACK sent, by the receiver | `entropy_value`, `receiver_congestion_window_penalty`, `M_Flag`, `PSN`, `CACK_PSN`, `SACK_base_PSN`, `ACK_bitmap_PSNs` |
| ACK received, by the sender | The columns of an ACK sent, plus `ECN_marked`, `job_ID`, the two prefetch columns, `bytes_fetched`, `bytes_in_flight`, `congestion_window`, and `RTT_ns` when the ACK gave an RTT sample |
| Probe sent | `PSN`, `job_ID`, the two prefetch columns, `bytes_fetched`, `bytes_in_flight`, `congestion_window`; `entropy_value` holds the local PDC id |
| Probe received | `PSN`, `ECN_marked`; `entropy_value` holds the remote PDC id |
| NACK sent or received | `nack_code`, `PSN`, `M_Flag`; `entropy_value` holds the local PDC id on a NACK sent and the remote PDC id on a NACK received |

### pcie-stats

With the global `EnablePCIeStatsLogging` (default `false`) set to `true`, `pcie-stats` has one row for each end of the NIC's PCIe link, the NIC side and the host side, every `NDStatsReportInterval`. This page does not describe its columns.
