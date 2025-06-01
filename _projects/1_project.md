---
layout: page
title: project 1
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


# Julia Essentials

This project adapts content from QuantEcon’s [Julia Essentials](https://quantecon.org/) course.

## Overview

Topics covered include:

- Common data types
- Iteration
- User-defined functions
- Logic and comparisons

## Setup

```julia
using LinearAlgebra, Statistics, Plots
```

## Data Types

```julia
x = true
typeof(x)

y = 1 > 2  # false

typeof(1.0)
typeof(1)
```

## Arithmetic

```julia
x = 2
y = 1.0

x * y
x^2
y / x
2x - 3y
@show 2x - 3y
@show x + y;
```

## Complex Numbers

```julia
x = 1 + 2im
y = 1 - 2im
x * y
```

## Strings

```julia
x = "foobar"
typeof(x)

x = 10
y = 20
"x = $x"
"x + y = $(x + y)"
"foo" * "bar"

s = "Charlie don't surf"
split(s)
replace(s, "surf" => "ski")
split("fee,fi,fo", ",")
strip(" foobar ")
match(r"(\d+)", "Top 10")
```

## Tuples and Dictionaries

```julia
x = ("foo", "bar")
y = ("foo", 2)
typeof(x), typeof(y)

x = "foo", 1
word, val = x
println("word = $word, val = $val")

d = Dict("name" => "Frodo", "age" => 33)
d["age"]
```

## Iteration

```julia
actions = ["surf", "ski"]
for action in actions
    println("Charlie doesn't $action")
end

for i in 1:3
    print(i)
end

d = Dict("name" => "Frodo", "age" => 33)
collect(keys(d))
```

## Comprehensions

```julia
doubles = [2i for i in 1:4]
animals = ["dog", "cat", "bird"]
plurals = [animal * "s" for animal in animals]
[i + j for i in 1:3, j in 4:6]
```

## Functions

```julia
function f1(a, b)
    return a * b
end

function f2(a, b)
    a * b
end

foo(x) = x > 0 ? "positive" : "nonpositive"
```

## Broadcasting

```julia
x_vec = [2.0, 4.0, 6.0, 8.0]
y_vec = sin.(x_vec)
```

## Closures

```julia
a = 0.2
f(x) = a * x^2
f(1)
```

## Final Example

```julia
function solve_model(x)
    a = x^2
    b = 2 * a
    c = a + b
    return (; a, b, c)
end
solve_model(0.1)
```
