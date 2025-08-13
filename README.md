# Umbrella-dash
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1, user-scalable=no" />
  <title>Umbrella Ascent</title>
  <style>
    :root {
      --ui-bg: rgba(0,0,0,.35);
      --ui-text: #fff;
      --accent: #6ee7ff;
    }
    html, body {
      margin: 0; height: 100%; overflow: hidden; background: #60a5fa;
      font-family: system-ui, -apple-system, Segoe UI, Roboto, Ubuntu, Cantarell, "Helvetica Neue", Arial, "Noto Sans";
      color: var(--ui-text);
    }
    #game {
      position: fixed; inset: 0; display: block; width: 100vw; height: 100vh; touch-action: none;
    }
    .hud {
      position: fixed; left: 0; right: 0; top: 0; display: flex; gap: 12px; padding: 10px 12px; pointer-events: none;
      align-items: center; justify-content: space-between; font-weight: 700; text-shadow: 0 1px 2px rgba(0,0,0,.6);
    }
    .badge { background: var(--ui-bg); padding: 6px 10px; border-radius: 12px; backdrop-filter: blur(2px); }
    .bar { position: relative; width: 180px; height: 12px; background: rgba(255,255,255,.2); border-radius: 999px; overflow: hidden; }
    .bar > i { position: absolute; left: 0; top: 0; bottom: 0; background: linear-gradient(90deg, #34d399, #22d3ee); width: 0%; }

    .overlay { position: fixed; inset: 0; display: grid; place-items: center; background: rgba(10,15,30,.5); backdrop-filter: blur(2px); }
    .card { background: rgba(17,24,39,.8); border: 1px solid rgba(255,255,255,.12); padding: 24px; border-radius: 16px; box-shadow: 0 10px 30px rgba(0,0,0,.35); text-align: center; }
    .title { font-size: clamp(24px, 4vw, 40px); margin: 0 0 8px; letter-spacing: .5px; }
    .subtitle { opacity: .8; margin: 0 0 14px; }
    .btn { pointer-events: auto; cursor: pointer; display: inline-block; background: #22d3ee; color: #0b1020; border: 0; font-weight: 900; padding: 10px 16px; border-radius: 999px; box-shadow: 0 6px 14px rgba(34,211,238,.35); }
    .btn:active { transform: translateY(1px); }

    .touch-zones { position: fixed; inset: 0; display: grid; grid-template-columns: 1fr 1fr 1fr; pointer-events: none; }
    .zone { pointer-events: auto; }
    .zone::after { content: ""; position: absolute; inset: 0; background: transparent; transition: background .15s; }
    .zone:active::after { background: rgba(255,255,255,.06); }

    @media (pointer:fine) { .touch-hint { display: none; } }
  </style>
</head>
<body>
  <canvas id="game"></canvas>

  <div class="hud" aria-live="polite">
    <div class="badge" id="level">Lvl 1</div>
    <div class="badge" id="score">Score 0</div>
    <div class="bar" title="Boost / Stamina"><i id="staminaFill"></i></div>
  </div>

  <div class="touch-zones" id="touchZones" aria-hidden="true">
    <div class="zone" id="leftZone"></div>
    <div class="zone" id="boostZone"></div>
    <div class="zone" id="rightZone"></div>
  </div>

  <div class="overlay" id="menu">
    <div class="card">
      <h1 class="title">☂️ Umbrella Ascent</h1>
      <p class="subtitle">Reach the sky. Avoid hazards. Boost wisely.</p>
      <p class="subtitle touch-hint">Tap left/right to steer • Tap middle to boost</p>
      <button class="btn" id="startBtn">Start</button>
    </div>
  </div>

  <div class="overlay" id="gameOver" style="display:none">
    <div class="card">
      <h2 class="title">Game Over</h2>
      <p class="subtitle" id="finalScore">Score: 0</p>
      <button class="btn" id="restartBtn">Restart</button>
    </div>
  </div>

  <script>
    // ---------- Utilities ----------
    const clamp = (v, a, b) => Math.max(a, Math.min(b, v));
    const rand = (a, b) => a + Math.random() * (b - a);

    // ---------- Canvas ----------
    const canvas = document.getElementById('game');
    const ctx = canvas.getContext('2d');

    function resize() {
      const dpr = Math.min(window.devicePixelRatio || 1, 2);
      canvas.width = Math.floor(innerWidth * dpr);
      canvas.height = Math.floor(innerHeight * dpr);
      canvas.style.width = innerWidth + 'px';
      canvas.style.height = innerHeight + 'px';
      ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
    }
    addEventListener('resize', resize);
    resize();

    // ---------- Input ----------
    const keys = { left:false, right:false, boost:false };
    addEventListener('keydown', (e) => {
      if (["ArrowLeft","ArrowRight"," ","Space","a","A","d","D"].includes(e.key)) e.preventDefault();
      if (e.key === 'ArrowLeft' || e.key === 'a' || e.key === 'A') keys.left = true;
      if (e.key === 'ArrowRight' || e.key === 'd' || e.key === 'D') keys.right = true;
      if (e.code === 'Space') keys.boost = true;
    }, {passive:false});
    addEventListener('keyup', (e) => {
      if (e.key === 'ArrowLeft' || e.key === 'a' || e.key === 'A') keys.left = false;
      if (e.key === 'ArrowRight' || e.key === 'd' || e.key === 'D') keys.right = false;
      if (e.code === 'Space') keys.boost = false;
    });

    // Touch zones
    const leftZone = document.getElementById('leftZone');
    const rightZone = document.getElementById('rightZone');
    const boostZone = document.getElementById('boostZone');
    function bindZone(zone, on, off) {
      zone.addEventListener('pointerdown', (e)=>{ e.preventDefault(); on(); zone.setPointerCapture(e.pointerId); });
      zone.addEventListener('pointerup', ()=> off());
      zone.addEventListener('pointercancel', ()=> off());
      zone.addEventListener('pointerleave', ()=> off());
    }
    bindZone(leftZone, ()=> keys.left = true, ()=> keys.left = false);
    bindZone(rightZone, ()=> keys.right = true, ()=> keys.right = false);
    bindZone(boostZone, ()=> keys.boost = true, ()=> keys.boost = false);

    // ---------- Assets ----------
    // Inline SVG umbrella sprite
    const umbrellaSVG = `
      <svg xmlns='http://www.w3.org/2000/svg' width='80' height='80' viewBox='0 0 80 80'>
        <defs>
          <linearGradient id='g' x1='0' x2='0' y1='0' y2='1'>
            <stop offset='0%' stop-color='#ef4444'/>
            <stop offset='100%' stop-color='#f97316'/>
          </linearGradient>
        </defs>
        <path d='M8 40 Q40 8 72 40 Q56 36 40 36 Q24 36 8 40Z' fill='url(#g)' stroke='#b91c1c' stroke-width='2'/>
        <line x1='40' y1='36' x2='40' y2='64' stroke='#4b5563' stroke-width='3'/>
        <path d='M40 64 q0 10 10 10' fill='none' stroke='#4b5563' stroke-width='3' stroke-linecap='round'/>
      </svg>`;
    const umbrellaImg = new Image();
    umbrellaImg.src = 'data:image/svg+xml;base64,' + btoa(umbrellaSVG);

    // ---------- Audio (WebAudio; no files needed) ----------
    let audioCtx = null;
    function ensureAudio() {
      if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    }
    function sfxBoost() {
      if (!audioCtx) return;
      const o = audioCtx.createOscillator();
      const g = audioCtx.createGain();
      o.type = 'sawtooth';
      o.frequency.setValueAtTime(220, audioCtx.currentTime);
      o.frequency.exponentialRampTo
