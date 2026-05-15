---
date: 2026-03-05
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
  Why simulate LLM inference infrastructure before scaling: a flight-simulator
  approach to capacity planning that runs on a laptop, no GPUs required.
---

# Why Simulate Before You Scale

Deploying large language models in production is one of the most expensive infrastructure decisions an organization can make. A single high-end GPU costs upwards of $30,000, and a production cluster can run into millions per year. Yet most teams make their first scaling decisions based on rough estimates, vendor benchmarks, or — worst of all — trial and error on live hardware.

What if you could test your deployment plan *before* spending a dollar on GPUs?

<!-- more -->

!!! info "Cross-posted from the BLIS blog"
    This post was originally published on the
    [BLIS](https://inference-sim.github.io/inference-sim/latest/) blog.

    [Continue reading on inference-sim →](https://inference-sim.github.io/inference-sim/latest/blog/2026/03/05/why-simulate-before-you-scale/){ .md-button .md-button--primary }
