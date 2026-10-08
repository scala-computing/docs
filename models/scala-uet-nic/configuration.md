---
title: "Configuration"
description: "This page lists the Scala UET NIC attributes, with their types and defaults, and shows how to set them."
---

This page lists the Scala UET NIC attributes, with their types and defaults, and shows how to set them.

A simulation configuration can set every attribute listed except the read-only `NumDownLinks`. The tables list the attributes that set the Scala UET NIC's behavior, grouped by the component that holds them: the NIC itself, its network interface and buffers, its transport processing layers (packet delivery, packet spraying, congestion control, segmentation, and the UET Monitor), and its PCIe host interface. The page ends with the configuration rules that stop a simulation. [Transport](./transport.md), [Multipath](./multipath.md), [Congestion control](./congestion-control.md), [Buffers and PFC](./buffers-and-pfc.md), and [Records](./records.md) explain the behavior behind them.

## How to read the tables

**Default** is the value an attribute carries in a NIC the platform adds to a configuration, for example with `POST /api/v1/configurations/{config_id}/components`. Such a NIC carries every attribute in these tables at its Default until you change it.

If you supply a NIC's `typedParameters` yourself instead, for example in the `components` of `POST /api/v1/configurations`, include every attribute of each sub-component you send, such as `IngressBufferManager` or `ScalaUETCongestionControl`: an attribute left out of it runs the model's built-in value, which can differ from the Default shown here. Attributes at the NIC's top level, and sub-components you leave out entirely, take their Defaults.

**Type** is the value of the attribute's `type` field in the API, which a patch must repeat. Values of type `bytes`, `queuesize`, `datarate`, and `timeval` carry a `unit`: sizes are in `B`, `KB`, `KiB`, `MB`, or `MiB`, where `KB` is 1,000 bytes, `KiB` 1,024 bytes, and `MB` 1,000,000 bytes; data rates are in `Gbps` or `Mbps`; times are in `ps`, `ns`, `us` (µs), `ms`, or `s`. A `uint` is a whole number of 0 or more, a `double` a decimal number, and a `bool` `true` or `false`. Where a description starts with a range, such as `0` to `15`, a value outside it is refused ([Configuration rules that stop a simulation](#configuration-rules-that-stop-a-simulation)). Attributes of type `string` that hold a list, such as `PoolAllocationVector`, hold eight comma-separated entries, where the index is the traffic class, 0 to 7; `PoolAllocationVector` also has a rule for where its non-zero entries go ([Buffer pools](./buffers-and-pfc.md#buffer-pools)). The platform shows them in square brackets, for example `[0.9, 0.1, 0, 0, 0, 0, 0, 0]`. Send them the same way when you change one: as a string, with the brackets and all eight entries ([Change values in a configuration](#change-values-in-a-configuration)).

## NIC

These attributes sit at the top level of the NIC component's `typedParameters`.

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `NumDownLinks` | `uint` | `1` | Read-only. Number of hosts the NIC serves: one, the server it is placed on. |
| `EnableBufferStats` | `bool` | `false` | When `true`, the NIC's receive and transmit pools, one buffer for each class with a pool, take part in the network buffer statistics record, `net-buff-stats`, when the global `EnableNetworkBufferStatsReporting` (default `false`) is also `true` ([net-buff-stats](./records.md#net-buff-stats)). |
| `EnableTransportStats` | `bool` | `true` | When `true`, the NIC writes one row of the `uet-transport-stats` record every `NDStatsReportInterval`, with its transport counters, congestion window, and round-trip times ([uet-transport-stats](./records.md#uet-transport-stats)). No global parameter turns this record on or off. |
| `EnableEventLogging` | `bool` | `false` | When `true`, the NIC writes a row of the `uet-events` record each time one of its PDCs or work requests is created, completes, or is closed ([uet-events](./records.md#uet-events)). |

## Network interface

The `NetworkInterface` component holds four sub-components: `UplinkNetworkInterface`, the NIC's Ethernet port to its rack switch; `TransmissionMedium`, the link; and `IngressBufferManager` and `EgressBufferManager`, the receive and transmit buffers.

### UplinkNetworkInterface

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `DataRate` | `datarate` | `400Gbps` | Line rate of the NIC's port. The link to the rack switch runs at the lower of this rate and the rack switch's downlink `DataRate`, and the NIC sends at, and computes with, that link rate: it sets the bandwidth-delay product behind congestion control's maximum window ([The congestion window](./congestion-control.md#the-congestion-window)) and the PFC pause time ([PFC](./buffers-and-pfc.md#pfc)). |

### TransmissionMedium

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `Delay` | `timeval` | `100ns` | Propagation delay of the link between the NIC and its rack switch: a frame arrives this long after its serialization ends. The link takes its delay from the NIC's `Delay`, not from the switch's. It is also added to the PFC pause time the NIC computes ([PFC](./buffers-and-pfc.md#pfc)). |

### IngressBufferManager

The receive buffer. [Receive buffer](./buffers-and-pfc.md#receive-buffer) and [PFC](./buffers-and-pfc.md#pfc) explain how it is used.

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `TotalBufferSize` | `bytes` | `4MB` | Total size of the receive buffer. Each traffic class's receive pool is its `PoolAllocationVector` entry times this size. A received packet that its pool cannot hold is dropped. |
| `QueueSchedulingType` | `string` | `StrictPriority` | Set to `StrictPriority`, `RoundRobin`, or `WeightedRoundRobin`. Received data packets are written to the host in arrival order with any of them ([Receive buffer](./buffers-and-pfc.md#receive-buffer)). |
| `PoolAllocationVector` | `string` | `[0.9, 0.1, 0, 0, 0, 0, 0, 0]` | Fraction (0.0 to 1.0) of `TotalBufferSize` that each traffic class gets as its receive pool; the index is the traffic class. The non-zero entries come first, from class 0 with no `0` between them, and the classes after them have no receive pool. Give classes 0 and 1 a non-zero entry each: with the applications' default traffic classes, data uses class 0 and control class 1 ([Traffic classes](./buffers-and-pfc.md#traffic-classes)). The pools are sized independently, so entries that sum to more than 1 allocate more than `TotalBufferSize` ([Buffer pools](./buffers-and-pfc.md#buffer-pools)). |
| `PFCEnableVector` | `string` | `[0, 0, 0, 0, 0, 0, 0, 0]` | Which traffic classes the NIC sends PFC pause frames for: `1` enables PFC for the class at that index, and any other value leaves it disabled. With the Default the NIC sends no PFC frames. The NIC honors pause frames it receives for every class, whatever this says ([Receiving a pause](./buffers-and-pfc.md#receiving-a-pause)). |
| `WRRQueueWeights` | `string` | `[1, 3, 0, 0, 0, 0, 0, 0]` | Weights for `WeightedRoundRobin`, one per traffic class. They don't change the order of received packets ([Receive buffer](./buffers-and-pfc.md#receive-buffer)). |
| `PfcStaticXoffThreshold` | `string` | `[0.8, 0.8, 0, 0, 0, 0, 0, 0]` | Fraction of each class's receive pool at which the NIC sends a PFC pause frame, for a class `PFCEnableVector` enables: when the pool's occupancy, with an arriving packet, reaches it. The rest of the pool above it is headroom for what arrives after the pause ([Sizing the headroom](./buffers-and-pfc.md#sizing-the-headroom)). |
| `PfcXonResumeThreshold` | `string` | `[0.4, 0.4, 0, 0, 0, 0, 0, 0]` | Fraction of each class's receive pool at or below which the NIC sends XON as received data drains to the host. At each refresh of a pause, every half pause time, the NIC also sends XON once occupancy is below the Xoff threshold ([Resuming](./buffers-and-pfc.md#resuming)). Set it below `PfcStaticXoffThreshold`: with Xon at or above Xoff, the NIC resumes the switch almost as soon as it pauses it. |
| `QuantaValue` | `string` | `[65535, 65535, 0, 0, 0, 0, 0, 0]` | Pause quanta the NIC requests in each pause frame it sends for a class. One quantum is 512 bit times, so 65,535 quanta pause a 400 Gbps port for 83.885 µs, plus the link's `Delay`: 83.985 µs at the defaults. The NIC also re-checks the pause every half of this time, measured at its own link rate with the link's `Delay` added ([Sending a pause](./buffers-and-pfc.md#sending-a-pause)). |

### EgressBufferManager

The transmit buffer. [Egress buffer](./buffers-and-pfc.md#egress-buffer) and [Transmit scheduling](./buffers-and-pfc.md#transmit-scheduling) explain how it is used.

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `TotalBufferSize` | `bytes` | `256KB` | Total size of the transmit buffer. Each traffic class's transmit pool is its `PoolAllocationVector` entry times this size. |
| `QueueSchedulingType` | `string` | `StrictPriority` | Set to `StrictPriority`, `RoundRobin`, or `WeightedRoundRobin`: how the network port chooses the next traffic class to send. `StrictPriority` sends from the highest-numbered class that has a packet waiting and is not paused. PFC frames always go first ([Transmit scheduling](./buffers-and-pfc.md#transmit-scheduling)). |
| `PoolAllocationVector` | `string` | `[0.9, 0.1, 0.0, 0, 0, 0, 0, 0]` | Fraction (0.0 to 1.0) of `TotalBufferSize` that each traffic class gets as its transmit pool; the index is the traffic class. The non-zero entries come first, from class 0 with no `0` between them. Give classes 0 and 1 a non-zero entry each, as for the receive buffer; the data class's pool must hold at least one full data packet, `RdmaDataMSS` + 106 bytes, and the control class's pool at least one 78-byte ACK ([Egress buffer](./buffers-and-pfc.md#egress-buffer)). A pool bounds the packets of its class waiting to be sent: with `PrefetchBufferEnable` `false`, a data packet holds its full wire size from the start of its payload fetch until it has been serialized. The pools are sized independently. |
| `WRRQueueWeights` | `string` | `[1, 5, 0, 0, 0, 0, 0, 0]` | With `WeightedRoundRobin`, the most packets each traffic class sends in one turn; the index is the traffic class. Give classes 0 and 1 a weight of at least 1 each. Not used by the other scheduling types. |

## Transport processing layers

The `TransportProcessingLayers` component holds the NIC's transport. The sections below cover its sub-components with attributes, in the order of the NIC's `typedParameters`.

### ScalaUETPdsManager

These attributes sit under `TransportProcessingLayers` → `ScalaUETPdsManager`, the NIC's packet delivery settings: packet marking, PDC groups, the prefetch buffer, the delivery mode, the spraying type, the retransmission timer, and ACK requests. [Transport](./transport.md) explains each.

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `TrimmingEnabled` | `bool` | `false` | When `true`, the NIC marks its data packets trimmable, so a switch that trims may cut a packet it cannot queue down to its headers instead of dropping it; when `false`, it marks them no-trim ([Packet marking](./transport.md#packet-marking)). The NIC only marks packets; switches do the trimming, which needs their UET policy and PFC off ([Packet trimmer](../scala-switch/configuration.md#packet-trimmer)). It also sets congestion control's operating point: only trimmed packets, not queueing delay, trigger Quick Adapt, and, unless `OverrideSpecTargetQueueDelay` is `true`, the target queue delay becomes 0.75 × `InitialBaseRTT` instead of `InitialBaseRTT` ([Target queue delay](./congestion-control.md#target-queue-delay)). |
| `UseDistinctDSCPForRetransmits` | `bool` | `true` | When `true`, every retransmitted data packet is marked as a trimmable retransmission, whatever `TrimmingEnabled` says, so a trimming switch can trim retransmissions even when `TrimmingEnabled` is `false`. A switch under the UET policy also gives such packets a larger egress queue limit ([Buffer under the UET policy](../scala-switch/shared-buffer.md#buffer-under-the-uet-policy)). When `false`, a retransmission carries the same mark as the packet's first transmission. |
| `PDCGroupsEnabled` | `bool` | `true` | When `true`, all the NIC's PDCs to the same remote IP address form one group: they share one congestion window and one load balancer, the packet spraying state, and the transmit scheduler serves the group one packet per turn. When `false`, each PDC has its own window, load balancer, and turn ([PDC groups and the transmit scheduler](./transport.md#pdc-groups-and-the-transmit-scheduler)). |
| `PrefetchBufferEnable` | `bool` | `false` | When `true`, a payload fetched from the host waits in a prefetch buffer the NIC's PDCs share, sized by `PrefetchBufferSize`, until the congestion window and the egress buffer let it go on the wire; its egress space is reserved then. When `false`, the egress space is reserved, and the window checked, when the fetch is issued ([Prefetch buffer](./transport.md#prefetch-buffer)). |
| `PrefetchBufferSize` | `uint` | `262144` | Size, in bytes, of the prefetch buffer, used only when `PrefetchBufferEnable` is `true`; the Default holds 62 full data packets. At least `4202`, one data packet on the wire at the default `RdmaDataMSS`: a smaller value is refused when written, whatever `PrefetchBufferEnable` is. With the prefetch buffer on, it must also hold one data packet on the wire, `RdmaDataMSS` + 106 bytes, or the run stops ([Configuration rules that stop a simulation](#configuration-rules-that-stop-a-simulation)). |
| `DefaultDeliveryMethod` | `string` | `Reliable Unordered Delivery (RUD)` | Accepts only `Reliable Unordered Delivery (RUD)`: the receiver writes each new data packet to the host as it arrives, in order or not, and nothing is reordered ([Receiving a packet](./transport.md#receiving-a-packet)). |
| `PacketSprayingType` | `string` | `Default Packet Spraying` | How the NIC chooses each data packet's entropy value, the UDP source port the switches' ECMP hash includes: `None` (one fixed value for each PDC group), `Default Packet Spraying` (a range of ports in turn), or `Recycled Entropy Packet Spraying` (reuses the values that ACKs report as uncongested). Each type reads only its own sub-component's attributes ([Spraying types](./multipath.md#spraying-types)). |
| `FailureDetectionRTTMultiplier` | `uint` | `3` | With `Recycled Entropy Packet Spraying` only. A data packet still not acknowledged this many base RTTs after it was sent is reported to the load balancer as a timeout on its path, as a retransmission timeout is; the packet is not retransmitted, and its retransmission timer is unchanged. `0` turns early failure detection off. At the defaults the deadline is 3 × 6 µs = 18 µs until a lower base RTT is measured ([Early failure detection](./multipath.md#early-failure-detection)). |
| `InitialRetransmitTimeout` | `timeval` | `150us` | Sets the scale of each data packet's retransmission timeout: with `FixedRexmitTimeEnable` `false`, a larger value lengthens every timeout in proportion. Each retransmission of a packet doubles that packet's timeout until it reaches the cap that `MaxExponentialRTOBackoffCount` sets, after which it stays fixed. A packet not acknowledged within its timeout is retransmitted, with no retry limit. Not used when `FixedRexmitTimeEnable` is `true` ([Retransmission timer](./transport.md#retransmission-timer)). |
| `MaxExponentialRTOBackoffCount` | `uint` | `5` | `0` to `15`. Caps how far a packet's retransmission timeout grows: each retransmission doubles the timeout until it reaches the cap, and every further retransmission uses the same timeout. A larger value lets the timeout grow longer. |
| `FixedRexmitTimeEnable` | `bool` | `false` | When `true`, every transmission of a data packet, first or retransmitted, uses `FixedRexmitTimeout`, with no doubling. |
| `FixedRexmitTimeout` | `timeval` | `150us` | With `FixedRexmitTimeEnable` `true`, the retransmission timeout of every transmission of a data packet. Not used otherwise. |
| `AckRequestFirstPacket` | `bool` | `true` | When `true`, a PDC's ACK requests count from its first data transmission: the 1st, the (N + 1)th, the (2N + 1)th, and so on request an ACK, where N is `AckRequestEveryNthPacket`. When `false`, the Nth, the 2Nth, and so on do ([Acknowledgements](./transport.md#acknowledgements)). |
| `AckRequestLastPacket` | `bool` | `true` | When `true`, the last packet of each message also requests an ACK. |
| `AckRequestEveryNthPacket` | `uint` | `1` | `1` to `1024`. N: every Nth data transmission on a PDC, retransmissions included, requests an ACK, and the receiver sends an ACK for each packet that requests one. At `1`, every data packet requests an ACK, whatever `AckRequestFirstPacket` and `AckRequestLastPacket` say. |

### DefaultPacketSpraying

These attributes sit under `TransportProcessingLayers` → `PacketSpraying` → `DefaultPacketSpraying`, and apply only when `PacketSprayingType` is `Default Packet Spraying` ([Default packet spraying](./multipath.md#default-packet-spraying)).

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `EntropyMaskBits` | `uint` | `7` | `0` to `8`. Each PDC group's data packets take 2^`EntropyMaskBits` UDP source ports in turn, starting at `StartingPortOffset`: 128 ports at the Default. At `0`, every packet takes the port `StartingPortOffset`. |
| `StartingPortOffset` | `uint` | `256` | `0` to `65535`. First UDP source port of the range: ports 256 to 383 at the defaults. A range that passes 65,535 continues from port 0. |

### RecycledEntropyPacketSpraying

These attributes sit under `TransportProcessingLayers` → `PacketSpraying` → `RecycledEntropyPacketSpraying`, and apply only when `PacketSprayingType` is `Recycled Entropy Packet Spraying` ([Recycled entropy packet spraying](./multipath.md#recycled-entropy-packet-spraying)).

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `REPSBufferSizeMaskBits` | `uint` | `3` | `0` to `16`. The load balancer keeps 2^`REPSBufferSizeMaskBits` entropy values that ACKs have reported as uncongested, 8 at the Default, and reuses the oldest for the next data packet. After freezing, one data packet in every 2^`REPSBufferSizeMaskBits` explores a new value ([Freezing and exploration](./multipath.md#freezing-and-exploration)). |
| `EntropyValueSetSize` | `uint` | `65536` | `1` to `65536`. Exploring draws an entropy value at random from 0 to this value minus 1: 0 to 65,535 at the Default. `StartingPortOffset` does not apply. |
| `FreezingTimeout` | `timeval` | `100us` | Shortest time the load balancer stays in freezing after a timeout on a path, from the retransmission timer or early failure detection. It leaves freezing when it next recycles a value after this time, normally on an ACK without the ECN echo; while freezing it reuses the values it holds and, once it has cached one, does not explore ([Freezing and exploration](./multipath.md#freezing-and-exploration)). |

### ScalaUETCongestionControl

These attributes sit under `TransportProcessingLayers` → `ScalaUETCongestionControl` and apply to every congestion window on the NIC, one for each PDC group. [How each attribute shapes the window](./congestion-control.md#how-each-attribute-shapes-the-window) puts them together.

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `PassThrough` | `bool` | `false` | When `true`, the congestion window no longer limits sending: NSCC still computes the window, and the record columns that show it keep moving, but it holds back no packet. Sending is still bounded by the egress buffer and the PCIe link ([Turning congestion control off](./congestion-control.md#turning-congestion-control-off)). |
| `NSCCEnable` | `bool` | `true` | When `true`, NSCC adjusts each window on ACKs, trim NACKs, and retransmission timeouts. When `false`, the NIC runs none of the NSCC routines, Quick Adapt included: each window starts at its maximum, 450,000 bytes at the defaults, and still limits sending, and a lower measured base RTT still lowers the maximum, and the window with it. |
| `InitialBaseRTT` | `timeval` | `6us` | Base RTT each congestion window starts from. With the link rate it sets the bandwidth-delay product (BDP), 300,000 bytes at the defaults, and from it the maximum window, 1.5 × BDP. It also sets the target queue delay and the scale of the increase routines. A measured RTT lowers the base RTT when it beats the current base by about 1 µs, which re-derives the maximum window but not the target; the base RTT never rises ([Base RTT](./congestion-control.md#base-rtt)). |
| `ReferenceNetworkRTT` | `timeval` | `6us` | With `ReferenceNetworkLinkSpeed`, defines a reference BDP. The proportional and fair increase gains and the additive term grow with the NIC's BDP relative to the reference BDP, so raising either reference value makes them smaller. It does not change the target queue delay, the maximum window, or multiplicative decrease. |
| `ReferenceNetworkLinkSpeed` | `datarate` | `400Gbps` | Link speed of the reference BDP (see `ReferenceNetworkRTT`). Usually the NIC's own link rate; NICs of different link speeds can share one reference value, which sets how their gains scale relative to each other. |
| `OverrideSpecTargetQueueDelay` | `bool` | `false` | When `true`, the target queue delay is `TargetQueueDelay`. When `false`, it is `InitialBaseRTT` with `TrimmingEnabled` `false`, or 0.75 × `InitialBaseRTT` with it `true`: 6 µs or 4.5 µs at the defaults ([Target queue delay](./congestion-control.md#target-queue-delay)). |
| `TargetQueueDelay` | `timeval` | `4500ns` | Target queue delay when `OverrideSpecTargetQueueDelay` is `true`; not used otherwise. The Default equals the target the NIC derives with trimming on. |
| `TargetQueueDelayMargin` | `timeval` | `1us` | Queueing delay below which ACKs count toward fast increase: once a full window's worth of ACKs has arrived with queueing delay below this margin, the window grows faster, until an ACK whose queueing delay is below the target but at or above this margin, or an ECN-marked ACK whose queueing delay is at or above the target. It is used only by fast increase ([How the window responds to each ACK](./congestion-control.md#how-the-window-responds-to-each-ack)). |
| `DisableQuickAdapt` | `bool` | `false` | When `true`, Quick Adapt never runs ([Quick Adapt](./congestion-control.md#quick-adapt)). |
| `QuickAdaptGate` | `uint` | `3` | `0` to `31`. Quick Adapt cuts the window, to the bytes acknowledged in its last observation window, only when a trigger is present and those bytes are below the maximum window divided by 2^`QuickAdaptGate`: 450,000 ÷ 2^3 = 56,250 bytes at the defaults. An observation window lasts the base RTT plus the target queue delay, 12 µs at the defaults. |
| `QuickAdaptThreshold` | `timeval` | `24us` | With `TrimmingEnabled` `false`, the average queueing delay above which Quick Adapt is triggered. With `TrimmingEnabled` `true`, only trimmed packets trigger Quick Adapt and this is not used. The Default is 4 × the 6 µs target queue delay. |
| `OverrideUetSpecMaxCWind` | `bool` | `false` | When `true`, the initial maximum window is `MaxCongestionWindow` nominal packets instead of 1.5 × BDP. Once a lower base RTT is measured, the maximum is re-derived from it and this setting no longer applies. |
| `MaxCongestionWindow` | `uint` | `107` | `1` to `4096`. Initial maximum window in nominal packets of `RdmaDataMSS` + 88 bytes, 4,184 bytes at the default `RdmaDataMSS`; used only with `OverrideUetSpecMaxCWind` `true`. The Default is the derived maximum in whole packets: 450,000 ÷ 4,184 = 107.55, rounded down. 107 packets are 447,688 bytes. |
| `ReceiverCongestionLowThreshold` | `double` | `25.0` | `0.0` to `100.0`. Acts on the NIC as a receiver: the occupancy of its receive pool for data, in percent, below which the ACKs it sends carry no receiver penalty. Between this and `ReceiverCongestionHighThreshold` the penalty rises in proportion to occupancy, and the sender applies it to its window. At the defaults, 900,000 bytes of the 3,600,000-byte pool ([Receiver penalty](./congestion-control.md#receiver-penalty)). |
| `ReceiverCongestionHighThreshold` | `double` | `75.0` | `0.0` to `100.0`. Acts on the NIC as a receiver: the occupancy of its receive pool for data, in percent, at or above which the ACKs it sends carry the full receiver penalty. At the defaults, 2,700,000 bytes. At the default PCIe settings the host interface drains received data faster than the link fills the pool, so the penalty stays 0 in a default run. |
| `RestoreCWindEnable` | `bool` | `false` | Acts on the NIC as a receiver: when `true`, the ACKs it sends tell their senders to restore the window they had before the receiver penalty reduced it, once the penalty is back to 0. |
| `MaxMultiplicativeDecreaseJump` | `double` | `0.5` | `0.001` to `1.0`. Smallest fraction of the window one multiplicative decrease keeps: at `0.5`, one decrease at most halves the window. |
| `AlphaMultiplier` | `double` | `4.0` | `1.0` to `100.0`. Scales proportional increase, the growth on unmarked ACKs whose queueing delay is below the target: a larger value grows the window faster. |
| `FairIncreaseMultiplier` | `double` | `5.0` | `1.0` to `100.0`. Scales fair increase, the growth on unmarked ACKs whose queueing delay is at or above the target: a larger value grows the window faster. |
| `FastIncreaseMultiplier` | `double` | `1.0` | `0.01` to `100.0`. Scales fast increase, the growth once queueing delay has stayed below `TargetQueueDelayMargin` for a full window of ACKs: a larger value grows the window faster. |
| `EtaMultiplier` | `double` | `0.15` | `0.0` to `1.0`. Scales eta, the additive term the window update adds after the increase and decrease routines. |
| `Gamma` | `double` | `0.8` | `0.001` to `1.0`. Strength of multiplicative decrease. On an ECN-marked ACK whose queueing delay is at or above the target, while the average queueing delay is above it, at most once per base RTT, the window is multiplied by 1 − `Gamma` × (average delay − target) ÷ average delay, keeping at least `MaxMultiplicativeDecreaseJump` of itself. |
| `DelayAlpha` | `double` | `0.25` | `0.0` to `1.0`. Weight of each new queueing-delay sample in the average queueing delay: average = `DelayAlpha` × sample + (1 − `DelayAlpha`) × average. The average decides multiplicative decrease and, with trimming off, the Quick Adapt delay trigger. |

### SemanticLayer

This attribute sits under `TransportProcessingLayers` → `SemanticLayer`, the layer that cuts messages into packets ([Segmentation](./transport.md#segmentation)).

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `RdmaDataMSS` | `uint` | `4096` | Largest payload of one data packet; each data packet is `RdmaDataMSS` + 106 bytes on the wire, 4,202 bytes at the Default. A message is cut into packets of at most this payload. It also sets the nominal packet, `RdmaDataMSS` + 88 bytes, in which congestion control counts its window and `MaxCongestionWindow` ([Packets on the wire](./transport.md#packets-on-the-wire)). |

### ScalaUETMonitor

These attributes sit under `TransportProcessingLayers` → `ScalaUETMonitor` and control the per-packet `uet-monitor` record ([uet-monitor](./records.md#uet-monitor)). The monitor's settings take effect once per simulator process, from one UET NIC: give every UET NIC in a configuration the same monitor settings.

Each filter matches exactly, and every filter that is not `All` must match: the filters combine. Once a packet of a connection matches, or a trigger selects it, the monitor logs the rest of that connection's packets for the run, at both of its ends when both NICs run in the same simulator process.

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `EnableUETMonitor` | `bool` | `false` | When `true`, `uet-monitor` logs every UET packet sent or received by the PDCs the four filters select. PFC frames are not logged. |
| `FilterOnFepId` | `string` | `All` | `All`, or the FEP id of one NIC: its simulator node id, as in the `fep_id` column of `uet-transport-stats` ([Identifying a NIC and a PDC](./records.md#identifying-a-nic-and-a-pdc)). Matches the packets that NIC sends and receives. |
| `FilterOnLocalIp` | `string` | `All` | `All`, or an IPv4 address in dotted form, such as `32.0.0.2`: selects the packets of PDCs whose local end has that address. |
| `FilterOnRemoteIp` | `string` | `All` | `All`, or an IPv4 address in dotted form: selects the packets of PDCs whose remote end has that address. |
| `FilterOnPdc` | `string` | `All` | `All`, or one PDC id in the form `<IPv4>:<PDC id>`, for example `32.0.0.2:4500`, where the number after the colon is a PDC id, not a port: selects the packets of the PDC whose local or remote id it is. |
| `TriggerLoggingOnRttEnable` | `bool` | `false` | When `true`, a connection whose sender measures an RTT at or above `TriggerLoggingOnRttThreshold` is selected for logging, within the `TriggeredLoggingPDCLimit` budget, and logged for the rest of the run. |
| `TriggerLoggingOnRttThreshold` | `timeval` | `100us` | RTT at or above which `TriggerLoggingOnRttEnable` selects a connection. |
| `TriggerLoggingOnCWindPen` | `bool` | `false` | When `true`, a connection on which a receiver penalty at or above `TriggerLoggingOnCWindPenThreshold` is seen is selected for logging, within the `TriggeredLoggingPDCLimit` budget, and logged for the rest of the run. At the default PCIe settings the penalty stays 0, so this trigger does not fire in a default run. |
| `TriggerLoggingOnCWindPenThreshold` | `uint` | `127` | `0` to `127`. Receiver penalty at or above which `TriggerLoggingOnCWindPen` selects a connection; `127` is the full penalty. |
| `TriggeredLoggingPDCLimit` | `uint` | `1` | `0` to `100`. The most connections the two triggers together can select in one simulator process. |

## Host interface

The `HostInterface` component holds the PCIe link between the NIC and its host, as two sub-components: `NICSide`, the NIC's end of the link, and `HostSide`, the host's end. [PCIe host interface](./buffers-and-pfc.md#pcie-host-interface) explains how a transfer uses them, and [Worked example: PCIe and network rates](./buffers-and-pfc.md#worked-example-pcie-and-network-rates) compares the link's rates at the defaults with the network port's.

Each end holds a container named `PCIeGen3.0`. The name does not set the PCIe generation: each end's `LaneBps` does, and the default 31.52 Gbps per lane is the PCIe 5.0 rate.

### NIC side

On the NIC's end of the link, each attribute's path in the NIC's `typedParameters` is:

- `RXHeaderBufferSize`, `TXHeaderBufferSize`, `RXDataBufferSize`, `TXDataBufferSize`: `HostInterface` → `NICSide` → `PCIeGen3.0` → `NicDeviceLayer` → `NicPCIeInterface` → `TransactionLayer`
- `LaneBps`, `LaneCount`: `HostInterface` → `NICSide` → `PCIeGen3.0` → `NicDeviceLayer` → `NicPCIeInterface` → `PhysicalLayer`

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `RXHeaderBufferSize` | `queuesize` | `16KB` | Size of the transaction-layer buffers for the headers of transfers the NIC receives from the host. |
| `TXHeaderBufferSize` | `queuesize` | `16KB` | Size of the transaction-layer buffers for the headers of transfers the NIC sends to the host. |
| `RXDataBufferSize` | `queuesize` | `256KB` | Size of transaction-layer buffers for transfer payload on the NIC's end of the link. `RXDataBufferSize` and `TXDataBufferSize` together size the payload buffers in both directions. |
| `TXDataBufferSize` | `queuesize` | `256KB` | Size of transaction-layer buffers for transfer payload on the NIC's end of the link. `RXDataBufferSize` and `TXDataBufferSize` together size the payload buffers in both directions. |
| `LaneBps` | `datarate` | `31.52Gbps` | Rate of one PCIe lane in the NIC-to-host direction. Any rate can be set, so the link can represent any PCIe generation. |
| `LaneCount` | `uint` | `16` | `1` to `16`. Number of PCIe lanes. The NIC-to-host direction runs at `LaneCount` × `LaneBps`. The NIC also uses that rate to estimate how long a received packet waits to reach the host, and holds the packet's ACK for that time ([Acknowledgements](./transport.md#acknowledgements)). |

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
| `LaneCount` | `uint` | `16` | `1` to `16`. Number of PCIe lanes. The host-to-NIC direction runs at `LaneCount` × `LaneBps`. |

## Setting attributes through the platform

Attributes are set per configuration, through the platform API. The reference pages for the two APIs involved are [Components](../../api-reference/components.md) and [Configurations](../../api-reference/configurations.md). Get an API token first ([Authentication](../../authentication.md)) and export it as `SCALA_API_TOKEN`.

### Read the NIC's parameters

`GET /api/v1/components` lists the component types the platform offers; the Scala UET NIC entry, `ScalaUETNIC`, gives its `id` and its `versions`. `GET /api/v1/components/{component_id}`, with the `version` query parameter set to the model version these pages describe, returns the NIC's `typedParameters`. Each attribute in the tables above appears there as an object with its `value`, its `type`, a `unit` for dimensional types, and, where the model sets them, its `default`, `description`, `min`, `max`, and allowed values in `enum`; `NumDownLinks` also carries `readonly`. A sub-component, such as `NetworkInterface` or `TransportProcessingLayers`, is a nested object named after it.

For example:

```bash
curl -X GET "https://api.scalacomputing.com/api/v1/components" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"

# Set COMPONENT_ID to the id of the Scala UET NIC entry in that response.
export COMPONENT_ID="model_..."

curl -X GET "https://api.scalacomputing.com/api/v1/components/$COMPONENT_ID?version=4.6.0" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

### Add the NIC to a server

A Scala UET NIC belongs to a server component: every instance of that server carries one NIC with the same `typedParameters`. Placing it takes two calls:

1. `POST /api/v1/configurations/{config_id}/components`, with the query parameters `modelId=ScalaUETNIC`, `type=nic`, a `name` for the NIC component, and `version`, adds the NIC to the configuration with every attribute at its Default.
2. `PATCH /api/v1/configurations/{config_id}/configurations/{component_name}`, where `component_name` is the server component, adds the NIC to that server. The body names the NIC component, with `type` `nic` and `count` 1.

Set `CONFIG_ID` to the configuration's ID and `SERVER_NAME` to the `name` of the server component, which must already be in the configuration; `GET /api/v1/configurations/{config_id}/components` lists the configuration's components and their names. `modelId` takes the model name, `ScalaUETNIC`, or its `id` from `GET /api/v1/components`.

```bash
curl -X POST "https://api.scalacomputing.com/api/v1/configurations/$CONFIG_ID/components?modelId=ScalaUETNIC&type=nic&name=Nic&version=4.6.0" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"

curl -X PATCH "https://api.scalacomputing.com/api/v1/configurations/$CONFIG_ID/configurations/$SERVER_NAME" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "name": "Nic", "type": "nic", "count": 1 }'
```

Servers that need different NIC settings need different NIC components, each added to its own server component. The applications that run over the NIC are the [Chakra workload](../chakra-workload/index.md) and the Scala RDMA application.

### Change values in a configuration

`PATCH /api/v1/configurations/{config_id}/components/{component_name}/parameters` merges a patch into the `typedParameters` of the component named `component_name` in the configuration; `GET /api/v1/configurations/{config_id}/components` lists the configuration's components and their names. Set `CONFIG_ID` to the configuration's ID and `NIC_NAME` to the `name` of the NIC component you are changing. A patch changes only the component it names, and so every server that carries that NIC component.

The patch repeats the nesting of the component's `typedParameters`, down to each attribute you change, as nested objects, one per name in the attribute's path:

- Only keys the component already has can be updated. An unknown key returns 400.
- Each attribute's `type` must match the stored type, or the request returns 400. The request also returns 400 for an empty patch, or when a nested object is sent where an attribute is expected, or the reverse.
- The `value` and the `unit` you send replace the stored ones, so send the `unit` with every dimensional value.
- A list attribute's `value` is a string with its square brackets and exactly eight entries, each made of letters, digits, `.`, `_`, `+`, or `-`, for example `"value": "[0.9, 0.1, 0, 0, 0, 0, 0, 0]"`. A list in any other form returns 400.
- Where an attribute has `enum` values, its `value` must be one of them, or the request returns 400. Send a value that contains spaces as it is, for example `"value": "Recycled Entropy Packet Spraying"`.
- Any other string `value`, such as a monitor filter, cannot contain shell metacharacters, among them `[` `]` `{` `}` `(` `)` `'` `"` `;` `$` `*` `?` `#` `~` `!` `&` `<` `>`, or the request returns 400. Spaces and `.`, `:`, `-`, and `_` are allowed, so an IPv4 address or a PDC id such as `32.0.0.2:4500` can be sent as it is.
- A 200 response returns the component's full, updated `typedParameters`.

This example turns on recycled entropy packet spraying and lengthens its freezing time to 200 µs:

```bash
curl -X PATCH "https://api.scalacomputing.com/api/v1/configurations/$CONFIG_ID/components/$NIC_NAME/parameters" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "TransportProcessingLayers": {
      "ScalaUETPdsManager": {
        "PacketSprayingType": { "value": "Recycled Entropy Packet Spraying", "type": "string" }
      },
      "PacketSpraying": {
        "RecycledEntropyPacketSpraying": {
          "FreezingTimeout": { "value": 200, "unit": "us", "type": "timeval" }
        }
      }
    }
  }'
```

A NIC the platform added carries every attribute at its Default, so a patch only needs the attributes you change. For a NIC whose `typedParameters` you supplied yourself, see [How to read the tables](#how-to-read-the-tables).

Global simulation parameters, such as `NDStatsReportInterval`, `EnableNetworkBufferStatsReporting`, and `EnablePCIeStatsLogging`, are not attributes of this model; change them with `PATCH /api/v1/configurations/{config_id}/parameters`.

## Configuration rules that stop a simulation

A configuration write returns 400 when a value breaks the rules in [Change values in a configuration](#change-values-in-a-configuration), when a dimensional value has a unit its type does not accept, when a `uint` is negative or fractional, when a number is outside the range its row gives, or when a `bool` is neither `true` nor `false`. `PrefetchBufferSize` below `4202` is refused this way even with the prefetch buffer off. A monitor filter that contains a character the platform refuses in string values also returns 400.

`POST /api/v1/configurations/{config_id}/validate` returns the configuration's validation errors and warnings. A configuration with a Scala UET NIC is `invalid` when:

- A server with a Scala UET NIC runs an application other than the Chakra workload or the Scala RDMA application.
- A switch has `UETPolicyEnabled` set to `true` and the configuration places a NIC other than the Scala UET NIC anywhere in the topology. Switch packet trimming also needs the UET policy, and PFC off for every class on that switch ([Configuration rules that stop a simulation](../scala-switch/configuration.md#configuration-rules-that-stop-a-simulation)).
- `LaneCount` is outside `1` to `16`.

The validation also warns, without making the configuration `invalid`, when a NIC's `PacketSprayingType` is `Recycled Entropy Packet Spraying` and a switch's `LoadBalancingMethod` is `flowlet`.

The NIC stops the run before the simulation starts when:

- An ingress or egress `PoolAllocationVector` has fewer than two non-zero entries. Classes 0 and 1 each need a pool ([Traffic classes](./buffers-and-pfc.md#traffic-classes)).
- `PrefetchBufferEnable` is `true` and `PrefetchBufferSize` is smaller than one data packet on the wire, `RdmaDataMSS` + 106 bytes.
- `QueueSchedulingType` is `ProbabilisticWeightedRoundRobin` in the `IngressBufferManager` or the `EgressBufferManager`.

The NIC checks its congestion control settings when it opens its first connection, as a sender or a receiver, and stops the run when:

- The target queue delay resolves to 0: `InitialBaseRTT` is `0` while `OverrideSpecTargetQueueDelay` is `false`, or `TargetQueueDelay` is `0` while it is `true`.
- `OverrideUetSpecMaxCWind` is `false` and 1.5 × the link rate × `InitialBaseRTT` ÷ 8 is smaller than one nominal packet, `RdmaDataMSS` + 88 bytes: `InitialBaseRTT` below about 56 ns at 400 Gbps.
- 1.5 × the link rate × `InitialBaseRTT` ÷ 8 is larger than 4,294,967,295 bytes: `InitialBaseRTT` above about 57.3 ms at 400 Gbps, whatever the override.
- `ReferenceNetworkRTT` is `0`, or `ReferenceNetworkLinkSpeed` is below 1 Gbps.
- `ReferenceNetworkLinkSpeed` × `ReferenceNetworkRTT` ÷ 8 is larger than 4,294,967,295 bytes: `ReferenceNetworkRTT` above about 85.9 ms at 400 Gbps.
- The NIC's link rate, the lower of its `DataRate` and the rack switch's downlink `DataRate`, is below 1 Gbps.
