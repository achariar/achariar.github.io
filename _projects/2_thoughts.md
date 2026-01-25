---
layout: page
title: Dynamic Realism
description: A political science analysis of how future uncertainty, economic power spheres, and trade-security dilemmas drive cycles of cooperation and conflict.
img: assets/img/realism.png
importance: 1
category: thoughts
related_publications: true
authors:
  - Jonathan K. Ramani
---

## Overview

This post introduces **Dynamic Realism** — a systemic theory of international relations designed to explain why great powers **oscillate between long periods of cooperation and sudden shifts toward coercion, crisis, or war**.

Dynamic realism integrates key insights from **offensive realism** and **defensive realism**, but shifts the analytical center of gravity to a variable that standard realism often underweights: **commercial access and economic power spheres**. The core idea is simple: great powers compete because they fear the future — but they often restrain themselves because they also fear spirals of escalation. The resulting behavior is **conditional, strategic, and dynamic**.

---

## Interactive Mini-Simulation: Three Islands

Below is a small interactive analogy. Think of **A** and **B** as competing great powers, and **C** as a strategically located intermediary (trade hub, chokepoint, swing region, etc.). The moving ball is “attention / commerce / influence.” Adjust:

- **C bias**: how often flows route through C (intermediation dependence)
- **Direct A↔B link**: whether A and B can interact without C
- **Speed**: how quickly interaction cycles occur

<div style="border: 1px solid rgba(0,0,0,0.12); border-radius: 12px; padding: 14px; margin: 14px 0;">
  <div style="display:flex; flex-wrap:wrap; gap:10px; align-items:center; justify-content:space-between; margin-bottom:10px;">
    <div style="display:flex; gap:8px; flex-wrap:wrap;">
      <button id="dr-start" style="padding:6px 10px; border-radius:10px; border:1px solid rgba(0,0,0,0.15); background:transparent; cursor:pointer;">Start</button>
      <button id="dr-pause" style="padding:6px 10px; border-radius:10px; border:1px solid rgba(0,0,0,0.15); background:transparent; cursor:pointer;">Pause</button>
      <button id="dr-reset" style="padding:6px 10px; border-radius:10px; border:1px solid rgba(0,0,0,0.15); background:transparent; cursor:pointer;">Reset</button>
    </div>

    <label style="display:flex; align-items:center; gap:8px; font-size:0.95em;">
      <input type="checkbox" id="dr-direct" checked />
      Allow direct A↔B link
    </label>
  </div>

  <div style="display:grid; grid-template-columns: 1fr; gap:10px; margin-bottom:10px;">
    <label style="display:flex; align-items:center; gap:10px; font-size:0.95em;">
      Speed
      <input id="dr-speed" type="range" min="0.5" max="3.5" step="0.1" value="1.6" style="flex:1;" />
      <span id="dr-speed-val" style="min-width:48px; text-align:right; font-variant-numeric: tabular-nums;">1.6x</span>
    </label>

    <label style="display:flex; align-items:center; gap:10px; font-size:0.95em;">
      C bias (route via C)
      <input id="dr-bias" type="range" min="0" max="1" step="0.01" value="0.78" style="flex:1;" />
      <span id="dr-bias-val" style="min-width:48px; text-align:right; font-variant-numeric: tabular-nums;">0.78</span>
    </label>
  </div>

  <canvas id="dr-canvas" width="860" height="280" style="width:100%; height:auto; display:block; border-radius:12px; background:rgba(0,0,0,0.03);"></canvas>

  <div style="margin-top:10px; font-size:0.92em; color: rgba(0,0,0,0.65); line-height:1.35;">
    <strong>Interpretation (optional):</strong> Higher C-bias means both A and B depend more on the intermediary. In dynamic realism terms, intermediation can stabilize exchange (cooperation) but also creates vulnerability: if expectations shift, competition over C can intensify.
  </div>
</div>

<script>
(() => {
  const canvas = document.getElementById("dr-canvas");
  const ctx = canvas.getContext("2d");

  const btnStart = document.getElementById("dr-start");
  const btnPause = document.getElementById("dr-pause");
  const btnReset = document.getElementById("dr-reset");

  const speedSlider = document.getElementById("dr-speed");
  const speedVal = document.getElementById("dr-speed-val");

  const biasSlider = document.getElementById("dr-bias");
  const biasVal = document.getElementById("dr-bias-val");

  const directToggle = document.getElementById("dr-direct");

  // Island positions (responsive-ish in canvas space)
  const islands = {
    A: { x: 140, y: 150, r: 46, label: "State A" },
    C: { x: 430, y: 105, r: 44, label: "State C" },
    B: { x: 720, y: 150, r: 46, label: "State B" },
  };

  // Ball state
  let ball = { x: islands.A.x, y: islands.A.y, r: 10 };
  let running = false;
  let lastTs = null;

  // Route state
  let route = [];         // array of points {x,y,name}
  let segmentIndex = 0;   // which segment we're traveling
  let t = 0;              // 0..1 progress along segment

  // Utility
  const lerp = (a,b,u) => a + (b-a)*u;

  function chooseNextRoute() {
    // Current "home" is the last point we've reached (or start).
    const current = route.length ? route[route.length - 1].name : "A";

    // Determine destination among A/B with some bias through C
    // We model a "flow cycle": it tries to alternate between A and B,
    // but with probability bias it goes via C as an intermediate hop.
    const bias = parseFloat(biasSlider.value);
    const allowDirect = directToggle.checked;

    let target = (current === "A") ? "B" : (current === "B") ? "A" : null;

    if (current === "C") {
      // If at C, go to whichever side is "next" based on last non-C visited.
      // If previous was A, go to B; if previous was B, go to A. Default A.
      let prevNonC = null;
      for (let i = route.length - 2; i >= 0; i--) {
        if (route[i].name !== "C") { prevNonC = route[i].name; break; }
      }
      target = (prevNonC === "A") ? "B" : "A";
    }

    // Decide if we route through C
    const viaC = (Math.random() < bias);

    // Build route: current -> (maybe C) -> target
    const startName = current;
    const pts = [{ ...islands[startName], name: startName }];

    if (viaC) {
      // If current isn't C, add C; if it is C, skip.
      if (startName !== "C") pts.push({ ...islands.C, name: "C" });
      pts.push({ ...islands[target], name: target });
    } else {
      // direct: only if allowed or if current is C
      if (!allowDirect && startName !== "C") {
        // force via C if direct is not allowed
        pts.push({ ...islands.C, name: "C" });
        pts.push({ ...islands[target], name: target });
      } else {
        pts.push({ ...islands[target], name: target });
      }
    }

    route = pts;
    segmentIndex = 0;
    t = 0;

    // Snap ball to start
    ball.x = route[0].x;
    ball.y = route[0].y;
  }

  function drawArrow(x1,y1,x2,y2, alpha=0.45) {
    // line
    ctx.save();
    ctx.globalAlpha = alpha;
    ctx.lineWidth = 3;
    ctx.strokeStyle = "#000";
    ctx.beginPath();
    ctx.moveTo(x1,y1);
    ctx.lineTo(x2,y2);
    ctx.stroke();

    // arrow head
    const angle = Math.atan2(y2-y1, x2-x1);
    const headlen = 12;
    ctx.beginPath();
    ctx.moveTo(x2, y2);
    ctx.lineTo(x2 - headlen*Math.cos(angle - Math.PI/7), y2 - headlen*Math.sin(angle - Math.PI/7));
    ctx.lineTo(x2 - headlen*Math.cos(angle + Math.PI/7), y2 - headlen*Math.sin(angle + Math.PI/7));
    ctx.closePath();
    ctx.fillStyle = "#000";
    ctx.fill();
    ctx.restore();
  }

  function draw() {
    // Clear
    ctx.clearRect(0,0,canvas.width,canvas.height);

    // Background grid-ish
    ctx.save();
    ctx.globalAlpha = 0.06;
    ctx.strokeStyle = "#000";
    for (let x = 0; x <= canvas.width; x += 40) {
      ctx.beginPath(); ctx.moveTo(x,0); ctx.lineTo(x,canvas.height); ctx.stroke();
    }
    for (let y = 0; y <= canvas.height; y += 40) {
      ctx.beginPath(); ctx.moveTo(0,y); ctx.lineTo(canvas.width,y); ctx.stroke();
    }
    ctx.restore();

    // Draw possible links
    const allowDirect = directToggle.checked;
    drawArrow(islands.A.x, islands.A.y, islands.C.x, islands.C.y, 0.25);
    drawArrow(islands.C.x, islands.C.y, islands.B.x, islands.B.y, 0.25);
    if (allowDirect) drawArrow(islands.A.x, islands.A.y, islands.B.x, islands.B.y, 0.12);

    // Highlight active route segments
    if (route.length >= 2) {
      ctx.save();
      ctx.globalAlpha = 0.55;
      ctx.lineWidth = 6;
      ctx.strokeStyle = "#000";
      for (let i = 0; i < route.length - 1; i++) {
        ctx.beginPath();
        ctx.moveTo(route[i].x, route[i].y);
        ctx.lineTo(route[i+1].x, route[i+1].y);
        ctx.stroke();
      }
      ctx.restore();
    }

    // Draw islands
    Object.entries(islands).forEach(([k, isl]) => {
      ctx.save();
      ctx.fillStyle = "#000";
      ctx.globalAlpha = 0.10;
      ctx.beginPath();
      ctx.arc(isl.x, isl.y, isl.r + 10, 0, Math.PI*2);
      ctx.fill();
      ctx.restore();

      ctx.save();
      ctx.fillStyle = "#000";
      ctx.globalAlpha = 0.18;
      ctx.beginPath();
      ctx.arc(isl.x, isl.y, isl.r, 0, Math.PI*2);
      ctx.fill();
      ctx.restore();

      ctx.save();
      ctx.strokeStyle = "#000";
      ctx.globalAlpha = 0.50;
      ctx.lineWidth = 2;
      ctx.beginPath();
      ctx.arc(isl.x, isl.y, isl.r, 0, Math.PI*2);
      ctx.stroke();
      ctx.restore();

      // Labels
      ctx.save();
      ctx.fillStyle = "#000";
      ctx.globalAlpha = 0.80;
      ctx.font = "600 14px system-ui, -apple-system, Segoe UI, Roboto, Arial";
      ctx.textAlign = "center";
      ctx.fillText(isl.label, isl.x, isl.y + 5);
      ctx.restore();
    });

    // Draw ball
    ctx.save();
    ctx.fillStyle = "#000";
    ctx.globalAlpha = 0.85;
    ctx.beginPath();
    ctx.arc(ball.x, ball.y, ball.r, 0, Math.PI*2);
    ctx.fill();
    ctx.restore();
  }

  function step(ts) {
    if (!running) return;
    if (lastTs === null) lastTs = ts;

    const dt = Math.min(0.03, (ts - lastTs) / 1000); // cap to avoid jumps
    lastTs = ts;

    const speed = parseFloat(speedSlider.value);
    const segSpeed = 0.55 * speed; // tweak constant for nice motion

    // Ensure route exists
    if (route.length < 2) chooseNextRoute();

    // Travel along current segment
    const a = route[segmentIndex];
    const b = route[segmentIndex + 1];

    t += dt * segSpeed;

    if (t >= 1) {
      // Arrive at next node
      ball.x = b.x;
      ball.y = b.y;
      segmentIndex += 1;
      t = 0;

      // If route finished, pick next route starting from current endpoint
      if (segmentIndex >= route.length - 1) {
        // make the endpoint the start for next cycle
        route = [{ ...b, name: b.name }];
        segmentIndex = 0;
        chooseNextRoute();
      }
    } else {
      ball.x = lerp(a.x, b.x, t);
      ball.y = lerp(a.y, b.y, t);
    }

    draw();
    requestAnimationFrame(step);
  }

  function start() {
    if (running) return;
    running = true;
    lastTs = null;
    requestAnimationFrame(step);
  }

  function pause() {
    running = false;
    lastTs = null;
  }

  function reset() {
    pause();
    route = [];
    segmentIndex = 0;
    t = 0;
    ball.x = islands.A.x;
    ball.y = islands.A.y;
    draw();
  }

  // UI bindings
  speedSlider.addEventListener("input", () => {
    speedVal.textContent = `${parseFloat(speedSlider.value).toFixed(1)}x`;
  });
  biasSlider.addEventListener("input", () => {
    biasVal.textContent = `${parseFloat(biasSlider.value).toFixed(2)}`;
  });
  directToggle.addEventListener("change", () => {
    // refresh route so it reflects the new constraint
    chooseNextRoute();
    draw();
  });

  btnStart.addEventListener("click", start);
  btnPause.addEventListener("click", pause);
  btnReset.addEventListener("click", reset);

  // Initialize labels
  speedVal.textContent = `${parseFloat(speedSlider.value).toFixed(1)}x`;
  biasVal.textContent = `${parseFloat(biasSlider.value).toFixed(2)}`;

  // First render
  chooseNextRoute();
  draw();
})();
</script>

---

## What is Dynamic Realism?

**Dynamic realism** argues that great powers behave the way they do because leaders are constantly managing a trade-off:

1. **Hedge against future insecurity** by expanding power (especially economic power)
2. **Avoid triggering backlash and spirals** that produce the very insecurity they are trying to prevent

Unlike theories that predict either persistent conflict (hard-line realism) or durable cooperation (institutional liberalism), dynamic realism expects **cycles**: engagement and restraint can hold for long periods, but shifts in expectations about the future can produce **sharp policy pivots**.

---

## The Taproot: Why Commerce Matters

Traditional systemic realism focuses heavily on **military capabilities** and **territory**. Dynamic realism treats those as crucial — but downstream.

The deeper “taproot” is **commercial strength**, because:

- Sustained military power requires a strong economic base  
- Economic growth depends on access to markets, trade, finance, and investment  
- Great powers fear not only invasion, but also **subversion**, **ideological competition**, and **economic exclusion**

In this view, foreign policy is often about building, protecting, and expanding an **economic power sphere** that can support national security over decades.

---

## Three Realms of Great Power Commerce

Dynamic realism explains state behavior by distinguishing *where* trade and investment ties sit in the international system. Any great power typically operates across three commercial realms:

### 1) The First Realm: The Home Sphere (Low Risk, High Control)

This includes:
- allies dependent on the great power for security
- smaller states in its neighborhood
- colonies, territories, or protectorates (historically)

Because influence is high, access is relatively secure. Commerce here is a **foundation** for power.

### 2) The Second Realm: The Neutral / Nonaligned Zone (Competitive Space)

This includes:
- states that try to trade with everyone
- “swing states” economically courted by multiple great powers

Competition is intense but often non-military. Great powers try to convert economic ties into political alignment.

### 3) The Third Realm: Trade with Rival Great Powers (High Risk, High Reward)

This includes:
- trade and investment with adversaries and their controlled spheres

This realm is the hardest for standard offensive realism to explain: if rivals might cut you off, why trade with them at all?

Dynamic realism’s answer: **because leaders weigh future expectations**. Trading with rivals can accelerate growth and stabilize relations — until leaders believe future access is likely to deteriorate.

---

## Key Concepts

### Future Uncertainty (Offensive Realist Baseline)

Dynamic realism retains the offensive realist insight that **uncertainty about future intentions** pushes leaders to seek better power positions today.

But it rejects the idea that leaders *must* always assume worst-case futures. Instead, leaders make **probabilistic bets** about the future commercial and strategic environment — and revise policy when those bets look wrong.

### The Security Dilemma (Defensive Realist Restraint)

Dynamic realism also retains defensive realism’s core insight: attempts to increase security can make others feel less secure, producing spirals.

But it extends this idea beyond arms races into economics.

### The Trade Security Dilemma (Dynamic Realism’s Addition)

Actions taken to expand or defend commercial influence can trigger retaliatory measures:
- sanctions
- embargoes
- exclusion from markets
- financial restrictions
- competing blocs

These can spiral just like military competition — and sometimes they *precede* military crises.

---

## Assumptions of Dynamic Realism

1. **Anarchy persists**: there is no higher authority to guarantee access or peace.  
2. **Great powers are security-driven**, but must plan for adverse futures.  
3. **Economic capability is foundational** to long-term security competition.  
4. Leaders are **forward-looking** and make decisions based on **expectations** about future access, not just current power.  
5. Policies reflect a **trade-off**: build power vs. avoid spirals and backlash.

---

## How Dynamic Realism Explains Policy Shifts

Dynamic realism predicts a recognizable pattern:

### When expectations are optimistic:
- engagement is rational
- leaders avoid provocation
- interdependence is tolerated (even with rivals)

### When expectations worsen:
- leaders shift toward hedging
- coercion becomes more attractive
- the state pushes harder to lock in access and influence

### When leaders foresee exclusion:
- economic measures escalate (sanctions, block-building)
- crises become more likely
- war can become a grim “solution” to prevent long-term decline

This is why foreign policy can look stable for decades — and then pivot rapidly.

---

## Applications

Dynamic realism helps explain:

- **Cycles of engagement and confrontation** in a single state’s history  
- Why great powers sometimes pursue **commercial integration** with rivals, despite risk  
- Why economic tools (sanctions, trade policy, tech controls) can become central to rivalry  
- How competition over “neutral” states in the second realm shapes alliances and blocs  
- Why conflict often becomes more likely when leaders think the future trade environment is closing

---

## Limitations

- Harder to model cleanly than single-variable theories: expectations and trade-offs are complex.
- Measuring leader expectations empirically is difficult (requires careful historical interpretation).
- Domestic politics can still matter — dynamic realism mainly claims you can explain *a lot* without dropping to the unit level, not that domestic factors never matter.

---

## Conclusion

Dynamic realism offers a systemic explanation for a basic puzzle in international relations:  
**Why do great powers cooperate for long stretches, yet repeatedly swing toward coercion and war?**

Its answer is that great powers live in a world where **future uncertainty** pushes them to expand power — especially commercial power — while the **risk of spirals** pushes them to restrain. Cooperation is therefore real, but conditional; conflict is avoidable, but recurrent. The engine of rivalry is not only territory and armies, but the struggle to build and protect **economic power spheres** in a shifting global trade environment.

---

## Related Work

This post draws on foundational debates in international relations theory, especially:

- Mearsheimer, John J. (2001), *The Tragedy of Great Power Politics*.
- Jervis, Robert (1978), “Cooperation Under the Security Dilemma.”
- Waltz, Kenneth (1979), *Theory of International Politics*.
- Gilpin, Robert (1981), *War and Change in World Politics*.
- Haas, Mark L. (2005), *The Ideological Origins of Great Power Politics, 1789–1989*.

---

*Last updated: January 2026*
