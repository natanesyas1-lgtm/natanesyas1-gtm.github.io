<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
<meta name="description" content="Math Survivor — shoot zombies by solving math questions!">
<meta property="og:title" content="Math Survivor 🧟">
<meta property="og:description" content="Zombies are coming. Answer math questions to shoot them. Survive!">
<meta name="theme-color" content="#04070f">
<title>Math Survivor 🧟</title>
<style>
*{box-sizing:border-box;margin:0;padding:0;}
html,body{width:100%;height:100%;background:#04070f;overflow:hidden;font-family:'Courier New',monospace;}
#wrap{display:flex;flex-direction:column;width:100vw;height:100vh;}
#gc{width:100%;flex:1;display:block;cursor:crosshair;}
#ibar{
  display:flex;align-items:center;gap:10px;
  padding:10px 16px;
  background:#0a0f05;
  border-top:2px solid #1a3010;
  flex-shrink:0;
}
#ibar label{color:#4a7a30;font-size:.8rem;letter-spacing:2px;text-transform:uppercase;white-space:nowrap;}
#ai{
  flex:1;padding:10px 14px;
  background:#0d1a08;
  border:1.5px solid #2a4a18;
  border-radius:8px;
  color:#e8ffe8;font-size:1.1rem;font-weight:bold;
  outline:none;caret-color:#10b981;
  transition:border-color .2s;
  font-family:'Courier New',monospace;
}
#ai::placeholder{color:#2a4a18;}
#ai:focus{border-color:#10b981;}
#fbtn{
  padding:10px 20px;
  background:linear-gradient(135deg,#1a5a0a,#2a8a14);
  border:none;border-radius:8px;
  color:#e8ffe8;font-size:.95rem;font-weight:bold;
  cursor:pointer;letter-spacing:1px;
  transition:all .15s;white-space:nowrap;
  font-family:'Courier New',monospace;
}
#fbtn:hover{background:linear-gradient(135deg,#2a7a14,#38a820);transform:scale(1.04);}
#fbtn:active{transform:scale(.97);}
#wn{
  position:fixed;top:50%;left:50%;
  transform:translate(-50%,-50%);
  font-family:Impact,sans-serif;
  font-size:clamp(2rem,8vw,3.5rem);
  letter-spacing:6px;
  color:#ef4444;
  text-shadow:0 0 30px rgba(239,68,68,.9),0 0 60px rgba(239,68,68,.4);
  pointer-events:none;
  opacity:0;
  transition:opacity .4s;
  z-index:100;
}
</style>
</head>
<body>
<div id="wrap">
  <canvas id="gc"></canvas>
  <div id="wn"></div>
  <div id="ibar">
    <label>Answer:</label>
    <input id="ai" type="number" placeholder="Type answer → press Enter to FIRE 🔫" autocomplete="off"/>
    <button id="fbtn" onclick="MB.fire()">🔫 FIRE</button>
  </div>
</div>
<script>
(function(){
'use strict';
const canvas=document.getElementById('gc');
const ctx=canvas.getContext('2d');
let W=0,H=0;

function resize(){
  const r=canvas.getBoundingClientRect();
  W=canvas.width=r.width;
  H=canvas.height=r.height;
  STARS.length=0;
  for(let i=0;i<90;i++) STARS.push({x:Math.random()*W,y:Math.random()*(H*.65),r:Math.random()*1.4+.3,t:Math.random()*Math.PI*2});
}
window.addEventListener('resize',resize);

const STARS=[];
let state='menu';
let score=0,lives=3,wave=1,kills=0,frame=0;
let zombies=[],bullets=[],particles=[],floats=[];
let screenShake=0,spawnTimer=0,playerFlash=0;
let pName=''
try{pName=localStorage.getItem('mb_name')||'';}catch(e){}

function rnd(a,b){return Math.floor(Math.random()*(b-a+1))+a;}

function genProblem(){
  const lvl=Math.min(wave,8);
  const ops=lvl<3?['+','-']:lvl<5?['+','-','×']:lvl<7?['+','-','×']:['+',' -','×','÷'];
  const op=ops[rnd(0,ops.length-1)];
  const mx=Math.min(4+lvl*3,40);
  let a,b,ans;
  if(op==='+'){a=rnd(1,mx);b=rnd(1,mx);ans=a+b;}
  else if(op==='-'){a=rnd(2,mx);b=rnd(1,a);ans=a-b;}
  else if(op==='×'){a=rnd(1,Math.min(mx,12));b=rnd(1,Math.min(mx,12));ans=a*b;}
  else{b=rnd(2,10);ans=rnd(1,10);a=b*ans;}
  return{text:`${a} ${op} ${b} = ?`,ans};
}

class Zombie{
  constructor(){
    const p=genProblem();
    this.text=p.text;this.ans=p.ans;
    this.x=W+80+rnd(0,120);
    this.y=H-80;
    this.speed=(0.55+wave*.11+Math.random()*.22)*(W/700);
    this.wf=rnd(0,100);
    this.dead=false;this.df=0;
    this.hue=rnd(95,140);
  }
  update(){
    if(this.dead){this.df++;return this.df>38;}
    this.x-=this.speed;this.wf++;
    return false;
  }
  draw(){
    ctx.save();
    if(this.dead){
      ctx.globalAlpha=Math.max(0,1-this.df/38);
      ctx.translate(this.x,this.y+this.df*1.8);
      ctx.rotate(this.df*.09);
    }else{
      ctx.translate(this.x,this.y);
    }
    const wk=this.dead?0:Math.sin(this.wf*.14)*7;
    const as=this.dead?0:Math.sin(this.wf*.14)*14;

    // shadow
    ctx.fillStyle='rgba(0,0,0,.3)';
    ctx.beginPath();ctx.ellipse(0,9,16,5,0,0,Math.PI*2);ctx.fill();

    // legs
    ctx.strokeStyle=`hsl(${this.hue-20},45%,22%)`;ctx.lineWidth=7;ctx.lineCap='round';
    ctx.beginPath();ctx.moveTo(-5,-12);ctx.lineTo(-7-wk*.4,9);ctx.stroke();
    ctx.beginPath();ctx.moveTo(5,-12);ctx.lineTo(7+wk*.4,9);ctx.stroke();
    ctx.strokeStyle=`hsl(${this.hue-30},35%,18%)`;ctx.lineWidth=5;
    ctx.beginPath();ctx.moveTo(-7-wk*.4,9);ctx.lineTo(-15-wk*.3,9);ctx.stroke();
    ctx.beginPath();ctx.moveTo(7+wk*.4,9);ctx.lineTo(15+wk*.3,9);ctx.stroke();

    // body
    ctx.fillStyle=`hsl(${this.hue-10},35%,28%)`;
    rRect(ctx,-13,-46,26,34,3);ctx.fill();
    ctx.strokeStyle=`hsl(${this.hue-10},35%,18%)`;ctx.lineWidth=1;
    ctx.beginPath();ctx.moveTo(-4,-46);ctx.lineTo(-4,-12);ctx.stroke();

    // arms
    ctx.strokeStyle=`hsl(${this.hue},50%,30%)`;ctx.lineWidth=6;ctx.lineCap='round';
    ctx.beginPath();ctx.moveTo(-12,-38);ctx.lineTo(-34,-28+as*.3);ctx.stroke();
    ctx.beginPath();ctx.moveTo(12,-38);ctx.lineTo(34,-24-as*.3);ctx.stroke();
    ctx.fillStyle=`hsl(${this.hue},55%,35%)`;
    ctx.beginPath();ctx.arc(-36,-28+as*.3,5,0,Math.PI*2);ctx.fill();
    ctx.beginPath();ctx.arc(36,-24-as*.3,5,0,Math.PI*2);ctx.fill();

    // head
    ctx.fillStyle=`hsl(${this.hue},55%,36%)`;
    ctx.beginPath();ctx.arc(0,-57+wk*.15,14,0,Math.PI*2);ctx.fill();

    // eyes
    ctx.fillStyle='#ff1a1a';ctx.shadowBlur=5;ctx.shadowColor='#f00';
    ctx.beginPath();ctx.arc(-5,-59+wk*.15,3,0,Math.PI*2);ctx.fill();
    ctx.beginPath();ctx.arc(5,-59+wk*.15,3,0,Math.PI*2);ctx.fill();
    ctx.shadowBlur=0;

    // mouth stitches
    ctx.strokeStyle='#111';ctx.lineWidth=1.5;
    ctx.beginPath();ctx.moveTo(-5,-52+wk*.15);ctx.lineTo(5,-52+wk*.15);ctx.stroke();
    for(let i=-4;i<=4;i+=2){ctx.beginPath();ctx.moveTo(i,-52+wk*.15);ctx.lineTo(i,-54+wk*.15);ctx.stroke();}

    // hair
    ctx.fillStyle=`hsl(30,40%,15%)`;
    ctx.beginPath();ctx.arc(0,-67+wk*.15,10,Math.PI,0);ctx.fill();
    ctx.fillRect(-10,-71+wk*.15,20,6);

    // math bubble
    if(!this.dead){
      ctx.font='bold 11px Courier New,monospace';
      const tw=ctx.measureText(this.text).width;
      const bw=tw+14,bh=20,bx=-bw/2,by=-92;
      ctx.fillStyle='rgba(5,10,3,.85)';
      rRect(ctx,bx,by,bw,bh,5);ctx.fill();
      ctx.strokeStyle=`hsl(${this.hue},60%,40%)`;ctx.lineWidth=1;
      rRect(ctx,bx,by,bw,bh,5);ctx.stroke();
      ctx.fillStyle='#dfffdf';ctx.textAlign='center';
      ctx.fillText(this.text,0,by+13);
    }
    ctx.restore();
  }
}

class Bullet{
  constructor(ta){
    this.x=100;this.y=H-105;this.ta=ta;
    this.vx=15*(W/700);this.trail=[];this.done=false;
  }
  update(){
    this.trail.push({x:this.x,y:this.y});
    if(this.trail.length>10)this.trail.shift();
    this.x+=this.vx;
    for(const z of zombies){
      if(!z.dead&&z.ans===this.ta&&Math.abs(z.x-this.x)<38&&Math.abs(z.y-28-this.y)<72){
        z.dead=true;kills++;score+=10*wave;
        spawnExplosion(z.x,z.y-30,z.hue);
        floats.push({txt:'+'+10*wave,x:z.x,y:z.y-55,life:1,vy:-1.6});
        if(kills>=wave*8){wave++;kills=0;showWave();}
        this.done=true;return true;
      }
    }
    return this.done||this.x>W+50;
  }
  draw(){
    this.trail.forEach((p,i)=>{
      const t=(i+1)/this.trail.length;
      ctx.beginPath();ctx.arc(p.x,p.y,3.5*t,0,Math.PI*2);
      ctx.fillStyle=`rgba(255,210,50,${t*.4})`;ctx.fill();
    });
    ctx.beginPath();ctx.arc(this.x,this.y,7,0,Math.PI*2);
    ctx.fillStyle='rgba(255,180,20,.25)';ctx.fill();
    ctx.beginPath();ctx.arc(this.x,this.y,4,0,Math.PI*2);
    ctx.fillStyle='#ffd700';ctx.fill();
    ctx.beginPath();ctx.arc(this.x,this.y,2,0,Math.PI*2);
    ctx.fillStyle='#fff';ctx.fill();
  }
}

function spawnExplosion(x,y,hue){
  for(let i=0;i<26;i++){
    const a=Math.random()*Math.PI*2,s=Math.random()*5+2;
    particles.push({x,y,vx:Math.cos(a)*s,vy:Math.sin(a)*s-3,life:1,decay:.032+Math.random()*.02,r:Math.random()*4+2,color:`hsl(${Math.random()*40+10},100%,${50+Math.random()*20}%)`});
  }
  for(let i=0;i<10;i++){
    const a=Math.random()*Math.PI*2,s=Math.random()*3+1;
    particles.push({x,y,vx:Math.cos(a)*s,vy:Math.sin(a)*s-1,life:1,decay:.025+Math.random()*.015,r:Math.random()*3+1,color:`hsl(${hue},70%,35%)`});
  }
}

function showWave(){
  const el=document.getElementById('wn');
  if(!el)return;
  el.textContent='WAVE '+wave;el.style.opacity='1';
  setTimeout(()=>el.style.opacity='0',1800);
}

function rRect(c,x,y,w,h,r){
  c.beginPath();c.moveTo(x+r,y);c.lineTo(x+w-r,y);c.quadraticCurveTo(x+w,y,x+w,y+r);
  c.lineTo(x+w,y+h-r);c.quadraticCurveTo(x+w,y+h,x+w-r,y+h);
  c.lineTo(x+r,y+h);c.quadraticCurveTo(x,y+h,x,y+h-r);
  c.lineTo(x,y+r);c.quadraticCurveTo(x,y,x+r,y);c.closePath();
}

function drawBg(){
  // Sky
  const sky=ctx.createLinearGradient(0,0,0,H*.75);
  sky.addColorStop(0,'#03050d');sky.addColorStop(1,'#0c1520');
  ctx.fillStyle=sky;ctx.fillRect(0,0,W,H);

  // Stars twinkle
  STARS.forEach(s=>{
    const tw=.35+.65*Math.abs(Math.sin(s.t+frame*.007));
    ctx.globalAlpha=tw;
    ctx.beginPath();ctx.arc(s.x,s.y,s.r,0,Math.PI*2);
    ctx.fillStyle='#fff';ctx.fill();
  });
  ctx.globalAlpha=1;

  // Moon
  ctx.fillStyle='#e6d8a8';
  ctx.beginPath();ctx.arc(W*.87,H*.12,Math.min(H*.065,28),0,Math.PI*2);ctx.fill();
  ctx.fillStyle='#0c1520';
  ctx.beginPath();ctx.arc(W*.87+11,H*.12-7,Math.min(H*.055,23),0,Math.PI*2);ctx.fill();

  // Distant fog
  const fog=ctx.createLinearGradient(0,H*.52,0,H*.72);
  fog.addColorStop(0,'rgba(20,35,10,0)');fog.addColorStop(1,'rgba(15,28,8,.6)');
  ctx.fillStyle=fog;ctx.fillRect(0,H*.52,W,H*.2);

  // Gravestones silhouette
  ctx.fillStyle='#060c04';
  [.18,.32,.46,.58,.72,.84].forEach((gx,i)=>{
    const gw=13+(i%3)*6,gh=20+(i%3)*9,gxp=gx*W,gyp=H-78-gh;
    ctx.fillRect(gxp-gw/2,gyp,gw,gh);
    ctx.beginPath();ctx.arc(gxp,gyp,gw/2,Math.PI,0);ctx.fill();
  });

  // Ground
  ctx.fillStyle='#0f1a07';ctx.fillRect(0,H-68,W,68);
  ctx.fillStyle='#182b0c';ctx.fillRect(0,H-68,W,10);

  // Grass tufts
  ctx.strokeStyle='#1c2e0e';ctx.lineWidth=2;
  for(let gx=12;gx<W;gx+=rnd(28,46)){
    ctx.beginPath();ctx.moveTo(gx,H-68);ctx.lineTo(gx-3,H-75);
    ctx.moveTo(gx,H-68);ctx.lineTo(gx+3,H-76);
    ctx.stroke();
  }

  // Left wall barrier (player side)
  ctx.fillStyle='rgba(20,30,10,.6)';ctx.fillRect(0,0,10,H);
  ctx.strokeStyle='#2a4a18';ctx.lineWidth=2;
  ctx.beginPath();ctx.moveTo(10,0);ctx.lineTo(10,H);ctx.stroke();
}

function drawPlayer(){
  const px=62,py=H-72;
  const bob=Math.sin(frame*.04)*1.5;
  const rc=bullets.length>0?-3:0;
  ctx.save();ctx.translate(px+rc,py+bob);

  if(playerFlash>0){
    ctx.globalAlpha=.5;ctx.fillStyle='#ef4444';
    ctx.fillRect(-22,-84,65,95);ctx.globalAlpha=1;
    playerFlash--;
  }

  // shadow
  ctx.fillStyle='rgba(0,0,0,.28)';
  ctx.beginPath();ctx.ellipse(10,4,18,5,0,0,Math.PI*2);ctx.fill();

  // legs
  ctx.strokeStyle='#1a2a4a';ctx.lineWidth=8;ctx.lineCap='round';
  ctx.beginPath();ctx.moveTo(-4,-12);ctx.lineTo(-6,5);ctx.stroke();
  ctx.beginPath();ctx.moveTo(8,-12);ctx.lineTo(10,5);ctx.stroke();

  // body
  ctx.fillStyle='#2a4a7a';rRect(ctx,-14,-48,28,36,4);ctx.fill();
  ctx.fillStyle='#1e3d6a';rRect(ctx,-14,-48,28,10,4);ctx.fill();

  // gun arm
  ctx.strokeStyle='#2a4a7a';ctx.lineWidth=7;ctx.lineCap='round';
  ctx.beginPath();ctx.moveTo(10,-34);ctx.lineTo(30,-27);ctx.stroke();
  ctx.fillStyle='#181818';rRect(ctx,22,-32,24,9,2);ctx.fill();
  ctx.fillStyle='#2a2a2a';ctx.fillRect(44,-30,5,5);

  // muzzle flash
  if(bullets.length>0&&Math.random()<.4){
    ctx.globalAlpha=.85;
    ctx.fillStyle='#ffd700';ctx.shadowBlur=12;ctx.shadowColor='#ffa500';
    ctx.beginPath();ctx.arc(50,-27,7,0,Math.PI*2);ctx.fill();
    ctx.shadowBlur=0;ctx.globalAlpha=1;
  }

  // head
  ctx.fillStyle='#f5c09a';
  ctx.beginPath();ctx.arc(2,-58,13,0,Math.PI*2);ctx.fill();

  // hair
  ctx.fillStyle='#3a1e08';
  ctx.beginPath();ctx.arc(2,-64,11,Math.PI*1.1,Math.PI*.1);ctx.fill();
  ctx.fillRect(-9,-68,19,8);

  // eye
  ctx.fillStyle='#fff';ctx.beginPath();ctx.arc(8,-58,4,0,Math.PI*2);ctx.fill();
  ctx.fillStyle='#222';ctx.beginPath();ctx.arc(9,-58,2.5,0,Math.PI*2);ctx.fill();

  ctx.restore();
}

function drawHUD(){
  // Score
  ctx.fillStyle='rgba(0,0,0,.55)';rRect(ctx,14,12,140,56,6);ctx.fill();
  ctx.strokeStyle='rgba(245,158,11,.25)';ctx.lineWidth=1;rRect(ctx,14,12,140,56,6);ctx.stroke();
  ctx.fillStyle='#f59e0b';ctx.font='bold 10px Courier New,monospace';ctx.textAlign='left';
  ctx.fillText('SCORE',24,28);
  ctx.fillStyle='#fff';ctx.font='bold 22px Impact,sans-serif';
  ctx.fillText(score,24,55);

  // Wave
  ctx.fillStyle='rgba(0,0,0,.55)';rRect(ctx,164,12,100,56,6);ctx.fill();
  ctx.strokeStyle='rgba(239,68,68,.25)';ctx.lineWidth=1;rRect(ctx,164,12,100,56,6);ctx.stroke();
  ctx.fillStyle='#ef4444';ctx.font='bold 10px Courier New,monospace';ctx.textAlign='left';
  ctx.fillText('WAVE',174,28);
  ctx.fillStyle='#fff';ctx.font='bold 22px Impact,sans-serif';
  ctx.fillText(wave,174,55);

  // Lives
  ctx.textAlign='right';
  for(let i=0;i<3;i++){
    ctx.globalAlpha=i<lives?1:.2;
    ctx.fillStyle=i<lives?'#ef4444':'#555';
    ctx.font='22px sans-serif';
    ctx.fillText('♥',W-12-(2-i)*28,36);
  }
  ctx.globalAlpha=1;

  // Wave progress bar
  const prog=Math.min(kills/(wave*8),1);
  ctx.fillStyle='rgba(0,0,0,.4)';rRect(ctx,W/2-85,16,170,12,6);ctx.fill();
  if(prog>0){
    ctx.fillStyle=prog>.75?'#ef4444':prog>.5?'#f59e0b':'#10b981';
    rRect(ctx,W/2-85,16,170*prog,12,6);ctx.fill();
  }
  ctx.fillStyle='rgba(255,255,255,.45)';ctx.font='8px Courier New,monospace';ctx.textAlign='center';
  ctx.fillText('NEXT WAVE: '+kills+'/'+(wave*8),W/2,25);
}

function drawMenu(){
  ctx.fillStyle='rgba(4,7,15,.88)';ctx.fillRect(0,0,W,H);
  ctx.textAlign='center';

  ctx.shadowBlur=28;ctx.shadowColor='#ef4444';
  ctx.fillStyle='#ef4444';ctx.font=`bold ${Math.min(42,W*.07)}px Impact,sans-serif`;
  ctx.fillText('MATH SURVIVOR',W/2,H/2-68);ctx.shadowBlur=0;

  ctx.fillStyle='#10b981';ctx.font=`bold 13px Courier New,monospace`;
  ctx.fillText('☠  ZOMBIES ARE COMING FOR YOU  ☠',W/2,H/2-30);

  ctx.fillStyle='rgba(200,255,200,.65)';ctx.font='12px Courier New,monospace';
  ctx.fillText('Each zombie carries a math problem',W/2,H/2+8);
  ctx.fillText('Type the correct answer & press ENTER to shoot it',W/2,H/2+28);
  ctx.fillText('If a zombie reaches your wall — you lose a life  ♥ ♥ ♥',W/2,H/2+50);

  if(pName){
    ctx.fillStyle='#f59e0b';ctx.font='bold 13px Courier New,monospace';
    ctx.fillText(`Welcome back, ${pName} — let's see what you've got!`,W/2,H/2+78);
  }

  if(Math.floor(frame/30)%2===0){
    ctx.fillStyle='#ffd700';ctx.font='bold 16px Impact,sans-serif';
    ctx.fillText('[ TYPE ANY ANSWER & PRESS ENTER / CLICK FIRE TO START ]',W/2,H/2+110);
  }
}

function drawGameOver(){
  ctx.fillStyle='rgba(4,3,10,.85)';ctx.fillRect(0,0,W,H);
  ctx.textAlign='center';

  ctx.shadowBlur=24;ctx.shadowColor='#ef4444';
  ctx.fillStyle='#ef4444';ctx.font='bold 44px Impact,sans-serif';
  ctx.fillText('GAME OVER',W/2,H/2-62);ctx.shadowBlur=0;

  ctx.fillStyle='#f59e0b';ctx.font='bold 26px Courier New,monospace';
  ctx.fillText('SCORE: '+score,W/2,H/2-8);

  ctx.fillStyle='rgba(200,255,200,.6)';ctx.font='14px Courier New,monospace';
  ctx.fillText('Survived to wave '+wave,W/2,H/2+26);

  if(pName){
    ctx.fillStyle='rgba(255,255,255,.5)';ctx.font='12px Courier New,monospace';
    ctx.fillText(score>=100?`Impressive, ${pName}! Keep it up.`:`Don't give up, ${pName}! The zombies fear you.`,W/2,H/2+52);
  }

  if(Math.floor(frame/30)%2===0){
    ctx.fillStyle='#10b981';ctx.font='bold 15px Impact,sans-serif';
    ctx.fillText('[ PRESS ENTER / CLICK FIRE TO TRY AGAIN ]',W/2,H/2+86);
  }
}

function update(){
  frame++;
  if(state!=='playing')return;
  if(screenShake>.4)screenShake-=.8;

  spawnTimer++;
  const iv=Math.max(50,185-wave*15);
  if(spawnTimer>=iv){
    spawnTimer=0;
    zombies.push(new Zombie());
    if(wave>=4&&Math.random()<.38)zombies.push(new Zombie());
    if(wave>=7&&Math.random()<.28)zombies.push(new Zombie());
  }

  zombies=zombies.filter(z=>{
    const rm=z.update();
    if(!z.dead&&z.x<16){
      z.dead=true;lives--;screenShake=20;playerFlash=22;
      if(lives<=0)setTimeout(()=>{state='dead';},900);
    }
    return!rm;
  });

  bullets=bullets.filter(b=>!b.update());

  particles=particles.filter(p=>{
    p.x+=p.vx;p.y+=p.vy;p.vy+=.18;p.vx*=.96;p.life-=p.decay;
    return p.life>0;
  });

  floats=floats.filter(t=>{t.y+=t.vy;t.life-=.024;return t.life>0;});
}

function draw(){
  ctx.save();
  if(screenShake>.3)ctx.translate((Math.random()-.5)*screenShake,(Math.random()-.5)*screenShake*.5);

  drawBg();

  if(state!=='menu'){
    drawPlayer();
    zombies.forEach(z=>z.draw());
    bullets.forEach(b=>b.draw());

    particles.forEach(p=>{
      ctx.globalAlpha=p.life;
      ctx.beginPath();ctx.arc(p.x,p.y,p.r,0,Math.PI*2);
      ctx.fillStyle=p.color;ctx.fill();
    });
    ctx.globalAlpha=1;

    floats.forEach(t=>{
      ctx.globalAlpha=t.life;
      ctx.fillStyle='#ffd700';ctx.font='bold 16px Impact,sans-serif';
      ctx.textAlign='center';ctx.fillText(t.txt,t.x,t.y);
    });
    ctx.globalAlpha=1;

    drawHUD();
  }

  if(state==='menu')drawMenu();
  if(state==='dead'){drawHUD();drawGameOver();}

  ctx.restore();
}

function loop(){requestAnimationFrame(loop);update();draw();}

function fireAnswer(val){
  if(state==='menu'||state==='dead'){startGame();return;}
  if(isNaN(val))return;
  const ok=zombies.some(z=>!z.dead&&z.ans===val);
  if(ok){
    bullets.push(new Bullet(val));
    document.getElementById('ai').style.borderColor='#10b981';
    setTimeout(()=>document.getElementById('ai').style.borderColor='',280);
  }else{
    screenShake=5;
    document.getElementById('ai').style.borderColor='#ef4444';
    setTimeout(()=>document.getElementById('ai').style.borderColor='',380);
  }
}

function startGame(){
  state='playing';score=0;lives=3;wave=1;kills=0;frame=0;
  zombies=[];bullets=[];particles=[];floats=[];
  spawnTimer=145;playerFlash=0;screenShake=0;
  document.getElementById('ai').focus();
}

window.MB={
  fire(){
    const inp=document.getElementById('ai');
    const val=parseInt(inp.value.trim());
    inp.value='';inp.focus();
    fireAnswer(val);
  }
};

document.getElementById('ai').addEventListener('keydown',e=>{
  if(e.key==='Enter'){
    const val=parseInt(e.target.value.trim());
    e.target.value='';
    fireAnswer(val);
  }
});

setTimeout(()=>{resize();loop();},80);
})();
</script>
</body>
</html>
