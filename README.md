<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<title>The Evolution of Humanity — WebGL Demo</title>
<meta name="viewport" content="width=device-width,initial-scale=1" />
<style>
  html,body { height:100%; margin:0; background:#000; overflow:hidden; font-family:system-ui,Segoe UI,Roboto,Arial; }
  #ui {
    position: absolute; left: 12px; top: 12px; color: #e8eef8;
    background: rgba(0,0,0,0.25); padding:8px 10px; border-radius:8px;
    backdrop-filter: blur(6px); box-shadow: 0 6px 18px rgba(0,0,0,0.6);
    max-width: 320px;
  }
  #ui h1 { margin:0 0 6px 0; font-size:16px; color:#fff; }
  #ui p { margin:4px 0; font-size:13px; color:#cfe0ff; }
  #canvas { display:block; }
  .btn { background:#2b6cff; color:white; padding:6px 8px; border-radius:6px; border:0; cursor:pointer; font-weight:600; }
</style>
</head>
<body>
<div id="ui">
  <h1>Evolution of Humanity — Demo</h1>
  <p><strong>Controls</strong>: WASD to move • Space to mine • M to craft RPG • P to toggle music</p>
  <p id="info">Loading...</p>
  <button id="toggleMusic" class="btn">Toggle Music</button>
</div>
<canvas id="canvas"></canvas>

<!-- Three.js CDN -->
<script src="https://cdn.jsdelivr.net/npm/three@0.162.0/build/three.min.js"></script>

<script>
// ---------- Basic setup ----------
const canvas = document.getElementById('canvas');
const renderer = new THREE.WebGLRenderer({canvas, antialias:true});
renderer.setPixelRatio(window.devicePixelRatio);
renderer.shadowMap.enabled = true;

const scene = new THREE.Scene();

// camera
const camera = new THREE.PerspectiveCamera(60, 2, 0.1, 2000);
camera.position.set(0, 25, 40);
camera.lookAt(0,0,0);

// resize
function resize(){ const w = innerWidth, h = innerHeight; renderer.setSize(w,h); camera.aspect = w/h; camera.updateProjectionMatrix(); }
addEventListener('resize', resize);
resize();

// ---------- Background / sky gradient (shader-like via big sphere) ----------
const skyGeo = new THREE.SphereGeometry(400, 32, 16);
const skyMat = new THREE.MeshBasicMaterial({side: THREE.BackSide, color: 0x051226});
const sky = new THREE.Mesh(skyGeo, skyMat);
scene.add(sky);

// animated gradient via fog and light color changes
scene.fog = new THREE.FogExp2(0x020714, 0.004);

// ---------- Ambient particle field (stars) ----------
const starCount = 800;
const starsGeo = new THREE.BufferGeometry();
const starPos = new Float32Array(starCount*3);
for(let i=0;i<starCount;i++){
  const r = 150 + Math.random()*200;
  const theta = Math.random()*Math.PI*2;
  const phi = Math.acos(2*Math.random()-1);
  starPos[i*3+0] = Math.sin(phi)*Math.cos(theta)*r;
  starPos[i*3+1] = Math.sin(phi)*Math.sin(theta)*r*0.6 + 30;
  starPos[i*3+2] = Math.cos(phi)*r;
}
starsGeo.setAttribute('position', new THREE.BufferAttribute(starPos,3));
const starsMat = new THREE.PointsMaterial({ size: 0.6, color: 0x88ccff, transparent:true, opacity:0.9 });
const stars = new THREE.Points(starsGeo, starsMat);
scene.add(stars);

// ---------- Ground ----------
const groundGeo = new THREE.PlaneGeometry(300,300,8,8);
const groundMat = new THREE.MeshStandardMaterial({ color:0x1b2b20, roughness:1, metalness:0.0 });
const ground = new THREE.Mesh(groundGeo, groundMat);
ground.rotation.x = -Math.PI/2; ground.receiveShadow = true;
scene.add(ground);

// subtle height variations
for(let i=0;i<groundGeo.attributes.position.count;i++){
  const v = groundGeo.attributes.position;
  v.setZ(i, (Math.random()-0.5)*0.5);
}
groundGeo.computeVertexNormals();

// ---------- Lighting ----------
const hemi = new THREE.HemisphereLight(0x88aaff, 0x0f2d2a, 0.75);
scene.add(hemi);
const dir = new THREE.DirectionalLight(0xfff1cc, 0.9);
dir.position.set(30,60,30); dir.castShadow = true;
dir.shadow.mapSize.set(1024,1024); dir.shadow.camera.left=-60; dir.shadow.camera.right=60; dir.shadow.camera.top=60; dir.shadow.camera.bottom=-60;
scene.add(dir);

// ---------- Basic materials ----------
const matPlayer = new THREE.MeshStandardMaterial({color:0x4da6ff, metalness:0.1, roughness:0.6});
const matEnemy = new THREE.MeshStandardMaterial({color:0xff6b6b, metalness:0.05, roughness:0.7});
const matTurret = new THREE.MeshStandardMaterial({color:0x999999, metalness:0.6, roughness:0.3});
const matOre = new THREE.MeshStandardMaterial({color:0xb3ff99, metalness:0.0, roughness:0.9});

// ---------- Game state (JS port of mechanics) ----------
const MAX_CRAFT_ITEMS = 16;
const INV_FUEL = MAX_CRAFT_ITEMS - 1;
const state = {
  player: { x:10, z:10, hp:100, mp:50, inventory: new Array(MAX_CRAFT_ITEMS).fill(0) },
  enemies: [],
  turrets: [],
  ores: []
};

// spawn some objects (positions mapped to world coords)
function spawnInitial(){
  // ores
  const oreSpecs = [
    {type:'WOOD', x:15, z:15},
    {type:'STONE', x:20, z:18},
    {type:'IRON', x:30, z:22},
    {type:'STRYDIUM', x:35, z:25},
    {type:'URANIUM', x:40, z:28}
  ];
  oreSpecs.forEach(o=>{
    const g = new THREE.IcosahedronGeometry(0.8+Math.random()*0.4,0);
    const m = new THREE.Mesh(g, matOre);
    m.position.set((o.x-20), 0.9, (o.z-20));
    scene.add(m);
    state.ores.push({mesh:m, type:o.type, x:o.x, z:o.z, mined:false});
  });

  // player representation
  const pgeo = new THREE.BoxGeometry(1.6,2,1.6);
  state.player.mesh = new THREE.Mesh(pgeo, matPlayer);
  state.player.mesh.position.set(state.player.x-20,1, state.player.z-20);
  state.player.mesh.castShadow = true;
  scene.add(state.player.mesh);

  // turret
  const tgeo = new THREE.CylinderGeometry(0.9, 1.1, 1.6, 10);
  const turretMesh = new THREE.Mesh(tgeo, matTurret);
  turretMesh.position.set(40-20,0.9, 40-20);
  turretMesh.castShadow = true;
  scene.add(turretMesh);
  state.turrets.push({mesh:turretMesh, x:40, z:40, range:30, damage:75, ammo:5});

  // enemies
  const eg = new THREE.BoxGeometry(1.5,1.5,1.5);
  for(let i=0;i<2;i++){
    const em = new THREE.Mesh(eg, matEnemy);
    const ex = 50 + i*8;
    const ez = 50 - i*2;
    em.position.set(ex-20,0.75, ez-20);
    em.castShadow = true; scene.add(em);
    state.enemies.push({mesh:em, x:ex, z:ez, hp:100, active:true, stunned:false, stunTimer:0});
  }
}
spawnInitial();

// ---------- Input ----------
const keys = {};
addEventListener('keydown', e => keys[e.key.toLowerCase()] = true);
addEventListener('keyup', e => keys[e.key.toLowerCase()] = false);

// ---------- WebAudio: ambient pad + SFX ----------
const AudioCtx = window.AudioContext || window.webkitAudioContext;
const audioCtx = new AudioCtx();
let musicOn = true;
document.getElementById('toggleMusic').onclick = () => { musicOn = !musicOn; if(musicOn) startMusic(); else stopMusic(); };

let masterGain = audioCtx.createGain(); masterGain.gain.value = 0.14; masterGain.connect(audioCtx.destination);
let musicNodes = [];
function startMusic(){
  if(audioCtx.state === 'suspended') audioCtx.resume();
  // create two detuned saw-ish oscillators and a slow LFO on filter
  const oscA = audioCtx.createOscillator(); oscA.type='sawtooth'; oscA.frequency.value = 110;
  const oscB = audioCtx.createOscillator(); oscB.type='sine'; oscB.frequency.value = 55;
  const amp = audioCtx.createGain(); amp.gain.value = 0.0;
  const lowpass = audioCtx.createBiquadFilter(); lowpass.type='lowpass'; lowpass.frequency.value = 800;
  oscA.connect(amp); oscB.connect(amp); amp.connect(lowpass); lowpass.connect(masterGain);
  oscA.start(); oscB.start();
  // slow attack to create pad
  const now = audioCtx.currentTime; amp.gain.linearRampToValueAtTime(0.08, now+1.5);
  musicNodes = [oscA, oscB, amp, lowpass];
}
function stopMusic(){
  if(!musicNodes.length) return;
  const now = audioCtx.currentTime;
  const amp = musicNodes[2];
  amp.gain.linearRampToValueAtTime(0.0, now+0.6);
  setTimeout(()=> {
    musicNodes.slice(0,2).forEach(n=>n.stop());
    musicNodes = [];
  }, 800);
}
startMusic();

// short SFX
function sfxMine(){ const o = audioCtx.createOscillator(); const g = audioCtx.createGain(); o.type='square'; o.frequency.value = 880; g.gain.value = 0.0; o.connect(g); g.connect(masterGain); o.start(); g.gain.linearRampToValueAtTime(0.12, audioCtx.currentTime+0.01); g.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime+0.25); setTimeout(()=>o.stop(), 300); }
function sfxExplode(){ const b = audioCtx.createBufferSource(); // quick crunchy noise using buffer
  const sampleRate = audioCtx.sampleRate; const len = sampleRate*0.25; const buf = audioCtx.createBuffer(1,len,sampleRate); const data = buf.getChannelData(0);
  for(let i=0;i<len;i++){ data[i] = (Math.random()*2-1) * Math.exp(-5*i/len); }
  b.buffer = buf; const g = audioCtx.createGain(); g.gain.value = 0.18; b.connect(g); g.connect(masterGain); b.start(); setTimeout(()=>b.disconnect(),300);
}

// ---------- Game interactions ----------
function mineNearby(){
  const px = state.player.x, pz = state.player.z;
  let mined = false;
  for(const ore of state.ores){
    if(ore.mined) continue;
    const d = Math.hypot(px - ore.x, pz - ore.z);
    if(d <= 2.0){
      ore.mined = true;
      scene.remove(ore.mesh);
      // map types to inventory index simple: use a small hash
      const idx = (ore.type.charCodeAt(0) % MAX_CRAFT_ITEMS);
      state.player.inventory[idx] = (state.player.inventory[idx]||0) + 1;
      mined = true;
      sfxMine();
      showInfo(`Mined ${ore.type} (inventory slot ${idx}).`);
      break;
    }
  }
  if(!mined) showInfo('No ore nearby to mine.');
}

function craftRpg(){
  // simplified: require inventory slot counts, treat INV_FUEL as index MAX-1
  const fuel = state.player.inventory[INV_FUEL] || 0;
  if(fuel < 1){ showInfo('Not enough fuel to craft.'); return false; }
  // for demo, we just consume 1 fuel and "craft"
  state.player.inventory[INV_FUEL] = fuel - 1;
  showInfo('Crafted 1 RPG (concept).');
  return true;
}

// turret behavior: target nearest active enemy in range
function turretTick(t){
  if(t.ammo <= 0) return;
  let best = null, bd = 1e9;
  for(const e of state.enemies){
    if(!e.active) continue;
    if(e.stunned) continue;
    const d = Math.hypot(t.x - e.x, t.z - e.z);
    if(d <= t.range && d < bd){ bd = d; best = e; }
  }
  if(best){
    best.hp -= t.damage; t.ammo -= 1;
    sfxExplode();
    showInfo(`Turret fired at enemy (hp=${Math.max(0,best.hp)}).`);
    if(best.hp <= 0){ best.active=false; scene.remove(best.mesh); showInfo('Enemy destroyed!'); }
  }
}

// enemy movement: move slowly toward player if active
function enemyTick(e, dt){
  if(!e.active) return;
  if(e.stunned){
    e.stunTimer -= dt; if(e.stunTimer <= 0){ e.stunned = false; showInfo('Enemy recovered from stun'); }
    return;
  }
  // move toward player
  const dx = state.player.x - e.x, dz = state.player.z - e.z;
  const L = Math.hypot(dx,dz);
  if(L > 0.01){
    e.x += (dx/L) * dt * 1.0;
    e.z += (dz/L) * dt * 1.0;
    e.mesh.position.set(e.x-20, 0.75, e.z-20);
  }
}

// ---------- UI helper ----------
const infoEl = document.getElementById('info');
let infoTimer = null;
function showInfo(text, ttl=2600){
  infoEl.textContent = text;
  if(infoTimer) clearTimeout(infoTimer);
  infoTimer = setTimeout(()=> infoEl.textContent = '', ttl);
}

// ---------- Main loop ----------
let last = performance.now();
function animate(t){
  const now = performance.now();
  const dt = (now - last)/1000;
  last = now;

  // subtle sky color animation
  const hue = (now*0.00002) % 1;
  sky.material.color.setHSL((0.6 + 0.05*Math.sin(now*0.0008)), 0.5, 0.06 + 0.015*Math.cos(now*0.0005));
  stars.rotation.y += dt * 0.01;

  // input movement
  let moved = false;
  const speed = 6.5;
  if(keys['w'] || keys['arrowup']){ state.player.z -= speed*dt; moved=true; }
  if(keys['s'] || keys['arrowdown']){ state.player.z += speed*dt; moved=true; }
  if(keys['a'] || keys['arrowleft']){ state.player.x -= speed*dt; moved=true; }
  if(keys['d'] || keys['arrowright']){ state.player.x += speed*dt; moved=true; }
  state.player.mesh.position.set(state.player.x-20, 1, state.player.z-20);

  // quick actions
  if(keys[' ']){ // space: mine (single action)
    if(!keys._spacePressed){ mineNearby(); keys._spacePressed = true; }
  } else keys._spacePressed = false;
  if(keys['m']){ if(!keys._mPressed){ craftRpg(); keys._mPressed = true; } } else keys._mPressed=false;
  if(keys['p']){ if(!keys._pPressed){ musicOn = !musicOn; if(musicOn) startMusic(); else stopMusic(); keys._pPressed=true; } } else keys._pPressed=false;

  // turret ticks (slow)
  turretTick(state.turrets[0]);

  // enemies
  for(const e of state.enemies) enemyTick(e, dt);

  // update enemy meshes positions and simple appearance
  for(const e of state.enemies){
    if(e.active) e.mesh.position.set(e.x-20, 0.75, e.z-20);
    else e.mesh.visible = false;
  }

  // camera follow player smoothly
  camera.position.lerp(new THREE.Vector3(state.player.x-20, 24, state.player.z-10), 0.06);
  camera.lookAt(state.player.mesh.position);

  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
requestAnimationFrame(animate);

// ---------- initial UI info ----------
showInfo('Welcome — demo running.');

// ---------- small guidance if AudioContext suspended (user gesture needed) ----------
document.body.addEventListener('pointerdown', ()=> { if(audioCtx.state==='suspended') audioCtx.resume(); }, {once:true});

</script>
</body>
</html>
