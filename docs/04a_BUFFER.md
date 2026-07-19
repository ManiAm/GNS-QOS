
# Ingress Buffer Architecture

Each ingress port supports up to eight Priority Groups (PG0–PG7), to which internal priorities are mapped. Each PG is configured as either lossy or lossless, and every active PG requires buffer memory to absorb arriving packets. With potentially hundreds of PGs across all ports, the ASIC must allocate its finite on-chip buffer efficiently — ensuring lossless PGs have sufficient depth to trigger PFC before overflow, while lossy PGs have enough space to absorb traffic bursts before dropping.

Statically assigning a fixed, worst-case memory chunk for every PG on every port would rapidly exhaust the on-chip buffer. In practice, most ports are idle at any given microsecond while a few may be heavily congested. Fixed allocation wastes buffer on idle ports while congested ports drop packets because they hit their limits.

To solve this, modern switch ASICs use a shared buffer architecture with three memory tiers:

```
┌──────────────────────────────────────────────┐
│              Buffer Pool                     │
├──────────────┬───────────────┬───────────────┤
│  Guaranteed  │    Shared     │   Headroom    │
│  (per-PG)    │  (dynamic or  │  (lossless    │
│              │   static)     │   PGs only)   │
└──────────────┴───────────────┴───────────────┘
```

## Tier 1: Guaranteed (Reserved) Pool

Every Priority Group receives a small, dedicated block of reserved memory that only it can use. This guaranteed allocation ensures that a PG always has a minimum place to land packets and cannot be entirely starved by congestion on other ports or priority groups.

## Tier 2: Shared Pool

Once a PG fills its reserved space, it begins consuming memory from a global shared buffer pool common across all ports. To prevent a single PG under heavy incast from starving the rest, the switch enforces a threshold that caps how much shared memory any one PG may claim. Two threshold modes are available:

- **Static threshold**: A fixed byte limit. The PG can consume up to a configured number of bytes from the shared pool, regardless of how much free space is available. Simple but inflexible — idle buffer capacity cannot be reclaimed by congested ports.

- **Dynamic threshold**: The limit adjusts in real time based on the remaining free buffer, governed by a parameter called `alpha` ($\alpha$). This is the dominant mode in modern deployments and is detailed below.

**Dynamic Thresholds (Alpha)**

Alpha determines the maximum fraction of the currently available shared buffer that a single PG may claim:

$$\text{Max Shared Limit} = \alpha \times \text{Remaining Shared Buffer}$$

Because "remaining shared buffer" changes in real time as other ports consume or release memory, the limit is dynamic. A port experiencing a burst can temporarily borrow deep into the shared pool if other ports are idle. As those idle ports begin receiving traffic, the available shared buffer shrinks, automatically tightening the limit for the congested port.

- **High Alpha**: The PG may claim a large multiple of the remaining buffer, allowing it to absorb massive bursts. Standard for lossless RoCE classes (e.g., PG3) to prevent unnecessary PFC pauses.

- **Low Alpha**: The PG is restricted to a small share. Typical for lossy traffic classes (e.g., PG0 for standard TCP) where occasional drops are acceptable.

For a lossless PG, occupancy grows into the shared pool until it reaches the Xoff threshold, at which point the switch fires a PFC PAUSE frame. For a lossy PG, packets exceeding its shared limit is simply dropped — no PAUSE frame is generated.

> **Naming note**: Buffer management alpha is unrelated to [DCQCN alpha](./05_DCQCN.md#measuring-congestion-severity-the-alpha-alpha-parameter), which serves as a congestion severity score in the sender's rate control algorithm. They share a Greek letter but operate in entirely different domains — one in the switch ASIC's memory controller, the other in the NIC's congestion control state machine.

**Alpha Values and the Power-of-2 Convention**

The alpha multiplier is not an arbitrary real number. Switch MMU hardware implements it as a power of 2, giving a discrete set of values from very restrictive (alpha < 1) to effectively unlimited (alpha >> 1). In the [SAI (Switch Abstraction Interface)](https://github.com/opencomputeproject/SAI) used by SONiC, alpha is configured through an integer exponent called `dynamic_th`:

$$\alpha = 2^{\,n}$$

Where $n$ is the configured `dynamic_th` value (range: **−8 to +8**), yielding alpha values from 1/256 to 256:

| `dynamic_th` | Alpha (multiplier) | Effect on PG |
|--------------|--------------------|-----------------------------------------|
| −8           | 1/256              | Near-zero shared borrowing — PG is almost entirely confined to its reserved pool |
| −3           | 1/8                | Heavily restricted — can only claim 12.5% of remaining free buffer |
| 0            | 1                  | Balanced — PG can claim up to 1× the remaining free buffer |
| 1            | 2                  | Moderate — typical for lossy traffic classes (e.g., PG0) |
| 3            | 8                  | Generous — PG can claim up to 8× remaining free buffer |
| 8            | 256                | Effectively unlimited — PG can absorb the entire shared pool. Typical for lossless RoCE (e.g., PG3) |

When alpha > 1, the formula produces a threshold larger than the remaining free buffer. This does not mean the PG can exceed physical memory — the ASIC enforces the hard limit of total shared pool size. A high alpha simply means "do not artificially restrict this PG; let it consume whatever is physically available before firing PFC."

When alpha < 1, the PG is restricted to a fraction of the free space. Multiple low-alpha PGs sharing the same pool naturally divide it among themselves: if 8 PGs each have alpha = 1/8, each can claim at most 12.5% of the free space, leaving room for all simultaneously.

This exponent-based representation is standard across SAI-compliant implementations (Broadcom Trident/Tomahawk, NVIDIA Spectrum, Intel Tofino). Vendor-native CLIs may display the value differently — for example, as a raw multiplier or a named profile — but the underlying hardware mechanism is identical.

## Tier 3: Headroom Allocation

Headroom applies exclusively to lossless Priority Groups — lossy PGs have no need for it, since they simply drop packets when their buffer limit is reached. The headroom must be physically reserved on the ASIC to absorb packets that are in flight after a PFC PAUSE frame has been sent (see [PFC Thresholds](04_PFC.md#pfc-thresholds-managing-the-buffer) for the Xoff/Xon mechanics). Two allocation strategies exist:

**Dedicated Headroom (Traditional)**

Each lossless PG on each port gets its own private headroom reservation, carved out at configuration time. This memory cannot be shared with or borrowed by other ports — in-flight packets are guaranteed a place to land regardless of congestion elsewhere. The downside is cost: on a 64-port 800G switch with two lossless PGs per port, dedicated headroom alone can consume ~48 MB (~30% of a 160 MB ASIC).

**Shared Headroom Pool (SHP)**

PFC pause events are rare and short-lived, and it is statistically unlikely that all ports will absorb in-flight packets at the same microsecond. The shared headroom pool replaces most per-port dedicated headroom with a single global pool that all lossless PGs draw from on demand. Each PG retains a small dedicated portion (typically the `xon` size, enough to trigger the resume signal), but the bulk of the headroom budget is pooled. When a PG fires Xoff and needs to absorb in-flight packets, it borrows from the shared headroom pool. Once the buffer drains and Xon fires, that borrowed headroom is released back.

The pool size is controlled by an **over-subscribe ratio**: the ratio of total dedicated headroom that would be needed versus the actual shared pool size. An over-subscribe ratio of 2 means the shared pool is half the size of full dedicated headroom, betting that at most half the ports will need headroom simultaneously.

|                   | Dedicated Headroom                                  | Shared Headroom Pool                                            |
| ----------------- | --------------------------------------------------- | --------------------------------------------------------------- |
| Buffer efficiency | Wastes memory on idle ports                         | Frees buffer for the shared data pool                           |
| Worst-case safety | Guaranteed — every port always has full headroom    | Risk of exhaustion if too many ports pause simultaneously       |
| Typical use case  | Small port-count switches, conservative deployments | Large-scale AI/HPC fabrics where buffer is precious             |

On modern large-scale switches, shared headroom pool can reduce headroom consumption from ~30% to ~10–15% of total buffer, freeing tens of megabytes for the shared data pool — directly improving incast absorption and reducing PFC pause frequency.

The total on-chip buffer usage is:

$$\text{Total Buffer} = \sum(\text{Per-PG Reserved}) + \text{Shared Pool} + \text{Headroom (Dedicated or Shared)}$$
