---
title: "Records"
description: "This page lists the records a Chakra workload simulation writes: what one row holds, when it is written, and which attribute turns it on."
---

This page lists the records a Chakra workload simulation writes: what one row holds, when it is written, and which attribute turns it on.

The records follow the run described in [Workload execution](./workload-execution.md): where each rank ran, how long each transfer took, how far each rank got, and how long each collective took against an ideal time for its algorithm. [Configuration](./configuration.md) lists the attributes that turn the optional records on.

## Records a simulation writes

With the defaults, a Chakra workload simulation writes these records. Each file is named after the record, in the form `{simulation-name}-{record}-{shard-index}.csv`, as [Simulation output files](../../simulation-output-files.md) describes. When a simulation runs as several simulator processes, each process writes its own shard, covering the ranks that process ran. Three records are exceptions: every shard of `chakra-capable-hosts` lists every server with a NIC; `chakra-mapping` is written in shard 0 only, with a row for every rank; and the percentiles record is one file for the whole run.

| Record | One row for | What a row holds | When |
| --- | --- | --- | --- |
| `chakra-capable-hosts` | Each server with a NIC | The server ID and whether a rank can be placed on it | At the start |
| `chakra-mapping` | Each rank | Where the rank runs: server, NIC, and simulator process | At the start, in shard 0 only |
| `chakra-apps` | Each rank the shard runs | The application name and the simulator process | At the start |
| `chakra-perf` | Each transfer, once acknowledged | The transfer's time from start to acknowledgment, its size, its collective, and its sender and receiver | As each transfer is acknowledged |
| `chakra-completion-stats` | Each rank the shard runs | The rank's completion time and its counts of bytes, transfers, and computations | When the rank ends |
| `chakra-cdf` | Each report interval | The fraction of the run's bytes, transfers, and computations completed so far | Every `BeaconReportInterval` |
| `chakra-comm-collective-stats` | Each collective the shard's ranks took part in | The collective's type, size, process group, and start and end time | At the end of the run |
| `chakra-comm-collective-percentiles-stats` | Each collective type, size, and group size | Run-time percentiles and their ratios to an ideal time | At the end of the run, one file |

The platform also delivers each rank's trace with the time of every node written into it ([Annotated traces](#annotated-traces)). `ProfilingEnabled` and `ScaleUpStatsEnabled` turn on two more records ([Records turned on by attributes](#records-turned-on-by-attributes)).

`BeaconReportInterval` is a global simulation parameter, not an attribute of this model: a `double` in seconds (default `0.001`) in the configuration's `SimulationParameters`, changed with `PATCH /api/v1/configurations/{config_id}/parameters`.

### Placement records

**`chakra-capable-hosts`** has one row for each server with a NIC, with two columns: `Host`, the server ID, and a second column that is `true` when the server has a Scala RoCE NIC, so that a rank can run on it. In a configuration that keeps to [Settings to keep](./configuration.md#settings-to-keep), it is `true` for every server.

**`chakra-mapping`** has one row for each rank, with the columns `ChakraId,Hostname,NicNodeID,ServerNodeID,ServerID,Core,IPAddress`: the rank, the name of its server, the simulator's node IDs of its NIC and server, the server ID, the simulator process that runs it, and the server's IP address. When a mapping file places ranks in scale-up groups, two more columns follow, `ScaleUpGroupId` and `ScaleUpSubgroupId`.

**`chakra-apps`** has the columns `AppName` and `Core`: each application, named `ChakraApp-<rank>`, and the simulator process that runs it.

### Transfers

**`chakra-perf`** has one row for each point-to-point transfer, written when the receiver's acknowledgment reaches the sender. Receives and compute get no row. Column names call a transfer a flow; `ChakraFCT` is the flow completion time. The columns are:

| Column | Holds |
| --- | --- |
| `time_utc` | Wall-clock time the row was written |
| `metric_name` | `ChakraFCT` |
| `sim_time_sec` | Simulation time of the acknowledgment, in seconds |
| `perf_metric_usec` | Time from the send node's start to the acknowledgment, in µs |
| `metric_path` | `ChakraFCT.<collective type>.GroupSize<N>.CommSize<S>Bytes` for a transfer that is part of a collective, with the collective's process group size and its size in the trace; `ChakraFCT.Send.GroupSize0.CommSize<bytes>Bytes` for a point-to-point send in the trace |
| `metric_aggregation_id` | An ID shared by every transfer of one collective; `-1` for a point-to-point send |
| `start_time_sec` | Simulation time the send node started, in seconds |
| `is_added_synchronization_flow` | `true` for a transfer of a collective synchronization barrier ([Ordering options](./workload-execution.md#ordering-options)) |
| `comm_tag` | The transfer's communication tag |
| `group_size` | The collective's process group size; `-1` for a point-to-point send |
| `flow_comm_size` | The transfer's size in bytes |
| `sending_app_name`, `receiving_app_name` | `ChakraApp-<rank>` of the sender and the receiver |

For a transfer over the scale-up fabric, the acknowledgment is one round trip after the last packet is sent ([Scale-up groups](./workload-execution.md#scale-up-groups)).

### Rank completion

**`chakra-completion-stats`** has one row for each rank, written when the rank ends. If the simulation reaches its stop time first, the row is written at the stop, with what the rank completed by then.

| Column | Holds |
| --- | --- |
| `metric_path` | `ChakraGenerator.ChakraApp` |
| `chakra_app_logical_id` | The rank |
| `chakra_trace_completion_time_us` | Time from the rank's start to its end, or to the stop, in µs |
| `chakra_full_payload_bytes_sent` | Bytes of the rank's completed sends |
| `chakra_full_payload_bytes_received` | Bytes of the rank's completed receives |
| `num_chakra_sends_completed` | Send nodes whose bytes have all been sent |
| `num_chakra_receives_completed` | Receive nodes that finished |
| `num_chakra_sends_fully_acked` | Sends whose transfer was acknowledged |
| `num_chakra_computations_completed` | `COMP_NODE`s that finished |
| `total_chakra_computation_time_us` | The sum of the finished `COMP_NODE`s' trace durations (`durationMicros`, after scaling), in µs. Sharing of compute capacity lengthens the time a node takes in the simulation, not this sum. |
| `post_annotated_results_path` | The rank's annotated trace, as `post_annotated/<prefix>.<rank>.json`; the platform delivers the file under `post-annotated/` ([Annotated traces](#annotated-traces)) |

The transfers and nodes of a collective synchronization barrier are not counted.

**`chakra-cdf`** shows the run's progress over simulation time. Every `BeaconReportInterval` it writes `sim_time_sec` and five fractions, each between 0 and 1: `CDF_send_bytes` (bytes sent), `CDF_chakra_sends_completed` (send nodes completed), `CDF_chakra_receives_completed` (receive nodes completed), `CDF_chakra_sends_fully_acked` (sends acknowledged), and `CDF_chakra_computations_completed` (`COMP_NODE`s completed). Each fraction divides what the shard's ranks have completed by the total for the whole run, so a shard reaches 1 only if it runs every rank; the fractions of all the shards at one time add up to the run's progress.

### Collectives

**`chakra-comm-collective-stats`** has one row for each collective the shard's ranks took part in, with the columns `collective_hash,comm_type,comm_size,pg_name,pg_size,simulation_start_time_us,simulation_end_time_us,use_max_as_collective_start_time,is_valid`. `collective_hash` identifies the collective, and `comm_type`, `comm_size`, `pg_name`, and `pg_size` are its type, its size in the trace, and its process group's name and size. `collective_hash` equals the `metric_aggregation_id` of the collective's transfers in `chakra-perf`. A shard's row covers only that shard's participants. The start time is taken from each participating rank's first transfer of the collective, and `UseMaxAsCommCollectiveStartTime` chooses whether the row reports the latest of those starts (the default) or the earliest. The end time is the latest acknowledgment or receive of the collective's transfers. `use_max_as_collective_start_time` repeats that setting as `1` or `0`. A row whose `is_valid` is `0` is a collective that none of the shard's ranks finished, for example because the run reached its stop time first. The transfers of a synchronization barrier are not included.

**`chakra-comm-collective-percentiles-stats`** combines the rows of every shard, written once, as `{simulation-name}-chakra-comm-collective-percentiles-stats-0.csv`. It has one row for each combination of collective type, size, and process group size, with the columns `comm_type,expansion_algo,comm_size,pg_size,count,mean_run_time_us,p50_run_time_us,p99_run_time_us,p99_9_run_time_us,p100_run_time_us,mean_ideal_ratio,p50_ideal_ratio,p99_ideal_ratio,p99_9_ideal_ratio,p100_ideal_ratio`. A collective's run time is its end time minus its start time, across all its participants. A collective with an `is_valid` of `0` in any shard is left out. Each ratio divides the run-time statistic by an ideal time: an analytical time for the selected algorithm (`expansion_algo`) to move the collective's bytes at the data rate of the NIC's link. For example, the ideal time of an AllReduce with `ring` is 2 × S × 8 × (N − 1) / (N × B) seconds, where S is the collective's size in bytes, 8 converts bytes to bits, N is its group size, and B is the link's rate in bit/s. A Barrier moves no data, so its ratios are `NA`.

### Annotated traces

Each rank's trace file is rewritten with three times for every node it started, and saved when the rank ends or the simulation stops:

- `s`: the simulation time the node started, in µs.
- `e`: the simulation time the node finished, in µs, except for a send node. A send node's `e` is the time its transfer was acknowledged, which is later than the time the node finished and released its dependents, when its last byte was sent ([Communication](./workload-execution.md#communication)). A send node's `e` can therefore be later than the `s` of the nodes that depend on it.
- `r`: the wall-clock time at which the simulator started the node, in µs since it created the rank's application.

A node that never started keeps the placeholder `XXXXXXXXXXXXX` in `s`, `e`, and `r`; a time that was written is padded with leading zeros to the same width. If the simulation stops before a rank ends, the nodes that were running have their start times written. The platform delivers the rewritten traces with the simulation's results, under `post-annotated/`, as `<prefix>.<rank>.json`, where `<prefix>` is the prefix the traceset's per-rank trace files share.

## Records turned on by attributes

### Profiling

With `ProfilingEnabled` set to `true`, each rank writes a text file, `{simulation-name}-chakra-profiling-<rank>-{shard-index}.txt`, that starts with the line `CHAKRA PROFILING`. Every `ProfilingInterval` seconds of simulation time, starting one interval after the rank starts, it appends a block that lists the rank's running nodes:

```text
================================
sim_time_sec: <time>
send_nodes:
    S(<node id>,<partner rank>,<bytes>,[<data dependencies>],[<control dependencies>])
receive_nodes:
    R(<node id>,<partner rank>,<bytes>,[<data dependencies>],[<control dependencies>])
computation_nodes:
    C(<node id>,[<data dependencies>],[<control dependencies>])
```

A node is listed from its start until it finishes; a send node stays listed until its transfer is acknowledged. `computation_nodes` lists compute, memory, and other nodes that take time without communicating. The dependencies are the node's lists from the trace, not the ones still outstanding. The file stops growing when the rank ends.

### Scale-up records

With `ScaleUpStatsEnabled` set to `true`, every rank that a mapping file places in a scale-up group gets a scale-up device in the `nd-stats` record, written every `NDStatsReportInterval` like the network's devices. Its rows have `nd_name` `ChakraScaleUpNetDevice`, `component_type` `ChakraScaleUpNic`, and the rank's server as `component_name`. They count the scale-up packets the rank sent and received, and their bytes including headers, with send and receive rates and utilization against the group's scale-up bandwidth. [Simulation output files](../../simulation-output-files.md) lists the `nd-stats` columns. Without this attribute, scale-up traffic has no `nd-stats` rows. It is still counted, like any other transfer, in `chakra-perf`, `chakra-completion-stats`, and `chakra-cdf`.
