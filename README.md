<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>Space Dodger — Mobile Version</title>

<style>
:root{--bg:#071029;--panel:#0b1b2b;--accent:#7be3ff;--danger:#ff6b6b}
html,body{height:100%;margin:0;font-family:Inter,system-ui,Roboto,Arial}
body{
  background:linear-gradient(180deg,var(--bg),#021020);
  display:flex;align-items:center;justify-content:center;
  padding:10px;color:#cfeffd;overflow:hidden;
}
.wrap{
  width:100%;max-width:980px;background:rgba(255,255,255,0.03);
  border-radius:14px;box-shadow:0 8px 30px rgba(2,10,20,0.6);
  overflow:hidden;
}
header{
  display:flex;align-items:center;justify-content:space-between;
  padding:14px 18px;background:rgba(255,255,255,0.04)
}
header h1{font-size:18px;margin:0}
header .info{font-size:13px;color:#9fcfe6}
.game-row{display:flex;gap:18px}
.board{flex:1;padding:14px}
canvas{
  width:100%;height:60vh;max-height:520px;
  background:radial-gradient(ellipse at center, rgba(255,255,255,0.01), transparent 40%),
             repeating-linear-gradient(transparent, transparent 30px, rgba(255,255,255,0.003) 31px);
  border-radius:8px;display:block;touch-action:none;
}
.panel{
  width:300px;padding:14px;background:linear-gradient(180deg,var(--panel),rgba(255,255,255,0.02));
  border-left:1px solid rgba(255,255,255,0.05)
}
.stat{display:flex;align-items:center;justify-content:space-between;
  padding:8px 0;border-bottom:1px dashed rgba(255,255,255,0.05)
}
button{
  appearance:none;border:0;background:linear-gradient(90deg,var(--accent),#58c9f1);
  color:#01222c;padding:10px 12px;border-radius:8px;font-weight:700;
  cursor:pointer;width:100%;margin-bottom:10px;
}
.small{padding:8px 10px;font-size:13px}
.danger{background:linear-gradient(90deg,#ff9b9b,var(--danger));color:#2c0000}
footer{padding:10px 14px;color:#8fbfd7;font-size:13px}
.score-big{font-size:28px;font-weight:800;color:var(--accent)}

.mobile-controls{
  display:none;
  position:fixed;bottom:10px;left:0;right:0;
  z-index:50;padding:10px;
  display:flex;justify-content:space-between;
}
.mobile-btn{
  width:31%;padding:16px 0;background:#234;padding:12px;
  background:rgba(255,255,255,0.08);border-radius:12px;
  text-align:center;font-size:18px;font-weight:700;
  color:#cfeffd;user-select:none;
}

/* mobile responsive */
@media(max-width:780px){
  .game-row{flex-direction:column}
  .panel{
    width:100%;border-left:none;
    border-top:1px solid rgba(255,255,255,0.05)
  }
  canvas{height:52vh}
  .mobile-controls{display:flex}
}
</style>
</head>

<body>
<div class="wrap">
<header>
  <h1>Space Dodger — Mobile</h1>
  <div class="info">Tap arrows or screen edges to move</div>
</header>

<div class="game-row">
  <div class="board">
    <canvas id="game" width="800" height="520"></canvas>
  </div>

  <aside class="panel">
    <div class="stat"><div>Score</div><div class="score-big" id="score">0</div></div>
    <div class="stat"><div>High Score</div><div id="best">0</div></div>
    <div class="stat"><div>Level</div><div id="level">1</div></div>

    <button id="start">Start</button>
    <button id="pause" class="small">Pause</button>
    <button id="reset" class="small danger">Reset</button>
  </aside>
</div>

<footer>Mobile Edition — Runs on any phone.</footer>
</div>

<!-- Touch Control Buttons -->
<div class="mobile-controls">
  <div class="mobile-btn" id="touch-left">◀</div>
  <div class="mobile-btn" id="touch-slow">⏳</div>
  <div class="mobile-btn" id="touch-right">▶</div>
</div>

<script>
// ===== GAME VARIABLES =====
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");
let W = canvas.width, H = canvas.height;

const scoreEl = document.getElementById("score");
const bestEl = document.getElementById("best");
const levelEl = document.getElementById("level");
const startBtn = document.getElementById("start");
const pauseBtn = document.getElementById("pause");
const resetBtn = document.getElementById("reset");

let running=false, paused=false, frame=0;
let score=0, best=0, level=1;
let keys={}, slowCooldown=0;

let player={x:W/2,y:H-60,w:34,h:20,speed:6};
let asteroids=[];

// Load best score
try{best=parseInt(localStorage.getItem("sd_best")||"0")}catch(e){}
bestEl.textContent=best;

function rand(a,b){return Math.random()*(b-a)+a}

// ===== TOUCH CONTROLS =====
let leftTouch=false, rightTouch=false;

const tLeft=document.getElementById("touch-left");
const tRight=document.getElementById("touch-right");
const tSlow=document.getElementById("touch-slow");

function bindTouch(el, down, up){
  el.addEventListener("touchstart", e=>{down();e.preventDefault()});
  el.addEventListener("touchend", e=>{up();e.preventDefault()});
}

bindTouch(tLeft, ()=>leftTouch=true, ()=>leftTouch=false);
bindTouch(tRight, ()=>rightTouch=true, ()=>rightTouch=false);
bindTouch(tSlow, ()=>slowCooldown=180, ()=>{});

// ===== GAME LOGIC =====
function spawnAsteroid(){
  const size=Math.max(12,Math.round(rand(14,48)));
  asteroids.push({
    x:rand(size, W-size),
    y:-size,
    size,
    speed:rand(1+level*0.2,2+level*0.7),
    rot:rand(0,Math.PI*2),
    rspd:rand(-0.03,0.03)
  });
}

function update(){
  if(!running||paused) return;

  frame++;
  if(frame%600===0){level++;levelEl.textContent=level}
  if(frame%Math.max(10,80-level*4)===0) spawnAsteroid();

  // keyboard move
  if(keys["ArrowLeft"]) player.x-=player.speed;
  if(keys["ArrowRight"]) player.x+=player.speed;

  // mobile move
  if(leftTouch) player.x-=player.speed*0.8;
  if(rightTouch) player.x+=player.speed*0.8;

  player.x=Math.max(player.w/2,Math.min(W-player.w/2,player.x));

  for(let i=asteroids.length-1;i>=0;i--){
    let a=asteroids[i];
    a.y+=a.speed*(slowCooldown>0?0.35:1);
    a.rot+=a.rspd;

    if(a.y-a.size>H){
      asteroids.splice(i,1);
      score+=Math.round(10+level*2);
      scoreEl.textContent=score;
      if(score>best){
        best=score;bestEl.textContent=best;
        localStorage.setItem("sd_best",best);
      }
    }

    if(collideRectCircle(player,a)){
      endGame();return;
    }
  }
  if(slowCooldown>0) slowCooldown--;
}

function draw(){
  ctx.clearRect(0,0,W,H);
  drawStars();

  // player
  ctx.save();
  ctx.translate(player.x,player.y);
  ctx.fillStyle="#9fe9ff";
  roundRect(ctx,-player.w/2,-player.h/2,player.w,player.h,4,true,false);
  ctx.fillStyle="#012a36";
  ctx.beginPath();ctx.ellipse(0,-4,6,4,0,0,Math.PI*2);ctx.fill();
  ctx.restore();

  // asteroids
  for(let a of asteroids){
    ctx.save();
    ctx.translate(a.x,a.y);
    ctx.rotate(a.rot);
    drawAsteroid(a.size);
    ctx.restore();
  }

  if(!running){
    ctx.fillStyle="rgba(0,0,0,0.45)";
    ctx.fillRect(0,H/2-36,W,72);
    ctx.fillStyle="#cfeffd";
    ctx.font="20px system-ui";ctx.textAlign="center";
    ctx.fillText("Press START to play",W/2,H/2+6);
  }

  if(paused){
    ctx.fillStyle="rgba(0,0,0,0.35)";
    ctx.fillRect(0,0,W,H);
    ctx.fillStyle="#fff";ctx.font="26px system-ui";
    ctx.fillText("PAUSED",W/2,H/2);
  }

  requestAnimationFrame(draw);
}

// ===== STARFIELD =====
const stars=Array.from({length:80},()=>({x:rand(0,W),y:rand(0,H),s:rand(0.4,1.6)}));
function drawStars(){
  for(const s of stars){
    s.y+=0.2;if(s.y>H)s.y=0;
    ctx.beginPath();
    ctx.fillStyle="rgba(255,255,255,"+(0.05+s.s*0.15)+")";
    ctx.arc(s.x,s.y,s.s,0,Math.PI*2);ctx.fill();
  }
}

// ===== ASTEROID SHAPE =====
function drawAsteroid(size){
  ctx.beginPath();
  const spikes=6;
  for(let i=0;i<=spikes;i++){
    const ang=(i/spikes)*Math.PI*2;
    const r=size*(0.6+Math.random()*0.6);
    ctx.lineTo(Math.cos(ang)*r,Math.sin(ang)*r);
  }
  ctx.fillStyle="#bda07b";ctx.fill();
  ctx.strokeStyle="rgba(0,0,0,0.25)";ctx.stroke();
}

function roundRect(ctx,x,y,w,h,r,f,s){
  ctx.beginPath();ctx.moveTo(x+r,y);
  ctx.arcTo(x+w,y,x+w,y+h,r);
  ctx.arcTo(x+w,y+h,x,y+h,r);
  ctx.arcTo(x,y+h,x,y,r);
  ctx.arcTo(x,y,x+w,y,r);
  ctx.closePath();if(f)ctx.fill();if(s)ctx.stroke();
}

function collideRectCircle(rect,circ){
  const rx=rect.x-rect.w/2, ry=rect.y-rect.h/2;
  const cx=circ.x, cy=circ.y, r=circ.size;
  const closestX=Math.max(rx,Math.min(cx,rx+rect.w));
  const closestY=Math.max(ry,Math.min(cy,ry+rect.h));
  const dx=cx-closestX, dy=cy-closestY;
  return dx*dx+dy*dy<r*r;
}

// ===== LOOP =====
function tick(){
  update();
  setTimeout(()=>{if(running&&!paused)requestAnimationFrame(tick)},16);
}

// ===== CONTROLS =====
window.addEventListener("keydown", e=>{
  keys[e.key]=true;
  if(e.key===" ") slowCooldown=180;
});
window.addEventListener("keyup", e=>keys[e.key]=false);

canvas.addEventListener("mousemove", e=>{
  const r=canvas.getBoundingClientRect();
  player.x=(e.clientX-r.left)*(canvas.width/r.width);
});

startBtn.addEventListener("click",()=>{running?restartGame():startGame()});
pauseBtn.addEventListener("click",()=>{if(running){paused=!paused;pauseBtn.textContent=paused?"Resume":"Pause"}});
resetBtn.addEventListener("click",resetAll);

function startGame(){
  running=true;paused=false;frame=0;score=0;level=1;
  asteroids=[];scoreEl.textContent=score;levelEl.textContent=level;
  requestAnimationFrame(draw);tick();
}
function restartGame(){running=false;setTimeout(startGame,120)}
function resetAll(){
  running=false;paused=false;asteroids=[];score=0;
  scoreEl.textContent=0;levelEl.textContent=1;
}

function endGame(){
  running=false;paused=false;
  ctx.fillStyle="rgba(0,0,0,0.45)";
  ctx.fillRect(0,H/2-60,W,120);
  ctx.fillStyle="#fff";ctx.font="28px system-ui";ctx.textAlign="center";
  ctx.fillText("GAME OVER",W/2,H/2-6);
  ctx.font="18px system-ui";
  ctx.fillText("Score: "+score+" — Press START",W/2,H/2+26);
}

draw();
</script>
</body>
</html>
