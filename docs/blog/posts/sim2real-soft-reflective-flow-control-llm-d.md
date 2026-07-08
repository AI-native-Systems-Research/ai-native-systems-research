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

## Observe: The Flow Control Problem

llm-d's flow control framework gates dispatch of queued requests to backend pods based on
cluster saturation. Each priority band is assigned a ceiling value: when saturation reaches
or exceeds a band's ceiling, dispatch halts for that band. The only built-in policy is a
constant ceiling of 1.0 for all bands — which is effectively a no-op. Every band dispatches
freely until pods are fully saturated, regardless of whether a request is a critical
interactive query or a background batch job.

This leaves operators with two choices, neither satisfactory. With the status quo, sheddable
traffic competes equally with critical traffic for dispatch slots. Under load, critical requests
queue behind sheddable work, inflating TTFT for the traffic that matters most. The alternative
— setting a lower static ceiling for sheddable bands (e.g., `ceiling=0.5`) — creates a binary
cliff: below that saturation level, sheddable traffic dispatches freely; above it, the band is
fully blocked. There is no middle ground. The system jumps from zero throttling to complete
starvation with no proportional response to intermediate load, and oscillates rather than settling.

Nous identified this as the next opportunity after admission control.

## Reason + Change: Simulation-Driven Discovery

### The Simulator: BLIS

The same simulator used in Part 1 — [BLIS](https://github.com/inference-sim/inference-sim),
a high-fidelity discrete-event simulator for distributed LLM inference systems — provides the
economics that make agentic exploration viable. A single workload evaluation that takes 30
minutes on real hardware completes in seconds in simulation. Nous can evaluate many candidate
policies across diverse workload conditions before a single GPU is reserved. (See
[Part 1](sim2real-probabilistic-admitter-llm-d.md) for a full treatment of the simulator and
its role in the loop.)

### The Discovery: Nous

Nous explored the space of flow control policies in simulation using the same hypothesis-driven
experimentation loop it applied to admission control. Given the objective of protecting
critical-band dispatch under load, it searched for a policy with two properties that binary
cliff policies lack: a dead zone at low saturation where no gating occurs (avoiding unnecessary
overhead when there is no contention), and a smooth ramp that becomes more aggressive as
saturation grows (avoiding oscillation at the threshold).

The algorithm that emerged has a strikingly natural form. For N priority bands, the
per-band ceiling is:

```
ceiling[0] = 1.0                         # critical: never gated
ceiling[i] = 1 - i * saturation / (N-1)  # linearly reflected
```

When the sheddable band's saturation exceeds its ceiling, rather than fully blocking, the
plugin rate-limits dispatch to every `round(saturation / (1 - saturation))`-th tick — a
proportional duty cycle that scales smoothly with load. The dispatch loop already treats
`saturation >= ceiling` as a head-of-line block, so alternating 1.0 and 0.0 on successive
ticks produces proportional throughput without any change to the dispatch code path.

For a two-tier deployment (critical + sheddable), the ceiling collapses to `1 - saturation`
for the sheddable band:

| Saturation | Sheddable ceiling | Regime |
|:---|:---|:---|
| 0.00–0.50 | 1.00–0.50 | Below saturation — no gating |
| 0.50 | 0.50 | Boundary: gating begins |
| 0.60 | 0.40 | Proportional |
| 0.75 | 0.25 | Proportional |
| 0.90 | 0.10 | Proportional |
| ≥ 1.00 | 0.00 | Hard block |

The dead zone below saturation 0.5 means sheddable traffic flows freely at half-capacity and
below. The steep tail beyond 0.8 reserves nearly all dispatch capacity for critical traffic
without ever fully closing the sheddable gate — bounded latency, not indefinite starvation.

Compare this to the Part 1 algorithm: the quintic admitter has two constants discovered by
Nous (exponent 5, multiplier 300) tuned to hit a shedding curve of the right shape. The
reflective ceiling has no free parameters — the shape is determined entirely by the number of
priority bands and the current saturation reading. Adding a new InferenceObjective priority
level automatically extends the reflection with no config change.

### Simulation Results

Before committing to real-hardware benchmarks, BLIS confirmed the algorithm's direction across
three workload types:

| Workload | Rate | Critical/Sheddable | Critical TTFT mean Δ | Critical TTFT P90 Δ | Throughput Δ |
|:---|:---|:---|:---|:---|:---|
| Code generation | 18 QPS | 30/70 | **−91.3%** | **−89.6%** | −2.5% |
| Interactive chat | 150 QPS | 30/70 | **−40.7%** | **−29.7%** | −0.4% |
| Reasoning | 1 QPS | 50/50 | −2.9% | −0.2% | +2.0% |

Code generation — heavy on large-input prefill — benefits most because sheddable
large-context requests occupy dispatch slots that the reflective ceiling keeps available for
critical traffic. Interactive chat sees substantial improvement at high throughput. Reasoning
at 1 QPS shows near-zero effect: saturation stays below the gating threshold and the policy is
correctly a no-op. Simulation is working as intended when it predicts "no benefit needed" as
accurately as it predicts "benefit exists."
