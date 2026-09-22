---
date: 2026-06-05
categories:
  - BLIS
  - llm-d
authors:
  - toslali-ibm
  - susiejojo
  - sriumcp
  - jgchn
  - namasl
  - vishakha-r
  - kalantar
  - atantawi
  - oliveira
  - chcost
description: >
  Deploying llm-d means choosing among coupled routing, admission, batching, and
  autoscaling policies whose tradeoffs are hard to predict. BLIS is the faster
  inner loop: a calibrated simulator for evaluating those choices before cluster validation.
---

# BLIS: Evolving llm-d at Simulation Speed

Deploying llm-d is not just a question of choosing a model server and adding GPUs. In a production inference deployment, operators have to choose routing policies, admission behavior, batching settings, KV-cache reuse strategies, prefill/decode placement, and autoscaling rules under concrete TTFT, ITL, throughput, and cost constraints.

These choices are coupled. A routing change that improves cache locality can concentrate load. A prefill/decode threshold that helps one workload can hurt another. An admission policy that protects critical traffic can reduce total served volume. A change in any one policy can shift TTFT, inter-token latency, throughput, SLO compliance, and accelerator cost in ways that are difficult to predict analytically.

The only reliable way to confirm those tradeoffs is to measure them in a GPU-backed llm-d cluster. But using cluster runs as the first step in every policy or capacity-planning experiment is too slow and expensive. BLIS provides a faster inner loop: a calibrated discrete-event simulator for distributed inference systems like llm-d. Developers can evaluate candidate policies and deployment configurations locally, then reserve cluster validation for the candidates most likely to matter.

<!-- more -->

!!! info "Cross-posted from the llm-d blog"
    This post was originally published on the
    [llm-d](https://llm-d.ai/) blog.

    [Continue reading on llm-d →](https://llm-d.ai/blog/blis-evolving-llm-d-at-simulation-speed){ .md-button .md-button--primary }
