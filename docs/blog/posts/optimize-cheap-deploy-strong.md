---
date: 2026-09-29
categories:
  - Deep Dives
authors:
  - taloved
  - roipony
  - oshri
  - udi
  - lakshya
description: >
  GEPA's cost is dominated by the task model that scores every candidate. Put that role
  on the cheapest model, keep a strong reflector for the rare edits, and deploy the evolved
  prompt zero-shot on a stronger model: cheaper to optimize and more accurate.
---

# Optimize Cheap, Deploy Strong: A Recipe for Cost-Efficient GEPA

Today we are sharing a simple recipe for running GEPA at a fraction of the cost, without giving up accuracy. In fact, you often get more of it. On HotpotQA, a prompt optimized this way deploys at 59.4% against 50.4% without the recipe. That is a +9.0 point gain while spending about 14x less ($7 vs $102).

The primary cost driver in GEPA is evaluation. Every iteration, a task LLM runs each candidate prompt across the full validation set, so over a run it is called hundreds or thousands of times. That is what dominates the budget. So we asked a simple question: what happens if we run all of those repeated calls on a very cheap model?

The surprising part is the direction. The cheaper route is also the better one. We call this positive transfer: a prompt tuned with a cheap task LLM can carry upward to a stronger model at deployment, and reach higher accuracy than that strong model reaches when it optimizes for itself.

<!-- more -->

!!! info "Cross-posted from the GEPA blog"
    This post was originally published on the
    [GEPA](https://gepa-ai.github.io/gepa/) blog.

    [Continue reading on GEPA →](https://gepa-ai.github.io/gepa/blog/2026/09/29/optimize-cheap-deploy-strong/){ .md-button .md-button--primary }
