---
title: "Configuration"
description: "This page lists the Scala MRC NIC attributes, with their types and defaults, and shows how to set them."
---

This page lists the Scala MRC NIC attributes, with their types and defaults, and shows how to set them.

A simulation configuration can set every attribute listed except the read-only `NumDownLinks`. The tables list the attributes that set the Scala MRC NIC's behavior, grouped by the component that holds them: the NIC itself, its network interface and buffers, its PCIe host interface, the queue pair manager, packet spraying, congestion control, and the MRC Monitor. The page ends with the configuration rules that stop a simulation. [Transport](./transport.md), [Multipath](./multipath.md), [Congestion control](./congestion-control.md), [Buffers and PCIe](./buffers-and-pcie.md), and [Records](./records.md) explain the behavior behind them.

## How to read the tables

**Default** is the value an attribute carries in a NIC the platform adds to a configuration, for example with `POST /api/v1/configurations/{config_id}/components`. Such a NIC carries every attribute in these tables at its Default until you change it.

If you supply a NIC's `typedParameters` yourself instead, for example in the `components` of `POST /api/v1/configurations`, include every attribute of each sub-component you send, such as `IngressBufferManager` or `ScalaMrcNscc`: an attribute left out of it runs the model's built-in value, which can differ from the Default shown here. Attributes at the NIC's top level, and sub-components you leave out entirely, take their Defaults.

**Type** is the value of the attribute's `type` field in the API, which a patch must repeat. Values of type `bytes`, `queuesize`, `datarate`, and `timeval` carry a `unit`: sizes are in `B`, `KB`, `KiB`, `MB`, or `MiB`, where `KB` is 1,000 bytes, `KiB` 1,024 bytes, and `MB` 1,000,000 bytes; data rates are in `Gbps` or `Mbps`; times are in `ps`, `ns`, `us` (µs), `ms`, or `s`. A `uint` is a whole number of 0 or more, a `double` a decimal number, and a `bool` `true` or `false`. A `string` takes one of the values its row lists or, for the monitor's filters and triggers, a comma-separated list. Where a description starts with a range, such as `0` to `7` or `1ns` to `8s`, a value outside it is refused when written or stops the run ([Configuration rules that stop a simulation](#configuration-rules-that-stop-a-simulation)).

## NIC

These attributes sit at the top level of the NIC component's `typedParameters`.

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `EnableTransportStats` | `bool` | `true` | When `true`, the NIC writes one row of the `mrc-transport-stats` record every `TransportStatsReportInterval`, with its transport counters, congestion windows, and round-trip times ([mrc-transport-stats](./records.md#mrc-transport-stats)). |
| `TransportStatsReportInterval` | `timeval` | `1ms` | At least `10us`. Interval between the rows of `mrc-transport-stats`, in simulation time; the first row is written one interval after the simulation starts. This record follows the NIC's own interval, not the global `NDStatsReportInterval`. |
| `EnableEventLogging` | `bool` | `false` | When `true`, the NIC writes a row of the `mrc-events` record each time one of its queue pairs or work requests is created, completes, enters the error state, or is closed ([mrc-events](./records.md#mrc-events)). |
| `EnableMonitor` | `bool` | `false` | When `true`, turns on the per-packet `mrc-monitor` record, which the `ScalaMrcMonitor` attributes configure ([mrc-monitor](./records.md#mrc-monitor)). The monitor's settings take effect once per simulator process: give every Scala MRC NIC in a configuration the same `EnableMonitor` and monitor settings. |
| `NumDownLinks` | `uint` | `1` | Read-only. Number of hosts the NIC serves: one, the server it is placed on. |

## Network interface

The `NetworkInterface` component holds four sub-components: `UplinkNetworkInterface`, the NIC's Ethernet port to its rack switch; `TransmissionMedium`, the link; and `IngressBufferManager` and `EgressBufferManager`, the receive and transmit buffers.

### UplinkNetworkInterface

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `DataRate` | `datarate` | `400Gbps` | Line rate of the NIC's port. The link to the rack switch runs at the lower of this rate and the rack switch's downlink `DataRate`, and the NIC sends at, and computes with, that link rate: it sets the bandwidth-delay product behind congestion control's maximum window ([The congestion window](./congestion-control.md#the-congestion-window)) and the size of the active set of entropy values ([The active set](./multipath.md#the-active-set)). |

### TransmissionMedium

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `Delay` | `timeval` | `100ns` | Propagation delay of the link between the NIC and its rack switch: a frame arrives this long after its serialization ends. The link takes its delay from the NIC's `Delay`, not from the switch's. |

### IngressBufferManager

The receive buffer. [Buffer pools](./buffers-and-pcie.md#buffer-pools) and [Receive buffer](./buffers-and-pcie.md#receive-buffer) explain how it is used.

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `TotalBufferSize` | `bytes` | `4MB` | Total size of the receive buffer. A fixed 90 % of it is the data class's pool and 10 % the control class's: 3,600,000 and 400,000 bytes at the Default. A received packet that its pool cannot hold is dropped. |

### EgressBufferManager

The transmit buffer. [Egress buffer](./buffers-and-pcie.md#egress-buffer) and [Transmit scheduling](./buffers-and-pcie.md#transmit-scheduling) explain how it is used.

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `TotalBufferSize` | `bytes` | `256KB` | Total size of the transmit buffer. A fixed 90 % of it is the data class's pool and 10 % the control class's: 230,400 and 25,600 bytes at the Default. A data packet reserves its full size on the wire, 4,174 bytes at the default `MSS`, from the start of its payload fetch until it has been serialized, so the data pool bounds the data fetched and not yet sent: 55 full packets at the Default. When the pool is full, the NIC fetches no more data until space frees; it does not drop data packets. |

## Host interface

The `HostInterface` component holds the PCIe link between the NIC and its host, as two sub-components: `NICSide`, the NIC's end of the link, and `HostSide`, the host's end. [PCIe host interface](./buffers-and-pcie.md#pcie-host-interface) explains how a transfer uses them, and [Worked example: PCIe and network rates](./buffers-and-pcie.md#worked-example-pcie-and-network-rates) compares the link's rates at the defaults with the network port's.

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
| `LaneCount` | `uint` | `16` | `1` to `16`. Number of PCIe lanes. The NIC-to-host direction runs at `LaneCount` × `LaneBps` and carries the NIC's payload read requests and the received data it writes to the host. To keep up with a full-rate 400 Gbps receive stream it needs about 418.6 Gbps: 14 lanes at the default `LaneBps` are enough, 13 are not, and the receive buffer then fills and drops packets. |
| `ReadRequestSize` | `bytes` | `64B` | Size of the read request the NIC sends to the host for each data packet's payload. It adds to the NIC-to-host PCIe traffic and does not change how much payload is fetched ([Fetching the payload over PCIe](./transport.md#fetching-the-payload-over-pcie)). |

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
| `LaneCount` | `uint` | `16` | `1` to `16`. Number of PCIe lanes. The host-to-NIC direction runs at `LaneCount` × `LaneBps` and carries the payload the NIC fetches for each data packet. |

## ScalaMrcQpManager

These attributes sit under `ScalaMrcQpManager`, at the top level of the NIC's `typedParameters`. They are the NIC's queue pair settings: packet marking, segmentation, the PSN window, QP groups, the spraying type, loss detection, the retransmission timer and retry budget, and RTT measurement. [Transport](./transport.md) explains each.

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `TrimmingEnabled` | `bool` | `true` | When `true`, the NIC marks the first transmission of each data packet, and each reliability probe, trimmable, so a switch that trims may cut a packet it cannot queue down to its headers instead of dropping it; when `false`, it marks them no-trim ([Packet marking](./transport.md#packet-marking)). As a receiver, the NIC sends a NACK for a trimmed packet only when its own `TrimmingEnabled` is `true`; when it is `false`, the trimmed packet's PSN stays a gap that loss detection from SACK gaps or the retransmission timer recovers ([NACKs](./transport.md#nacks)). The NIC only marks packets; switches do the trimming, which needs their UET policy and PFC off ([Packet trimmer](../scala-switch/configuration.md#packet-trimmer)). It also sets congestion control's operating point: with `true`, the target queue delay is 0.75 × `ConfigBaseRTT` and Quick Adapt has no delay trigger; with `false`, the target is `ConfigBaseRTT` and `QuickAdaptThreshold` applies. `OverrideSpecTargetQueueDelay` `true` replaces the target, not the trigger rule ([Target queue delay](./congestion-control.md#target-queue-delay)). |
| `UseDistinctDSCPForRetransmit` | `bool` | `true` | When `true`, every retransmitted data packet is marked as a trimmable retransmission, whatever `TrimmingEnabled` says, so a trimming switch can trim retransmissions even when `TrimmingEnabled` is `false`. A switch under the UET policy also gives such packets a larger egress queue limit ([Buffer under the UET policy](../scala-switch/shared-buffer.md#buffer-under-the-uet-policy)). When `false`, a retransmission carries the same mark as the packet's first transmission. |
| `MSS` | `uint` | `4096` | At least `1`. Largest payload of one data packet; a message is cut into packets of at most this payload. Each full-size data packet is `MSS` + 78 bytes on the wire, 4,174 bytes at the Default, or `MSS` + 82 bytes with `RTTTimestampMode` `WireTimestamp`. It also sets the window's unit, a full packet of `MSS` + 60 bytes, the unit of congestion control's window floor and of `MaxCongestionWindow`; with `WireTimestamp`, each data packet counts 4 bytes more in flight. At `4096` or more the receiver sends a SACK for every data packet; below it, one SACK can cover two, three, or four data packets, more as `MSS` falls ([Segmentation](./transport.md#segmentation), [Selective acknowledgements](./transport.md#selective-acknowledgements)). |
| `MaxPSNRange` | `uint` | `1024` | `1` to `32640`. The PSN window: a queue pair sends no new data packet more than this many PSNs beyond its cumulative ACK, 4 MiB of payload at the defaults. Retransmissions are not held back by it. As a receiver, the NIC drops a data packet whose PSN is more than its own `MaxPSNRange` beyond its cumulative PSN, and answers it with a PSN-out-of-range NACK ([NACKs](./transport.md#nacks)). |
| `QPGroupsEnabled` | `bool` | `true` | When `true`, all the NIC's queue pairs to the same remote IP address form one group: they share one congestion window, and the transmit scheduler serves the group one packet per turn. When `false`, each queue pair has its own window and turn. Either way, the NIC keeps one packet spraying state for each destination ([QP groups and the transmit scheduler](./transport.md#qp-groups-and-the-transmit-scheduler), [Spraying and QP groups](./multipath.md#spraying-and-qp-groups)). |
| `PacketSprayingType` | `string` | `Default Packet Spraying` | How the NIC chooses each packet's entropy value, the UDP source port the switches' ECMP hash includes: `None` (every packet of a queue pair takes that queue pair's own port, with no path feedback or probes) or `Default Packet Spraying` (the NIC sprays over an active set of ports drawn from the range `DefaultPacketSpraying` sets, takes a path out of use after a retransmission timeout, and probes it back) ([Spraying types](./multipath.md#spraying-types)). |
| `SACKLossMinRetxPkts` | `uint` | `512` | Reorder depth for loss detection from SACK gaps: a data packet not yet acknowledged is presumed lost, and retransmitted, once a SACK reports a PSN more than this many above it. At the defaults that is about 42.8 µs of line-rate sending after the loss. At `0`, any gap below the highest PSN a SACK reports is presumed lost at once, so packets that spraying only reordered are retransmitted. At `MaxPSNRange` − 1 or more, SACK gaps never lead to a retransmission ([Loss detection from SACK gaps](./transport.md#loss-detection-from-sack-gaps)). |
| `InitialAckTimeout` | `timeval` | `150us` | `1ns` to `8s`. Base retransmission timeout of each data packet: a packet's first `LinearRetryCount` timeouts last this long, and each later one doubles, until it reaches a fixed cap of about 17.18 s. A retransmission after a NACK or a SACK gap starts the packet's schedule again. The NIC checks its timers every 1 µs, so a value below 1 µs acts as 1 µs and a fraction of a µs is dropped. It also sets how often the sender asks again for the cumulative ACK that completes a message, every 2 × `InitialAckTimeout`, 300 µs at the Default ([Retransmission timer and retry budget](./transport.md#retransmission-timer-and-retry-budget), [Completion](./transport.md#completion)). |
| `ReliabilityProbePeriod` | `timeval` | `150us` | `1ns` to `8s`. Interval between reliability probes: every period, the NIC sends one probe to each destination with paths it has taken out of use after a retransmission timeout, taking those paths in turn. A path whose probe is answered goes back into use. Used only with `Default Packet Spraying` ([Path feedback and probes](./multipath.md#path-feedback-and-probes)). |
| `FixedRexmitTimeEnable` | `bool` | `false` | When `true`, every retransmission timeout of a packet lasts `InitialAckTimeout`, with no doubling. The retry budget still applies. |
| `LinearRetryCount` | `uint` | `7` | `0` to `7`. Number of a packet's timeouts that last `InitialAckTimeout` before the doubling begins; also part of the retry budget. At `0`, the first timeout already lasts 2 × `InitialAckTimeout`. |
| `ExponentialRetryCount` | `uint` | `25` | `0` to `25`. The rest of the retry budget: after `LinearRetryCount` + `ExponentialRetryCount` retransmissions of a packet after timeouts, the next timeout puts its queue pair into an error state, in which it sends nothing more and its message never completes. Retransmissions after NACKs have a budget of the same size, and a NACK beyond it also puts the queue pair into the error state. `25` means the NIC never gives up. With `ExponentialRetryCount` `0` and the other defaults, a packet is retransmitted 7 times and its queue pair enters the error state at the 8th timeout, 1,350 µs after the first send ([Worked example: the retry schedule at the defaults](./transport.md#worked-example-the-retry-schedule-at-the-defaults)). |
| `RTTTimestampMode` | `string` | `LocalDatabase` | How the sender measures an RTT sample from a SACK: `LocalDatabase` from the time it stored when the data packet finished sending, less the service time the receiver reports; `WireTimestamp` from a timestamp each data packet carries and the SACK reflects, in units of 128 ns, which adds 4 bytes to every data packet on the wire ([RTT measurement](./transport.md#rtt-measurement)). |

## PacketSpraying

The `PacketSpraying` component, at the top level of the NIC's `typedParameters`, holds one sub-component, `DefaultPacketSpraying`.

### DefaultPacketSpraying

These attributes sit under `PacketSpraying` → `DefaultPacketSpraying`, and apply only when `PacketSprayingType` is `Default Packet Spraying` ([Default packet spraying](./multipath.md#default-packet-spraying)).

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `EntropyMaskBits` | `uint` | `7` | `0` to `8`. The range of entropy values is 2^`EntropyMaskBits` UDP source ports, starting at `StartingPortOffset`: 128 ports, 256 to 383, at the defaults. The NIC sprays over an active set drawn from that range, 72 of the 128 ports at the defaults ([The active set](./multipath.md#the-active-set)). At `0`, every packet takes the port `StartingPortOffset`. |
| `StartingPortOffset` | `uint` | `256` | `0` to `65535`. First UDP source port of the range. The range must end at port 65,535 or below: `StartingPortOffset` + 2^`EntropyMaskBits` − 1 must not exceed 65,535, so the largest offset is 65,408 at 7 bits and 65,280 at 8. A larger one stops the run when the NIC sets up its first queue pair ([Configuration rules that stop a simulation](#configuration-rules-that-stop-a-simulation)). |
| `ActiveEVSetSize` | `uint` | `0` | Number of entropy values the NIC sprays over for each destination, drawn at random from the range. At `0`, the NIC sizes it to the packets in one bandwidth-delay product at `ConfigBaseRTT`, at most 100: 300,000 ÷ 4,156 = 72 at the defaults. Any value is capped at 2^`EntropyMaskBits`. At `1`, or at `0` when one bandwidth-delay product holds less than one packet, each destination uses a single port. |

## ScalaMrcNscc

These attributes sit under `ScalaMrcNscc`, at the top level of the NIC's `typedParameters`, and apply to every congestion window on the NIC: one for each destination with `QPGroupsEnabled` `true`, one for each queue pair otherwise. [How each attribute shapes the window](./congestion-control.md#how-each-attribute-shapes-the-window) puts them together.

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `PassThrough` | `bool` | `false` | When `true`, the congestion window no longer limits sending: NSCC still computes the window, and the record columns that show it keep moving, but it holds back no packet. Sending is then bounded by the egress buffer, `MaxPSNRange`, and the PCIe link, and almost every data packet requests a cumulative ACK ([Turning congestion control off](./congestion-control.md#turning-congestion-control-off)). |
| `ConfigBaseRTT` | `timeval` | `6000ns` | At least `1ns`. Round-trip time of the longest unloaded path in the fabric, the base RTT each congestion window starts from. With the link rate it sets the bandwidth-delay product (BDP), 300,000 bytes at the defaults, and from it the initial and maximum window, 1.5 × BDP = 450,000 bytes. It also sets the target queue delay, 0.75 × `ConfigBaseRTT` with `TrimmingEnabled` `true` or `ConfigBaseRTT` with it `false`, unless `OverrideSpecTargetQueueDelay` is `true`, sets the scale of the increase routines, and sizes the active set of entropy values. A lower measured RTT becomes the base RTT and re-derives the maximum window, 75 bytes for each ns of base RTT at 400 Gbps, but not the target ([Base RTT](./congestion-control.md#base-rtt)). |
| `AlphaMultiplier` | `double` | `4.0` | `1.0` to `100.0`. Scales proportional increase, the growth on SACKs whose queueing delay is below the target: a larger value grows the window faster. |
| `FairIncreaseMultiplier` | `double` | `5.0` | `1.0` to `100.0`. Scales fair increase, the growth on SACKs whose queueing delay is at or above the target: a larger value grows the window faster. |
| `FastIncreaseMultiplier` | `double` | `1.0` | `0.01` to `100.0`. Scales fast increase, the growth once SACKs with queueing delay below both the target and `TargetQueueDelayMargin` have acknowledged more than a full window of data: a larger value grows the window faster. |
| `EtaMultiplier` | `double` | `0.15` | `0.0` to `1.0`. Scales eta, the additive term the window update adds, at most once per `ConfigBaseRTT`. At `0.0`, the window update adds no such term. |
| `DelayAlpha` | `double` | `0.25` | `0.0` to `1.0`. Weight of each new queueing-delay sample in the average queueing delay that `mrc-transport-stats` records: average = `DelayAlpha` × sample + (1 − `DelayAlpha`) × average. |
| `QuickAdaptGate` | `uint` | `3` | `0` to `31`. Quick Adapt cuts the window, to the bytes acknowledged in its last observation window, only when a trigger is present and those bytes are below the maximum window divided by 2^`QuickAdaptGate`: 450,000 ÷ 2^3 = 56,250 bytes at the defaults. An observation window lasts the base RTT plus the target queue delay, 10.5 µs at the defaults ([Quick Adapt](./congestion-control.md#quick-adapt)). |
| `DisableQuickAdapt` | `bool` | `false` | When `true`, Quick Adapt never runs. |
| `QuickAdaptThreshold` | `timeval` | `24000ns` | With `TrimmingEnabled` `false`, the queueing delay above which a SACK is a Quick Adapt trigger: the SACK's own delay sample is compared, not the average. With `TrimmingEnabled` `true`, the default, it is not used. A NACK for a trimmed packet is the other trigger ([Quick Adapt](./congestion-control.md#quick-adapt)). |
| `OverrideSpecMaxCongestionWindow` | `bool` | `false` | When `true`, the maximum window, and the window each congestion window starts at, is `MaxCongestionWindow` nominal packets instead of 1.5 × BDP, and a lower measured base RTT no longer changes it ([The congestion window](./congestion-control.md#the-congestion-window)). |
| `MaxCongestionWindow` | `uint` | `107` | `1` to `4096`. Maximum window in nominal packets of `MSS` + 60 bytes, 4,156 bytes at the default `MSS`; used only with `OverrideSpecMaxCongestionWindow` `true`. 107 packets are 107 × 4,156 = 444,692 bytes, just under the 450,000 bytes the NIC derives at the defaults. |
| `OverrideSpecTargetQueueDelay` | `bool` | `false` | When `true`, the target queue delay is `TargetQueueDelay`. When `false`, it is 0.75 × `ConfigBaseRTT` with `TrimmingEnabled` `true`, 4.5 µs at the defaults, or `ConfigBaseRTT` with `TrimmingEnabled` `false` ([Target queue delay](./congestion-control.md#target-queue-delay)). |
| `TargetQueueDelay` | `timeval` | `4500ns` | Target queue delay when `OverrideSpecTargetQueueDelay` is `true`; not used otherwise. It then also scales proportional increase and sets the length of Quick Adapt's observation window, as the derived target does. With the override, a value below `1ns`, `0` included, stops the run. |
| `TargetQueueDelayMargin` | `timeval` | `1000ns` | Queueing delay below which SACKs whose queueing delay is also below the target count toward fast increase: once such SACKs have acknowledged more than a full window of data, the window grows faster, until a SACK whose queueing delay is at or above the margin but below the target. It is used only by fast increase ([How the window responds to each SACK](./congestion-control.md#how-the-window-responds-to-each-sack)). |
| `ReferenceNetworkRTT` | `timeval` | `6000ns` | With a fixed reference link rate of 400 Gbps, defines a reference BDP, 300,000 bytes at the Default. The proportional, fair, and fast increase gains and the additive term scale with the NIC's BDP relative to the reference BDP, so raising this value makes them smaller. It does not change the target queue delay or the maximum window. A value below `1ns`, `0` included, stops the run. |

## ScalaMrcMonitor

These attributes sit under `ScalaMrcMonitor`, at the top level of the NIC's `typedParameters`, and configure the per-packet `mrc-monitor` record, which `EnableMonitor` on the NIC turns on ([mrc-monitor](./records.md#mrc-monitor)). The monitor's settings take effect once per simulator process: give every Scala MRC NIC in a configuration the same `EnableMonitor` and monitor settings.

The filters select queue pairs. Each is `All`, which matches every queue pair, or a comma-separated list, which a queue pair matches when it matches any entry; a queue pair is logged when it matches every filter that is not `All`. The `Log` toggles then choose which of its packets are logged: a packet needs the toggle for its type and the toggle for its direction, and a received NACK also the toggle for its code.

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `EnableMonitorRowCap` | `bool` | `true` | When `true`, the monitor writes at most `MonitorRowCap` rows in each simulator process. When `false`, there is no limit. |
| `MonitorRowCap` | `uint` | `1000000` | The most rows the monitor writes in one simulator process, sent and received rows of all its NICs together, with `EnableMonitorRowCap` `true`. Once the cap is reached, the monitor logs nothing more for the rest of the run. At `0`, it writes no rows. |
| `MonitorTriggers` | `string` | `None` | `None`, or a comma-separated set of the events that select a queue pair for logging from the moment they happen: `trim` when the NIC receives a trimmed data packet (from a sender with the default `RTTTimestampMode`), `nack` when it receives a NACK, `qp_error` when a queue pair enters the error state, and `rto` when a retransmission timer expires. Written in lower case. A trigger selects only the queue pair the event happened on, and only with `EnableMonitor` `true`; with every filter at `All`, every queue pair is already logged. |
| `EnableMonitorTriggerQpCap` | `bool` | `true` | When `true`, the triggers select at most `MonitorTriggerQpCap` queue pairs in each simulator process. When `false`, there is no limit. |
| `MonitorTriggerQpCap` | `uint` | `64` | The most queue pairs the triggers together can select in one simulator process, with `EnableMonitorTriggerQpCap` `true`. At `0`, the triggers select none. |
| `FilterLocalEndpoint` | `string` | `All` | `All`, or a list of queue pair ends written `<IPv4 address>:<QP id>`, matched against the queue pair's local end: this NIC's address and its own QP id ([Identifying a NIC and a queue pair](./records.md#identifying-a-nic-and-a-queue-pair)). |
| `FilterRemoteEndpoint` | `string` | `All` | `All`, or a list of ends written `<IPv4 address>:<QP id>`, matched against the queue pair's remote end: the other NIC's address and its QP id, which reads `0` until this NIC learns it. |
| `FilterEitherEndpoint` | `string` | `All` | `All`, or a list of ends written `<IPv4 address>:<QP id>`: a queue pair matches when its local or its remote end is listed. |
| `FilterLocalAddr` | `string` | `All` | `All`, or a list of IPv4 addresses in dotted form, matched against the queue pair's local address. |
| `FilterRemoteAddr` | `string` | `All` | `All`, or a list of IPv4 addresses in dotted form, matched against the queue pair's remote address. |
| `FilterEitherAddr` | `string` | `All` | `All`, or a list of IPv4 addresses in dotted form: a queue pair matches when its local or its remote address is listed. |
| `FilterNodeId` | `string` | `All` | `All`, or a list of node ids, in decimal, of the NICs whose queue pairs are logged, as in the `node_id` column of the records. |
| `LogWrite` | `bool` | `true` | When `true`, data packets are logged. |
| `LogAck` | `bool` | `true` | When `true`, cumulative ACKs are logged. |
| `LogSack` | `bool` | `true` | When `true`, SACKs are logged, the probe responses the NIC sends included. |
| `LogNack` | `bool` | `true` | When `true`, NACKs are logged. |
| `LogProbe` | `bool` | `true` | When `true`, reliability probes are logged. |
| `LogTx` | `bool` | `true` | When `true`, packets the NIC sends are logged, other than retransmissions and completion re-requests. |
| `LogRx` | `bool` | `true` | When `true`, packets the NIC receives are logged. |
| `LogRtx` | `bool` | `true` | When `true`, retransmitted data packets are logged, with the packets the sender sends again to ask for a message's cumulative ACK. |
| `LogNackTrimmed` | `bool` | `true` | When `true`, received NACKs with the trimmed code are logged. |
| `LogNackTrimmedLastHop` | `bool` | `true` | When `true`, received NACKs with the trimmed-on-the-last-hop code are logged. |
| `LogNackPsnOor` | `bool` | `true` | When `true`, received NACKs with the PSN-out-of-range code are logged. |
| `LogNackUnexp` | `bool` | `true` | When `true`, received NACKs with the unexpected-event code are logged. |

## Setting attributes through the platform

Attributes are set per configuration, through the platform API. The reference pages for the two APIs involved are [Components](../../api-reference/components.md) and [Configurations](../../api-reference/configurations.md). Get an API token first ([Authentication](../../authentication.md)) and export it as `SCALA_API_TOKEN`.

### Read the NIC's parameters

`GET /api/v1/components` lists the component types the platform offers; the Scala MRC NIC entry, `ScalaMrcNic`, gives its `id` and its `versions`. `GET /api/v1/components/{component_id}`, with the `version` query parameter set to the model version these pages describe, returns the NIC's `typedParameters`. Each attribute in the tables above appears there as an object with its `value`, its `type`, a `unit` for dimensional types, and, where the model sets them, its `default`, `description`, `min`, `max`, and allowed values in `enum`; `NumDownLinks` also carries `readonly`. A sub-component, such as `NetworkInterface` or `ScalaMrcNscc`, is a nested object named after it. Sub-components with no attribute to set, such as the four layers under `TransportProcessingLayers` and `ScalaMrcPCIePayloadFetchManager`, appear as empty objects.

For example:

```bash
curl -X GET "https://api.scalacomputing.com/api/v1/components" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"

# Set COMPONENT_ID to the id of the Scala MRC NIC entry in that response.
export COMPONENT_ID="model_..."

curl -X GET "https://api.scalacomputing.com/api/v1/components/$COMPONENT_ID?version=4.0.5" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

### Add the NIC to a server

A Scala MRC NIC belongs to a server component: every instance of that server carries one NIC with the same `typedParameters`. Placing it takes two calls:

1. `POST /api/v1/configurations/{config_id}/components`, with the query parameters `modelId=ScalaMrcNic`, `type=nic`, a `name` for the NIC component, and `version`, adds the NIC to the configuration with every attribute at its Default.
2. `PATCH /api/v1/configurations/{config_id}/configurations/{component_name}`, where `component_name` is the server component, adds the NIC to that server. The body names the NIC component, with `type` `nic` and `count` 1.

Set `CONFIG_ID` to the configuration's ID and `SERVER_NAME` to the `name` of the server component, which must already be in the configuration; `GET /api/v1/configurations/{config_id}/components` lists the configuration's components and their names. `modelId` takes the model name, `ScalaMrcNic`, or its `id` from `GET /api/v1/components`.

```bash
curl -X POST "https://api.scalacomputing.com/api/v1/configurations/$CONFIG_ID/components?modelId=ScalaMrcNic&type=nic&name=Nic&version=4.0.5" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"

curl -X PATCH "https://api.scalacomputing.com/api/v1/configurations/$CONFIG_ID/configurations/$SERVER_NAME" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "name": "Nic", "type": "nic", "count": 1 }'
```

Servers that need different NIC settings need different NIC components, each added to its own server component. A Scala MRC NIC exchanges traffic only with other Scala MRC NICs, so give every server it sends to or receives from a Scala MRC NIC as well. The workload that runs over the NIC on the platform is the [Chakra workload](../chakra-workload/index.md).

### Change values in a configuration

`PATCH /api/v1/configurations/{config_id}/components/{component_name}/parameters` merges a patch into the `typedParameters` of the component named `component_name` in the configuration; `GET /api/v1/configurations/{config_id}/components` lists the configuration's components and their names. Set `CONFIG_ID` to the configuration's ID and `NIC_NAME` to the `name` of the NIC component you are changing. A patch changes only the component it names, and so every server that carries that NIC component.

The patch repeats the nesting of the component's `typedParameters`, down to each attribute you change, as nested objects, one per name in the attribute's path:

- Only keys the component already has can be updated. An unknown key returns 400.
- Each attribute's `type` must match the stored type, or the request returns 400. The request also returns 400 for an empty patch, or when a nested object is sent where an attribute is expected, or the reverse.
- The `value` and the `unit` you send replace the stored ones, so send the `unit` with every dimensional value.
- Where an attribute has `enum` values, its `value` must be one of them, or the request returns 400. Send a value that contains spaces as it is, for example `"value": "Default Packet Spraying"`.
- Any other string `value`, such as a monitor filter, cannot contain shell metacharacters, among them `[` `]` `{` `}` `(` `)` `'` `"` `;` `$` `*` `?` `#` `~` `!` `&` `<` `>`, or the request returns 400. Spaces and `.`, `:`, `,`, `-`, and `_` are allowed, so a list of queue pair ends, each an IPv4 address, a colon, and a QP id, or a trigger set such as `trim,nack`, can be sent as it is.
- A 200 response returns the component's full, updated `typedParameters`.

This example enlarges the transmit buffer to 512 KB, turns packet spraying off, and lengthens the base retransmission timeout to 200 µs:

```bash
curl -X PATCH "https://api.scalacomputing.com/api/v1/configurations/$CONFIG_ID/components/$NIC_NAME/parameters" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "NetworkInterface": {
      "EgressBufferManager": {
        "TotalBufferSize": { "value": 512, "unit": "KB", "type": "bytes" }
      }
    },
    "ScalaMrcQpManager": {
      "PacketSprayingType": { "value": "None", "type": "string" },
      "InitialAckTimeout": { "value": 200, "unit": "us", "type": "timeval" }
    }
  }'
```

A NIC the platform added carries every attribute at its Default, so a patch only needs the attributes you change. For a NIC whose `typedParameters` you supplied yourself, see [How to read the tables](#how-to-read-the-tables).

Global simulation parameters, such as `NDStatsReportInterval` and `EnableNetDeviceStatLogging`, are not attributes of this model; change them with `PATCH /api/v1/configurations/{config_id}/parameters`.

## Configuration rules that stop a simulation

A Scala MRC NIC exchanges traffic only with other Scala MRC NICs: every NIC that sends to or receives from a Scala MRC NIC must be a Scala MRC NIC ([What is and is not modeled](./index.md#what-is-and-is-not-modeled)).

A configuration write returns 400 when a value breaks the rules in [Change values in a configuration](#change-values-in-a-configuration), when a dimensional value has a unit its type does not accept, when a `uint` is negative or fractional, when a `bool` is neither `true` nor `false`, or when one of these attributes is outside the range its row gives: `MSS`, `MaxPSNRange`, `LinearRetryCount`, `ExponentialRetryCount`, `EntropyMaskBits`, `StartingPortOffset`, and either `LaneCount`.

`POST /api/v1/configurations/{config_id}/validate` returns the configuration's validation errors and warnings. A configuration with a Scala MRC NIC is `invalid` when one of the attributes in the previous paragraph is outside its range, for example `LaneCount` outside `1` to `16`.

The switch rules that involve this NIC are on the Scala Switch's page ([Configuration rules that stop a simulation](../scala-switch/configuration.md#configuration-rules-that-stop-a-simulation)). A switch with `UETPolicyEnabled` set to `true` accepts the Scala MRC NIC, which sends the codepoints the UET policy reads, and switch packet trimming needs the UET policy and PFC off on that switch.

The validation also warns, without making the configuration `invalid`, when a Scala MRC NIC has `PacketSprayingType` set to `Default Packet Spraying`, its Default, and a switch has `LoadBalancingMethod` set to `flowlet`; the simulation still runs. Packet spraying and flowlet switching both choose the paths packets take: the NIC through each packet's entropy value ([Multipath](./multipath.md)), and the switch by moving flows between equal-cost members ([Dynamic load balancing (flowlet)](../scala-switch/packet-handling.md#dynamic-load-balancing-flowlet)).

The validation also warns, without making the configuration `invalid`, when a Scala MRC NIC with `TrimmingEnabled` `true`, its Default, shares the topology with a switch that does not trim: a Scala Switch whose `PacketTrimmer` `TrimmingEnabled` is not `true`, or a switch of another model. With `TrimmingEnabled` `true`, Quick Adapt fires only after a trim NACK, which only a trimming switch produces ([Quick Adapt](./congestion-control.md#quick-adapt)). The simulation still runs. To clear the warning, set the NIC's `TrimmingEnabled` to `false`, or turn trimming on, with the UET policy, at every switch ([Packet trimmer](../scala-switch/configuration.md#packet-trimmer)).

The NIC stops the run before the simulation starts when one of these attributes is outside the range its row gives: `TransportStatsReportInterval`, `InitialAckTimeout`, `ReliabilityProbePeriod`, `ConfigBaseRTT`, `AlphaMultiplier`, `FairIncreaseMultiplier`, `FastIncreaseMultiplier`, `EtaMultiplier`, `DelayAlpha`, `QuickAdaptGate`, or `MaxCongestionWindow`.

The NIC stops the run when it sets up its first queue pair, as a sender or a receiver, when:

- `PacketSprayingType` is `Default Packet Spraying` and `StartingPortOffset` + 2^`EntropyMaskBits` − 1 is larger than 65,535. The largest `StartingPortOffset` is 65,408 at the default 7 bits, and 65,280 at 8.
- `OverrideSpecTargetQueueDelay` is `true` and `TargetQueueDelay` is below `1ns`, `0` included.
- `ReferenceNetworkRTT` is below `1ns`, `0` included.
- `TrimmingEnabled` is `true`, `OverrideSpecTargetQueueDelay` is `false`, and `ConfigBaseRTT` is below `2ns`.
