---
layout: page
title: project 4
description: Optimal Growth Model via QuantEcon
img: assets/img/12.jpg
importance: 4
category: work
related_publications: true
jupytext:
  text_representation:
    extension: .md
    format_name: myst
kernelspec:
  display_name: Julia
  language: julia
  name: julia-1.11
---

# Optimal Growth I: The Stochastic Optimal Growth Model

This project is based on the QuantEcon lecture: [Optimal Growth I](https://python.quantecon.org/optimal_growth_model.html), adapted for Julia.

It covers:

- Dynamic programming in stochastic environments
- Policy function approaches
- The Bellman equation and operator
- Fitted value function iteration
- Computational techniques using Julia

The full lecture includes mathematical exposition, algorithmic implementation, and simulation results. For full interactivity, see the [QuantEcon Julia lectures](https://julia.quantecon.org/).

## Example Code Snippet (Julia)

```julia
using LinearAlgebra, Statistics, Plots

f(x) = 2 .* cos.(6x) .+ sin.(14x) .+ 2.5
c_grid = 0:0.2:1
f_grid = range(0, 1, length = 150)

Af = LinearInterpolation(c_grid, f(c_grid))

plt = plot(xlim = (0, 1), ylim = (0, 6))
plot!(plt, f, f_grid, lw = 2, label = "true function")
plot!(plt, f_grid, Af.(f_grid), lw = 2, label = "linear approximation")
plot!(plt, legend = :top)
```

For the complete interactive notebook and mathematical details, see [QuantEcon - Optimal Growth Model](https://julia.quantecon.org/dynamic_programming/optimal_growth_model.html).
