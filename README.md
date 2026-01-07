<!DOCTYPE html>
<html lang="fa">
<head>
<meta charset="UTF-8">
<title>🌠 Heliopause – Voyager Mission</title>
<style>
html,body{margin:0;overflow:hidden;background:#00020a;font-family:system-ui}
canvas{display:block}
.hud{
 position:absolute;left:16px;top:16px;
 padding:14px 18px;border-radius:14px;
 background:rgba(255,255,255,0.06);
 backdrop-filter:blur(10px);
 color:#d9f3ff;font-size:13px;min-width:380px;
}
.good{color:#9ff0ff}
.mid{color:#ffd29f}
.bad{color:#ff9f9f}
.arn{color:#ffb366}
.legend{
 position:absolute;right:16px;bottom:16px;
 font-size:12px;color:#aac;
}
</style>
</head>
<body>

<canvas id="c"></canvas>
<div class="hud" id="hud"></div>
<div class="legend">
🌞 Solar Wind<br>
🌠 Termination Shock (~85 AU)<br>
🌌 Heliopause (~120 AU)
</div>

<script>
const c=document.getElementById("c");
const ctx=c.getContext("2d");
let w,h;
function resize(){w=c.width=innerWidth;h=c.height=innerHeight}
resize();addEventListener("resize",resize);

// Scale
const AU=65;

// Sun
const sun={x:w/2,y:h/2};

// Planets
const earth ={a:AU*1 ,angle:0 ,r:6};
const jupiter={a:AU*5.2,angle:1.2,r:9};
const saturn ={a:AU*9.6,angle:2.4,r:10};

// Boundaries
const terminationShock=85;
const heliopause=120;

// Spacecraft
const ship={
  distance:AU*1,
  angle:0.4,
  velocity:0.22,
  r:3,
  phase:"INNER SOLAR SYSTEM"
};

// Storm (only inside heliosphere)
let storm={active:false,intensity:0};

// Radio
const txPower=210;

// Helpers
function pos(a,ang){
 return {x:sun.x+Math.cos(ang)*a,y:sun.y+Math.sin(ang)*a};
}

// Solar storm
function updateStorm(){
 if(ship.distance/AU<20){
  if(!storm.active && Math.random()<0.001){
    storm.active=true;
    storm.intensity=1+Math.random()*3;
  }
  if(storm.active){
    storm.intensity*=0.996;
    if(storm.intensity<0.1) storm.active=false;
  }
 } else {
  storm.active=false;
 }
}

// Deep space link model
function link(distAU){
 let signal=txPower/(distAU*distAU*140);
 let noise=0.09+Math.random()*0.07;

 if(distAU>terminationShock) noise+=0.15; // turbulent plasma
 if(distAU>heliopause) noise+=0.25;        // interstellar medium
 if(storm.active) noise+=storm.intensity*0.4;

 let snr=signal/noise;
 let latency=distAU*8.3; // minutes
 return {snr,latency};
}

// Update physics
function update(){
 earth.angle+=0.0012;
 jupiter.angle+=0.0005;
 saturn.angle+=0.0003;

 // Gravity assists
 if(Math.abs(ship.distance-jupiter.a)<3){
  ship.velocity+=0.05;
 }
 if(Math.abs(ship.distance-saturn.a)<3){
  ship.velocity+=0.04;
 }

 ship.distance+=ship.velocity;
 ship.angle+=0.002;

 const dAU=ship.distance/AU;
 if(dAU<15) ship.phase="INNER SOLAR SYSTEM";
 else if(dAU<terminationShock) ship.phase="OUTER SOLAR SYSTEM";
 else if(dAU<heliopause) ship.phase="TERMINATION SHOCK";
 else ship.phase="INTERSTELLAR SPACE";

 updateStorm();
}

// Draw rings
function drawRing(radiusAU,color){
 ctx.strokeStyle=color;
 ctx.beginPath();
 ctx.arc(sun.x,sun.y,radiusAU*AU,0,Math.PI*2);
 ctx.stroke();
}

// Render
function draw(){
 ctx.fillStyle="rgba(0,2,10,0.4)";
 ctx.fillRect(0,0,w,h);

 update();

 const pe=pos(earth.a,earth.angle);
 const pj=pos(jupiter.a,jupiter.angle);
 const ps=pos(saturn.a,saturn.angle);
 const pship={
  x:sun.x+Math.cos(ship.angle)*ship.distance,
  y:sun.y+Math.sin(ship.angle)*ship.distance
 };

 // Orbits
 ctx.strokeStyle="rgba(255,255,255,0.04)";
 [earth,jupiter,saturn].forEach(p=>{
  ctx.beginPath();
  ctx.arc(sun.x,sun.y,p.a,0,Math.PI*2);
  ctx.stroke();
 });

 // Boundaries
 drawRing(terminationShock,"rgba(255,180,120,0.25)");
 drawRing(heliopause,"rgba(160,120,255,0.25)");

 // Sun
 ctx.beginPath();
 ctx.arc(sun.x,sun.y,12,0,Math.PI*2);
 ctx.fillStyle="#ffcc66";
 ctx.fill();

 // Planets
 ctx.fillStyle="#2a7fff";
 ctx.beginPath();ctx.arc(pe.x,pe.y,earth.r,0,Math.PI*2);ctx.fill();

 ctx.fillStyle="#ffaa66";
 ctx.beginPath();ctx.arc(pj.x,pj.y,jupiter.r,0,Math.PI*2);ctx.fill();

 ctx.fillStyle="#d6c38a";
 ctx.beginPath();ctx.arc(ps.x,ps.y,saturn.r,0,Math.PI*2);ctx.fill();

 // Ship
 ctx.fillStyle="#ffffff";
 ctx.beginPath();ctx.arc(pship.x,pship.y,ship.r,0,Math.PI*2);ctx.fill();

 // Link
 const distAU=Math.hypot(pship.x-pe.x,pship.y-pe.y)/AU;
 const q=link(distAU);
 let cls=q.snr>4?"good":q.snr>1.8?"mid":"bad";

 ctx.strokeStyle=
  cls==="good"?"rgba(120,220,255,0.5)":
  cls==="mid" ?"rgba(255,200,120,0.5)":
               "rgba(255,120,120,0.5)";
 ctx.beginPath();
 ctx.moveTo(pe.x,pe.y);
 ctx.lineTo(pship.x,pship.y);
 ctx.stroke();

 // HUD
 document.getElementById("hud").innerHTML=`
🌠 Voyager – Heliopause Crossing<br>
🚀 Phase: <b>${ship.phase}</b><br>
📏 Distance: ${distAU.toFixed(1)} AU<br>
📊 SNR: <span class="${cls}">${q.snr.toFixed(2)}</span><br>
🕒 Latency: ${(q.latency/60).toFixed(2)} hours<br>
🌞 Solar Storm: ${storm.active?'<span class="warn">ACTIVE</span>':'NO'}<br>
📡 Link: <span class="${cls}">
${q.snr>1.2?'WEAK SIGNAL':'SIGNAL LOST'}
</span>
`;

 requestAnimationFrame(draw);
}

draw();
</script>
</body>
</html>
