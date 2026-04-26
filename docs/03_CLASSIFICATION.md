
# Traffic Classification and Scheduling

## Marking the Traffic (Tagging)

Before a switch can apply any specialized rules, the packets must carry an identification tag. This happens at two different layers of the network model:

- **Layer 3 Tagging** (DiffServ / DSCP): When a server or edge router generates an IP packet, it marks the 6-bit **Differentiated Services Code Point** (DSCP) field in the IP header. Because this tag lives at Layer 3, it survives routing across different subnets. It provides up to 64 possible classifications (e.g., EF for VoIP, AF41 for Video). DiffServ was covered in the [Service Models document](02_SERVICE_MODELS.md).

- **Layer 2 Tagging** (IEEE 802.1Q / PCP): When that packet is encapsulated into an Ethernet frame, a 3-bit **Priority Code Point** (PCP) is added to the VLAN tag. This provides 8 priority levels (0–7). Because it is a Layer 2 tag, it is visible to the switch hardware immediately, but it is stripped off if the packet is routed without a VLAN tag.

<img src="../pics/pcp.jpg" width="500"/>


## Data Center Bridging (DCB) — Why it exists

### The "Wild West" of QoS (1998–2011)

Standards like 802.1Q and DiffServ defined the tags (PCP and DSCP) but failed to specify exactly how switch silicon should handle those tags at the hardware level. Because there was no standard for how queues should be scheduled or how losslessness should be guaranteed, switch vendors bolted on these behaviors in proprietary, incompatible ways. For over a decade, building a multi-vendor fabric with consistent QoS was nearly impossible.

### The Catalyst for Convergence (FCoE)

In the late 2000s, data centers typically maintained two completely separate physical networks:

- **The LAN** (Local Area Network): Standard Ethernet for user and application traffic. It was expected to be "lossy" and relied on upper-layer protocols like TCP to retransmit dropped packets.

- **The SAN** (Storage Area Network): Fibre Channel (FC) networks dedicated entirely to storage. FC protocols assumed a physically flawless, lossless medium; storage arrays would severely degrade or crash if packets were dropped.

To reduce cabling and hardware costs, the industry pushed to converge these two networks running both regular data and storage traffic over the exact same Ethernet switches. The flagship technology for this was **Fibre Channel over Ethernet** (FCoE). However, to make Ethernet acceptable for SAN traffic, engineers needed a standardized way to guarantee that specific storage priorities would never drop packets, while traditional TCP traffic remained lossy.

<img src="../pics/Storage_FCoE.png" width="400"/>

### The Standardization of DCB (2011)

To solve the FCoE convergence problem and end the era of proprietary vendor lock-in, the IEEE crystallized a family of standards around 2011 under the umbrella of **Data Center Bridging (DCB)**. This suite provided the missing hardware instructions for Ethernet silicon. For the first time, operators could confidently run a lossless storage class alongside a lossy best-effort class on a single, standardized physical wire.

### The Modern Rebirth (RoCEv2 and AI Fabrics)

Ultimately, FCoE proved overwhelmingly complex to deploy and largely faded from the headline data center story. For a brief period, it seemed the DCB standards suite might become obsolete alongside it. However, the explosive growth of High-Performance Computing (HPC) and AI clusters brought a nearly identical problem back to the surface: InfiniBand.

Just like Fibre Channel, InfiniBand relies on a natively lossless architecture. When the industry developed RoCEv2 (RDMA over Converged Ethernet) to run InfiniBand operations over standard Ethernet, they inherited the exact same requirement: they needed to prevent packet drops at all costs. Network engineers dusted off the DCB protocols. The exact same building blocks originally invented to protect legacy storage arrays were perfectly suited to protect modern, high-speed AI workloads. Today, DCB remains the foundational QoS architecture for RDMA-based data centers.

> For an in-depth treatment of RDMA, InfiniBand, and RoCE, refer to the [RDMA Primer](https://github.com/ManiAm/RDMA-Primer).



### DCB building blocks (ETS, PFC, DCBX, QCN)

| Feature | IEEE Standard | Primary Mechanism | Core Function |
| ------- | ------------- | ----------------- | ------------- |
| Enhanced Transmission Selection (ETS) | IEEE 802.1Qaz | Schedules egress queues using strict-priority or Weighted Round-Robin policies and allocates minimum bandwidth percentages per traffic class. | **The Traffic Cop**: Ensures fair bandwidth distribution under congestion. Guarantees that massive RDMA flows cannot completely starve vital network management or control-plane traffic. |
| Priority Flow Control (PFC) | IEEE 802.1Qbb | Enables per-priority PAUSE frames on a single Ethernet link, allowing the switch to halt one priority (e.g., RoCE on PCP 3) while others continue flowing. | **The Emergency Brake**: Physically prevents packet loss by pausing a lossless priority before its ingress buffer overflows, without stopping other traffic. |
| Data Center Bridging Exchange (DCBX) | IEEE 802.1Qaz (Extension of 802.1AB LLDP) | A discovery and capability exchange protocol that allows directly connected devices to communicate their DCB settings. | **The Auto-Negotiator**: Prevents silent misconfigurations by ensuring both sides of the cable agree on which priorities have PFC enabled and what ETS bandwidth weights are applied. |
| Quantized Congestion Notification (QCN) | IEEE 802.1Qau | Uses Congestion Notification Messages (CNMs) to signal the original sender to slow down its transmission rate at Layer 2. | **The Layer 2 Soft Brake**: Obsolete in modern data centers. RoCEv2 networks bypass QCN and use Layer 3 ECN and DCQCN instead, because QCN cannot route across Layer 3 IP networks. |





## The Egress Path

When a packet arrives at an egress port, the switch must determine exactly when and how to send it onto the wire. This pipeline relies on four sequential steps:

- **Classification**: Deriving a single internal priority from the packet's headers.

- **Traffic Class Assignment**: Assigning that priority to a specific Traffic Class (TC).

- **Queuing**: Placing the packet into that class's dedicated egress queue.

- **Scheduling**: Determining the order in which those egress queues are emptied onto the wire.

No single standard governs the entire pipeline. Classification depends on the packet's layer. The 3-bit PCP field comes from the IEEE 802.1Q base standard, while the 6-bit DSCP field comes from the IETF DiffServ RFCs. Traffic Class assignment and egress scheduling are standardized by **IEEE 802.1Qaz**, commonly known as **Enhanced Transmission Selection (ETS)**. The following sections break down this pipeline, starting from how a switch reads a packet's tags to how ETS ultimately schedules the traffic.

### Classification and Trust Mode

When a packet arrives, the switch must decide which header field (DSCP or PCP) dictates the packet's priority. This is defined by a configurable **Trust Mode** on the ingress port:

- **Trust DSCP**: The switch reads the 6-bit DSCP field. This is the standard for routed fabrics.
- **Trust PCP**: The switch reads the 3-bit PCP field. This is standard for Layer 2 fabrics where the VLAN tag is preserved end-to-end.
- **Trust Both**: The switch dynamically uses DSCP for IP packets and PCP for non-IP traffic (e.g., ARP, LLDP).

Switch pipelines (including Priority Flow Control and egress queuing) typically operate on a 3-bit **internal priority**. If the switch trusts the 3-bit PCP, the mapping is 1:1. However, if the switch trusts the 6-bit DSCP, it must use a mapping table to compress 64 possible DSCP values down to 8 internal priority values. By design, multiple DSCP values will map to the same internal priority (for example, standard policies often map DSCP 46 to internal priority 5).

```
 IP Header                  Internal Priority
┌────────────┐             ┌────────┐
│ DSCP (6b)  │ ──mapping──►│ 3 bits │
└────────────┘             └────────┘
  64 values                 8 values
```

### Traffic Class Assignment

Once the switch has established a single internal priority, it must logically group the packet for scheduling. This is where IEEE 802.1Qaz (ETS) begins. ETS defines the concept of the Traffic Class (TC). A Traffic Class (TC0–TC7) is a logical grouping that dictates how the traffic will be treated by the scheduler. The switch uses a configurable Priority-to-TC table to map the 3-bit internal priority to one of these logical classes.

| Internal Priority | DSCP Example | Application Type   | Traffic Class |
| ----------------- | ------------ | ------------------ | ------------- |
| 5                 | EF (46)      | VoIP               | TC7 (highest) |
| 4                 | AF41 (34)    | Interactive Video  | TC5           |
| 2                 | AF21 (18)    | Transactional Data | TC3           |
| 1                 | CS1 (8)      | Scavenger/Bulk     | TC1           |
| 0                 | DF / CS0 (0) | Best Effort        | TC0 (lowest)  |


### Queuing (Silicon Implementation)

With the logical Traffic Class assigned, the packet must be placed into physical memory.

While ETS defines the logical concept of up to 8 Traffic Classes, it does not dictate the physical queue architecture. How many physical queues actually exist on the port, how their buffers are sized, and how packets are enqueued is strictly a silicon implementation detail determined by the switch ASIC. However, the overarching rule is that packets mapped to the same TC will share the same dedicated physical egress queue.

The hardware ensures that high-priority voice traffic physically sits in a different queue than best-effort background data, rather than dropping everything into one undifferentiated FIFO queue.


### Egress Scheduling (Emptying the Queues)

With multiple egress queues holding different classes of traffic, the switch requires a scheduler to decide which queue gets to transmit next when the port is free. This is the core of IEEE 802.1Qaz (ETS). ETS allows network operators to apply one of two scheduling algorithms to each Traffic Class:

- **Strict Priority (SP)**: The highest-priority queue is always drained first. As long as the SP queue has a packet waiting, no other queue is allowed to transmit. This algorithm guarantees the absolute lowest possible latency for highly critical traffic (like VoIP or RDMA Congestion Notification Packets). However, it carries the risk of starvation. If the high-priority queue is constantly full, lower-priority queues will never transmit.

- **Deficit Weighted Round Robin (DWRR)**: Queues take turns transmitting based on a configured weight (percentage of bandwidth). DWRR tracks a "deficit counter" to account for variable packet sizes. This ensures true byte-level fairness across the queues, rather than just packet-count fairness, preventing starvation while respecting bandwidth tiers.

Modern switch deployments typically combine both algorithms to balance performance and fairness. Network engineers will usually assign Strict Priority to one or two queues for latency-sensitive traffic, and apply DWRR to the remaining queues to fairly divide the leftover bandwidth.

```
Egress Port
    ▲
    │
Scheduler
    ▲
    ├── TC7: VoIP / CNPs          ◄── Strict Priority (always first)
    ├── TC6: Routing Control      ◄── Strict Priority
    ├── TC5: Interactive Video    ◄── DWRR (weight: 30%)
    ├── TC3: Transactional Data   ◄── DWRR (weight: 25%)
    ├── TC1: Scavenger            ◄── DWRR (weight: 10%)
    └── TC0: Best Effort          ◄── DWRR (weight: 35%)
```




## The Ingress Path — Preventing Loss (IEEE 802.1Qbb / PFC)

While the egress path (802.1Qaz) focuses on how a switch schedules and sends traffic, the ingress path focuses on how a switch receives and buffers traffic.


### The Problem with Standard Flow Control

Standard Ethernet was originally designed to be "lossy" (it drops packets when congested and relies on upper-layer protocols like TCP to retransmit). To mitigate massive drops, early Ethernet introduced the IEEE 802.3x PAUSE mechanism.

- **The Flaw (Global Pause)**: 802.3x PAUSE is a blunt instrument. It pauses the entire physical link. In a modern converged network, this is unacceptable. If a switch pauses a port to prevent dropping bulk storage data, it will simultaneously halt critical, latency-sensitive traffic like VoIP.

- **The Solution (PFC)**: Priority-based Flow Control (PFC), introduced by IEEE 802.1Qbb, solves this by sending PAUSE frames that target a single priority. This allows a switch to pause high-volume, lossless traffic (like RDMA) while allowing other traffic (like best-effort data) to continue flowing over the exact same physical cable.


### Ingress Buffers and Priority Groups (PG)

Before a switch can pause a priority, it must know how to allocate memory to it. When a packet arrives, its 3-bit internal priority (derived from the DSCP or PCP tags) is mapped to an ingress buffer known as a **Priority Group** (PG). A Priority Group dictates two critical things:

- **Buffer Allocation**: How much memory space is reserved for this specific group of traffic.

- **Lossless Behavior**: Whether PFC is enabled or disabled for this group. Only priorities mapped to a "lossless" PG will trigger PFC PAUSE frames. Priorities in a "lossy" PG will simply be dropped if their buffer overflows.


> PFC is covered in much more detail in **[Priority-based Flow Control (PFC)](04_PFC.md)**.




## Putting It All Together

Whether a packet arrives with a 6-bit DSCP tag or a 3-bit PCP tag, the switch’s first task is to normalize that value into a single internal priority. Although internal priority is traditionally a 3-bit value (0–7), modern ASIC architectures may employ larger internal widths to support highly granular queuing hierarchies and drop profiles. This internal priority acts as the central pivot point, simultaneously driving both the ingress and egress QoS pipelines:

```text
                                       ┌─► Traffic Class (TC) ──► Egress Queue & ETS Scheduling
                                       │   (IEEE 802.1Qaz)        (Determines WHEN it leaves)
                                       │
Incoming Packet ──► Internal Priority ─┤
(DSCP or PCP)                          │
                                       │
                                       └─► Priority Group (PG) ──► Ingress Buffer & PFC State
                                           (IEEE 802.1Qbb)         (Determines IF it pauses)
```

Once the packet is assigned its internal priority, its fate is determined by two parallel processes:

- **The Ingress Path** (Buffering & Flow Control): Governed by IEEE 802.1Qbb, the internal priority dictates the Priority Group (PG). The PG allocates the necessary memory buffer as the packet enters the switch and determines if the traffic is lossless. If the buffer fills, it triggers a PFC PAUSE frame upstream for this specific priority.

- **The Egress Path** (Queuing & Scheduling): Governed by IEEE 802.1Qaz, the internal priority dictates the Traffic Class (TC). The TC assigns the packet to a specific physical egress queue, and the ETS scheduler (using Strict Priority or DWRR) decides exactly when that packet is serialized onto the wire.

Ultimately, TCs control how traffic exits the switch (bandwidth allocation and latency guarantees), while PGs control how traffic is buffered as it enters (memory isolation and loss prevention). This dual-mapping architecture is the foundation of modern high-performance networks. It is the mechanism that enables a data center to run lossless RoCEv2 traffic (which relies on strict buffering and PFC to prevent drops) on the exact same physical infrastructure as lossy best-effort data (which relies on DWRR scheduling to fairly share leftover bandwidth). They share the same cable, but their behavior is completely isolated through distinct PGs and TCs.


## The DCB Toolbox: Tailoring the Pipeline to Application Needs

While the dual-mapping architecture described above makes converged networks possible, it also leads to a common misconception. Because DCB is frequently marketed under the umbrella of "Lossless Ethernet," it is easy to assume it is an all-or-nothing feature that latency-sensitive traffic (like VoIP) completely bypasses. In reality, DCB is a modular toolbox. A converged network succeeds by allowing each application class to consume only the specific DCB components that serve its needs, while ignoring the rest.

Here is how different traffic profiles navigate the exact same DCB pipeline:

- **The Need for Speed (Latency-Sensitive)**: Applications like real-time Voice over IP (VoIP) or routing control protocols (BGP) explicitly reject the lossless tool, PFC. For these applications, pausing a packet in a buffer ruins the real-time audio or breaks the network topology. Therefore, they map to standard lossy Priority Groups on the ingress side. If the queue fills up, the switch simply drops the packet. However, because they share the egress port with massive data flows, they must interact with the scheduling tool, ETS (802.1Qaz). They utilize ETS to secure a Strict Priority (SP) fast-lane, allowing them to instantly jump ahead of bulk data.

- **The Need for Perfection (Loss-Sensitive)**: Applications like RoCEv2 or traditional storage cannot tolerate dropped packets. They actively enable PFC on the ingress port to guarantee zero packet loss, creating a lossless Priority Group. On the egress port, instead of demanding a strict fast-lane (which would completely starve the rest of the switch given their massive volume), they rely on ETS configured with DWRR weights. This allows them to fairly share the remaining bandwidth percentages alongside best-effort web traffic.

The table below illustrates how various applications mix and match these tools based on their core requirements:

| Traffic Type        | Example Protocol  | Needs PFC? (Lossless)                       | ETS Scheduling Algorithm    |
| ------------------- | ----------------- | ------------------------------------------- | --------------------------- |
| Standard Data       | HTTP, general TCP | NO (Let TCP handle loss)                    | DWRR                        |
| Network Control     | BGP, OSPF, BFD    | NO (Drop, don't delay)                      | Strict Priority             |
| Real-Time Voice     | VoIP, SIP         | NO (Drop, don't delay)                      | Strict Priority (Fast lane) |
| Traditional Storage | iSCSI             | Optional (often YES if lossless end-to-end) | DWRR                        |
| RDMA / AI Data      | RoCEv2, NVMe-oF   | YES (Pause, don't drop)                     | DWRR (Bandwidth share)      |
| RDMA CNPs           | RoCEv2 CNP        | NO (Tiny, infrequent — no PFC needed)       | Strict Priority             |


### Layer 3 Marking (DSCP) Best Practices and Pitfalls

To translate the table above into actual network configurations, devices must tag these packets at Layer 3 using DiffServ markings. When converging massive AI flows, voice, and standard data onto a single fabric, assigning the correct DSCP codepoint is important to avoid disastrous overlaps:

- **Standard Data** — **CS0 / DF (DSCP 0)**. General TCP, HTTP, and management traffic. Usually left at the default; no special marking required.

- **Network Control** — **CS6 (DSCP 48)**. Routing keep-alives and topology updates (BGP, OSPF, BFD) are critical to network survival. CS6 is reserved almost exclusively for this purpose.

- **Real-Time Voice** — **EF (DSCP 46)**. The universally accepted codepoint for VoIP bearer audio. EF guarantees low latency, low jitter, and minimal loss via Strict Priority scheduling.

- **Traditional Storage (iSCSI)** — No single global standard. Common choices are **AF21 (DSCP 18)** or **CS4 (DSCP 32)**, depending on vendor and whether the fabric is trusted end-to-end. Do not assume **AF41 (DSCP 34)**, which in the Cisco QoS baseline is mapped to interactive video.

- **RDMA Data (RoCEv2)** — **AF31 (DSCP 26)** is the most widely documented default in NVIDIA, Broadcom, and Intel RoCEv2 deployment guides. Some designs use **EF (DSCP 46)** instead; the choice depends on the vendor blueprint and whether voice shares the fabric.

- **Congestion Notification Packets (CNPs)** — CNPs are tiny Layer 3 packets generated by RoCEv2 receivers to signal congestion back to senders. Because they are latency-critical control messages (not bulk data), they are typically placed in a **Strict Priority** queue. Common markings include **DSCP 48 (CS6)** or **EF (DSCP 46)**, depending on whether the operator separates CNPs from routing control or groups all low-volume SP traffic together.

> **Critical pitfall:** Never mark massive bulk RDMA flows with CS6 (DSCP 48). Doing so places multi-gigabit AI transfers in the exact same Strict Priority queue as routing keep-alives. A single sustained RDMA burst can starve BGP and OSPF, collapsing the entire network topology.


## Data Center Bridging Exchange (DCBX) — IEEE 802.1Qaz

Configuring DCB parameters such as PFC priorities, ETS weights, and application mappings manually across thousands of switch ports and server Network Interface Cards (NICs) is highly error-prone. The Data Center Bridging Exchange Protocol (DCBX) automates this process by allowing directly connected devices to dynamically discover and negotiate their QoS settings before actual data flows.

Without an automated handshake, converged networks are highly vulnerable to **silent misconfigurations**. For example, if a switch is configured to expect lossless RoCEv2 traffic on Priority 3, but a server NIC is misconfigured to send it on Priority 0, the data will still flow. However, it will land in a standard, lossy queue. Under heavy load, the switch will drop those packets, causing catastrophic performance degradation without triggering any obvious configuration alarms. DCBX provides a **declared** common configuration on the link. It dramatically reduces—but does not mathematically guarantee elimination of—all misconfigurations: both endpoints must implement DCBX, the policy must be applied correctly, and operators still validate with test traffic and monitoring.

### Phase 1: The Communication Channel (LLDP)

DCBX is not a standalone protocol. It is packaged as an extension layered on top of the Link Layer Discovery Protocol (LLDP, IEEE 802.1AB). When a server NIC connects to a switch, LLDP immediately opens a continuous, low-level dialogue between the two devices. DCBX uses this existing channel to transmit its parameters as specific Type-Length-Value (TLV) fields within the LLDP frames.

### Phase 2: The Parameter Exchange

Over this LLDP channel, DCBX exchanges the three core pillars of the QoS architecture discussed in the previous sections:

- **Application Mapping**: Which application protocols (e.g., RoCEv2 tagged with DSCP 26) must be mapped to which internal Traffic Classes.

- **PFC Configuration (802.1Qbb)**: Which specific priority groups have Priority-based Flow Control enabled (making them lossless) versus disabled (lossy).

- **ETS Bandwidth Allocation (802.1Qaz)**: Which Traffic Classes are granted Strict Priority, and what specific DWRR percentage weights are assigned to the remaining queues.

### Phase 3: Enforcement and "Willing Mode"

If the negotiation flags a mismatch (e.g., the NIC and Switch disagree on which priority is lossless), the system can halt transmission for that specific traffic class until the error is resolved. However, in modern, large-scale data centers, it is highly inefficient to independently manage configurations on both the switches and the servers. To solve this, DCBX relies on a deployment model known as "Willing Mode":

- **The Switch (Authoritative)**: The network administrator configures the "master" QoS policy exclusively on the switch.

- **The Server NIC (Willing)**: The NIC is configured to operate in "willing mode." This means the NIC passively listens to the DCBX advertisements from the switch and automatically overrides its own local settings to match the switch's configuration.

```text
Switch (Authoritative)                        Server NIC (Willing Mode)
┌──────────────────┐                          ┌──────────────────┐
│ QoS Policy Maker │─── LLDP / DCBX Packets ─►│ QoS Policy Taker │
│                  │                          │                  │
│ [x] PFC: Pri 3   │◄── 1. Switch Advertises ─│ (Awaits Config)  │
│ [x] ETS: 50%     │                          │                  │
│ [x] App: DSCP 26 │─── 2. NIC Applies ──────►│ [x] PFC: Pri 3   │
└──────────────────┘                          │ [x] ETS: 50%     │
         ▲                                    │ [x] App: DSCP 26 │
         │                                    └──────────────────┘
         │          3. Safe Data Transmission          ▼
         └─────────────────────────────────────────────┘
```

### DCBX on the ConnectX-4

On a ConnectX-4, we can query the firmware-level DCBX settings to see how the NIC is configured to participate in this negotiation:

```bash
sudo mlxconfig -d /dev/mst/mt4115_pciconf0 q | grep -iE "dcbx"

        LLDP_NB_DCBX_P1                             False(0)
        DCBX_IEEE_P1                                True(1)
        DCBX_CEE_P1                                 True(1)
        DCBX_WILLING_P1                             True(1)
```

- **`LLDP_NB_DCBX_P1 = False`**: LLDP is not in non-blocking mode, meaning DCBX negotiation runs normally over the LLDP channel. If this were `True`, LLDP frames would be forwarded to the host instead of being consumed by the NIC firmware, disabling automatic DCBX negotiation.

- **`DCBX_IEEE_P1 = True`**: The NIC supports the modern IEEE 802.1Qaz DCBX standard. This is the version used in current RoCEv2 deployments and is the protocol that carries the PFC, ETS, and application mapping TLVs.

- **`DCBX_CEE_P1 = True`**: The NIC also supports the older Converged Enhanced Ethernet (CEE) version of DCBX, originally developed by Cisco and Intel before the IEEE standard was ratified. Having both enabled allows the NIC to negotiate with legacy switches that only speak CEE.

- **`DCBX_WILLING_P1 = True`**: The NIC is in "willing" mode, meaning it will accept the switch's DCB parameters rather than insisting on its own. In a typical deployment, the switch is the authoritative source of truth for PFC priorities, ETS bandwidth weights, and application mappings. A willing NIC defers to whatever the switch advertises.




## The Legacy Standard: Layer 2 Congestion Control (IEEE 802.1Qau / QCN)

Quantized Congestion Notification (QCN) was part of the original Data Center Bridging (DCB) suite. It was designed to provide direct, rate-limiting congestion control for early networks (such as initial FCoE or RoCEv1 deployments). However, due to fundamental shifts in how modern data centers are built, QCN is now largely obsolete.

To understand why QCN failed, one must understand network boundaries. In a flat Layer 2 network, devices communicate using MAC addresses, and frames can flow freely. However, modern data centers are built with Layer 3 routers separating the switch tiers. Native Layer 2 frames cannot pass through a Layer 3 router; they are strictly trapped within their local broadcast domain.

In early, flat Layer 2 data centers, QCN acted as a simple, direct feedback loop between the congested switch and the sender:

- **Congestion Detection**: A switch detects that its buffers are filling up.

- **The CNM Frame**: The switch generates a Congestion Notification Message (CNM).

- **Direct Feedback**: The switch sends this CNM as a native Layer 2 frame directly backward to the MAC address of the offending server, instructing the server's NIC to throttle its transmission rate.

As data centers scaled to massive sizes, they abandoned flat Layer 2 designs in favor of Layer 3 IP routing between every switch tier. This architectural shift immediately broke QCN. Because CNMs are strictly Layer 2 frames, they lack IP headers. If a server in Rack A sends traffic to a server in Rack B, and an intermediate switch experiences congestion, that switch attempts to send a L2 CNM back to Rack A. However, because the CNM hits a Layer 3 routing boundary, it is immediately dropped. The congestion signal never reaches the sender, rendering the protocol useless.

Because congestion signals must now traverse routed IP fabrics, modern RoCEv2 networks replace QCN entirely with a Layer 3-aware solution: Explicit Congestion Notification (ECN) coupled with the Data Center Quantized Congestion Notification (DCQCN) algorithm. Instead of the switch generating a L2 message and sending it backward, the switch simply marks the IP header of the forward-flowing packet (ECN). When the destination server sees this mark, it generates a fully routable Layer 3 packet (a Congestion Notification Packet, or CNP) and sends it back to the source. This ensures the congestion signal can successfully navigate the routers and reach the sender.

> DCQCN is covered in much more detail in **[Data Center Quantized Congestion Notification](05_DCQCN.md)**.
