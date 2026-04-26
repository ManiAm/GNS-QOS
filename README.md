
# GNS-QOS




## Documentation and Learning Path

This project includes a structured set of guides designed to build knowledge progressively. The documents introduce QoS concepts step by step, beginning with why traffic needs differentiated treatment and progressing through the switch-level mechanisms, lossless Ethernet extensions, and congestion control algorithms used in modern RDMA fabrics.

- **[Quality of Service (QoS)](docs/01_QOS.md):** Foundational concepts — the four network metrics (reliability, latency, jitter, throughput), why Best-Effort delivery fails for sensitive workloads, and how QoS solves the problem.
- **[QoS Service Models and Switch Implementation](docs/02_SERVICE_MODELS.md):** Industry standards and switch internals — IntServ vs. DiffServ, DSCP and Per-Hop Behaviors, the control plane vs. data plane, and the full ingress-to-egress DiffServ pipeline (classification, metering, marking, policing, shaping, and WRED/ECN).
- **[Traffic Classification and Scheduling](docs/03_CLASSIFICATION.md):** How packets are sorted and sent — Layer 2 (PCP) and Layer 3 (DSCP) tagging, Data Center Bridging (DCB), trust modes, Traffic Class assignment, and ETS scheduling (Strict Priority and DWRR).
- **[Priority-based Flow Control (PFC)](docs/04_PFC.md):** Lossless Ethernet — per-priority PAUSE frames (IEEE 802.1Qbb), Priority Groups, ingress buffer management (headroom, Xoff/Xon thresholds, shared headroom pools), and PFC Watchdog.
- **[DCQCN and ECN](docs/05_DCQCN.md):** RoCEv2 congestion control — the three-role feedback loop (CP, NP, RP), CNP anatomy, and the DCQCN rate-control state machine (alpha, recovery phases, clamp).
- **[Beyond Standard DCQCN](docs/05_DCQCN_.md):** Next-generation congestion control — per-hop telemetry (INT and IFA), direct rate control (HPCC/HPCC++), lighter congestion signals (eCNP, CSIG), and Programmable Congestion Control (PCC).

