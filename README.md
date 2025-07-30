<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no"/>
<title>Fortnite Mini Clone</title>
<style>
  body { margin: 0; overflow: hidden; background: #000; }
  canvas { display: block; }
  #ui { position: fixed; top: 10px; left: 50%; transform: translateX(-50%); color: white; font-family: Arial, sans-serif; font-size: 20px; text-align: center; z-index: 10;}
  #health {margin-bottom: 5px;}
  #crosshair { position: fixed; top: 50%; left: 50%; width: 20px; height: 20px; margin-left: -10px; margin-top: -10px; border: 2px solid white; border-radius: 50%; pointer-events: none; }
  #shootBtn, #joystick {position: fixed; z-index: 20;}
  #shootBtn { bottom: 20px; right: 20px; width: 80px; height: 80px; background: rgba(255,0,0,0.5); border-radius: 50%; display:none; }
  #joystick { bottom: 20px; left: 20px; width: 120px; height: 120px; background: rgba(255,255,255,0.1); border-radius: 50%; display:none; }
  #miniMap {position:fixed; top:10px; right:10px; width:150px; height:150px; border-radius:50%; background:rgba(0,0,0,0.5); z-index:15;}
  #miniCanvas {width:100%; height:100%;}
  #rotateWarn {position:fixed; top:0; left:0; width:100%; height:100%; background:#000; color:#fff; font-size:2em; display:none; justify-content:center; align-items:center; z-index:999;}
</style>
</head>
<body>
<div id="ui">
  <div id="health">Health: 100</div>
  <div id="ammo">Ammo: ∞</div>
</div>
<div id="crosshair"></div>
<div id="miniMap"><canvas id="miniCanvas" width="150" height="150"></canvas></div>
<div id="rotateWarn">Rotate your phone!</div>
<div id="shootBtn"></div>
<div id="joystick"></div>
<script src="https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.160.0/examples/js/controls/PointerLockControls.js"></script>
<script>
// --- BASIC SETUP ---
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x87ceeb);
const camera = new THREE.PerspectiveCamera(75, window.innerWidth/window.innerHeight, 0.1, 1000);
const renderer = new THREE.WebGLRenderer();
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// Controls
const controls = new THREE.PointerLockControls(camera, document.body);
document.body.addEventListener('click', ()=>controls.lock());
scene.add(controls.getObject());

// Lights
const light = new THREE.DirectionalLight(0xffffff, 1);
light.position.set(1,1,1);
scene.add(light);

// Ground
const groundGeo = new THREE.PlaneGeometry(1000, 1000);
const groundMat = new THREE.MeshPhongMaterial({color: 0x228b22});
const ground = new THREE.Mesh(groundGeo, groundMat);
ground.rotation.x = -Math.PI/2;
scene.add(ground);

// Gun
const gunGeo = new THREE.BoxGeometry(0.2,0.2,1);
const gunMat = new THREE.MeshBasicMaterial({color:0x444444});
const gun = new THREE.Mesh(gunGeo, gunMat);
gun.position.set(0.3,-0.2,-0.5);
camera.add(gun);

// --- UI ---
const healthEl = document.getElementById('health');
let health = 100;

// --- AUDIO ---
const gunSound = new Audio('https://actions.google.com/sounds/v1/alarms/beep_short.ogg');
const buildSound = new Audio('https://actions.google.com/sounds/v1/cartoon/wood_plank_flicks.ogg');
const pickupSound = new Audio('https://actions.google.com/sounds/v1/cartoon/pop.ogg');

// --- ENEMIES ---
let enemies = [];
function spawnEnemy(){
  const geo = new THREE.BoxGeometry(1,2,1);
  const mat = new THREE.MeshBasicMaterial({color:0xff0000});
  const enemy = new THREE.Mesh(geo, mat);
  enemy.position.set((Math.random()-0.5)*50,1,(Math.random()-0.5)*50 - 30);
  enemy.userData = {health:50, shootCooldown:0};
  scene.add(enemy);
  enemies.push(enemy);
}
setInterval(spawnEnemy, 3000);

// Enemy bullets
let enemyBullets = [];

// Player bullets
let bullets = [];
function shoot(){
  gunSound.play();
  const bulletGeo = new THREE.SphereGeometry(0.1,8,8);
  const bulletMat = new THREE.MeshBasicMaterial({color:0xffff00});
  const bullet = new THREE.Mesh(bulletGeo, bulletMat);
  bullet.position.copy(camera.position);
  bullet.quaternion.copy(camera.quaternion);
  bullets.push({mesh:bullet,dir:new THREE.Vector3(0,0,-1).applyQuaternion(camera.quaternion)});
  scene.add(bullet);
}

// --- BUILDING ---
let builds = [];
window.addEventListener('keydown',(e)=>{
  if(e.code==='KeyB') buildWall();
  if(e.code==='KeyR') buildRamp();
});
function buildWall(){
  buildSound.play();
  const wallGeo = new THREE.BoxGeometry(4,4,0.2);
  const wallMat = new THREE.MeshBasicMaterial({color:0x8888ff});
  const wall = new THREE.Mesh(wallGeo, wallMat);
  const dir = new THREE.Vector3(0,0,-5).applyQuaternion(camera.quaternion);
  wall.position.copy(camera.position.clone().add(dir)); wall.position.y=2;
  scene.add(wall); builds.push(wall);
}
function buildRamp(){
  buildSound.play();
  const rampGeo = new THREE.BoxGeometry(4,0.2,4);
  const rampMat = new THREE.MeshBasicMaterial({color:0x88ff88});
  const ramp = new THREE.Mesh(rampGeo, rampMat);
  ramp.rotation.x = -Math.PI/4;
  const dir = new THREE.Vector3(0,0,-5).applyQuaternion(camera.quaternion);
  ramp.position.copy(camera.position.clone().add(dir));
  scene.add(ramp); builds.push(ramp);
}

// --- MOBILE CONTROLS DETECT ---
const isMobile = /Android|iPhone|iPad/i.test(navigator.userAgent);
if(isMobile){
  document.getElementById('shootBtn').style.display='block';
  document.getElementById('joystick').style.display='block';
  document.getElementById('shootBtn').addEventListener('touchstart',shoot);
}

// --- MINI-MAP ---
const miniCanvas = document.getElementById('miniCanvas');
const miniCtx = miniCanvas.getContext('2d');
function drawMiniMap(){
  miniCtx.clearRect(0,0,150,150);
  miniCtx.beginPath();
  miniCtx.arc(75,75,70,0,2*Math.PI);
  miniCtx.fillStyle='rgba(0,0,0,0.5)';
  miniCtx.fill();
  miniCtx.fillStyle='blue';
  miniCtx.beginPath();
  miniCtx.arc(75,75,5,0,2*Math.PI); miniCtx.fill();
  enemies.forEach(e=>{
    const dx=e.position.x-camera.position.x;
    const dz=e.position.z-camera.position.z;
    const angle=Math.atan2(dz,dx)-camera.rotation.y;
    const dist=Math.min(60,Math.sqrt(dx*dx+dz*dz));
    const px=75+dist*Math.cos(angle);
    const py=75+dist*Math.sin(angle);
    miniCtx.fillStyle='red';
    miniCtx.beginPath();
    miniCtx.arc(px,py,4,0,2*Math.PI); miniCtx.fill();
  });
}

// --- HEALTH DROPS ---
let drops = [];
function spawnHealthDrop(pos){
  const geo = new THREE.BoxGeometry(0.5,0.5,0.5);
  const mat = new THREE.MeshBasicMaterial({color:0x00ff00});
  const drop = new THREE.Mesh(geo,mat);
  drop.position.copy(pos);
  scene.add(drop); drops.push(drop);
}

// --- MOVEMENT ---
let keys={};
let canJump=false;
window.addEventListener('keydown',(e)=>keys[e.code]=true);
window.addEventListener('keyup',(e)=>keys[e.code]=false);
let velocity = new THREE.Vector3();
let speed=0.1;

// --- ORIENTATION LOCK ---
const rotateWarn = document.getElementById('rotateWarn');
function checkOrientation(){
  if(window.innerHeight>window.innerWidth){rotateWarn.style.display='flex';}
  else{rotateWarn.style.display='none';}
}
window.addEventListener('resize',checkOrientation);
checkOrientation();

// --- ANIMATION LOOP ---
function animate(){
  requestAnimationFrame(animate);
  const sprint = keys['ShiftLeft'] ? 0.2 : 0.1;
  if(keys['KeyW']) controls.moveForward(sprint);
  if(keys['KeyS']) controls.moveForward(-sprint);
  if(keys['KeyA']) controls.moveRight(-sprint);
  if(keys['KeyD']) controls.moveRight(sprint);
  if(keys['Space'] && canJump){velocity.y=0.2; canJump=false;}
  controls.getObject().position.y += velocity.y;
  velocity.y -= 0.01;
  if(controls.getObject().position.y<2){velocity.y=0;controls.getObject().position.y=2;canJump=true;}

  bullets.forEach((b,i)=>{
    b.mesh.position.add(b.dir.clone().multiplyScalar(1));
    enemies.forEach((enemy,ei)=>{
      if(b.mesh.position.distanceTo(enemy.position)<1){
        enemy.userData.health -= 25;
        if(enemy.userData.health<=0){
          spawnHealthDrop(enemy.position.clone());
          scene.remove(enemy); enemies.splice(ei,1);
        }
        scene.remove(b.mesh); bullets.splice(i,1);
      }
    });
    if(b.mesh.position.length()>200){scene.remove(b.mesh);bullets.splice(i,1);}
  });

  enemies.forEach(enemy=>{
    const dir = new THREE.Vector3().subVectors(camera.position, enemy.position).normalize();
    enemy.position.add(dir.multiplyScalar(0.02));
    enemy.lookAt(camera.position);
    if(enemy.userData.shootCooldown<=0){
      const ebGeo=new THREE.SphereGeometry(0.1,8,8);
      const ebMat=new THREE.MeshBasicMaterial({color:0xff0000});
      const eb=new THREE.Mesh(ebGeo,ebMat);
      eb.position.copy(enemy.position);
      enemyBullets.push({mesh:eb,dir:dir});
      scene.add(eb); enemy.userData.shootCooldown=60;
    } else enemy.userData.shootCooldown--;
  });

  enemyBullets.forEach((b,i)=>{
    b.mesh.position.add(b.dir.clone().multiplyScalar(0.5));
    if(b.mesh.position.distanceTo(camera.position)<1){
      health -= 10; healthEl.textContent='Health: '+health;
      scene.remove(b.mesh); enemyBullets.splice(i,1);
      if(health<=0){alert('You Died! Restarting...');location.reload();}
    }
    if(b.mesh.position.length()>200){scene.remove(b.mesh);enemyBullets.splice(i,1);}
  });

  drops.forEach((d,i)=>{
    if(d.position.distanceTo(camera.position)<1.5){
      pickupSound.play();
      health=Math.min(100,health+20); healthEl.textContent='Health: '+health;
      scene.remove(d); drops.splice(i,1);
    }
  });

  drawMiniMap();
  renderer.render(scene,camera);
}
animate();
</script>
</body>
</html>
