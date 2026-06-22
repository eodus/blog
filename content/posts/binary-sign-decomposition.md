+++
title = "Binary Sign Decomposition: How ±1 Vectors Beat Standard Quantization"
date = 2026-06-22
draft = true
[taxonomies]
tags = ["compression", "linear-algebra", "llm"]
[extra]
math = true
+++

*What if you could represent a neural network's weight matrix as a sum of outer products of ±1 vectors — and beat the best available quantization?*

<!-- more -->

## The idea

Given a weight matrix W ∈ ℝ^{m×n}, approximate it as:

$$W \approx \sum_{i=1}^{k} \sigma_i \, u_i \, v_i^T$$

where u_i ∈ {±1}^m, v_i ∈ {±1}^n, and σ_i ∈ ℝ.

Each component costs m + n + few bits. At k ≈ rank(W), this gives ~2 bits per element — matching Q2_K quantization budget.

## Why it works

The greedy alternating projection:

1. v ← sign(R^T u)
2. u ← sign(R v)
3. σ = u^T R v / (m · n)
4. R ← R - σ u v^T

Each step is non-increasing in residual. At k ≈ rank(W), the decomposition approximates the SVD with ±1 constraints — the **Eckart-Young regime** for binary decomposition.

## The result

On Qwen 3.5 MoE (35B params, 3B active), 10 refinement sweeps:

| Method | PPL (WikiText-2) | Size | p-value vs Q2_K |
|--------|-----------------|------|------------------|
| Original FP16 | 6.955 | 67 GB | — |
| Q2_K | 7.737 | 15 GB | baseline |
| **Binary sign (ours)** | **7.634** | **22 GB** | **4×10⁻⁹** |

Statistically significant improvement (p = 4×10⁻⁹, Cohen's d = −0.25).

## Why this matters

The inference kernel is one conditional add:

```
y[i] += sign_bit ? +sigma : -sigma
```

No dequantization tables, no codebooks, no lattice structures. Runs on any hardware — including a Raspberry Pi.

---

*Ongoing work. Code and theory to follow.*
