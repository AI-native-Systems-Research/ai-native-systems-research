---
date: 2026-08-06
categories:
  - Kernels
authors:
  - assaft
  - Gal-bloch
  - gilv
description: >
  How we used K-Search to translate CUDA kernels to MLX, enabling high-performance
  inference kernels on Apple Silicon with minimal manual effort.
---

# AI-Native Systems for Portable Kernels Across Hardware Architectures

Modern AI kernels are overwhelmingly written for CUDA, making it difficult to
bring state-of-the-art optimizations to emerging hardware backends. In this
post, we show how IBM Research collaborated with the K-Search team to
automatically translate CUDA kernels to MLX, demonstrating that evolutionary
kernel optimization can dramatically reduce the engineering effort required to
port high-performance kernels while maintaining competitive performance.

<!-- more -->

!!! info "Originally published on the BAIR Blog"
    This post was originally published on the Berkeley AI Research (BAIR) blog.

    [Continue reading on BAIR →]([https://bair.berkeley.edu/blog/](https://bair.berkeley.edu/blog/2026/07/29/cuda-to-mlx-k-search/)){ .md-button .md-button--primary }
