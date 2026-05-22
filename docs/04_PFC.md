
# Priority-based Flow Control (PFC)

Priority-based Flow Control (PFC), defined by IEEE 802.1Qbb, solves the limitations of legacy Ethernet flow control by enabling lossless transmission on a per-priority basis. While the older 802.3x standard acted as a blunt instrument that paused an entire physical link during congestion (unacceptably halting latency-sensitive traffic like VoIP), PFC can selectively pause individual traffic classes. This allows a switch to momentarily halt high-volume, lossless traffic while permitting other traffic to continue flowing over the exact same physical cable.

To achieve this granular control, PFC relies on **Priority Groups** (PGs) to manage the switch's ingress buffers. When a packet arrives, its internal priority is mapped to a specific PG, which dictates how much buffer memory it receives and whether it operates in a lossless or lossy mode. Only traffic mapped to a lossless PG will trigger targeted PFC PAUSE frames when its specific memory buffer nears capacity. Conversely, traffic in a lossy PG ignores PFC entirely and is simply dropped if its allocated buffer overflows.

<img src="../pics/pfc-classes.webp" width="500"/>

This document covers PFC hardware requirements, threshold mechanics (Xoff, headroom, Xon), and the operational risks it introduces along with the mitigations that modern deployments use to keep those risks contained.


## 802.3x and PFC: Mutual Exclusion

Legacy 802.3x pause and PFC cannot coexist on the same port. 802.3x pauses all traffic indiscriminately, which directly conflicts with PFC's per-priority granularity. If both were active simultaneously, a single 802.3x pause frame would halt all eight Priority Groups, negating the isolation that PFC provides.

In practice, enabling PFC on a port must automatically disable 802.3x flow control. A well-implemented switch enforces this constraint: if an operator attempts to enable 802.3x on a PFC-enabled port, the system either rejects the command or auto-disables 802.3x with a warning. This mutual exclusion is critical for maintaining the integrity of the lossless fabric.


## Layer 2 and Hardware-Driven Logic

Understanding where PFC operates is critical to understanding its hardware requirements:

- **Layer 2 (Hop-by-Hop)**: PFC operates directly between two connected Ethernet ports (e.g., Server NIC ↔ Switch). It is a Layer 2 conversation. No routers or IP addresses are involved in the PAUSE frame transaction itself.

- **Hardware-Driven (Silicon)**: PFC PAUSE frame generation and processing happen entirely in the hardware ASIC (both in switches and NICs), completely bypassing the CPU, operating system, and software drivers.

- **Microsecond Response**: When a buffer threshold is crossed, the network hardware must instantly construct, transmit, and react to PAUSE frames within microseconds to prevent packet drops.


## Hardware Requirements: Data Center vs. Consumer NICs

Because PFC requires dedicated per-priority hardware queues, specific buffer pools, and a full DCB firmware stack (PFC, ETS, DCBX), it cannot be implemented on consumer or commodity Ethernet NICs. The dividing line for PFC support is not link speed, but the target market: Data Center NICs are specifically engineered for lossless Ethernet fabrics.

| Consumer NICs (No PFC Support)                    | Data Center NICs (Full DCB/PFC Support) |
| ------------------------------------------------- | --------------------------------------- |
| Intel I210 / I211                                 | Mellanox/NVIDIA ConnectX-4, 5, 6, 7     |
| Intel I225 / I226 (2.5 GbE)                       | Intel E810 (Ice Lake server NIC)        |
| Realtek RTL8111 / RTL8125                         | Broadcom BCM57500 / NetXtreme-E         |
| Broadcom BCM5720 (Standard 1G server NIC)         | Marvell/Cavium FastLinQ                 |
| Aquantia AQC107 (10 GbE)                          |                                         |
| Standard consumer motherboard NICs / USB Adapters |                                         |


## PFC Thresholds: Managing the Buffer

PFC does not trigger the instant a switch's ingress buffer receives its first packet. Instead, it relies on a carefully managed system of watermarks or thresholds within the ingress buffer to maintain the delicate balance between maximum throughput and zero packet loss. To understand how a switch avoids overflowing, we have to look at the three critical zones of a PFC-enabled buffer.

<img src="../pics/xon-xoff.webp" width="650"/>

### The Xoff Threshold (The Brakes)

This is the high-water mark. When the ingress buffer for a lossless priority group fills up to this specific level, the switch generates and transmits a PFC Pause frame to the upstream sender. Xoff threshold must be set high enough to avoid premature pausing (which unnecessarily halts traffic and wastes bandwidth) but low enough to leave physical space at the top of the buffer for the "pipeline drain" (explained below).

### Headroom (The Stopping Distance)

When a switch hits the Xoff threshold and sends a Pause frame, the upstream sender doesn't stop instantly. The Pause frame takes time to travel backward across the physical cable, and the sender's NIC takes time to process the command. Meanwhile, packets are still flying down the wire at the speed of light. **Headroom** is the dedicated, reserved buffer space above the Xoff threshold designed specifically to catch these in-flight packets. If Xoff is hitting the brakes, headroom is the physical stopping distance required before you hit the wall. Sizing this correctly is paramount:

- **Undersized Headroom**: Packets arriving during the delay will overflow the top of the buffer and be dropped, completely defeating the purpose of a lossless fabric.

- **Oversized Headroom**: Wastes valuable, expensive on-chip buffer memory that could otherwise be dynamically shared among other ports.

The required headroom is a function of how fast the link is transmitting and the total delay before the upstream sender actually stops. The faster the link, the more data accumulates during that delay window. The formula is:

    headroom_bytes = line_rate × total_response_delay

Line rate is the port speed in bytes per second. This is the dominant scaling factor. Doubling the speed doubles the bytes in flight during any fixed delay. The total response delay includes every source of latency in the pause-to-stop path:

- **Cable propagation delay**: The time for the pause frame to travel back across the physical cable. At ~5 ns/m in copper, a 5 m cable adds ~25 ns. A 300 m fiber adds ~1.5 μs.

- **Switch-internal delay**: The time for the local switch to detect the threshold crossing, generate the PAUSE frame, and serialize it through its MAC/PHY pipeline. This typically adds 1–2 μs depending on the ASIC architecture.

- **NIC response time**: The time between the upstream NIC receiving the pause frame and actually halting transmission. Modern ConnectX-class NICs react in roughly ~1 μs.

In short-reach deployments, cable propagation is negligible compared to the switch-internal and NIC processing delays, so headroom scales almost linearly with port speed:

| Port Speed | Cable Delay (5 m) | Switch + NIC Delay | Required Headroom |
| ---------- | ----------------- | ------------------ | ----------------- |
| 100G       | ~25 ns            | ~3 μs              | ~48 KB            |
| 200G       | ~25 ns            | ~3 μs              | ~96 KB            |
| 400G       | ~25 ns            | ~3 μs              | ~192 KB           |
| 800G       | ~25 ns            | ~3 μs              | ~384 KB           |

These are approximate values for short-reach (5 m) deployments. Actual headroom depends on the switch ASIC (cell size, pipeline latency, MAC/PHY delay), NIC generation, and configured safety margins. Production deployments typically reserve 1.5–2x the theoretical minimum to account for worst-case burst alignment and cell overhead. Longer cables (40 m, 300 m) increase the propagation delay significantly and require proportionally larger headroom.

### The Xon Threshold (The Green Light)

Once the upstream sender stops, the switch continues to forward the packets it already has, causing the buffer to drain. However, the switch does not tell the sender to resume the moment it drops one byte below the Xoff line. Instead, it waits for the buffer to drain down to a lower watermark: the Xon threshold. When the buffer hits this level, the switch sends a resume signal (a Pause frame with a timer of zero), allowing the upstream sender to transmit again.

The gap between the Xoff (stop) and Xon (go) thresholds creates **hysteresis**. This prevents rapid pause/resume oscillation. Without Xon, a switch hovering at the Xoff line would rapidly spam stop and go commands, thrashing the network.




## Switch Buffer Architecture: The Problem with Fixed Allocation

The Xoff, Xon, and headroom thresholds govern when PFC fires on a given priority group (PG). But to understand where those packets actually sit in memory, we must look at how the switch ASIC organizes its on-chip buffer.

If a switch carved out a fixed, worst-case chunk of memory for every priority group on every port, it would rapidly exhaust its on-chip buffer. In a real-world network, most ports are idle at any given microsecond while a few might be getting hammered by congestion. Statically assigning memory means that valuable buffer space is wasted on idle ports, while congested ports drop packets because they hit their fixed limits.

To solve this and prevent wasted space, modern switch ASICs use a dynamic, shared buffer architecture. As traffic arrives, each priority group passes through three distinct memory tiers:


### Tier 1: Reserved Pool (The Guarantee)

Every priority group is allocated a small, dedicated block of reserved memory that only it can use. This guaranteed allocation ensures that high-priority traffic always has a minimum place to land and cannot be entirely starved by severe congestion on other ports or priority groups.


### Tier 2: Shared Pool and Dynamic Borrowing (Alpha)

Once a PG fills its reserved space, it begins consuming memory from a massive, global shared buffer pool that is common across all ports. However, we cannot let one port under a heavy incast attack consume the entire shared pool, or it would starve everyone else. To prevent this, the switch uses dynamic thresholds governed by a configuration parameter called `alpha` ($\alpha$).

Alpha determines the maximum fraction of the currently available shared buffer that a single PG is allowed to claim. The standard dynamic threshold limit is calculated as:

$$\text{Max Shared Limit} = \alpha \times \text{Remaining Shared Buffer}$$

Because the "remaining shared buffer" changes in real time as other ports consume or release memory, the limit is dynamic. A port experiencing a massive traffic burst can temporarily borrow deep into the shared pool if other ports are idle. As those idle ports wake up and start receiving traffic, the available shared buffer shrinks, which automatically tightens the limit for the congested port.

- **High Alpha**: The PG is permitted to claim a large multiple of the remaining buffer, allowing it to absorb massive bursts. This is standard for lossless RoCE classes (e.g., PG3) to prevent unnecessary PFC pauses.

- **Low Alpha**: The PG is restricted to a much smaller share. This is typical for lossy traffic classes (e.g., PG0 for standard TCP) where occasional drops are acceptable, protecting the switch from buffer starvation.

A PG continues to grow into the shared pool until its total occupancy reaches the Xoff threshold. At that point, the switch fires a PFC Pause frame to halt the upstream sender.

> Naming note: The buffer management alpha is completely unrelated to the [DCQCN alpha](./05_DCQCN.md#measuring-congestion-severity-the-alpha-alpha-parameter) used as a congestion severity score in a sender's rate control algorithm. They share a Greek letter but operate in entirely different domains. One lives in the switch ASIC's memory controller, the other in the NIC's congestion control state machine.

**Alpha Values and the Power-of-2 Convention**

The alpha multiplier is not an arbitrary real number. Switch MMU hardware implements it as a power of 2, giving a discrete set of values from very restrictive (alpha < 1) to effectively unlimited (alpha >> 1). In the [SAI (Switch Abstraction Interface)](https://github.com/opencomputeproject/SAI) used by SONiC, alpha is configured indirectly through an integer exponent called `dynamic_th`:

$$\alpha = 2^{\,n}$$

Where $n$ is the configured `dynamic_th` integer value.

The `dynamic_th` parameter ranges from **−8 to +8**, yielding alpha values from 1/256 to 256:

| `dynamic_th` | Alpha (multiplier) | Effect on PG |
|--------------|--------------------|-----------------------------------------|
| −8           | 1/256              | Near-zero shared borrowing — PG is almost entirely confined to its reserved pool |
| −3           | 1/8                | Heavily restricted — can only claim 12.5% of remaining free buffer |
| 0            | 1                  | Balanced — PG can claim up to 1× the remaining free buffer |
| 1            | 2                  | Moderate — typical for lossy traffic classes (e.g., PG0) |
| 3            | 8                  | Generous — PG can claim up to 8× remaining free buffer |
| 8            | 256                | Effectively unlimited — PG can absorb the entire shared pool. Typical for lossless RoCE (e.g., PG3) |

When alpha > 1, the formula produces a threshold *larger* than the remaining free buffer. This does not mean the PG can exceed physical memory — the ASIC still enforces the hard limit of total shared pool size. A high alpha simply means "do not artificially restrict this PG; let it consume whatever is physically available before firing PFC."

When alpha < 1, the PG is restricted to a fraction of the free buffer. Multiple low-alpha PGs sharing the same pool naturally divide it among themselves: if 8 PGs each have alpha = 1/8, each can claim at most 12.5% of the free space, leaving room for all of them simultaneously.

This exponent-based representation is standard across SAI-compliant implementations (Broadcom Trident/Tomahawk, NVIDIA Spectrum, Intel Tofino). Vendor-native CLIs may display the value differently — for example, as a raw multiplier or a named profile — but the underlying hardware mechanism and formula are identical.

### Tier 3: Headroom (In-Flight Absorption)

After the Xoff signal fires, the headroom pool absorbs the packets that were already on the wire before the sender could react. How the ASIC allocates this headroom memory comes in two flavors:

**Dedicated Headroom (Traditional)**

Each lossless PG on each port gets its own private headroom reservation, carved out at configuration time. This memory cannot be shared with or borrowed by other ports — in-flight packets are guaranteed a place to land regardless of congestion elsewhere. The downside is cost: on a 64-port 800G switch with two lossless PGs per port, dedicated headroom alone can consume ~48 MB (~30% of a 160 MB ASIC).

**Shared Headroom Pool (SHP)**

PFC pause events are rare and short-lived, and it is statistically unlikely that all ports will be absorbing in-flight packets at the same microsecond. The shared headroom pool exploits this by replacing most of the per-port dedicated headroom with a single, global pool that all lossless PGs draw from on demand. Each PG still retains a small dedicated portion (typically the `xon` size, enough to trigger the resume signal), but the bulk of the headroom budget is pooled. When a PG fires Xoff and needs to absorb in-flight packets, it borrows from the shared headroom pool. Once the buffer drains and Xon fires, that borrowed headroom is released back.

The size of the shared pool is controlled by an **over-subscribe ratio**: the ratio of total dedicated headroom that *would* be needed versus the actual shared pool size. For example, an over-subscribe ratio of 2 means the shared pool is half the size of what full dedicated headroom would require, betting that at most half the ports will need headroom simultaneously.

|                   | Dedicated Headroom                                  | Shared Headroom Pool                                            |
| ----------------- | --------------------------------------------------- | --------------------------------------------------------------- |
| Buffer efficiency | Wastes memory on idle ports                         | Frees buffer for the shared data pool                           |
| Worst-case safety | Guaranteed — every port always has full headroom    | Risk of exhaustion if too many ports pause simultaneously       |
| Typical use case  | Small port-count switches, conservative deployments | Large-scale AI/HPC fabrics where buffer is precious             |

On modern large-scale switches, shared headroom pool can cut headroom consumption from ~30% to ~10–15% of total buffer, freeing tens of megabytes for the shared data pool which directly improves incast absorption and reduces PFC pause frequency.

The total on-chip buffer usage can be summarized as:

$$\text{Total Buffer} = \sum(\text{Per-PG Reserved}) + \text{Shared Pool} + \text{Headroom (Dedicated or Shared)}$$




## The Dangers of PFC: Managing the Lossless Safety Net

While PFC is a mandatory safety net for creating the lossless Ethernet fabrics required by RDMA, it is essentially a reactive "emergency brake." Relying on this brake too heavily introduces severe operational risks. Because PFC pauses traffic at the Priority Group level rather than managing individual data flows, it can cause localized congestion to spiral into network-wide failures.

These challenges fall into two categories: performance degradation and catastrophic network meltdowns.

### Performance Degradation

Even when functioning exactly as designed, PFC can severely degrade network efficiency under heavy load due to its lack of granularity.

**Unfair Bandwidth Allocation**: When a lossless Priority Group on a receiving port gets congested, the switch sends a PFC Pause frame to multiple upstream sending ports. When the congestion clears, all upstream ports are told to resume transmitting simultaneously. If one sending port is handling multiple active flows while another is handling only one, the port with the single flow will consistently grab a disproportionately large share of the bandwidth. This results in unfair resource allocation across the network.

<img src="../pics/pfc-unfairness.webp" width="650"/>

The Scenario (a & b): Multiple flows from different sending ports converge on a single lossless Priority Group on the receiving port. When that PG hits its Xoff threshold, the switch broadcasts a PFC Pause frame, halting both upstream sending interfaces.

The Imbalance (c & d): When the Priority Group drains and the switch sends a resume signal, both upstream ports begin transmitting simultaneously. Because Flow 2 and Flow 3 are sharing a single sending port while Flow 1 has its own, Flow 1 will consistently secure a larger share of the bandwidth. This results in unfair resource allocation across the network.

**Head-of-Line (HoL) Blocking**: Because PFC pauses an entire Priority Group, it cannot distinguish between congested and healthy flows sharing that group. If Flow A is heading to a congested server and Flow B is heading to a completely idle server, a PFC pause triggered by Flow A will trap Flow B behind it. The congested flow at the "head of the line" unfairly blocks the innocent flow trapped behind it.

<img src="../pics/pfc-hol.png" width="550"/>

As illustrated, Flow 1 (red), Flow 2 (blue), and Flow 3 (green) share the same Priority Group. Flow 1 and Flow 3 are destined for a downstream port that becomes congested. When the downstream switch sends a PFC back-pressure signal upstream, it pauses that entire Priority Group. Consequently, Flow 2 which is destined for an entirely different, un-congested port is also paused. The congested flows at the "head of the line" block the unimpeded flows trapped behind them.

### Catastrophic Network Failures

If network congestion is exceptionally severe, or if hardware malfunctions, PFC can trigger cascading failures that paralyze the entire fabric.

**PFC Storms** (Congestion Spreading): This is the domino effect. If a server NIC malfunctions and stops accepting data, the local switch buffer fills up and sends a PAUSE frame to its upstream neighbor. That neighbor's buffer then fills up, pausing its neighbor. This chain reaction cascades outward, flooding the network with endless PFC PAUSE frames and bringing entire sections of the fabric to a standstill.

**PFC Deadlock**: This is a permanent, catastrophic gridlock state usually caused by a transient routing loop. For example: Switch B pauses Switch A, Switch A pauses Switch C, and because of the routing loop, Switch C sends a pause frame back to Switch B. All switches are now stuck in a circular dependency, waiting for the others to release resources. Traffic permanently drops to zero.

<img src="../pics/pfc-deadlock.png" width="300"/>




## The Battlefield: PFC in a Leaf-Spine Topology

The dangers described above are inherent to the PFC protocol itself. However, in a modern data center built on a Leaf-Spine architecture, the physical topology determines exactly how far these side effects can spread. In a standard Leaf-Spine fabric, every server connects to a Top-of-Rack (ToR) leaf switch, and every leaf switch connects to a layer of spine switches via high-speed uplinks.

<img src="../pics/leaf-spine-new.png" width="480"/>

PFC conversations happen strictly hop-by-hop at each of these links:

- Between the server NIC and the ToR.
- Between the ToR and the Spine.

Because it operates hop-by-hop, a pause on any single link can easily propagate to the next. Without deliberate architectural controls, a single malfunctioning server NIC can trigger a chain reaction that propagates from the ToR, up through the spine, and down into every other leaf switch in the fabric. To prevent this, engineers must intentionally decide exactly where in the topology PFC is allowed to operate.


## Architectural Defenses: Blast-Radius Controls

To contain the spread of pause frames, network architects generally choose between two deployment models:

**Model 1: Edge-Only PFC (Lossy Core)**

In this conservative approach, PFC is enabled only on the host-facing ports (from the server NIC to the ToR leaf switch). It is explicitly disabled on the uplinks connecting the ToR to the spine. By intentionally making the spine layer "lossy," engineers create a structural firebreak. If a rogue server triggers a massive pause event, the pause frames can only propagate one hop up to the local ToR. They cannot cross into the spine layer.

The Trade-off: Under extreme congestion, the spine will simply drop RDMA packets rather than propagate back-pressure. This model is common in traditional enterprise data centers where minimizing the blast radius takes priority over guaranteeing absolute losslessness.

**Model 2: Fabric-Wide PFC (Lossless Core)**

Modern AI and High-Performance Computing (HPC) fabrics running RoCEv2 cannot tolerate packet drops at any tier. Therefore, they enable PFC across the entire Leaf-Spine topology (Host-to-ToR and ToR-to-Spine). Disabling PFC on the uplinks would force drops at the ToR during heavy incast events, defeating the fundamental purpose of RoCEv2. Instead, these fabrics accept the risk of a larger blast radius to guarantee end-to-end losslessness.

The Trade-off: Because the risk of a fabric-wide PFC storm is significantly higher, this model strictly requires the automated hardware failsafes described below.



## The PFC Watchdog (The Circuit Breaker)

Because the risks associated with Fabric-Wide PFC are severe, modern data centers never deploy it in isolation. It is always paired with automated safety protocols that detect and break failure conditions before they can cripple the network.

To survive PFC storms and circular routing deadlocks, modern switches utilize an automated hardware agent called the **PFC Watchdog**. The Watchdog acts as a circuit breaker, independently monitoring the state of every lossless Priority Group on the switch. It operates in three distinct phases:

- **Detection**: If the Watchdog notices that a Priority Group's transmit queue has been stuck in a continuous paused state for an abnormally long period (e.g., 100 milliseconds), it assumes the fabric is deadlocked and flags a PFC storm.

- **Action**: To break the deadlock, the Watchdog intervenes based on its administrative configuration. The most common and effective action is `drop`. The switch temporarily ignores the upstream PFC back-pressure, intentionally drops the stuck packets to physically drain the Priority Group, and shatters the circular dependency. Alternatively, it can be configured to simply `alert` the operator or blindly `forward` the traffic.

- **Restoration**: After a brief, configurable recovery window (e.g., 200 milliseconds), the Watchdog relinquishes control and re-enables normal PFC operations. If the root cause (such as an underlying routing loop) persists, the Watchdog will simply catch the ensuing deadlock and break it again.


> While the Watchdog breaks storms after they happen, modern fabrics also rely heavily on end-to-end marking to prevent them from happening in the first place. This mechanism is covered in detail in **[Data Center Quantized Congestion Notification](05_DCQCN.md)**.
