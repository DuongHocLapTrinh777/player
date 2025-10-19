<!doctype html>
<html lang="vi">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>Pro 2-Player HTML Game — Local Coop / PvP — Bots Waves (Updated Controls)</title>
<style>
  :root{
    --bg:#0b1220; --panel:rgba(10,14,22,0.6); --accent:#ffb86b; --muted:#9aa4b2;
  }
  html,body{height:100%;margin:0;background:var(--bg);color:#e6eef6;font-family:Inter, system-ui, Roboto, Arial;}
  #app{display:flex;flex-direction:column;height:100vh;align-items:stretch;}
  header{display:flex;align-items:center;justify-content:space-between;padding:10px 14px;background:linear-gradient(180deg,rgba(255,255,255,0.02),transparent);}
  header .left, header .right{display:flex;align-items:center;gap:8px;}
  button{background:var(--panel);border:1px solid rgba(255,255,255,0.04);color:#e6eef6;padding:8px 10px;border-radius:8px;cursor:pointer;}
  button:active{transform:translateY(1px);}
  #canvas-wrap{flex:1;display:flex;align-items:center;justify-content:center;padding:10px;}
  canvas{background:linear-gradient(#7fb8ff,#3a7fbf);box-shadow:0 8px 40px rgba(2,6,20,0.6);image-rendering:pixelated;}
  .hud{position:fixed;right:18px;top:12px;display:flex;flex-direction:column;gap:10px;z-index:10}
  .hp{display:flex;align-items:center;gap:8px;background:var(--panel);padding:8px;border-radius:8px;width:300px}
  .hp .bar{flex:1;height:12px;background:#222;border-radius:6px;overflow:hidden}
  .hp .bar > i{display:block;height:100%;background:linear-gradient(90deg,#ff6b6b,#ffb86b);width:100%}
  .label{font-size:13px;color:var(--muted)}
  .center-ui{position:fixed;left:50%;top:72px;transform:translateX(-50%);z-index:11}
  .small{font-size:12px;color:var(--muted)}
  footer{padding:8px 14px;background:linear-gradient(0deg,transparent,rgba(255,255,255,0.01));display:flex;justify-content:space-between;align-items:center;font-size:13px;color:var(--muted)}
  .panel{background:var(--panel);padding:8px;border-radius:8px}
  @media (max-width:880px){
    canvas{width:100%;height:auto}
    .hp{width:200px}
  }
</style>
</head>
<body>
<div id="app">
  <header>
    <div class="left">
      <div style="font-weight:700;font-size:16px">Pro Local 2-Player — Bots Waves</div>
      <div class="small" style="margin-left:8px">Co-op & PvP — Desktop keyboard</div>
    </div>
    <div class="right">
      <button id="modeBtn">Mode: Coop</button>
      <button id="pauseBtn">Pause</button>
      <button id="resetBtn">Reset</button>
    </div>
  </header>

  <div id="canvas-wrap">
    <canvas id="game" width="1280" height="720"></canvas>
  </div>

  <div class="hud">
    <div class="hp panel">
      <div style="width:82px;font-weight:600">Player 1</div>
      <div class="bar"><i id="p1bar"></i></div>
      <div id="p1hp" class="label" style="min-width:36px;text-align:right">100</div>
    </div>
    <div class="hp panel">
      <div style="width:82px;font-weight:600">Player 2</div>
      <div class="bar"><i id="p2bar" style="background:linear-gradient(90deg,#6be3ff,#6bb3ff)"></i></div>
      <div id="p2hp" class="label" style="min-width:36px;text-align:right">100</div>
    </div>
  </div>

  <div class="center-ui">
    <div id="tips" class="panel small">
      P1: A/D di chuyển · W nhảy · Q đánh  —  P2: Left/Right di chuyển · Up nhảy · Numpad0 đánh  · Nhấn M để đổi chế độ
    </div>
  </div>

  <footer>
    <div>Waves of bots in Coop mode — bots require 3-4 hits to die. Waves wait 5s between them. Bots block each other and players.</div>
    <div class="small">Open in desktop browser (NumLock for numpad)</div>
  </footer>
</div>

<script>
/*
  Updated controls & camera behavior:
  - Player1: A/D move, W jump, Q attack
  - Player2: Left/Right move, Up jump, Numpad0 attack
  - If a player falls into the void (too far below), they are teleported back to their spawn (no death).
  - When one player dies, camera focuses fully on the surviving player (zoom & center).
  - Waves/bots logic unchanged: waves spawn, need 3-4 hits to die, 5s between waves.
  - Bots and players cannot pass through each other; separation logic preserves that.
*/

"use strict";

/* -----------------------------
   Utility & Config
   ----------------------------- */
const CANVAS_W = 1280, CANVAS_H = 720;
const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d', {alpha:false});
const DPR = Math.min(window.devicePixelRatio || 1, 2);
canvas.style.width = CANVAS_W + 'px';
canvas.style.height = CANVAS_H + 'px';
canvas.width = CANVAS_W * DPR;
canvas.height = CANVAS_H * DPR;
ctx.setTransform(DPR,0,0,DPR,0,0);
ctx.imageSmoothingEnabled = false;

const GRAVITY = 2000; // px/s^2
const FLOOR_Y = 520;
const VOID_FALL_LIMIT = FLOOR_Y + 400; // if player.y > this => teleport back to spawn
const DEBUG = false;

/* -----------------------------
   Input system
   ----------------------------- */
class Input {
  constructor(){
    this.keys = new Map();
    this.down = new Map();
    window.addEventListener('keydown', e => {
      this.keys.set(e.code, true);
      if (!this.down.get(e.code)) this.down.set(e.code, true);
      if (["ArrowUp","ArrowDown","ArrowLeft","ArrowRight","Space"].includes(e.code)) e.preventDefault();
    });
    window.addEventListener('keyup', e => {
      this.keys.set(e.code, false);
      this.down.set(e.code, false);
    });
  }
  is(code){ return !!this.keys.get(code); }
  justDown(code){ if (!code) return false; if (this.down.get(code)) { this.down.set(code,false); return true; } return false; }
}
const input = new Input();

/* -----------------------------
   Audio (procedural FX)
   ----------------------------- */
class Sfx {
  constructor(){
    this.ctx = null;
    try { this.ctx = new (window.AudioContext || window.webkitAudioContext)(); } catch(e) { this.ctx = null; }
  }
  beep(freq=440, t=0.06, type='sine', vol=0.15){
    if (!this.ctx) return;
    const o = this.ctx.createOscillator();
    const g = this.ctx.createGain();
    o.type = type; o.frequency.value = freq;
    g.gain.value = vol;
    o.connect(g); g.connect(this.ctx.destination);
    const now = this.ctx.currentTime;
    o.start(now); g.gain.exponentialRampToValueAtTime(0.001, now + t);
    o.stop(now + t + 0.02);
  }
  hit(){ this.beep(880,0.06,'square',0.16); }
  jump(){ this.beep(520,0.09,'sine',0.12); }
  attack(){ this.beep(660,0.06,'sawtooth',0.14); }
  botAttack(){ this.beep(300,0.08,'sawtooth',0.16); }
}
const sfx = new Sfx();

/* -----------------------------
   Particle system (lightweight)
   ----------------------------- */
class PS {
  constructor(){ this.p = []; }
  spawn(x,y,opts={}){
    const life = opts.life || (0.45 + Math.random()*0.25);
    const size = opts.size || (4 + Math.random()*8);
    const vx = opts.vx !== undefined ? opts.vx : (Math.random()*200-100);
    const vy = opts.vy !== undefined ? opts.vy : (Math.random()*-140 - 40);
    const color = opts.color || 'rgba(255,255,255,0.9)';
    this.p.push({x,y,vx,vy,size,life,age:0,color,fade:true});
  }
  update(dt){
    for (let i=this.p.length-1;i>=0;i--){
      const pa = this.p[i];
      pa.age += dt;
      pa.vy += GRAVITY * 0.4 * dt;
      pa.x += pa.vx * dt; pa.y += pa.vy * dt;
      if (pa.age >= pa.life) this.p.splice(i,1);
    }
  }
  draw(ctx,cam){
    for (let pa of this.p){
      const a = Math.max(0,1 - pa.age/pa.life);
      ctx.globalAlpha = a;
      ctx.fillStyle = pa.color;
      ctx.beginPath();
      ctx.ellipse(pa.x-cam.x, pa.y-cam.y, pa.size*a, pa.size*a, 0, 0, Math.PI*2);
      ctx.fill();
    }
    ctx.globalAlpha = 1;
  }
}
const particles = new PS();

/* -----------------------------
   Camera
   ----------------------------- */
class Camera {
  constructor(){ this.x = 0; this.y = 0; this.zoom = 1; this.targetZoom = 1; }
  followBetween(a,b,dt){
    const cx = (a.x + b.x)/2;
    const cy = (a.y + b.y)/2;
    const targetX = cx - CANVAS_W/2;
    const targetY = cy - CANVAS_H/2;
    this.x += (targetX - this.x) * Math.min(1, 6*dt);
    this.y += (targetY - this.y) * Math.min(1, 6*dt);
    const dist = Math.hypot(a.x-b.x, a.y-b.y);
    this.targetZoom = Math.max(0.7, Math.min(1.1, 1.0 - (dist - 220)/1400));
    this.zoom += (this.targetZoom - this.zoom) * Math.min(1, 3*dt);
  }
  followSingle(p, dt){
    const targetX = p.x + p.w/2 - CANVAS_W/2;
    const targetY = p.y + p.h/2 - CANVAS_H/2;
    this.x += (targetX - this.x) * Math.min(1, 6*dt);
    this.y += (targetY - this.y) * Math.min(1, 6*dt);
    this.targetZoom = 1.05; // zoom closer when single player alive
    this.zoom += (this.targetZoom - this.zoom) * Math.min(1, 4*dt);
  }
}
const camera = new Camera();

/* -----------------------------
   World & Entities
   ----------------------------- */
class Platform {
  constructor(x,y,w,h, color='#5b3d2b'){ this.x=x;this.y=y;this.w=w;this.h=h;this.color=color; }
  rect(){ return {x:this.x,y:this.y,w:this.w,h:this.h}; }
  draw(ctx,cam){
    const dx = this.x - cam.x, dy=this.y - cam.y;
    ctx.fillStyle = this.color;
    roundRect(ctx, dx, dy, this.w, this.h, 6);
    ctx.fill();
    ctx.fillStyle = 'rgba(255,255,255,0.04)';
    ctx.fillRect(dx, dy, this.w, 6);
  }
}

class Player {
  constructor(opts){
    this.x = opts.x; this.y = opts.y;
    this.startX = opts.x; this.startY = opts.y; // spawn point to teleport back
    this.w = 44; this.h = 72;
    this.vx = 0; this.vy = 0;
    this.speed = opts.speed || 360;
    this.jumpForce = opts.jumpForce || 720;
    this.maxJumps = 1;
    this.jumpsLeft = this.maxJumps;
    this.facing = 1;
    this.color = opts.color;
    this.name = opts.name;
    this.hp = 100;
    this.attackTimer = 0;
    this.attackDur = 0.12;
    this.attackCd = 0.28;
    this.attackCdTimer = 0;
    this.attackRange = {w:62,h:32,ox:36,oy:22};
    this.onGround = false;
    this.coyote = 0;
    this.jumpBuffer = 0;
    this.hitFlash = 0;
  }
  bbox(){ return {x:this.x, y:this.y, w:this.w, h:this.h}; }
  center(){ return {x:this.x+this.w/2, y:this.y+this.h/2}; }

  update(dt, inputMap, otherPlayer, bots, mode){
    const alive = this.hp > 0;

    // Input horizontal
    let dir = 0;
    if (alive){
      if (input.is(inputMap.left)) dir -= 1;
      if (input.is(inputMap.right)) dir += 1;
      this.vx = dir * this.speed;
      if (dir !== 0) this.facing = dir > 0 ? 1 : -1;

      // Jump buffer & coyote
      if (input.justDown(inputMap.jump) || (inputMap.altJump && input.justDown(inputMap.altJump))) this.jumpBuffer = 0.12;
      this.jumpBuffer = Math.max(0, this.jumpBuffer - dt);
      this.coyote = Math.max(0, this.coyote - dt);

      if (this.jumpBuffer > 0 && (this.onGround || this.coyote > 0 || this.jumpsLeft > 0)){
        this.vy = -this.jumpForce;
        this.onGround = false;
        this.coyote = 0;
        if (!this.onGround) this.jumpsLeft = Math.max(0, this.jumpsLeft-1);
        particles.spawn(this.x + this.w/2, this.y + this.h, {life:0.45, size:8, color:'rgba(200,180,140,0.9)'});
        sfx.jump();
        this.jumpBuffer = 0;
      }

      // Attack against bots and optionally other player
      if ((input.justDown(inputMap.attack) || (inputMap.altAttack && input.justDown(inputMap.altAttack))) && this.attackCdTimer <= 0 && this.attackTimer<=0){
        this.attackTimer = this.attackDur; this.attackCdTimer = this.attackCd;
        sfx.attack();
        for (let i=0;i<8;i++) particles.spawn(this.x + this.w/2 + (this.facing*20), this.y + this.h/2 + (Math.random()*20-10), {vx:(Math.random()*200-100)+this.facing*100, vy:Math.random()*-180, life:0.35, size:4, color:'rgba(255,230,120,0.95)'});
        const ar = this.attackAABB();
        for (let b of bots){
          if (rectIntersect(ar, b.bbox())){
            b.takeDamage(18, {x: this.facing * 160, y: -200});
          }
        }
        if (mode === 'PvP' && otherPlayer && rectIntersect(ar, otherPlayer.bbox())){
          otherPlayer.takeDamage(18, {x: this.facing * 180, y: -200});
        }
      }
    }

    // Integrate physics
    this.vy += GRAVITY * dt;
    this.x += this.vx * dt;
    this.y += this.vy * dt;

    // If fell into void, teleport back to spawn (no death)
    if (this.y > VOID_FALL_LIMIT){
      this.teleportToSpawn();
    }

    // timers
    this.attackTimer = Math.max(0, this.attackTimer - dt);
    this.attackCdTimer = Math.max(0, this.attackCdTimer - dt);
    if (this.hitFlash>0) this.hitFlash = Math.max(0, this.hitFlash - dt);
  }

  teleportToSpawn(){
    this.x = this.startX;
    this.y = this.startY;
    this.vx = 0; this.vy = 0;
    // small effect
    for (let i=0;i<8;i++) particles.spawn(this.x + this.w/2 + Math.random()*20-10, this.y + this.h/2 + Math.random()*20-10, {life:0.6, size:6, color:'rgba(160,200,255,0.9)'});
    sfx.jump();
  }

  resolvePlatforms(platforms){
    this.onGround = false;
    for (let p of platforms){
      if (aabbIntersect(this.bbox(), p.rect())){
        const overlapX = Math.min(this.x+this.w, p.x+p.w) - Math.max(this.x, p.x);
        const overlapY = Math.min(this.y+this.h, p.y+p.h) - Math.max(this.y, p.y);
        if (overlapY < overlapX){
          if (this.y < p.y){
            this.y = p.y - this.h;
            this.vy = 0;
            this.onGround = true;
            this.jumpsLeft = this.maxJumps;
            this.coyote = 0.08;
          } else {
            this.y = p.y + p.h;
            this.vy = 0;
          }
        } else {
          if (this.x < p.x) this.x = p.x - this.w;
          else this.x = p.x + p.w;
          this.vx = 0;
        }
      }
    }
  }

  attackAABB(){
    const aw = this.attackRange.w, ah = this.attackRange.h;
    const ax = (this.facing>0) ? this.x + this.w + this.attackRange.ox : this.x - aw - this.attackRange.ox;
    const ay = this.y + this.attackRange.oy;
    return {x: ax, y: ay, w: aw, h: ah};
  }

  takeDamage(amount, knockback){
    if (this.hp <= 0) return;
    this.hp = Math.max(0, this.hp - amount);
    this.vx = knockback.x; this.vy = knockback.y;
    this.hitFlash = 0.16;
    particles.spawn(this.x + this.w/2, this.y + this.h/2, {life:0.6, size:10, color:'rgba(255,120,120,0.95)'});
    sfx.hit();
  }

  draw(ctx, cam){
    const x = this.x - cam.x, y = this.y - cam.y;
    ctx.save();
    ctx.globalAlpha = 0.6;
    ctx.fillStyle = 'rgba(10,10,10,0.35)';
    roundRect(ctx, x+6, FLOOR_Y - cam.y + 4, this.w*1.1, 8, 6);
    ctx.fill();
    ctx.restore();

    ctx.fillStyle = (this.hitFlash>0) ? '#ffffff' : this.color;
    roundRect(ctx, x, y, this.w, this.h, 8);
    ctx.fill();

    ctx.fillStyle = '#0b1220';
    const ex = x + (this.facing>0 ? this.w-12 : 8), ey = y + 18;
    ctx.fillRect(ex, ey, 6, 6);

    if (this.attackTimer > 0){
      const ar = this.attackAABB();
      ctx.save(); ctx.globalAlpha = 0.12; ctx.fillStyle = '#ffeb8a';
      rect(ctx, ar.x - cam.x, ar.y - cam.y, ar.w, ar.h); ctx.fill(); ctx.restore();
    }
  }
}

class Bot {
  constructor(x,y){
    this.x = x; this.y = y;
    this.w = 44; this.h = 72;
    this.vx = 0; this.vy = 0;
    this.speed = 160 + Math.random()*40;
    this.maxHp = (Math.random() < 0.5) ? 54 : 72;
    this.hp = this.maxHp;
    this.attackTimer = 0;
    this.attackDur = 0.12;
    this.attackCd = 0.9 + Math.random()*0.6;
    this.attackCdTimer = 0;
    this.attackRange = {w:52,h:26,ox:26,oy:28};
    this.hitFlash = 0;
  }
  bbox(){ return {x:this.x, y:this.y, w:this.w, h:this.h}; }

  update(dt, players, platforms){
    let target = null; let minDist = 1e9;
    for (let pl of players){
      if (pl.hp <= 0) continue;
      const dx = (pl.x + pl.w/2) - (this.x + this.w/2);
      const dy = (pl.y + pl.h/2) - (this.y + this.h/2);
      const d = Math.hypot(dx, dy);
      if (d < minDist){ minDist = d; target = pl; }
    }
    if (target){
      const dir = Math.sign((target.x + target.w/2) - (this.x + this.w/2));
      this.vx = dir * this.speed;
      if (minDist < 72 && this.attackCdTimer <= 0 && this.attackTimer <= 0){
        this.attackTimer = this.attackDur; this.attackCdTimer = this.attackCd;
        sfx.botAttack();
        const ar = this.attackAABB(dir);
        if (rectIntersect(ar, target.bbox())){
          target.takeDamage(12, {x: dir * 220, y: -160});
        }
        for (let i=0;i<6;i++) particles.spawn(this.x + this.w/2 + dir*12, this.y + this.h/2 + (Math.random()*20-10), {vx: (Math.random()*120-60)+dir*80, vy: Math.random()*-160, life:0.35, size:4, color:'rgba(255,200,160,0.9)'});
      }
    } else {
      this.vx = 0;
    }

    this.vy += GRAVITY * dt;
    this.x += this.vx * dt;
    this.y += this.vy * dt;

    for (let p of platforms){
      if (aabbIntersect(this.bbox(), p.rect())){
        const overlapX = Math.min(this.x+this.w, p.x+p.w) - Math.max(this.x, p.x);
        const overlapY = Math.min(this.y+this.h, p.y+p.h) - Math.max(this.y, p.y);
        if (overlapY < overlapX){
          if (this.y < p.y){
            this.y = p.y - this.h;
            this.vy = 0;
          } else {
            this.y = p.y + p.h;
            this.vy = 0;
          }
        } else {
          if (this.x < p.x) this.x = p.x - this.w;
          else this.x = p.x + p.w;
          this.vx = 0;
        }
      }
    }

    this.attackTimer = Math.max(0, this.attackTimer - dt);
    this.attackCdTimer = Math.max(0, this.attackCdTimer - dt);
    if (this.hitFlash > 0) this.hitFlash = Math.max(0, this.hitFlash - dt);
  }

  attackAABB(dir){
    const aw = this.attackRange.w, ah = this.attackRange.h;
    const ax = (dir > 0) ? this.x + this.w + this.attackRange.ox : this.x - aw - this.attackRange.ox;
    const ay = this.y + this.attackRange.oy;
    return {x: ax, y: ay, w: aw, h: ah};
  }

  takeDamage(amount, knockback){
    if (this.hp <= 0) return;
    this.hp = Math.max(0, this.hp - amount);
    this.vx = knockback.x; this.vy = knockback.y;
    this.hitFlash = 0.12;
    particles.spawn(this.x + this.w/2, this.y + this.h/2, {life:0.6, size:10, color:'rgba(255,120,120,0.95)'});
    sfx.hit();
  }

  draw(ctx, cam){
    const x = this.x - cam.x, y = this.y - cam.y;
    ctx.save();
    ctx.globalAlpha = 0.5;
    ctx.fillStyle = 'rgba(0,0,0,0.25)';
    roundRect(ctx, x+6, FLOOR_Y - cam.y + 4, this.w*1.1, 8, 6);
    ctx.fill();
    ctx.restore();

    ctx.fillStyle = (this.hitFlash>0) ? '#fff' : '#ffd06b';
    roundRect(ctx, x, y, this.w, this.h, 8);
    ctx.fill();

    const hpW = this.w;
    ctx.fillStyle = '#333';
    rect(ctx, x, y-10, hpW, 6); ctx.fill();
    ctx.fillStyle = '#ff6b6b';
    rect(ctx, x, y-10, hpW * (this.hp / this.maxHp), 6); ctx.fill();
  }
}

/* -----------------------------
   Helpers & collision utilities
   ----------------------------- */
function rect(ctx,x,y,w,h){ ctx.beginPath(); ctx.rect(x,y,w,h); }
function roundRect(ctx,x,y,w,h,r){ ctx.beginPath(); ctx.moveTo(x+r,y); ctx.arcTo(x+w,y,x+w,y+h,r); ctx.arcTo(x+w,y+h,x,y+h,r); ctx.arcTo(x,y+h,x,y,r); ctx.arcTo(x,y,x+w,y,r); ctx.closePath();}
function aabbIntersect(a,b){ return !(a.x + a.w <= b.x || a.x >= b.x + b.w || a.y + a.h <= b.y || a.y >= b.y + b.h); }
function rectIntersect(a,b){ return !(a.x + a.w < b.x || a.x > b.x + b.w || a.y + a.h < b.y || a.y > b.y + b.h); }

/* Separate overlapping entities (simple push apart) */
function separateEntities(entities){
  for (let i=0;i<entities.length;i++){
    for (let j=i+1;j<entities.length;j++){
      const A = entities[i], B = entities[j];
      const a = A.bbox ? A.bbox() : {x:A.x,y:A.y,w:A.w,h:A.h};
      const b = B.bbox ? B.bbox() : {x:B.x,y:B.y,w:B.w,h:B.h};
      if (aabbIntersect(a,b)){
        const overlapX = Math.min(a.x+a.w, b.x+b.w) - Math.max(a.x, b.x);
        const overlapY = Math.min(a.y+a.h, b.y+b.h) - Math.max(a.y, b.y);
        if (overlapX < overlapY){
          const push = overlapX / 2 + 0.5;
          if (a.x < b.x){
            if (A.x !== undefined) A.x -= push; if (B.x !== undefined) B.x += push;
          } else {
            if (A.x !== undefined) A.x += push; if (B.x !== undefined) B.x -= push;
          }
          if (A.vx !== undefined) A.vx = 0;
          if (B.vx !== undefined) B.vx = 0;
        } else {
          const push = overlapY / 2 + 0.5;
          if (a.y < b.y){
            if (A.y !== undefined) A.y -= push; if (B.y !== undefined) B.y += push;
          } else {
            if (A.y !== undefined) A.y += push; if (B.y !== undefined) B.y -= push;
          }
          if (A.vy !== undefined) A.vy = 0;
          if (B.vy !== undefined) B.vy = 0;
        }
      }
    }
  }
}

/* -----------------------------
   Game Manager / Waves / Bootstrap
   ----------------------------- */
const modeBtn = document.getElementById('modeBtn');
const pauseBtn = document.getElementById('pauseBtn');
const resetBtn = document.getElementById('resetBtn');
const p1bar = document.getElementById('p1bar');
const p2bar = document.getElementById('p2bar');
const p1hp = document.getElementById('p1hp');
const p2hp = document.getElementById('p2hp');

let mode = 'Coop';
modeBtn.addEventListener('click', ()=>{ mode = (mode==='Coop')?'PvP':'Coop'; modeBtn.textContent = `Mode: ${mode}`; });
let paused = false;
pauseBtn.addEventListener('click', ()=>{ paused = !paused; pauseBtn.textContent = paused ? 'Resume' : 'Pause'; });
resetBtn.addEventListener('click', ()=>{ reset(); });

const inputMap1 = { left:'KeyA', right:'KeyD', up:'KeyW', down:'KeyS', jump:'KeyW', attack:'KeyQ' }; // P1: W jump, A/D move, Q attack
const inputMap2 = { left:'ArrowLeft', right:'ArrowRight', up:'ArrowUp', down:'ArrowDown', jump:'ArrowUp', attack:'Numpad0' }; // P2: Up jump, Left/Right move, Numpad0 attack

let platforms = [];
let p1, p2;
let bots = [];

let waveNumber = 0;
let waveActive = false;
let waveCooldown = 0;
const WAVE_PAUSE = 5.0;
const INITIAL_WAVE_SIZE = 3;

function spawnWave(count){
  bots = [];
  for (let i=0;i<count;i++){
    const sx = 800 + i*60 - count*30 + (Math.random()*60-30);
    const sy = FLOOR_Y - 80 - Math.random()*140;
    const b = new Bot(sx, sy);
    bots.push(b);
  }
  waveActive = true;
  waveCooldown = 0;
  waveNumber++;
}

function updateWaves(dt){
  if (!waveActive && waveCooldown <= 0){
    const size = INITIAL_WAVE_SIZE + Math.floor((waveNumber)/2);
    spawnWave(size);
  } else if (!waveActive && waveCooldown > 0){
    waveCooldown -= dt;
  } else if (waveActive && bots.length === 0){
    waveActive = false;
    waveCooldown = WAVE_PAUSE;
  }
}

/* build world */
function build(){
  platforms = [];
  platforms.push(new Platform(-400, FLOOR_Y+80, 2400, 200, '#634a33'));
  platforms.push(new Platform(80, FLOOR_Y-140, 260, 20, '#2b7a3b'));
  platforms.push(new Platform(520, FLOOR_Y-220, 240, 20, '#2b7a3b'));
  platforms.push(new Platform(1000, FLOOR_Y-120, 300, 20, '#2b7a3b'));
  platforms.push(new Platform(320, FLOOR_Y-320, 160, 16, '#315'));

  p1 = new Player({x:120, y:200, color:'#ff7f7f', name:'Player1'});
  p2 = new Player({x:420, y:200, color:'#6be3ff', name:'Player2'});
  camera.x = 0; camera.y = 0; camera.zoom = 1;
  particles.p = [];

  waveNumber = 0;
  waveActive = false;
  waveCooldown = 0.5;
  bots = [];
  updateHUD();
}

/* reset */
function reset(){ build(); }

/* HUD update */
function updateHUD(){
  p1bar.style.width = Math.max(0, Math.round(p1.hp)) + '%';
  p2bar.style.width = Math.max(0, Math.round(p2.hp)) + '%';
  p1hp.textContent = Math.round(p1.hp);
  p2hp.textContent = Math.round(p2.hp);
}

/* -----------------------------
   Main loop
   ----------------------------- */
let last = performance.now();
function tick(t){
  const now = t;
  let dt = (now - last)/1000;
  last = now;
  dt = Math.min(1/30, dt);
  if (!paused){
    updateWaves(dt);

    p1.update(dt, inputMap1, p2, bots, mode);
    p2.update(dt, inputMap2, p1, bots, mode);

    const playersList = [p1, p2];
    for (let b of bots) b.update(dt, playersList, platforms);

    p1.resolvePlatforms(platforms);
    p2.resolvePlatforms(platforms);

    // separation so bots can't pass through players or each other
    const entities = [p1, p2].concat(bots);
    separateEntities(entities);

    for (let i=bots.length-1;i>=0;i--){
      if (bots[i].hp <= 0){
        for (let k=0;k<12;k++) particles.spawn(bots[i].x + bots[i].w/2 + Math.random()*20-10, bots[i].y + bots[i].h/2 + Math.random()*20-10, {life:0.8, size:6, color:'rgba(255,120,120,0.95)'});
        bots.splice(i,1);
      }
    }

    // camera: if both players alive -> follow between; if one alive -> focus that player
    const p1Alive = p1.hp > 0;
    const p2Alive = p2.hp > 0;
    if (p1Alive && p2Alive){
      camera.followBetween({x:p1.x + p1.w/2, y:p1.y + p1.h/2}, {x:p2.x + p2.w/2, y:p2.y + p2.h/2}, dt);
    } else if (p1Alive){
      camera.followSingle(p1, dt);
    } else if (p2Alive){
      camera.followSingle(p2, dt);
    }

    particles.update(dt);

    // continue running even if one player is dead; stop only when both dead
    if (!p1Alive && !p2Alive){
      paused = true;
      pauseBtn.textContent = 'Resume';
    }
  }

  // render
  ctx.save();
  ctx.fillStyle = '#0c1730';
  ctx.fillRect(0,0,CANVAS_W,CANVAS_H);
  drawParallax();
  ctx.translate(-camera.x, -camera.y);

  for (let pl of platforms) pl.draw(ctx, camera);
  particles.draw(ctx, camera);
  for (let b of bots) b.draw(ctx, camera);
  p1.draw(ctx, camera);
  p2.draw(ctx, camera);

  if (DEBUG){
    ctx.fillStyle = '#fff';
    ctx.fillText(`bots:${bots.length} wave:${waveNumber} cooldown:${waveCooldown.toFixed(2)}`, camera.x + 20, camera.y + 20);
  }

  ctx.setTransform(DPR,0,0,DPR,0,0);
  if (p1.hp<=0 || p2.hp<=0){
    ctx.fillStyle = 'rgba(0,0,0,0.45)';
    ctx.fillRect(0, CANVAS_H/2 - 50, CANVAS_W, 120);
    ctx.fillStyle = '#fff';
    ctx.font = '36px Inter, Arial';
    ctx.textAlign = 'center';
    let winner = 'Draw';
    if (p1.hp<=0 && p2.hp>0) winner = 'Player 2 survives';
    else if (p2.hp<=0 && p1.hp>0) winner = 'Player 1 survives';
    else if (p1.hp<=0 && p2.hp<=0) winner = 'Both players died';
    ctx.fillText(winner, CANVAS_W/2, CANVAS_H/2 + 6);
    ctx.font = '14px Inter, Arial';
    ctx.fillText('Press Reset to play again', CANVAS_W/2, CANVAS_H/2 + 40);
    ctx.textAlign = 'start';
  }
  ctx.setTransform(DPR,0,0,DPR,0,0);

  drawVignette();
  updateHUD();
  requestAnimationFrame(tick);
}

/* Parallax & Vignette */
function drawParallax(){
  const cx = camera.x * 0.1;
  for (let i=0;i<6;i++){
    const sy = 40 + i*60 + Math.sin((performance.now()/1200) + i) * 12;
    ctx.globalAlpha = 0.12;
    ctx.fillStyle = (i%2===0)?'#cde7ff':'#eaf6ff';
    const ox = (i*230 - cx*0.2) % (CANVAS_W + 400) - 200;
    ctx.beginPath();
    ctx.ellipse(ox, sy, 220, 44, 0, 0, Math.PI*2);
    ctx.fill();
  }
  ctx.globalAlpha = 1;
}
function drawVignette(){
  const g = ctx.createLinearGradient(0,0,0,CANVAS_H);
  g.addColorStop(0,'rgba(0,0,0,0.06)');
  g.addColorStop(0.6,'rgba(0,0,0,0.00)');
  g.addColorStop(1,'rgba(0,0,0,0.16)');
  ctx.fillStyle = g;
  ctx.fillRect(0,0,CANVAS_W,CANVAS_H);
}

/* -----------------------------
   Startup & event hooks
   ----------------------------- */
build();
last = performance.now();
requestAnimationFrame(tick);

window.addEventListener('keydown', e=>{
  if (e.code === 'KeyM'){ mode = (mode==='Coop')?'PvP':'Coop'; modeBtn.textContent = `Mode: ${mode}`; }
  if (e.code === 'KeyR'){ reset(); paused = false; pauseBtn.textContent = 'Pause'; }
});
window.addEventListener('resize', ()=>{ /* nothing needed */ });

if (DEBUG){
  window.getState = ()=>({p1,p2,bots,platforms,waveNumber,waveCooldown});
}

</script>
</body>
</html>
