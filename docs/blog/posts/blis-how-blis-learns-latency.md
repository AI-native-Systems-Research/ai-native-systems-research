---
date: 2026-08-06
categories:
  - BLIS
  - Deep Dives
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
  How BLIS predicts a single forward pass on a CPU, how well those predictions
  hold up on models and GPUs it never trained on, and where they still fall short.
---

# What We've Learned About Modeling LLM Latency

The [last post in the BLIS series](blis-building-trust-physics-of-simulation.md) covered how BLIS models the engine,
data plane, and control plane to predict full-pipeline latency without touching a GPU. What it
didn't answer is whether the predictions can be trusted.

That comes down to one number. Everything BLIS reports at the cluster level rests on its
estimate of how long a single **forward pass** takes, so if that's off, nothing above it can
be right. This post is about how we estimate it, how well it holds up, and where it falls
short.

The headline, up front: fit once on H100, BLIS predicts held-out configurations (six models
across three GPU types) at **6.7% median end-to-end error**, roughly **200× faster** than
running them for real, and those predictions have already steered serving policies we later
confirmed on a physical cluster: a
[better admission controller](sim2real-probabilistic-admitter-llm-d.md)
and [soft reflective flow control](sim2real-soft-reflective-flow-control-llm-d.md)
for llm-d. The rest of this post is how we get there, and where it still falls short.

<!-- more -->

!!! info "Cross-posted from the BLIS blog"
    This post was originally published on the
    [BLIS](https://inference-sim.github.io/inference-sim/latest/) blog.

    [Continue reading on inference-sim →](https://inference-sim.github.io/inference-sim/latest/blog/2026/08/06/what-weve-learned-about-modeling-llm-latency/){ .md-button .md-button--primary }
