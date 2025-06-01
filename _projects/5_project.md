---
layout: page
title: project 5
description: with background image
img: assets/img/12.jpg
importance: 1
category: work
related_publications: true
authors:
  - Jesse Perla
  - Thomas J. Sargent
  - John Stachurski
---

# Project 5: A Simple Model in Asset Pricing

<span style="font-size: 0.9em; color: gray;">Jesse Perla, Thomas J. Sargent, and John Stachurski</span>

This research project presents a stylized model of asset pricing in discrete time. We use standard methods in macro-finance, including consumption-based asset pricing, stochastic discount factors, and numerical simulation.

## 1. Model Overview

We consider a representative agent with power utility over consumption:

$$
u(c_t) = \frac{c_t^{1 - \gamma}}{1 - \gamma}, \quad \gamma > 0
$$

The agent discounts the future at a constant rate \( \beta \in (0, 1) \). The endowment \( y_t \) follows a log-normal process:

$$
\log y_{t+1} = \log y_t + \mu + \sigma \varepsilon_{t+1}, \quad \varepsilon_{t+1} \sim \mathcal{N}(0, 1)
$$

The stochastic discount factor (SDF) \( m_{t+1} \) is given by:

$$
m_{t+1} = \beta \left(\frac{c_{t+1}}{c_t}\right)^{-\gamma}
$$

Assuming \( c_t = y_t \), asset prices can be computed as expectations over future payoffs discounted by the SDF.

## 2. Pricing a Risk-Free Asset

The price of a one-period risk-free bond \( q^f_t \) is:

$$
q^f_t = \mathbb{E}_t[m_{t+1}]
$$

Using log-normality and log utility (\( \gamma = 1 \)), we get an analytical expression:

$$
\log q^f_t = \log \beta + (1 - \gamma) \mu + \frac{1}{2} (1 - \gamma)^2 \sigma^2
$$

The corresponding gross risk-free return is \( R^f_t = 1/q^f_t \).

## 3. Pricing an Equity Claim

Consider an equity claim that pays the endowment \( y_{t+1} \) each period. The price \( q^e_t \) is given by:

$$
q^e_t = \mathbb{E}_t[m_{t+1} y_{t+1}]
$$

Using log-normal properties, and \( \log y_{t+1} \sim \mathcal{N}(\log y_t + \mu, \sigma^2) \), we can derive:

$$
\log q^e_t = \log \beta + (1 - \gamma + 1)\mu + \frac{1}{2}[(1 - \gamma)^2 + 1]\sigma^2 + \log y_t
$$

This shows that equity is riskier and therefore has a higher expected return.

## 4. Julia Code: Simulating the Model

Here is a basic simulation of the model in Julia.

```julia
using Distributions, Random

function simulate_asset_prices(T=100, γ=2.0, β=0.96, μ=0.02, σ=0.1)
    y = zeros(T)
    m = zeros(T - 1)
    qf = zeros(T - 1)
    ye = zeros(T - 1)

    y[1] = 1.0
    for t in 1:T-1
        ε = rand(Normal(0, 1))
        y[t+1] = y[t] * exp(μ + σ * ε)
        m[t] = β * (y[t+1] / y[t])^(-γ)
        qf[t] = mean(m[t])
        ye[t] = m[t] * y[t+1]
    end
    return qf, ye
end
