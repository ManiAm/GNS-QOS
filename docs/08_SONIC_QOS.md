# Configuring QoS on SONiC

This document is a step-by-step guide to configuring Quality of Service on a SONiC switch. It starts from a clean switch with no QoS configuration loaded and walks through generating the defaults, inspecting them, and customizing the pipeline.

> **Scope:** This guide targets SONiC's default configuration model, which uses JSON templates rendered by `sonic-cfggen` and loaded into Redis via `config qos reload`. The examples use the default "AZURE" QoS map profile, which is the standard profile shipped with SONiC. Platform-specific parameters (buffer pool sizes, headroom values) vary by ASIC; the values shown here are representative of Memory-Table (MT) and Memory-Memory (MM) Memory Models.


## Prerequisites

Before configuring QoS, the switch must be in a working state:

1. **SONiC is installed and booted.** You can SSH into the switch and run `show version` to confirm the build.

2. **Interfaces are up.** Verify with `show interfaces status`. At minimum, the ports you intend to configure must show `up` in the Admin and Oper columns.

3. **Basic L3 reachability exists.** Assign IP addresses to the interfaces and confirm end-to-end ping between the switch and connected hosts. QoS configuration modifies *how* traffic is treated, not *whether* it is forwarded.

```bash
show version
show interfaces status
```


## How SONiC Stores QoS Configuration

SONiC does not use flat configuration files like traditional network operating systems. Instead, all configuration lives in a Redis database called **CONFIG_DB**. The QoS pipeline is defined by a set of interrelated tables within this database.

CONFIG_DB is Redis database index **4**. There are two equivalent CLI interfaces to access it:

```bash
# SONiC wrapper — references the database by name
sonic-db-cli CONFIG_DB HGETALL "DSCP_TO_TC_MAP|AZURE"

# Raw Redis — references the database by index (-n 4)
redis-cli -n 4 HGETALL "DSCP_TO_TC_MAP|AZURE"
```

Both commands return the same result. Any Redis operation (`HGETALL`, `HSET`, `HGET`, etc.) works with either interface.

> **Formatting tip:** `HGETALL` returns key-value pairs as alternating lines, which is hard to read. Pipe through `paste - -` to join each pair into a tab-separated row, and `sort -n` to sort numerically:
>
> ```bash
> redis-cli -n 4 HGETALL "DSCP_TO_TC_MAP|AZURE" | paste - - | sort -n
> ```

The key format is `TABLE_NAME|ENTRY_NAME`, where:

- **`DSCP_TO_TC_MAP`** is the **table** — the type of configuration (in this case, the DSCP-to-TC mapping table).
- **`AZURE`** is the **entry name** (also called the profile or map name) — a named instance within that table. A single table can hold multiple named profiles, but SONiC ships with one default profile called `AZURE`. The name is inherited from SONiC's origins as the network OS for Microsoft Azure's data center switches.

Some tables use a different key format. For per-port tables, the entry name includes the port and optionally a queue or Priority Group index:

```
PORT_QOS_MAP|Ethernet0      →  per-port QoS bindings for Ethernet0
QUEUE|Ethernet0|3            →  queue 3 policy on Ethernet0
BUFFER_PG|Ethernet0|3-4      →  buffer profile for PG 3–4 on Ethernet0
```

> **Important:** On a freshly installed switch, these tables are **empty**. SONiC stores its QoS defaults in Jinja2 templates on disk, not in Redis. You must explicitly run `config qos reload` (Step 1) to render those templates and populate CONFIG_DB. Until you do, any `HGETALL` query against these keys returns an empty result.

### The QoS Tables

The tables are organized into four functional groups. Each group corresponds to a stage of the DiffServ pipeline.

**Group 1 — Classification Maps** (What class does this packet belong to?)

These tables define the named mapping profiles that convert packet header fields into the switch's internal Traffic Class (TC). The TC is the central pivot point — it drives both the ingress buffer path and the egress queue path.

| CONFIG_DB Table             | What It Controls |
| --------------------------- | ---------------- |
| `DSCP_TO_TC_MAP`            | Maps each of the 64 DSCP values to a Traffic Class (TC 0–7) |
| `TC_TO_PRIORITY_GROUP_MAP`  | Maps each TC to an ingress Priority Group (PG) for buffer allocation |
| `TC_TO_QUEUE_MAP`           | Maps each TC to a physical egress queue |
| `MAP_PFC_PRIORITY_TO_QUEUE` | Maps PFC priority indices to egress queue indices |

**Group 2 — Buffer Management** (How much memory does each port and priority get?)

These tables form a hierarchy: pools define the total memory available, profiles define how individual consumers draw from pools, and the PG/queue binding tables assign profiles to specific ports. Buffer configuration must be established before PFC, because PFC headroom is a buffer reservation.

| CONFIG_DB Table  | What It Controls |
| ---------------- | ---------------- |
| `BUFFER_POOL`    | Defines ingress/egress memory pools (total size, type, static or dynamic mode) |
| `BUFFER_PROFILE` | Defines buffer profiles (pool reference, reserved size, dynamic threshold alpha) |
| `BUFFER_PG`      | Binds a buffer profile to a specific port and ingress Priority Group range |
| `BUFFER_QUEUE`   | Binds a buffer profile to a specific port and egress queue range |
| `CABLE_LENGTH`   | Per-port cable length used to compute PFC headroom (round-trip delay) |

**Group 3 — Egress Scheduling and Congestion Management** (How are queues drained, and what happens when they fill?)

These tables control the egress behavior: how bandwidth is divided among queues (scheduling) and how the switch responds to congestion (WRED/ECN). The `QUEUE` table is the join point — it binds a scheduler and a WRED profile to each port-queue pair.

| CONFIG_DB Table | What It Controls |
| --------------- | ---------------- |
| `SCHEDULER`     | Defines scheduler objects with type (DWRR or STRICT) and weight |
| `WRED_PROFILE`  | Defines per-color K_min, K_max, P_max thresholds and ECN mode |
| `QUEUE`         | Binds a scheduler and/or WRED profile to a specific port-queue pair |

**Group 4 — Per-Port Binding** (Gluing everything together)

This single table is the top-level configuration point for each physical port. It references the named map profiles from Group 1 and specifies which priorities have PFC enabled.

| CONFIG_DB Table | What It Controls |
| --------------- | ---------------- |
| `PORT_QOS_MAP`  | Associates each port with named map profiles and sets PFC-enabled priorities |

### How These Tables Connect

The following diagram traces the data flow. A packet arriving on an ingress port is classified by the maps in Group 1, buffered through the Group 2 hierarchy, crosses the fabric, and is scheduled out by the Group 3 policies:

```text
                     CONFIG_DB Table Relationships
                     ─────────────────────────────

                         Group 1                         Group 2
                     (Classification)              (Buffer Management)

  DSCP_TO_TC_MAP ──► TC_TO_PRIORITY_GROUP_MAP ──► BUFFER_PG ──► BUFFER_PROFILE ──► BUFFER_POOL
       │                                                                (ingress)
       │
       └──────────► TC_TO_QUEUE_MAP ──► QUEUE ──► SCHEDULER          Group 3
                                          │                    (Scheduling + AQM)
                                          ├──► WRED_PROFILE
                                          └──► BUFFER_QUEUE ──► BUFFER_PROFILE ──► BUFFER_POOL
                                                                        (egress)

  PORT_QOS_MAP (Group 4):  binds all named maps to each physical port
                           also sets pfc_enable (which priorities trigger PFC)
```


## Step 1 — Load the Default QoS Configuration

As noted above, the QoS tables start empty. The default configuration exists only as Jinja2 templates on disk (`qos.json.j2` and `buffers.json.j2`). You must render and load these templates before any QoS behavior takes effect.

### Preview the Defaults Before Applying

Before writing anything to CONFIG_DB, preview the rendered QoS and buffer configuration to verify the templates produce valid output.

SONiC provides a built-in `--dry_run` option for this:

```bash
sudo config qos reload --dry_run /tmp/qos_output.json
```

However, this has a known bug. When `--dry_run` is set, the code strips the `-d` flag from the internal `sonic-cfggen` call. Without `-d`, CONFIG_DB variables (`DEVICE_METADATA`, `PORT`, `CABLE_LENGTH`, etc.) are not loaded into the template context, and the templates fail with:

```
jinja2.exceptions.UndefinedError: 'DEVICE_METADATA' is undefined
```

To work around this, run `sonic-cfggen` directly with the `-d` flag and `--print-data` instead of `--write-to-db`. This renders the templates using live CONFIG_DB data and writes the result to a file without modifying the database:

```bash
HWSKU_PATH=$(python3 -c "from sonic_py_common import device_info; print(device_info.get_paths_to_platform_and_hwsku_dirs()[1])")

sonic-cfggen -d \
  -t ${HWSKU_PATH}/buffers.json.j2,/tmp/cfg_buffer.json \
  -t ${HWSKU_PATH}/qos.json.j2,/tmp/cfg_qos.json \
  -y /etc/sonic/sonic_version.yml

sonic-cfggen -j /tmp/cfg_buffer.json -j /tmp/cfg_qos.json \
  --print-data | python3 -m json.tool --sort-keys > /tmp/qos_preview.json
```

Review the output to confirm the DSCP maps, scheduler weights, WRED thresholds, and buffer pool sizes look correct before proceeding.

### Apply the Default Configuration

Once you are satisfied with the preview, apply the configuration:

```bash
sudo config qos reload
```

This command performs the following sequence:

1. Clears all existing QoS-related tables from CONFIG_DB (DSCP maps, TC maps, schedulers, WRED profiles, queues, buffers).
2. Runs `sonic-cfggen -d` with the device's `qos.json.j2` and `buffers.json.j2` templates, loading CONFIG_DB context (device metadata, port definitions, etc.) for template rendering.
3. Writes the rendered JSON into CONFIG_DB.
4. The `swss` container (specifically `QosOrch` and `BufferOrch`) detects the changes and programs the ASIC via SAI.

After this command completes, all QoS tables are populated with the platform's default AZURE profile and you can begin inspecting and customizing them.

> **Warning:** `config qos reload` is destructive — it replaces all QoS tables. If you run it again later, any manual changes made via `redis-cli` or `sonic-db-cli` since the last reload will be overwritten. Always customize the configuration *after* loading the defaults, and use `config save -y` to persist your changes to disk.


## Step 2 — Inspect the Default Configuration

With the defaults loaded, examine each stage of the QoS pipeline to understand the baseline before making changes.

### DSCP-to-TC Map

```bash
redis-cli -n 4 HGETALL "DSCP_TO_TC_MAP|AZURE" | paste - - | sort -n
```

This shows the full mapping of all 64 DSCP values to Traffic Classes. The default AZURE profile maps traffic as follows:

| DSCP Values | Traffic Class | Typical Use |
| ----------- | ------------- | ----------- |
| 0 (CS0)     | TC 1          | Default / Best Effort |
| 3           | TC 3          | Lossless (RoCEv2 data) |
| 4           | TC 4          | Lossless (reserved) |
| 5           | TC 2          | Reserved |
| 8 (CS1)     | TC 0          | Background / Scavenger |
| 46 (EF)     | TC 5          | Voice / latency-sensitive |
| 48 (CS6)    | TC 6          | Network control (BGP, OSPF) |
| All others  | TC 1          | Best Effort |

> **Note on DSCP 3 and 4:** These are non-standard DSCP values chosen by the AZURE profile specifically for RoCEv2 traffic. They do not correspond to any IETF-defined Per-Hop Behavior. If your NIC firmware uses a different DSCP for RoCEv2 (commonly DSCP 26 / AF31), you must update this map accordingly (see Step 3).

> **Why DSCP 0 maps to TC 1 (not TC 0):** TC 0 is assigned to DSCP 8 (CS1) and receives the lowest scheduler weight, making it a scavenger class for bulk background traffic. Default traffic (DSCP 0) is placed in TC 1, which receives a higher weight.

### TC-to-PG Map (Ingress Path)

```bash
redis-cli -n 4 HGETALL "TC_TO_PRIORITY_GROUP_MAP|AZURE" | paste - - | sort -n
```

After classification, ingress buffering is the first decision. Each TC maps to a Priority Group, which determines the ingress buffer profile:

| TC | Priority Group | Lossless?  |
| -- | -------------- | ---------- |
| 0  | PG 0           | No (lossy) |
| 1  | PG 0           | No (lossy) |
| 2  | PG 0           | No (lossy) |
| 3  | PG 3           | Yes        |
| 4  | PG 4           | Yes        |
| 5  | PG 0           | No (lossy) |
| 6  | PG 0           | No (lossy) |
| 7  | PG 7           | No (lossy) |

TC 3 and TC 4 are the lossless traffic classes — they map to dedicated Priority Groups (PG 3 and PG 4), separating them from lossy traffic. All other TCs share PG 0, which is lossy (tail-drop only). Note that while this map *classifies* TC 3 and TC 4 into their own PGs, the default buffer template does not actually allocate ingress headroom for PG 3 and PG 4 (see the "What is missing from the defaults" note in the Buffer Configuration section below). Headroom must be configured separately for PFC to function.

### TC-to-Queue Map (Egress Path)

```bash
redis-cli -n 4 HGETALL "TC_TO_QUEUE_MAP|AZURE" | paste - - | sort -n
```

On the egress side, each TC maps to a physical output queue. The default is a 1:1 identity mapping: TC 0 → Queue 0, TC 1 → Queue 1, ..., TC 7 → Queue 7.

### Scheduler (ETS) Configuration

```bash
redis-cli -n 4 HGETALL "SCHEDULER|scheduler.0" | paste - -
redis-cli -n 4 HGETALL "SCHEDULER|scheduler.1" | paste - -
```

Default schedulers:

| Scheduler     | Type | Weight | Assigned To |
| ------------- | ---- | ------ | ----------- |
| `scheduler.0` | DWRR | 14     | Lossy queues (0, 1, 2, 5, 6) |
| `scheduler.1` | DWRR | 15     | Lossless queues (3, 4) |

Under contention, bandwidth is divided proportionally: lossless queues receive 15/(14+15) ≈ 52% of bandwidth.

> **No strict-priority queues:** The default AZURE profile uses DWRR for all queues — there is no strict-priority (SP) scheduler. This means even network control (TC 6) and voice/EF (TC 5) traffic competes proportionally with best-effort under congestion, rather than preempting it. This is intentional for datacenter RoCEv2 workloads where lossless delivery via PFC matters more than latency differentiation. If your deployment requires SP for certain queues (e.g., network control), add a new scheduler with `"type": "STRICT"` and reassign those queues to it (see Step 5).

### WRED/ECN Profile

```bash
redis-cli -n 4 HGETALL "WRED_PROFILE|AZURE_LOSSLESS" | paste - -
```

WRED/ECN is applied to egress queues to provide early congestion signaling. The default AZURE_LOSSLESS profile is attached to the lossless queues (3 and 4):

| Parameter                | Value              | Meaning |
| ------------------------ | ------------------ | ------- |
| `wred_green_enable`      | `true`             | WRED enabled for green packets |
| `green_min_threshold`    | `250000` (250 KB)  | K_min — queue depth at which ECN marking begins for green (conforming) traffic |
| `green_max_threshold`    | `2097152` (2 MB)   | K_max — queue depth at which marking probability reaches P_max |
| `green_drop_probability` | `5`                | P_max = 5% marking/drop probability at K_max |
| `wred_yellow_enable`     | `true`             | WRED enabled for yellow packets |
| `yellow_min_threshold`   | `1048576` (1 MB)   | K_min for yellow (rate-exceeding) packets |
| `yellow_max_threshold`   | `2097152` (2 MB)   | K_max for yellow packets |
| `yellow_drop_probability`| `5`                | P_max = 5% for yellow packets |
| `wred_red_enable`        | `true`             | WRED enabled for red packets |
| `red_min_threshold`      | `1048576` (1 MB)   | K_min for red (rate-violating) packets |
| `red_max_threshold`      | `2097152` (2 MB)   | K_max for red packets |
| `red_drop_probability`   | `5`                | P_max = 5% for red packets |
| `ecn`                    | `ecn_all`          | ECN marking enabled for all colors |

### Per-Port QoS Bindings

```bash
redis-cli -n 4 HGETALL "PORT_QOS_MAP|Ethernet0" | paste - -
```

This shows which map profiles are bound to the port and which priorities have PFC enabled:

```
dscp_to_tc_map   → AZURE
tc_to_queue_map  → AZURE
tc_to_pg_map     → AZURE
pfc_to_queue_map → AZURE
pfc_enable       → 3,4
pfcwd_sw_enable  → 3,4
```

The `pfc_enable` field lists the priority indices on which PFC PAUSE frames are generated and honored. The default enables PFC on priorities 3 and 4, corresponding to the lossless TCs.

### Buffer Configuration

```bash
show buffer configuration
```

This displays the buffer pools, buffer profiles, and per-port PG/queue buffer assignments. The output has three sections: **Pools**, **Profiles**, and per-port assignments. Understanding the relationship between these concepts is essential before customizing buffers.




#### What is a Buffer Pool?

Every packet that enters or exits a switch port is stored temporarily in on-chip memory (packet buffer) while the forwarding pipeline processes it. A **buffer pool** is a named region of that memory reserved for a specific direction and purpose.

Think of the switch's total packet buffer as a building, and each pool as a floor in that building. Traffic can only use the floor it is assigned to — it cannot spill into another floor.

> **Dual admission:** A packet must be admitted to *both* an ingress pool and an egress pool to be forwarded. When a packet arrives on a port, the ingress admission check decides whether there is room to accept it. Simultaneously, the egress admission check decides whether the destination output queue has room. If *either* check fails, the packet is dropped (or PFC is triggered for lossless priorities). This means buffer tuning on one side alone is not sufficient — both ingress and egress pools must be sized and configured correctly.

The default configuration defines three pools:

| Pool                    | Direction | Mode    | Size     | Purpose |
| ----------------------- | --------- | ------- | -------- | ------- |
| `ingress_lossless_pool` | Ingress   | Dynamic | ~10.4 MB | Holds incoming packets for all ingress priorities. Also has an `xoff` headroom reserve (~4 MB) for lossless PFC operation. |
| `egress_lossy_pool`     | Egress    | Dynamic | ~8.8 MB  | Holds outgoing packets for lossy queues. When full, excess packets are tail-dropped. |
| `egress_lossless_pool`  | Egress    | Static  | ~15.2 MB | Holds outgoing packets for lossless queues (PFC-protected priorities 3 and 4). Sized large enough to absorb bursts without dropping while PFC backpressure propagates. |

#### Inside a Buffer Pool: Three Memory Regions

Each pool's memory is internally divided into three regions. These correspond to the three buffer tiers described in [Ingress Buffer Architecture](04a_BUFFER.md):

1. **Guaranteed** — A small reserved allocation per Priority Group (ingress) or queue (egress), configured via the buffer profile's `size` field. Each consumer is guaranteed this memory regardless of congestion.

2. **Shared** — The remaining pool memory after guaranteed allocations. Consumers borrow from this region up to a limit set by their profile's threshold (`dynamic_th` or `static_th`).

3. **Headroom** — (Ingress lossless pool only.) Reserved for packets in flight after a PFC PAUSE frame has been sent. Configured via the pool's `xoff` field.

**Key pool fields:**

- **`type`** — `ingress` or `egress`. Determines whether the pool serves incoming or outgoing traffic.
- **`mode`** — `static` or `dynamic`. Controls how the shared region is divided among consumers (see profile thresholds below).
- **`size`** — Total memory allocated to the pool, in bytes (includes all three regions).
- **`xoff`** — (Ingress lossless pool only.) Size of the headroom region in bytes.

#### What is a Buffer Profile?

A buffer pool is a block of memory, but it says nothing about *who* can use it or *how much* each consumer can take. That is the job of a **buffer profile**.

A buffer profile is a policy that says: "Consumers attached to me may draw from *this* pool, up to *this* threshold." You can think of a pool as a shared bank account and a profile as a spending policy card — each card is linked to one account and has its own spending limit.

Each profile has these fields:

- **`pool`** — Which pool this profile draws memory from.
- **`size`** — A guaranteed (reserved) allocation in bytes. This memory is exclusively reserved for each consumer that uses this profile. Even when the pool is congested, each consumer is guaranteed at least this much. A `size` of `0` means no guaranteed reservation; the consumer relies entirely on the shared portion of the pool.

The remaining fields depend on the pool's mode:

**Static mode** (`egress_lossless_pool`):

- **`static_th`** — A hard byte limit. Each consumer can use up to `size + static_th` bytes total. Once this threshold is reached, additional packets are dropped (or PFC is triggered). In the default config, `static_th` equals the entire pool size, meaning each lossless queue can potentially use the full pool (since lossless traffic is PFC-protected, not dropped).

**Dynamic mode** (`ingress_lossless_pool`, `egress_lossy_pool`):

- **`dynamic_th`** — An alpha exponent (integer, range −8 to 8) that controls how much of the pool's *currently available* shared space a consumer may use. The limit adjusts in real time as the pool fills and drains — a higher exponent allows more aggressive borrowing, while a lower exponent restricts the consumer to a smaller fraction. For the full alpha formula, value table, and behavioral details, see [Dynamic Thresholds](04a_BUFFER.md#tier-2-shared-pool).

**Lossless ingress profiles** have additional fields that control PFC PAUSE behavior (see [PFC Thresholds](04_PFC.md#pfc-thresholds-managing-the-buffer) for conceptual background):

- **`xoff`** — XOFF threshold in bytes. Sized for round-trip propagation delay (cable length, port speed) plus sender response time.
- **`xon`** — XON threshold in bytes. The gap between `xoff` and `xon` provides hysteresis to prevent rapid pause/resume oscillation.
- **`xon_offset`** — Alternative way to express the XON point as `headroom_size - xon_offset`. When both are set, the effective XON threshold is `max(xon, size - xon_offset)`.
- **`size`** — Total headroom reserved for the PG. Must be at least `xoff` plus a safety margin.

These values depend on port speed and cable length. SONiC provides a lookup table at `pg_profile_lookup.ini` in the hwsku directory with pre-calculated values:

```bash
cat $(python3 -c "from sonic_py_common import device_info; print(device_info.get_paths_to_platform_and_hwsku_dirs()[1])")/pg_profile_lookup.ini
```

Example entries (from this switch):

```
# speed cable size    xon  xoff    threshold xon_offset
 100000  5m   1248    2288 165568  0         2288
 100000  40m  1248    2288 177632  0         2288
 100000  300m 1248    2288 268736  0         2288
```

A 100 Gbps port with a 5m cable needs 165568 bytes of XOFF headroom, while a 300m cable needs 268736 bytes — longer cables mean more in-flight data during the PAUSE round-trip.

The default `config qos reload` defines three profiles (all lossy — no lossless ingress profile):

| Profile | Pool | Size | Threshold | Used By |
| ------- | ---- | ---- | --------- | ------- |
| `ingress_lossy_profile` | `ingress_lossless_pool` | 0 | `dynamic_th` = 3 | Lossy ingress PGs — no guaranteed reservation, can borrow up to 8× the available shared buffer. Draws from the lossless pool because SONiC uses a single ingress pool for all traffic. |
| `egress_lossy_profile` | `egress_lossy_pool` | 1518 | `dynamic_th` = 3 | Lossy egress queues — 1518 bytes (one max-size Ethernet frame) guaranteed per queue, plus dynamic sharing. |
| `egress_lossless_profile` | `egress_lossless_pool` | 1518 | `static_th` = 15982720 | Lossless egress queues — one frame guaranteed, static threshold set to the full pool size. |

> **No lossless ingress profile in the defaults:** The default template does not create a lossless ingress buffer profile with `xoff`/`xon` thresholds, and does not create `BUFFER_PG` entries for PG 3 or PG 4. This means PFC cannot function out of the box — you must create the profile and PG bindings manually (see Step 4).

#### How Pools, Profiles, and Ports Fit Together

The relationship flows like this:

```
Pool  ←  Profile  ←  Port PG / Queue
```

1. A **pool** allocates a chunk of switch memory.
2. A **profile** references one pool and defines the admission policy (thresholds).
3. Each **port's Priority Group (PG)** or **queue** is bound to one profile, which determines where its packets are buffered and how much space it can use.

The binding is stored in two CONFIG_DB tables: `BUFFER_PG` (ingress) and `BUFFER_QUEUE` (egress). To inspect them:

```bash
# All ingress PG → profile bindings
redis-cli -n 4 KEYS 'BUFFER_PG|*' | sort -V | while IFS= read -r k; do
  printf '%s -> %s\n' "$k" "$(redis-cli -n 4 HGET "$k" profile < /dev/null)"
done

# All egress queue → profile bindings
redis-cli -n 4 KEYS 'BUFFER_QUEUE|*' | sort -V | while IFS= read -r k; do
  printf '%s -> %s\n' "$k" "$(redis-cli -n 4 HGET "$k" profile < /dev/null)"
done
```

> **Note:** The inner `redis-cli` call must have its stdin redirected (`< /dev/null`), otherwise it consumes the piped KEYS output and the loop processes only the first key.

In the default AZURE configuration, **all 32 ports have identical assignments**. There is no per-port differentiation:

**Ingress (BUFFER_PG):**

| PG | Profile | Pool | Guaranteed | Threshold |
| -- | ------- | ---- | ---------- | --------- |
| 0  | `ingress_lossy_profile` | `ingress_lossless_pool` | 0 bytes | `dynamic_th` = 3 |

**Egress (BUFFER_QUEUE):**

| Queues | Profile | Pool | Guaranteed | Threshold |
| ------ | ------- | ---- | ---------- | --------- |
| 0-2    | `egress_lossy_profile` | `egress_lossy_pool` | 1518 B | `dynamic_th` = 3 |
| 3-4    | `egress_lossless_profile` | `egress_lossless_pool` | 1518 B | `static_th` = 15982720 |
| 5-6    | `egress_lossy_profile` | `egress_lossy_pool` | 1518 B | `dynamic_th` = 3 |

> **What is missing from the defaults:** Only PG 0 is configured on the ingress side. PG 3 and PG 4 (the lossless priority groups) have **no `BUFFER_PG` entry**, which means no ingress headroom is allocated for them. Queue 7 has **no `BUFFER_QUEUE` entry** on the egress side. This means `config qos reload` sets up the QoS classification maps (DSCP → TC 3/4 → PG 3/4) and enables PFC on priorities 3 and 4, but the buffer template does not actually allocate the lossless ingress headroom needed for PFC to function correctly. To enable working PFC, you must add `BUFFER_PG` entries for PG 3 and PG 4 with a lossless ingress profile (see Step 4).

> **Why does the ingress lossy profile use the "lossless" pool?** SONiC's Memory-Table buffer model typically has a single ingress pool. The pool is named `ingress_lossless_pool` because it contains the lossless headroom (`xoff` reserve), but all ingress traffic — lossy and lossless — draws from it. The distinction between lossy and lossless at the ingress is handled by PG-level profile thresholds and PFC, not by separate pools.




## Step 3 — Customize the DSCP-to-TC Map

If your deployment uses different DSCP markings than the AZURE defaults, update the `DSCP_TO_TC_MAP`. For example, if your RoCEv2 NICs mark data traffic with DSCP 26 (AF31) instead of DSCP 3, map DSCP 26 to the lossless TC 3:

```bash
sonic-db-cli CONFIG_DB HSET "DSCP_TO_TC_MAP|AZURE" "26" "3"
```

Verify the change:

```bash
redis-cli -n 4 HGET "DSCP_TO_TC_MAP|AZURE" "26"
# Expected output: 3
```

> **Persistence:** Changes made directly to CONFIG_DB via `redis-cli` or `sonic-db-cli` are written to `/etc/sonic/config_db.json` only on the next `config save`. If the switch reboots before a save, the changes are lost. Running `config qos reload` also overwrites these changes — always customize *after* loading the defaults, not before.


## Step 4 — Customize Buffers

Buffer configuration determines how much switch memory is allocated to each port and priority. This step must precede PFC configuration because PFC headroom — the buffer space reserved to absorb in-flight packets between a PAUSE frame and the sender stopping — is a buffer allocation. Refer to the pool/profile explanation in Step 2 for background on how pools, profiles, and thresholds work.

### Cable Length Configuration

PFC headroom is calculated from the round-trip propagation delay, which is a function of cable length. The `CABLE_LENGTH` table stores the cable length per port. Incorrect cable lengths cause headroom to be undersized (leading to drops despite PFC being enabled) or oversized (wasting buffer memory).

```bash
# View current cable lengths
redis-cli -n 4 HGETALL "CABLE_LENGTH|AZURE" | paste - - | sort

# Update cable length for a port
sonic-db-cli CONFIG_DB HSET "CABLE_LENGTH|AZURE" "Ethernet0" "5m"
```

After changing cable lengths, reload the QoS configuration to recalculate headroom:

```bash
sudo config qos reload
```

> **Note:** Running `config qos reload` here will also overwrite any other manual QoS customizations you have made (e.g., DSCP map changes from Step 3). If you need to change cable lengths *and* customize other tables, change the cable lengths first, reload, and then apply the other customizations.

### Create Lossless Ingress Profiles (Required for PFC)

As noted in Step 2, the default template does not create lossless ingress buffer profiles or `BUFFER_PG` entries for PG 3 and PG 4. Without these, PFC PAUSE frames have no headroom to absorb in-flight packets, making lossless behavior non-functional. You must create them manually.

**1. Look up the correct thresholds** from `pg_profile_lookup.ini` based on port speed and cable length:

```bash
cat $(python3 -c "from sonic_py_common import device_info; print(device_info.get_paths_to_platform_and_hwsku_dirs()[1])")/pg_profile_lookup.ini
```

For example, for a 100 Gbps port with a 5m cable, the table gives: `size=1248`, `xon=2288`, `xoff=165568`, `threshold=0`, `xon_offset=2288`.

**2. Create the lossless buffer profile:**

```bash
sonic-db-cli CONFIG_DB HMSET "BUFFER_PROFILE|pg_lossless_100000_5m_profile" \
  "pool" "ingress_lossless_pool" \
  "size" "1248" \
  "xon" "2288" \
  "xoff" "165568" \
  "dynamic_th" "0" \
  "xon_offset" "2288"
```

The profile name convention `pg_lossless_<speed>_<cable>_profile` is not enforced, but following it makes the configuration self-documenting.

**3. Bind PG 3 and PG 4 to the lossless profile on each port:**

```bash
sonic-db-cli CONFIG_DB HSET "BUFFER_PG|Ethernet0|3-4" "profile" "pg_lossless_100000_5m_profile"
```

Repeat for every port that needs lossless behavior. To apply to all ports at once:

```bash
for port in $(redis-cli -n 4 KEYS 'PORT|Ethernet*' | sed 's/PORT|//' | sort -V); do
  sonic-db-cli CONFIG_DB HSET "BUFFER_PG|${port}|3-4" "profile" "pg_lossless_100000_5m_profile" < /dev/null
done
```

> **Different speeds or cable lengths:** If ports have different speeds or cable lengths, create separate profiles for each combination and bind accordingly. For example, ports with 40m cables need a `pg_lossless_100000_40m_profile` with `xoff=177632`.

**4. Verify the bindings:**

```bash
redis-cli -n 4 KEYS 'BUFFER_PG|*' | sort -V | while IFS= read -r k; do
  printf '%s -> %s\n' "$k" "$(redis-cli -n 4 HGET "$k" profile < /dev/null)"
done
```

You should now see both PG 0 (lossy) and PG 3-4 (lossless) entries for each port.

### Adjust Buffer Profiles (Dynamic Mode — Mellanox/NVIDIA)

On dynamic-mode platforms, buffer profile names encode the port speed and cable length:

```
pg_lossless_<speed_mbps>_<cable_length>_profile
```

For example, `pg_lossless_100000_40m_profile` is the lossless PG profile for a 100 Gbps port with a 40-meter cable.

To change the dynamic threshold (alpha) for this profile:

```bash
sudo config mmu -p pg_lossless_100000_40m_profile -a 2
```

This sets alpha = 2, meaning the PG may consume up to `2^2 = 4×` the per-port fair share of the shared pool.


## Step 5 — Customize the Scheduler (ETS)

The scheduler determines how egress bandwidth is divided among queues when multiple queues are backlogged simultaneously. SONiC's `SCHEDULER` table defines named scheduler objects, and the `QUEUE` table binds them to specific port-queue pairs.

### Modify Scheduler Weights

```bash
# Set scheduler.0 to DWRR with weight 25 (for lossy queues)
sonic-db-cli CONFIG_DB HSET "SCHEDULER|scheduler.0" "type" "DWRR" "weight" "25"

# Set scheduler.1 to DWRR with weight 75 (for lossless queues)
sonic-db-cli CONFIG_DB HSET "SCHEDULER|scheduler.1" "type" "DWRR" "weight" "75"

# Create a strict priority scheduler for network control traffic
sonic-db-cli CONFIG_DB HSET "SCHEDULER|scheduler.2" "type" "STRICT"
```

With these weights, lossless queues receive 75/(25+75) = 75% of bandwidth under contention.

### Bind Schedulers to Queues

```bash
# Assign strict priority to queue 7 (network control) on Ethernet0
sonic-db-cli CONFIG_DB HSET "QUEUE|Ethernet0|7" "scheduler" "scheduler.2"
```

### Example Queue-to-Scheduler Assignment

| Queue | Scheduler   | Type   | Weight | Traffic |
| ----- | ----------- | ------ | ------ | ------- |
| 0     | scheduler.0 | DWRR   | 25     | Background |
| 1     | scheduler.0 | DWRR   | 25     | Best Effort |
| 3     | scheduler.1 | DWRR   | 75     | RoCEv2 data (lossless) |
| 4     | scheduler.1 | DWRR   | 75     | Reserved lossless |
| 7     | scheduler.2 | STRICT | —      | Network control, CNPs |

> **Design rule:** Only assign Strict Priority to traffic that is both latency-critical **and** inherently low-volume (e.g., BGP keep-alives, CNPs). Placing bulk data in a SP queue starves all DWRR queues beneath it.


## Step 6 — Customize the WRED/ECN Profile

WRED/ECN operates on the egress queues configured in Step 5. When a queue's instantaneous depth exceeds K_min, the switch begins probabilistically marking packets with ECN CE bits (or dropping them if the packet is not ECN-capable). This provides early congestion signals to DCQCN-enabled endpoints before PFC fires.

The default WRED profile uses identical thresholds for all three drop-precedence colors. For a properly tuned RoCEv2 deployment, differentiate the thresholds so that over-budget traffic is marked earlier than conforming traffic:

```bash
# View current ECN configuration
show ecn

# Green (conforming traffic) — mark later, protect longer
sudo ecnconfig -p AZURE_LOSSLESS -gmin 1048576 -gmax 2097152

# Yellow (rate-exceeding traffic) — mark earlier
sudo ecnconfig -p AZURE_LOSSLESS -ymin 524288 -ymax 1048576

# Red (rate-violating traffic) — mark earliest
sudo ecnconfig -p AZURE_LOSSLESS -rmin 262144 -rmax 524288
```

Parameters:

| Flag | Meaning |
| ---- | ------- |
| `-p` | Profile name (e.g., `AZURE_LOSSLESS`) |
| `-gmin` / `-gmax` | Green min/max thresholds in bytes |
| `-ymin` / `-ymax` | Yellow min/max thresholds in bytes |
| `-rmin` / `-rmax` | Red min/max thresholds in bytes |
| `-gdrop` / `-ydrop` / `-rdrop` | Drop/mark probability (%) at K_max for each color |

Verify:

```bash
show ecn
```


## Step 7 — Configure PFC

PFC (Priority-based Flow Control) provides per-priority lossless behavior by pausing the upstream sender when ingress buffer occupancy exceeds the XOFF threshold. PFC depends on buffer headroom being correctly allocated (Step 4): the headroom must absorb all in-flight packets between the PAUSE frame being sent and the sender actually stopping.

### Enable or Disable PFC on a Priority

```bash
# Enable PFC on priority 3 for Ethernet0
sudo config interface pfc priority Ethernet0 3 on

# Disable PFC on priority 4 for Ethernet0
sudo config interface pfc priority Ethernet0 4 off
```

### Verify PFC Configuration and Counters

```bash
# PFC enabled/disabled status per port per priority
show pfc priority

# PFC PAUSE frame TX/RX counters per priority
show pfc counters
```

Under normal operation, PFC TX counters should remain at zero or increment slowly. Rapidly rising PFC TX counters indicate sustained egress congestion on that priority — investigate whether WRED/ECN thresholds are too high or buffer headroom is undersized.


## Step 8 — Configure the PFC Watchdog

The PFC Watchdog detects and recovers from PFC storms — situations where a port is stuck in a perpetual PAUSE state, blocking all traffic on a lossless priority. Without the watchdog, a single malfunctioning NIC or cable can freeze an entire lossless priority across the fabric.

### Start the Watchdog

```bash
# Start on a single port with explicit timers
sudo pfcwd start --action drop Ethernet0 200 --restoration-time 200

# Start on all ports with default timers
sudo pfcwd start_default
```

Parameters:

| Parameter        | Default | Meaning |
| ---------------- | ------- | ------- |
| Detection time   | 200 ms  | How long a queue must be stuck (receiving PFC PAUSE but not draining) before the watchdog triggers |
| Restoration time | 200 ms  | How long the watchdog waits after the storm clears before re-enabling PFC on the affected queue |
| Action           | `drop`  | What to do when a storm is detected — `drop` discards incoming traffic on the stuck queue to break the deadlock; `forward` continues forwarding (use only for debugging) |

### Verify PFC Watchdog

```bash
show pfcwd config
show pfcwd stats
```


## Step 9 — Save the Configuration

After completing your customizations, persist the current CONFIG_DB state to disk so it survives reboots:

```bash
sudo config save -y
```

This writes the full CONFIG_DB contents to `/etc/sonic/config_db.json`, which SONiC loads on every boot.

Without this step, all customizations made in Steps 3–8 exist only in Redis and will be lost on reboot.

> **`config qos reload` vs. `config save`:** These serve opposite purposes. `config qos reload` writes *from templates into* CONFIG_DB (overwriting manual changes). `config save` writes *from CONFIG_DB to disk* (preserving manual changes). The typical workflow is: reload the defaults once (Step 1), customize (Steps 3–8), then save (this step).


## Step 10 — Verify the Pipeline End-to-End

After applying and saving the configuration, verify that each stage of the pipeline is programmed correctly.

### Classification

```bash
redis-cli -n 4 HGETALL "DSCP_TO_TC_MAP|AZURE" | paste - - | sort -n
redis-cli -n 4 HGETALL "PORT_QOS_MAP|Ethernet0" | paste - -
```

### Queuing and Scheduling

```bash
show queue counters Ethernet0
```

The output shows per-queue packet/byte counts and drop counts. Under no congestion, all queues should show zero drops. Packet distribution across queues should match the DSCP → TC → Queue mapping.

### ECN/WRED

```bash
show ecn
show queue wredcounters Ethernet0
```

### PFC and PFC Watchdog

```bash
show pfc priority
show pfc counters
show pfcwd config
show pfcwd stats
```

### Buffers

```bash
show buffer configuration
show buffer pool watermark
show buffer pool persistent-watermark
```


## Step 11 — Generate Traffic and Validate

With the pipeline verified, generate real traffic to confirm QoS behavior under load.

### Basic Throughput Test

Use `iperf3` to generate TCP traffic between two hosts connected to the switch:

```bash
# Server (Host B)
iperf3 -s

# Client (Host A) — mark traffic with DSCP 26 (AF31)
iperf3 -c <Host_B_IP> -t 30 -S 0x68
```

The `-S 0x68` flag sets the ToS byte to `0x68`. Since DSCP occupies the upper 6 bits of the ToS byte, the DSCP value is `0x68 >> 2 = 26` (AF31). While the test runs, monitor the switch:

```bash
watch -n 1 'show queue counters Ethernet<egress_port>'
```

Traffic marked with DSCP 26 should appear in the queue corresponding to TC 3 (the lossless queue, assuming you mapped DSCP 26 → TC 3 in Step 3).

### RoCEv2 Traffic Test

If the connected hosts have RDMA-capable NICs (e.g., Mellanox ConnectX-5/6), use the `perftest` suite:

```bash
# Server
ib_write_bw -d <rdma_device> --report_gbits -D 30

# Client
ib_write_bw -d <rdma_device> --report_gbits -D 30 <Server_IP>
```

While the test runs, verify on the switch:

- **PFC counters** should remain at zero (no congestion) or increment slowly.
- **Queue counters** should show traffic concentrated on the lossless queue.
- **ECN mark counters** should appear if queue depth crosses K_min.

```bash
show pfc counters
show queue counters Ethernet<egress_port>
show queue wredcounters Ethernet<egress_port>
```


## Quick Reference: SONiC QoS CLI Commands

### Configuration Commands

| Command | Purpose |
| ------- | ------- |
| `sudo config qos reload` | Render and load all QoS/buffer config from templates into CONFIG_DB |
| `sudo config qos reload --dry_run <file>` | **Broken** — omits `-d` flag so `sonic-cfggen` cannot load CONFIG_DB context; use `sonic-cfggen -d ... --print-data` instead (see Step 1) |
| `sudo config qos clear` | Remove all QoS configuration from CONFIG_DB |
| `sudo config interface pfc priority <port> <pri> on/off` | Enable/disable PFC on a specific priority |
| `sudo config interface pfc asymmetric <port> on/off` | Enable/disable asymmetric PFC |
| `sudo ecnconfig -p <profile> -gmin/-gmax/-ymin/-ymax/-rmin/-rmax <val>` | Modify WRED/ECN thresholds |
| `sudo config mmu -p <profile> -a <alpha>` | Set dynamic buffer threshold (alpha) |
| `sudo pfcwd start <port> <detect_time> --restoration-time <val>` | Start PFC Watchdog on a port |
| `sudo pfcwd start_default` | Start PFC Watchdog on all ports with defaults |
| `sudo pfcwd stop` | Stop PFC Watchdog |
| `sudo config save -y` | Persist CONFIG_DB to disk |

### Show / Monitoring Commands

| Command | Purpose |
| ------- | ------- |
| `show ecn` | Display WRED/ECN profile configuration |
| `show mmu` | Display buffer profiles and thresholds |
| `show buffer configuration` | Display buffer pools, profiles, PG/queue bindings |
| `show buffer pool watermark` | Display current buffer pool usage watermarks |
| `show pfc priority` | Display PFC enabled/disabled per port per priority |
| `show pfc counters` | Display PFC PAUSE frame TX/RX counters per priority |
| `show pfcwd config` | Display PFC Watchdog configuration |
| `show pfcwd stats` | Display PFC Watchdog detection/recovery events |
| `show queue counters [port]` | Display per-queue packet/byte/drop counters |
| `show queue wredcounters [port]` | Display per-queue WRED drop and ECN mark counters |
| `show interfaces counters` | Display per-port TX/RX and error counters |
