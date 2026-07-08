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

# From Simulation to Production: Soft-Reflective Flow Control for llm-d

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
matters. Admission control acts at the gate: it decides whether an incoming request enters the
system at all, and may reject it. Flow control acts at the dispatch queue: it decides which
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
constant ceiling of 1.0 for all bands, which is effectively a no-op. Every band dispatches
freely until pods are fully saturated, regardless of whether a request is a critical
interactive query or a background batch job.

This leaves operators with two choices, neither satisfactory. With the status quo, sheddable
traffic competes equally with critical traffic for dispatch slots. Under load, critical requests
queue behind sheddable work, inflating TTFT for the traffic that matters most. The alternative,
setting a lower static ceiling for sheddable bands (e.g., `ceiling=0.5`), creates a binary
cliff: below that saturation level, sheddable traffic dispatches freely; above it, the band is
fully blocked. There is no middle ground. The system jumps from zero throttling to complete
starvation with no proportional response to intermediate load, and oscillates rather than settling.

Nous identified this as the next opportunity after admission control.

## Reason + Change: Simulation-Driven Discovery

### The Simulator: BLIS

The same simulator used in Part 1, [BLIS](https://github.com/inference-sim/inference-sim)
(a high-fidelity discrete-event simulator for distributed LLM inference systems), provides the
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
plugin rate-limits dispatch to every `round(saturation / (1 - saturation))`-th tick, a
proportional duty cycle that scales smoothly with load. The dispatch loop already treats
`saturation >= ceiling` as a head-of-line block, so alternating 1.0 and 0.0 on successive
ticks produces proportional throughput without any change to the dispatch code path.

For a two-tier deployment (critical + sheddable), the ceiling collapses to `1 - saturation`
for the sheddable band:

| Saturation | Sheddable ceiling | Regime |
|:---|:---|:---|
| 0.00–0.50 | 1.00–0.50 | Below saturation: no gating |
| 0.50 | 0.50 | Boundary: gating begins |
| 0.60 | 0.40 | Proportional |
| 0.75 | 0.25 | Proportional |
| 0.90 | 0.10 | Proportional |
| ≥ 1.00 | 0.00 | Hard block |

The dead zone below saturation 0.5 means sheddable traffic flows freely at half-capacity and
below. The steep tail beyond 0.8 reserves nearly all dispatch capacity for critical traffic
without ever fully closing the sheddable gate: bounded latency, not indefinite starvation.

Compare this to the Part 1 algorithm: the quintic admitter has two constants discovered by
Nous (exponent 5, multiplier 300) tuned to hit a shedding curve of the right shape. The
reflective ceiling has no free parameters; the shape is determined entirely by the number of
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

Code generation (heavy on large-input prefill) benefits most because sheddable
large-context requests occupy dispatch slots that the reflective ceiling keeps available for
critical traffic. Interactive chat sees substantial improvement at high throughput. Reasoning
at 1 QPS shows near-zero effect: saturation stays below the gating threshold and the policy is
correctly a no-op. Simulation is working as intended when it predicts "no benefit needed" as
accurately as it predicts "benefit exists."

## AI-Assisted sim2real Translation: Simulation to Production Code

The same translation pipeline used in Part 1 converts the simulation algorithm into a
production Go plugin. AI agents read the simulation code, understand llm-d-router's plugin
framework, map simulation signals to their production equivalents
(`SaturationDetector.Saturation()` to `ComputeLimit()` parameter), and produce a complete
package with unit tests and parameter validation that passes the full build and test suite.
The target interface this time is `UsageLimitPolicy.ComputeLimit()` rather than the admitter
interface, but the process is identical: the framework is generic, and the translation agents
work from the interface contract rather than the internals.

The output is not a prototype. The plugin is a proper production artifact with full provenance:
a simulation-validated algorithm, translated by AI agents, ready to be registered and enabled
with a YAML config change.

## Validate: Real-Cluster Benchmark Results

Simulation produced promising results. The only way to know the true value is to test on real
hardware. Expensive GPU time is used only for validation; the exploration already happened in
simulation.

### Setup

| Parameter | Value |
|:---|:---|
| Hardware | 2 × H100-SXM-80GB |
| Model | Qwen/Qwen3-14B |
| Baseline | Default llm-d-router flow control (constant ceiling 1.0) |
| Treatment | soft-reflective-ceiling-policy |
| Load generator | blis observe (open-loop, --max-concurrency 10000) |

Note that this benchmark uses 2 × H100 rather than the 4 × H100 cluster used in Part 1,
reflecting a different hardware configuration while keeping the same model.

### Workloads

Three workload shapes cover the range of production LLM deployment patterns:

**Code generation** (large input ~1,176 tokens, moderate output ~195 tokens, 30% critical /
70% sheddable): models IDE inline suggestions and CI/CD code review pipelines where critical
suggestions share capacity with background analysis jobs.

**Interactive chat** (short input ~39 tokens, moderate output ~300 tokens, 30% critical /
70% sheddable): high-throughput conversational APIs serving mixed priority tiers.

**Reasoning** (moderate input ~780 tokens, large output ~6,200 tokens, 50% critical /
50% sheddable): agent chains and multi-step planning where critical interactive queries
compete with batch reasoning jobs.

All metrics below are for **critical-class traffic only**. Delta percentages show treatment
vs. baseline (negative = improvement).

### Results

| Workload | QPS | TTFT mean Δ | TTFT p99 Δ | E2E mean Δ | E2E p99 Δ | Tput Δ |
|:---|:---|:---|:---|:---|:---|:---|
| Code Generation | 4 | −0.4% | +1.0% | −0.2% | −2.0% | +0.0% |
| Code Generation | 10 | +2.9% | +8.9% | +1.0% | −2.4% | −0.8% |
| Code Generation | 16 | **−98.1%** | **−96.7%** | **−49.1%** | −0.1% | −0.8% |
| Code Generation | 24 | −96.6% | −91.2% | −55.6% | −9.3% | +8.0% |
| Interactive Chat | 20 | −1.6% | −4.8% | −0.1% | +0.2% | −0.0% |
| Interactive Chat | 40 | +2.6% | +18.4% | +1.9% | +2.2% | −0.0% |
| Interactive Chat | 60 | +6.8% | +30.9% | +0.8% | +0.5% | −0.0% |
| Interactive Chat | 80 | −31.1% | **−22.4%** | −17.8% | **−14.6%** | −0.9% |
| Reasoning | 0.2 | −4.6% | −1.4% | +0.5% | +3.6% | −1.6% |
| Reasoning | 0.4 | −36.6% | +16.7% | −7.1% | −1.2% | −3.1% |
| Reasoning | 0.7 | **−95.1%** | **−91.2%** | **−66.5%** | **−48.5%** | **+64.4%** |
| Reasoning | 1.2 | −44.6% | −44.6% | −35.5% | −37.3% | +35.8% |

### Reading the Results

The plugin's operating sweet spot is near-capacity load, where saturation fluctuates in the
proportional gating regime (0.5–1.0):

**Code generation @ 16 QPS** is the clearest signal. The baseline saturates on prefill
from 70%-sheddable large-input requests; critical TTFT p99 reaches 13.6 s. The treatment
proportionally gates sheddable dispatch, letting critical requests reach pods with minimal
wait: critical TTFT p99 falls to 263 ms, a 96.7% reduction. The difference between a 13-second
and a 263-millisecond time-to-first-token is the difference between a usable and an unusable
developer experience.

**Reasoning @ 0.7 QPS** shows the mechanism at work on long-running requests. Decode-heavy
workloads cause sustained high saturation, and the proportional gating continuously protects
critical prefill. Critical TTFT p99 drops from 330 s to 16 s (−91.2%), E2E improves −48.5%,
and throughput recovers 64.4%, because the baseline was overloaded enough to drop requests
that the treatment successfully completed.

**Under-capacity workloads** (code generation @ 4/10 QPS, interactive chat @ 20 QPS,
reasoning @ 0.2 QPS) show neutral results: saturation stays below the gating threshold and
the plugin is correctly a no-op. This is expected and important: a flow control policy that
penalizes traffic when there is no contention would cause harm rather than help.

**Interactive chat @ 40/60 QPS** shows the one range where the treatment is modestly worse
than baseline. These workloads operate near but not yet at capacity, in the transition zone
where the reflective ceiling begins to engage but the benefit of protecting critical dispatch
has not yet exceeded the overhead of rate-limiting sheddable traffic. The effect is small
(TTFT p99 +18% and +31% respectively at low absolute baseline values) and disappears at 80
QPS when the system is clearly above capacity.

### Sim-to-Real Alignment

Simulation correctly predicted the direction of improvement on every workload type. Real-hardware
gains at near-capacity load exceeded the simulation estimates, because real GPU memory pressure,
vLLM preemption events, and KV cache contention amplify the effect of keeping sheddable traffic
out of dispatch slots in ways the simulator models conservatively. The sim-to-real gap is not a
failure of simulation; it is a reminder that simulation's job is ranking, not absolute prediction.
BLIS ranked treatment over baseline on overloaded workloads. Real hardware confirmed and
amplified that ranking.

## Deploy: A Contribution to llm-d

With real-cluster benchmarks confirming the gains predicted by simulation, the algorithm
ships upstream as a new plugin in llm-d-router: `soft-reflective-ceiling-policy`. Users can
enable it today with a single YAML change and no new infrastructure:

```yaml
apiVersion: llm-d.ai/v1alpha1
kind: EndpointPickerConfig
featureGates:
  - flowControl
plugins:
  - type: queue-scorer
  - type: kv-cache-utilization-scorer
  - type: prefix-cache-scorer
  - type: soft-reflective-ceiling-policy
schedulingProfiles:
  - name: default
    plugins:
      - pluginRef: queue-scorer
        weight: 2.0
      - pluginRef: kv-cache-utilization-scorer
        weight: 2.0
      - pluginRef: prefix-cache-scorer
        weight: 3.0
flowControl:
  usageLimitPolicyPluginRef: soft-reflective-ceiling-policy
```

The plugin is most effective for deployments operating at 60-100% capacity with a meaningful
fraction of sheddable traffic: code generation pipelines, batch reasoning jobs, background
inference tasks running alongside latency-sensitive critical traffic. Under-capacity deployments
see no effect. Deployments above 150% capacity are better served by admission control (which
can shed load at the gate) rather than flow control (which cannot reject requests, only
prioritize among admitted ones).

## Closing the Loop

This case study traces a second complete traversal of the AI-native loop:

- **Observe:** identified that constant-ceiling flow control provides no priority differentiation
  under load, leaving critical traffic to queue behind sheddable work
- **Reason:** Nous and BLIS simulation explored the space of dispatch gating policies at machine
  speed, converging on the reflective ceiling algorithm
- **Change:** Nous agents translated the simulation algorithm into production Go code via the
  same sim2real pipeline used in Part 1
- **Validate:** real-cluster benchmarks on 2 × H100 with Qwen3-14B confirmed up to 98% TTFT
  improvement for critical traffic at near-capacity load
- **Deploy:** contributed as a new plugin into llm-d-router, enabled via a YAML config change

## What We Learned

Three lessons emerged from this iteration that were not obvious at the outset:

**Proportionality beats binary cliffs.** The constant-ceiling baseline flips abruptly between
fully-open and fully-blocked as saturation crosses a threshold. This causes oscillation and
provides no graceful degradation for the cases that matter most: moderate overload, where the
system is stressed but not catastrophic. A policy that smoothly ramps throttling intensity with
saturation is more stable and more useful across a wider operating range.

**A no-op at low load is a feature, not a bug.** The reflective ceiling guarantees the plugin
does nothing when saturation is below the band's reflection point. Confirming this in simulation
(the near-zero reasoning result at 0.2 QPS) gave confidence before real-hardware testing.
A flow control policy that penalizes traffic when there is no contention would cause harm.
Testing the absence of harm is as important as testing the presence of benefit.

**Simulation predicts direction; real hardware reveals magnitude.** BLIS correctly ranked
treatment over baseline on every overloaded workload. But the absolute gains on real hardware
at near-capacity load exceeded simulation estimates. The sim-to-real gap here is not a
calibration failure; the simulator is correctly conservative because it cannot fully model
GPU memory pressure, vLLM preemption cascades, and batching interference under real load.
The right expectation for simulation is: *if A beats B in simulation, A will beat B on real
hardware.* What the magnitude will be is something only real hardware can answer.

## What Comes Next

This article and Part 1 together cover two distinct protection layers: admission control
reduces total load at the gate by probabilistically rejecting sheddable requests before they
enter the dispatch queue; flow control prioritizes dispatch among already-admitted requests.
Composing both provides layered protection: admission control prevents overload, flow control
ensures critical traffic is served first under normal-to-near-capacity operation.

The natural next frontier is exploring multiple interacting policies simultaneously: how
admission control and flow control compose under varying load, and what the joint policy space
looks like when Nous searches it rather than each mechanism in isolation. With BLIS becoming
the official llm-d simulator, this kind of multi-policy search becomes tractable. The same
simulation infrastructure that validated these two algorithms can explore interactions,
emergent behaviors, and entirely new problem classes: routing, autoscaling,
prefill-decode disaggregation. The loop is proven. The simulator is ready. Now we scale it.
