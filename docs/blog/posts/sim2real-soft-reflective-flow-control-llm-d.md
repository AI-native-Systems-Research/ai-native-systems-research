---
date: 2026-07-08
categories:
  - llm-d
  - Deep Dives
authors:
  - jgchn
  - toslali-ibm
  - kalantar
description: >
  Part 2 of the sim2real series: simulation-driven discovery of the soft-reflective ceiling
  policy cuts critical-class TTFT by up to 98% at near-capacity load, without rejecting a
  single request.
---

# From Simulation to Production: How an AI-Native Pipeline Discovered a Better Flow Controller for llm-d

*Part 2 of the sim2real series. A case study in closing the AI-native loop across a second
problem class: dispatch prioritization.*

## Introduction

The [first article in this series](sim2real-probabilistic-admitter-llm-d.md) traced a
complete traversal of the AI-native loop for llm-d's admission control layer: an AI agent
framework ([Nous](https://github.com/AI-native-Systems-Research/agentic-strategy-evolution))
used a high-fidelity simulator ([BLIS](https://github.com/inference-sim/inference-sim)) to
discover the probabilistic admitter, which cuts critical-traffic TTFT by up to 97% at
overloaded workloads. That article also introduced the core methodology: simulation lets the
loop run at machine speed, reserving expensive GPU time for validating the few candidates that
actually merit real-hardware testing.

This article applies the same loop to a different problem: **flow control**. The distinction
matters. Admission control acts at the gate — it decides whether an incoming request enters the
system at all, and may reject it. Flow control acts at the dispatch queue — it decides which
already-admitted request gets sent to a backend pod next. The two mechanisms protect different
points in the pipeline and neither replaces the other.

The concrete outcome is the **soft-reflective ceiling policy**: a new flow control plugin for
llm-d-router that reduces critical-class TTFT p99 by up to 98% at near-capacity load without
rejecting a single request. It is now merged into
[llm-d-router](https://github.com/llm-d/llm-d-router), and users can enable it today with a
single YAML change.

<!-- more -->
