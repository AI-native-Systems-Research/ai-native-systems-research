# llm-d — Kubernetes-Native LLM Inference

[llm-d](https://llm-d.ai) is a Kubernetes-native, high-performance distributed LLM inference framework. It provides the fastest time-to-value and competitive performance per dollar for serving language models across diverse hardware accelerators.

---

## Why llm-d for AI-native Research?

LLM inference platforms are ideal proving grounds for the AI-native approach. They are:

- **Complex and multi-component** — schedulers, KV caches, model servers, routing layers
- **Under constant pressure** — new models, new hardware, shifting traffic patterns
- **Multi-objective** — latency, throughput, cost, and fairness must be traded off
- **Observable** — rich telemetry from every request through the pipeline

---

## Key Capabilities

- **Inference scheduling** — intelligent request routing and scheduling
- **KV-cache optimization** — hierarchical offloading, cache-aware LoRA routing
- **Prefill/Decode disaggregation** — separating compute phases for efficiency
- **Wide Expert Parallelism** — load balancing across MoE experts
- **Scale-to-zero autoscaling** — resource-efficient scaling
- **Resilient networking with UCCL** — reliable distributed communication

---

## AI-Native Opportunities

In the AI-native vision, the inference platform continuously observes its own behavior — request latencies, queue depths, cache hit rates, GPU utilization — and uses this telemetry to drive improvements. These improvements span:

- **Configuration tuning** — batch sizes, scheduling policies, cache parameters
- **Code changes** — algorithmic improvements to routing, scheduling, or memory management
- **Structural evolution** — new components, refactored interfaces, optimized pipelines

The Controlling System reasons about multi-objective tradeoffs and experiments with alternatives before deploying validated changes.

---

## Resources

- **Website:** [llm-d.ai](https://llm-d.ai)
- **GitHub:** [github.com/llm-d](https://github.com/llm-d)
