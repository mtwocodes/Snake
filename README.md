# Snake
Snake game
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Snake — Classic</title>
<style>
  :root {
    --bg: #0f1115;
    --panel: #161a22;
    --text: #e8ecf1;
    --muted: #9aa4b2;
    --accent: #4ade80;  /* apple / success */
    --danger: #f87171;  /* game over */
    --snake: #60a5fa;
    --snake-head: #93c5fd;
    --grid: #222733;
  }
  * { box-sizing: border-box; }
  body {
    margin: 0;
    background: radial-gradient(1200px 800px at 70% 20%, #121722 0%, #0e121a 60%, #0b0f16 100%);
    font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Inter, Arial;
    color: var(--text);
    display: grid;
    place-items: center;
    min-height: 100svh;
    padding: 20px;
  }
  .wrap {
    width: min(92vw, 560px);
  }
  header {
    display: flex; align-items: center; justify-content: space-between;
    margin-bottom: 12px;
  }
  h1 { font-size: 18px; margin: 0; letter-spacing: .3px; color: #cfd6de; }
  .stats {
    display: flex; gap: 10px; align-items: center; font-weight: 600;
  }
  .badge {
    background: var(--panel);
    border: 1px solid #202634;
    padding: 6px 10px;
    border-radius: 8px;
    color: var(--muted);
    font-size: 12px;
  }
  .badge strong { color: var(--text); margin-left: 6px; }
  .frame {
    background: var(--panel);
    border: 1px solid #1d2230;
    border-radius: 14px;
    padding: 14px;
    box-shadow: 0 10px 30px rgba(0,0,0,.35), inset 0 0 0 1px rgba(255,255,255,.02);
  }
  canvas {
    width: 100%;
    aspect-ratio: 1 / 1;
    display: block;
    background:
      linear-gradient(var(--grid) 1px, transparent 1px) 0 0 / 100% calc(100%/24),
      linear-gradient(90deg, var(--grid) 1px, transparent 1px) 0 0 / calc(100%/24) 100%,
      #0b0f16;
    border-radius: 10px;
  }
  .controls {
    display: grid;
    grid-template-columns: 1fr auto;
    gap: 10px;
    margin-top: 12px;
  }
  .buttons {
    display: flex; gap: 8px; flex-wrap: wrap;
  }
  button {
    background: #1c2330;
    border: 1px solid #263044;
    color: var(--text);
    border-radius: 10px;
    padding: 10px 12px;
    font-weight: 600;
    cursor: pointer;
  }
  button:hover { border-color: #344055; }
  button:active { transform: translateY(1px); }
  .hint { color: var(--muted); font-size: 12px; text-align: right; }
  /* Touch D-pad */
  .dpad {
    user-select: none;
    display: grid;
    grid-template-columns: 60px 60px 60px;
    grid-template-rows: 60px 60px 60px;
    gap: 8px;
    justify-content: center;
    margin-top: 12px;
  }
  .dpad button {
    padding: 0;
    width: 60px; height: 60px;
    border-radius: 12px;
  }
  .dpad .blank { visibility: hidden; }
  .toast {
    position: absolute; inset: 0;
    display: grid; place-items: center;
    pointer-events: none;
  }
  .toast .card {
    background: rgba(11,15,22,.86);
    border: 1px solid #273043;
    padding: 18px 16px;
    border-radius: 12px;
    text-align: center;
    backdrop-filter: blur(6px);
  }
  .toast .title { font-weight: 800; margin-bottom: 8px; }
  .ok { color: var(--accent); }
  .bad { color: var(--danger); }
</style>
</head>
<body>
  <div class="wrap">
    <header>
      <h1>🐍 Snake — Classic</h1>
      <div class="stats">
        <div class="badge">Score <strong id="score">0</strong></div>
        <div class="badge">High <strong id="high">0</strong></div>
        <div class="badge">Speed <strong id="spd">1.0x</strong></div>
      </div>
    </header>

    <div class="frame" style="position:relative">
      <canvas id="game" width="600" height="600" aria-label="Snake game area"></canvas>
      <div class="toast" id="toast" hidden>
        <div class="card">
          <div class="title bad" id="toastTitle">Game Over</div>
          <div class="msg" id="toastMsg">Press R to restart</div>
        </div>
      </div>
    </div>

    <div class="controls">
      <div class="buttons">
        <button id="pauseBtn" aria-label="Pause (Space)">⏸ Pause</button>
        <button id="restartBtn" aria-label="Restart (R)">🔁 Restart</button>
        <button id="wrapBtn" aria-label="Toggle wrap walls">🚧 Walls: On</button>
      </div>
      <div class="hint">Keys: ⬅️⬆️⬇️➡️ / WASD · Pause: Space · Restart: R</div>
    </div>

    <!-- Touch D-Pad -->
    <div class="dpad" aria-hidden="false">
      <span class="blank"></span>
      <button data-dir="up">⬆️</button>
      <span class="blank"></span>
      <button data-dir="left">⬅️</button>
      <span class="blank"></span>
      <button data-dir="right">➡️</button>
      <span class="blank"></span>
      <button data-dir="down">⬇️</button>
      <span class="blank"></span>
    </div>
  </div>

<script>
(() => {
  // --- Config ---
  const COLS = 24, ROWS = 24;     // grid size
  const BASE_SPEED = 6;           // cells per second initially
  const GROWTH_PER_APPLE = 1;     // how many segments per apple
  const SPEED_STEP = 0.3;         // speed increase each apple
  const MAX_SPEED = 16;
  const STORAGE_KEY = "snake_highscore_v1";

  // --- Canvas & DPI scaling ---
  const canvas = document.getElementById('game');
  const ctx = canvas.getContext('2d');
  function resizeCanvasForDPI() {
    const dpr = Math.max(1, window.devicePixelRatio || 1);
    const cssSize = canvas.getBoundingClientRect();
    canvas.width = Math.round(cssSize.width * dpr);
    canvas.height = Math.round(cssSize.height * dpr);
    ctx.setTransform(dpr, 0, 0, dpr, 0, 0); // draw in CSS pixels
  }
  resizeCanvasForDPI();
  new ResizeObserver(resizeCanvasForDPI).observe(canvas);

  // --- State ---
  let snake, dir, lastDir, food, score, high, speed, wrapWalls, paused, alive;
  let accumulator = 0, prevTs = 0;

  // --- Helpers ---
  const randInt = (min, max) => Math.floor(Math.random() * (max - min + 1)) + min;
  const cellSize = () => Math.min(canvas.clientWidth / COLS, canvas.clientHeight / ROWS);

  function init() {
    snake = [
      {x: Math.floor(COLS/2), y: Math.floor(ROWS/2)},
      {x: Math.floor(COLS/2)-1, y: Math.floor(ROWS/2)},
      {x: Math.floor(COLS/2)-2, y: Math.floor(ROWS/2)}
    ];
    dir = {x: 1, y: 0};
    lastDir = {x: 1, y: 0};
    placeFood();
    score = 0;
    speed = BASE_SPEED;
    wrapWalls = false;
    paused = false;
    alive = true;
    accumulator = 0; prevTs = 0;
    updateHUD();
    hideToast();
    updateWrapButton();
    loop(0);
  }

  function updateHUD() {
    document.getElementById('score').textContent = score;
    document.getElementById('high').textContent = high ?? 0;
    document.getElementById('spd').textContent = `${(speed/BASE_SPEED).toFixed(1)}x`;
    document.getElementById('pauseBtn').textContent = paused ? "▶️ Resume" : "⏸ Pause";
  }

  function loadHigh() {
    try { high = parseInt(localStorage.getItem(STORAGE_KEY) || "0", 10); }
    catch { high = 0; }
    document.getElementById('high').textContent = high;
  }

  function saveHigh() {
    if (score > (high||0)) {
      high = score;
      try { localStorage.setItem(STORAGE_KEY, String(high)); } catch {}
    }
  }

  function placeFood() {
    const occupied = new Set(snake.map(n => `${n.x},${n.y}`));
    let x, y, tries = 0;
    do {
      x = randInt(0, COLS-1);
      y = randInt(0, ROWS-1);
      tries++;
      if (tries > 2000) break; // safety
    } while (occupied.has(`${x},${y}`));
    food = {x, y};
  }

  function setDirection(nx, ny) {
    // prevent instant reversal
    if (nx === -lastDir.x && ny === -lastDir.y) return;
    dir = {x: nx, y: ny};
  }

  function step() {
    if (paused || !alive) return;

    const head = snake[0];
    let nx = head.x + dir.x;
    let ny = head.y + dir.y;

    if (wrapWalls) {
      nx = (nx + COLS) % COLS;
      ny = (ny + ROWS) % ROWS;
    } else {
      // wall collision = death
      if (nx < 0 || nx >= COLS || ny < 0 || ny >= ROWS) {
        return gameOver("You hit a wall.");
      }
    }

    // self collision?
    for (let i = 0; i < snake.length; i++) {
      if (snake[i].x === nx && snake[i].y === ny) return gameOver("You bit yourself.");
    }

    // move
    snake.unshift({x: nx, y: ny});
    lastDir = { ...dir };

    // ate food?
    if (nx === food.x && ny === food.y) {
      score++;
      speed = Math.min(MAX_SPEED, speed + SPEED_STEP);
      placeFood();
    } else {
      // regular move: remove tail
      snake.pop();
    }
    updateHUD();
  }

  function gameOver(msg) {
    alive = false;
    saveHigh();
    showToast("Game Over", `${msg} Press R to restart.`, true);
    updateHUD();
  }

  // --- Render ---
  function draw() {
    const w = canvas.clientWidth, h = canvas.clientHeight;
    const cell = cellSize();
    // clear is unnecessary because background handles it; but we want crisp redraw:
    ctx.clearRect(0, 0, w, h);

    // draw food
    drawCell(food.x, food.y, "#10161f");
    drawApple(food.x, food.y);

    // draw snake
    for (let i = snake.length - 1; i >= 0; i--) {
      const c = snake[i];
      if (i === 0) {
        drawRoundedCell(c.x, c.y, 0.28, getHeadColor());
      } else {
        drawRoundedCell(c.x, c.y, 0.18, getBodyColor(i));
      }
    }
  }

  function drawCell(x, y, fill) {
    const s = canvas.clientWidth / COLS;
    ctx.fillStyle = fill;
    ctx.fillRect(x*s, y*s, s, s);
  }

  function drawRoundedCell(x, y, rRatio, fill) {
    const s = canvas.clientWidth / COLS;
    const r = s * rRatio;
    const px = x*s, py = y*s;
    ctx.fillStyle = fill;
    ctx.beginPath();
    ctx.moveTo(px+r, py);
    ctx.arcTo(px+s, py,   px+s, py+s, r);
    ctx.arcTo(px+s, py+s, px,   py+s, r);
    ctx.arcTo(px,   py+s, px,   py,   r);
    ctx.arcTo(px,   py,   px+s, py,   r);
    ctx.closePath();
    ctx.fill();
  }

  function drawApple(x, y) {
    const s = canvas.clientWidth / COLS;
    const px = x*s, py = y*s;
    ctx.save();
    ctx.translate(px + s/2, py + s/2);
    // body
    ctx.fillStyle = getComputedStyle(document.documentElement).getPropertyValue('--accent').trim();
    ctx.beginPath();
    ctx.ellipse(0, 0, s*0.32, s*0.28, 0, 0, Math.PI*2);
    ctx.fill();
    // leaf
    ctx.rotate(-0.2);
    ctx.fillStyle = "#7dd3fc";
    ctx.beginPath();
    ctx.ellipse(s*0.12, -s*0.28, s*0.12, s*0.06, 0, 0, Math.PI*2);
    ctx.fill();
    ctx.restore();
  }

  function getHeadColor()  { return getComputedStyle(document.documentElement).getPropertyValue('--snake-head').trim(); }
  function getBodyColor(i) { return getComputedStyle(document.documentElement).getPropertyValue('--snake').trim(); }

  // --- Loop with fixed timestep (grid speed independent of FPS) ---
  function loop(ts) {
    const dt = (ts - prevTs) / 1000 || 0;
    prevTs = ts;
    const stepTime = 1 / speed;
    accumulator += dt;
    while (accumulator >= stepTime) {
      step();
      accumulator -= stepTime;
    }
    draw();
    requestAnimationFrame(loop);
  }

  // --- UI: toast ---
  function showToast(title, msg, isBad=false) {
    const t = document.getElementById('toast');
    const ti = document.getElementById('toastTitle');
    const m = document.getElementById('toastMsg');
    ti.textContent = title;
    ti.className = 'title ' + (isBad ? 'bad' : 'ok');
    m.textContent = msg;
    t.hidden = false;
  }
  function hideToast() {
    document.getElementById('toast').hidden = true;
  }

  // --- Controls ---
  const keyMap = {
    ArrowLeft: [-1,0], ArrowUp: [0,-1], ArrowRight: [1,0], ArrowDown: [0,1],
    a: [-1,0], A: [-1,0], w: [0,-1], W: [0,-1], d: [1,0], D: [1,0], s: [0,1], S:[0,1]
  };
  window.addEventListener('keydown', (e) => {
    if (keyMap[e.key]) {
      const [x,y] = keyMap[e.key];
      setDirection(x,y);
      e.preventDefault();
    } else if (e.key === ' '){ // pause
      paused = !paused;
      hideToast();
      updateHUD();
      e.preventDefault();
    } else if (e.key === 'r' || e.key === 'R'){
      init();
    }
  }, {passive:false});

  // Touch D-pad
  document.querySelectorAll('.dpad button[data-dir]').forEach(btn => {
    btn.addEventListener('click', () => {
      const d = btn.getAttribute('data-dir');
      if (d === 'up') setDirection(0,-1);
      if (d === 'down') setDirection(0, 1);
      if (d === 'left') setDirection(-1,0);
      if (d === 'right') setDirection(1, 0);
    });
  });

  // Buttons
  document.getElementById('pauseBtn').addEventListener('click', () => {
    paused = !paused; hideToast(); updateHUD();
  });
  document.getElementById('restartBtn').addEventListener('click', init);
  document.getElementById('wrapBtn').addEventListener('click', () => {
    wrapWalls = !wrapWalls;
    updateWrapButton();
  });
  function updateWrapButton() {
    document.getElementById('wrapBtn').textContent = wrapWalls ? "🌍 Wrap: On" : "🚧 Walls: On";
  }

  // --- Start ---
  loadHigh();
  init();
})();
</script>
</body>
</html>