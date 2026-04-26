
# Quality of Service (QoS)

## The Four Key Metrics (What the Network Measures)

Before we can manage traffic, we have to know how to measure it. Every piece of data sent across a network is judged on four criteria:

- **Reliability**: Does the data arrive safely without pieces missing?
- **Delay (Latency)**: How fast does the data travel from point A to point B?
- **Delay Variation (Jitter)**: Is the data arriving smoothly and consistently, or is it arriving in unpredictable bursts?
- **Throughput**: How much total data can the network push through at once (bandwidth)?


## Elastic vs. Inelastic Traffic (How Applications Behave)

Applications care about those four metrics in entirely different ways. Because of this, network traffic is divided into two distinct personalities:

- **Elastic Traffic** (The Perfectionists): Think of web browsing, file downloads (FTP), and emails. These applications are highly adaptable to speed, but they demand 100% reliability. If an email takes an extra second to send, you won't care (low delay sensitivity). But if half the words are missing, the email is useless (high reliability need).

- **Inelastic Traffic** (The Impatient Ones): Think of Zoom calls, VoIP phone systems, and live streaming. These applications are incredibly strict about time. They require very low delay and low jitter. If a packet of your voice arrives half a second late, the conversation is already ruined. At the **playout / codec layer**, they can tolerate **occasional loss** of a media frame (brief glitch) better than long delay.

| Traffic Type | Application          | Reliability Need | Delay Sensitivity | Jitter Sensitivity | Throughput Need |
| ------------ | -------------------- | ---------------- | ----------------- | ------------------ | --------------- |
| Elastic      | FTP                  | High             | Low               | Low                | Medium          |
|              | HTTP                 | High             | Medium            | Low                | Medium          |
| Inelastic    | On-demand Audio      | Low              | Medium            | Medium             | Medium          |
|              | On-demand Video      | Low              | Medium            | Medium             | High            |
|              | Voice over IP (VoIP) | Low              | High              | High               | Low             |
|              | Video Conferencing   | Low              | High              | High               | High            |

## The Problem with the Original Internet

The original internet was built on the End-to-End Principle (Saltzer, Reed & Clark, 1984). This philosophy stated that the network itself should be "dumb" and just forward packets as fast as possible (known as "Best-Effort" delivery). Any "smart" tasks like checking for lost data and resending it were the responsibility of the end devices (your computer or the server).

This was perfect for Elastic traffic. However, as Inelastic traffic (like voice and video) became popular, this "dumb network" model broke down. A network blindly forwarding packets causes bottlenecks, and waiting for an endpoint to realize a voice packet was dropped and request it again takes too much time. Real-time applications simply cannot survive on "Best-Effort" delivery.

## Quality of Service (The Solution)

Because applications could no longer solve these delays on their own, the network itself had to get smarter. Quality of Service (QoS) is the umbrella term for the tools and rules used to give VIP treatment to the traffic that needs it most. It does not create more internet speed; it acts as a traffic cop, ensuring that during a traffic jam, the ambulance (VoIP) gets to use the shoulder of the road while the passenger cars (emails) wait in line. QoS is built in five distinct layers, each answering a specific question:

```
┌─────────────────────────────────────────────────────────────────────┐
│  Layer 1: Architecture                                              │
│  "How do we organize QoS across the network?"                       │
│   → IntServ (per-flow reservations) vs. DiffServ (per-class marks)  │
│   → DiffServ dominates Internet/campus; IntServ/RSVP in niches (MPLS-TE, etc.) │
├─────────────────────────────────────────────────────────────────────┤
│  Layer 2: Marking (Tagging)                                         │
│  "How does a packet declare its priority?"                          │
│   → L3: DSCP (6-bit, 64 classes) — in the IP header                 │
│   → L2: PCP  (3-bit, 8 levels)  — in the 802.1Q VLAN tag            │
├─────────────────────────────────────────────────────────────────────┤
│  Layer 3: Per-Packet Enforcement                                    │
│  "Is this individual packet within its allowed profile?"            │
│   → Metering (Token Bucket / trTCM)                                 │
│   → Policing (drop or remark out-of-profile packets)                │
│   → Shaping (delay excess packets to smooth bursts)                 │
│   → AQM / WRED (proactive drop/mark before queue overflow)          │
├─────────────────────────────────────────────────────────────────────┤
│  Layer 4: Per-Class Treatment (DCB)                                 │
│  "How does the switch treat entire traffic classes differently?"    │
│   → Traffic Classes (TC0–TC7) — group priorities into queues        │
│   → Egress Scheduling — SP, DWRR, or Hybrid per queue               │
│   → ETS (802.1Qaz) — per-TC bandwidth allocation                    │
├─────────────────────────────────────────────────────────────────────┤
│  Layer 5: Lossless Flow Control                                     │
│  "How do we prevent packet loss for specific priorities?"           │
│   → Priority Groups (PG) — ingress buffer allocation per priority   │
│   → PFC (802.1Qbb) — per-priority PAUSE frames                      │
│   → Threshold mechanics (Xoff, headroom, Xon)                       │
└─────────────────────────────────────────────────────────────────────┘
```
