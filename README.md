<!DOCTYPE html>
<html lang="zh-Hant">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no">
<title>Soul Game 原型</title>
<style>
  html,body{margin:0;height:100%;background:#070a10;overflow:hidden;touch-action:none;-webkit-user-select:none;user-select:none;
    font-family:'Microsoft JhengHei','PingFang TC','Noto Sans TC',sans-serif;color:#fff}
  #wrap{position:absolute;left:50%;top:50%;width:960px;height:540px;transform-origin:center center}
  canvas{position:absolute;left:0;top:0;width:960px;height:540px;background:#000;image-rendering:pixelated;touch-action:none}
  #ui{position:absolute;left:0;top:0;width:960px;height:540px;display:none}
  .bg{position:absolute;inset:0;background:rgba(7,11,18,.94);display:flex;flex-direction:column;align-items:center;justify-content:center;gap:10px;text-align:center}
  h1{margin:0;font-size:52px;color:#9bf0a8;letter-spacing:4px}
  h2{margin:0;font-size:30px;color:#ffe36e}
  h3{margin:4px 0;font-size:20px;color:#ffe36e}
  button{font:bold 16px 'Microsoft JhengHei','PingFang TC',sans-serif;padding:8px 18px;background:#2c8a50;color:#fff;border:3px solid #4fcf7a;cursor:pointer}
  button:disabled{background:#444;border-color:#666;color:#999;cursor:default}
  button.red{background:#8a2c3a;border-color:#cf4f5f}
  button.gray{background:#34414f;border-color:#5a6d82}
  .cards{display:flex;gap:12px}
  .card{width:270px;padding:10px;border:3px solid #2c4a5c;background:#14212e;font-size:14px;line-height:1.6;cursor:pointer}
  .card.sel{border-color:#ffe36e}
  .dim{color:#9aa8b5;font-size:13px}
  .row{display:flex;gap:10px;align-items:center;justify-content:center;flex-wrap:wrap}
  .slot{width:62px;height:62px;border:3px solid #5a6d82;background:#14212e;font-size:32px;font-weight:bold;display:flex;align-items:center;justify-content:center;cursor:pointer}
  .slot.g{background:#2c8a50;border-color:#4fcf7a}.slot.y{background:#b58a1a;border-color:#ffe36e}.slot.r{background:#8a2c3a;border-color:#cf4f5f}
  .tl{font-size:22px;font-weight:bold;color:#ff5a5a}
  .shopgrid{display:grid;grid-template-columns:repeat(5,170px);gap:8px}
  .item{background:#14212e;border:2px solid #2c4a5c;padding:6px;font-size:13px;min-height:92px}
  table{border-collapse:collapse;font-size:15px}td,th{border:1px solid #3a4a5c;padding:4px 14px}
</style>
</head>
<body>
<div id="wrap"><canvas id="c" width="960" height="540"></canvas><div id="ui"></div></div>
<script>
"use strict";
/* ============ 基本工具 ============ */
const CW = 960, CH = 540;
const cv = document.getElementById("c"), ctx = cv.getContext("2d");
const uiEl = document.getElementById("ui"), wrapEl = document.getElementById("wrap");
const rnd = (a, b) => a + Math.random() * (b - a);
const ri = (a, b) => Math.floor(rnd(a, b + 1));
const pick = a => a[Math.floor(Math.random() * a.length)];
const clamp = (v, a, b) => Math.max(a, Math.min(b, v));
const dist = (a, b) => Math.hypot(a.x - b.x, a.y - b.y);
const shuffle = a => { for (let i = a.length - 1; i > 0; i--) { const j = Math.floor(Math.random() * (i + 1)); [a[i], a[j]] = [a[j], a[i]]; } return a; };
function fit() { const s = Math.min(innerWidth / CW, innerHeight / CH); wrapEl.style.transform = "translate(-50%,-50%) scale(" + s + ")"; }
addEventListener("resize", fit); fit();

/* ============ 設定資料 ============ */
const WEAPONS = {
  pistol:  { name: "手槍", cost: 1, dmg: 3,   rate: 4,   spd: 520, count: 1, spread: 0.03, pierce: false, color: "#ffe36e" },
  wand:    { name: "魔杖", cost: 2, dmg: 4,   rate: 3,   spd: 460, count: 1, spread: 0.02, pierce: false, color: "#9fe0ff" },
  bow:     { name: "長弓", cost: 2, dmg: 3,   rate: 2.5, spd: 560, count: 3, spread: 0.2,  pierce: false, color: "#c8f0a0" },
  shotgun: { name: "霰彈槍", cost: 2, dmg: 2, rate: 1.8, spd: 480, count: 5, spread: 0.5,  pierce: false, color: "#ffb070" },
  mg:      { name: "機關槍", cost: 1, dmg: 1.5, rate: 9, spd: 540, count: 1, spread: 0.12, pierce: false, color: "#ffd0d0" },
  sniper:  { name: "狙擊槍", cost: 3, dmg: 12, rate: 1,  spd: 900, count: 1, spread: 0,    pierce: true,  color: "#ffffff" }
};
const CHARS = {
  knight: { name: "騎士", hp: 8, sh: 6, en: 200, weapon: "pistol", speed: 150, color: "#c9d4e0", accent: "#e0554a", cost: 0,
    passive: "近戰傷害 +100%", skill: { name: "衝鋒", cd: 6, dur: 0.3, desc: "向前衝刺並短暫無敵" } },
  mage:   { name: "法師", hp: 5, sh: 8, en: 300, weapon: "wand", speed: 145, color: "#8a6ae0", accent: "#5ee0f0", cost: 150,
    passive: "能量消耗 -25%", skill: { name: "冰環", cd: 10, dur: 3, desc: "凍結周圍敵人 3 秒" } },
  ranger: { name: "遊俠", hp: 6, sh: 5, en: 240, weapon: "bow", speed: 155, color: "#4fae5a", accent: "#e6c46a", cost: 250,
    passive: "子彈速度 +20%", skill: { name: "疾風", cd: 12, dur: 5, desc: "移速與射速提升 5 秒" } }
};
const FLOORS = [
  { t: "norm", n: 1, label: "第 1 層" }, { t: "norm", n: 2, label: "第 2 層" }, { t: "rest", n: 2, label: "休息層" },
  { t: "norm", n: 3, label: "第 3 層" }, { t: "norm", n: 4, label: "第 4 層" }, { t: "boss", n: 5, label: "第 5 層 最終 Boss" },
  { t: "reward", n: 5, label: "獎勵房間" }
];
const ET = { grunt: { r: 11, hp: 8, spd: 62 }, fast: { r: 9, hp: 4, spd: 108 }, shooter: { r: 10, hp: 6, spd: 46 },
  elite: { r: 20, hp: 90, spd: 48 }, boss: { r: 30, hp: 320, spd: 38 } };
const WORDS = ["SWORD", "GHOST", "FLAME", "STORM", "CRANE", "LIGHT", "BRAVE", "NIGHT", "PLANT", "STONE"];

/* ============ 局外資料 ============ */
let meta = { coins: 0, exp: 0, unlocked: ["knight"] };
try { const m = JSON.parse(localStorage.getItem("soulgame_meta")); if (m && m.unlocked) meta = m; } catch (e) {}
const saveMeta = () => { try { localStorage.setItem("soulgame_meta", JSON.stringify(meta)); } catch (e) {} };

/* ============ 音效 ============ */
const Snd = {
  ctx: null,
  init() {
    if (!this.ctx) { try { this.ctx = new (window.AudioContext || window.webkitAudioContext)(); } catch (e) {} }
    if (this.ctx && this.ctx.state === "suspended") { try { this.ctx.resume(); } catch (e) {} }
  },
  tone(f, d, type, v, to) {
    if (!this.ctx) return;
    try {
      const t = this.ctx.currentTime, o = this.ctx.createOscillator(), g = this.ctx.createGain();
      o.type = type || "square"; o.frequency.setValueAtTime(f, t);
      if (to) o.frequency.exponentialRampToValueAtTime(to, t + d);
      g.gain.setValueAtTime(v || 0.04, t); g.gain.exponentialRampToValueAtTime(0.0001, t + d);
      o.connect(g); g.connect(this.ctx.destination); o.start(t); o.stop(t + d);
    } catch (e) {}
  },
  play(n) {
    switch (n) {
      case "shoot": this.tone(500, 0.06, "square", 0.025, 250); break;
      case "hit": this.tone(200, 0.06, "square", 0.03, 120); break;
      case "hurt": this.tone(260, 0.25, "sawtooth", 0.05, 70); break;
      case "break": this.tone(1200, 0.2, "triangle", 0.06, 300); break;
      case "pick": this.tone(900, 0.06, "triangle", 0.04); break;
      case "ok": this.tone(660, 0.1, "square", 0.04); setTimeout(() => this.tone(990, 0.12, "square", 0.04), 90); break;
      case "bad": this.tone(150, 0.15, "square", 0.05, 90); break;
      case "skill": this.tone(300, 0.25, "sawtooth", 0.05, 800); break;
      case "win": [523, 659, 784, 1047].forEach((f, i) => setTimeout(() => this.tone(f, 0.18, "triangle", 0.06), i * 110)); break;
      case "lose": this.tone(300, 0.7, "sawtooth", 0.05, 60); break;
    }
  }
};

/* ============ 全域狀態 ============ */
const G = {
  screen: "title", paused: false, sel: "knight", bot: true,
  world: null, actors: [], P: null, B: null, enemies: [], bullets: [], pickups: [], texts: [], parts: [],
  shake: 0, floorIdx: 0, timeLeft: 600, gas: false, elapsed: 0, clues: 0, inserted: false, hasKey: false,
  spawnersDead: false, puzzleSolved: false, floorKills: 0, totalKills: 0, coinsRun: 0, lobbyBonus: 0, clock: 0,
  msg: "", msgT: 0, prompt: null, cam: { x: 0, y: 0 }, boss: null, totalPlayers: 1, solids: [], slot: null, ana: null, last: 0, ended: null
};
const toast = s => { G.msg = s; G.msgT = 2.6; };
const aliveCount = () => G.actors.filter(a => !a.dead).length;

/* ============ 輸入 ============ */
const keys = {}, M = { x: 480, y: 270, down: false, moved: false };
const T = { enabled: false, ih: -1,
  move: { active: false, id: -1, ox: 0, oy: 0, dx: 0, dy: 0 },
  aim: { active: false, id: -1, ox: 0, oy: 0, dx: 0, dy: 0, mag: 0 } };
const cpos = e => { const r = cv.getBoundingClientRect(); return { x: (e.clientX - r.left) * CW / r.width, y: (e.clientY - r.top) * CH / r.height }; };
const hitC = (p, x, y, r) => Math.hypot(p.x - x, p.y - y) < r;
addEventListener("keydown", ev => {
  if (["Space", "ArrowUp", "ArrowDown", "ArrowLeft", "ArrowRight"].includes(ev.code)) ev.preventDefault();
  Snd.init();
  if (ev.repeat) { keys[ev.code] = true; return; }
  keys[ev.code] = true; onKey(ev.code);
}, { passive: false });
addEventListener("keyup", ev => { keys[ev.code] = false; });
addEventListener("blur", () => { for (const k in keys) keys[k] = false; M.down = false; if (G.screen === "play") G.paused = true; });
cv.addEventListener("mousemove", e => { const p = cpos(e); M.x = p.x; M.y = p.y; M.moved = true; });
cv.addEventListener("mousedown", e => {
  Snd.init();
  if (G.screen === "slot") { slotConfirm(); return; }
  if (e.button === 0) M.down = true;
  if (e.button === 2 && G.screen === "play" && !G.paused) useSkill(G.P);
});
addEventListener("mouseup", () => { M.down = false; });
cv.addEventListener("contextmenu", e => e.preventDefault());
cv.addEventListener("pointerdown", e => {
  if (e.pointerType === "mouse") return;
  e.preventDefault(); T.enabled = true; Snd.init();
  const p = cpos(e);
  if (G.screen === "slot") { slotConfirm(); return; }
  if (G.screen !== "play") return;
  if (G.paused) { G.paused = false; return; }
  if (hitC(p, 870, 330, 46)) { useSkill(G.P); return; }
  if (hitC(p, 770, 385, 40)) { T.ih = e.pointerId; doInteract(); return; }
  if (hitC(p, 920, 36, 26)) { G.paused = true; return; }
  try { cv.setPointerCapture(e.pointerId); } catch (_) {}
  if (p.x < CW * 0.45) T.move = { active: true, id: e.pointerId, ox: p.x, oy: p.y, dx: 0, dy: 0 };
  else T.aim = { active: true, id: e.pointerId, ox: p.x, oy: p.y, dx: 0, dy: 0, mag: 0 };
}, { passive: false });
cv.addEventListener("pointermove", e => {
  if (e.pointerType === "mouse") return;
  e.preventDefault();
  const p = cpos(e);
  for (const k of ["move", "aim"]) {
    const s = T[k];
    if (s.active && s.id === e.pointerId) {
      let dx = (p.x - s.ox) / 60, dy = (p.y - s.oy) / 60; const m = Math.hypot(dx, dy);
      if (m > 1) { dx /= m; dy /= m; }
      s.dx = dx; s.dy = dy; s.mag = Math.min(1, m);
    }
  }
}, { passive: false });
const upT = e => {
  if (e.pointerId === T.ih) T.ih = -1;
  if (T.move.id === e.pointerId) T.move = { active: false, id: -1, ox: 0, oy: 0, dx: 0, dy: 0 };
  if (T.aim.id === e.pointerId) T.aim = { active: false, id: -1, ox: 0, oy: 0, dx: 0, dy: 0, mag: 0 };
};
cv.addEventListener("pointerup", upT); cv.addEventListener("pointercancel", upT);
document.addEventListener("touchmove", e => e.preventDefault(), { passive: false });

function onKey(code) {
  if (code === "KeyP") { if (G.screen === "play") G.paused = !G.paused; return; }
  if (code === "Escape") {
    if (G.screen === "slot" || G.screen === "anagram" || G.screen === "shop") { G.screen = "play"; setUI(""); }
    return;
  }
  if (G.screen === "slot") { if (code === "Space" || code === "Enter") slotConfirm(); return; }
  if (G.screen !== "play" || G.paused) return;
  if (code === "KeyE") doInteract();
  if (code === "Space") useSkill(G.P);
}
function moveVec() {
  let x = (keys.KeyD || keys.ArrowRight ? 1 : 0) - (keys.KeyA || keys.ArrowLeft ? 1 : 0);
  let y = (keys.KeyS || keys.ArrowDown ? 1 : 0) - (keys.KeyW || keys.ArrowUp ? 1 : 0);
  if (T.move.active) { x += T.move.dx; y += T.move.dy; }
  const l = Math.hypot(x, y); if (l > 1) { x /= l; y /= l; }
  return [x, y];
}

/* ============ 角色與敵人建立 ============ */
function mkActor(def, isBot, name) {
  return { isBot, name, def, x: 0, y: 0, r: 10, hp: def.hp, maxHp: def.hp, sh: def.sh, maxSh: def.sh, en: def.en, maxEn: def.en,
    coins: 0, aim: 0, face: 1, downed: false, dead: false, revive: 0, reviving: false, hitT: 99, flash: 0, fireCd: 0, meleeCd: 0,
    skillCd: 0, skillT: 0, inv: 0, kills: 0, taken: 0, dashV: null, anim: 0, moving: false, weapon: isBot ? null : WEAPONS[def.weapon] };
}
function spawnEnemy(kind, x, y) {
  const t = ET[kind], n = FLOORS[G.floorIdx].n, alive = aliveCount();
  let hp = t.hp * (1 + 0.3 * (n - 1));
  if (kind === "elite") hp = t.hp * (1 + 0.25 * (n - 1));
  if (kind === "elite" || kind === "boss") hp *= 1 - 0.25 * (G.totalPlayers - alive);
  const e = { kind, x, y, r: t.r, hp, maxHp: hp, spd: t.spd, flash: 0, frozen: 0, atk: 0, anim: rnd(0, 6), dead: false,
    shootT: rnd(0.8, 2), pat: 0, chargeT: 5, charge: 0, cdir: 0, rot: 0 };
  G.enemies.push(e);
  if (kind === "elite" || kind === "boss") G.boss = e;
  return e;
}

/* ============ 地圖生成 ============ */
function placeFurn(r, k, out) {
  let placed = 0, tries = 0;
  while (placed < k && tries < 80) {
    tries++;
    const w = ri(36, 64), h = ri(26, 44);
    const fx = ri(r.x + 60, r.x + r.w - 60 - w), fy = ri(r.y + 40, r.y + r.h - 40 - h);
    if (fy < 50 && fy + h > -50 && (fx < r
