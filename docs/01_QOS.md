
# Quality of Service (QoS)


## Network Performance Metrics

Every flow traversing a network is measured against four fundamental metrics. These metrics define what "good" and "bad" mean for different applications, and they are the basis for every QoS decision that follows.

- **Packet Loss** — The fraction of transmitted packets that never reach the destination. Loss can result from buffer overflow (congestion), bit errors on the link, or policy-based drops (policing). Transport protocols like TCP detect and retransmit lost segments, but retransmission adds delay. Real-time protocols (RTP) and RDMA transports cannot afford the retransmission cost.

- **Latency (Delay)** — The time a packet takes to travel from source to destination. Total latency is the sum of propagation delay (speed of light in the medium), serialization delay (time to push bits onto the wire), processing delay (lookup and forwarding decisions at each hop), and queuing delay (time spent waiting in switch buffers). Of these, queuing delay is the only component that varies with load — and it is the component that QoS mechanisms directly control.

- **Jitter (Delay Variation)** — The variation in latency between consecutive packets of the same flow. A flow with consistent 5 µs latency has zero jitter; a flow that alternates between 2 µs and 50 µs has high jitter. Jitter matters because receiving applications (codec playout buffers, RDMA completion queues) expect packets at predictable intervals. High jitter forces larger buffers or causes dropped frames.

- **Throughput** — The sustained data rate delivered to the application, measured after accounting for protocol overhead, retransmissions, and congestion-induced rate reductions. Throughput is bounded by the narrowest link on the path (bottleneck bandwidth) and reduced by any packet loss or congestion backoff.


## Elastic vs. Inelastic Traffic

All network traffic falls into one of two categories based on whether the application can adapt its transmission rate to available bandwidth:

- **Elastic Traffic** — Applications that tolerate variable delay and throughput by adjusting their sending rate. TCP's congestion control governs this adaptation: the sender slows down when it detects loss or congestion and speeds up when the path is clear. File transfers (FTP), web browsing (HTTP), and email (SMTP) are elastic. They require complete, reliable delivery — every byte must arrive — but they are flexible about *when* it arrives.

- **Inelastic Traffic** — Applications with strict timing or loss constraints that cannot simply back off and retry. Inelastic traffic spans a wide spectrum:

  - **Loss-tolerant, delay-sensitive** — VoIP, video conferencing, and live streaming. These require low latency and low jitter. A voice packet that arrives 200 ms late is useless — the conversation has moved on. However, the codec / playout layer can interpolate over an occasional dropped frame, so moderate packet loss causes only a brief glitch rather than a failure. The constraint is on *timing*, not on perfect delivery.

  - **Lossless, delay-sensitive** — RDMA-based workloads (GPU-to-GPU collectives, NVMe-over-Fabrics, distributed databases). These demand **both** zero packet loss **and** ultra-low latency (microseconds, not milliseconds). Unlike VoIP, RDMA has no codec to mask a missing packet. Any loss triggers an expensive retransmission or go-back-N recovery at the transport layer, destroying tail latency and stalling the application. This traffic profile is the reason lossless flow control (PFC) and end-to-end congestion control (DCQCN) exist — topics covered in depth in later sections.

| Traffic Type                 | Application          | Packet Loss Tolerance | Latency Sensitivity | Jitter Sensitivity | Throughput Need |
| ---------------------------- | -------------------- | --------------------- | ------------------- | ------------------ | --------------- |
| Elastic                      | FTP                  | None (retransmit)     | Low                 | Low                | Medium          |
|                              | HTTP                 | None (retransmit)     | Medium              | Low                | Medium          |
| Inelastic (loss-tolerant)    | On-demand Audio      | Moderate              | Medium              | Medium             | Medium          |
|                              | On-demand Video      | Moderate              | Medium              | Medium             | High            |
|                              | Voice over IP (VoIP) | Moderate              | High                | High               | Low             |
|                              | Video Conferencing   | Moderate              | High                | High               | High            |
| Inelastic (lossless)         | RDMA / RoCEv2        | Zero                  | Very High (µs)      | High               | Very High       |
|                              | NVMe-over-Fabrics    | Zero                  | Very High (µs)      | High               | Very High       |
|                              | AI Collectives       | Zero                  | Very High (µs)      | High               | Very High       |


## Best-Effort Delivery and Its Limits

The original internet was designed around the **End-to-End Principle** (Saltzer, Reed & Clark, 1984): the network's only job is to forward packets toward their destination. It makes no promises about delivery order, timing, or loss. If a packet is dropped due to congestion, the endpoints — not the network — are responsible for detecting and recovering from the loss. This model is called **best-effort delivery**.

Best-effort works well for elastic traffic. TCP detects loss, retransmits, and adapts its sending rate — the application simply waits a little longer. But best-effort has no mechanism to differentiate one flow from another. When a link is congested, all flows sharing that link experience the same queuing delay and the same drop probability, regardless of their requirements.

This breaks down for inelastic traffic. A VoIP flow sharing a congested link with a bulk file transfer will experience the same queue-induced delay and drops — but the VoIP call cannot retransmit a 200 ms-old voice sample. For lossless workloads the problem is even more severe: RDMA traffic treats any dropped packet as a transport-layer failure, triggering expensive recovery sequences that destroy the microsecond-scale latency the application depends on. Best-effort delivery offers no way to protect these flows.


## Quality of Service

**Quality of Service (QoS)** is the set of mechanisms that allow the network to treat different traffic classes differently. It does not add bandwidth — it controls how existing bandwidth, buffer space, and scheduling priority are allocated among competing flows. The goal is to guarantee that traffic with strict loss, latency, or jitter requirements receives the resources it needs, even when the network is congested.

QoS is organized in six layers, each answering a distinct question:

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
├─────────────────────────────────────────────────────────────────────┤
│  Layer 6: End-to-End Congestion Control                             │
│  "How does the sender know when to slow down?"                      │
│   → ECN marking + CNP feedback (DCQCN)                              │
│   → Richer signals — per-hop telemetry (IFA/INT), CSIG, RTT         │
│   → Programmable reaction — custom algorithms on NIC (PCC)          │
└─────────────────────────────────────────────────────────────────────┘
```
