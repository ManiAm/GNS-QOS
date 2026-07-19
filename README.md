
# GNS-QOS




## Documentation and Learning Path

This project includes a structured set of guides designed to build knowledge progressively. The documents introduce QoS concepts step by step, beginning with why traffic needs differentiated treatment and progressing through the switch-level mechanisms, lossless Ethernet extensions, and congestion control algorithms used in modern RDMA fabrics.

- **[Quality of Service (QoS)](docs/01_QOS.md):** Foundational concepts — the four network metrics (reliability, latency, jitter, throughput), why Best-Effort delivery fails for sensitive workloads, and how QoS solves the problem.
- **[QoS Service Models and Switch Implementation](docs/02_SERVICE_MODELS.md):** Industry standards and switch internals — IntServ vs. DiffServ, DSCP and Per-Hop Behaviors, the control plane vs. data plane, and the full ingress-to-egress DiffServ pipeline (classification, metering, marking, policing, shaping, and WRED/ECN).
- **[Traffic Classification and Scheduling](docs/03_CLASSIFICATION.md):** How packets are sorted and sent — Layer 2 (PCP) and Layer 3 (DSCP) tagging, Data Center Bridging (DCB), trust modes, Traffic Class assignment, and ETS scheduling (Strict Priority and DWRR).
- **[Priority-based Flow Control (PFC)](docs/04_PFC.md):** Lossless Ethernet — per-priority PAUSE frames (IEEE 802.1Qbb), Priority Groups, Xoff/Xon thresholds, and PFC Watchdog.
- **[Ingress Buffer Architecture](docs/04a_BUFFER.md):** How switch ASICs allocate on-chip buffer across Priority Groups — the three-tier model (guaranteed reserved pool, dynamic/static shared pool with alpha thresholds, headroom), dedicated vs. shared headroom pools, and the power-of-2 alpha convention.
- **[DCQCN and ECN](docs/05_DCQCN.md):** RoCEv2 congestion control — the three-role feedback loop (CP, NP, RP), CNP anatomy, and the DCQCN rate-control state machine (alpha, recovery phases, clamp).
- **[Beyond Standard DCQCN](docs/06_DCQCN_.md):** Next-generation congestion control — telemetry-driven rate control (HPCC/HPCC++), Congestion Signaling (CSIG), delay-based control (Swift), switch-direct backward feedback (Fast CNP, Bolt), and Programmable Congestion Control (PCC).
- **[Per-Hop In-Band Telemetry](docs/07_PerHop_Inband_Telemetry.md):** General-purpose telemetry infrastructure — INT and IFA (IFAv2), operational modes (inband vs. clone), header format, per-hop metadata, and overhead trade-offs (PINT, CSIG). Not a congestion control algorithm; a data-plane mechanism that feeds CC and other applications.
- **[Configuring QoS on SONiC](docs/08_SONIC_QOS.md):** Hands-on guide — step-by-step instructions for configuring the full QoS pipeline on a SONiC switch, covering CONFIG_DB tables, DSCP-to-TC mapping, WRED/ECN profiles, PFC, PFC Watchdog, ETS scheduling, buffer management, and traffic validation with `iperf3` and `perftest`.

