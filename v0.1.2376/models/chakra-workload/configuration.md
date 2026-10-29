---
title: "Configuration"
description: "This page lists the Chakra workload attributes a simulation configuration can set, with their types and defaults, and shows how to set the workload up."
---

This page lists the Chakra workload attributes a simulation configuration can set, with their types and defaults, and shows how to set the workload up.

The Chakra workload is an application in a simulation configuration. Its `typedParameters` hold `LogicalGroupId` at the top level and the `ChakraConfiguration` object, which holds every other attribute; the tables group those by what they control. The page ends with the rules that stop a simulation and the settings to keep. [Workload execution](./workload-execution.md) and [Records](./records.md) explain the behavior behind the attributes.

## How to read the tables

**Default** is the value an attribute carries in a Chakra application the platform adds to a configuration with `POST /api/v1/configurations/{config_id}/applications` ([Add the application](#add-the-application)). Such an application carries every attribute in these tables at its Default until you change it.

If you supply the application's `typedParameters` yourself instead, for example in the `applications` of `POST /api/v1/configurations`, include every attribute of its `ChakraConfiguration`. Set `NumOfChakraFiles` to the number of servers in the topology; the request returns 400 when they differ. `LogicalGroupId` takes its Default if you leave it out.

**Type** is the value of the attribute's `type` field in the API, which a patch must repeat. Values of type `bytes` carry a `unit`: `B`, `KB` (1,000 bytes), `KiB` (1,024 bytes), `MB` (1,000,000 bytes), or `MiB` (1,048,576 bytes). Values of type `double` that are times are in seconds of simulation time.

## Application

This attribute sits at the top level of the application's `typedParameters`.

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `LogicalGroupId` | `uint` | `0` | Logical group of the application. Keep it at `0`, the logical group of the servers the platform adds. Rank placement does not use it. |

## ChakraConfiguration

These attributes sit in the `ChakraConfiguration` object of the application's `typedParameters`.

### Ranks and placement

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `NumOfChakraFiles` | `uint` | `256` | Number of ranks: one Chakra application, on one server, for each. The platform sets it to the traceset's rank count whenever you attach a traceset ([Attach a traceset](#attach-a-traceset)); the components API marks it read-only. |
| `UseMapping` | `bool` | `true` | Whether ranks are placed by an attached mapping file. The platform sets it to `true` when you attach a mapping file and to `false` when you detach one, and keeps it fixed while a file is attached. Without a mapping file, ranks are placed in order whatever its value. See [Placement](./workload-execution.md#placement). |

### Collective algorithms

Each attribute selects the algorithm that expands one collective type into point-to-point transfers. [Collectives](./workload-execution.md#collectives) describes each algorithm, with a worked example.

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `AllReduceAlgorithm` | `string` | `ring` | Algorithm for AllReduce: `ring`, `tree`, `double-tree`, or `halving-doubling`. `halving-doubling` needs process groups whose size is a power of two. |
| `ReduceAlgorithm` | `string` | `ring` | Algorithm for Reduce: `ring` or `tree`. |
| `AllGatherAlgorithm` | `string` | `ring` | Algorithm for AllGather: `ring`. |
| `ReduceScatterAlgorithm` | `string` | `ring` | Algorithm for ReduceScatter: `ring` or `halving`. `halving` needs process groups whose size is a power of two. |
| `ReduceScatterBlockAlgorithm` | `string` | `ring` | Algorithm for ReduceScatterBlock: `ring`. |
| `BroadcastAlgorithm` | `string` | `interleaved` | Algorithm for Broadcast: `sequential`, where the root sends to one rank after another, or `interleaved`, where it sends to all at once. |
| `GatherAlgorithm` | `string` | `direct` | Algorithm for Gather: `direct`. |
| `ScatterAlgorithm` | `string` | `direct` | Algorithm for Scatter: `direct`. |
| `AllToAllAlgorithm` | `string` | `interleaved` | Algorithm for AllToAll: `sequential`, where each rank's sends go one after another, or `interleaved`, all at once. |
| `BarrierAlgorithm` | `string` | `ring` | Algorithm for Barrier: `ring`, `tree`, `double-tree`, or `halving-doubling`, with the pattern of AllReduce and no data. `halving-doubling` needs process groups whose size is a power of two. |

### Ordering and preparation

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `ProcessGroupSerialization` | `bool` | `true` | When `true`, and the traces record the order in which collectives were issued, each rank runs at most one collective of a process group at a time, in that order. See [Ordering options](./workload-execution.md#ordering-options). |
| `CommCollectiveSynchronization` | `bool` | `true` | When `true`, every rank of a process group passes a zero-byte barrier before each of the group's collectives, so the collective's transfers start together on every rank. When `false`, each rank starts its part of a collective as soon as its own dependencies have finished. |
| `UseMaxAsCommCollectiveStartTime` | `bool` | `true` | Which participant's start the collective records report as a collective's start time: the latest when `true`, the earliest when `false`. It changes `chakra-comm-collective-stats` and `chakra-comm-collective-percentiles-stats` only, not when any transfer starts. Keep it `true`. |
| `AbortOnChakraWarning` | `bool` | `false` | When `true`, the run stops before the simulation starts if trace preparation logs a warning or an error. Lines that `MaxCountWarningsLoggedPerInstance` holds back are not counted. |
| `MaxCountWarningsLoggedPerInstance` | `uint` | `30000` | Maximum number of log lines (information, warnings, and errors) that each trace-preparation step writes on each compute instance, from `0` to `40000`. Lines past the limit are dropped and processing continues. |

### Compute and scaling

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `TraceFamily` | `string` | `synthesized` | Where the traceset's compute durations come from, which decides whether compute nodes running at the same time on a rank share its capacity. `synthesized` (generated traces) and `compiled` (extracted while compiling the training code) give each node the time it needs with the NPU to itself, so concurrent nodes share the NPU equally. `collected` (captured on a real system) gives measured times, so each node runs for its recorded duration. See [Compute](./workload-execution.md#compute). |
| `Scaling` | `bool` | `false` | When `true`, every transfer size and every duration in the traces is multiplied by `ScalingFactor`, which shortens the run while keeping the trace's pattern of computation and communication at a coarser granularity. See [Scaling](./workload-execution.md#scaling). |
| `ScalingFactor` | `double` | `1.0` | The factor `Scaling` applies, from `0.0` to `1.0`. Results are truncated to whole bytes and microseconds; at `0.0` every transfer has zero bytes and every node takes no time. |
| `InMemoryTraceCompression` | `bool` | `true` | When `true`, a rank's trace larger than 25 MiB and at most 1,000 MiB is held in the simulator's memory in compressed form. A trace of 25 MiB or less is always held in memory uncompressed. A larger trace is read from disk when the option is `false`, or when it is larger than 1,000 MiB. |

### Profiling

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `ProfilingEnabled` | `bool` | `false` | When `true`, each rank writes a profiling file that lists its running nodes every `ProfilingInterval`. See [Profiling](./records.md#profiling). |
| `ProfilingInterval` | `double` | `0.001` | Spacing, in seconds of simulation time, between the profiling file's blocks. Set it greater than `0`. For example, `0.0001` writes a block every 100 µs. |

### Scale-up

These attributes apply to transfers between ranks that a mapping file places in the same scale-up group ([Scale-up groups](./workload-execution.md#scale-up-groups)). The two sizes are checked at the start of every run, whether or not the configuration has a mapping file or scale-up groups.

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `ScaleUpHeaderSize` | `bytes` | `24B` | Header bytes added to each scale-up packet. At most 64 MiB (67,108,864 bytes). |
| `ScaleUpMaxPayloadSize` | `bytes` | `4096B` | Maximum number of data bytes in one scale-up packet; a transfer is cut into packets of this size. From 1 byte to 64 MiB (67,108,864 bytes). |
| `ScaleUpStatsEnabled` | `bool` | `false` | When `true`, every rank in a scale-up group gets a scale-up device in `nd-stats`, with its scale-up packets, bytes, rates, and utilization. See [Scale-up records](./records.md#scale-up-records). |

## Setting up a Chakra simulation through the platform

A Chakra simulation needs a configuration with the Chakra application and a traceset attached to it, and it can have a mapping file. The reference pages for the APIs involved are [Components](../../api-reference/components.md), [Configurations](../../api-reference/configurations.md), and [Tracesets](../../api-reference/tracesets.md). Get an API token first ([Authentication](../../authentication.md)) and export it as `SCALA_API_TOKEN`.

### Read the workload's parameters

`GET /api/v1/components?category=application` lists the application models the platform offers; the `ScalaChakraGenerator` entry gives the Chakra workload's `id` and its `versions`. `GET /api/v1/components/{component_id}`, with the `version` query parameter set to the model version these pages describe, returns the workload's `typedParameters`. Each attribute in the tables above appears there as an object with its `value`, its `type`, a `unit` for `bytes` values, and, where the model sets them, its `default` (with `defaultUnit` for `bytes` values), `description`, `min` and `max` (each an object with a `value`), `readonly`, and allowed values in `enum`. `ChakraConfiguration` is a nested object.

```bash
curl -X GET "https://api.scalacomputing.com/api/v1/components?category=application" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"

# Set COMPONENT_ID to the id of the ScalaChakraGenerator entry in that response.
export COMPONENT_ID="model_..."

curl -X GET "https://api.scalacomputing.com/api/v1/components/$COMPONENT_ID?version=4.0.3" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

### Add the application

`POST /api/v1/configurations/{config_id}/applications` adds the Chakra workload to a configuration, with every attribute at its Default. The query parameters are `modelId`, the workload's component `id`; `name`, a name for the application that is unique in the configuration, 1 to 64 characters; and `version`. Set `CONFIG_ID` to the configuration's ID.

```bash
curl -X POST "https://api.scalacomputing.com/api/v1/configurations/$CONFIG_ID/applications?modelId=$COMPONENT_ID&name=chakra-workload&version=4.0.3" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

The Chakra workload is the only application in its simulation. Give the configuration's topology exactly as many servers as the traceset has ranks, all with a Scala RoCE NIC or all with a Scala UET NIC ([Placement](./workload-execution.md#placement)).

### Attach a traceset

`PATCH /api/v1/configurations/{config_id}/traceset` attaches a traceset to the configuration, and `GET` on the same path lists the tracesets you can attach. Set the configuration's traceset with this endpoint. The configuration must already have the Chakra application, and the traceset needs a rank count between 1 and 4,096: a traceset you upload reports its rank count as `rankCountDerived` on `GET /api/v1/tracesets/{id}` ([Chakra mapping files](../../chakra-mapping-files.md#current-limitations) explains where it comes from). The attach sets `NumOfChakraFiles` to the rank count and returns the configuration's `activeTraceset`.

```bash
curl -X PATCH "https://api.scalacomputing.com/api/v1/configurations/$CONFIG_ID/traceset" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "tracesetId": "ts_..." }'
```

To place ranks yourself or to declare scale-up groups, attach a mapping file after the traceset; [Chakra mapping files](../../chakra-mapping-files.md) describes the file, the upload, and the attach. While a mapping file is attached, the platform keeps `UseMapping` fixed.

### Change values in a configuration

`PATCH /api/v1/configurations/{config_id}/applications/{app_name}/parameters` merges a patch into the `typedParameters` of the application named `app_name`; `GET /api/v1/configurations/{config_id}/applications?withDetails=true` lists the configuration's applications with their parameters. Set `APP_NAME` to the application's `name`.

The patch repeats the nesting of the application's `typedParameters`, down to each attribute you change:

- Only keys the application already has can be updated. An unknown key returns 400.
- Each attribute's `type` must match the stored type, or the request returns 400. The request also returns 400 for an empty patch, or when a nested object is sent where an attribute is expected, or the reverse.
- The `value` and the `unit` you send replace the stored ones, so send the `unit` with every `bytes` value.
- A string `value` cannot contain shell metacharacters, among them `[` `]` `{` `}` `(` `)` `'` `"` `;` `$` `*` `?` `#` `~`, or the request returns 400.
- While a mapping file is attached, a patch that changes `UseMapping` returns 422.
- A 200 response returns the application's full, updated `typedParameters`.

This example selects `tree` for AllReduce and turns profiling on:

```bash
curl -X PATCH "https://api.scalacomputing.com/api/v1/configurations/$CONFIG_ID/applications/$APP_NAME/parameters" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "ChakraConfiguration": {
      "AllReduceAlgorithm": { "value": "tree", "type": "string" },
      "ProfilingEnabled": { "value": true, "type": "bool" }
    }
  }'
```

Global simulation parameters, such as `RunTime` and `BeaconReportInterval`, are not attributes of this model; change them with `PATCH /api/v1/configurations/{config_id}/parameters`. Set `RunTime` long enough for the traces to finish ([How a run ends](./workload-execution.md#how-a-run-ends)).

## Configuration rules that stop a simulation

A simulation can start only from a configuration whose `status` is `validated`; `POST /api/v1/simulations` returns 422 for any other. `POST /api/v1/configurations/{config_id}/validate` returns the configuration's validation errors and warnings. A configuration with the Chakra workload is `invalid` when:

- It has no traceset attached.
- It has another application besides the Chakra workload, or a second Chakra workload.
- A `ChakraConfiguration` value is outside the attribute's allowed values (`enum`) or its range (`min`, `max`), or a `bytes` value has no valid unit.
- `NumOfChakraFiles` is greater than the number of servers connected to a NIC.
- A server has a NIC other than the Scala RoCE NIC or the Scala UET NIC.
- `LogicalGroupId` is not the logical group of the servers.

The traceset attach returns 422, and attaches nothing, when the traceset has no rank count or its rank count is 0 or more than 4,096. `POST /api/v1/configurations` and `PATCH /api/v1/configurations/{config_id}` return 400 when a Chakra workload's `NumOfChakraFiles` differs from the number of servers in the topology.

The run stops before the simulation starts when:

- The traceset's rank count differs from the number of per-rank traces it holds.
- Trace preparation cannot process a trace: for example, a collective with `halving` or `halving-doubling` runs over a process group whose size is not a power of two.
- `AbortOnChakraWarning` is `true` and trace preparation logged a warning or an error.
- `ScaleUpMaxPayloadSize` is 0 or larger than 64 MiB, or `ScaleUpHeaderSize` is larger than 64 MiB.
- An attached mapping file breaks a rule that depends on the topology ([Chakra mapping files](../../chakra-mapping-files.md#when-each-rule-is-checked)).

## Settings to keep

These pages describe a Chakra simulation that follows these settings:

- Give the topology exactly as many servers as the traceset has ranks, all with a Scala RoCE NIC or all with a Scala UET NIC.
- Set the traceset with `PATCH /api/v1/configurations/{config_id}/traceset`, not as `activeTraceset` in a configuration create or update body.
- Keep `UseMaxAsCommCollectiveStartTime` at `true`.
- Set `ProfilingInterval` greater than `0`.
