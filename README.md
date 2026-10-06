<!doctype html>
<html lang="ru">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no">
<meta name="theme-color" content="#080b16">
<title>Night Drive</title>
<style>
  * { box-sizing: border-box; }
  body {
    margin: 0; overflow: hidden; background: #080b16; color: white;
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
    touch-action: none;
  }
  canvas { display: block; width: 100vw; height: 100dvh; }
  #top {
    position: fixed; z-index: 2; top: max(18px, env(safe-area-inset-top));
    left: 18px; right: 18px; display: flex; align-items: flex-start;
    justify-content: space-between; pointer-events: none;
  }
  .label { color: #63e5ff; font-size: 11px; letter-spacing: 3px; font-weight: 900; }
  .sub { color: #b7bdcb; font-size: 11px; margin-top: 4px; }
  #score {
    font-size: 22px; font-weight: 900; font-variant-numeric: tabular-nums;
    background: #111827dd; border: 1px solid #ffffff33;
    border-radius: 15px; padding: 7px 12px;
  }
  #stats {
    position: fixed; z-index: 2; top: max(74px, calc(env(safe-area-inset-top) + 56px));
    left: 18px; display: flex; gap: 8px; pointer-events: none;
  }
  .badge {
    padding: 6px 9px; border-radius: 12px; background: #111827cc;
    border: 1px solid #ffffff22; font-size: 12px; font-weight: 800;
  }
  #panel {
    position: fixed; z-index: 3; left: 24px; right: 24px; top: 48%;
    transform: translateY(-50%); text-align: center; padding: 22px 18px;
    border: 1px solid #ffffff30; border-radius: 24px;
    background: #080b16ed; box-shadow: 0 12px 50px #0009;
  }
  #panel h1 { margin: 0 0 8px; font-size: 23px; letter-spacing: 2px; }
  #panel p { margin: 0 0 18px; color: #c0c6d3; font-size: 14px; line-height: 1.5; }
  button {
    border: 0; color: #061018; background: #61e6ff; border-radius: 16px;
    font-size: 15px; font-weight: 900; padding: 14px 25px;
  }
  #controls {
    position: fixed; z-index: 2; left: 0; right: 0;
    bottom: max(22px, env(safe-area-inset-bottom));
    display: flex; justify-content: center; align-items: center; gap: 22px;
  }
  .arrow {
    width: 64px; height: 56px; color: white; font-size: 25px;
    background: #ffffff20; border: 1px solid #ffffff35;
    padding: 0; touch-action: manipulation;
  }
  #hint { color: #d3d8e2; font-size: 10px; letter-spacing: 2px; }
</style>
</head>
<body>
<canvas id="game"></canvas>

<div id="top">
  <div><div class="label">NIGHT DRIVE</div><div class="sub">Ночная трасса</div></div>
  <div id="score">0000</div>
</div>

<div id="stats">
  <div class="badge">🪙 <span id="coins">0</span></div>
  <div class="badge" id="power">БОНУСОВ НЕТ</div>
</div>

<div id="panel">
  <h1 id="title">ГОТОВ К ЗАЕЗДУ?</h1>
  <p id="message">Собирай монеты и бонусы.<br>Уворачивайся от машин свайпом или кнопками.</p>
  <button id="start">ПОЕХАЛИ</button>
</div>

<div id="controls">
  <button class="arrow" id="left" aria-label="Влево">←</button>
  <span id="hint">ПЕРЕСТРОЕНИЕ</span>
  <button class="arrow" id="right" aria-label="Вправо">→</button>
</div>

<script>
const canvas = document.querySelector("#game");
const ctx = canvas.getContext("2d");
const panel = document.querySelector("#panel");
const scoreBox = document.querySelector("#score");
const coinsBox = document.querySelector("#coins");
const powerBox = document.querySelector("#power");
const title = document.querySelector("#title");
const message = document.querySelector("#message");
const startButton = document.querySelector("#start");

let w, h, dpr;
let playerLane = 1, targetLane = 1;
let score = 0, coins = 0, running = false;
let objects = [];
let lastTime = 0, spawnTimer = 0, coinTimer = 0, bonusTimer = 0;
let roadOffset = 0, boostTime = 0, shieldTime = 0, hitFlash = 0;

function resize() {
  dpr = Math.min(devicePixelRatio || 1, 2);
  w = innerWidth;
  h = innerHeight;
  canvas.width = Math.floor(w * dpr);
  canvas.height = Math.floor(h * dpr);
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
}
addEventListener("resize", resize);
resize();

function roadHalf(z) {
  return w * (0.12 + 0.27 * z);
}
function roadY(z) {
  return h * (0.20 + 0.63 * z);
}
function laneX(lane, z) {
  return w / 2 + (lane - 1) * roadHalf(z) * 0.66;
}
function rounded(x, y, width, height, radius, color) {
  ctx.fillStyle = color;
  ctx.beginPath();
  ctx.roundRect(x, y, width, height, radius);
  ctx.fill();
}

function drawRoad() {
  const horizon = h * 0.20;
  const bottom = h * 0.92;
  const topHalf = roadHalf(0);
  const bottomHalf = roadHalf(1);

  const sky = ctx.createLinearGradient(0, 0, 0, horizon);
  sky.addColorStop(0, "#101832");
  sky.addColorStop(1, "#342b53");
  ctx.fillStyle = sky;
  ctx.fillRect(0, 0, w, horizon + 4);

  // Далёкие огни города
  for (let i = 0; i < 22; i++) {
    const x = (i * 71 + 23) % w;
    const bh = 8 + (i * 13 % 25);
    ctx.fillStyle = i % 3 === 0 ? "#a06bff66" : "#6bdfff55";
    ctx.fillRect(x, horizon - bh, 4 + (i % 3), bh);
  }

  // Трапециевидная дорога создаёт перспективу
  ctx.beginPath();
  ctx.moveTo(w / 2 - topHalf, horizon);
  ctx.lineTo(w / 2 + topHalf, horizon);
  ctx.lineTo(w / 2 + bottomHalf, bottom);
  ctx.lineTo(w / 2 - bottomHalf, bottom);
  ctx.closePath();
  const roadGrad = ctx.createLinearGradient(0, horizon, 0, bottom);
  roadGrad.addColorStop(0, "#222431");
  roadGrad.addColorStop(1, "#11141c");
  ctx.fillStyle = roadGrad;
  ctx.fill();

  // Неоновые края дороги
  ctx.lineWidth = 3;
  ctx.strokeStyle = "#59eaffbb";
  ctx.beginPath();
  ctx.moveTo(w/2-topHalf, horizon);
  ctx.lineTo(w/2-bottomHalf, bottom);
  ctx.moveTo(w/2+topHalf, horizon);
  ctx.lineTo(w/2+bottomHalf, bottom);
  ctx.stroke();

  // Движущаяся разметка
  for (let i = 0; i < 12; i++) {
    let z = ((i / 12 + roadOffset) % 1);
    const y = roadY(z);
    const half = roadHalf(z);
    const dashH = 3 + z * 27;
    ctx.fillStyle = `rgba(255,255,255,${0.18 + z * 0.35})`;
    for (const divider of [-1/3, 1/3]) {
      const x = w/2 + half * divider;
      ctx.fillRect(x - 1, y, 2 + z * 2, dashH);
    }
  }

  // Боковые огни
  for (let i = 0; i < 11; i++) {
    const z = ((i / 11 + roadOffset * 0.7) % 1);
    const y = roadY(z);
    const half = roadHalf(z);
    const size = 2 + z * 5;
    ctx.fillStyle = "#56eaffcc";
    ctx.fillRect(w/2-half-size*3, y, size, size*2.6);
    ctx.fillStyle = "#c07affcc";
    ctx.fillRect(w/2+half+size, y, size, size*2.6);
  }
}

function drawVehicle(x, y, z, kind, color, player=false) {
  const scale = 0.26 + z * 0.83;
  const width = (kind === "van" ? 56 : kind === "sport" ? 46 : 51) * scale;
  const height = (kind === "van" ? 77 : kind === "sport" ? 66 : 73) * scale;

  ctx.save();
  ctx.translate(x, y);
  ctx.scale(scale, scale);

  // Тень
  ctx.fillStyle = "#0009";
  ctx.beginPath();
  ctx.ellipse(0, height * 0.48, width * 0.68, height * 0.18, 0, 0, Math.PI*2);
  ctx.fill();

  // Колёса
  rounded(-width*.53/scale, -height*.25/scale, 9, height*.26/scale, 3, "#050609");
  rounded(width*.36/scale, -height*.25/scale, 9, height*.26/scale, 3, "#050609");
  rounded(-width*.53/scale, height*.12/scale, 9, height*.26/scale, 3, "#050609");
  rounded(width*.36/scale, height*.12/scale, 9, height*.26/scale, 3, "#050609");

  // Корпус разных типов машин
  const bodyW = width / scale;
  const bodyH = height / scale;
  let bodyY = -bodyH/2;
  let bodyColor = color;

  if (player && boostTime > 0) bodyColor = "#13d8ff";

  const gradient = ctx.createLinearGradient(-bodyW/2, bodyY, bodyW/2, bodyY+bodyH);
  gradient.addColorStop(0, "#ffffff88");
  gradient.addColorStop(0.23, bodyColor);
  gradient.addColorStop(1, "#10131b");
  ctx.fillStyle = gradient;

  ctx.beginPath();
  if (kind === "sport") {
    ctx.moveTo(-bodyW*.46, bodyY+bodyH*.22);
    ctx.lineTo(-bodyW*.32, bodyY+bodyH*.04);
    ctx.lineTo(bodyW*.30, bodyY+bodyH*.04);
    ctx.lineTo(bodyW*.47, bodyY+bodyH*.25);
    ctx.lineTo(bodyW*.46, bodyY+bodyH*.86);
    ctx.quadraticCurveTo(0, bodyY+bodyH*1.04, -bodyW*.46, bodyY+bodyH*.86);
  } else {
    ctx.roundRect(-bodyW/2, bodyY, bodyW, bodyH, kind === "van" ? 7 : 11);
  }
  ctx.closePath();
  ctx.fill();

  // Окна
  ctx.fillStyle = "#101a2b";
  ctx.beginPath();
  if (kind === "van") {
    ctx.roundRect(-bodyW*.34, bodyY+bodyH*.13, bodyW*.68, bodyH*.32, 5);
  } else {
    ctx.moveTo(-bodyW*.30, bodyY+bodyH*.16);
    ctx.lineTo(-bodyW*.20, bodyY+bodyH*.09);
    ctx.lineTo(bodyW*.20, bodyY+bodyH*.09);
    ctx.lineTo(bodyW*.31, bodyY+bodyH*.30);
    ctx.lineTo(bodyW*.30, bodyY+bodyH*.42);
    ctx.lineTo(-bodyW*.30, bodyY+bodyH*.42);
  }
  ctx.closePath();
  ctx.fill();

  // Блики и фары
  ctx.fillStyle = player ? "#e7fcff" : "#ff6976";
  rounded(-bodyW*.34, bodyY+bodyH*.82, bodyW*.20, 4, 2, ctx.fillStyle);
  rounded(bodyW*.14, bodyY+bodyH*.82, bodyW*.20, 4, 2, ctx.fillStyle);

  ctx.restore();
}

function drawCoin(obj) {
  const z = obj.z, x = laneX(obj.lane, z), y = roadY(z);
  const size = 5 + z * 13;
  const spin = Math.abs(Math.cos(performance.now() / 170));
  ctx.save();
  ctx.translate(x, y);
  ctx.scale(0.45 + spin * 0.55, 1);
  ctx.shadowColor = "#ffd84a";
  ctx.shadowBlur = 14;
  ctx.fillStyle = "#ffd84a";
  ctx.beginPath();
  ctx.arc(0, 0, size, 0, Math.PI*2);
  ctx.fill();
  ctx.shadowBlur = 0;
  ctx.strokeStyle = "#fff3a6";
  ctx.lineWidth = Math.max(1, size*.16);
  ctx.stroke();
  ctx.restore();
}

function drawBonus(obj) {
  const x = laneX(obj.lane, obj.z), y = roadY(obj.z);
  const size = 9 + obj.z * 11;
  const color = obj.kind === "shield" ? "#55eaff" : "#ff9d42";
  ctx.save();
  ctx.shadowColor = color;
  ctx.shadowBlur = 16;
  ctx.fillStyle = color;
  ctx.beginPath();
  ctx.arc(x, y, size, 0, Math.PI*2);
  ctx.fill();
  ctx.shadowBlur = 0;
  ctx.fillStyle = "#10131b";
  ctx.font = `bold ${Math.max(10,size)}px sans-serif`;
  ctx.textAlign = "center";
  ctx.textBaseline = "middle";
  ctx.fillText(obj.kind === "shield" ? "S" : "⚡", x, y);
  ctx.restore();
}

function draw() {
  ctx.clearRect(0, 0, w, h);
  drawRoad();

  // Дальние объекты рисуются первыми
  const sorted = [...objects].sort((a,b) => a.z-b.z);
  for (const obj of sorted) {
    if (obj.type === "car") {
      const kinds = { sedan: ["#f04455"], sport: ["#a64dff"], van: ["#e38a34"] };
      drawVehicle(laneX(obj.lane,obj.z), roadY(obj.z), obj.z,
        obj.kind, kinds[obj.kind][0]);
    } else if (obj.type === "coin") {
      drawCoin(obj);
    } else {
      drawBonus(obj);
    }
  }

  drawVehicle(laneX(playerLane,.86), roadY(.86), .86, "sedan", "#087ff5", true);

  if (hitFlash > 0) {
    ctx.fillStyle = `rgba(255,40,60,${hitFlash*.35})`;
    ctx.fillRect(0,0,w,h);
  }
}

function setPowerText() {
  const active = [];
  if (shieldTime > 0) active.push("🛡 ЩИТ");
  if (boostTime > 0) active.push("⚡ УСКОРЕНИЕ");
  powerBox.textContent = active.join(" · ") || "БОНУСОВ НЕТ";
}

function endGame() {
  running = false;
  title.textContent = "ЗАЕЗД ОКОНЧЕН";
  message.innerHTML = `Счёт: <b>${score}</b> · Монеты: <b>${coins}</b><br>Попробуешь ещё раз?`;
  startButton.textContent = "ЕЩЁ РАЗ";
  panel.style.display = "block";
}

function spawn(type, extra={}) {
  objects.push({type, lane: Math.floor(Math.random()*3), z: 0, ...extra});
}

function update(dt) {
  if (!running) return;

  // Мягкое перестроение
  playerLane += (targetLane-playerLane) * Math.min(1, dt*10);

  boostTime = Math.max(0, boostTime-dt);
  shieldTime = Math.max(0, shieldTime-dt);
  hitFlash = Math.max(0, hitFlash-dt*2);
  setPowerText();

  const speed = (0.40 + Math.min(score*.0015, .22)) * (boostTime > 0 ? 1.65 : 1);
  roadOffset = (roadOffset + dt*speed*.62) % 1;

  spawnTimer += dt;
  coinTimer += dt;
  bonusTimer += dt;

  if (spawnTimer > Math.max(.72, 1.35-score*.006)) {
    spawnTimer = 0;
    const types = ["sedan", "sport", "van"];
    spawn("car", {kind: types[Math.floor(Math.random()*types.length)]});
  }
  if (coinTimer > 1.05) {
    coinTimer = 0;
    spawn("coin");
  }
  if (bonusTimer > 8) {
    bonusTimer = 0;
    spawn("bonus", {kind: Math.random() < .5 ? "shield" : "boost"});
  }

  for (const obj of objects) obj.z += dt*speed;
  const playerZ = .86;

  for (const obj of objects) {
    if (obj.used || Math.abs(obj.lane-playerLane) > .38) continue;

    if (obj.type === "car" && Math.abs(obj.z-playerZ) < .055) {
      if (shieldTime > 0) {
        shieldTime = 0;
        obj.used = true;
        hitFlash = .45;
      } else {
        endGame();
        return;
      }
    }

    if ((obj.type === "coin" || obj.type === "bonus") &&
        Math.abs(obj.z-playerZ) < .065) {
      obj.used = true;
      if (obj.type === "coin") {
        coins++;
        score += 50;
        coinsBox.textContent = coins;
      } else if (obj.kind === "shield") {
        shieldTime = 7;
      } else {
        boostTime = 5;
      }
      setPowerText();
    }
  }

  objects = objects.filter(obj => obj.z < 1.08 && !obj.used);
  score += dt * 4;
  scoreBox.textContent = String(Math.floor(score)).padStart(4,"0");
}

function frame(time) {
  const dt = Math.min((time-lastTime)/1000 || 0, .04);
  lastTime = time;
  update(dt);
  draw();
  requestAnimationFrame(frame);
}

function move(direction) {
  if (running) targetLane = Math.max(0, Math.min(2, targetLane+direction));
}

document.querySelector("#left").addEventListener("pointerdown", e => {
  e.preventDefault(); move(-1);
});
document.querySelector("#right").addEventListener("pointerdown", e => {
  e.preventDefault(); move(1);
});

let touchStartX = null;
canvas.addEventListener("touchstart", e => {
  touchStartX = e.changedTouches[0].clientX;
}, {passive:true});
canvas.addEventListener("touchend", e => {
  if (touchStartX === null) return;
  const dx = e.changedTouches[0].clientX-touchStartX;
  if (Math.abs(dx) > 28) move(dx > 0 ? 1 : -1);
  touchStartX = null;
}, {passive:true});

addEventListener("keydown", e => {
  if (e.key === "ArrowLeft") move(-1);
  if (e.key === "ArrowRight") move(1);
});

startButton.addEventListener("click", () => {
  playerLane = 1;
  targetLane = 1;
  score = 0;
  coins = 0;
  objects = [];
  spawnTimer = 0;
  coinTimer = 0;
  bonusTimer = 0;
  boostTime = 0;
  shieldTime = 0;
  scoreBox.textContent = "0000";
  coinsBox.textContent = "0";
  powerBox.textContent = "БОНУСОВ НЕТ";
  running = true;
  panel.style.display = "none";
});

requestAnimationFrame(frame);
</script>
</body>
</html>
