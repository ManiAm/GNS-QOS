
# Beyond Standard DCQCN

The [DCQCN feedback loop](05_DCQCN.md) is effective, but it has a fundamental limitation: the signal is binary. The standard CNP tells the sender exactly one thing — "you caused congestion" — with no information about how severe the congestion is, which switch is the bottleneck, or how deep the queue has grown. The sender reacts to every CNP with the same formula, regardless of whether the switch buffer is 10% full or 90% full. This can lead to over-correction (slashing the rate when a gentle tap would suffice) or under-correction (applying the same mild cut to both minor and severe congestion).

The evolution beyond standard DCQCN follows three threads:

- Richer congestion signals that give the sender real data about network conditions
- Faster feedback paths that cut the control loop latency by having switches signal the sender directly
- A programmable engine that lets the sender act on any of these signals with custom logic


## Per-Hop Telemetry (INT and IFA)

The most ambitious answer to the binary signal problem is per-hop telemetry: instrumenting the network so that switches stamp metadata — their identity, the ports used, the instantaneous queue depth, and nanosecond-precision timestamps — into packets as they transit the fabric. By the time a telemetry-carrying packet reaches its destination, it holds a complete, ordered record of every switch it touched and the congestion state at each one.

This idea was first realized by **In-Network Telemetry (INT)**, developed by Barefoot Networks (now Intel) around 2015–2016 as part of the P4 programmable switch ecosystem. INT was specified by the P4.org Applications Working Group and designed for programmable Tofino ASICs.

**In-Band Flow Analytics (IFA)** is Broadcom's independent implementation of the same concept, designed for their fixed-function ASICs (Trident and Tomahawk). IFA was submitted to the IETF for industry-wide standardization and is documented as an Internet-Draft in the IP Performance Measurement (IPPM) working group ([draft-kumar-ippm-ifa](https://datatracker.ietf.org/doc/draft-kumar-ippm-ifa/)), now at version 2 (IFAv2).

IFA is a general-purpose telemetry standard — not specific to RDMA or RoCEv2. It can stamp telemetry into any IP packet that matches a configured policy. Its broader uses include network monitoring, path tracing, and SLA verification. However, congestion control for lossless RDMA fabrics is one of its most impactful applications, because the standard DCQCN feedback loop provides so little information to work with.

> INT and IFA are conceptually identical but not wire-compatible. They carry the same per-hop information using different header formats and target different switch silicon. For the purposes of congestion control, they are interchangeable: both provide the same queue depth, utilization, and timing data that a sender's rate-control algorithm needs.

### Operational Modes

The IFA standard ([draft-kumar-ippm-ifa-08](https://datatracker.ietf.org/doc/draft-kumar-ippm-ifa/)) defines three roles within an **IFA zone** — the monitored domain. An **Initiating node** at the edge marks or clones selected traffic, **Transit nodes** inside the zone append their metadata to every IFA-marked packet they forward, and a **Terminating node** at the far edge collects the accumulated telemetry. The standard defines two operational modes, controlled by the **I (Inband)** flag in the IFA header:

- **Inband mode (I = 1)** — Telemetry is inserted directly into live data packets. Every matched packet receives metadata at every transit hop. The Terminating node strips the IFA headers and forwards the original packet to its destination. The draft describes this as inserting metadata "on a per packet basis in live traffic."

- **Clone mode (I = 0)** — Rather than modifying live traffic, the Initiating node creates synthetic copies of selected packets, inserts the IFA header into the clone, and forwards both. Only the clones carry telemetry and accumulate metadata at each transit hop; the original data packets traverse the network untouched. The Terminating node drops the clones after extracting the telemetry and reporting it to a collector.

The choice between modes is made at the Initiating node through configuration. Transit switches do not distinguish between the two — they see an IFA-marked packet and stamp it unconditionally.

> **Sampling in clone mode:** Cloning every packet would double the traffic on the fabric, so the draft requires that cloned traffic "be at a sampled ratio to keep the network overhead to a minimum." The Initiating node selects only a subset of matched packets — for example, one per burst — and clones only those. The sampling ratio is an implementation choice; for congestion control purposes, one probe per burst is sufficient because rate adjustments only need to happen once per transmission window, not once per packet.

### Packet Lifecycle

The following diagram illustrates how per-hop telemetry accumulates in **inband mode** as a packet traverses the network fabric.

<img src="../pics/ifa.png" width="500"/>

The source server transmits a standard data packet (Step 1). As this packet moves through the network (Steps 2, 3, and 4), each telemetry-enabled switch recognizes the IFA header, inserts its own metadata block, and forwards the packet. The packet builds a chronological, stacked record of its journey and the precise conditions at every node it passed through.

Upon reaching the destination server, the Terminating node extracts the complete telemetry stack, strips the IFA headers, and delivers the original payload. The receiving NIC embeds the extracted multi-hop metadata into the returning ACK packet (Step 5). When the original sender receives this ACK, its congestion control engine uses the full path-level view to pinpoint the exact location and severity of any congestion, and adjusts the flow rate accordingly (Step 6).

In **clone mode**, the lifecycle is similar except the telemetry rides in a separate probe packet — not in the data packet itself. The probe follows the same network path and queues as the data flow it represents, so it experiences the same congestion conditions. The Terminating node drops the probe and reports the telemetry to a collector or reflects it back to the sender.


### Header Format

Per-hop telemetry metadata is not stored in the IP header or any existing protocol field. Instead, each switch inserts a metadata block between the packet's transport header (e.g., UDP) and the original payload — the default placement, called **payload stamping**. The standard also defines **tail stamping**, where metadata is appended after the payload before the FCS, which preserves layer-5 visibility for legacy devices. Each hop appends its own metadata block to a growing stack, so the headers accumulate in chronological order as the packet traverses the fabric.

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

The fixed header at the top of the stack contains bookkeeping fields: a protocol identifier, the maximum number of hops allowed, and the current metadata length (updated by each switch as it appends). The IFA standard also defines a **Hop Limit** field — each transit node decrements it, and once it reaches zero, subsequent nodes must stop inserting metadata. Together, Max Length and Hop Limit act as safety bounds that prevent unbounded header growth.

### Per-Hop Metadata

The actual content of each per-hop metadata block is not fixed by the standard. IFA uses a namespace system: a **Global Name Space (GNS)** configured zone-wide, or a **Local Name Space (LNS)** defined per hop. The GNS dictates which fields every switch in the zone must insert (uniform mode), while LNS allows each hop to include its own most relevant data (non-uniform mode). The only field fixed by the standard itself is a 28-bit **Device ID**; everything else is profile-dependent.

The table below shows a representative telemetry profile commonly used for congestion-aware applications:

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


### HPCC — Direct Rate Control from Telemetry

**HPCC (High Precision Congestion Control)**, introduced by Alibaba (Li et al., SIGCOMM 2019), is the most prominent algorithm built on per-hop telemetry. It was the first to prove that a sender with real network measurements can dramatically outperform DCQCN's indirect estimation.

Standard DCQCN's rate adjustment is inherently indirect. The sender never learns the actual state of the network. It receives a binary CNP, updates an estimated severity score ($\alpha$), and applies a multiplicative formula. The resulting rate is a guess — an educated one, but a guess nonetheless. If the congestion is mild, the sender may over-correct and waste bandwidth. If the congestion is severe, the sender may under-correct and trigger PFC.

HPCC eliminates this guesswork. It requires every switch on the path to stamp per-hop telemetry into data packets (the original paper used INT in inband mode). By the time the packet reaches the receiver, it carries the actual link utilization and queue depth recorded at every hop. The receiver reflects this telemetry back to the sender, and the sender computes its new rate directly from the measurements.

The algorithm is straightforward: the sender scans the per-hop telemetry, finds the hop with the highest utilization, and sets its transmission rate to drive that bottleneck link toward a target utilization (typically 95%). There is no alpha, no phased recovery, no additive or hyper-additive increase. The rate is calculated from a single formula applied to real data:

$$Rate_{new} = \frac{Rate_{current} \times U_{target}}{U_{bottleneck}}$$

Where $U_{target}$ is the desired link utilization (e.g., 95%) and $U_{bottleneck}$ is the measured utilization at the most loaded hop. If the bottleneck link is running at 100% utilization and the target is 95%, the sender cuts its rate by exactly 5%. If the bottleneck is at 60%, the sender increases its rate. The adjustment is always proportional to the actual gap, never a blind multiplicative slash.


### HPCC++ — Solving the Overhead Problem

HPCC's precision comes at a cost. Each switch along the path appends its own metadata block to the packet. In a multi-tier leaf-spine fabric where a packet traverses five hops, the cumulative telemetry header can grow to tens of bytes per packet. For large data transfers, this overhead is negligible. For small RoCEv2 messages (common in collective operations like all-reduce), it becomes significant.

**HPCC++** (also called HPCC-PINT) addresses this by replacing full per-hop headers with **Probabilistic In-band Network Telemetry (PINT)**. Instead of every switch appending a complete metadata block, the telemetry is encoded probabilistically into a single, fixed-size field regardless of hop count. Each switch overwrites the same field with its own data at a configured probability. Over multiple packets, the sender reconstructs a statistically accurate picture of per-hop congestion from many small samples rather than one large header. This compresses the overhead to a constant size (typically 1–2 bytes) while preserving HPCC's core advantage: rate decisions driven by real measurements.

| Algorithm     | Signal                         | Rate Calculation                                    | Overhead             |
|---------------|--------------------------------|-----------------------------------------------------|----------------------|
| **DCQCN**     | Binary ECN / CNP               | Indirect (alpha estimation + multiplicative cut)    | None (ECN is 2 bits) |
| **HPCC**      | Full per-hop INT/IFA telemetry | Direct (proportional to bottleneck utilization)     | Grows with hop count |
| **HPCC++**    | Probabilistic (PINT)           | Direct (same formula, sampled data)                 | Fixed (1–2 bytes)    |


## Congestion Signaling (CSIG)

Full per-hop telemetry provides maximum visibility but is not always necessary or practical. Some deployments need a congestion signal that is richer than binary ECN but avoids the growing overhead of a per-hop metadata stack.

CSIG addresses this with a fundamentally different architectural approach from IFA/INT. Rather than stacking per-hop metadata that grows with each switch, CSIG uses a fixed-size **L2 tag** (4 bytes compact, 8 bytes expanded) positioned between the MAC and IP headers — structurally similar to a VLAN tag. As a packet traverses the fabric, each switch performs a **compare-and-replace** on the same tag: if the switch's local congestion metric is worse than the value already in the tag, it overwrites it; otherwise, it leaves the tag untouched. By the time the packet arrives at its destination, the tag carries the **bottleneck summary** for the entire path — not a per-hop breakdown, but the single worst value.

<img src="../pics/csig-tag.png" width="650"/>

CSIG was developed by Google and submitted to the IETF as an Internet-Draft ([draft-ravi-ippm-csig](https://datatracker.ietf.org/doc/draft-ravi-ippm-csig/)), with co-authorship from Broadcom. It is also being standardized by the Ultra Ethernet Consortium (UEC) as part of the UE 1.1 specification. NVIDIA supports CSIG in Spectrum-4 switches and ConnectX-8 NICs, but the protocol is an open, multi-vendor standard — not proprietary.

The draft defines four concrete signals, each computed locally by every transit switch:

| Signal Type     | What It Captures                                       | Aggregation                        |
|-----------------|--------------------------------------------------------|------------------------------------|
| **min(ABW)**    | Minimum available bandwidth across all hops            | min() — bottleneck bandwidth       |
| **min(ABW/C)**  | Minimum ratio of available bandwidth to link capacity  | min() — bottleneck utilization     |
| **max(PD)**     | Maximum per-hop delay across all hops                  | max() — worst queuing delay        |
| **max(nQD)**    | Maximum normalized queue depth across all hops         | max() — worst queue occupancy      |

Because the tag is fixed-size and tiny, CSIG can be enabled on **every data packet** at line rate — no sampling required. The receiver extracts the tag and reflects the values back to the sender via an L4+ reflection header (e.g., a TCP option or UDP payload field). The sender's congestion control engine then has direct, per-packet access to the bottleneck bandwidth, utilization, or delay along the forward path, all within one round-trip time.

> [This video](https://youtu.be/wDXTqw_bFFY) explains how CSIG provides actionable insights for AI workloads.

> CSIG explicitly builds on lessons from INT and IFA. The draft states: "CSIG builds upon the successful aspects of prior work such as switch in-band network telemetry (INT) that incorporates multibit signals in live data packets. At the same time, CSIG's end-to-end mechanism for carrying the signals via fixed size header is simple, practical and deployable akin to Explicit Congestion Notification (ECN)."

CSIG trades per-hop granularity for simplicity and deployability. An operator chooses along a spectrum based on workload requirements:

| Signal           | Layer     | Overhead             | Visibility                                       | Best For                                                 |
|------------------|-----------|----------------------|--------------------------------------------------|----------------------------------------------------------|
| **Standard ECN** | L3 (IP)   | None (2 bits)        | Binary — congested or not                        | Baseline DCQCN deployments                               |
| **CSIG**         | L2 tag    | Fixed (4 or 8 bytes) | Bottleneck summary (ABW, utilization, or delay)  | Every-packet telemetry at line rate, tunnel-friendly     |
| **IFA / INT**    | L3+       | Grows with hop count | Full per-hop path telemetry                      | Maximum visibility for complex multi-tier fabrics        |


## Swift — Delay-Based Congestion Control

All the congestion control algorithms discussed so far — DCQCN, HPCC, HPCC++ — share a common design: the **network** generates the congestion signal, whether that signal is a binary ECN mark, a per-hop telemetry stack, or a CSIG bottleneck summary. The sender reacts to what the switches tell it. **Swift** takes a fundamentally different approach: the sender measures congestion **itself**, using round-trip time (RTT).

The premise is straightforward. When queues build up at switches, packets take longer to traverse the network. A sender with access to precise timestamps can detect this delay increase and use it directly as a congestion signal — no switch cooperation required beyond basic forwarding. Deeper queues produce longer RTT; empty queues produce minimal RTT. The signal is inherently proportional: unlike binary ECN, which says only "congested or not," RTT tells the sender *how much* congestion exists.

Swift was developed by Google and published at SIGCOMM 2020 (*"Swift: Delay is Simple and Effective for Congestion Control in the Datacenter,"* Kumar et al.). It evolved from an earlier Google protocol called **TIMELY** (SIGCOMM 2015), which was the first to demonstrate that NIC hardware timestamps could drive delay-based congestion control in data centers. Swift simplified TIMELY's design and proved it at production scale.

### How Swift Works

Swift uses an **AIMD (Additive Increase, Multiplicative Decrease)** control loop centered on a configurable **target delay**:

- If the measured RTT is **below** the target delay, the network is uncongested — Swift additively increases the congestion window.
- If the measured RTT is **above** the target delay, queues are building — Swift multiplicatively decreases the window in proportion to how far the delay exceeds the target.

The decrease is not a fixed cut. It scales with the severity of the congestion:

$$md = \beta \times \frac{RTT - target\_delay}{RTT}$$

$$cwnd_{new} = (1 - md) \times cwnd_{current}$$

If the RTT is barely above the target, $md$ is small and the window shrinks gently. If the RTT is far above the target, $md$ is large and the window shrinks aggressively. This proportional response eliminates the over-correction and under-correction problems inherent in DCQCN's fixed multiplicative slash.

### Fabric vs. Host Delay Decomposition

A subtle but important feature of Swift is its ability to separate **fabric delay** (queueing in the network switches) from **host delay** (processing time at the receiving server). End-to-end RTT includes both, but only fabric delay indicates network congestion — a slow receiver does not mean the fabric is overloaded. Swift uses NIC hardware timestamps at both endpoints to decompose the total RTT into its fabric and host components, and reacts only to the fabric portion. This prevents false congestion signals caused by slow hosts or software overhead.

### Where Swift Fits

|                         | DCQCN                                 | HPCC                                    | Swift |
|-------------------------|---------------------------------------|-----------------------------------------|---|
| **Signal source**       | Switch (ECN mark)                     | Switch (per-hop telemetry)              | End-host (RTT measurement) |
| **Switch requirements** | ECN marking thresholds                | IFA/INT telemetry support               | None — basic forwarding only |
| **Reaction**            | Indirect (alpha + multiplicative cut) | Direct (bottleneck utilization formula) | Direct (proportional to delay overshoot) |
| **Overhead**            | None (2 bits)                         | Grows with hop count                    | None |

### Swift and CSIG

Swift is particularly attractive for deployments where switch telemetry support (IFA/INT/CSIG) is unavailable or incomplete, because the sender derives its signal entirely from end-host timestamps. In deployments that *do* have CSIG, Swift's delay-based logic can be augmented with CSIG's `max(PD)` signal for even higher precision. The CSIG draft itself uses Swift as its baseline congestion control algorithm and demonstrates how CSIG signals improve Swift's ramp-up and reaction accuracy.


## Switch-Direct Backward Feedback

Every congestion control mechanism discussed so far shares a common architectural assumption: the feedback signal travels **forward** through the network before being **reflected** back to the sender. In standard DCQCN, the switch marks a packet with ECN, the packet continues forward to the receiver, the receiver generates a CNP, and the CNP travels back. Even with richer signals (IFA, CSIG), this forward-then-reflect model is unchanged — the receiver is always the relay point. Swift removes the switch from the signaling path entirely, but the sender still waits for a full round-trip before it has any measurement to act on.

This means the sender is blind for at least one full RTT after congestion begins. At 400 Gbps over a 4 μs fabric RTT, a single sender injects approximately 200 KB of additional data before it even *learns* there is a problem. At scale — hundreds of flows hitting the same bottleneck — this blind window is the primary reason switch buffers overflow into PFC pauses.

**Switch-direct backward feedback** eliminates the receiver from the control loop. The congested switch itself generates a message directly back to the sender at the moment congestion is detected. The sender learns about the problem in roughly half the time — the signal travels directly from the congestion point to the sender, skipping the entire receiver-side leg of the loop — and the blind injection window shrinks proportionally.

The idea of switch-generated backward signals is not new. [QCN](./03_CLASSIFICATION.md#the-legacy-standard-layer-2-congestion-control-ieee-8021qau--qcn) implemented it in 2010 for Layer 2 domains. When a bridge's queue exceeds a threshold, its ASIC generates a Congestion Notification Message (CNM) directly back to the source MAC address, carrying quantized feedback that enables proportional rate adjustment. QCN proved the concept in hardware but was limited to a single VLAN — it cannot cross routed boundaries. The mechanisms below extend switch-direct backward feedback to routable Layer 3 RoCEv2 fabrics.


### Fast CNP

**Fast CNP** ([draft-xiao-rtgwg-rocev2-fast-cnp](https://datatracker.ietf.org/doc/draft-xiao-rtgwg-rocev2-fast-cnp/), ZTE, 2024) applies switch-direct feedback to RoCEv2 specifically. When a switch ASIC detects congestion (queue depth exceeding threshold), it generates a CNP directly back to the sender without waiting for the receiver.

<img src="../pics/fast-cnp.png" width="700"/>

The key technical challenge is flow identification. The standard RoCEv2 Base Transport Header (BTH) carries only the *Destination* QP — the receiver's queue pair number. The sender needs to know its own *Source* QP to throttle the correct flow. But a sender may communicate with multiple receivers, and different receivers may use the same Destination QP mapped to different Source QPs at the sender. The Destination QP alone is ambiguous.

Fast CNP solves this by prepending an IPv6 Destination Options extension header containing the original packet's destination address (the receiver's IP). The sender uses the combination of Destination QP + Destination Address to perform a unique lookup of the correct Source QP. The switch sets the Fast CNP's source IP to its own loopback address, allowing the sender to identify which switch along the path detected congestion and optionally reroute via a different ECMP path.

The draft also specifies an optional IOAM (In situ OAM) payload, allowing the Fast CNP to carry per-hop telemetry from the congested switch back to the sender — combining backward feedback with rich signal data for algorithms like HPCC++.

> If the switch does not know whether the sender supports Fast CNP, it MAY simultaneously mark the data packet with ECN CE (triggering the standard receiver-originated CNP as a fallback). If the switch knows the sender supports Fast CNP, it MUST NOT mark the ECN bits — avoiding a duplicate signal.


### Bolt

**Bolt** (Stanford/Google, NSDI 2023) extends the concept from simple notification to **sub-RTT congestion control** by leveraging programmable switch ASICs (P4/Tofino). Rather than generating a minimal backward notification, Bolt's switch data plane computes precise congestion signals — actual queue depths, available bandwidth, and per-flow state — and feeds them directly to the sender at the point of congestion. The sender reacts before the packet even reaches the receiver, shrinking the control loop below one RTT.

Bolt is built on three hardware-level mechanisms:

- **Sub-RTT Control (SRC)**: The switch pipeline detects congestion and generates backward feedback within a single packet processing cycle — no store-and-forward delay, no software involvement.

- **Proactive Ramp-up (PRU)**: The switch tracks flow completions in its pipeline and signals neighboring senders to claim released bandwidth immediately, rather than waiting for the slow additive-increase ramp.

- **Supply Matching (SM)**: The switch ASIC explicitly computes available bandwidth and distributes it to competing flows, replacing the sender's indirect estimation with a direct allocation.

Bolt reports 80% reduction in 99th-percentile tail latency and up to 3× improvement in flow completion time compared to Swift and HPCC at 400 Gbps. It requires P4-programmable switches (e.g., Intel Tofino) and is not yet available on fixed-function merchant silicon (Broadcom Tomahawk, NVIDIA Spectrum).


## Summary Reference

### Hardware Congestion Control Algorithms

All congestion control in the RoCEv2 ecosystem runs in NIC hardware or firmware — kernel bypass demands it. The table below lists every NIC-side rate control algorithm discussed in this document and its [predecessor](05_DCQCN.md), ordered chronologically.

| Year | Algorithm | Execution Layer | Signal Used | Rate Adjustment |
|------|-----------|-----------------|-------------|-----------------|
| 2015 | DCQCN | NIC hardware (ConnectX) | Binary ECN / CNP | α-scaled multiplicative cut + phased recovery |
| 2015 | TIMELY | NIC firmware | End-host RTT (NIC timestamps) | RTT gradient-based AIMD |
| 2019 | HPCC | NIC hardware | Full per-hop INT/IFA telemetry | Direct: rate ∝ target utilization / bottleneck utilization |
| 2020 | HPCC++ | NIC hardware | Probabilistic telemetry (PINT) | Same as HPCC, sampled data |
| 2020 | Swift | NIC hardware/firmware | End-host RTT (NIC timestamps) | Proportional delay-based AIMD |

### Forward-Path Signaling Mechanisms

These are the in-band signals that travel with data packets toward the receiver, then get reflected back to the sender.

| Mechanism | Layer | Generated By | Information Carried | Overhead |
|-----------|-------|--------------|---------------------|----------|
| ECN / CNP | L3 (IP) | Switch marks CE; receiver generates CNP | Binary — "congestion occurred" | None (2 bits in IP header) |
| CSIG | L2 tag | Each switch updates tag via compare-and-replace | Bottleneck summary: min(ABW), max(delay), or max(queue depth) | Fixed (4 or 8 bytes) |
| IFA / INT | L3+ | Each switch appends metadata block | Full per-hop: switch ID, queue depth, timestamps, ports | Grows linearly with hop count |
| PINT | L3+ | Each switch overwrites shared field probabilistically | Same data as IFA, reconstructed statistically over many packets | Fixed (1–2 bytes) |

### Switch-Direct Backward Feedback Mechanisms

These are signals generated by the switch and sent directly back to the sender, bypassing the receiver.

| Mechanism | Year | Switch Hardware | What It Signals | Flow ID Method | Scope |
|-----------|------|-----------------|-----------------|----------------|-------|
| QCN (CNM) | 2010 | Bridge ASIC (fixed-function) | Quantized congestion severity | Source MAC | L2 only (single VLAN) |
| Bolt | 2023 | Programmable ASIC (P4/Tofino) | Precise congestion + bandwidth allocation | Programmable match-action | L3 routable |
| Fast CNP | 2024 | Switch ASIC (fixed-function) | "Slow down" + optional IOAM telemetry | IPv6 ext header + Dest QP | L3 RoCEv2 |


## Programmable Congestion Control (PCC)

Richer signals — whether full per-hop telemetry, CSIG bottleneck summaries, or precise delay measurements — are only useful if the sender can act on them intelligently. Standard DCQCN's Reaction Point cannot. The entire state machine (the alpha update, the three recovery phases, the clamp behavior) is burned into the NIC firmware. Network engineers can tune the parameters (`RPG_GD`, `AI`, `HAI`, `G`, etc.), but the algorithm itself cannot be changed. It has no concept of "queue depth" or "per-hop telemetry." It understands one input (CNP received: yes or no) and runs one fixed algorithm.

**Programmable Congestion Control (PCC)** solves this by replacing the fixed Reaction Point with a programmable engine running on the NIC itself. PCC is a framework provided by NVIDIA's DOCA SDK that allows engineers to write custom congestion control algorithms in C, compile them, and deploy them onto the embedded processing cores of ConnectX NICs. These cores sit on the NIC's data path — they execute at line rate without involving the host CPU, preserving the zero-kernel-bypass property that makes RDMA fast.

With PCC, the Reaction Point is no longer a black box with tunable knobs. It becomes a blank slate. The NIC exposes congestion events (standard CNPs, CSIG signals, IFA telemetry, RTT measurements) to the custom algorithm as structured inputs. The algorithm processes these inputs however it sees fit and produces a single output: the new transmission rate. The NIC's hardware rate limiter then enforces that rate, exactly as it would enforce a rate computed by the standard DCQCN state machine.

This means engineers can implement fundamentally different strategies depending on the workload:

- A large-scale all-reduce training job might use IFA telemetry to identify the single most congested switch in a multi-hop path and surgically reduce the rate just enough to relieve that specific bottleneck, rather than applying a blanket multiplicative cut.

- A latency-sensitive inference workload might react to CSIG's max per-hop delay with a proportional controller (higher delay = harder brake), replacing the indirect alpha-based estimation with a direct measurement.

- A mixed workload might combine multiple signals — using IFA for full path-level diagnostics and CSIG for per-packet bottleneck bandwidth and utilization — and weight them differently depending on the traffic class.

PCC does not change the three-role model (CP, NP, RP) that DCQCN established — it changes the *implementation* of the Reaction Point: from a fixed state machine with tunable parameters to a fully programmable algorithm with structured inputs. The Congestion Point is still the switch detecting congestion. The Notification Point is still the entity generating feedback (the receiver in forward-path mode, or the switch itself in backward-feedback mode). The Reaction Point is still the sender adjusting its rate.

| Component         | Standard DCQCN                                       | Next-Generation (PCC + Telemetry)                             |
|-------------------|------------------------------------------------------|---------------------------------------------------------------|
| **CP (Switch)**   | Marks ECN CE bit                                     | Stamps IFA/CSIG metadata, and/or generates Fast CNP directly  |
| **NP**            | Receiver generates minimal CNP                       | Receiver reflects telemetry, or switch sends Fast CNP (backward) |
| **RP (Sender)**   | Fixed alpha / phase A-B-C state machine in firmware  | Custom algorithm on NIC embedded cores (DOCA PCC)             |
| **Signal**        | Binary (CNP received or not)                         | Quantitative (queue depth, per-hop path data, timestamps)     |
| **Tunability**    | Parameter knobs (`AI`, `HAI`, `G`, `RPG_GD`, etc.)   | Entire algorithm is replaceable                               |
