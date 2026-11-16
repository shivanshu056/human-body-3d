import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

const stage = document.getElementById('stage');
const loading = document.getElementById('loading');
const viewLabel = document.getElementById('viewLabel');

let scene, camera, renderer, controls, human;
let autoRotate = true;
let showLabels = false;
const pickables = [];
const labelSprites = [];

function material(color, roughness=0.72, metalness=0.02) {
  return new THREE.MeshStandardMaterial({ color, roughness, metalness });
}
const skin = material(0xc98f70);
const skinLight = material(0xdba487);
const muscle = material(0xb86f5e);
const dark = material(0x28354b);
const shorts = material(0x263c60);
const eyeMat = new THREE.MeshStandardMaterial({color:0x182333, roughness:.3});
const groundMat = new THREE.MeshStandardMaterial({color:0x14243a, roughness:.9});

function addMesh(parent, geometry, mat, pos, scale, name='') {
  const mesh = new THREE.Mesh(geometry, mat);
  mesh.position.set(...pos);
  if (scale) mesh.scale.set(...scale);
  mesh.castShadow = true;
  mesh.receiveShadow = true;
  if (name) { mesh.userData.part = name; pickables.push(mesh); }
  parent.add(mesh);
  return mesh;
}
function sphere(parent, pos, scale, mat, name='') {
  return addMesh(parent, new THREE.SphereGeometry(1, 32, 24), mat, pos, scale, name);
}
function capsule(parent, a, b, radius, mat, name='', radius2=null) {
  const start = new THREE.Vector3(...a), end = new THREE.Vector3(...b);
  const direction = new THREE.Vector3().subVectors(end,start);
  const length = direction.length();
  const geometry = new THREE.CylinderGeometry(radius2 ?? radius, radius, length, 24, 1);
  const mesh = addMesh(parent, geometry, mat, [0,0,0], [1,1,1], name);
  mesh.position.copy(start).add(end).multiplyScalar(.5);
  mesh.quaternion.setFromUnitVectors(new THREE.Vector3(0,1,0), direction.normalize());
  sphere(parent, a, [radius,radius,radius], mat, name);
  sphere(parent, b, [radius2 ?? radius,radius2 ?? radius,radius2 ?? radius], mat, name);
  return mesh;
}
function makeLabel(text, position) {
  const canvas = document.createElement('canvas');
  canvas.width = 256; canvas.height = 72;
  const ctx = canvas.getContext('2d');
  ctx.fillStyle = 'rgba(8,18,32,.82)';
  ctx.beginPath(); ctx.roundRect(4,4,248,64,18); ctx.fill();
  ctx.strokeStyle = '#65e8e5'; ctx.lineWidth = 3; ctx.stroke();
  ctx.fillStyle = '#e8fbff'; ctx.font = 'bold 27px Arial'; ctx.textAlign = 'center';
  ctx.fillText(text.toUpperCase(),128,47);
  const texture = new THREE.CanvasTexture(canvas);
  const sprite = new THREE.Sprite(new THREE.SpriteMaterial({map:texture, transparent:true, depthTest:false}));
  sprite.position.set(...position); sprite.scale.set(1.35,.38,1); sprite.visible=false;
  scene.add(sprite); labelSprites.push(sprite);
}
function buildHuman() {
  human = new THREE.Group();
  scene.add(human);

  // Feet and legs
  sphere(human,[-.18,.12,.04],[.19,.12,.34],skin,'Legs');
  sphere(human,[.18,.12,.04],[.19,.12,.34],skin,'Legs');
  capsule(human,[-.18,.23,0],[-.19,.88,0],.115,skin,'Legs',.14);
  capsule(human,[.18,.23,0],[.19,.88,0],.115,skin,'Legs',.14);
  sphere(human,[-.19,.91,0],[.145,.16,.15],skin,'Legs');
  sphere(human,[.19,.91,0],[.145,.16,.15],skin,'Legs');
  capsule(human,[-.19,1.02,0],[-.19,1.72,0],.16,skin,'Legs',.19);
  capsule(human,[.19,1.02,0],[.19,1.72,0],.16,skin,'Legs',.19);

  // Pelvis, waist, torso and shoulders
  sphere(human,[0,1.82,0],[.39,.28,.22],shorts,'Abdomen');
  const torso = sphere(human,[0,2.27,0],[.48,.57,.255],skin,'Chest');
  sphere(human,[0,2.02,.012],[.37,.29,.225],skin,'Abdomen');
  sphere(human,[-.27,2.61,0],[.23,.19,.22],skin,'Chest');
  sphere(human,[.27,2.61,0],[.23,.19,.22],skin,'Chest');
  // Subtle chest and abdominal forms
  sphere(human,[-.13,2.43,.205],[.16,.19,.075],skinLight,'Chest');
  sphere(human,[.13,2.43,.205],[.16,.19,.075],skinLight,'Chest');
  for (let y of [2.13,2.29]) {
    sphere(human,[-.105,y,.218],[.085,.07,.045],muscle,'Abdomen');
    sphere(human,[.105,y,.218],[.085,.07,.045],muscle,'Abdomen');
  }
  // Neck and head
  capsule(human,[0,2.69,0],[0,2.91,0],.105,skin,'Head');
  sphere(human,[0,3.15,0],[.235,.30,.205],skinLight,'Head');
  sphere(human,[0,3.01,.16],[.14,.12,.09],skinLight,'Head');
  sphere(human,[-.237,3.15,0],[.045,.075,.05],skin,'Head');
  sphere(human,[.237,3.15,0],[.045,.075,.05],skin,'Head');
  sphere(human,[-.085,3.19,.183],[.018,.024,.012],eyeMat,'Head');
  sphere(human,[.085,3.19,.183],[.018,.024,.012],eyeMat,'Head');
  // Hair cap
  const hair = material(0x283044);
  sphere(human,[0,3.34,-.005],[.239,.15,.211],hair,'Head');
  // Arms, elbows, forearms and hands
  capsule(human,[-.43,2.59,0],[-.68,2.12,0],.125,skin,'Arms',.105);
  capsule(human,[.43,2.59,0],[.68,2.12,0],.125,skin,'Arms',.105);
  sphere(human,[-.68,2.12,0],[.11,.12,.11],skin,'Arms');
  sphere(human,[.68,2.12,0],[.11,.12,.11],skin,'Arms');
  capsule(human,[-.68,2.12,0],[-.78,1.58,.02],.095,skin,'Arms',.075);
  capsule(human,[.68,2.12,0],[.78,1.58,.02],.095,skin,'Arms',.075);
  sphere(human,[-.79,1.48,.02],[.09,.14,.065],skinLight,'Arms');
  sphere(human,[.79,1.48,.02],[.09,.14,.065],skinLight,'Arms');

  // Small shoulder definition and simple fingers
  sphere(human,[-.47,2.61,0],[.18,.18,.18],skinLight,'Arms');
  sphere(human,[.47,2.61,0],[.18,.18,.18],skinLight,'Arms');
  for (const side of [-1,1]) {
    for (let i=0;i<4;i++) {
      const x=side*(.735+i*.028);
      capsule(human,[x,1.43,.045],[x+side*.012,1.34,.05],.018,skinLight,'Arms');
    }
  }

  // Simple clavicle detail
  capsule(human,[-.25,2.69,.19],[0,2.65,.21],.025,skinLight,'Chest');
  capsule(human,[0,2.65,.21],[.25,2.69,.19],.025,skinLight,'Chest');

  makeLabel('Head',[0,3.72,0]);
  makeLabel('Chest',[0,2.82,.2]);
  makeLabel('Abdomen',[0,2.05,.38]);
  makeLabel('Arms',[1.05,2.15,0]);
  makeLabel('Legs',[.48,.82,.18]);
}

function init() {
  scene = new THREE.Scene();
  scene.fog = new THREE.Fog(0x101d31, 7, 15);
  camera = new THREE.PerspectiveCamera(35, stage.clientWidth/stage.clientHeight, .1, 100);
  camera.position.set(4.2,3.1,7.4);

  renderer = new THREE.WebGLRenderer({antialias:true, alpha:true});
  renderer.setPixelRatio(Math.min(window.devicePixelRatio,2));
  renderer.setSize(stage.clientWidth,stage.clientHeight);
  renderer.shadowMap.enabled=true;
  renderer.shadowMap.type=THREE.PCFSoftShadowMap;
  renderer.outputColorSpace=THREE.SRGBColorSpace;
  renderer.toneMapping=THREE.ACESFilmicToneMapping;
  renderer.toneMappingExposure=1.15;
  stage.insertBefore(renderer.domElement, stage.firstChild);

  scene.add(new THREE.HemisphereLight(0xb9d9ff,0x273047,2.0));
  const key = new THREE.DirectionalLight(0xffffff,3.1);
  key.position.set(3,6,5); key.castShadow=true; key.shadow.mapSize.set(1024,1024); scene.add(key);
  const rim = new THREE.PointLight(0x58dce8,22,9); rim.position.set(-3,3,1); scene.add(rim);
  const fill = new THREE.PointLight(0x7b91ff,16,9); fill.position.set(2,2,-3); scene.add(fill);

  const floor = new THREE.Mesh(new THREE.CircleGeometry(2.1,64),groundMat);
  floor.rotation.x=-Math.PI/2; floor.position.y=-.02; floor.receiveShadow=true; scene.add(floor);
  const ring = new THREE.Mesh(new THREE.TorusGeometry(1.55,.008,8,100),new THREE.MeshBasicMaterial({color:0x4a8da9,transparent:true,opacity:.42}));
  ring.rotation.x=Math.PI/2; ring.position.y=.015; scene.add(ring);

  buildHuman();
  controls = new OrbitControls(camera,renderer.domElement);
  controls.target.set(0,1.9,0);
  controls.enableDamping=true; controls.dampingFactor=.06;
  controls.minDistance=3.5; controls.maxDistance=10;
  controls.maxPolarAngle=Math.PI*.88;
  controls.autoRotate=autoRotate; controls.autoRotateSpeed=1.15;
  controls.update();

  renderer.domElement.addEventListener('pointerdown',()=>{viewLabel.textContent='INTERACTIVE VIEW';});
  renderer.domElement.addEventListener('pointerup',()=>{updateViewLabel();});
  renderer.domElement.addEventListener('click',onModelClick);
  loading.classList.add('hidden');
  animate();
}
function updateViewLabel(){
  const z=camera.position.z;
  viewLabel.textContent = camera.position.x > 2 ? '3/4 VIEW' : (z<0 ? 'BACK VIEW' : 'FRONT VIEW');
}
function onModelClick(event) {
  const rect=renderer.domElement.getBoundingClientRect();
  const mouse=new THREE.Vector2(((event.clientX-rect.left)/rect.width)*2-1,-((event.clientY-rect.top)/rect.height)*2+1);
  const raycaster=new THREE.Raycaster(); raycaster.setFromCamera(mouse,camera);
  const hits=raycaster.intersectObjects(pickables,false);
  if(hits.length) showPart(hits[0].object.userData.part);
}
function showPart(part) {
  const notes={
    Head:['Head & senses','The head contains the brain and the organs that help us see, hear, smell and taste.'],
    Chest:['Chest','The chest protects important organs, including the heart and lungs, which help keep the body working.'],
    Arms:['Arms','The arms, hands and shoulder muscles help us lift, reach, hold and move objects.'],
    Abdomen:['Abdomen','The abdomen contains parts of the digestive system, which helps the body process food.'],
    Legs:['Legs','The legs support body weight and help us stand, walk, run and jump.']
  };
  const item=notes[part] || ['Human body','Explore the model by dragging, zooming and selecting a body region.'];
  document.getElementById('infoTitle').textContent=item[0];
  document.getElementById('infoText').textContent=item[1];
  document.querySelectorAll('.region').forEach(b=>b.classList.toggle('selected',b.dataset.part===part));
}
function animate(){
  requestAnimationFrame(animate);
  controls.update();
  renderer.render(scene,camera);
  updateViewLabel();
}
document.getElementById('rotateBtn').addEventListener('click',()=>{
  autoRotate=!autoRotate; controls.autoRotate=autoRotate;
  document.getElementById('rotateBtn').classList.toggle('active',autoRotate);
});
document.getElementById('resetBtn').addEventListener('click',()=>{
  camera.position.set(4.2,3.1,7.4); controls.target.set(0,1.9,0); controls.update();
});
document.getElementById('labelsBtn').addEventListener('click',()=>{
  showLabels=!showLabels; labelSprites.forEach(s=>s.visible=showLabels);
  document.getElementById('labelsBtn').classList.toggle('active',showLabels);
});
document.querySelectorAll('.region').forEach(btn=>btn.addEventListener('click',()=>showPart(btn.dataset.part)));
window.addEventListener('resize',()=>{
  if(!renderer)return;
  const w=stage.clientWidth,h=stage.clientHeight;
  camera.aspect=w/h; camera.updateProjectionMatrix(); renderer.setSize(w,h);
});
window.addEventListener('error',e=>{
  if(loading){loading.classList.remove('hidden');loading.innerHTML='<b>Could not load 3D library. Check your internet connection and reload.</b>';}
});

try { init(); }
catch (error) {
  console.error(error);
  loading.classList.remove('hidden');
  loading.innerHTML='<b>3D model could not start. Open this page with internet enabled.</b>';
}
