# Jumpy-bird
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Flappy Bird Deluxe</title>
<style>
* { box-sizing:border-box; }
body {
  margin:0;
  min-height:100vh;
  display:flex;
  justify-content:center;
  align-items:center;
  background:#172033;
  color:white;
  font-family:Arial,sans-serif;
}
#game { position:relative;width:400px;max-width:100vw; }
canvas {
  display:block;width:100%;
  background:#70c5ce;
  border:4px solid white;
  border-radius:12px;
}
.panel {
  position:absolute;inset:0;
  display:flex;flex-direction:column;
  justify-content:center;align-items:center;
  text-align:center;padding:15px;
  background:rgba(0,0,0,.78);
  border-radius:10px;
}
.hidden { display:none!important; }
h1 { margin:5px;font-size:34px;text-shadow:2px 3px #222; }
h2 { margin:5px; }
p { margin:6px; }
button {
  border:0;border-radius:10px;
  padding:12px 20px;margin:5px;
  font-size:15px;font-weight:bold;
  cursor:pointer;background:#ffd43b;color:#222;
}
button:hover { filter:brightness(1.12); }
button:disabled { opacity:.5;cursor:not-allowed; }
.small { font-size:12px; }
#shop {
  justify-content:flex-start;
  overflow-y:auto;padding-top:12px;
}
#shopList {
  width:100%;display:flex;
  flex-direction:column;gap:6px;margin:8px 0;
}
.skin {
  background:rgba(255,255,255,.13);
  border:1px solid rgba(255,255,255,.3);
  border-radius:9px;padding:8px;
  display:flex;align-items:center;
  justify-content:space-between;gap:6px;
  text-align:left;font-size:12px;
}
.skin strong { display:block;font-size:14px; }
.skin button { margin:0;padding:8px 10px;font-size:12px; }
#topbar {
  position:absolute;top:10px;left:10px;right:10px;
  display:flex;justify-content:space-between;
  font-weight:bold;text-shadow:1px 2px #222;
  pointer-events:none;
}
</style>
</head>
<body>
<div id="game">
  <canvas id="canvas" width="400" height="600"></canvas>

  <div id="topbar">
    <span id="score">Score: 0</span>
    <span id="coins">🪙 0</span>
  </div>

  <div class="panel" id="start">
    <h1>FLAPPY BIRD</h1>
    <p>DELUXE EDITION</p>
    <p id="selectedName">Selected: Classic Bird</p>
    <p id="bestStart">🏆 Best: 0</p>
    <p id="multiplierStart">💰 Multiplier: 1X</p>
    <p id="rebirthStart">♻️ Rebirths: 0</p>
    <button id="playBtn">PLAY</button>
    <button id="openShop">🐦 BIRD SHOP</button>
    <button id="rebirthBtn">♻️ REBIRTH</button>
    <p class="small">SPACE or tap to fly</p>
  </div>

  <div class="panel hidden" id="over">
    <h1>HAHA YOU LOST!</h1>
    <h2 id="deathMessage">SKILL ISSUE 💀</h2>
    <p id="finalScore">Score: 0</p>
    <p id="newBest"></p>
    <button id="againBtn">PLAY AGAIN</button>
    <button id="overShopBtn">BIRD SHOP</button>
    <button id="overRebirthBtn">♻️ REBIRTH</button>
  </div>

  <div class="panel hidden" id="shop">
    <h2>🐦 BIRD SHOP</h2>
    <p>🪙 Coins: <span id="shopCoins">0</span></p>
    <p id="shopMultiplier">Multiplier: 1X</p>
    <p class="small">Better birds earn more coins per pipe!</p>
    <div id="shopList"></div>
    <button id="closeShop">BACK</button>
  </div>
</div>

<script>
const canvas = document.getElementById("canvas");
const ctx = canvas.getContext("2d");
const $ = id => document.getElementById(id);

const skins = [
  {id:"classic",name:"Classic Bird",cost:0,reward:1,color:"#ffd43b",type:"classic"},
  {id:"skinny",name:"Skinny Bird",cost:50,reward:2,color:"#ffe66d",type:"skinny"},
  {id:"fatty",name:"Fatty Bird",cost:100,reward:3,color:"#ffb347",type:"fatty"},
  {id:"muscle",name:"Muscular Bird",cost:200,reward:4,color:"#ffca3a",type:"muscle"},
  {id:"super",name:"Super Bird",cost:350,reward:5,color:"#65d6ff",type:"super"},
  {id:"gold",name:"Golden Bird",cost:500,reward:10,color:"#ffdf00",type:"gold"},
  {id:"nerd",name:"Nerd Bird",cost:650,reward:12,color:"#b9e6ff",type:"nerd"},
  {id:"gangster",name:"Gangster Bird",cost:800,reward:15,color:"#b8a1ff",type:"gangster"},
  {id:"business",name:"Business Bird",cost:1000,reward:20,color:"#e9e9e9",type:"business"}
];

let coins = Number(localStorage.getItem("fbCoins")) || 0;
let best = Number(localStorage.getItem("fbBest")) || 0;
let rebirths = Number(localStorage.getItem("fbRebirths")) || 0;
let owned = JSON.parse(localStorage.getItem("fbOwned") || '["classic"]');
let selected = localStorage.getItem("fbSelected") || "classic";

if (!skins.some(s => s.id === selected) || !owned.includes(selected)) {
  selected = "classic";
}

let bird,pipes,score,frame,running,gameOver;
let wingTime = 0;
let sceneryOffset = 0;
let scenerySpeed = 0;
let flapBoost = 0;

const multiplier = () => Math.pow(2,rebirths);
const rebirthCost = () => 1000 * Math.pow(2,rebirths);

const clouds = [
  {x:40,y:100,s:1},
  {x:230,y:170,s:.8},
  {x:350,y:70,s:1.1}
];

function save() {
  localStorage.setItem("fbCoins",coins);
  localStorage.setItem("fbBest",best);
  localStorage.setItem("fbOwned",JSON.stringify(owned));
  localStorage.setItem("fbSelected",selected);
  localStorage.setItem("fbRebirths",rebirths);
}

function currentSkin() {
  return skins.find(s => s.id === selected) || skins[0];
}

function updateLabels() {
  $("coins").textContent = "🪙 " + coins;
  $("shopCoins").textContent = coins;
  $("bestStart").textContent = "🏆 Best: " + best;
  $("selectedName").textContent = "Selected: " + currentSkin().name;
  $("multiplierStart").textContent = "💰 Multiplier: " + multiplier() + "X";
  $("rebirthStart").textContent = "♻️ Rebirths: " + rebirths;
  $("shopMultiplier").textContent = "Multiplier: " + multiplier() + "X";
  $("rebirthBtn").textContent = "♻️ REBIRTH (" + rebirthCost() + " coins)";
  $("overRebirthBtn").textContent = "♻️ REBIRTH (" + rebirthCost() + " coins)";
}

function showPanel(id) {
  ["start","over","shop"].forEach(p => $(p).classList.add("hidden"));
  if(id) $(id).classList.remove("hidden");
}

// REBIRTH SYSTEM
function doRebirth() {
  const cost = rebirthCost();

  if(coins < cost) {
    alert("Not enough coins! You need " + cost + " coins to rebirth.");
    return;
  }

  const confirmRebirth = confirm(
    "REBIRTH NOW?\n\nCost: " + cost + " coins\n" +
    "New multiplier: " + (multiplier()*2) + "X\n\n" +
    "You keep your coins and birds!"
  );

  if(!confirmRebirth) return;

  coins -= cost;
  rebirths++;
  save();
  updateLabels();

  alert("REBIRTH SUCCESSFUL! 🔥\nYour multiplier is now " + multiplier() + "X!");
}

function drawHouse(x,y,scale) {
  ctx.save();
  ctx.translate(x,y);
  ctx.scale(scale,scale);

  // House body
  ctx.fillStyle="#f0c58a";
  ctx.fillRect(0,20,85,65);

  // Roof
  ctx.fillStyle="#a64d35";
  ctx.beginPath();
  ctx.moveTo(-10,22);
  ctx.lineTo(42,-20);
  ctx.lineTo(95,22);
  ctx.closePath();
  ctx.fill();

  // Door
  ctx.fillStyle="#75452c";
  ctx.fillRect(34,49,20,36);
  ctx.fillStyle="#f6d45b";
  ctx.beginPath();
  ctx.arc(49,67,2,0,Math.PI*2);
  ctx.fill();

  // Windows
  ctx.fillStyle="#77c8ef";
  ctx.fillRect(9,34,17,17);
  ctx.fillRect(59,34,17,17);
  ctx.strokeStyle="white";
  ctx.lineWidth=2;
  ctx.strokeRect(9,34,17,17);
  ctx.strokeRect(59,34,17,17);
  ctx.beginPath();
  ctx.moveTo(17.5,34);ctx.lineTo(17.5,51);
  ctx.moveTo(9,42.5);ctx.lineTo(26,42.5);
  ctx.moveTo(67.5,34);ctx.lineTo(67.5,51);
  ctx.moveTo(59,42.5);ctx.lineTo(76,42.5);
  ctx.stroke();

  ctx.restore();
}

function drawTree(x,y,scale) {
  ctx.save();
  ctx.translate(x,y);
  ctx.scale(scale,scale);

  ctx.fillStyle="#79502d";
  ctx.fillRect(-6,15,12,50);

  ctx.fillStyle="#318c43";
  ctx.beginPath();
  ctx.arc(0,0,25,0,Math.PI*2);
  ctx.arc(-17,12,19,0,Math.PI*2);
  ctx.arc(17,12,19,0,Math.PI*2);
  ctx.fill();

  ctx.fillStyle="#50b95b";
  ctx.beginPath();
  ctx.arc(-8,-7,10,0,Math.PI*2);
  ctx.fill();

  ctx.restore();
}

function drawBackground() {
  const sky = ctx.createLinearGradient(0,0,0,600);
  sky.addColorStop(0,"#62c9ed");
  sky.addColorStop(1,"#b4edee");
  ctx.fillStyle=sky;
  ctx.fillRect(0,0,400,600);

  ctx.fillStyle="rgba(255,255,255,.8)";
  clouds.forEach(c => {
    ctx.beginPath();
    ctx.arc(c.x,c.y,18*c.s,0,Math.PI*2);
    ctx.arc(c.x+20*c.s,c.y-8*c.s,22*c.s,0,Math.PI*2);
    ctx.arc(c.x+43*c.s,c.y,16*c.s,0,Math.PI*2);
    ctx.fill();
  });

  // Distant hills
  ctx.fillStyle="#9bdc72";
  ctx.beginPath();
  ctx.moveTo(0,470);
  ctx.quadraticCurveTo(100,410,200,470);
  ctx.quadraticCurveTo(300,420,400,470);
  ctx.fill();

  // Moving houses and trees
  const spacing = 230;
  for(let i=0;i<4;i++) {
    let x = ((i*spacing - sceneryOffset*.65) % 920 + 920) % 920 - 90;
    drawHouse(x,425,0.85);
    drawTree(x+105,470,0.9);
    drawTree(x+175,465,0.75);
  }

  // Ground
  ctx.fillStyle="#ded895";
  ctx.fillRect(0,550,400,50);
  ctx.fillStyle="#75c747";
  ctx.fillRect(0,545,400,12);

  ctx.fillStyle="#b8a96c";
  for(let x=0;x<400;x+=25) {
    let offset=(x-sceneryOffset*1.5%25+25)%400;
    ctx.fillRect(offset,565,13,5);
  }
}

function drawPipe(x,y,height,top) {
  ctx.fillStyle="#258b36";
  ctx.fillRect(x,y,60,height);
  ctx.fillStyle="#48c64e";
  ctx.fillRect(x+7,y,12,height);
  ctx.fillStyle="#176c2a";
  ctx.fillRect(x+50,y,7,height);

  const capY=top?y+height-18:y;
  ctx.fillStyle="#258b36";
  ctx.fillRect(x-5,capY,70,18);
  ctx.fillStyle="#51d65b";
  ctx.fillRect(x+2,capY+3,10,12);
}

function drawBird(x,y) {
  const skin=currentSkin();
  const t=skin.type;

  ctx.save();
  ctx.translate(x,y);
  ctx.rotate(Math.min(.5,Math.max(-.5,bird.vy*.06)));

  let width=34,height=27;
  if(t==="skinny"){width=37;height=19;}
  if(t==="fatty"){width=37;height=34;}
  if(t==="muscle"){width=39;height=30;}
  if(t==="super"||t==="gold"){width=36;height=29;}

  ctx.fillStyle=skin.color;
  ctx.beginPath();
  ctx.ellipse(0,0,width/2,height/2,0,0,Math.PI*2);
  ctx.fill();
  ctx.lineWidth=2;
  ctx.strokeStyle="rgba(0,0,0,.2)";
  ctx.stroke();

  // Two animated wings
  const flap=Math.sin(wingTime)*7;
  ctx.fillStyle=t==="gold"?"#f5a900":"#e5a928";
  ctx.strokeStyle="#9b671d";
  ctx.lineWidth=1.5;

  [-1,1].forEach(side=>{
    ctx.save();
    ctx.translate(-4,side===-1?-5:5);
    ctx.rotate(side===-1?-flap*.035:flap*.035);
    ctx.beginPath();
    ctx.ellipse(0,side===-1?-8:8,12,6,side*.25,0,Math.PI*2);
    ctx.fill();
    ctx.stroke();
    ctx.restore();
  });

  if(t==="muscle") {
    ctx.fillStyle="#e5a928";
    ctx.beginPath();
    ctx.arc(-17,-3,7,0,Math.PI*2);
    ctx.arc(-17,7,7,0,Math.PI*2);
    ctx.fill();
  }

  if(t==="super") {
    ctx.fillStyle="#e53935";
    ctx.beginPath();
    ctx.moveTo(-12,-10);ctx.lineTo(-26,-17);
    ctx.lineTo(-19,-2);ctx.closePath();ctx.fill();
  }

  if(t==="nerd") {
    ctx.strokeStyle="#333";ctx.lineWidth=2;
    ctx.beginPath();
    ctx.arc(7,-5,7,0,Math.PI*2);
    ctx.arc(21,-5,7,0,Math.PI*2);
    ctx.moveTo(14,-5);ctx.lineTo(15,-5);
    ctx.stroke();
  }

  if(t==="gangster") {
    ctx.fillStyle="#222";
    ctx.fillRect(3,-14,24,5);
    ctx.fillRect(7,-19,15,6);
  }

  if(t==="business") {
    ctx.fillStyle="#e53935";
    ctx.beginPath();
    ctx.moveTo(0,8);ctx.lineTo(5,8);
    ctx.lineTo(3,17);ctx.lineTo(-1,12);
    ctx.fill();
  }

  if(t==="gold") {
    ctx.fillStyle="#fff1a6";
    ctx.beginPath();ctx.arc(-8,-9,5,0,Math.PI*2);ctx.fill();
    ctx.fillStyle="#d99b00";ctx.fillRect(-14,-19,23,4);
  }

  // Eye
  ctx.fillStyle="white";
  ctx.beginPath();ctx.arc(12,-7,7,0,Math.PI*2);ctx.fill();
  ctx.fillStyle="#222";
  ctx.beginPath();ctx.arc(14,-7,3,0,Math.PI*2);ctx.fill();

  // Beak
  ctx.fillStyle="#ff792b";
  ctx.beginPath();
  ctx.moveTo(width/2-2,-1);
  ctx.lineTo(width/2+13,4);
  ctx.lineTo(width/2-2,9);
  ctx.closePath();ctx.fill();

  // Feet
  ctx.strokeStyle="#e87822";ctx.lineWidth=2;
  ctx.beginPath();
  ctx.moveTo(-5,height/2-1);ctx.lineTo(-5,height/2+7);
  ctx.lineTo(-10,height/2+7);
  ctx.moveTo(7,height/2-1);ctx.lineTo(7,height/2+7);
  ctx.lineTo(12,height/2+7);
  ctx.stroke();

  ctx.restore();
}

function resetGame() {
  bird={x:90,y:280,vy:0};
  pipes=[];
  score=0;
  frame=0;
  wingTime=0;
  flapBoost=0;
  running=false;
  gameOver=false;
  $("score").textContent="Score: 0";
}

function startGame() {
  resetGame();
  running=true;
  showPanel(null);
}

function flap() {
  if(!running||gameOver)return;
  bird.vy=-6.8;
  wingTime+=1;
  flapBoost=8;
}

function endGame() {
  if(gameOver)return;
  running=false;
  gameOver=true;

  const previousBest=best;
  if(score>best)best=score;

  const messages=[
    "SKILL ISSUE. 💀",
    "BRO FELL OFF 💀",
    "THE PIPES WON 😂",
    "BETTER LUCK NEXT TIME!",
    "BIRD.exe HAS STOPPED WORKING",
    "THAT WAS EMBARRASSING 💀",
    "NOT YOUR BEST MOMENT 😂"
  ];

  $("deathMessage").textContent=messages[Math.floor(Math.random()*messages.length)];
  $("finalScore").textContent="Score: "+score;

  if(score>previousBest&&score>0) {
    $("newBest").textContent="WOW! THAT'S PRETTY GOOD! 🔥 NEW HIGH SCORE!";
  } else {
    $("newBest").textContent="🏆 Best: "+best;
  }

  save();
  updateLabels();
  showPanel("over");
}

function addPipe() {
  const gap=155;
  const topHeight=70+Math.random()*250;
  pipes.push({x:410,top:topHeight,bottom:topHeight+gap,scored:false});
}

function update() {
  if(!running)return;

  frame++;
  wingTime+=.25;
  bird.vy+=.36;
  bird.y+=bird.vy;

  // Scenery moves during flight and gets a little faster after a flap
  scenerySpeed=2.1+(flapBoost>0?1.7:0);
  sceneryOffset+=scenerySpeed;
  if(flapBoost>0)flapBoost--;

  if(frame%95===0)addPipe();

  pipes.forEach(p=>{
    p.x-=2.6;

    if(!p.scored&&p.x+60<bird.x) {
      p.scored=true;
      score++;

      const earned=currentSkin().reward*multiplier();
      coins+=earned;

      save();
      updateLabels();
      $("score").textContent="Score: "+score;
    }

    if(
      bird.x+13>p.x-5&&
      bird.x-13<p.x+65&&
      (bird.y-12<p.top||bird.y+12>p.bottom)
    )endGame();
  });

  pipes=pipes.filter(p=>p.x>-80);

  if(bird.y<0||bird.y>535)endGame();

  clouds.forEach(c=>{
    c.x-=.3*c.s;
    if(c.x<-60)c.x=450;
  });
}

function draw() {
  drawBackground();

  pipes.forEach(p=>{
    drawPipe(p.x,0,p.top,true);
    drawPipe(p.x,p.bottom,550-p.bottom,false);
  });

  if(bird)drawBird(bird.x,bird.y);

  update();
  requestAnimationFrame(draw);
}

function renderShop() {
  $("shopList").innerHTML="";

  skins.forEach((skin,index)=>{
    const isOwned=owned.includes(skin.id);
    const isSelected=selected===skin.id;

    const row=document.createElement("div");
    row.className="skin";

    const info=document.createElement("div");
    info.innerHTML=
      `<strong>${index+1}. ${skin.name}</strong>
       🪙 ${skin.cost} coins · Earn ${skin.reward*multiplier()} per pipe`;

    const button=document.createElement("button");

    if(isSelected) {
      button.textContent="SELECTED";
      button.disabled=true;
    } else if(isOwned) {
      button.textContent="SELECT";
      button.onclick=()=>{
        selected=skin.id;
        save();
        updateLabels();
        renderShop();
      };
    } else {
      button.textContent="BUY";
      button.disabled=coins<skin.cost;
      button.onclick=()=>{
        if(coins<skin.cost)return;
        coins-=skin.cost;
        owned.push(skin.id);
        selected=skin.id;
        save();
        updateLabels();
        renderShop();
      };
    }

    row.append(info,button);
    $("shopList").appendChild(row);
  });

  updateLabels();
}

function openShop() {
  renderShop();
  showPanel("shop");
}

$("playBtn").onclick=startGame;
$("againBtn").onclick=startGame;
$("openShop").onclick=openShop;
$("overShopBtn").onclick=openShop;
$("rebirthBtn").onclick=doRebirth;
$("overRebirthBtn").onclick=doRebirth;

$("closeShop").onclick=()=>{
  if(gameOver)showPanel("over");
  else if(!running)showPanel("start");
  updateLabels();
};

canvas.addEventListener("pointerdown",()=>{
  if(running)flap();
});

document.addEventListener("keydown",e=>{
  if(e.code==="Space") {
    e.preventDefault();
    if(running)flap();
  }
});

resetGame();
updateLabels();
showPanel("start");
draw();
</script>
</body>
</html>
