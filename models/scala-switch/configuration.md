---
title: "Configuration"
description: "This page lists the Scala Switch attributes a simulation configuration can set, with their types and defaults, and shows how to change them."
---

This page lists the Scala Switch attributes a simulation configuration can set, with their types and defaults, and shows how to change them.

The tables list the attributes the components API offers for the Scala Switch, grouped by the component that holds them. The page ends with the configuration rules: what stops a simulation, what the switch does not check, and the PFC class-to-pool rule. [Packet handling](./packet-handling.md) and [Shared buffer manager](./shared-buffer.md) explain the behavior behind them.

## How to read the tables

**Default** is the value an attribute carries in a switch the platform adds to a configuration, for example with `POST /api/v1/configurations/{config_id}/components`. Such a switch carries every attribute in these tables at its Default until you change it.

If you supply a switch's `typedParameters` yourself instead, for example in the `components` of `POST /api/v1/configurations`, include every attribute of each sub-component you send, such as `SharedBufferManager` or `EcnHandler`: an attribute left out of it runs the model's built-in value, which can differ from the Default shown here. Attributes at the switch's top level, and sub-components you leave out entirely, take their Defaults.

**Type** is the value of the attribute's `type` field in the API, which a patch must repeat. Values of type `bytes`, `datarate`, and `timeval` carry a `unit`: `KB` is 1,000 bytes and `MB` is 1,000,000 bytes; data rates are in `Gbps`; times are in `ps`, `ns`, `us` (µs), `ms`, or `s`. A `bytes` value can be at most 4,294,967,295 bytes. Attributes of type `string` that hold a list, such as `PoolAllocationMap`, hold eight comma-separated entries, where the index is the traffic class or the pool number, 0 to 7. The platform shows them in square brackets, for example `[0.9, 0.1, 0, 0, 0, 0, 0, 0]`; send them without the brackets when you change one ([Change values in a configuration](#change-values-in-a-configuration)).

## Switch

These attributes sit at the top level of the switch component's `typedParameters`.

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `NumUpLinks` | `uint` | `64` | Number of uplink connections available: the ports dedicated to uplink connections. The number is advisory; a topology that needs more uplink ports is allowed. |
| `NumDownLinks` | `uint` | `64` | Number of downlink connections available: the ports dedicated to downlink connections. The number is advisory; a topology that needs more downlink ports is allowed. |
| `NumUpLinksPerSwitch` | `uint` | `8` | Number of parallel uplink connections to each uplink peer switch. |
| `SwitchingCapacity` | `datarate` | `51200Gbps` | Compared with the sum of the switch's port speeds to show whether the switch is oversubscribed. It does not limit what the switch forwards. |
| `EcmpHashMethod` | `string` | `crc32WithSalt` | Hash that sets a flow's identity for equal-cost path selection: `dpr`, `hash`, `hashWithSalt`, `crc32`, `crc32WithSalt`, or `dprWithFallback`. `crc32WithSalt` de-correlates adjacent ECMP tiers; `hashWithSalt` only partially; `crc32`, `hash`, `dpr`, and `dprWithFallback` do not. See [Flow identity](./packet-handling.md#flow-identity). |
| `LoadBalancingMethod` | `string` | `none` | Load balancing method across an equal-cost set: `none` or `flowlet`. `none` selects the member as the flow hash modulo the set size. `flowlet` moves a flow, at a gap in that flow, to the member of its equal-cost set with the least egress occupancy, and by default prefers members that PFC is not pausing for the flow's traffic class. See [Dynamic load balancing](./packet-handling.md#dynamic-load-balancing-flowlet). |
| `EnableLoadBalancingStats` | `bool` | `true` | Write the load balancing interval record. Each row states what was assigned to one member of one equal-cost set in the interval. Written with `none` as well as `flowlet`. |
| `LoadBalancingStatsReportInterval` | `timeval` | `1ms` | Spacing between rows of the load balancing interval record. Minimum `10us`. |

## Network interfaces

The `NetworkInterface` component holds three sub-components: `UplinkNetworkInterface` and `DownlinkNetworkInterface`, the switch's uplink and downlink ports, and `TransmissionMedium`, the link.

### UplinkNetworkInterface

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `DataRate` | `datarate` | `400Gbps` | Link speed of the uplink interfaces. The link runs at the lower of this device's rate and its uplink peer's. |

### DownlinkNetworkInterface

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `DataRate` | `datarate` | `400Gbps` | Link speed of the downlink interfaces. The link runs at the lower of this device's rate and its downlink peer's. |

### TransmissionMedium

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `Delay` | `timeval` | `500ns` | Propagation delay of the links between this switch and its uplink peers (`ps`, `ns`, `us`, or `ms`). A link takes its delay from the device at its lower end, so the links to this switch's downlink peers use those devices' own `Delay`. |

## Load balancing flowlet

The `LoadBalancingFlowlet` component holds the parameters of the `flowlet` load balancing method. The switch reads them only when `LoadBalancingMethod` is `flowlet`, but checks the range of `Gap` whenever the configuration carries the component.

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `Gap` | `timeval` | `100us` | Smallest period with no packet on a flow after which the switch selects that flow's equal-cost member again. At `0s` the switch selects a member for each packet. Range `0s` to `2s`. A move keeps a flow's packets in order only when the gap is at least the difference in latency between the old and the new path. |
| `PFCAware` | `bool` | `true` | When `true`, a load balancing decision prefers members whose egress port PFC has not paused for the packet's traffic class. The switch selects a paused member only when every member of the equal-cost set is paused. |

## Shared buffer manager

These attributes sit in the `SharedBufferManager` object of the switch's `typedParameters`. [Shared buffer manager](./shared-buffer.md) shows how they combine at startup, with a worked example of the defaults.

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `TotalSharedBufferSize` | `bytes` | `256MB` | Total packet buffer of the switch. Headroom is reserved from it first, and the pools are allocated from the rest. |
| `PoolAllocationMap` | `string` | `[0.9, 0.1, 0, 0, 0, 0, 0, 0]` | Fraction (0.0 to 1.0) of the buffer left after headroom that each pool gets; the index is the pool number. The entries are meant to sum to 1 ([what happens when they do not](#settings-the-switch-does-not-check)). A `0` disables that pool. |
| `TrafficClassPoolMapping` | `string` | `[0, 1, 0, 0, 0, 0, 0, 0]` | Pool for each traffic class: the index is the traffic class and the value the pool number, so index 3 set to 2 assigns TC3 to pool 2. This forms the priority groups. Map each PFC-enabled class to the pool with its own number ([PFC traffic classes and pools](#pfc-traffic-classes-and-pools)). |
| `EnablePFC` | `string` | `[1, 1, 0, 0, 0, 0, 0, 0]` | Which traffic classes are lossless: `1` enables PFC for the class at that index, `0` leaves it lossy. |
| `HeadRoomPerPortLimit` | `bytes` | `256KB` | Headroom per port. The switch reserves this much for every connected port, and a paused port may hold this much in headroom in each pool that carries a lossless class. |
| `StaticPoolXoffThreshold` | `bytes` | `700KB` | Ingress occupancy per port and pool above which a PFC pause (Xoff) frame is sent to the upstream sender. The same value applies to every port and pool. |
| `StaticPoolXonThreshold` | `bytes` | `300KB` | Ingress occupancy per port and pool at or below which a PFC resume (Xon) frame is sent to the upstream sender. The switch checks it each time a lossless packet that arrived on the port leaves the switch with the port's headroom empty. Set it below `StaticPoolXoffThreshold`: the gap between the two is the flow control hysteresis band. With Xon at or above Xoff there is no band, so a paused port resumes soon after its headroom drains, and the next arrival pauses it again. |
| `LossyAlpha` | `double` | `1.0` | Multiplier on a pool's free egress space that gives one lossy egress queue's limit, recomputed for each packet. `0` switches to a fixed per-port limit, chosen by `QueueDropThresholdEnable`. |
| `QueueDropThresholdEnable` | `bool` | `false` | Applies only when `LossyAlpha` is `0`. When `true`, each lossy egress queue is capped at `QueueDropThreshold × PlaneBDP` bytes. When `false`, each pool is divided equally between the connected ports. |
| `QueueDropThreshold` | `uint` | `5` | Size of each lossy egress queue as a multiple of `PlaneBDP`. Applies only when `LossyAlpha` is `0` and `QueueDropThresholdEnable` is `true`. |
| `PlaneBDP` | `bytes` | `300KB` | The fabric's bandwidth-delay product. Applies only when `LossyAlpha` is `0` and `QueueDropThresholdEnable` is `true`, and then sets the egress drop limit and the base of the ECN thresholds. Set it to the longest-path round-trip time × the lower of the sender and receiver NIC speeds: for a 6 µs base round-trip time at 400 Gbps, 300 KB; at 800 Gbps, 600 KB. |
| `BufferStatsReportInterval` | `timeval` | `1ms` | Spacing, in simulation time, between rows of the `scala-switch-agg-buff-stats` record. |

## ECN handler

The `EcnHandler` object in the switch's `typedParameters` configures the ECN marker. [ECN marking](./packet-handling.md#ecn-marking) explains which limit the two thresholds are fractions of.

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `EnableEcn` | `bool` | `true` | When `true`, packets marked ECT (ECN-Capable Transport) are considered for ECN marking. |
| `EwmaWeight` | `double` | `0.10` | Weight of the exponentially weighted moving average of egress queue usage: `meanQlen = (1 - weight) × meanQlen + weight × currQlen`. |
| `MinThreshold` | `double` | `0.2` | Fraction (0.0 to 1.0) of the egress queue's static limit below which no packet is marked. Above it, packets are marked with rising probability. |
| `MaxThreshold` | `double` | `0.8` | Fraction (0.0 to 1.0) of the egress queue's static limit at or above which every ECT packet is marked. |
| `MaxMarkProbability` | `double` | `1.0` | The highest marking probability while the average is between the two thresholds (0.0 to 1.0). The probability rises linearly from 0 at `MinThreshold` toward this value at `MaxThreshold`. |

## Setting attributes through the platform

Attributes are set per configuration, through the platform API. The reference pages for the two APIs involved are [Components](../../api-reference/components.md) and [Configurations](../../api-reference/configurations.md). Get an API token first ([Authentication](../../authentication.md)) and export it as `SCALA_API_TOKEN`.

### Read the switch's parameters

`GET /api/v1/components` lists the component types the platform offers; the Scala Switch entry gives its `id` and its `versions`. `GET /api/v1/components/{component_id}`, with the `version` query parameter set to the model version these pages describe, returns the switch's `typedParameters`. Each attribute in the tables above appears there as an object with its `value`, its `type`, a `unit` for dimensional types, and, where the model sets them, its `default`, `description`, `min`, `max`, and allowed values in `enum`. A sub-component, such as `NetworkInterface` or `SharedBufferManager`, is a nested object named after it.

For example:

```bash
curl -X GET "https://api.scalacomputing.com/api/v1/components" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"

# Set COMPONENT_ID to the id of the Scala Switch entry in that response.
export COMPONENT_ID="model_..."

curl -X GET "https://api.scalacomputing.com/api/v1/components/$COMPONENT_ID?version=3.7.0" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

### Change values in a configuration

`PATCH /api/v1/configurations/{config_id}/components/{component_name}/parameters` merges a patch into the `typedParameters` of the component named `component_name` in the configuration; `GET /api/v1/configurations/{config_id}/components` lists the configuration's components and their names. Set `CONFIG_ID` to the configuration's ID and `SWITCH_NAME` to the `name` of the switch component you are changing. A configuration can hold several Scala Switch components, for example one for each tier; each has its own `typedParameters`, and a patch changes only the component it names.

The patch repeats the nesting of the component's `typedParameters`, down to each attribute you change:

- Only keys the component already has can be updated. An unknown key returns 400.
- Each attribute's `type` must match the stored type, or the request returns 400. The request also returns 400 for an empty patch, or when a nested object is sent where an attribute is expected, or the reverse.
- The `value` and the `unit` you send replace the stored ones, so send the `unit` with every dimensional value.
- A string `value` cannot contain shell metacharacters, among them `[` `]` `{` `}` `(` `)` `'` `"` `;` `$` `*` `?` `#` `~`, or the request returns 400. Send a list attribute without its brackets, for example `"value": "0.8, 0.2, 0, 0, 0, 0, 0, 0"`.
- A 200 response returns the component's full, updated `typedParameters`.

This example sets the switch's uplink `DataRate` to 800 Gbps and its `Delay` to 750 ns. An uplink runs at 800 Gbps only where the `DataRate` of the uplink peer's `DownlinkNetworkInterface` is also 800 Gbps or more, and the switch's `Delay` applies to the links to its uplink peers (see [Network interfaces](#network-interfaces)):

```bash
curl -X PATCH "https://api.scalacomputing.com/api/v1/configurations/$CONFIG_ID/components/$SWITCH_NAME/parameters" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "NetworkInterface": {
      "UplinkNetworkInterface": {
        "DataRate": { "value": 800, "unit": "Gbps", "type": "datarate" }
      },
      "TransmissionMedium": {
        "Delay": { "value": 750, "unit": "ns", "type": "timeval" }
      }
    }
  }'
```

A switch the platform added carries every attribute at its Default, so a patch only needs the attributes you change. For a switch whose `typedParameters` you supplied yourself, see [How to read the tables](#how-to-read-the-tables).

## Configuration rules that stop a simulation

The switch checks these rules before or as the simulation starts, and stops the run when one is broken:

- `EnablePFC` or `TrafficClassPoolMapping` does not have exactly eight entries, or has an entry that is not a whole number.
- `PoolAllocationMap` does not have exactly eight entries, or has an entry that is not a number.
- `PoolAllocationMap` has no non-zero entry.
- `PoolAllocationMap` sums to more than 1 by more than its last non-zero entry.
- The total headroom, `HeadRoomPerPortLimit` × connected ports, is larger than `TotalSharedBufferSize`.
- `LoadBalancingMethod` is `flowlet` and `EcmpHashMethod` is `dpr` or `dprWithFallback`.
- `Gap` is outside `0s` to `2s`. This is checked whenever the configuration carries the `LoadBalancingFlowlet` component, whatever the method.
- `LoadBalancingStatsReportInterval` is below `10us`.

### Settings the switch does not check

- **A `PoolAllocationMap` that does not sum to 1** is otherwise accepted. The shortfall or excess is added to or taken from the last pool with a non-zero entry, but that pool's ECN thresholds, and with `LossyAlpha` = `0` its fixed egress limit, still come from its size before the adjustment. Make the entries sum to 1.

## PFC traffic classes and pools

PFC behaves as these pages describe when each PFC-enabled class n is mapped to pool n, as the default mapping does.
