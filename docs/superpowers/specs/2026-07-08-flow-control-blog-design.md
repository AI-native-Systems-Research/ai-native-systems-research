---
name: flow-control-blog-part2
description: Design spec for the second sim2real blog article covering the soft-reflective flow control plugin for llm-d
metadata:
  type: project
---

# Blog Article Design: From Simulation to Production — Flow Control (Part 2)

## Overview

A second entry in the sim2real blog series, following the admission control article (Part 1).
This article covers the soft-reflective ceiling policy: a flow control plugin for llm-d
discovered via Nous+BLIS simulation and validated on real hardware, reducing critical-class
TTFT p99 by up to 98% at near-capacity load without rejecting any requests.

**Output file:** `docs/blog/posts/sim2real-soft-reflective-flow-control-llm-d.md`

**Authors:** Jing Chen, Michael Kalantar, Mert Toslali (mirroring the admission control article)

**Categories:** llm-d, Deep Dives

---

## Title

*From Simulation to Production: How an AI-Native Pipeline Discovered a Better Flow Controller for llm-d*

**Description (for frontmatter):**
> Part 2 of the sim2real series: simulation-driven discovery of the soft-reflective ceiling policy
> cuts critical-class TTFT by up to 98% at near-capacity load, without rejecting a single request.

---

## Section-by-Section Design

### Introduction

Two jobs:

1. **Orient new readers** in one paragraph: the AI-native loop (observe → reason → change →
   validate → deploy), BLIS as the simulator, Nous as the discovery framework. Link to Part 1
   for depth. Keep it to 3–4 sentences — enough to stand alone, not enough to re-tell the story.

2. **Draw the Part 1 / Part 2 distinction** explicitly: admission control acts at the gate and
   may reject requests; flow control acts at the dispatch queue and prioritizes among admitted
   requests. Neither replaces the other. This article covers the second layer.

End with the concrete outcome: the soft-reflective ceiling policy is now merged into
llm-d-router and users can enable it today.

---

### Observe: The Flow Control Problem

Describe the gap in llm-d's existing flow control:

- `ceiling=1.0` for all bands (the default) is effectively a no-op: sheddable traffic competes
  equally with critical traffic for dispatch slots until pods are fully saturated.
- Operators have two unsatisfying alternatives:
  - **No control (status quo):** critical requests queue behind sheddable work under load.
  - **Static ceiling:** binary cliff — below the threshold, sheddable traffic flows freely;
    above it, the band is fully blocked. No proportional middle ground. Causes oscillation.
- Neither handles moderate overload gracefully: critical TTFT spikes because sheddable requests
  occupy dispatch slots with equal priority.

Establish that Nous identified this as the next opportunity after admission control.

---

### Reason + Change: Simulation-Driven Discovery

**BLIS subsection (brief):**
Same simulator as Part 1; same economics argument (seconds vs. 30 minutes on GPU). Keep to
2–3 sentences pointing back to Part 1 rather than re-explaining. The key point: simulation
lets the loop run at machine speed.

**Nous discovery subsection:**
Describe the Nous-driven exploration at a high level (same narrative style as Part 1's
two-evolution-phase account). Nous explored candidate flow control policies in simulation,
iterated on what worked and what didn't, and converged on an approach with two key properties:

1. A *reflective ceiling* that scales each band's threshold linearly with saturation, creating
   a dead zone at low load where no gating occurs at all.
2. A *proportional enforcement* mechanism that, once past the threshold, rate-limits dispatch
   on a periodic tick rather than fully blocking — smooth degradation instead of a cliff.

**Algorithm presentation:**
Show the formula clearly with the regime table from the proposal:

```
ceiling[i] = 1 - i * saturation / (N - 1)
```

For two tiers (critical + sheddable), ceiling collapses to `1 - saturation` for the sheddable
band. The regime table (saturation 0.0→≥1.0 vs. ceiling) illustrates the dead zone, the
proportional gating region, and the hard block at full saturation.

Note the contrast with the Part 1 algorithm: the quintic admitter has two discovered constants
(exponent=5, multiplier=300); this algorithm has no free parameters — the shape is determined
entirely by the number of priority bands and the current saturation reading.

**BLIS simulation results:**
Present the headline simulation numbers from the README:

| Workload | Rate | Split | Critical TTFT mean Δ | TTFT P90 Δ |
|---|---|---|---|---|
| Code generation | 18 QPS | 30/70 | −91.3% | −89.6% |
| Interactive chat | 150 QPS | 30/70 | −40.7% | −29.7% |
| Reasoning | 1 QPS | 50/50 | −2.9% | −0.2% |

Note that reasoning at low load shows near-zero effect — consistent with expectation since
the policy is a no-op below the saturation threshold. This is the sim correctly predicting
"no benefit needed" as well as "benefit exists."

---

### AI-Assisted sim2real Translation

One focused paragraph. Same pipeline as Part 1: AI agents read the simulation algorithm code,
understand the target production framework, map simulation signals to production interfaces,
and generate a complete Go plugin with unit tests. The target interface this time is
`UsageLimitPolicy.ComputeLimit()` in llm-d-router rather than the admitter interface — the
framework is generic, the translation process is identical. Output is a complete package that
passes the full build and test suite.

---

### Validate: Real-Cluster Benchmark Results

**Setup:**
- Hardware: 2×H100-SXM-80GB (vs. 4×H100 in Part 1 — note this explicitly)
- Model: Qwen/Qwen3-14B (same as Part 1)
- Baseline: default llm-d-router flow control (constant ceiling=1.0)
- Treatment: soft-reflective-ceiling-policy

**Three workload shapes** (explain the rationale for each, as Part 1 did):
- *Code generation* (large input ~1176 tokens, moderate output): models IDE assistants and
  CI/CD code review pipelines with mixed-priority traffic.
- *Interactive chat* (short input ~39 tokens, moderate output): high-throughput conversational
  APIs serving mixed priority tiers.
- *Reasoning* (moderate input ~780 tokens, large output ~6200 tokens): agent chains and
  multi-step planning where critical interactive queries compete with batch reasoning jobs.

**Results grouped by operating regime:**

- **Under-capacity** (code gen @ 4/10 QPS, chat @ 20 QPS, reasoning @ 0.2 QPS): neutral,
  as expected. Saturation stays below the gating threshold; the policy is a no-op.
- **Near-capacity / above-capacity** (the sweet spot):
  - Code gen @ 16 QPS: critical TTFT p99 −96.7%, E2E p99 −0.1%
  - Reasoning @ 0.7 QPS: critical TTFT p99 −91.2%, E2E p99 −48.5%, throughput +64.4%
    (baseline was too degraded to complete requests the treatment successfully served)
- **Over-capacity** (chat @ 80 QPS): critical TTFT p99 −22.4%, E2E p99 −14.6%

Include the full results table from the proposal.

**Sim→real alignment note:** Simulation correctly predicted the direction of improvement
across all workload types. Real-hardware gains at near-capacity load were larger than
simulation suggested — at near-saturation, real GPU memory pressure and vLLM preemption
amplify the benefit of keeping sheddable traffic out of dispatch slots.

---

### Deploy: A Contribution to llm-d

The soft-reflective ceiling policy is now part of llm-d-router as `soft-reflective-ceiling-policy`.
Opt-in via a single YAML change (include the config snippet from the proposal). No new
infrastructure or metrics required. The same note as Part 1: who benefits most (deployments
operating at 60–100% capacity with meaningful sheddable traffic fractions).

---

### Closing the Loop

Same five-point recap as Part 1, with flow control content:

- **Observe:** identified that constant-ceiling flow control provides no priority differentiation
- **Reason:** Nous and BLIS explored the space of dispatch gating policies at machine speed
- **Change:** Nous agents evolved and translated the algorithm into production Go code
- **Validate:** real-cluster benchmarks confirmed up to 98% latency improvement for critical traffic
- **Deploy:** contributed as a plugin into llm-d-router

Keep shorter than Part 1's version — the loop structure is now familiar to series readers.

---

### What We Learned

Three lessons distinct from Part 1's lessons:

1. **Proportionality beats binary cliffs.** A policy that smoothly ramps throttling intensity
   with saturation is more stable than a threshold that flips between fully-open and
   fully-blocked. The dead zone at low load means no unnecessary overhead; the proportional
   region means the system never oscillates.

2. **A no-op at low load is a feature.** The reflective ceiling guarantees the policy does
   nothing when saturation is below the band's reflection point. This is by design: a flow
   control policy that gates traffic when there is no contention would hurt rather than help.
   Confirming this in simulation (the near-zero reasoning result at 0.2 QPS) gave confidence
   before real-hardware testing.

3. **Simulation predicts direction; real hardware reveals magnitude.** BLIS correctly ranked
   treatment over baseline on all overloaded workloads. But real-hardware gains at
   near-capacity load exceeded simulation estimates — because real GPU memory pressure,
   vLLM preemption events, and batching dynamics amplify the effect of protecting dispatch
   slots in ways the simulator models conservatively. The sim→real gap is not a failure of
   simulation; it is a reminder that simulation's job is ranking, not absolute prediction.

---

### What Comes Next

Close the layered defense picture:

- Admission control (Part 1) reduces total load at the gate by probabilistically rejecting
  sheddable requests before they enter the dispatch queue.
- Flow control (this article) prioritizes dispatch among already-admitted requests.
- **Composing both** provides two-layer protection: admission control prevents overload,
  flow control ensures critical traffic is served first during normal-to-near-capacity
  operation. Operators who want delay-over-rejection use flow control alone; those who
  want to shed load aggressively use admission control alone; both together gives layered
  resilience.

Brief forward look: the loop is now proven across two distinct problem classes (admission
and dispatch) with two distinct algorithm shapes (probabilistic curve vs. reflective
formula). The methodology scales to other llm-d problems — routing, autoscaling, prefill-decode
disaggregation — each of which is a search problem over a large design space where
simulation-driven agentic exploration excels.

---

## Data Sources

| Data | Source |
|---|---|
| BLIS simulation results | `/Users/jchen/go/src/inference-sim/soft-reflective/README.md` (headline table) |
| Detailed simulation results | `/Users/jchen/go/src/inference-sim/soft-reflective/docs/soft-reflective-results-summary.md` |
| Real-hardware benchmark results | Provided proposal document (the full results table) |
| Algorithm source | `/Users/jchen/go/src/inference-sim/soft-reflective/algorithms/soft_reflective_ceiling.go` |
| Regime table | Proposal document |

## Key Numbers to Highlight

- Code gen @ 16 QPS: TTFT p99 −96.7% (baseline 13.6 s → treatment 263 ms)
- Reasoning @ 0.7 QPS: TTFT p99 −91.2%, throughput +64.4%
- BLIS simulation: code gen −91.3% TTFT mean (direction confirmed before hardware)
- Under-capacity: neutral (policy is a no-op below saturation threshold — by design)

## Style Notes

- Match Part 1's voice: evidence-forward, technical but accessible, no hype
- Keep the BLIS and Nous subsections shorter than Part 1 (readers of the series know the framework)
- The algorithm section should be self-contained enough for readers who haven't read Part 1
- Use the same image/table formatting conventions as `sim2real-probabilistic-admitter-llm-d.md`
