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

<h2>CAPM Expected Return (Julia)</h2>

<div class="language-julia highlighter-rouge">
<div class="highlight">
<pre class="highlight"><code>
function capm_expected_return(Rf::Float64, Rm::Float64, beta::Float64)
    return Rf + beta * (Rm - Rf)
end

Rf = 0.03
Rm = 0.09
betas = [0.8, 1.0, 1.2, 1.5]

expected_returns = [capm_expected_return(Rf, Rm, b) for b in betas]
println(expected_returns)
</code></pre>
</div>
</div>


<h2>Estimating Beta (OLS)</h2>

<div class="language-julia highlighter-rouge">
<div class="highlight">
<pre class="highlight"><code>
using CSV, DataFrames, GLM

df = CSV.read("returns.csv", DataFrame)

df.asset_excess  = df.asset_return .- df.rf
df.market_excess = df.market_return .- df.rf

model = lm(@formula(asset_excess ~ market_excess), df)
println(coef(model))
</code></pre>
</div>
</div>
