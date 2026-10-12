---
title: "Buffers and PCIe"
description: "The Scala MRC NIC holds what it sends and receives in two buffers, each split into data and control pools, and exchanges payload with its host over PCIe."
---

The Scala MRC NIC holds what it sends and receives in two buffers, each split into data and control pools, and exchanges payload with its host over PCIe.

This page covers the NIC's transmit and receive buffers, the order in which it sends, and the PCIe link to its host that feeds the transmit buffer and drains the receive buffer. [Transport](./transport.md) follows a message through the rest of the NIC, [Congestion control](./congestion-control.md) explains the window that limits sending alongside the buffers, and [Configuration](./configuration.md#ingressbuffermanager) lists each attribute's type and default.

## Overview

The NIC has two buffers, each divided into two pools, one for data and one for control packets:

- **The egress buffer** holds what the NIC sends. Space for each data packet is reserved before its payload is fetched from the host, and released when the packet has been serialized onto the link, so the egress data pool bounds the data the NIC has fetched and not yet sent. Its size is the `EgressBufferManager` `TotalBufferSize`, 256 KB by default.
- **The ingress buffer** holds what the NIC has received and not yet handed on: data until its write to host memory is accepted by the PCIe link, control packets until the NIC has processed them. Its size is the `IngressBufferManager` `TotalBufferSize`, 4 MB by default.

The two sizes are the buffers' only attributes. The split of each buffer into its pools, and the order in which the classes are sent, are fixed.

```mermaid
flowchart LR
  H1["Host memory"] -->|"payload fetched over PCIe"| E["Egress buffer: data pool and control pool"]
  E -->|"control first, then data"| P["Network port and link to the rack switch"]
  P --> I["Ingress buffer: data pool and control pool"]
  I -->|"received data written over PCIe"| H2["Host memory"]
```

Three behaviors are worth knowing before anything else:

- **The egress buffer never drops data.** When the next data packet cannot reserve space, the NIC stops generating data until the pool drains, so a full transmit pool holds the sender back rather than losing data ([Egress buffer](#egress-buffer)).
- **The ingress buffer drops what does not fit.** A received packet whose pool cannot hold it is dropped, and the sender of a dropped data packet recovers it as a lost packet. The NIC sends no pause frames to hold the rack switch back ([Pause frames](#pause-frames)).
- **At the default PCIe settings, the ingress buffer stays nearly empty.** The PCIe link writes received data to the host faster than the 400 Gbps network link delivers it, so the data pool holds about one packet at a time ([PCIe host interface](#pcie-host-interface)).

## Traffic classes

The NIC has two traffic classes, data and control. It takes each packet's class from the role of the packet's DSCP codepoint ([Packet marking](./transport.md#packet-marking)), on transmit and on receive alike:

| Class | Codepoint roles | Packets |
| --- | --- | --- |
| Control | Control, trimmed, trimmed on the last hop | Cumulative ACKs, SACKs (including the responses to probes), NACKs, and data packets a switch has trimmed |
| Data | No-trim, trimmable, trimmable retransmission | Data packets, retransmissions included, and probes |

No attribute changes a packet's class: `TrimmingEnabled` and `UseDistinctDSCPForRetransmit` choose among the data roles, and every data role uses the data class. A packet uses the same class in the egress buffer of the NIC that sends it and in the ingress buffer of the NIC that receives it, except a data packet that a switch has trimmed, which arrives with a trimmed codepoint and is charged to the receiving NIC's control class. Probes travel in the data class, so they wait behind the data already queued ([Path feedback and probes](./multipath.md#path-feedback-and-probes)).

The class decides only which pool a packet uses inside the NIC and when the NIC sends it ([Transmit scheduling](#transmit-scheduling)). A switch classifies the same codepoints in its own way under its UET policy ([Buffer under the UET policy](../scala-switch/shared-buffer.md#buffer-under-the-uet-policy)).

## Buffer pools

Each buffer is divided into its two pools when the simulation starts, in the same fixed way for both:

```text
data pool      = 0.9 × TotalBufferSize, rounded down to a whole byte
control pool   = TotalBufferSize − data pool
```

No attribute changes the split. Each pool serves only its own class: a packet that does not fit in its class's pool cannot use free space in the other pool. Every byte a packet holds is charged to the pool of its class.

Sizes are given in `KB` of 1,000 bytes and `MB` of 1,000,000 bytes, so the default `256KB` and `4MB` are 256,000 and 4,000,000 bytes:

| Buffer | `TotalBufferSize` | Data pool | Control pool |
| --- | --- | --- | --- |
| Egress | `256KB`, 256,000 B | 230,400 B | 25,600 B |
| Ingress | `4MB`, 4,000,000 B | 3,600,000 B | 400,000 B |

## Egress buffer

Each data packet reserves its size on the wire in the egress data pool: its payload plus 78 bytes, 4,174 bytes for a full packet at the default `MSS` of 4096, or its payload plus 82 bytes with `RTTTimestampMode` set to `WireTimestamp` ([Packets on the wire](./transport.md#packets-on-the-wire)). When the reservation is made depends on the packet:

- **A new data packet** reserves its space before its payload fetch over PCIe is issued, before the payload has arrived from the host.
- **A retransmission** reserves its space when it is replayed. Its payload is not fetched again ([Fetching the payload over PCIe](./transport.md#fetching-the-payload-over-pcie)).
- **Cumulative ACKs, SACKs, and NACKs** reserve their size on the wire in the control pool when they are built. **Probes** reserve theirs in the data pool.

Every reservation is released when the packet's serialization onto the link ends.

- **When the data pool has room,** the data packet goes ahead: its payload is fetched, or the retransmission joins the queue for the wire.
- **When the data pool has no room,** the NIC stops generating data packets. It tries again 15 ns after the pool drains from above half its size to half or less, and at once whenever a cumulative ACK, SACK, or NACK arrives or a retransmission timer expires.
- **Data is never dropped here.** A full pool holds the sender back rather than losing data.

The egress data pool is therefore the most data the NIC has fetched from the host, or replayed for retransmission, and not yet sent: 55 full packets at the defaults ([Worked example: the default configuration](#worked-example-the-default-configuration)). A larger pool lets more data wait in the NIC; a smaller one makes the NIC fetch closer to the moment it sends. The congestion window bounds the same data together with the data already sent and not yet acknowledged ([The congestion window](./congestion-control.md#the-congestion-window)).

## Transmit scheduling

The NIC's network port sends one frame at a time. Each time it can send, it takes the next frame in strict priority:

1. **The control class first.** A waiting cumulative ACK, SACK, or NACK always goes before waiting data.
2. **Then the data class,** whose data packets, retransmissions, and probes leave in the order they joined the queue.

Within each class, frames leave first in, first out. A frame already being serialized is never interrupted, so a control packet waits only for the frame on the link and for the control frames queued ahead of it. No attribute changes this order.

The receiver counts the control frames queued ahead of a SACK in the service time it reports in that SACK, so the sender's RTT sample leaves that wait out ([Base RTT](./congestion-control.md#base-rtt)).

## Receive buffer

Each packet the NIC receives is charged to the ingress pool of its class at its full size on the wire, Ethernet header and frame check sequence included: 4,174 bytes for a full data packet at the default `MSS`. A frame shorter than the 64-byte Ethernet minimum is charged 64 bytes, so a cumulative ACK or a trimmed data packet, 62 bytes on the wire, takes 64. A packet that arrives when its pool cannot hold it is dropped and counted in the `nd-stats` `rx_drops` column. It never reaches the transport, so it is not counted in `mrc-transport-stats` and not logged by `mrc-monitor` ([Records](#records)).

How long a packet stays charged depends on what it is:

- **Data** stays charged until the NIC has handed it to the host interface, that is, until the PCIe transmit buffer accepts the write of its payload to host memory ([Receiving a packet](./transport.md#receiving-a-packet)). Only the payload is written; the headers are not.
- **Cumulative ACKs, SACKs, NACKs, and probes** are released as soon as the NIC has processed them.
- **Trimmed data packets** are released as soon as the NIC has recorded the trim.
- **Data the NIC discards after admitting it,** such as a packet beyond the receiver's PSN window, is released when it is discarded ([NACKs](./transport.md#nacks)).

Received data packets are written to the host in the order they arrive. The NIC sends the acknowledgements a data packet draws when the packet arrives, before its payload is written to the host ([Selective acknowledgements](./transport.md#selective-acknowledgements)), and the service time a SACK reports covers the NIC's control-class transmit queue, not its PCIe link ([Base RTT](./congestion-control.md#base-rtt)).

The occupancy of the data pool is therefore the received data the NIC holds for the host. It grows only when the network delivers faster than the PCIe link writes, and at the default PCIe settings it stays near empty. When it does fill, arriving data packets are dropped, and their senders recover them as lost packets, through SACK-gap loss detection or the retransmission timer ([Loss detection from SACK gaps](./transport.md#loss-detection-from-sack-gaps), [Worked example: PCIe and network rates](#worked-example-pcie-and-network-rates)).

## Pause frames

The NIC does not implement priority flow control (PFC). It sends no pause frames, so a filling receive pool never holds the rack switch back, and it does not pause for the pause frames it receives, so the rack switch cannot hold back the NIC's transmission. A receive pool that cannot hold an arriving packet drops it ([Receive buffer](#receive-buffer)).

Packet trimming on a switch, which produces the trim NACKs the NIC's congestion control reacts to, runs only with the switch's UET policy on and PFC off on every traffic class ([Packet trimmer](../scala-switch/configuration.md#packet-trimmer)).

## PCIe host interface

The NIC and its host are joined by a PCIe link, with a protocol stack at each end: lanes, and transaction-layer buffers on the NIC side and on the host side ([Host interface](./configuration.md#host-interface)). The link carries the NIC's payload fetches for the data it sends ([Fetching the payload over PCIe](./transport.md#fetching-the-payload-over-pcie)) and the writes that deliver received data to host memory.

**Link rate.** Each direction runs at the sending side's `LaneCount` × `LaneBps`, and the two directions run independently. The NIC side's lanes carry the NIC-to-host direction: the read requests for payload the NIC sends, and the writes of the data it receives. The host side's lanes carry the host-to-NIC direction: the payload the NIC fetches. The defaults, 31.52 Gbps per lane on 16 lanes at both ends, give 504.32 Gbps in each direction; 31.52 Gbps is the PCIe 5.0 lane rate. The lane rate alone sets the generation the link represents: the `PCIeGen3.0` container that holds these attributes in the configuration does not.

**Transfers.** PCIe moves data in transfers of at most 512 bytes of payload, and each transfer adds 34 bytes of PCIe protocol overhead on the link. The host answers each read and accepts each write at once: host memory latency is not modeled, and the PCIe link has no propagation delay, so what a transfer costs is its time on the link.

**Payload fetch.** For each new segment, the NIC sends one read request of `ReadRequestSize` bytes, 64 by default, on the NIC-to-host direction, and the host answers with the segment's payload on the host-to-NIC direction. `ReadRequestSize` sets the size of the request, not of the payload fetched; a request larger than 512 bytes takes several transfers. Retransmissions do not use the link: the NIC rebuilds a retransmitted packet itself.

**Buffers.** Each end of the link has transaction-layer buffers for transfer headers and for payload, on its transmit side and its receive side, sized by `RXHeaderBufferSize`, `TXHeaderBufferSize`, `RXDataBufferSize`, and `TXDataBufferSize`. A transfer waits in its transmit buffer until it can go on the link, and is sent only when the receiving end has buffer space for it. On the NIC side, received data is released from the NIC's ingress pool as soon as its write is accepted into the PCIe transmit buffer ([Receive buffer](#receive-buffer)), so the PCIe buffers set how much received data can wait for the link outside the NIC's ingress pools.

### Worked example: PCIe and network rates

At the defaults, `MSS` 4096, 16 lanes of 31.52 Gbps, and a 400 Gbps network link:

```text
PCIe link, each direction         16 × 31.52 Gbps                     = 504.32 Gbps
4,096 B payload on the link       4,096 B ÷ 512 B = 8 transfers
                                  8 × (512 B + 34 B)                  =  4,368 B
time on the link                  4,368 B × 8 ÷ 504.32 Gbps           =  69.29 ns
PCIe payload rate                 504.32 Gbps × 4,096 ÷ 4,368         =  472.9 Gbps
network payload rate              400 Gbps × 4,096 ÷ 4,174            =  392.5 Gbps
```

The 4,174 bytes are the data packet's size on the wire ([Packets on the wire](./transport.md#packets-on-the-wire)). Both directions of the PCIe link carry payload faster than the network link can, so at the defaults the network link, not PCIe, limits a transfer. On the receive side, each packet's write is on the PCIe link for 69.29 ns, while the next packet takes 83.48 ns to arrive on the network link. Arrivals can never exceed the NIC's own link rate, so at the defaults the ingress data pool holds about one packet at most, even when many senders converge on the NIC.

With fewer NIC-side lanes, the NIC-to-host direction can fall behind the network:

```text
needed to keep up with received data   4,368 B × 8 ÷ 83.48 ns    = 418.6 Gbps
14 lanes × 31.52 Gbps                                             = 441.28 Gbps
13 lanes × 31.52 Gbps                                             = 409.76 Gbps
```

With 14 lanes or more at 31.52 Gbps, the NIC-to-host direction keeps up with full-rate receiving, even while the NIC sends at full rate too and its read requests share that direction. With 13 lanes or fewer, data received at full rate backs up into the ingress data pool. The pool fills, then drops arriving data packets, counted in `rx_drops`, and their senders recover them through SACK-gap loss detection or the retransmission timer ([Receive buffer](#receive-buffer)).

Fewer host-side lanes slow the payload fetch instead, which limits how fast the NIC sends.

## Worked example: the default configuration

A NIC with every attribute at its default: link rate 400 Gbps, `MSS` 4096.

**Egress pools.**

```text
EgressBufferManager TotalBufferSize            256,000 B   (256KB)
data pool      0.9 × 256,000 B                 230,400 B   = 55 data packets of 4,174 B (55 × 4,174 = 229,570 B)
control pool   256,000 B − 230,400 B            25,600 B   = 272 SACKs of 94 B
data pool restart point, half the pool         115,200 B
```

As many as 55 full-size data packets can be fetched and waiting to be sent at once. That is 55 × 83.48 ns = 4.59 µs of sending on the network link, against about 70.8 ns for one payload fetch at the default PCIe settings (a 64-byte read request plus 34 bytes of overhead on the NIC-to-host direction, 1.55 ns, then the 4,368-byte completion on the host-to-NIC direction, 69.29 ns), so at the defaults the egress buffer does not limit how fast one NIC sends. The congestion window, 450,000 bytes at the start, allows more than the data pool holds; the pool bounds only the part not yet sent ([Worked example: the defaults](./congestion-control.md#worked-example-the-defaults)).

**Ingress pools.**

```text
IngressBufferManager TotalBufferSize         4,000,000 B   (4MB)
data pool      0.9 × 4,000,000 B             3,600,000 B   = 862 data packets of 4,174 B
control pool   4,000,000 B − 3,600,000 B       400,000 B
```

At the default PCIe settings the data pool holds about one packet at a time ([Worked example: PCIe and network rates](#worked-example-pcie-and-network-rates)). It fills only when the NIC's host link writes received data more slowly than its network link delivers it.

## Records

These records show the buffers ([Records](./records.md)):

| Record | One row for | What it shows here |
| --- | --- | --- |
| `nd-stats` | The NIC's network port, each `NDStatsReportInterval` | Frames and bytes the port sent and received, with bytes counting the Ethernet header. `rx_drops` includes the packets dropped because their receive pool was full ([nd-stats](./records.md#nd-stats)). |
| `mrc-transport-stats` | Each Scala MRC NIC, each `TransportStatsReportInterval` | At the senders, `total_rtx_sack`, the retransmissions of packets presumed lost from SACK gaps, and `total_rto_events`, the retransmission timer expiries, include the recovery of packets a full receive pool dropped ([mrc-transport-stats](./records.md#mrc-transport-stats)). |

Both records are written by default. `NDStatsReportInterval` is a global simulation parameter, not an attribute of this model, and is changed with `PATCH /api/v1/configurations/{config_id}/parameters`. `TransportStatsReportInterval` is an attribute of the NIC ([NIC](./configuration.md#nic)).
