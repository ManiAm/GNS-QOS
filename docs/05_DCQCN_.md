
# Beyond Standard DCQCN

The [DCQCN feedback loop](05_DCQCN.md) is effective, but it has a fundamental limitation: the signal is binary. The standard CNP tells the sender exactly one thing — "you caused congestion" — with no information about how severe the congestion is, which switch is the bottleneck, or how deep the queue has grown. The sender reacts to every CNP with the same formula, regardless of whether the switch buffer is 10% full or 90% full. This can lead to over-correction (slashing the rate when a gentle tap would suffice) or under-correction (applying the same mild cut to both minor and severe congestion).

The evolution beyond standard DCQCN follows two threads:

- Richer congestion signals that give the sender real data about network conditions
- A programmable engine that lets the sender act on that data with custom logic


## Per-Hop Telemetry (INT and IFA)

The most ambitious answer to the binary signal problem is to embed telemetry directly into data packets. As each packet transits the network, every switch along the path stamps per-hop metadata — its own identity, the ingress and egress ports used, the instantaneous queue depth, and nanosecond-precision timestamps — into the packet itself. By the time the packet arrives at its destination, it carries a complete, ordered record of every switch it touched and the exact congestion state at each one.

This idea was first realized by **In-Network Telemetry (INT)**, developed by Barefoot Networks (now Intel) around 2015–2016 as part of the P4 programmable switch ecosystem. INT was specified by the P4.org Applications Working Group and designed for programmable Tofino ASICs.

**In-Band Flow Analytics (IFA)** is Broadcom's independent implementation of the same concept, designed for their fixed-function ASICs (Trident and Tomahawk). IFA was submitted to the IETF for industry-wide standardization and is documented as an Internet-Draft in the IP Performance Measurement (IPPM) working group ([draft-kumar-ippm-ifa](https://datatracker.ietf.org/doc/draft-kumar-ippm-ifa/)), now at version 2 (IFAv2).

IFA is a general-purpose telemetry standard — not specific to RDMA or RoCEv2. It can stamp telemetry into any IP packet that matches a configured policy. Its broader uses include network monitoring, path tracing, and SLA verification. However, congestion control for lossless RDMA fabrics is one of its most impactful applications, because the standard DCQCN feedback loop provides so little information to work with.

> INT and IFA are conceptually identical but not wire-compatible. They carry the same per-hop information using different header formats and target different switch silicon. For the purposes of congestion control, they are interchangeable: both provide the same queue depth, utilization, and timing data that a sender's rate-control algorithm needs.

### Packet Lifecycle

The following diagram illustrates how per-hop telemetry accumulates as a packet traverses the network fabric.

<img src="../pics/ifa.png" width="500"/>

The source server transmits a standard data packet (Step 1). As this packet moves through the network (Steps 2, 3, and 4), each telemetry-enabled switch recognizes the traffic, inserts its own metadata block into the packet, and forwards it. The packet builds a chronological, stacked record of its journey and the precise conditions at every node it passed through.

Upon reaching the destination server, the receiving NIC extracts the complete telemetry stack. The receiver embeds this multi-hop metadata into the returning ACK packet (Step 5). When the original sender receives this ACK, its congestion control engine uses the full path-level view to pinpoint the exact location and severity of any congestion, and adjusts the flow rate accordingly (Step 6).


### Header Format

Per-hop telemetry metadata is not stored in the IP header or any existing protocol field. Instead, each switch inserts a metadata header between the packet's transport header (e.g., UDP) and the original payload. Each hop appends its own metadata block to a growing stack, so the headers accumulate in chronological order as the packet traverses the fabric.

```
┌────────────────────────────┐
│  Ethernet Header           │
├────────────────────────────┤
│  IP Header                 │
├────────────────────────────┤
│  UDP / TCP Header          │
├────────────────────────────┤
│  Telemetry Header (fixed)  │  Protocol indicator, max hop count, current length
├────────────────────────────┤
│  Metadata — Hop 1          │  ◄── Inserted by first switch
├────────────────────────────┤
│  Metadata — Hop 2          │  ◄── Appended by second switch
├────────────────────────────┤
│  Metadata — Hop N          │  ◄── Appended by Nth switch
├────────────────────────────┤
│  Original Payload          │
└────────────────────────────┘
```

The fixed header at the top of the stack contains bookkeeping fields: a protocol identifier, the maximum number of hops allowed, and the current metadata length (updated by each switch as it appends). Below it, each per-hop metadata block carries the following fields:

| Field              | Description                                                                |
|--------------------|----------------------------------------------------------------------------|
| **Switch ID**      | Unique identifier of the switch that inserted this metadata block.         |
| **Ingress Port**   | The physical port where the packet entered this switch.                    |
| **Egress Port**    | The physical port where the packet left this switch.                       |
| **Queue Depth**    | The instantaneous occupancy of the egress queue at the moment of transit.  |
| **Ingress Timestamp** | Nanosecond-precision time when the packet arrived at this switch.       |
| **Egress Timestamp**  | Nanosecond-precision time when the packet departed this switch.         |
| **Queue ID**       | The specific queue (Traffic Class) the packet was placed into.             |

Because each switch appends a fixed-size metadata block, the total overhead grows linearly with hop count. In a three-hop leaf-spine fabric, the packet accumulates three metadata blocks; in a five-hop fat-tree, it accumulates five. This overhead is negligible for large data transfers but becomes significant for small RDMA messages — a problem addressed by HPCC++ below.


## HPCC — Direct Rate Control from Telemetry

**HPCC (High Precision Congestion Control)**, introduced by Alibaba (Li et al., SIGCOMM 2019), is the most prominent algorithm built on per-hop telemetry. It was the first to prove that a sender with real network measurements can dramatically outperform DCQCN's indirect estimation.

Standard DCQCN's rate adjustment is inherently indirect. The sender never learns the actual state of the network. It receives a binary CNP, updates an estimated severity score ($\alpha$), and applies a multiplicative formula. The resulting rate is a guess — an educated one, but a guess nonetheless. If the congestion is mild, the sender may over-correct and waste bandwidth. If the congestion is severe, the sender may under-correct and trigger PFC.

HPCC eliminates this guesswork. It requires every switch on the path to stamp per-hop telemetry (INT in the original paper) into each data packet. By the time the packet reaches the receiver, it carries the actual link utilization and queue depth recorded at every hop. The receiver reflects this telemetry back to the sender, and the sender computes its new rate directly from the measurements.

The algorithm is straightforward: the sender scans the per-hop telemetry, finds the hop with the highest utilization, and sets its transmission rate to drive that bottleneck link toward a target utilization (typically 95%). There is no alpha, no phased recovery, no additive or hyper-additive increase. The rate is calculated from a single formula applied to real data:

$$Rate_{new} = \frac{Rate_{current} \times U_{target}}{U_{bottleneck}}$$

Where $U_{target}$ is the desired link utilization (e.g., 95%) and $U_{bottleneck}$ is the measured utilization at the most loaded hop. If the bottleneck link is running at 100% utilization and the target is 95%, the sender cuts its rate by exactly 5%. If the bottleneck is at 60%, the sender increases its rate. The adjustment is always proportional to the actual gap, never a blind multiplicative slash.


## HPCC++ — Solving the Overhead Problem

HPCC's precision comes at a cost. Each switch along the path appends its own metadata block to the packet. In a multi-tier leaf-spine fabric where a packet traverses five hops, the cumulative telemetry header can grow to tens of bytes per packet. For large data transfers, this overhead is negligible. For small RoCEv2 messages (common in collective operations like all-reduce), it becomes significant.

**HPCC++** (also called HPCC-PINT) addresses this by replacing full per-hop headers with **Probabilistic In-band Network Telemetry (PINT)**. Instead of every switch appending a complete metadata block, the telemetry is encoded probabilistically into a single, fixed-size field regardless of hop count. Each switch overwrites the same field with its own data at a configured probability. Over multiple packets, the sender reconstructs a statistically accurate picture of per-hop congestion from many small samples rather than one large header. This compresses the overhead to a constant size (typically 1–2 bytes) while preserving HPCC's core advantage: rate decisions driven by real measurements.

| Algorithm     | Signal                         | Rate Calculation                                    | Overhead             |
|---------------|--------------------------------|-----------------------------------------------------|----------------------|
| **DCQCN**     | Binary ECN / CNP               | Indirect (alpha estimation + multiplicative cut)    | None (ECN is 2 bits) |
| **HPCC**      | Full per-hop INT/IFA telemetry | Direct (proportional to bottleneck utilization)     | Grows with hop count |
| **HPCC++**    | Probabilistic (PINT)           | Direct (same formula, sampled data)                 | Fixed (1–2 bytes)    |


## Lighter Signals (eCNP and CSIG)

Full per-hop telemetry provides maximum visibility but is not always necessary or practical. Some deployments need a congestion signal that is richer than binary ECN but lighter than a growing telemetry stack. Two lighter alternatives fill this gap.

**Extended CNP (eCNP)** enriches the feedback message rather than the data packet. In standard DCQCN, the switch's role at the Congestion Point ends the moment it flips the ECN bits to CE — the switch has access to rich, real-time information (queue depth, port, timestamps) but none of it reaches the sender. With eCNP, the switch embeds a subset of this telemetry directly into the packet as it marks it. When the receiving NIC detects the CE mark and generates the feedback message, it copies this embedded telemetry into the CNP. The sender's congestion control engine now receives not just "you caused congestion" but quantitative data like "the egress queue on switch X was 73% full when your packet passed through." This transforms the feedback from a binary alarm into a graduated measurement, with zero overhead on the data-plane path (the telemetry rides in the feedback packet, not the data packet).

**Congestion Signal (CSIG)** compresses the congestion indicator even further. CSIG is NVIDIA's proprietary mechanism for Spectrum-4 switches and ConnectX-8 NICs. It encodes a compact, fixed-size congestion indicator — enough for the NIC's rate-control engine to make proportional adjustments — with negligible bandwidth cost. Where full IFA/INT provides a detailed per-hop map of the entire path, CSIG provides just enough information for informed rate decisions at near-zero overhead.

These lighter signals trade visibility for efficiency. An operator chooses along a spectrum based on workload requirements:

| Signal           | Overhead             | Visibility                              | Best For                                                 |
|------------------|----------------------|-----------------------------------------|----------------------------------------------------------|
| **IFA / INT**    | Grows with hop count | Full per-hop path telemetry             | Maximum visibility for complex multi-tier fabrics        |
| **Standard ECN** | None (2 bits)        | Binary — congested or not               | Baseline DCQCN deployments                               |
| **eCNP**         | None on data path    | Single-switch queue depth in feedback   | Deployments wanting richer feedback without per-hop cost |
| **CSIG**         | Fixed, minimal       | Compact congestion severity             | Line-rate workloads where overhead matters               |

## Programmable Congestion Control (PCC)

Richer signals — whether full per-hop telemetry, eCNP, or CSIG — are only useful if the sender can act on them intelligently. Standard DCQCN's Reaction Point cannot. The entire state machine (the alpha update, the three recovery phases, the clamp behavior) is burned into the NIC firmware. Network engineers can tune the parameters (`RPG_GD`, `AI`, `HAI`, `G`, etc.), but the algorithm itself cannot be changed. It has no concept of "queue depth" or "per-hop telemetry." It understands one input (CNP received: yes or no) and runs one fixed algorithm.

**Programmable Congestion Control (PCC)** solves this by replacing the fixed Reaction Point with a programmable engine running on the NIC itself. PCC is a framework provided by NVIDIA's DOCA SDK that allows engineers to write custom congestion control algorithms in C, compile them, and deploy them onto the embedded processing cores of ConnectX NICs (ConnectX-7 and later). These cores sit on the NIC's data path — they execute at line rate without involving the host CPU, preserving the zero-kernel-bypass property that makes RDMA fast.

With PCC, the Reaction Point is no longer a black box with tunable knobs. It becomes a blank slate. The NIC exposes congestion events (standard CNPs, eCNPs, CSIG, IFA telemetry) to the custom algorithm as structured inputs. The algorithm processes these inputs however it sees fit and produces a single output: the new transmission rate. The NIC's hardware rate limiter then enforces that rate, exactly as it would enforce a rate computed by the standard DCQCN state machine.

This means engineers can implement fundamentally different strategies depending on the workload:

- A large-scale all-reduce training job might use IFA telemetry to identify the single most congested switch in a multi-hop path and surgically reduce the rate just enough to relieve that specific bottleneck, rather than applying a blanket multiplicative cut.

- A latency-sensitive inference workload might react to eCNP queue-depth data with a proportional controller (deeper queue = harder brake), replacing the indirect alpha-based estimation with a direct measurement.

- A mixed workload might combine multiple signals — using IFA for path-level awareness and CSIG for lightweight flow-level feedback — and weight them differently depending on the traffic class.

PCC does not change the three-role model (CP, NP, RP) that DCQCN established. The Congestion Point is still the switch marking packets. The Notification Point is still the receiver generating feedback. The Reaction Point is still the sender adjusting its rate. What changes is the *implementation* of the Reaction Point: from a fixed state machine with tunable parameters to a fully programmable algorithm with structured telemetry inputs.

| Component         | Standard DCQCN                                       | Next-Generation (PCC + Telemetry)                             |
|-------------------|------------------------------------------------------|---------------------------------------------------------------|
| **CP (Switch)**   | Marks ECN CE bit                                     | Marks ECN CE bit + embeds IFA/eCNP/CSIG telemetry             |
| **NP (Receiver)** | Generates minimal CNP                                | Generates eCNP with telemetry, or forwards IFA data           |
| **RP (Sender)**   | Fixed alpha / phase A-B-C state machine in firmware  | Custom algorithm on NIC embedded cores (DOCA PCC)             |
| **Signal**        | Binary (CNP received or not)                         | Quantitative (queue depth, per-hop path data, timestamps)     |
| **Tunability**    | Parameter knobs (`AI`, `HAI`, `G`, `RPG_GD`, etc.)   | Entire algorithm is replaceable                               |
