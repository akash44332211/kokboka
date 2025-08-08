<!doctype html>
<html lang="bn">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>গরু গেম 🐄</title>
<style>
  body{margin:0;background:linear-gradient(#9be7ff,#7fd0a6);font-family:sans-serif;display:flex;flex-direction:column;align-items:center}
  #game {border:4px solid #2b7a5a; background: linear-gradient(#bdf7ff,#eafde8); margin-top:12px;}
  #hud{margin-top:8px;color:#063; font-weight:700}
  button{margin-left:8px;padding:6px 10px;border-radius:6px;border:2px solid #2b7a5a;background:white;cursor:pointer}
  .small{font-size:13px;color:#055}
</style>
</head>
<body>
<h1>গরু গেম 🐄</h1>
<canvas id="game" width="800" height="420"></canvas>
<div id="hud">
  Score: <span id="score">0</span>
  <button id="restart">রিস্টার্ট</button>
  <span class="small"> (Space → jump, ← → move)</span>
</div>

<script>
// --- Game settings
const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');
const W = canvas.width, H = canvas.height;

let scoreEl = document.getElementById('score');
let restartBtn = document.getElementById('restart');

let keys = {};
window.addEventListener('keydown', e => { keys[e.code]=true; if(e.code==='Space') e.preventDefault(); });
window.addEventListener('keyup', e => keys[e.code]=false);

// --- Game objects
function rand(min,max){ return Math.random()*(max-min)+min; }

class Cow {
  constructor(){
    this.w = 90; this.h = 60;
    this.x = 80; this.y = H - 80 - this.h;
    this.vy = 0;
    this.onGround = true;
    this.speed = 4;
  }
  update(){
    // left/right
    if(keys['ArrowLeft'] || keys['KeyA']) this.x -= this.speed;
    if(keys['ArrowRight'] || keys['KeyD']) this.x += this.speed;
    // bounds
    this.x = Math.max(10, Math.min(W - this.w - 10, this.x));
    // jump
    if((keys['Space'] || keys['ArrowUp'] || keys['KeyW']) && this.onGround){
      this.vy = -12;
      this.onGround = false;
    }
    // gravity
    this.vy += 0.6;
    this.y += this.vy;
    if(this.y + this.h >= H - 80){
      this.y = H - 80 - this.h;
      this.vy = 0;
      this.onGround = true;
    }
  }
  draw(){
    // simple cartoon cow using shapes
    ctx.save();
    // shadow
    ctx.fillStyle = 'rgba(0,0,0,0.12)';
    ctx.beginPath();
    ctx.ellipse(this.x+this.w/2, H-80+18, 46, 12, 0, 0, Math.PI*2);
    ctx.fill();

    // body
    ctx.fillStyle = '#fff';
    ctx.fillRect(this.x+10, this.y+20, 70, 36);
    // spots
    ctx.fillStyle = '#222';
    ctx.beginPath(); ctx.ellipse(this.x+30, this.y+34, 12,8,0,0,Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.ellipse(this.x+54, this.y+44, 14,9,0,0,Math.PI*2); ctx.fill();

    // head
    ctx.fillStyle = '#fff';
    ctx.fillRect(this.x-6, this.y+6, 40, 30);
    ctx.fillStyle='#222';
    ctx.fillRect(this.x+4, this.y+14, 8,8); // eye patch
    ctx.fillStyle='#000';
    ctx.beginPath(); ctx.arc(this.x+12, this.y+20, 3,0,Math.PI*2); ctx.fill(); // eye

    // horns
    ctx.strokeStyle='#e6c07a'; ctx.lineWidth=4;
    ctx.beginPath(); ctx.moveTo(this.x+2, this.y+6); ctx.lineTo(this.x-6, this.y-2); ctx.stroke();
    ctx.beginPath(); ctx.moveTo(this.x+26, this.y+6); ctx.lineTo(this.x+36, this.y-4); ctx.stroke();

    // legs
    ctx.fillStyle='#333';
    ctx.fillRect(this.x+14, this.y+56, 8,14);
    ctx.fillRect(this.x+36, this.y+56, 8,14);

    // tail
    ctx.strokeStyle='#333'; ctx.lineWidth=4;
    ctx.beginPath(); ctx.moveTo(this.x+82, this.y+36); ctx.lineTo(this.x+94, this.y+42); ctx.stroke();

    ctx.restore();
  }
  getBox(){ return {x:this.x, y:this.y, w:this.w, h:this.h}; }
}

class Milk {
  constructor(){
    this.r = 12;
    this.x = W + rand(100,400);
    this.y = rand(80, H-140);
    this.vx = -3 - rand(0,2);
    this.collected = false;
  }
  update(){ this.x += this.vx; }
  draw(){
    ctx.save();
    // bottle style
    ctx.fillStyle = '#fff';
    ctx.beginPath();
    ctx.ellipse(this.x, this.y, this.r, this.r+3,0,0,Math.PI*2);
    ctx.fill();
    ctx.fillStyle='#d0f'; ctx.fillRect(this.x-6,this.y-14,12,8);
    ctx.fillStyle='#9cf'; ctx.fillRect(this.x-6,this.y-4,12,6);
    ctx.restore();
  }
  offscreen(){ return this.x < -40; }
  box(){ return {x:this.x-this.r, y:this.y-this.r, w:this.r*2, h:this.r*2}; }
}

class Obstacle {
  constructor(){
    this.w = rand(30,60);
    this.h = rand(30,70);
    this.x = W + rand(0,600);
    this.y = H - 80 - this.h;
    this.vx = -4 - rand(0,2);
  }
  update(){ this.x += this.vx; }
  draw(){
    ctx.save();
    ctx.fillStyle = '#7b3f00';
    ctx.fillRect(this.x, this.y, this.w, this.h);
    // grass on top
    ctx.fillStyle = '#2c9a46';
    ctx.fillRect(this.x-4, this.y-6, this.w+8, 6);
    ctx.restore();
  }
  offscreen(){ return this.x + this.w < -20; }
  box(){ return {x:this.x, y:this.y, w:this.w, h:this.h}; }
}

// --- Game state
let cow = new Cow();
let milks = [];
let obstacles = [];
let spawnTimerMilk = 0;
let spawnTimerObs = 0;
let score = 0;
let gameOver = false;

// ground draw
function drawGround(){
  ctx.fillStyle = '#6bbf5f';
  ctx.fillRect(0, H-80, W, 80);
  // fence lines
  ctx.strokeStyle = '#3f7a45'; ctx.lineWidth=2;
  for(let i=0;i<W;i+=30){
    ctx.beginPath(); ctx.moveTo(i, H-80); ctx.lineTo(i, H-70); ctx.stroke();
  }
}

// collision
function intersects(a,b){
  return !(a.x+a.w < b.x || a.x > b.x+b.w || a.y+a.h < b.y || a.y > b.y+b.h);
}

// main loop
function update(){
  if(gameOver) return;
  cow.update();

  // spawn milk
  spawnTimerMilk -= 1;
  if(spawnTimerMilk <= 0){
    milks.push(new Milk());
    spawnTimerMilk = 120 + Math.floor(rand(0,120));
  }
  // spawn obstacles
  spawnTimerObs -= 1;
  if(spawnTimerObs <= 0){
    obstacles.push(new Obstacle());
    spawnTimerObs = 160 + Math.floor(rand(0,160));
  }

  milks.forEach(m => m.update());
  obstacles.forEach(o => o.update());

  // collision milk
  milks.forEach(m => {
    if(!m.collected && intersects(cow.getBox(), m.box())){
      m.collected = true;
      score += 10;
      scoreEl.textContent = score;
    }
  });
  // remove offscreen or collected
  milks = milks.filter(m => !m.offscreen() && !m.collected);
  obstacles = obstacles.filter(o => !o.offscreen());

  // obstacle collision => game over
  for(let o of obstacles){
    if(intersects(cow.getBox(), o.box())){
      gameOver = true;
      break;
    }
  }
}

// draw
function draw(){
  // sky background already via body; fill canvas clear
  ctx.clearRect(0,0,W,H);

  // clouds (simple)
  ctx.fillStyle = '#ffffff';
  ctx.globalAlpha = 0.9;
  ctx.beginPath(); ctx.ellipse(140,70,58,26,0,0,Math.PI*2); ctx.ellipse(180,76,38,20,0,0,Math.PI*2); ctx.fill();
  ctx.beginPath(); ctx.ellipse(480,50,70,28,0,0,Math.PI*2); ctx.ellipse(530,55,40,22,0,0,Math.PI*2); ctx.fill();
  ctx.globalAlpha = 1;

  // barn or sun
  ctx.fillStyle = '#ffd24a';
  ctx.beginPath(); ctx.arc(W-70, 70, 40, 0, Math.PI*2); ctx.fill();

  // draw ground
  drawGround();

  // draw milks and obstacles behind cow
  milks.forEach(m => m.draw());
  obstacles.forEach(o => o.draw());

  // cow
  cow.draw();

  if(gameOver){
    ctx.fillStyle = 'rgba(0,0,0,0.45)';
    ctx.fillRect(0,0,W,H);
    ctx.fillStyle = '#fff';
    ctx.font = '34px sans-serif';
    ctx.textAlign = 'center';
    ctx.fillText('গেম আর নেই — গরু টা ধাক্কা খেয়েছে!', W/2, H/2 - 10);
    ctx.font = '22px sans-serif';
    ctx.fillText(`স্কোর: ${score}`, W/2, H/2 + 28);
    ctx.fillText('রিস্টার্ট করতে নিচের বোতন চাপুন', W/2, H/2 + 56);
  }
}

// loop runner
function loop(){
  update();
  draw();
  requestAnimationFrame(loop);
}

restartBtn.addEventListener('click', ()=>{
  // reset state
  cow = new Cow();
  milks = [];
  obstacles = [];
  score = 0;
  scoreEl.textContent = score;
  spawnTimerMilk = 30;
  spawnTimerObs = 90;
  gameOver = false;
});

// start
spawnTimerMilk = 30;
spawnTimerObs = 90;
loop();

</script>
</body>
</html>
