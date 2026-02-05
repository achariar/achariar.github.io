---
layout: page
title: Dynamic Realism
description: Political science. 
img: assets/img/red.png
importance: 1
category: thoughts
related_publications: true
authors:
  - Jonathan K. Ramani
---



## Interactive Simulation: Trade, Growth, and Preemptive Occupation

<div style="border: 1px solid rgba(0,0,0,0.12); border-radius: 12px; padding: 14px; margin: 14px 0;">
  <div style="display:flex; flex-wrap:wrap; gap:10px; align-items:center; justify-content:space-between; margin-bottom:10px;">
    <div style="display:flex; gap:8px; flex-wrap:wrap;">
      <button id="sim-start" style="padding:6px 10px; border-radius:10px; border:1px solid rgba(0,0,0,0.18); background:transparent; cursor:pointer;">Start</button>
      <button id="sim-pause" style="padding:6px 10px; border-radius:10px; border:1px solid rgba(0,0,0,0.18); background:transparent; cursor:pointer;">Pause</button>
      <button id="sim-reset" style="padding:6px 10px; border-radius:10px; border:1px solid rgba(0,0,0,0.18); background:transparent; cursor:pointer;">Reset</button>
    </div>

    <div style="font-size:0.95em; color: rgba(0,0,0,0.70);">
      <span id="sim-status">Ready.</span>
    </div>
  </div>

  <div style="display:grid; grid-template-columns: 1fr; gap:10px; margin-bottom:10px;">
    <label style="display:flex; align-items:center; gap:10px; font-size:0.95em;">
      B catch-up growth
      <input id="sim-catchup" type="range" min="1.0" max="3.2" step="0.1" value="2.2" style="flex:1;" />
      <span id="sim-catchup-val" style="min-width:54px; text-align:right; font-variant-numeric: tabular-nums;">2.2x</span>
    </label>

    <label style="display:flex; align-items:center; gap:10px; font-size:0.95em;">
      Trade intensity (spawns)
      <input id="sim-trade" type="range" min="0.6" max="3.0" step="0.1" value="1.8" style="flex:1;" />
      <span id="sim-trade-val" style="min-width:54px; text-align:right; font-variant-numeric: tabular-nums;">1.8x</span>
    </label>

    <label style="display:flex; align-items:center; gap:10px; font-size:0.95em;">
      Occupation aggressiveness
      <input id="sim-aggro" type="range" min="0.6" max="2.6" step="0.1" value="1.4" style="flex:1;" />
      <span id="sim-aggro-val" style="min-width:54px; text-align:right; font-variant-numeric: tabular-nums;">1.4x</span>
    </label>
  </div>

  <canvas id="sim-canvas" width="980" height="380"
    style="width:100%; height:auto; display:block; border-radius:12px; background:#0b5f8a;"></canvas>

  <div style="margin-top:10px; font-size:0.92em; color: rgba(0,0,0,0.65); line-height:1.35;">
    <strong>Reading the toy model:</strong> Trade raises wealth (more dots), wealth sustains military capacity (triangles),
    and as B catches up, A’s perceived future risk rises—raising the likelihood of preemptively occupying C.
  </div>
</div>

<script>
(() => {
  const canvas = document.getElementById("sim-canvas");
  const ctx = canvas.getContext("2d");

  const btnStart = document.getElementById("sim-start");
  const btnPause = document.getElementById("sim-pause");
  const btnReset = document.getElementById("sim-reset");
  const statusEl = document.getElementById("sim-status");

  const catchup = document.getElementById("sim-catchup");
  const catchupVal = document.getElementById("sim-catchup-val");
  const trade = document.getElementById("sim-trade");
  const tradeVal = document.getElementById("sim-trade-val");
  const aggro = document.getElementById("sim-aggro");
  const aggroVal = document.getElementById("sim-aggro-val");

  // Colors
  const RED = "rgba(220, 55, 55, 0.95)";
  const BLUE = "rgba(55, 125, 235, 0.95)";
  const OCEAN = "#0b5f8a";
  const OCEAN_DARK = "#084c6e";
  const SAND = "#e9d2a6";
  const SAND_EDGE = "rgba(140, 100, 50, 0.35)";
  const TEXT = "rgba(255,255,255,0.92)";

  // Island geometry
  const islands = {
    A: { name:"A", label:"State A", x: 170, y: 235, r: 58, color: RED },
    C: { name:"C", label:"State C", x: 490, y: 145, r: 52, color: "rgba(0,0,0,0.18)" },
    B: { name:"B", label:"State B", x: 810, y: 235, r: 58, color: BLUE },
  };

  // Make missile range circle around C touch A and B:
  // choose radius so it touches A & B centerline roughly; set rRange = distance(C,A) - A.r (touch outer edge)
  const dist = (p,q) => Math.hypot(p.x-q.x, p.y-q.y);
  const rangeRadius = Math.min(
    dist(islands.C, islands.A) - islands.A.r,
    dist(islands.C, islands.B) - islands.B.r
  );

  // Simulation state
  let running = false;
  let ended = false;
  let lastTs = null;

  // Economies
  let econA = 100.0;
  let econB = 45.0;   // starts lower (catch-up)
  const baseGrowthA = 0.0022;  // per second baseline
  const baseGrowthB = 0.0018;  // baseline, then catch-up multiplies

  // Trade & dots
  let dots = [];
  let dotSpawnAcc = 0;

  // Military (constant share of economy)
  const milShare = 0.08; // 8% of economy supports active units

  // Occupation dynamics
  let occupyProgress = 0; // 0..1 capture
  let flashPhase = 0;

  // Helpers
  const clamp = (x,a,b) => Math.max(a, Math.min(b,x));
  const lerp = (a,b,t) => a + (b-a)*t;
  const sigmoid = (x) => 1/(1+Math.exp(-x));

  function reset() {
    running = false;
    ended = false;
    lastTs = null;

    econA = 100.0;
    econB = 45.0;

    dots = [];
    dotSpawnAcc = 0;

    occupyProgress = 0;
    flashPhase = 0;

    statusEl.textContent = "Ready.";
    draw();
  }

  // Trade dot model: each dot loops between home and C with curved motion.
  function spawnDot(from) {
    const src = islands[from];
    const mid = islands.C;

    // A curve control point (creates "back and around" motion)
    // randomize curvature
    const dir = (from === "A") ? -1 : 1;
    const ctrl = {
      x: lerp(src.x, mid.x, 0.5) + dir * (40 + Math.random()*60),
      y: lerp(src.y, mid.y, 0.5) - (30 + Math.random()*80)
    };

    dots.push({
      from,
      color: (from === "A") ? RED : BLUE,
      t: Math.random()*1,        // phase along curve
      speed: 0.12 + Math.random()*0.18,
      ctrl
    });
  }

  function bezier(p0, p1, p2, t) {
    // quadratic bezier
    const x = (1-t)*(1-t)*p0.x + 2*(1-t)*t*p1.x + t*t*p2.x;
    const y = (1-t)*(1-t)*p0.y + 2*(1-t)*t*p1.y + t*t*p2.y;
    return {x,y};
  }

  function drawOcean() {
    // base fill
    ctx.fillStyle = OCEAN;
    ctx.fillRect(0,0,canvas.width,canvas.height);

    // subtle waves
    ctx.save();
    ctx.globalAlpha = 0.10;
    ctx.strokeStyle = "#ffffff";
    ctx.lineWidth = 2;

    for (let i=0;i<11;i++) {
      const y = 30 + i*32;
      ctx.beginPath();
      for (let x=0;x<=canvas.width;x+=40) {
        const amp = 6 + (i%3)*2;
        const yy = y + Math.sin((x/90) + i)*amp;
        if (x===0) ctx.moveTo(x,yy);
        else ctx.lineTo(x,yy);
      }
      ctx.stroke();
    }
    ctx.restore();

    // darker gradient bottom
    ctx.save();
    const g = ctx.createLinearGradient(0, 0, 0, canvas.height);
    g.addColorStop(0, "rgba(0,0,0,0)");
    g.addColorStop(1, "rgba(0,0,0,0.22)");
    ctx.fillStyle = g;
    ctx.fillRect(0,0,canvas.width,canvas.height);
    ctx.restore();
  }

  function drawIsland(isl, outlineColor=null, fillBoost=0) {
    // sand base
    ctx.save();
    ctx.fillStyle = SAND;
    ctx.globalAlpha = 0.98;
    ctx.beginPath();
    ctx.arc(isl.x, isl.y, isl.r, 0, Math.PI*2);
    ctx.fill();
    ctx.restore();

    // beach edge
    ctx.save();
    ctx.strokeStyle = SAND_EDGE;
    ctx.lineWidth = 6;
    ctx.beginPath();
    ctx.arc(isl.x, isl.y, isl.r-2, 0, Math.PI*2);
    ctx.stroke();
    ctx.restore();

    // inner tint (subtle)
    ctx.save();
    ctx.globalAlpha = 0.10 + fillBoost;
    ctx.fillStyle = "#000";
    ctx.beginPath();
    ctx.arc(isl.x, isl.y, isl.r-10, 0, Math.PI*2);
    ctx.fill();
    ctx.restore();

    // outline
    ctx.save();
    ctx.lineWidth = 3;
    ctx.globalAlpha = 0.75;
    ctx.strokeStyle = outlineColor ? outlineColor : "rgba(0,0,0,0.25)";
    ctx.beginPath();
    ctx.arc(isl.x, isl.y, isl.r, 0, Math.PI*2);
    ctx.stroke();
    ctx.restore();

    // label
    ctx.save();
    ctx.fillStyle = TEXT;
    ctx.font = "700 14px system-ui, -apple-system, Segoe UI, Roboto, Arial";
    ctx.textAlign = "center";
    ctx.fillText(isl.label, isl.x, isl.y + isl.r + 22);
    ctx.restore();
  }

  function drawMissileRange() {
    // dashed circle around C touching A and B (outer edges)
    ctx.save();
    ctx.setLineDash([8, 10]);
    ctx.lineWidth = 3;
    ctx.strokeStyle = "rgba(255,255,255,0.55)";
    ctx.beginPath();
    ctx.arc(islands.C.x, islands.C.y, rangeRadius, 0, Math.PI*2);
    ctx.stroke();
    ctx.restore();

    // label
    ctx.save();
    ctx.fillStyle = "rgba(255,255,255,0.75)";
    ctx.font = "600 12px system-ui, -apple-system, Segoe UI, Roboto, Arial";
    ctx.textAlign = "center";
    ctx.fillText("Missile range (C)", islands.C.x, islands.C.y - rangeRadius - 10);
    ctx.restore();
  }

  function drawDots() {
    for (const d of dots) {
      const src = islands[d.from];
      const mid = islands.C;

      // move along loop: src -> C -> src
      // We'll map d.t in [0,1) and use a triangular wave for back-and-forth
      const phase = d.t % 1;
      const u = phase < 0.5 ? (phase*2) : (1 - (phase-0.5)*2);

      const pos = bezier(src, d.ctrl, mid, u);

      // small dot
      ctx.save();
      ctx.fillStyle = d.color;
      ctx.globalAlpha = 0.85;
      ctx.beginPath();
      ctx.arc(pos.x, pos.y, 4.2, 0, Math.PI*2);
      ctx.fill();
      ctx.restore();
    }
  }

  function drawMilitary(isl, econ, color) {
    // number of units proportional to economy * share
    // keep it bounded for visuals
    const units = Math.floor(clamp((econ * milShare) / 2.4, 0, 28));

    // place triangles in a ring-ish pattern
    const innerR = isl.r - 18;
    for (let i=0; i<units; i++) {
      const ang = (i/Math.max(units,1)) * Math.PI*2;
      const jitter = (Math.sin(i*13.7) * 3);
      const x = isl.x + Math.cos(ang) * (innerR + jitter);
      const y = isl.y + Math.sin(ang) * (innerR + jitter);

      // triangle size scales slightly with econ
      const s = 7 + clamp((econ/120), 0, 5);

      ctx.save();
      ctx.fillStyle = color;
      ctx.globalAlpha = 0.85;
      ctx.translate(x,y);
      ctx.rotate(ang + Math.PI/2);

      ctx.beginPath();
      ctx.moveTo(0, -s);
      ctx.lineTo(s*0.85, s);
      ctx.lineTo(-s*0.85, s);
      ctx.closePath();
      ctx.fill();

      ctx.restore();
    }
  }

  function drawHUD() {
    ctx.save();
    ctx.fillStyle = "rgba(0,0,0,0.20)";
    ctx.fillRect(14, 14, 350, 92);
    ctx.restore();

    ctx.save();
    ctx.fillStyle = TEXT;
    ctx.font = "700 14px system-ui, -apple-system, Segoe UI, Roboto, Arial";
    ctx.textAlign = "left";
    ctx.fillText(`Economy A: ${econA.toFixed(1)}   |   Economy B: ${econB.toFixed(1)}`, 24, 42);

    ctx.font = "600 13px system-ui, -apple-system, Segoe UI, Roboto, Arial";
    const ratio = (econB/econA);
    ctx.fillText(`B/A ratio: ${ratio.toFixed(2)}   |   Occupation risk: ${(occupationRisk()*100).toFixed(1)}%`, 24, 66);

    ctx.fillText(`Occupation progress: ${(occupyProgress*100).toFixed(1)}%`, 24, 90);
    ctx.restore();
  }

  function occupationRisk() {
    // Risk rises as B approaches and exceeds A.
    // Use a smooth S-curve around ratio ~0.85..1.15
    const ratio = econB / econA;
    const a = parseFloat(aggro.value);

    // x = 0 at ratio=1, negative below, positive above
    const x = (ratio - 1.0) * 6.0;
    const base = sigmoid(x) * 0.18; // max ~0.18 baseline per second-ish
    // also allow some risk when ratio approaches 1 from below
    const pre = sigmoid((ratio - 0.85) * 10) * 0.10;
    return clamp((base + pre) * a, 0, 0.35); // cap
  }

  function updateEconomies(dt) {
    // Wealth grows from trade volume: more dots => more trade => more growth
    const tradeIntensity = parseFloat(trade.value);

    // effective trade factor depends on number of dots (circulating flows)
    const tradeFactor = clamp(dots.length / 120, 0, 1.6);

    // A grows steady
    econA *= (1 + (baseGrowthA + 0.0040*tradeFactor*tradeIntensity) * dt);

    // B catch-up: higher growth multiplier while smaller; fades as it approaches A
    const catchMult = parseFloat(catchup.value);
    const gap = clamp((econA - econB) / econA, 0, 1); // 1 when far behind
    const catchBoost = 1 + gap * (catchMult - 1);

    econB *= (1 + (baseGrowthB*catchBoost + 0.0046*tradeFactor*tradeIntensity*catchBoost) * dt);
  }

  function updateDots(dt) {
    // Spawn rate increases with "trade intensity" and with total wealth
    const tradeIntensity = parseFloat(trade.value);

    const totalWealth = econA + econB;
    const wealthScale = clamp(totalWealth / 240, 0.6, 2.6);

    // dots per second baseline
    const spawnRate = 1.6 * tradeIntensity * wealthScale;

    dotSpawnAcc += dt * spawnRate;

    while (dotSpawnAcc >= 1) {
      dotSpawnAcc -= 1;
      // spawn from both sides, but bias slightly toward the faster-growing B as it integrates more
      const pB = clamp(0.45 + (econB/(econA+econB))*0.20, 0.35, 0.70);
      const from = (Math.random() < pB) ? "B" : "A";
      spawnDot(from);
    }

    // Move dots
    for (const d of dots) {
      d.t += dt * d.speed;
    }

    // Cap dot count for performance
    const cap = 280;
    if (dots.length > cap) dots.splice(0, dots.length - cap);
  }

  function updateOccupation(dt) {
    if (ended) return;

    const risk = occupationRisk();

    // Occupation progress increases stochastically with risk
    // Think of it as repeated crises/pressure that accumulate.
    const roll = Math.random();
    const step = risk * dt;

    // occasional jumps + steady drift
    occupyProgress += step * 0.9;
    if (roll < risk * 0.12 * dt * 60) {
      occupyProgress += 0.02 + 0.04*Math.random();
    }

    occupyProgress = clamp(occupyProgress, 0, 1);

    // Flashing intensifies as progress grows
    flashPhase += dt * (1.5 + occupyProgress*6.0);

    // End condition: C occupied by A (blue outline in your description was “occupied state end”,
    // but you described C flashing blue first as pressure grows; here we show pressure on C as "blue flash"
    // then final takeover is "blue outline + fill shift". You can invert colors if you meant A occupies = red.
    if (occupyProgress >= 1) {
      ended = true;
      running = false;
      statusEl.textContent = "Outcome: C is occupied. Simulation ended.";
    } else {
      statusEl.textContent = `Running.`;
    }
  }

  function draw() {
    // ocean
    drawOcean();

    // missile range (dashed circle around C)
    drawMissileRange();

    // trade dots
    drawDots();

    // islands (with C visual state changing)
    // C starts neutral; as occupation grows it flashes blue and gains blue outline.
    const flash = (Math.sin(flashPhase) * 0.5 + 0.5); // 0..1
    const cFlashStrength = (occupyProgress > 0.15) ? clamp((occupyProgress - 0.15) / 0.65, 0, 1) : 0;

    // If you want “A occupies C” to be RED instead of BLUE, swap BLUE->RED below.
    const cOutline = (ended || occupyProgress > 0.75) ? BLUE : (cFlashStrength > 0 ? `rgba(55,125,235,${0.25 + 0.55*cFlashStrength})` : null);

    const cFillBoost = (cFlashStrength > 0)
      ? (0.05 + 0.12*cFlashStrength*flash)
      : 0;

    drawIsland(islands.A);
    drawIsland(islands.C, cOutline, cFillBoost);
    drawIsland(islands.B);

    // Military units proportional to economy
    drawMilitary(islands.A, econA, RED);
    drawMilitary(islands.B, econB, BLUE);

    // If occupied, add heavier outline on C
    if (ended) {
      ctx.save();
      ctx.lineWidth = 6;
      ctx.strokeStyle = "rgba(55,125,235,0.95)";
      ctx.beginPath();
      ctx.arc(islands.C.x, islands.C.y, islands.C.r+2, 0, Math.PI*2);
      ctx.stroke();
      ctx.restore();
    }

    // HUD
    drawHUD();

    // End banner
    if (ended) {
      ctx.save();
      ctx.fillStyle = "rgba(0,0,0,0.45)";
      ctx.fillRect(0, canvas.height/2 - 44, canvas.width, 88);
      ctx.fillStyle = "rgba(255,255,255,0.95)";
      ctx.font = "800 22px system-ui, -apple-system, Segoe UI, Roboto, Arial";
      ctx.textAlign = "center";
      ctx.fillText("C OCCUPIED — INTERMEDIARY CONTROL SECURED", canvas.width/2, canvas.height/2 - 6);
      ctx.font = "600 14px system-ui, -apple-system, Segoe UI, Roboto, Arial";
      ctx.fillText("Toy model outcome: growth → threat perception → preemption", canvas.width/2, canvas.height/2 + 22);
      ctx.restore();
    }
  }

  function step(ts) {
    if (!running) return;
    if (lastTs === null) lastTs = ts;
    const dt = Math.min(0.04, (ts - lastTs) / 1000);
    lastTs = ts;

    if (!ended) {
      updateDots(dt);
      updateEconomies(dt);
      updateOccupation(dt);
    }

    draw();
    requestAnimationFrame(step);
  }

  function start() {
    if (ended) return; // if ended, use reset first
    if (running) return;
    running = true;
    lastTs = null;
    statusEl.textContent = "Running.";
    requestAnimationFrame(step);
  }

  function pause() {
    running = false;
    lastTs = null;
    statusEl.textContent = ended ? "Ended." : "Paused.";
  }

  // UI
  function syncLabels() {
    catchupVal.textContent = `${parseFloat(catchup.value).toFixed(1)}x`;
    tradeVal.textContent = `${parseFloat(trade.value).toFixed(1)}x`;
    aggroVal.textContent = `${parseFloat(aggro.value).toFixed(1)}x`;
  }

  catchup.addEventListener("input", syncLabels);
  trade.addEventListener("input", syncLabels);
  aggro.addEventListener("input", syncLabels);

  btnStart.addEventListener("click", start);
  btnPause.addEventListener("click", pause);
  btnReset.addEventListener("click", () => reset());

  syncLabels();
  reset();
})();
</script>
