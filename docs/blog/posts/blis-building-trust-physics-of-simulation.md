---
date: 2026-04-09
categories:
  - BLIS
  - llm-d
authors:
  - susiejojo
  - toslali-ibm
  - jgchn
  - namasl
  - kalantar
  - atantawi
  - vishakha-r
  - sriumcp
  - oliveira
description: >
  What it takes to build a discrete-event simulator faithful enough to guide
  LLM inference capacity planning and configuration search at production scale.
---

# The Physics of High-Fidelity Distributed Inference Platform Simulation

Production LLM inference platforms are distributed systems where routing policies, admission control, autoscaling, and engine-level scheduling all interact to determine latencies and throughput. How do you explore how different policies and configurations affect these KPIs before deploying to production? Testing a new routing policy or autoscaling threshold on live traffic risks cascading bugs across the fleet, while building separate test environments burns GPU-hours and still cannot predict interactions between cluster-level policies and engine-level batch dynamics.

<!-- more -->

The answer is **end-to-end simulation**: model the entire distributed inference stack to explore how policies and configurations affect latencies and throughput for your workloads. *What does it take to build a simulator accurate enough to guide these decisions?* The challenge lies in capturing the right mechanisms. At the engine level, batches process together — all requests wait for the slowest operation to finish, so KV cache fills trigger preemptions and long prompts stall short decodes. At the cluster level, routing policies operate on stale cache state, admission control gates overload, and prefill/decode disaggregation trades utilization for latency. At the control plane, autoscalers react to lagged metrics, creating oscillations. When these couplings are not modeled, predictions diverge: a back-of-the-envelope model might predict 50ms time-to-first-token while production measures 200ms.

!!! info "Cross-posted from the BLIS blog"
    This post was originally published on the
    [BLIS](https://inference-sim.github.io/inference-sim/latest/) blog.

    [Continue reading on inference-sim →](https://inference-sim.github.io/inference-sim/latest/blog/2026/04/09/the-physics-of-high-fidelity-distributed-inference-platform-simulation/){ .md-button .md-button--primary }
