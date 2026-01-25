---
layout: page
title: The CAPM
description: with background image
img: assets/img/12.jpg
importance: 1
category: projects
related_publications: true
authors:
  - Jonathan K. Ramani
---

# Project 5: A Simple Model in Asset Pricing

<span style="font-size: 0.9em; color: gray;">Jonathan K. Ramani</span>

---

## Overview

This project introduces and explains the **Capital Asset Pricing Model (CAPM)** — a fundamental framework in financial economics used to understand the relationship between risk and expected return. CAPM provides a simple but powerful formula to price risky securities and is widely used by investors and financial analysts.

---

## What is CAPM?

The **Capital Asset Pricing Model** states that the expected return of a security is equal to the risk-free rate plus a risk premium:

$E(R_i) = R_f + \beta_i (E(R_m) - R_f)$

Where:
- $E(R_i)$: Expected return of asset *i*
- $R_f$: Risk-free rate
- $\beta_i$: Beta of the asset
- $E(R_m)$: Expected return of the market portfolio

---

## CAPM Expected Return (Julia)

```julia
function capm_expected_return(Rf::Float64, Rm::Float64, beta::Float64)
    return Rf + beta * (Rm - Rf)
end

Rf = 0.03
Rm = 0.09
betas = [0.8, 1.0, 1.2, 1.5]

expected_returns = [capm_expected_return(Rf, Rm, b) for b in betas]
println(expected_returns)
```

---

## Estimating Beta (OLS)

```julia
using CSV, DataFrames, GLM

df = CSV.read("returns.csv", DataFrame)
df.asset_excess = df.asset_return .- df.rf
df.market_excess = df.market_return .- df.rf

model = lm(@formula(asset_excess ~ market_excess), df)
println(coef(model))
```

---

*Last updated: June 2025*
