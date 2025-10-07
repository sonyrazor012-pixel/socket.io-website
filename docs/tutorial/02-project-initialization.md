<!doctype html>
<html>
<head>
  <meta charset="utf-8"/>
  <title>Survivish — single-player top-down shooter</title>
  <style>
    html,body { height:100%; margin:0; background:#0b0f13; color:#ddd; font-family:Inter,Arial,sans-serif; }
    #gameWrap { display:flex; gap:12px; padding:12px; box-sizing:border-box; }
    canvas { background: linear-gradient(#10303a,#0b1b20); border-radius:8px; box-shadow:0 6px 18px rgba(0,0,0,0.6); }
    #ui { width:260px; }
    .panel { background:rgba(255,255,255,0.03); padding:12px; border-radius:8px; margin-bottom:12px; }
    h1{ margin:0 0 8px 0; font-size:18px; color:#fff;}
    button { padding:8px 10px; border-radius:6px; border:none; background:#2e9; color:#023; font-weight:700; cursor:pointer; }
    .small { font-size:13px; opacity:.9; }
    .muted { opacity:.7; font-size:13px; }
    #controls label { display:block; margin:6px 0; }
    .stat { display:flex; justify-content:space-between; margin:6px 0; }
    #footer { font-size:12px; opacity:.7; margin-top:8px; }
  </style>
</head>
<body>
  <div id="gameWrap">
    <canvas id="game" width="960" height="640"></canvas>
    <div id="ui">
      <div class="panel">
        <h1>Survivish — single-player</h1>
        <div class="small">WASD to move • Mouse to aim • Left click to shoot • R to reload</div>
        <div class="stat"><span>Score</span><span id="score">0</span></div>
        <div class="stat"><span>HP</span><span id="hp">100</span></div>
        <div class="stat"><span>Ammo</span><span id="ammo">—</span></div>
        <div class="stat"><span>Enemies</span><span id="enemies">0</span></div>
        <div style="margin-top:10px;">
          <button id="startBtn">Start / Restart</button>
          <button id="toggleAuto" style="margin-left:6px">Auto fire: OFF</button>
        </div>
      </div>

      <div class="panel" id="controls">
        <div><strong>Game options</strong></div>
        <label>Enemy count: <input id="enemyCount" type="range" min="1" max="12" value="4" /></label>
        <label>Enemy speed: <input id="enemySpeed" type="range" min="0.3" max="2.0" step="0.1" value="0.9" /></label>
        <label>Spawn rate (pickups): <input id="pickupRate" type="range" min="3" max="14" value="8" /></label>
      </div>

      <div class="panel">
        <div><strong>Tips</strong></div>
        <div class="muted">
          - Keep moving: enemies home in on you.<br>
          - Pick health packs when low; conserve ammo.<br>
          - Upgrade: modify code to add new weapons, map, or multiplayer.
        </div>
      </div>

      <div class="panel" id="footer">
        Build and modify the code to learn game mechanics. Single-file HTML + JS.
      </div>
    </div>
  </div>

<script>
/*
  Survivish — single-file demo
  Author: ChatGPT
  Features: movement, shooting, bullets, enemies, pickups, mini-map
*/

// ---------- Globals ----------
const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');
const W = canvas.width, H = canvas.height;
let lastTime=0, dt=0;
let game=null;

// UI elements
const scoreEl = document.getElementById('score');
const hpEl = document.getElementById('hp');
const ammoEl = document.getElementById('ammo');
const enemiesEl = document.getElementById('enemies');
const startBtn = document.getElementById('startBtn');
const toggleAutoBtn = document.getElementById('toggleAuto');

const enemyCountRange = document.getElementById('enemyCount');
const enemySpeedRange = document.getElementById('enemySpeed');
const pickupRateRange = document.getElementById('pickupRate');

// ---------- Utility ----------
function rand(min,max){ return Math.random()*(max-min)+min; }
function clamp(v,a,b){ return Math.max(a, Math.min(b, v)); }
function dist(a,b){ const dx=a.x-b.x, dy=a.y-b.y; return Math.hypot(dx,dy); }
function angleTo(a,b){ return Math.atan2(b.y-a.y, b.x-a.x); }

// ---------- Game classes ----------
class Player {
  constructor(x,y){
    this.x=x; this.y=y;
    this.radius=12;
    this.speed=180; // px/s
    this.hp=100;
    this.maxHp=100;
    this.rotation=0; // aim angle
    this.weapon = { name:'Pistol', mag:12, maxMag:12, fireRate:6, reloadTime:1.2, bulletSpeed:520, bulletDamage:18 };
    this.reloadTimer=0;
    this.fireCooldown=0;
    this.score=0;
    this.autoFire=false;
  }
  update(dt, input){
    // Movement
    let vx=0, vy=0;
    if(input.keys['w']) vy-=1;
    if(input.keys['s']) vy+=1;
    if(input.keys['a']) vx-=1;
    if(input.keys['d']) vx+=1;
    const len = Math.hypot(vx,vy) || 1;
    this.x += (vx/len)*this.speed*dt;
    this.y += (vy/len)*this.speed*dt;
    // keep inside world bounds
    this.x = clamp(this.x, 20, W-20);
    this.y = clamp(this.y, 20, H-20);

    // aiming
    this.rotation = Math.atan2(input.mouse.y - this.y, input.mouse.x - this.x);

    // firing
    if(this.reloadTimer > 0){ this.reloadTimer -= dt; if(this.reloadTimer <= 0){ this.weapon.mag = this.weapon.maxMag; } }
    if(this.fireCooldown > 0) this.fireCooldown -= dt;

    if((input.mouse.down || this.autoFire) && this.fireCooldown <= 0 && this.reloadTimer <= 0){
      if(this.weapon.mag > 0){
        this.shoot();
        this.weapon.mag--;
        this.fireCooldown = 1/this.weapon.fireRate;
      } else {
        // empty -> reload automatically
        this.reload();
      }
    }
  }
  shoot(){
    // spawn bullet
    const spread = (Math.random()-0.5)*0.06;
    const ang = this.rotation + spread;
    const speed = this.weapon.bulletSpeed;
    game.spawnBullet(this.x + Math.cos(ang)*this.radius, this.y + Math.sin(ang)*this.radius, ang, speed, this.weapon.bulletDamage, 'player');
  }
  reload(){
    if(this.reloadTimer <= 0){
      this.reloadTimer = this.weapon.reloadTime;
    }
  }
  draw(ctx){
    // body
    ctx.save();
    ctx.translate(this.x, this.y);
    // shadow
    ctx.fillStyle = 'rgba(0,0,0,0.25)';
    ctx.beginPath(); ctx.ellipse(0,6,16,8,0,0,Math.PI*2); ctx.fill();
    // player circle
    ctx.rotate(this.rotation);
    ctx.fillStyle = '#4fe0ff';
    ctx.beginPath(); ctx.arc(0,0,this.radius,0,Math.PI*2); ctx.fill();
    // gun
    ctx.fillStyle = '#222';
    ctx.fillRect(6,-4,18,8);
    ctx.restore();

    // health bar
    const w=50, h=6;
    const pct=this.hp/this.maxHp;
    ctx.fillStyle='rgba(0,0,0,0.5)';
    ctx.fillRect(this.x-w/2, this.y - 28, w, h);
    ctx.fillStyle = pct>0.5 ? '#2ecc71' : pct>0.25 ? '#f1c40f' : '#e74c3c';
    ctx.fillRect(this.x-w/2, this.y - 28, w*pct, h);
  }
}

class Enemy {
  constructor(x,y, speedMult=1){
    this.x=x; this.y=y;
    this.radius=11;
    this.hp=40;
    this.maxHp=40;
    this.rotation=0;
    this.speed = rand(40,70) * speedMult;
    this.color = '#ff7f7f';
    this.fireCooldown = rand(0.8,1.8);
    this.alive=true;
  }
  update(dt){
    // simple AI: chase player if in range, otherwise wander
    const p = game.player;
    const d = dist(this, p);
    const ang = angleTo(this, p);
    this.rotation = ang;

    if(d > 40){
      // move towards player
      this.x += Math.cos(ang)*this.speed*dt;
      this.y += Math.sin(ang)*this.speed*dt;
    } else {
      // small jitter to avoid stacking
      this.x += Math.cos(ang+Math.PI/2)*10*dt;
      this.y += Math.sin(ang+Math.PI/2)*10*dt;
    }

    // keep in bounds
    this.x = clamp(this.x, 10, W-10);
    this.y = clamp(this.y, 10, H-10);

    // shooting occasionally
    this.fireCooldown -= dt;
    if(this.fireCooldown <= 0){
      this.fireCooldown = rand(1.2,2.2);
      // fire towards player
      const spd = 320;
      game.spawnBullet(this.x + Math.cos(ang)*this.radius, this.y + Math.sin(ang)*this.radius, ang, spd, 12, 'enemy');
    }
  }
  draw(ctx){
    // shadow
    ctx.fillStyle = 'rgba(0,0,0,0.25)';
    ctx.beginPath(); ctx.ellipse(this.x, this.y+6, this.radius+6, this.radius/1.6, 0,0,Math.PI*2); ctx.fill();

    ctx.save();
    ctx.translate(this.x, this.y);
    ctx.rotate(this.rotation);
    ctx.fillStyle = this.color;
    ctx.beginPath(); ctx.arc(0,0,this.radius,0,Math.PI*2); ctx.fill();
    ctx.fillStyle = '#222';
    ctx.fillRect(-6,-4,12,8);
    ctx.restore();

    // health
    const w=36,h=5;
    const pct=this.hp/this.maxHp;
    ctx.fillStyle='rgba(0,0,0,0.35)';
    ctx.fillRect(this.x-w/2, this.y - this.radius - 16, w, h);
    ctx.fillStyle = pct>0.5 ? '#2ecc71' : pct>0.25 ? '#f1c40f' : '#e74c3c';
    ctx.fillRect(this.x-w/2, this.y - this.radius - 16, w*pct, h);
  }
}

class Bullet {
  constructor(x,y,ang,speed,damage, owner){
    this.x=x; this.y=y;
    this.vx = Math.cos(ang)*speed;
    this.vy = Math.sin(ang)*speed;
    this.damage = damage;
    this.owner = owner; // 'player' or 'enemy'
    this.age = 0;
    this.maxAge = 3.0;
    this.radius = 3;
  }
  update(dt){
    this.x += this.vx*dt;
    this.y += this.vy*dt;
    this.age += dt;
    // remove outside bounds or expired
    if(this.age > this.maxAge) this.dead=true;
    if(this.x < -50 || this.x > W+50 || this.y < -50 || this.y > H+50) this.dead=true;
  }
  draw(ctx){
    ctx.fillStyle = this.owner === 'player' ? '#aee' : '#ffb3b3';
    ctx.beginPath(); ctx.arc(this.x,this.y,this.radius,0,Math.PI*2); ctx.fill();
  }
}

class Pickup {
  constructor(x,y, type){
    this.x=x; this.y=y;
    this.type=type; // 'health' or 'ammo'
    this.radius = 10;
  }
  draw(ctx){
    ctx.save();
    ctx.translate(this.x,this.y);
    ctx.fillStyle = this.type === 'health' ? '#e74c3c' : '#f39c12';
    ctx.beginPath(); ctx.arc(0,0,this.radius,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='#111'; ctx.font='10px sans-serif'; ctx.textAlign='center'; ctx.textBaseline='middle';
    ctx.fillText(this.type==='health'?'+HP':'AMMO',0,0);
    ctx.restore();
  }
}

// ---------- Game management ----------
class Game {
  constructor(){
    this.player = new Player(W/2, H/2);
    this.enemies = [];
    this.bullets = [];
    this.pickups = [];
    this.spawnTimer = 0;
    this.pickupTimer = 0;
    this.running = false;
    this.input = { keys:{}, mouse:{x:W/2,y:H/2,down:false} };
    this.enemyBaseSpeedMult = Number(enemySpeedRange.value);
    this.pickupRate = Number(pickupRateRange.value);
  }

  start(){
    this.player = new Player(W/2, H/2);
    this.enemies = [];
    this.bullets = [];
    this.pickups = [];
    this.spawnTimer = 0;
    this.pickupTimer = 0;
    this.running = true;
    this.player.score = 0;
    this.spawnInitialEnemies(Number(enemyCountRange.value));
    updateUI(this);
  }

  spawnInitialEnemies(n){
    for(let i=0;i<n;i++){
      this.spawnEnemy();
    }
  }
  spawnEnemy(){
    // spawn on edges
    const edge = Math.floor(rand(0,4));
    let x,y;
    if(edge===0){ x = rand(-30,0); y = rand(0,H); }
    else if(edge===1){ x = rand(W, W+30); y = rand(0,H); }
    else if(edge===2){ y = rand(-30,0); x = rand(0,W); }
    else { y = rand(H, H+30); x = rand(0,W); }
    const e = new Enemy(x,y, this.enemyBaseSpeedMult);
    this.enemies.push(e);
  }

  spawnBullet(x,y,ang,speed,damage, owner){
    const b = new Bullet(x,y,ang,speed,damage, owner);
    this.bullets.push(b);
  }

  spawnPickup(){
    const type = Math.random() < 0.6 ? 'ammo' : 'health';
    const p = new Pickup(rand(40,W-40), rand(40,H-40), type);
    this.pickups.push(p);
  }

  update(dt){
    if(!this.running) return;
    this.player.update(dt, this.input);

    // update enemies
    for(const e of this.enemies) e.update(dt);

    // bullets
    for(const b of this.bullets) b.update(dt);

    // collision: bullets -> entities
    for(const b of this.bullets){
      if(b.dead) continue;
      if(b.owner === 'player'){
        for(const e of this.enemies){
          if(e.hp > 0 && dist(b, e) < e.radius + b.radius){
            e.hp -= b.damage;
            b.dead = true;
            if(e.hp <= 0) { e.alive=false; this.player.score += 10; }
            break;
          }
        }
      } else { // enemy bullet
        const p = this.player;
        if(dist(b,p) < p.radius + b.radius){
          p.hp -= b.damage;
          b.dead = true;
          if(p.hp <= 0){ this.onPlayerKilled(); }
        }
      }
    }

    // bullets vs bullets (optional) — ignore for speed

    // remove dead enemies and bullets
    this.bullets = this.bullets.filter(b => !b.dead);
    const beforeEnemies = this.enemies.length;
    this.enemies = this.enemies.filter(e => e.alive && e.hp > 0);
    if(this.enemies.length < Number(enemyCountRange.value)){
      // spawn slowly to maintain count
      this.spawnTimer += dt;
      if(this.spawnTimer > 1.0){
        this.spawnTimer = 0;
        this.spawnEnemy();
      }
    }

    // pickups: timer
    this.pickupTimer += dt;
    if(this.pickupTimer > Number(pickupRateRange.value)){
      this.pickupTimer = 0;
      this.spawnPickup();
    }

    // pickups collection
    for(let i=this.pickups.length-1;i>=0;i--){
      const p = this.pickups[i];
      if(dist(p,this.player) < p.radius + this.player.radius + 4){
        if(p.type === 'health'){ this.player.hp = clamp(this.player.hp + 30, 0, this.player.maxHp); }
        else { this.player.weapon.mag = clamp(this.player.weapon.mag + 8, 0, this.player.weapon.maxMag*3 ); }
        this.pickups.splice(i,1);
      }
    }

    // update UI
    updateUI(this);
  }

  onPlayerKilled(){
    this.running = false;
    alert('You died — score: ' + this.player.score + '. Click Start to try again.');
  }

  draw(ctx){
    // clear
    ctx.clearRect(0,0,W,H);

    // grid/ground for depth cues
    ctx.save();
    ctx.translate(0,0);
    ctx.strokeStyle = 'rgba(255,255,255,0.02)';
    ctx.lineWidth = 1;
    for(let x=0;x<W; x+=48){ ctx.beginPath(); ctx.moveTo(x,0); ctx.lineTo(x,H); ctx.stroke(); }
    for(let y=0;y<H; y+=48){ ctx.beginPath(); ctx.moveTo(0,y); ctx.lineTo(W,y); ctx.stroke(); }
    ctx.restore();

    // draw pickups
    for(const p of this.pickups) p.draw(ctx);
    // draw enemies behind player maybe: sort by y if you want depth
    const allEntities = [...this.enemies];
    allEntities.sort((a,b)=> a.y - b.y);
    for(const e of allEntities) e.draw(ctx);

    // bullets
    for(const b of this.bullets) b.draw(ctx);

    // player last
    this.player.draw(ctx);

    // minimap
    this.drawMinimap(ctx);
  }

  drawMinimap(ctx){
    const size = 160;
    const pad = 10;
    const x = W - size - pad, y = pad;
    ctx.save();
    ctx.globalAlpha = 0.85;
    ctx.fillStyle = 'rgba(6,10,14,0.85)';
    ctx.fillRect(x,y,size,size);
    ctx.strokeStyle='rgba(255,255,255,0.06)';
    ctx.strokeRect(x,y,size,size);
    // scale world -> minimap
    const scaleX = size / W, scaleY = size / H;
    // player
    ctx.fillStyle = '#4fe0ff';
    ctx.beginPath(); ctx.arc(x + this.player.x*scaleX, y + this.player.y*scaleY, 4,0,Math.PI*2); ctx.fill();
    // enemies
    for(const e of this.enemies){
      ctx.fillStyle = '#ff7f7f';
      ctx.beginPath(); ctx.arc(x + e.x*scaleX, y + e.y*scaleY, 3,0,Math.PI*2); ctx.fill();
    }
    // pickups
    for(const p of this.pickups){
      ctx.fillStyle = p.type==='health' ? '#e74c3c' : '#f39c12';
      ctx.beginPath(); ctx.arc(x + p.x*scaleX, y + p.y*scaleY, 2.5,0,Math.PI*2); ctx.fill();
    }
    ctx.restore();
  }
}

// ---------- Input handling ----------
const input = { keys:{}, mouse:{x:W/2,y:H/2,down:false} };
canvas.addEventListener('mousemove', (e) => {
  const rect = canvas.getBoundingClientRect();
  input.mouse.x = e.clientX - rect.left;
  input.mouse.y = e.clientY - rect.top;
});
canvas.addEventListener('mousedown', (e) => { input.mouse.down = true; });
canvas.addEventListener('mouseup', (e) => { input.mouse.down = false; });
window.addEventListener('keydown', (e) => {
  input.keys[e.key.toLowerCase()] = true;
  if(e.key.toLowerCase() === 'r' && game) game.player.reload();
  if(e.key === 'Escape' && game) game.running=false;
});
window.addEventListener('keyup', (e) => { input.keys[e.key.toLowerCase()] = false; });

// ---------- UI wiring ----------
startBtn.addEventListener('click', ()=> {
  if(!game) game = new Game();
  game.enemyBaseSpeedMult = Number(enemySpeedRange.value);
  game.pickupRate = Number(pickupRateRange.value);
  game.start();
});
toggleAutoBtn.addEventListener('click', ()=> {
  if(!game) return;
  game.player.autoFire = !game.player.autoFire;
  toggleAutoBtn.textContent = 'Auto fire: ' + (game.player.autoFire ? 'ON' : 'OFF');
});
enemyCountRange.addEventListener('input', ()=> {
  if(game) {
    // if lowering below current enemy number, trim later automatically
  }
});
enemySpeedRange.addEventListener('input', ()=> { if(game) game.enemyBaseSpeedMult = Number(enemySpeedRange.value); });
pickupRateRange.addEventListener('input', ()=> { if(game) game.pickupRate = Number(pickupRateRange.value); });

// ---------- Update UI helper ----------
function updateUI(g){
  scoreEl.textContent = Math.max(0, g.player.score);
  hpEl.textContent = Math.round(g.player.hp);
  ammoEl.textContent = `${g.player.weapon.mag} / ${g.player.weapon.maxMag}`;
  enemiesEl.textContent = g.enemies.length;
}

// ---------- Game loop ----------
function loop(ts){
  if(!lastTime) lastTime = ts;
  dt = Math.min(0.05, (ts - lastTime)/1000); // clamp large dt
  lastTime = ts;

  if(game){
    game.input = input;
    game.update(dt);
    game.draw(ctx);
  } else {
    // draw intro
    ctx.clearRect(0,0,W,H);
    ctx.fillStyle='#fff'; ctx.font='20px sans-serif'; ctx.textAlign='center';
    ctx.fillText('Click "Start / Restart" to begin', W/2, H/2);
  }

  requestAnimationFrame(loop);
}
requestAnimationFrame(loop);

// ---------- Simple collisions: bullets vs enemies handled inside Game.update ----------
// ---------- End of file ----------
</script>
</body>
</html>
