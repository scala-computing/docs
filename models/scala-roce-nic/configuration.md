---
title: "Configuration"
description: "This page lists the Scala RoCE NIC attributes, with their types and defaults, and shows how to set them. A simulation configuration can set every attribute..."
---

This page lists the Scala RoCE NIC attributes, with their types and defaults, and shows how to set them. A simulation configuration can set every attribute listed except the read-only `NumDownLinks`.

The tables list the attributes the components API offers for the Scala RoCE NIC, grouped by the component that holds them: the NIC itself, its network interface and buffers, its PCIe host interface, its transport layer, its queue pair manager, and its ECN handler. The page ends with the configuration rules that stop a simulation. [Transport](./transport.md), [Buffers and PFC](./buffers-and-pfc.md), and [Congestion control](./congestion-control.md) explain the behavior behind them.

## How to read the tables

**Default** is the value an attribute carries in a NIC the platform adds to a configuration, for example with `POST /api/v1/configurations/{config_id}/components`. Such a NIC carries every attribute in these tables at its Default until you change it.

If you supply a NIC's `typedParameters` yourself instead, for example in the `components` of `POST /api/v1/configurations`, include every attribute of each sub-component you send, such as `IngressBufferManager` or `ECNHandler`: an attribute left out of it runs the model's built-in value, which can differ from the Default shown here. Attributes at the NIC's top level, and sub-components you leave out entirely, take their Defaults.

**Type** is the value of the attribute's `type` field in the API, which a patch must repeat. Values of type `bytes`, `queuesize`, `datarate`, and `timeval` carry a `unit`: sizes are in `B`, `KB`, `KiB`, `MB`, or `MiB`, where `KB` is 1,000 bytes, `KiB` 1,024 bytes, and `MB` 1,000,000 bytes; data rates are in `Gbps` or `Mbps`; times are in `ps`, `ns`, `us` (µs), `ms`, or `s`. Attributes of type `string` that hold a list, such as `PoolAllocationVector`, hold eight comma-separated entries, where the index is the traffic class, 0 to 7; `PoolAllocationVector` also has a rule for where its non-zero entries go ([Buffer pools](./buffers-and-pfc.md#buffer-pools)). The platform shows them in square brackets, for example `[0.9, 0.1, 0, 0, 0, 0, 0, 0]`. Send them the same way when you change one: as a string, with the brackets and all eight entries ([Change values in a configuration](#change-values-in-a-configuration)).

## NIC

These attributes sit at the top level of the NIC component's `typedParameters`.

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `NumDownLinks` | `uint` | `1` | Read-only. Number of hosts the NIC serves: one, the server it is placed on. |
| `EnableBufferStats` | `bool` | `false` | When `true`, the NIC's receive and transmit pools, one buffer for each class with a pool, take part in the network buffer statistics record, `net-buff-stats`, when the global `EnableNetworkBufferStatsReporting` (default `false`) is also `true` ([Records the buffers and PFC write](./buffers-and-pfc.md#records-the-buffers-and-pfc-write)). |

## Network interface

The `NetworkInterface` component holds four sub-components: `UplinkNetworkInterface`, the NIC's Ethernet port to its rack switch; `TransmissionMedium`, the link; and `IngressBufferManager` and `EgressBufferManager`, the receive and transmit buffers.

### UplinkNetworkInterface

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `DataRate` | `datarate` | `400Gbps` | Rate of the NIC's port. The link to the rack switch runs at the lower of this rate and the `DataRate` of the rack switch's `DownlinkNetworkInterface` ([Scala Switch configuration](../scala-switch/configuration.md#downlinknetworkinterface)). That lower rate is the NIC's line rate, and every other mention of `DataRate` on these pages means line rate. Every frame the NIC sends goes out at line rate, except a congested queue pair's data, which goes out at that queue pair's allowed rate ([Congestion control](./congestion-control.md#how-the-rate-is-applied)). Line rate is also the base of the PFC pause time and of the congestion control floor. |

### TransmissionMedium

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `Delay` | `timeval` | `100ns` | Propagation delay of the link between the NIC and its rack switch: a frame arrives this long after its serialization ends. It is also added to the PFC pause time the NIC computes ([PFC](./buffers-and-pfc.md#pfc)). |

### IngressBufferManager

The receive buffer. [Receive buffer](./buffers-and-pfc.md#receive-buffer) and [PFC](./buffers-and-pfc.md#pfc) explain how it is used.

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `TotalBufferSize` | `bytes` | `4MB` | Total size of the receive buffer. Each traffic class's receive pool is its `PoolAllocationVector` entry times this size. |
| `QueueSchedulingType` | `string` | `StrictPriority` | Accepts `StrictPriority`, `RoundRobin`, or `WeightedRoundRobin`. Received data packets are written to the host in arrival order with any of them ([Receive buffer](./buffers-and-pfc.md#receive-buffer)). |
| `PoolAllocationVector` | `string` | `[0.9, 0.1, 0, 0, 0, 0, 0, 0]` | Fraction (0.0 to 1.0) of `TotalBufferSize` that each traffic class gets as its receive pool; the index is the traffic class. The non-zero entries must come first, from class 0 with no `0` between them; the classes after them have no receive pool. The pools are sized independently, so entries that sum to more than 1 allocate more than `TotalBufferSize` ([Buffer pools](./buffers-and-pfc.md#buffer-pools)). |
| `PFCEnableVector` | `string` | `[1, 1, 0, 0, 0, 0, 0, 0]` | Which traffic classes the NIC sends PFC pause frames for: `1` enables PFC for the class at that index, and any other value leaves it disabled. The NIC honors pause frames it receives for every class, whatever this says. |
| `WRRQueueWeights` | `string` | `[1, 3, 0, 0, 0, 0, 0, 0]` | Weights for `WeightedRoundRobin`, one per traffic class. They don't change the order of received packets ([Receive buffer](./buffers-and-pfc.md#receive-buffer)). |
| `PfcStaticXoffThreshold` | `string` | `[0.8, 0.8, 0, 0, 0, 0, 0, 0]` | Fraction of each class's receive pool at which the NIC sends a PFC pause frame: when the pool's occupancy, with an arriving packet, reaches it. The rest of the pool above it is headroom for what arrives after the pause ([Sizing the headroom](./buffers-and-pfc.md#sizing-the-headroom)). |
| `PfcXonResumeThreshold` | `string` | `[0.4, 0.4, 0, 0, 0, 0, 0, 0]` | Fraction of each class's receive pool at or below which the NIC sends XON as received data drains to the host. At each refresh of a pause, every half pause time, the NIC also sends XON once occupancy is below the Xoff threshold ([Resuming](./buffers-and-pfc.md#resuming)). Set it below `PfcStaticXoffThreshold`: with Xon at or above Xoff, the NIC resumes the switch almost as soon as it pauses it. |
| `QuantaValue` | `string` | `[65535, 65535, 0, 0, 0, 0, 0, 0]` | Pause quanta the NIC requests in each pause frame it sends for a class. One quantum is 512 bit times, so 65535 quanta pause a 400 Gbps port for 83.9 µs plus the link's propagation delay. The NIC also re-checks the pause every half of this time, measured at its own `DataRate` with the link's `Delay` added. |

### EgressBufferManager

The transmit buffer. [Egress buffer](./buffers-and-pfc.md#egress-buffer) and [Transmit scheduling](./buffers-and-pfc.md#transmit-scheduling) explain how it is used.

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `TotalBufferSize` | `bytes` | `256KB` | Total size of the transmit buffer. Each traffic class's transmit pool is its `PoolAllocationVector` entry times this size. |
| `QueueSchedulingType` | `string` | `StrictPriority` | How the network port chooses the next traffic class to send: `StrictPriority` (highest-numbered class first), `RoundRobin`, or `WeightedRoundRobin`. PFC and congestion notification frames always go first ([Transmit scheduling](./buffers-and-pfc.md#transmit-scheduling)). |
| `PoolAllocationVector` | `string` | `[0.9, 0.1, 0, 0, 0, 0, 0, 0]` | Fraction (0.0 to 1.0) of `TotalBufferSize` that each traffic class gets as its transmit pool; the index is the traffic class. The non-zero entries must come first, from class 0 with no `0` between them. Each queue pair's data class needs a pool of at least one full packet, and its control class a pool ([Egress buffer](./buffers-and-pfc.md#egress-buffer)). A pool bounds the data of its class the NIC has fetched from the host and not yet sent. The pools are sized independently. |
| `WRRQueueWeights` | `string` | `[1, 3, 0, 0, 0, 0, 0, 0]` | With `WeightedRoundRobin`, the most packets each traffic class sends in one turn; the index is the traffic class. Give each queue pair's data class and control class a weight of at least 1 ([Queue pairs and messages](./transport.md#queue-pairs-and-messages)). Not used by the other scheduling types. |

## Host interface

The `HostInterface` component holds the PCIe link between the NIC and its host, as two sub-components: `NICSide`, the NIC's end of the link, and `HostSide`, the host's end. [Fetching the payload over PCIe](./transport.md#fetching-the-payload-over-pcie) explains how a transfer uses them, with a worked example of the rates.

Each end holds a container named `PCIeGen3.0`. The name does not set the PCIe generation: each end's `LaneBps` does, and the default 31.52 Gbps per lane is the PCIe 5.0 rate.

### NIC side

On the NIC's end of the link, each attribute's path in the NIC's `typedParameters` is:

- `RXHeaderBufferSize`, `TXHeaderBufferSize`, `RXDataBufferSize`, `TXDataBufferSize`: `HostInterface` → `NICSide` → `PCIeGen3.0` → `NicDeviceLayer` → `NicPCIeInterface` → `TransactionLayer`
- `LaneBps`, `LaneCount`: `HostInterface` → `NICSide` → `PCIeGen3.0` → `NicDeviceLayer` → `NicPCIeInterface` → `PhysicalLayer`
- `ReadRequestSize`: `HostInterface` → `NICSide` → `NICPCIeHandler`

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `RXHeaderBufferSize` | `queuesize` | `16KB` | Size of the transaction-layer buffers for the headers of transfers the NIC receives from the host. |
| `TXHeaderBufferSize` | `queuesize` | `16KB` | Size of the transaction-layer buffers for the headers of transfers the NIC sends to the host. |
| `RXDataBufferSize` | `queuesize` | `256KB` | Size of transaction-layer buffers for transfer payload on the NIC's end of the link. `RXDataBufferSize` and `TXDataBufferSize` together size the payload buffers in both directions. |
| `TXDataBufferSize` | `queuesize` | `256KB` | Size of transaction-layer buffers for transfer payload on the NIC's end of the link. `RXDataBufferSize` and `TXDataBufferSize` together size the payload buffers in both directions. |
| `LaneBps` | `datarate` | `31.52Gbps` | Rate of one PCIe lane in the NIC-to-host direction. Any rate can be set, so the link can represent any PCIe generation. |
| `LaneCount` | `uint` | `16` | Number of PCIe lanes, `1` to `16`. The NIC-to-host direction runs at `LaneCount` × `LaneBps`. |
| `ReadRequestSize` | `bytes` | `64B` | Size of each PCIe read request the NIC sends to fetch a segment's payload from host memory, one request per segment. It is the request's own cost on the NIC-to-host direction, not the size of the data it fetches. |

### Host side

On the host's end of the link:

- `RXHeaderBufferSize`, `TXHeaderBufferSize`, `RXDataBufferSize`, `TXDataBufferSize`: `HostInterface` → `HostSide` → `PCIeGen3.0` → `ScalaHostPCIeDeviceLayer` → `HostPCIeInterface` → `TransactionLayer`
- `LaneBps`, `LaneCount`: `HostInterface` → `HostSide` → `PCIeGen3.0` → `ScalaHostPCIeDeviceLayer` → `HostPCIeInterface` → `PhysicalLayer`

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `RXHeaderBufferSize` | `queuesize` | `16KB` | Size of the transaction-layer buffers for the headers of transfers the host receives from the NIC. |
| `TXHeaderBufferSize` | `queuesize` | `16KB` | Size of the transaction-layer buffers for the headers of transfers the host sends to the NIC. |
| `RXDataBufferSize` | `queuesize` | `256KB` | Size of transaction-layer buffers for transfer payload on the host's end of the link. `RXDataBufferSize` and `TXDataBufferSize` together size the payload buffers in both directions. |
| `TXDataBufferSize` | `queuesize` | `256KB` | Size of transaction-layer buffers for transfer payload on the host's end of the link. `RXDataBufferSize` and `TXDataBufferSize` together size the payload buffers in both directions. |
| `LaneBps` | `datarate` | `31.52Gbps` | Rate of one PCIe lane in the host-to-NIC direction. |
| `LaneCount` | `uint` | `16` | Number of PCIe lanes, `1` to `16`. The host-to-NIC direction runs at `LaneCount` × `LaneBps`. |

## Transport processing layers

The `TransportProcessingLayers` component holds the NIC's processing layers. Only the transport layer has attributes.

### RoceTransportLayer

[Acknowledgements](./transport.md#acknowledgements) and [Retransmission timer](./transport.md#retransmission-timer) explain how these combine.

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `RetransmitTimeout` | `timeval` | `64ms` | Interval of each queue pair's transport timer. The timer starts when a packet is given its PSN while no other PSN is outstanding and restarts on each ACK that retires a PSN while PSNs remain outstanding. It stops when no PSN is outstanding. On expiry the queue pair resends from its oldest outstanding PSN. There is no retry limit, and any time value is accepted. |
| `AckCoalescingEnabled` | `bool` | `true` | When `true`, the receiver acknowledges by count and by timer: an ACK goes out on every (`AckThreshold` + 1)-th packet, on the last packet of each message, and when `AckDelay` expires with packets held. When `false`, every in-order or duplicate data packet is acknowledged. |
| `AckDelay` | `timeval` | `1ms` | With coalescing, the longest the receiver holds an acknowledgement: the timer starts with a held packet when it is idle, and again with each ACK sent by the count or by the last packet of a message, and sends an ACK for the held packets when it expires. |
| `AckThreshold` | `uint` | `16` | With coalescing, the number of packets the receiver holds before the next one sends an ACK. |

## ScalaRoceQpManager

The `ScalaRoceQpManager` component manages the NIC's queue pairs.

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `MSS` | `bytes` | `4096B` | Largest payload of one data packet. A message of N bytes is sent as `ceil(N ÷ MSS)` packets, each `MSS` + 58 bytes on the wire at most. `MSS` must be from 1 to 9,216 bytes, and `MSS` + 58 bytes must be smaller than the configuration's global `Mtu` ([Segmentation](./transport.md#segmentation)). |

## ECNHandler

The `ECNHandler` component holds the rate control loop's settings, which apply to every queue pair on the NIC. [The rate control loop](./congestion-control.md#the-rate-control-loop) explains each stage.

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `EnableECN` | `bool` | `true` | When `true`, data is sent ECN-capable, ECT(0), and the CNPs a queue pair receives drive its rate control loop. When `false`, data is sent Not-ECT, so switches do not mark it, CNPs are ignored, and every queue pair runs at line rate. The NIC sends CNPs for CE-marked packets it receives either way. |
| `MinimumRatePercentage` | `double` | `0.10` | Floor of a congested queue pair's rate, as a fraction (0.0 to 1.0) of `DataRate`. |
| `FastRecoveryAttempt` | `uint` | `3` | Number of fast-recovery increases after the reduction that begins an episode or a reduction made during fast recovery, each moving the rate halfway toward the target, before additive increase begins. Reductions during additive increase count back down toward fast recovery ([The rate control loop](./congestion-control.md#the-rate-control-loop)). |
| `InitialTargetRatePercentage` | `double` | `0.90` | Sizes the additive step of an episode: one minus this value, times the target rate at the start of the episode, divided by `NumberActiveIncreaseSteps`. The episode's first target rate is the rate the queue pair had when it began. |
| `NumberActiveIncreaseSteps` | `uint` | `10` | Divisor of the additive step. More steps mean a smaller step and a slower climb back to line rate. |
| `ControlIntervalDurationNs` | `uint` | `45000` | Length of the control interval, in nanoseconds. The loop runs every half interval, alternating a reduction half and an increase half, and counts CNPs and packets over one interval. |
| `CongestionProbabilityWeight` | `double` | `0.5` | Weight of each new interval in the moving average of the congestion estimate. The higher the weight, the faster the estimate rises with marks and the sooner it forgets them. |
| `NumOfControlLoopsPerIncrease` | `uint` | `5` | While CNPs keep arriving, the rate is raised only on every (this value + 1)-th increase half that saw CNPs. |

## Setting attributes through the platform

Attributes are set per configuration, through the platform API. The reference pages for the two APIs involved are [Components](../../api-reference/components.md) and [Configurations](../../api-reference/configurations.md). Get an API token first ([Authentication](../../authentication.md)) and export it as `SCALA_API_TOKEN`.

### Read the NIC's parameters

`GET /api/v1/components` lists the component types the platform offers; the Scala RoCE NIC entry, `ScalaRoceNic`, gives its `id` and its `versions`. `GET /api/v1/components/{component_id}`, with the `version` query parameter set to the model version these pages describe, returns the NIC's `typedParameters`. Each attribute in the tables above appears there as an object with its `value`, its `type`, a `unit` for dimensional types, and, where the model sets them, its `default`, `description`, `min`, `max`, and allowed values in `enum`; `NumDownLinks` also carries `readonly`. A sub-component, such as `NetworkInterface` or `ECNHandler`, is a nested object named after it.

For example:

```bash
curl -X GET "https://api.scalacomputing.com/api/v1/components" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"

# Set COMPONENT_ID to the id of the Scala RoCE NIC entry in that response.
export COMPONENT_ID="model_..."

curl -X GET "https://api.scalacomputing.com/api/v1/components/$COMPONENT_ID?version=4.6.0" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

### Add the NIC to a server

A Scala RoCE NIC belongs to a server component: every instance of that server carries one NIC with the same `typedParameters`. Placing it takes two calls:

1. `POST /api/v1/configurations/{config_id}/components`, with the query parameters `modelId=ScalaRoceNic`, `type=nic`, a `name` for the NIC component, and `version`, adds the NIC to the configuration with every attribute at its Default.
2. `PATCH /api/v1/configurations/{config_id}/configurations/{component_name}`, where `component_name` is the server component, adds the NIC to that server. The body names the NIC component, with `type` `nic` and `count` 1.

Set `CONFIG_ID` to the configuration's ID and `SERVER_NAME` to the `name` of the server component, which must already be in the configuration; `GET /api/v1/configurations/{config_id}/components` lists the configuration's components and their names. `modelId` takes the model name, `ScalaRoceNic`, or its `id` from `GET /api/v1/components`.

```bash
curl -X POST "https://api.scalacomputing.com/api/v1/configurations/$CONFIG_ID/components?modelId=ScalaRoceNic&type=nic&name=Nic&version=4.6.0" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"

curl -X PATCH "https://api.scalacomputing.com/api/v1/configurations/$CONFIG_ID/configurations/$SERVER_NAME" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "name": "Nic", "type": "nic", "count": 1 }'
```

Servers that need different NIC settings need different NIC components, each added to its own server component. The applications that run over the NIC are the [Chakra workload](../chakra-workload/configuration.md) and the Scala RDMA application.

### Change values in a configuration

`PATCH /api/v1/configurations/{config_id}/components/{component_name}/parameters` merges a patch into the `typedParameters` of the component named `component_name` in the configuration; `GET /api/v1/configurations/{config_id}/components` lists the configuration's components and their names. Set `CONFIG_ID` to the configuration's ID and `NIC_NAME` to the `name` of the NIC component you are changing. A patch changes only the component it names, and so every server that carries that NIC component.

The patch repeats the nesting of the component's `typedParameters`, down to each attribute you change, as nested objects, one per name in the attribute's path:

- Only keys the component already has can be updated. An unknown key returns 400.
- Each attribute's `type` must match the stored type, or the request returns 400. The request also returns 400 for an empty patch, or when a nested object is sent where an attribute is expected, or the reverse.
- The `value` and the `unit` you send replace the stored ones, so send the `unit` with every dimensional value.
- A list attribute's `value` is a string with its square brackets and exactly eight entries, each made of letters, digits, `.`, `_`, `+`, or `-`, for example `"value": "[0.7, 0.7, 0, 0, 0, 0, 0, 0]"`. A list in any other form returns 400.
- Where an attribute has `enum` values, its `value` must be one of them, or the request returns 400.
- A 200 response returns the component's full, updated `typedParameters`.

This example lowers the PFC thresholds of classes 0 and 1 to 70% (Xoff) and 30% (Xon) of each receive pool, and shortens the transport timer to 16 ms:

```bash
curl -X PATCH "https://api.scalacomputing.com/api/v1/configurations/$CONFIG_ID/components/$NIC_NAME/parameters" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "NetworkInterface": {
      "IngressBufferManager": {
        "PfcStaticXoffThreshold": { "value": "[0.7, 0.7, 0, 0, 0, 0, 0, 0]", "type": "string" },
        "PfcXonResumeThreshold": { "value": "[0.3, 0.3, 0, 0, 0, 0, 0, 0]", "type": "string" }
      }
    },
    "TransportProcessingLayers": {
      "RoceTransportLayer": {
        "RetransmitTimeout": { "value": 16, "unit": "ms", "type": "timeval" }
      }
    }
  }'
```

A NIC the platform added carries every attribute at its Default, so a patch only needs the attributes you change. For a NIC whose `typedParameters` you supplied yourself, see [How to read the tables](#how-to-read-the-tables).

Global simulation parameters, such as `NDStatsReportInterval` and the link MTU, `Mtu` in the `GenericNetDevice` section of `DefaultNetworkParameters` (4,184 bytes on the platform), are not attributes of this model; change them with `PATCH /api/v1/configurations/{config_id}/parameters`.

## Configuration rules that stop a simulation

A configuration write returns 400 when a value breaks the rules in [Change values in a configuration](#change-values-in-a-configuration), when a dimensional value has a unit its type does not accept, when a `uint` is negative or fractional, when `LaneCount` is outside `1` to `16`, or when a `bool` is neither `true` nor `false`.

The NIC checks these rules before the simulation starts, and stops the run when one is broken:

- An ingress or egress `PoolAllocationVector` has an entry that is not a number, has a negative entry, or has no non-zero entry.
- An entry of `WRRQueueWeights` or `QuantaValue` is not a whole number, or an entry of `PFCEnableVector`, `PfcStaticXoffThreshold`, or `PfcXonResumeThreshold` is not a number.
- `MSS` is outside 1 to 9,216 bytes, or `MSS` + 58 bytes is not smaller than the configuration's `Mtu`. With the platform's 4,184-byte `Mtu`, the largest `MSS` is 4,125 bytes.
- `LaneCount` is outside `1` to `16`.
- A dimensional value has no number, is negative, or has a unit its type does not accept, or a `timeval` is finer than 1 ps.
- A server with a Scala RoCE NIC runs an application other than the Chakra workload or the Scala RDMA application.
- A switch in the topology has `UETPolicyEnabled` set to `true`. That policy assigns traffic classes from codepoints the Scala RoCE NIC does not send ([Buffer under the UET policy](../scala-switch/shared-buffer.md#buffer-under-the-uet-policy)).
