---
title: "Records"
description: "This page lists the records a simulation writes for the Scala MRC NIC, when each is written, what turns it on, and what one row holds."
---

This page lists the records a simulation writes for the Scala MRC NIC, when each is written, what turns it on, and what one row holds.

The records show the behavior the other pages describe: [Transport](./transport.md) for packets, acknowledgements, retransmissions, and completions; [Multipath](./multipath.md) for the entropy values; [Congestion control](./congestion-control.md) for the congestion window and round-trip times; and [Buffers and PCIe](./buffers-and-pcie.md) for the buffers. [Configuration](./configuration.md) lists the attributes that turn the records on and off.

## Records a simulation writes

With the defaults, a simulation writes these records for every Scala MRC NIC. Each file is named after the record, in the form `{simulation-name}-{record}-{shard-index}.csv`, as [Simulation output files](../../simulation-output-files.md#file-naming-convention) describes. When a simulation runs as several simulator processes, each process writes its own shard.

| Record | One row for | Written | Turned on by |
| --- | --- | --- | --- |
| `mrc-transport-stats` | Each Scala MRC NIC, combining all its queue pairs | Every `TransportStatsReportInterval` | `EnableTransportStats` on the NIC (default `true`) |
| `nd-stats` | The NIC's network port | Every `NDStatsReportInterval` | The global `EnableNetDeviceStatLogging` (default `true`) |

These records are written only when turned on ([Records turned on by attributes](#records-turned-on-by-attributes)):

| Record | One row for | Written | Turned on by |
| --- | --- | --- | --- |
| `mrc-events` | Each queue pair or work-request event | As each event happens | `EnableEventLogging` on the NIC (default `false`) |
| `mrc-monitor` | Each packet that a selected queue pair sends or receives | As each packet is sent or processed | `EnableMonitor` on the NIC (default `false`) |

`mrc-transport-stats` writes its first row one `TransportStatsReportInterval` into the simulation, 1 ms by default, and a row every interval after that. `nd-stats` writes its first row at `StartTime` + `ClientStartOffset` + `NDStatsReportInterval` of simulation time, 0.001 + 0.001 + 0.001 = 0.003 s with the platform's defaults, and a row every interval after that.

`TransportStatsReportInterval` is an attribute of the NIC. `NDStatsReportInterval`, `StartTime`, `ClientStartOffset`, and `EnableNetDeviceStatLogging` are global simulation parameters, not attributes of this model. They sit in the configuration's `SimulationParameters` and are changed with `PATCH /api/v1/configurations/{config_id}/parameters`; `NDStatsReportInterval` is a `double` in seconds (default `0.001`).

## Getting the records

`GET /api/v1/simulations/{sim_id}/results` lists a simulation's result files, among them every record on this page; its `type` query parameter filters by file type, for example `csv`. `GET /api/v1/simulations/{sim_id}/results/download-url`, with the `fileName` query parameter set to a file's name, returns a link to download that file. The [Simulations API reference](../../api-reference/simulations.md) lists both endpoints' parameters.

Set `SIM_ID` to the simulation's ID and `FILE_NAME` to a name from the list:

```bash
curl -X GET "https://api.scalacomputing.com/api/v1/simulations/$SIM_ID/results?type=csv" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"

curl -X GET "https://api.scalacomputing.com/api/v1/simulations/$SIM_ID/results/download-url?fileName=$FILE_NAME" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

## Identifying a NIC and a queue pair

**A NIC** is identified by its node id in the simulation, the `node_id` column of every record on this page, with its component name in `component_name`. `FilterNodeId` takes the same number. In a Chakra workload simulation, the `NicNodeID` column of `chakra-mapping` gives the node id of each rank's NIC ([Placement records](../chakra-workload/records.md#placement-records)).

**A queue pair** (QP) connects two NICs, with an end on each: the sending end, which the host opens for a message, and the receiving end, which the receiving NIC opens when the first data packet of the message arrives ([Queue pairs and messages](./transport.md#queue-pairs-and-messages)). Each end has its own QP id and is named by its NIC's IPv4 address and that id. `mrc-events` and `mrc-monitor` show the end on the NIC that wrote the row as `local_addr` and `local_qp_id`, and the other end as `remote_addr` and `remote_qp_id`. The monitor's endpoint filters write an end as `<IPv4 address>:<QP id>`.

- The receiving NIC assigns its QP ids from 32,769 to 65,535, and takes an id back when its QP is closed, so an id names a QP only while that QP is open.
- The Chakra workload assigns the sending end's ids from 1 to 32,767, and opens a new QP for each message.
- On the sending end, `remote_qp_id` reads `0` until the sender learns the receiving end's id from its first SACK or NACK.

## mrc-transport-stats

`mrc-transport-stats` has one row for each Scala MRC NIC whose `EnableTransportStats` is `true`, every `TransportStatsReportInterval`. A row combines every queue pair and every congestion window on the NIC: the record has no row for a single queue pair or destination. Each simulator process writes one file for all its NICs. A row has 112 columns. Columns that start with `ivl_` cover the interval since the previous row; the matching columns without the prefix cover the simulation so far, so the first row's `ivl_` columns cover everything before it. The part of the run after the last full interval has no row, and a run shorter than one interval writes no file.

The counters:

| Columns | What they hold |
| --- | --- |
| `time_utc`, `sim_time_ns` | When the row was written: wall-clock time, updated once per second, and simulation time in ns |
| `component_name`, `node_id` | The NIC ([Identifying a NIC and a queue pair](#identifying-a-nic-and-a-queue-pair)) |
| `total_pkts_tx`, `total_pkts_rx` | All packets the NIC sent and received |
| `total_write_pkts_tx`, `total_write_pkts_rx` | Data packets sent, retransmissions and completion re-requests included, and data packets received, trimmed packets and duplicates included |
| `total_ack_pkts_tx`, `total_ack_pkts_rx` | Cumulative ACKs sent and received ([Cumulative acknowledgements](./transport.md#cumulative-acknowledgements)) |
| `total_sack_pkts_tx`, `total_sack_pkts_rx` | SACKs sent and received ([Selective acknowledgements](./transport.md#selective-acknowledgements)) |
| `total_nack_pkts_tx`, `total_nack_pkts_rx` | NACKs sent and received ([NACKs](./transport.md#nacks)) |
| `total_probe_pkts_tx`, `total_probe_pkts_rx` | Reliability probes sent and received ([Path feedback and probes](./multipath.md#path-feedback-and-probes)) |
| `total_bytes_tx`, `total_bytes_rx` | Bytes of the packets sent and received, at their IP size, without the Ethernet header. A received packet shorter than 46 bytes counts as 46 |
| `total_rtx_bytes` | Bytes of the data packets retransmitted, at their nominal size |
| `total_wqes_created`, `total_wqes_completed` | Work requests (messages) the host posted to the NIC, and work requests a cumulative ACK has completed ([Completion](./transport.md#completion)) |
| `total_trim_pkts_rx`, `total_trim_lasthop_pkts_rx` | Trimmed data packets received from senders with the default `RTTTimestampMode`, trimmed in the fabric or on the last hop ([NACKs](./transport.md#nacks)) |
| `total_nack_trimmed_rx`, `total_nack_trimmed_lasthop_rx`, `total_nack_psnoor_rx`, `total_nack_unexp_rx` | NACKs received, by code: trimmed, trimmed on the last hop, PSN out of range, and unexpected event |
| `total_rtx_pkts`, `total_rtx_rto`, `total_rtx_sack`, `total_rtx_nack` | Data packets retransmitted, in all and by what queued them: a retransmission timeout, a SACK gap, or a NACK. Completion re-requests are not counted ([Retransmission timer and retry budget](./transport.md#retransmission-timer-and-retry-budget)) |
| `total_rto_events` | Retransmission timer expiries that queued a packet for retransmission |
| `total_ooo_events`, `total_dup_events` | Data packets received ahead of the next expected PSN, and data packets received again ([Receiving a packet](./transport.md#receiving-a-packet)) |
| `total_qp_error` | Queue pairs that entered the error state |
| `total_qa_count` | Times Quick Adapt cut a congestion window ([Quick Adapt](./congestion-control.md#quick-adapt)) |
| `total_fi_count` | Window updates made in fast increase ([How the window responds to each SACK](./congestion-control.md#how-the-window-responds-to-each-sack)) |

The record also has the counters `total_bytes_acked`, `total_ecn_marked_tx`, `total_ecn_marked_rx`, and `total_md_count`.

Every counter has an `ivl_total_` twin, for example `ivl_total_rtx_pkts`. A packet the NIC sends is counted when it has finished going out on the wire. A packet it receives is counted once the receive buffer has admitted it and the NIC has found its queue pair; a packet the receive buffer drops is not counted here.

The samples come in blocks. In each column's name, `min`, `mean`, or `max` says whether the column holds the least, the mean, or the greatest sample, over the interval for an `ivl_` column and over the simulation so far for the others: `ivl_calc_max_rtt_ns` is the greatest RTT sampled in the interval.

| Columns | Sample |
| --- | --- |
| `calc_min_cwnd_bytes`, `calc_mean_cwnd_bytes`, `ivl_calc_min_cwnd_bytes`, `ivl_calc_mean_cwnd_bytes`, `ivl_calc_max_cwnd_bytes` | The congestion window, in bytes, at each SACK the NIC receives, before that SACK changes it ([The congestion window](./congestion-control.md#the-congestion-window)) |
| `calc_mean_bytes_in_flight`, `calc_max_bytes_in_flight`, `ivl_calc_min_bytes_in_flight`, `ivl_calc_mean_bytes_in_flight`, `ivl_calc_max_bytes_in_flight` | The bytes in flight under that window, at the same SACKs |
| `calc_min_rtt_ns`, `calc_mean_rtt_ns`, `calc_max_rtt_ns`, `ivl_calc_min_rtt_ns`, `ivl_calc_mean_rtt_ns`, `ivl_calc_max_rtt_ns` | The RTT of each SACK that gives an RTT sample, in ns ([RTT measurement](./transport.md#rtt-measurement)) |
| `calc_mean_avg_queue_delay_ns`, `calc_max_avg_queue_delay_ns`, `ivl_calc_min_avg_queue_delay_ns`, `ivl_calc_mean_avg_queue_delay_ns`, `ivl_calc_max_avg_queue_delay_ns` | The average queueing delay congestion control keeps, in ns, after each SACK that gives an RTT sample ([How the window responds to each SACK](./congestion-control.md#how-the-window-responds-to-each-sack)) |
| `base_rtt_ns` | The lowest base RTT any congestion window on the NIC has reached so far, in ns; `0` before the first sample ([Base RTT](./congestion-control.md#base-rtt)) |
| `calc_max_max_tx_count`, `ivl_calc_max_max_tx_count` | The most transmissions of one data packet, counted at each retransmission; `0` while nothing has been retransmitted |
| `calc_mean_reorder_depth`, `calc_max_reorder_depth`, `ivl_calc_min_reorder_depth`, `ivl_calc_mean_reorder_depth`, `ivl_calc_max_reorder_depth` | At each data packet the NIC receives, the number of PSNs its queue pair holds out of order ([Selective acknowledgements](./transport.md#selective-acknowledgements)) |

The record also has the block `calc_mean_rcvr_cwnd_pen`, `calc_max_rcvr_cwnd_pen`, `ivl_calc_min_rcvr_cwnd_pen`, `ivl_calc_mean_rcvr_cwnd_pen`, and `ivl_calc_max_rcvr_cwnd_pen`.

A mean is taken over every sample from every congestion window on the NIC, not per queue pair. An interval with no samples writes 0. With `PassThrough` set to `true`, the congestion window columns keep moving although the window holds back no packet ([Turning congestion control off](./congestion-control.md#turning-congestion-control-off)).

## nd-stats

With the defaults, `nd-stats` has a row for the NIC's network port every `NDStatsReportInterval`, with `tier` `3`. Its byte counts include the Ethernet header. `rx_drops` includes the received packets the NIC discards, among them those the receive buffer could not hold ([Receive buffer](./buffers-and-pcie.md#receive-buffer)), data packets for a queue pair the NIC does not have, and data packets beyond the receiving end's PSN window ([NACKs](./transport.md#nacks)). [Network Device Statistics](../../simulation-output-files.md#network-device-statistics-nd-stats) lists the columns.

## Records turned on by attributes

### mrc-events

With `EnableEventLogging` set to `true` on the NIC, `mrc-events` has one row for each queue pair or work-request event on that NIC, written as the event happens. Each simulator process writes one file for all its NICs, from its first event.

| Column | Holds |
| --- | --- |
| `time_utc` | Wall-clock time, updated once per second |
| `sim_time_ns` | Simulation time of the event, in ns |
| `component_name`, `node_id` | The NIC |
| `local_qp_id`, `local_addr`, `remote_qp_id`, `remote_addr` | The queue pair's end on this NIC and on the other NIC ([Identifying a NIC and a queue pair](#identifying-a-nic-and-a-queue-pair)) |
| `wqe_id`, `wqe_size_bytes` | The work request and its size in bytes, on work-request events; `0` on queue pair events and on `RECV_COMPLETE` |
| `event_type` | The event, one of the seven below |
| `event_detail` | On `QP_ERROR`, the condition that caused it; empty on the other events |

| `event_type` | When |
| --- | --- |
| `QP_CREATED` | A queue pair is created: on the sending NIC when the host posts a message on a new queue pair, on the receiving NIC when the first data packet of a new message arrives |
| `WQE_CREATED` | The host posts a work request on the queue pair |
| `SEND_COMPLETE` | The payload of the work request's last packet has come back from the host and been queued for the wire; its data may not yet be sent or acknowledged |
| `WQE_COMPLETE` | A cumulative ACK covers every packet of the work request |
| `RECV_COMPLETE` | On the receiving NIC, the message's last packet has arrived and no gap remains before it. This row has no work request |
| `QP_ERROR` | The queue pair enters the error state because a packet's retry budget ran out ([Retransmission timer and retry budget](./transport.md#retransmission-timer-and-retry-budget)) |
| `QP_DESTROYED` | The host closes the queue pair |

In a Chakra workload simulation each message has its own queue pair, so `QP_CREATED` and `QP_DESTROYED` come once per message on each NIC.

### mrc-monitor

`mrc-monitor` logs individual packets of the queue pairs it selects: data packets, cumulative ACKs, SACKs, NACKs, and reliability probes. Each simulator process writes one file, from its first row.

The monitor's settings take effect once per simulator process: give every Scala MRC NIC in a configuration the same `EnableMonitor` and monitor settings ([ScalaMrcMonitor](./configuration.md#scalamrcmonitor)).

**Filters.** With `EnableMonitor` set to `true`, the filters select queue pairs. Each filter is `All`, which matches every queue pair, or a comma-separated list, which a queue pair matches when it matches any entry. A queue pair is selected when it matches every filter that is not `All`; with every filter at `All`, the default, every queue pair is selected.

- `FilterNodeId`: the node id of the NIC the queue pair is on, in decimal.
- `FilterLocalAddr`, `FilterRemoteAddr`, and `FilterEitherAddr`: an IPv4 address in dotted form, matched against the queue pair's local address, its remote address, or either.
- `FilterLocalEndpoint`, `FilterRemoteEndpoint`, and `FilterEitherEndpoint`: a queue pair end, `<IPv4 address>:<QP id>`, matched against the queue pair's local end, its remote end, or either ([Identifying a NIC and a queue pair](#identifying-a-nic-and-a-queue-pair)). The remote end's QP id reads `0` until the NIC learns it.

**Triggers.** `MonitorTriggers` lists events that select a queue pair from the moment they happen: `trim` when the NIC receives a trimmed data packet (from a sender with the default `RTTTimestampMode`), `nack` when it receives a NACK, `qp_error` when a queue pair enters the error state, and `rto` when a retransmission timer expires. A trigger selects only the queue pair the event happened on, on that NIC: the packets before the event are not logged. The NACK that fires `nack` is logged; the trimmed packet that fires `trim` is not. Triggers work only with `EnableMonitor` `true`, and matter only when the filters leave queue pairs out. With `EnableMonitorTriggerQpCap` `true`, the triggers select at most `MonitorTriggerQpCap` queue pairs in each simulator process.

**Packets.** In a selected queue pair, a packet is logged when the toggle for its type and the toggle for its direction are both `true`:

- Type: `LogWrite` for data packets, `LogAck` for cumulative ACKs, `LogSack` for SACKs, `LogNack` for NACKs, and `LogProbe` for reliability probes. A received NACK also needs the toggle for its code: `LogNackTrimmed`, `LogNackTrimmedLastHop`, `LogNackPsnOor`, or `LogNackUnexp`.
- Direction: `LogRtx` for a retransmission or a completion re-request, `LogTx` for any other packet the NIC sends, and `LogRx` for a packet the NIC receives.

**When rows are written.** A packet the NIC sends is logged when it has finished going out on the wire, so a packet that never finishes sending is never logged. A packet it receives is logged when the NIC processes it. Trimmed data packets, data packets beyond the PSN window, probes, and probe responses that the NIC receives are not logged, and neither are packets the receive buffer drops. With `EnableMonitorRowCap` `true`, a simulator process writes at most `MonitorRowCap` rows, sent and received, over all its NICs, and the monitor then logs nothing more for the rest of the run.

**Columns.** A row has 32 columns. A column that does not apply to a row's packet reads `0` or is empty.

| Columns | What they hold |
| --- | --- |
| `sim_time_ns` | Simulation time of the row, in ns: for a packet sent, the end of its serialization |
| `component_name`, `node_id` | The NIC that logged the packet |
| `wqe_id` | On the sending end, the work request the queue pair carries; `0` on the receiving end |
| `direction` | `tx` for a packet sent, `rtx` for a retransmitted data packet or a completion re-request, `rx` for a packet received |
| `local_addr`, `local_qp_id`, `remote_addr`, `remote_qp_id` | The queue pair's end on this NIC and on the other NIC |
| `packet_uid` | The packet's id in the simulation |
| `packet_type` | `WRITE_FIRST`, `WRITE_MIDDLE`, `WRITE_LAST`, or `WRITE_ONLY` for a data packet ([Segmentation](./transport.md#segmentation)); `ACK` for a cumulative ACK; `SACK`; `NACK`; or `PROBE_REQUEST` for a reliability probe |
| `dscp` | The packet's marking: `NO_TRIM`, `TRIMMABLE`, `TRIMMABLE_RTX`, `TRIMMED`, `TRIMMED_LAST_HOP`, `CONTROL`, or `UNRECOGNIZED` ([Packet marking](./transport.md#packet-marking)) |
| `rtx_origin` | On a retransmission, what queued it: `rto`, `sack`, or `nack`; empty on a completion re-request |
| `tx_count` | On a data packet sent or retransmitted, its transmissions so far, this one included; `0` on a completion re-request |
| `ev` | The entropy value, the UDP source port ([Multipath](./multipath.md)). For a data packet sent, the value spraying chose; for a SACK or NACK sent, the value of the packet it answers; for a cumulative ACK sent, a value the NIC chooses as it does for its own data toward that peer; for a probe, the value of the path it probes; for a packet received, the value it arrived with |
| `ar_bit` | Whether the packet requests a cumulative ACK, as sent or as received ([Cumulative acknowledgements](./transport.md#cumulative-acknowledgements)) |
| `rtt_ns` | On a received SACK that gave an RTT sample, that RTT in ns |
| `ooo_count` | On a received SACK, the number of PSNs the receiving end holds out of order, at most 32,767 |
| `cc_tx_timestamp` | On a received SACK, the timestamp field it carries, in units of 128 ns ([RTT measurement](./transport.md#rtt-measurement)) |
| `service_time_ns` | On a received SACK, `cc_tx_timestamp` in ns |
| `nack_code` | On a received NACK, its code: `TRIMMED`, `TRIMMED_LASTHOP`, `PSN_OOR_WINDOW`, or `UNEXP_EVENT` ([NACKs](./transport.md#nacks)) |
| `bytes_in_flight`, `cwnd_bytes` | The bytes in flight and the congestion window, in bytes, of the window this NIC sends toward the queue pair's peer under: for a packet sent, when its headers were built; for a received SACK or cumulative ACK, after the NIC has processed it |
| `nominal_packet_size_bytes` | For a data packet, its size at the IP layer; `0` for a control packet |
| `wire_packet_size_bytes` | The packet's size on the wire: its IP size plus 18 bytes of Ethernet header and FCS. A received frame shorter than the 64-byte Ethernet minimum, such as a cumulative ACK, reads `64` |
| `bth_psn` | For a data packet, its PSN; for a received SACK or cumulative ACK, its cumulative PSN; for a received NACK, the PSN it reports |
| `sack_psn`, `sack_base_psn`, `sack_bitmap_psns` | On a received SACK, the PSN of the packet that drew it, the base of its bitmap, and the PSNs its bitmap lists, joined with `;` ([Selective acknowledgements](./transport.md#selective-acknowledgements)) |

The row also has the columns `ecn_marked`, `m_flag`, and `rcvr_cwnd_pen`.
