---
layout: page
title: Empty
description: Empty
img: assets/img/red.jpg
importance: 1
category: projects
related_publications: true
authors:
  - Jonathan K. Ramani
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

## Estimating Beta (OLS)

```julia
using CSV, DataFrames, GLM

df = CSV.read("returns.csv", DataFrame)

df.asset_excess  = df.asset_return .- df.rf
df.market_excess = df.market_return .- df.rf

model = lm(@formula(asset_excess ~ market_excess), df)
println(coef(model))
```
