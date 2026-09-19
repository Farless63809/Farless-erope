<!DOCTYPE html>
<html lang="tr">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1.0,maximum-scale=1.0,user-scalable=no">
<title>3D Ninja Stars</title>
<style>
*{box-sizing:border-box;-webkit-tap-highlight-color:transparent}
html,body{margin:0;width:100%;height:100%;overflow:hidden;background:#000;touch-action:none;font-family:Arial,sans-serif}
canvas{display:block}
#hud{position:fixed;top:10px;left:10px;z-index:20;color:#fff;font-size:18px;font-weight:bold;text-shadow:0 0 6px #000;pointer-events:none}
#fps{font-size:14px;margin-top:4px}
#crosshair{position:fixed;left:50%;top:50%;width:24px;height:24px;transform:translate(-50%,-50%);z-index:20;pointer-events:none}
#crosshair:before,#crosshair:after{content:"";position:absolute;background:#fff;box-shadow:0 0 5px #f00}
#crosshair:before{width:24px;height:2px;left:0;top:11px}
#crosshair:after{width:2px;height:24px;left:11px;top:0}
#yellowWarning{position:fixed;top:28%;left:50%;transform:translateX(-50%);z-index:25;display:none;color:#ffff00;font-size:24px;font-weight:900;text-shadow:0 0 5px #000,0 0 12px #ff0,0 0 25px #f90;pointer-events:none;white-space:nowrap}
#yellowArrow{position:fixed;left:50%;top:39%;width:0;height:0;z-index:25;display:none;pointer-events:none}
#yellowArrow:before{content:"";position:absolute;left:-4px;top:-12px;width:8px;height:25px;background:#ff0;border-radius:5px;box-shadow:0 0 10px #ff0}
#yellowArrow:after{content:"";position:absolute;left:-10px;top:7px;width:20px;height:20px;border-left:6px solid #ff0;border-bottom:6px solid #ff0;transform:rotate(-45deg);box-shadow:-3px 3px 8px #ff0}
#gameOver{position:fixed;inset:0;z-index:50;display:none;flex-direction:column;align-items:center;justify-content:center;background:rgba(0,0,0,.65);color:#fff;text-align:center;pointer-events:none}
#gameOver h1{margin:0 0 12px;color:#f22;font-size:48px;text-shadow:0 0 15px #f00}
#finalScore{font-size:25px;margin-bottom:12px}
#restartText{font-size:16px;opacity:.9}
#chargeBar{position:fixed;left:50%;bottom:30px;transform:translateX(-50%);width:150px;height:10px;border:2px solid rgba(255,255,255,.75);border-radius:8px;background:rgba(0,0,0,.65);z-index:30;overflow:hidden;pointer-events:none;box-shadow:0 0 6px #000}
#chargeFill{width:100%;height:100%;background:#00ff44;border-radius:5px;box-shadow:0 0 8px #00ff44;transform-origin:left center}
#hint{position:fixed;bottom:10px;left:50%;transform:translateX(-50%);z-index:10;color:rgba(255,255,255,.55);font-size:12px;pointer-events:none;white-space:nowrap}
</style>
</head>
<body>

<div id="hud">
<div id="score">Skor: 0</div>
<div id="fps">FPS: --</div>
</div>

<div id="crosshair"></div>
<div id="yellowWarning">DİKKAT SARI KAFA</div>
<div id="yellowArrow"></div>

<div id="chargeBar">
<div id="chargeFill"></div>
</div>

<div id="gameOver">
<h1>YAKALANDIN!</h1>
<div id="finalScore">Skor: 0</div>
<div id="restartText">Tekrar başlamak için ekrana dokun</div>
</div>

<div id="hint">Sürükle: Kamera • Dokun: Ninja yıldızı</div>

<script src="https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.min.js"></script>
<script>
let scene,camera,renderer,clock;
let enemies=[],stars=[];
let score=0,gameOver=false,spawnTimer=0;
let yellowActive=false;

let cameraYaw=0,cameraPitch=0;
let dragging=false,moved=false,lastX=0,lastY=0;

let fpsFrameCount=0,fpsLastUpdate=performance.now();

let starCharge=1;
const STAR_CHARGE_TIME=.5;

const PLATFORM_RADIUS=21;
const ENEMY_MIN_DISTANCE=18;
const ENEMY_MAX_DISTANCE=22;
const ENEMY_MIN_HEIGHT=1.2;
const ENEMY_MAX_HEIGHT=3.6;
const DRAG_SENSITIVITY=.004;
const STAR_SPEED=13;

const yellowWarning=document.getElementById("yellowWarning");
const yellowArrow=document.getElementById("yellowArrow");
const chargeFill=document.getElementById("chargeFill");

scene=new THREE.Scene();
scene.background=new THREE.Color(0x010101);
scene.fog=new THREE.FogExp2(0x010101,.038);

camera=new THREE.PerspectiveCamera(75,innerWidth/innerHeight,.05,100);
camera.position.set(0,1.5,0);
camera.rotation.order="YXZ";

renderer=new THREE.WebGLRenderer({antialias:true,powerPreference:"high-performance"});
renderer.setSize(innerWidth,innerHeight);
renderer.setPixelRatio(Math.min(devicePixelRatio||1,1.5));
renderer.shadowMap.enabled=true;
renderer.shadowMap.type=THREE.PCFSoftShadowMap;
document.body.appendChild(renderer.domElement);

scene.add(new THREE.AmbientLight(0x444444,1.1));

const redLight=new THREE.PointLight(0xff1800,18,38);
redLight.position.set(0,3,-5);
scene.add(redLight);

const directionalLight=new THREE.DirectionalLight(0x8899ff,2);
directionalLight.position.set(-8,14,-10);
directionalLight.castShadow=true;
directionalLight.shadow.mapSize.width=1024;
directionalLight.shadow.mapSize.height=1024;
scene.add(directionalLight);

const lavaLight=new THREE.PointLight(0xff3500,5,13);
lavaLight.position.set(0,.5,0);
scene.add(lavaLight);

function createMagmaTexture(){
const c=document.createElement("canvas");
c.width=c.height=1024;
const x=c.getContext("2d");

x.fillStyle="#130b08";
x.fillRect(0,0,1024,1024);

const rocks=["#211512","#291b15","#321d15","#18100d","#3a2117","#24130f"];

for(let i=0;i<900;i++){
const px=Math.random()*1024,py=Math.random()*1024,s=8+Math.random()*38;

x.fillStyle=rocks[Math.floor(Math.random()*rocks.length)];
x.beginPath();

for(let j=0;j<6;j++){
const a=Math.PI*2*j/6;
const r=s*(.55+Math.random()*.45);
const xx=px+Math.cos(a)*r;
const yy=py+Math.sin(a)*r;

if(j===0)x.moveTo(xx,yy);
else x.lineTo(xx,yy);
}

x.closePath();
x.fill();
}

for(let i=0;i<120;i++){
const px=Math.random()*1024,py=Math.random()*1024;
const rx=15+Math.random()*50;
const ry=8+Math.random()*28;

const g=x.createRadialGradient(px,py,0,px,py,Math.max(rx,ry));

g.addColorStop(0,"rgba(255,245,100,1)");
g.addColorStop(.25,"rgba(255,145,0,.98)");
g.addColorStop(.65,"rgba(230,45,0,.85)");
g.addColorStop(1,"rgba(60,0,0,0)");

x.fillStyle=g;
x.beginPath();
x.ellipse(px,py,rx,ry,Math.random()*Math.PI,0,Math.PI*2);
x.fill();
}

for(let i=0;i<22;i++){
const px=Math.random()*1024,py=Math.random()*1024;
const rx=25+Math.random()*70;
const ry=15+Math.random()*45;

const g=x.createRadialGradient(px,py,0,px,py,Math.max(rx,ry));

g.addColorStop(0,"#fff06a");
g.addColorStop(.2,"#ff9d00");
g.addColorStop(.55,"#ff3b00");
g.addColorStop(.8,"#b71900");
g.addColorStop(1,"rgba(25,0,0,0)");

x.fillStyle=g;
x.beginPath();
x.ellipse(px,py,rx,ry,Math.random()*Math.PI,0,Math.PI*2);
x.fill();
}

for(let i=0;i<260;i++){
const px=Math.random()*1024,py=Math.random()*1024,s=5+Math.random()*18;

x.fillStyle=Math.random()>.5?"#0d0908":"#1a0f0b";

x.beginPath();
x.moveTo(px,py-s);
x.lineTo(px+s,py+s*.4);
x.lineTo(px-s*.7,py+s);
x.closePath();
x.fill();
}

const t=new THREE.CanvasTexture(c);
t.colorSpace=THREE.SRGBColorSpace;
t.anisotropy=renderer.capabilities.getMaxAnisotropy();

return t;
}

const magmaTexture=createMagmaTexture();

const platform=new THREE.Mesh(
new THREE.CircleGeometry(PLATFORM_RADIUS,96),
new THREE.MeshStandardMaterial({
map:magmaTexture,
roughness:.72,
metalness:.05
})
);

platform.rotation.x=-Math.PI/2;
platform.receiveShadow=true;
scene.add(platform);

const platformBase=new THREE.Mesh(
new THREE.CylinderGeometry(PLATFORM_RADIUS+.4,PLATFORM_RADIUS+1.2,1.5,96),
new THREE.MeshStandardMaterial({
color:0x090706,
roughness:1
})
);

platformBase.position.y=-.8;
platformBase.receiveShadow=true;
scene.add(platformBase);

const lavaRing=new THREE.Mesh(
new THREE.RingGeometry(PLATFORM_RADIUS-.08,PLATFORM_RADIUS+.12,96),
new THREE.MeshBasicMaterial({
color:0xff2600,
transparent:true,
opacity:.9
})
);

lavaRing.rotation.x=-Math.PI/2;
lavaRing.position.y=.025;
scene.add(lavaRing);

for(let i=0;i<26;i++){
const a=Math.random()*Math.PI*2;
const d=2.5+Math.random()*17;

const rock=new THREE.Mesh(
new THREE.DodecahedronGeometry(.18+Math.random()*.35,0),
new THREE.MeshStandardMaterial({
color:0x17100d,
roughness:1
})
);

rock.position.set(
Math.sin(a)*d,
.12+Math.random()*.22,
Math.cos(a)*d
);

rock.rotation.set(
Math.random()*3,
Math.random()*3,
Math.random()*3
);

rock.castShadow=true;
rock.receiveShadow=true;

scene.add(rock);
}

const skullMaterial=new THREE.MeshStandardMaterial({
color:0xd7d7d7,
roughness:.72,
metalness:.05
});

const jawMaterial=new THREE.MeshStandardMaterial({
color:0xc9c9c9,
roughness:.75
});

const redEyeMaterial=new THREE.MeshBasicMaterial({color:0xff0000});
const blueEyeMaterial=new THREE.MeshBasicMaterial({color:0x0088ff});
const greenEyeMaterial=new THREE.MeshBasicMaterial({color:0x00ff22});
const yellowEyeMaterial=new THREE.MeshBasicMaterial({color:0xffff00});
const blackMaterial=new THREE.MeshBasicMaterial({color:0x000000});

const specialGrayMaterial=new THREE.MeshStandardMaterial({
color:0x9b9b9b,
roughness:.5,
metalness:.55
});

const specialDarkMaterial=new THREE.MeshStandardMaterial({
color:0x555555,
roughness:.55,
metalness:.7
});

const specialBlackMaterial=new THREE.MeshStandardMaterial({
color:0x111111,
roughness:.4,
metalness:.5
});

const specialYellowMaterial=new THREE.MeshBasicMaterial({
color:0xffff00
});

function createSpecialYellowHead(enemy){

const group=new THREE.Group();

const head=new THREE.Mesh(
new THREE.IcosahedronGeometry(.92,1),
specialGrayMaterial
);

head.scale.set(1.08,1.28,.92);
head.castShadow=true;
group.add(head);

const jaw=new THREE.Mesh(
new THREE.BoxGeometry(1.05,.34,.70),
jawMaterial
);

jaw.position.y=-.62;
jaw.castShadow=true;
group.add(jaw);

const jawFront=new THREE.Mesh(
new THREE.OctahedronGeometry(.34,0),
specialGrayMaterial
);

jawFront.scale.set(1.35,.7,.45);
jawFront.position.set(0,-.61,-.39);
group.add(jawFront);

const sideLeft=new THREE.Mesh(
new THREE.OctahedronGeometry(.32,0),
specialDarkMaterial
);

sideLeft.position.set(-.82,-.08,0);
sideLeft.rotation.z=.35;
group.add(sideLeft);

const sideRight=sideLeft.clone();
sideRight.position.x=.82;
sideRight.rotation.z=-.35;
group.add(sideRight);

const browLeft=new THREE.Mesh(
new THREE.BoxGeometry(.42,.17,.22),
specialDarkMaterial
);

browLeft.position.set(-.31,.30,-.76);
browLeft.rotation.z=-.15;
group.add(browLeft);

const browRight=browLeft.clone();
browRight.position.x=.31;
browRight.rotation.z=.15;
group.add(browRight);

const forehead=new THREE.Mesh(
new THREE.ConeGeometry(.26,.52,6),
specialGrayMaterial
);

forehead.position.set(0,.63,-.08);
group.add(forehead);

const eyeSocketLeft=new THREE.Mesh(
new THREE.BoxGeometry(.38,.27,.08),
specialBlackMaterial
);

eyeSocketLeft.position.set(-.31,.12,-.81);
group.add(eyeSocketLeft);

const eyeSocketRight=eyeSocketLeft.clone();
eyeSocketRight.position.x=.31;
group.add(eyeSocketRight);

const eyeLeft=new THREE.Mesh(
new THREE.OctahedronGeometry(.105,0),
specialYellowMaterial
);

eyeLeft.position.set(-.31,.12,-.86);
group.add(eyeLeft);

const eyeRight=eyeLeft.clone();
eyeRight.position.x=.31;
group.add(eyeRight);

const nose=new THREE.Mesh(
new THREE.ConeGeometry(.19,.42,4),
specialDarkMaterial
);

nose.position.set(0,-.10,-.79);
nose.rotation.x=Math.PI/2;
group.add(nose);

const mouth=new THREE.Mesh(
new THREE.BoxGeometry(.58,.16,.08),
specialBlackMaterial
);

mouth.position.set(0,-.40,-.78);
group.add(mouth);

const mouthArmor=new THREE.Mesh(
new THREE.BoxGeometry(.72,.19,.13),
specialGrayMaterial
);

mouthArmor.position.set(0,-.49,-.74);
group.add(mouthArmor);

const cheekLeft=new THREE.Mesh(
new THREE.ConeGeometry(.24,.46,6),
specialDarkMaterial
);

cheekLeft.position.set(-.65,-.34,-.43);
cheekLeft.rotation.z=Math.PI/2;
group.add(cheekLeft);

const cheekRight=cheekLeft.clone();
cheekRight.position.x=.65;
cheekRight.rotation.z=-Math.PI/2;
group.add(cheekRight);

const core=new THREE.Mesh(
new THREE.OctahedronGeometry(.20,0),
specialYellowMaterial
);

core.position.set(0,-.75,-.02);
group.add(core);

const shoulderLeft=new THREE.Mesh(
new THREE.OctahedronGeometry(.38,0),
specialGrayMaterial
);

shoulderLeft.position.set(-.92,-.60,0);
group.add(shoulderLeft);

const shoulderRight=shoulderLeft.clone();
shoulderRight.position.x=.92;
group.add(shoulderRight);

const crownBase=new THREE.Mesh(
new THREE.CylinderGeometry(.48,.56,.22,8),
specialGrayMaterial
);

crownBase.position.y=1.02;
group.add(crownBase);

const crownRing=new THREE.Mesh(
new THREE.TorusGeometry(.48,.055,8,16),
specialYellowMaterial
);

crownRing.rotation.x=Math.PI/2;
crownRing.position.y=1.12;
group.add(crownRing);

for(let i=0;i<5;i++){
const spike=new THREE.Mesh(
new THREE.ConeGeometry(.11,.38,5),
specialGrayMaterial
);

spike.position.set((i-2)*.21,1.28,0);
group.add(spike);
}

const hornLeft=new THREE.Mesh(
new THREE.ConeGeometry(.18,.58,6),
specialGrayMaterial
);

hornLeft.position.set(-.73,.82,.04);
hornLeft.rotation.z=-.55;
group.add(hornLeft);

const hornRight=hornLeft.clone();
hornRight.position.x=.73;
hornRight.rotation.z=.55;
group.add(hornRight);

const faceArmor=new THREE.Mesh(
new THREE.OctahedronGeometry(1.02,0),
new THREE.MeshBasicMaterial({
color:0xffff00,
transparent:true,
opacity:.055,
side:THREE.DoubleSide
})
);

faceArmor.scale.set(1.03,1.18,.9);
group.add(faceArmor);

const aura=new THREE.PointLight(0xffff00,5,8);
aura.position.set(0,0,-.2);
group.add(aura);

const guitar=new THREE.Group();

const guitarBody=new THREE.Mesh(
new THREE.SphereGeometry(.27,12,8),
new THREE.MeshStandardMaterial({
color:0x8b4513,
roughness:.5,
metalness:.2
})
);

guitarBody.scale.set(1,1.25,.35);
guitar.add(guitarBody);

const guitarNeck=new THREE.Mesh(
new THREE.BoxGeometry(.10,.65,.10),
new THREE.MeshStandardMaterial({
color:0x5a3219,
roughness:.5
})
);

guitarNeck.position.y=.42;
guitar.add(guitarNeck);

for(let i=0;i<3;i++){
const string=new THREE.Mesh(
new THREE.CylinderGeometry(.008,.008,.72,6),
new THREE.MeshBasicMaterial({color:0xffffff})
);

string.position.set((i-1)*.025,.43,-.055);
guitar.add(string);
}

guitar.position.set(-.72,-.28,-.65);
guitar.rotation.z=.25;
guitar.visible=false;
group.add(guitar);

const shield=new THREE.Group();

const shieldMain=new THREE.Mesh(
new THREE.CylinderGeometry(.38,.38,.12,8),
new THREE.MeshStandardMaterial({
color:0x777777,
metalness:.85,
roughness:.22
})
);

shieldMain.rotation.x=Math.PI/2;
shield.add(shieldMain);

const shieldCenter=new THREE.Mesh(
new THREE.OctahedronGeometry(.13,0),
specialYellowMaterial
);

shieldCenter.position.z=.08;
shield.add(shieldCenter);

shield.position.set(.78,-.28,-.55);
shield.rotation.y=.25;
shield.visible=false;
group.add(shield);

enemy.add(group);

return {group,aura,guitar,shield,core};
}

function createEnemy(type,customPosition=null,customScale=1){

if(type==="specialYellow")
return createSpecialEnemy(customPosition);

const enemy=new THREE.Group();

const isBlue=type==="blue";
const isGreen=type==="green";
const isYellow=type==="yellowMinion";
const isSmall=type==="smallYellow";

const skull=new THREE.Mesh(
new THREE.SphereGeometry(.62,20,16),
skullMaterial
);

skull.scale.y=1.08;
skull.castShadow=true;
enemy.add(skull);

const jaw=new THREE.Mesh(
new THREE.BoxGeometry(.65,.30,.45),
jawMaterial
);

jaw.position.y=-.48;
enemy.add(jaw);

const eyeMaterial=
isGreen?greenEyeMaterial:
isBlue?blueEyeMaterial:
(isYellow||isSmall)?yellowEyeMaterial:
redEyeMaterial;

const leftEye=new THREE.Mesh(
new THREE.SphereGeometry(.105,12,12),
eyeMaterial
);

const rightEye=leftEye.clone();

leftEye.position.set(-.22,.12,-.55);
rightEye.position.set(.22,.12,-.55);

enemy.add(leftEye,rightEye);

const nose=new THREE.Mesh(
new THREE.ConeGeometry(.09,.25,4),
jawMaterial
);

nose.rotation.x=Math.PI/2;
nose.position.set(0,-.04,-.58);
enemy.add(nose);

const mouth=new THREE.Mesh(
new THREE.BoxGeometry(.42,.08,.03),
blackMaterial
);

mouth.position.set(0,-.27,-.56);
enemy.add(mouth);

let glasses=null;

if(isYellow){

glasses=new THREE.Group();

const glassesFrameMaterial=new THREE.MeshBasicMaterial({
color:0x111111
});

const glassesLensMaterial=new THREE.MeshBasicMaterial({
color:0x000000
});

const leftLens=new THREE.Mesh(
new THREE.BoxGeometry(.27,.15,.05),
glassesLensMaterial
);

const rightLens=leftLens.clone();

leftLens.position.set(-.22,.12,-.60);
rightLens.position.set(.22,.12,-.60);

const bridge=new THREE.Mesh(
new THREE.BoxGeometry(.13,.035,.05),
glassesFrameMaterial
);

bridge.position.set(0,.12,-.60);

const leftArm=new THREE.Mesh(
new THREE.BoxGeometry(.20,.035,.035),
glassesFrameMaterial
);

const rightArm=leftArm.clone();

leftArm.position.set(-.39,.12,-.58);
rightArm.position.set(.39,.12,-.58);

glasses.add(
leftLens,
rightLens,
bridge,
leftArm,
rightArm
);

enemy.add(glasses);
}

const auraColor=
isGreen?0x00ff22:
isBlue?0x0088ff:
(isYellow||isSmall)?0xffff00:
0xff0000;

const aura=new THREE.PointLight(
auraColor,
isSmall?2.2:isYellow?3:isGreen?2.8:isBlue?3:2.5,
5
);

aura.position.z=-.15;
enemy.add(aura);

const angle=Math.random()*Math.PI*2;
const distance=
ENEMY_MIN_DISTANCE+
Math.random()*(ENEMY_MAX_DISTANCE-ENEMY_MIN_DISTANCE);

const spawnY=
ENEMY_MIN_HEIGHT+
Math.random()*(ENEMY_MAX_HEIGHT-ENEMY_MIN_HEIGHT);

if(customPosition){

enemy.position.copy(customPosition);

}else{

enemy.position.set(
camera.position.x+Math.sin(angle)*distance,
spawnY,
camera.position.z+Math.cos(angle)*distance
);

}

enemy.scale.setScalar(customScale);

let speed;

if(isSmall)
speed=1.50;
else if(isYellow)
speed=3.40;
else if(isGreen)
speed=1.50;
else if(isBlue)
speed=3.15+Math.random()*.05;
else
speed=1.85+Math.random()*.15;

enemy.userData={
alive:true,
type:type,
isSpecialYellow:false,
isGreen:isGreen,
isYellowMinion:isYellow,
isSmallYellow:isSmall,
speed:speed,
baseY:customPosition?enemy.position.y:spawnY,
phase:Math.random()*Math.PI*2,
sway:.8+Math.random()*.4,
bob:.8+Math.random()*.35,
highSpawn:!customPosition&&spawnY>2.4,
diving:false,
teleportTimer:0,
teleportStage:0,
moveDirection:null,
skull:skull,
jaw:jaw,
aura:aura,
glasses:glasses
};

scene.add(enemy);
enemies.push(enemy);

return enemy;
}
function createSpecialEnemy(customPosition=null){

const enemy=new THREE.Group();
const special=createSpecialYellowHead(enemy);

const angle=Math.random()*Math.PI*2;

const distance=
customPosition?
15:
ENEMY_MIN_DISTANCE+
Math.random()*(ENEMY_MAX_DISTANCE-ENEMY_MIN_DISTANCE);

const spawnY=
ENEMY_MIN_HEIGHT+
Math.random()*(ENEMY_MAX_HEIGHT-ENEMY_MIN_HEIGHT);

if(customPosition){

enemy.position.copy(customPosition);

}else{

enemy.position.set(
camera.position.x+Math.sin(angle)*distance,
spawnY,
camera.position.z+Math.cos(angle)*distance
);

}

enemy.userData={
alive:true,
type:"specialYellow",
isSpecialYellow:true,
speed:1.35,
baseY:enemy.position.y,
phase:Math.random()*Math.PI*2,
sway:.8,
bob:.8,
highSpawn:!customPosition&&spawnY>2.4,
diving:false,
aura:special.aura,
guitar:special.guitar,
shield:special.shield,
core:special.core,
hitCount:0,
evolvedTimer:3+Math.random()*2,
guitarTimer:0,
guitarPlaying:false,
guitarUsed:false
};

scene.add(enemy);
enemies.push(enemy);

yellowActive=true;
showYellowWarning();

return enemy;
}

function createYellowMinions(position){

for(let i=0;i<3;i++){

const offset=new THREE.Vector3(
(Math.random()-.5)*.9,
(Math.random()-.5)*.6,
(Math.random()-.5)*.9
);

const spawnPosition=position.clone().add(offset);

createEnemy("yellowMinion",spawnPosition,1);
}
}

function createSmallYellowHeads(origin,direction){

const forward=direction.clone().normalize();

const side=new THREE.Vector3(0,1,0)
.cross(forward)
.normalize();

const distance=20;
const gap=.9;

for(let i=0;i<2;i++){

const offset=i===0?-gap/2:gap/2;

const position=origin.clone()
.addScaledVector(forward,distance)
.addScaledVector(side,offset);

createEnemy(
"smallYellow",
position,
.55
);

const enemy=enemies[enemies.length-1];

enemy.userData.moveDirection=forward.clone();
}
}

function createNinjaStar(){

const shape=new THREE.Shape();
const spikes=8;
const outerRadius=.27;
const innerRadius=.08;

for(let i=0;i<spikes*2;i++){

const angle=-Math.PI/2+i*Math.PI/spikes;
const radius=i%2===0?outerRadius:innerRadius;

const x=Math.cos(angle)*radius;
const y=Math.sin(angle)*radius;

if(i===0)
shape.moveTo(x,y);
else
shape.lineTo(x,y);
}

shape.closePath();

const geometry=new THREE.ExtrudeGeometry(shape,{
depth:.055,
bevelEnabled:true,
bevelSegments:1,
bevelSize:.015,
bevelThickness:.015
});

geometry.center();

const star=new THREE.Mesh(
geometry,
new THREE.MeshStandardMaterial({
color:0x555b61,
metalness:.9,
roughness:.25
})
);

const center=new THREE.Mesh(
new THREE.CylinderGeometry(.075,.075,.07,16),
new THREE.MeshStandardMaterial({
color:0x222222,
metalness:.9,
roughness:.2
})
);

center.rotation.x=Math.PI/2;
star.add(center);

const colors=[
0xff0000,
0x0088ff,
0x00ff22,
0xffff00
];

for(let i=0;i<4;i++){

const ring=new THREE.Mesh(
new THREE.TorusGeometry(.09,.018,8,16,Math.PI*2/4),
new THREE.MeshBasicMaterial({color:colors[i]})
);

ring.rotation.x=Math.PI/2;
ring.rotation.z=i*Math.PI/2;

star.add(ring);
}

return star;
}

function shootStar(){

if(gameOver)return;

if(starCharge<1)return;

starCharge=0;
updateChargeBar();

const star=createNinjaStar();

const direction=new THREE.Vector3(0,0,-1);
direction.applyQuaternion(camera.quaternion);

star.position.copy(camera.position);
star.position.addScaledVector(direction,.7);

star.userData={
direction:direction.clone()
};

scene.add(star);
stars.push(star);
}

function updateCharge(delta){

if(gameOver)return;

if(starCharge<1){

starCharge+=delta/STAR_CHARGE_TIME;

if(starCharge>1)
starCharge=1;

updateChargeBar();
}
}

function updateChargeBar(){

chargeFill.style.transform=
"scaleX("+starCharge+")";
}

const particleGeometry=new THREE.SphereGeometry(.035,6,6);

const redParticleMaterials=[
new THREE.MeshBasicMaterial({color:0xff0000}),
new THREE.MeshBasicMaterial({color:0xff2200}),
new THREE.MeshBasicMaterial({color:0xcc0000})
];

const blueParticleMaterials=[
new THREE.MeshBasicMaterial({color:0x0088ff}),
new THREE.MeshBasicMaterial({color:0x00bfff}),
new THREE.MeshBasicMaterial({color:0x66ddff})
];

const greenParticleMaterials=[
new THREE.MeshBasicMaterial({color:0x00ff22}),
new THREE.MeshBasicMaterial({color:0x00ff88}),
new THREE.MeshBasicMaterial({color:0x66ff99})
];

const yellowParticleMaterials=[
new THREE.MeshBasicMaterial({color:0xffff00}),
new THREE.MeshBasicMaterial({color:0xffd500}),
new THREE.MeshBasicMaterial({color:0xffa800}),
new THREE.MeshBasicMaterial({color:0xffffff})
];

const particlePool=[];

for(let i=0;i<180;i++){

const p=new THREE.Mesh(
particleGeometry,
redParticleMaterials[i%redParticleMaterials.length]
);

p.visible=false;

p.userData={
velocity:new THREE.Vector3(),
life:0
};

scene.add(p);
particlePool.push(p);
}

function createExplosion(position,type="red",amount=16){

let created=0;

for(let i=0;i<particlePool.length&&created<amount;i++){

const p=particlePool[i];

if(p.visible)continue;

p.visible=true;
p.position.copy(position);
p.scale.setScalar(.8+Math.random()*1.5);

p.userData.velocity.set(
(Math.random()-.5)*4,
(Math.random()-.5)*4,
(Math.random()-.5)*4
);

p.userData.life=.45+Math.random()*.35;

const materials=
type==="blue"?blueParticleMaterials:
type==="green"?greenParticleMaterials:
type==="yellow"?yellowParticleMaterials:
redParticleMaterials;

p.material=materials[
Math.floor(Math.random()*materials.length)
];

created++;
}
}

function updateParticles(delta){

for(let i=0;i<particlePool.length;i++){

const p=particlePool[i];

if(!p.visible)continue;

p.userData.life-=delta;

if(p.userData.life<=0){

p.visible=false;
continue;

}

p.position.addScaledVector(
p.userData.velocity,
delta
);

p.userData.velocity.y-=5*delta;
p.scale.multiplyScalar(.97);
}
}

function updateSpecialYellow(enemy,delta){

const data=enemy.userData;

data.phase+=delta*3;

const direction=new THREE.Vector3()
.subVectors(camera.position,enemy.position)
.normalize();

enemy.position.addScaledVector(
direction,
data.speed*delta
);

enemy.position.y=
data.baseY+
Math.sin(data.phase*data.bob)*.12;

enemy.rotation.y=
Math.atan2(-direction.x,-direction.z);

enemy.rotation.z=
Math.sin(data.phase*1.5)*.04;

const pulse=
1+
Math.sin(data.phase*2.2)*.25;

data.aura.intensity=5*pulse;

if(
!data.guitarUsed&&
!data.guitarPlaying&&
data.evolvedTimer<=0
){

data.guitarUsed=true;
data.guitarPlaying=true;
data.guitarTimer=3;

if(data.guitar)
data.guitar.visible=true;

if(data.shield)
data.shield.visible=true;

data.aura.intensity=8;
}

if(data.evolvedTimer>0)
data.evolvedTimer-=delta;

if(data.guitarPlaying){

data.guitarTimer-=delta;

if(data.guitar){

data.guitar.rotation.y+=delta*2;
data.guitar.rotation.x=
Math.sin(data.phase*8)*.12;
}

if(data.guitarTimer<=0){

data.guitarPlaying=false;

if(data.guitar)
data.guitar.visible=false;

if(data.shield)
data.shield.visible=false;

data.aura.intensity=5;
}
}

if(
enemy.position.distanceToSquared(camera.position)
<
1.05*1.05
)
loseGame();
}

function updateEnemies(delta){

for(let i=enemies.length-1;i>=0;i--){

const enemy=enemies[i];

if(!enemy.userData.alive)continue;

const data=enemy.userData;

if(data.isSpecialYellow){

updateSpecialYellow(enemy,delta);
continue;
}

if(data.isGreen)
data.teleportTimer+=delta;

const direction=new THREE.Vector3()
.subVectors(camera.position,enemy.position)
.normalize();

const moveDirection=
data.isSmallYellow&&data.moveDirection
?data.moveDirection
:direction;

enemy.position.addScaledVector(
moveDirection,
data.speed*delta
);

data.phase+=delta*4;

if(!data.diving){

enemy.position.y=
data.baseY+
Math.sin(data.phase*data.bob)*.16;

}else{

enemy.position.y=
THREE.MathUtils.lerp(
enemy.position.y,
camera.position.y+.15,
Math.min(1,delta*2.8)
);

}

const horizontalDistance=Math.sqrt(
Math.pow(
camera.position.x-enemy.position.x,
2
)+
Math.pow(
camera.position.z-enemy.position.z,
2
)
);

if(
data.highSpawn&&
horizontalDistance<6.5
)
data.diving=true;

enemy.rotation.y=
Math.atan2(-moveDirection.x,-moveDirection.z);

enemy.rotation.z=
Math.sin(data.phase*1.3)*.035;

const pulse=
1+
Math.sin(data.phase*2.2)*.22;

data.aura.intensity=
pulse*
(
data.type==="green"?2.8:
data.type==="blue"?3:
(data.isYellowMinion||data.isSmallYellow)?2.2:
2.5
);

if(data.isGreen){

if(
data.teleportStage===0&&
data.teleportTimer>=2
){

teleportGreenEnemy(enemy);

}else if(
data.teleportStage===1&&
data.teleportTimer>=2
){

teleportGreenEnemy(enemy);

}
}

if(data.isYellowMinion){

if(
enemy.position.distanceToSquared(camera.position)
<
1.05*1.05
){

data.alive=false;

createExplosion(
enemy.position,
"yellow",
20
);

const direction=new THREE.Vector3()
.subVectors(camera.position,enemy.position)
.normalize();

createSmallYellowHeads(
enemy.position,
direction
);

scene.remove(enemy);
enemies.splice(i,1);

continue;
}
}

if(data.isSmallYellow){

if(
enemy.position.distanceToSquared(camera.position)
<
1.05*1.05
){

loseGame();
return;

}
}

if(
!data.isYellowMinion &&
!data.isSmallYellow &&
enemy.position.distanceToSquared(camera.position)
<
1.05*1.05
){

loseGame();
return;
}
}
}

/* YEŞİL KAFA IŞINLANMA SİSTEMİ */
function teleportGreenEnemy(enemy){

const data=enemy.userData;

if(data.teleportStage===undefined)
data.teleportStage=0;

createExplosion(
enemy.position,
"green",
18
);

if(data.teleportStage===0){

const forward=new THREE.Vector3(0,0,-1);
forward.applyQuaternion(camera.quaternion);

enemy.position.set(
camera.position.x+forward.x*8,
data.baseY,
camera.position.z+forward.z*8
);

data.teleportStage=1;
data.teleportTimer=0;

}else{

const angle=Math.random()*Math.PI*2;

enemy.position.set(
camera.position.x+Math.sin(angle)*7,
data.baseY,
camera.position.z+Math.cos(angle)*7
);

data.teleportStage=2;
data.teleportTimer=0;
}

createExplosion(
enemy.position,
"green",
18
);
}

function loseGame(){

if(gameOver)return;

gameOver=true;

yellowWarning.style.display="none";
yellowArrow.style.display="none";

document.getElementById("finalScore").textContent=
"Skor: "+score;

document.getElementById("gameOver").style.display="flex";
}

function addScore(amount){

score+=amount;

document.getElementById("score").textContent=
"Skor: "+score;
}
function updateYellowActive(){

let found=false;

for(let i=0;i<enemies.length;i++){

if(
enemies[i].userData.alive&&
enemies[i].userData.isSpecialYellow
){

found=true;
break;

}
}

yellowActive=found;

if(!found){

yellowWarning.style.display="none";
yellowArrow.style.display="none";

}
}

function spawnRandomEnemy(){

if(yellowActive)return;

if(score>=100){

const angle=Math.random()*Math.PI*2;

const spawnPosition=new THREE.Vector3(
camera.position.x+Math.sin(angle)*15,
ENEMY_MIN_HEIGHT+
Math.random()*(ENEMY_MAX_HEIGHT-ENEMY_MIN_HEIGHT),
camera.position.z+Math.cos(angle)*15
);

const r=Math.random();

if(r<.60){

createEnemy(
"blue",
spawnPosition
);

}else if(r<.90){

createEnemy(
"green",
spawnPosition
);

}else{

createSpecialEnemy(
spawnPosition
);

}

return;
}

const r=Math.random();

if(score<60){

if(r<.79)
createEnemy("red");

else if(r<.94)
createEnemy("blue");

else if(r<.99)
createEnemy("green");

else
createSpecialEnemy();

}else{

if(r<.49)
createEnemy("red");

else if(r<.82)
createEnemy("blue");

else if(r<.97)
createEnemy("green");

else
createSpecialEnemy();

}
}

function updateStars(delta){

for(let i=stars.length-1;i>=0;i--){

const star=stars[i];

star.position.addScaledVector(
star.userData.direction,
STAR_SPEED*delta
);

star.rotation.x+=delta*13;
star.rotation.y+=delta*9;
star.rotation.z+=delta*7;

let destroyed=false;

for(let j=enemies.length-1;j>=0;j--){

const enemy=enemies[j];

if(!enemy.userData.alive)continue;

const hitRadius=
enemy.userData.isSpecialYellow?1.35:
enemy.userData.isSmallYellow?.55:
enemy.userData.isYellowMinion?.85:
.85;

if(
star.position.distanceToSquared(enemy.position)
<
hitRadius*hitRadius
){

if(enemy.userData.isSpecialYellow){

if(enemy.userData.guitarPlaying){

scene.remove(star);
stars.splice(i,1);

destroyed=true;
break;
}

enemy.userData.hitCount++;

createExplosion(
star.position,
"yellow",
10
);

scene.remove(star);
stars.splice(i,1);

destroyed=true;

if(enemy.userData.hitCount>=5){

const explosionPosition=
enemy.position.clone();

enemy.userData.alive=false;

createExplosion(
explosionPosition,
"yellow",
55
);

createYellowMinions(
explosionPosition
);

scene.remove(enemy);
enemies.splice(j,1);

yellowActive=false;

yellowWarning.style.display="none";
yellowArrow.style.display="none";

addScore(5);
}

break;
}

if(enemy.userData.isSmallYellow){

enemy.userData.alive=false;

createExplosion(
enemy.position,
"yellow",
16
);

scene.remove(enemy);
enemies.splice(j,1);

addScore(1);

scene.remove(star);
stars.splice(i,1);

destroyed=true;
break;
}

if(enemy.userData.isYellowMinion){

enemy.userData.alive=false;

createExplosion(
enemy.position,
"yellow",
18
);

scene.remove(enemy);
enemies.splice(j,1);

addScore(1);

scene.remove(star);
stars.splice(i,1);

destroyed=true;
break;
}

enemy.userData.alive=false;

createExplosion(
enemy.position,
enemy.userData.type,
16
);

scene.remove(enemy);
enemies.splice(j,1);

addScore(1);

scene.remove(star);
stars.splice(i,1);

destroyed=true;
break;
}
}

if(destroyed)continue;

const distanceFromCenter=Math.sqrt(
star.position.x*star.position.x+
star.position.z*star.position.z
);

if(
distanceFromCenter>
PLATFORM_RADIUS+1.2
){

scene.remove(star);
stars.splice(i,1);

}
}
}

function showYellowWarning(){

yellowWarning.style.display="block";
yellowArrow.style.display="block";

}

function updateYellowWarning(){

if(!yellowActive){

yellowWarning.style.display="none";
yellowArrow.style.display="none";

return;
}

let target=null;

for(let i=0;i<enemies.length;i++){

if(
enemies[i].userData.alive&&
enemies[i].userData.isSpecialYellow
){

target=enemies[i];
break;

}
}

if(!target){

updateYellowActive();
return;

}

yellowWarning.style.display="block";
yellowArrow.style.display="block";

const dx=
target.position.x-camera.position.x;

const dz=
target.position.z-camera.position.z;

const angle=Math.atan2(dx,dz);

let relativeAngle=
angle-camera.rotation.y;

while(relativeAngle>Math.PI)
relativeAngle-=Math.PI*2;

while(relativeAngle<-Math.PI)
relativeAngle+=Math.PI*2;

yellowArrow.style.transform=
"translate(-50%,-50%) rotate("+
relativeAngle+
"rad)";
}

function restartGame(){

for(let i=enemies.length-1;i>=0;i--)
scene.remove(enemies[i]);

enemies.length=0;

for(let i=stars.length-1;i>=0;i--)
scene.remove(stars[i]);

stars.length=0;

score=0;
spawnTimer=0;
yellowActive=false;
gameOver=false;

starCharge=1;
updateChargeBar();

yellowWarning.style.display="none";
yellowArrow.style.display="none";

cameraYaw=0;
cameraPitch=0;

camera.rotation.order="YXZ";
camera.rotation.x=0;
camera.rotation.y=0;
camera.rotation.z=0;

document.getElementById("score").textContent="Skor: 0";
document.getElementById("gameOver").style.display="none";

spawnRandomEnemy();
}

renderer.domElement.addEventListener(
"pointerdown",
e=>{

dragging=true;
moved=false;
lastX=e.clientX;
lastY=e.clientY;

});

renderer.domElement.addEventListener(
"pointermove",
e=>{

if(!dragging)return;

const dx=e.clientX-lastX;
const dy=e.clientY-lastY;

if(Math.abs(dx)>2||Math.abs(dy)>2)
moved=true;

cameraYaw-=dx*DRAG_SENSITIVITY;
cameraPitch-=dy*DRAG_SENSITIVITY;

cameraPitch=
Math.max(
-1.25,
Math.min(1.25,cameraPitch)
);

camera.rotation.order="YXZ";
camera.rotation.y=cameraYaw;
camera.rotation.x=cameraPitch;
camera.rotation.z=0;

lastX=e.clientX;
lastY=e.clientY;

});

renderer.domElement.addEventListener(
"pointerup",
()=>{

if(!dragging)return;

dragging=false;

if(gameOver){

if(!moved)
restartGame();

return;
}

if(!moved)
shootStar();

});

renderer.domElement.addEventListener(
"pointercancel",
()=>{
dragging=false;
}
);

window.addEventListener(
"resize",
()=>{

camera.aspect=
innerWidth/innerHeight;

camera.updateProjectionMatrix();

renderer.setSize(
innerWidth,
innerHeight
);

}
);

clock=new THREE.Clock();

let lavaTime=0;

function animate(now){

requestAnimationFrame(animate);

const delta=
Math.min(clock.getDelta(),.05);

fpsFrameCount++;

if(now-fpsLastUpdate>=1000){

document.getElementById("fps").textContent=
"FPS: "+fpsFrameCount;

fpsFrameCount=0;
fpsLastUpdate=now;

}

if(!gameOver){

updateCharge(delta);

spawnTimer+=delta;

const spawnInterval=
score>=100?2:3;

if(spawnTimer>=spawnInterval){

spawnTimer=0;

if(!yellowActive)
spawnRandomEnemy();

}

updateEnemies(delta);
updateStars(delta);
updateParticles(delta);
updateYellowActive();
updateYellowWarning();

lavaTime+=delta;

lavaLight.intensity=
5+Math.sin(lavaTime*3)*1.2;

lavaRing.material.opacity=
.65+Math.sin(lavaTime*4)*.2;

}

renderer.render(scene,camera);
}

updateChargeBar();

spawnRandomEnemy();
requestAnimationFrame(animate);

</script>
</body>
</html>
