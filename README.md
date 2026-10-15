
<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>DOOM - Green Ray Survival</title>
<style>
* { box-sizing: border-box; }
body {
  margin: 0;
  background: #020905;
  color: #d8ffe0;
  font-family: system-ui, sans-serif;
  overflow: hidden;
}
#game {
  position: relative;
  width: 100%;
  height: 100vh;
  height: 100dvh;
  overflow: hidden;
  background:
    linear-gradient(#00140799, #000800cc),
    url("doom.png") center 22% / cover,
    #07150b;
  isolation: isolate;
}
#game::before {
  content: "";
  position: absolute;
  inset: 0;
  background: radial-gradient(ellipse at 50% 40%,
    transparent 10%, #000900bb 100%);
  pointer-events: none;
  z-index: -1;
}
header {
  position: absolute;
  z-index: 5;
  top: 20px;
  left: 0;
  width: 100%;
  text-align: center;
  pointer-events: none;
}
h1 {
  margin: 0;
  font-size: clamp(23px, 5vw, 42px);
  letter-spacing: .15em;
  color: #c7ffb5;
  text-shadow: 0 0 15px #39ff57;
}
.sub {
  margin-top: 6px;
  font-size: 11px;
  letter-spacing: .3em;
  color: #77d989;
}
#hud {
  position: absolute;
  z-index: 5;
  top: 105px;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  gap: 22px;
  white-space: nowrap;
  font-size: 14px;
  font-weight: 800;
}
#hud span { color: #8cff78; }
canvas {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
}
#message {
  position: absolute;
  z-index: 6;
  top: 50%;
  left: 50%;
  width: 90%;
  transform: translate(-50%, -50%);
  text-align: center;
  padding: 25px 18px;
  background: #031509d9;
  border: 1px solid #58ff6c88;
  box-shadow: 0 0 40px #1cff4c22;
  backdrop-filter: blur(8px);
}
#message h2 {
  color: #a6ff8b;
  font-size: clamp(22px, 5vw, 36px);
  margin: 0 0 12px;
  text-shadow: 0 0 18px #28ff4a77;
}
#message p {
  color: #c2d9c5;
  font-size: 13px;
  line-height: 1.9;
}
button {
  color: #041006;
  font-weight: 900;
  letter-spacing: .08em;
  background: linear-gradient(135deg, #caff8a, #39e95c);
  border: 0;
  padding: 15px 28px;
  cursor: pointer;
  box-shadow: 0 0 25px #38ff5977;
}
button:hover { filter: brightness(1.15); }
#controls {
  position: absolute;
  z-index: 5;
  bottom: max(24px, env(safe-area-inset-bottom));
  left: 0;
  width: 100%;
  display: flex;
  justify-content: space-between;
  padding: 0 8%;
  pointer-events: none;
}
.move {
  pointer-events: auto;
  width: 76px;
  height: 65px;
  border: 1px solid #85ff9c88;
  color: white;
  background: #063915aa;
  font-size: 25px;
  box-shadow: 0 0 18px #23ff4822;
  touch-action: none;
}
.move:active { background: #24a944; }
#hint {
  position: absolute;
  z-index: 5;
  bottom: 112px;
  width: 100%;
  text-align: center;
  color: #9fc4a3;
  font-size: 11px;
  pointer-events: none;
}
.hidden { display: none !important; }
</style>
</head>
<body>
<div id="game">
  <header>
    <h1>DOOM</h1>
    <div class="sub">GREEN RAY SURVIVAL</div>
  </header>

  <div id="hud">
    <div>TIME <span id="time">0.0</span></div>
    <div>BEST <span id="best">0.0</span></div>
  </div>

  <canvas id="canvas"></canvas>

  <div id="message">
    <h2 id="title">魔法光線を回避せよ</h2>
    <p id="description">
      ドゥームの緑の魔法が迫ってくる。<br>
      光線を避けて、どこまで生き残れるか？<br>
      PC：← → キー / A・Dキー<br>
      スマホ：左右のボタン
    </p>
    <button id="start">SURVIVE</button>
  </div>

  <div id="hint">WARNING — GREEN MAGIC DETECTED</div>

  <div id="controls">
    <button class="move" id="left" aria-label="左へ移動">◀</button>
    <button class="move" id="right" aria-label="右へ移動">▶</button>
  </div>
</div>

<script>
const canvas = document.getElementById("canvas");
const ctx = canvas.getContext("2d");
const $ = id => document.getElementById(id);

let W, H, dpr;
let running = false;
let last = 0;
let elapsed = 0;
let score = 0;
let best = 0;
let player;
let beams = [];
let particles = [];
let warning = null;
let spawnTimer = 0;
let shake = 0;
let keys = { left: false, right: false };
let animationId = 0;

function resize() {
  W = innerWidth;
  H = innerHeight;
  dpr = Math.min(devicePixelRatio || 1, 2);
  canvas.width = W * dpr;
  canvas.height = H * dpr;
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
  if (player) player.y = H * .79;
}
addEventListener("resize", resize);
resize();

function getBest() {
  try {
    return Number(localStorage.getItem("doomBest") || 0);
  } catch {
    return 0;
  }
}

best = getBest();
$("best").textContent = best.toFixed(1);

function resetGame() {
  elapsed = 0;
  score = 0;
  beams = [];
  particles = [];
  warning = null;
  spawnTimer = 1;
  shake = 0;
  player = {
    x: W / 2,
    y: H * .79,
    r: 15,
    speed: Math.max(270, W * .65)
  };
}

function startGame() {
  cancelAnimationFrame(animationId);
  resetGame();
  running = true;
  last = performance.now();
  $("message").classList.add("hidden");
  $("hint").textContent = "SURVIVE THE MAGIC";
  animationId = requestAnimationFrame(loop);
}

function endGame() {
  running = false;
  best = Math.max(best, elapsed);
  try {
    localStorage.setItem("doomBest", best);
  } catch {}

  $("best").textContent = best.toFixed(1);
  $("title").textContent = "YOU WERE CONSUMED";
  $("description").innerHTML =
    `生存時間：${elapsed.toFixed(1)} 秒<br>` +
    `最高記録：${best.toFixed(1)} 秒<br>` +
    `ドゥームの魔法を突破できるか？`;
  $("start").textContent = "RETRY";
  $("message").classList.remove("hidden");
  $("hint").textContent = "THE POWER OF DOOM IS ABSOLUTE";
}

$("start").addEventListener("click", startGame);

addEventListener("keydown", e => {
  if (["ArrowLeft", "ArrowRight", "a", "d", "A", "D"].includes(e.key)) {
    e.preventDefault();
  }
  if (e.key === "ArrowLeft" || e.key.toLowerCase() === "a")
    keys.left = true;
  if (e.key === "ArrowRight" || e.key.toLowerCase() === "d")
    keys.right = true;
});
addEventListener("keyup", e => {
  if (e.key === "ArrowLeft" || e.key.toLowerCase() === "a")
    keys.left = false;
  if (e.key === "ArrowRight" || e.key.toLowerCase() === "d")
    keys.right = false;
});
addEventListener("blur", () => {
  keys.left = false;
  keys.right = false;
});

function bindMove(id, direction) {
  const el = $(id);
  el.addEventListener("pointerdown", e => {
    e.preventDefault();
    keys[direction] = true;
    el.setPointerCapture(e.pointerId);
  });
  const stop = () => keys[direction] = false;
  el.addEventListener("pointerup", stop);
  el.addEventListener("pointercancel", stop);
  el.addEventListener("lostpointercapture", stop);
}
bindMove("left", "left");
bindMove("right", "right");

// 地面の魔法陣と発射予告
function createWarning() {
  const horizontal = Math.random() < .72;
  warning = {
    horizontal,
    pos: horizontal
      ? H * (.25 + Math.random() * .43)
      : W * (.12 + Math.random() * .76),
    age: 0,
    duration: Math.max(.48, 1.05 - elapsed * .008),
    width: 20 + Math.min(elapsed, 40) * .7
  };
}

function fireBeam() {
  if (!warning) return;
  const w = warning;
  beams.push({
    horizontal: w.horizontal,
    pos: w.pos,
    width: w.width,
    age: 0,
    duration: .38,
    hit: false
  });
  warning = null;
  shake = Math.max(shake, 5);
}

function burst(x, y, count) {
  for (let i = 0; i < count; i++) {
    const a = Math.random() * Math.PI * 2;
    const s = 50 + Math.random() * 280;
    particles.push({
      x, y,
      vx: Math.cos(a) * s,
      vy: Math.sin(a) * s,
      life: .35 + Math.random() * .55,
      maxLife: .9,
      size: 2 + Math.random() * 5
    });
  }
}

function update(dt) {
  elapsed += dt;
  score = elapsed;
  $("time").textContent = score.toFixed(1);

  if (keys.left) player.x -= player.speed * dt;
  if (keys.right) player.x += player.speed * dt;
  player.x = Math.max(18, Math.min(W - 18, player.x));

  spawnTimer -= dt;
  if (spawnTimer <= 0 && !warning) {
    createWarning();
    spawnTimer = Math.max(.52, 1.6 - elapsed * .012) +
                 Math.random() * .45;
  }

  if (warning) {
    warning.age += dt;
    if (warning.age >= warning.duration) fireBeam();
  }

  for (const b of beams) {
    b.age += dt;
    if (!b.hit && b.age < .18) {
      const danger = b.width / 2 + player.r * .7;
      const hit = b.horizontal
        ? Math.abs(player.y - b.pos) < danger
        : Math.abs(player.x - b.pos) < danger;
      if (hit) {
        b.hit = true;
        burst(player.x, player.y, 55);
        shake = 18;
        endGame();
        return;
      }
    }
  }
  beams = beams.filter(b => b.age < b.duration);

  for (const p of particles) {
    p.x += p.vx * dt;
    p.y += p.vy * dt;
    p.vx *= .97;
    p.vy *= .97;
    p.life -= dt;
  }
  particles = particles.filter(p => p.life > 0);
  shake = Math.max(0, shake - dt * 35);
}

function glowLine(horizontal, pos, width, alpha) {
  ctx.save();
  ctx.globalAlpha = alpha;
  ctx.strokeStyle = "#63ff43";
  ctx.lineWidth = width;
  ctx.shadowColor = "#38ff2c";
  ctx.shadowBlur = 28;

  ctx.beginPath();
  if (horizontal) {
    ctx.moveTo(-20, pos);
    ctx.lineTo(W + 20, pos);
  } else {
    ctx.moveTo(pos, -20);
    ctx.lineTo(pos, H + 20);
  }
  ctx.stroke();

  ctx.shadowBlur = 0;
  ctx.strokeStyle = "#e1ffbc";
  ctx.lineWidth = Math.max(2, width * .12);
  ctx.stroke();
  ctx.restore();
}

function drawBackground() {
  // 魔法の霧
  const g = ctx.createRadialGradient(
    W * .5, H * .5, 0, W * .5, H * .5, H * .7
  );
  g.addColorStop(0, "#1aff3210");
  g.addColorStop(1, "#00000055");
  ctx.fillStyle = g;
  ctx.fillRect(0, 0, W, H);

  // 床の遠近グリッド
  ctx.save();
  ctx.strokeStyle = "#59ff6a15";
  ctx.lineWidth = 1;
  for (let i = 0; i < 12; i++) {
    const y = H * .45 + i * i * H * .004;
    ctx.beginPath();
    ctx.moveTo(0, y);
    ctx.lineTo(W, y);
    ctx.stroke();
  }
  for (let i = -8; i <= 8; i++) {
    ctx.beginPath();
    ctx.moveTo(W / 2, H * .44);
    ctx.lineTo(W / 2 + i * W * .13, H);
    ctx.stroke();
  }
  ctx.restore();
}

function drawPlayer() {
  if (!player) return;
  const x = player.x, y = player.y;

  ctx.save();
  ctx.translate(x, y);

  // 足元の緑の魔法シールド
  ctx.strokeStyle = "#6bff84";
  ctx.lineWidth = 2;
  ctx.shadowColor = "#3dff63";
  ctx.shadowBlur = 20;
  ctx.beginPath();
  ctx.ellipse(0, 13, 24, 8, 0, 0, Math.PI * 2);
  ctx.stroke();

  // プレイヤー（SFスーツ）
  ctx.shadowBlur = 12;
  ctx.fillStyle = "#c7f8d2";
  ctx.beginPath();
  ctx.arc(0, -17, 8, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = "#4bff75";
  ctx.fillRect(-10, -8, 20, 24);
  ctx.fillStyle = "#d5ffe0";
  ctx.fillRect(-15, -5, 5, 15);
  ctx.fillRect(10, -5, 5, 15);
  ctx.fillStyle = "#1e7c3c";
  ctx.fillRect(-8, 15, 6, 12);
  ctx.fillRect(2, 15, 6, 12);
  ctx.restore();
}

function draw() {
  ctx.clearRect(0, 0, W, H);
  ctx.save();
  if (shake > 0) {
    ctx.translate(
      (Math.random() - .5) * shake,
      (Math.random() - .5) * shake
    );
  }

  drawBackground();

  // 予告ライン：時間とともに明るくなる
  if (warning) {
    const progress = warning.age / warning.duration;
    const pulse = .25 + Math.abs(Math.sin(warning.age * 22)) * .75;
    glowLine(warning.horizontal, warning.pos,
      warning.width * (.5 + progress), pulse * .85);

    ctx.fillStyle = "#d7ffb8";
    ctx.font = "bold 12px system-ui";
    ctx.textAlign = "center";
    ctx.fillText("⚠ MAGIC LOCK", W / 2, H * .19);
  }

  // 発射された魔法光線
  for (const b of beams) {
    const alpha = Math.max(0, 1 - b.age / b.duration);
    glowLine(b.horizontal, b.pos, b.width, alpha);
  }

  // 魔法の破片
  for (const p of particles) {
    ctx.globalAlpha = Math.max(0, p.life / p.maxLife);
    ctx.fillStyle = "#9aff9a";
    ctx.shadowColor = "#35ff55";
    ctx.shadowBlur = 12;
    ctx.fillRect(p.x, p.y, p.size, p.size);
  }
  ctx.globalAlpha = 1;
  ctx.shadowBlur = 0;

  drawPlayer();
  ctx.restore();
}

function loop(now) {
  if (!running) return;
  const dt = Math.min((now - last) / 1000, .035);
  last = now;
  update(dt);
  draw();
  if (running) animationId = requestAnimationFrame(loop);
}



// 初期画面
resetGame();
draw();
</script>
</body>
</html>
# doom-game
