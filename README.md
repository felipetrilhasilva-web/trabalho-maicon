(() => {
'use strict';
const $ = id => document.getElementById(id);
const clamp = (v,a,b) => Math.max(a, Math.min(b, v));
const isTouch = (window.matchMedia && matchMedia('(pointer: coarse)').matches) || ('ontouchstart' in window);
const FH = 4.6, H = 4, T = 0.4, GW = 3;
const FLOORNAME = ['Térreo','1º andar'];
const FONT = (px,w) => (w||700)+' '+px+'px Fredoka, "Arial Rounded MT Bold", Arial, sans-serif';

/* ================= RENDERER / CENA ================= */
const canvas = $('c');
let renderer;
try { renderer = new THREE.WebGLRenderer({ canvas, antialias: true }); }
catch (e) {
  $('sTitle').textContent = 'WebGL indisponível';
  $('sText').textContent = 'Seu navegador não conseguiu iniciar o gráfico 3D. Tente outro navegador ou ative a aceleração de hardware.';
  $('sBtn').classList.add('hidden');
  return;
}
renderer.setPixelRatio(Math.min(window.devicePixelRatio || 1, 1.75));
renderer.setSize(innerWidth, innerHeight);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x05070f);
scene.fog = new THREE.Fog(0x05070f, 5, 38);
const camera = new THREE.PerspectiveCamera(74, innerWidth/innerHeight, 0.1, 90);
camera.rotation.order = 'YXZ';
scene.add(camera);

const hemi = new THREE.HemisphereLight(0xcfd8ff, 0x40404c, 0.5);
scene.add(hemi);
const flash = new THREE.SpotLight(0xfff0c8, 2.0, 34, 0.62, 0.5, 1);
camera.add(flash); camera.add(flash.target);
flash.target.position.set(0, 0, -6);
let flashOn = true;

window.addEventListener('resize', () => {
  renderer.setSize(innerWidth, innerHeight);
  camera.aspect = innerWidth/innerHeight;
  camera.updateProjectionMatrix();
});

/* ================= TEXTURAS ================= */
function mkTex(w,h,draw){
  const c = document.createElement('canvas'); c.width=w; c.height=h;
  const g = c.getContext('2d'); draw(g,w,h);
  const t = new THREE.CanvasTexture(c);
  t.wrapS = t.wrapT = THREE.RepeatWrapping; t.anisotropy = 4;
  return t;
}
function noise(g,w,h,a){
  const n=(w*h)/10;
  for(let i=0;i<n;i++){
    g.fillStyle = Math.random()<.5 ? 'rgba(0,0,0,'+a+')' : 'rgba(255,255,255,'+a+')';
    g.fillRect(Math.random()*w, Math.random()*h, 2, 2);
  }
}
const texTile = mkTex(128,128,(g,w,h)=>{
  g.fillStyle='#d5dde0'; g.fillRect(0,0,w,h);
  g.fillStyle='#b4c1c8'; g.fillRect(0,0,w/2,h/2); g.fillRect(w/2,h/2,w/2,h/2);
  g.strokeStyle='rgba(0,0,0,.28)'; g.lineWidth=3; g.strokeRect(0,0,w,h);
  g.beginPath(); g.moveTo(w/2,0); g.lineTo(w/2,h); g.moveTo(0,h/2); g.lineTo(w,h/2); g.stroke();
  noise(g,w,h,.06);
});
const texTileSm = mkTex(64,64,(g,w,h)=>{
  g.fillStyle='#e7f3f7'; g.fillRect(0,0,w,h);
  g.strokeStyle='#8fb7c4'; g.lineWidth=2; g.strokeRect(0,0,w,h);
  g.beginPath(); g.moveTo(w/2,0); g.lineTo(w/2,h); g.moveTo(0,h/2); g.lineTo(w,h/2); g.stroke();
  noise(g,w,h,.05);
});
const texWood = mkTex(128,128,(g,w,h)=>{
  const cols=['#a8713a','#9b6632','#b27a42','#a06b36'];
  for(let i=0;i<4;i++){ g.fillStyle=cols[i]; g.fillRect(0,i*h/4,w,h/4); g.fillStyle='rgba(0,0,0,.35)'; g.fillRect(0,i*h/4,w,2); }
  g.fillStyle='rgba(0,0,0,.25)'; g.fillRect(40,0,2,h/2); g.fillRect(90,h/2,2,h/2);
  noise(g,w,h,.07);
});
const mkCarpet = col => mkTex(128,128,(g,w,h)=>{ g.fillStyle=col; g.fillRect(0,0,w,h); noise(g,w,h,.14); });
const texCarpetRed = mkCarpet('#7b2230');
const texCarpetNavy = mkCarpet('#233a6b');
const texWall = mkTex(256,256,(g,w,h)=>{
  g.fillStyle='#e8dfc2'; g.fillRect(0,0,w,h*.66);
  g.fillStyle='#2f7f86'; g.fillRect(0,h*.66,w,h*.34);
  g.fillStyle='#1d5157'; g.fillRect(0,h*.66-3,w,6);
  noise(g,w,h,.05);
});
const texGrass = mkTex(128,128,(g,w,h)=>{ g.fillStyle='#1b3a22'; g.fillRect(0,0,w,h); noise(g,w,h,.12); });
const texBooks = mkTex(128,128,(g,w,h)=>{
  g.fillStyle='#4a2c14'; g.fillRect(0,0,w,h);
  const pal=['#c0392b','#2980b9','#27ae60','#f1c40f','#8e44ad','#e67e22','#ecf0f1','#16a085'];
  for(let r=0;r<4;r++){
    let x=2;
    while(x<w-4){ const bw=5+Math.random()*7, bh=h/4-10-Math.random()*6;
      g.fillStyle=pal[Math.floor(Math.random()*pal.length)];
      g.fillRect(x, r*h/4+h/4-6-bh, bw, bh); x+=bw+1; }
    g.fillStyle='#2e1a0a'; g.fillRect(0,r*h/4+h/4-6,w,6);
  }
});
function labelTex(text,w,h,bg,fg,font,border){
  const c=document.createElement('canvas'); c.width=w; c.height=h; const g=c.getContext('2d');
  g.fillStyle=bg; g.fillRect(0,0,w,h);
  if(border){ g.strokeStyle=border; g.lineWidth=10; g.strokeRect(5,5,w-10,h-10); }
  g.fillStyle=fg; g.font=font; g.textAlign='center'; g.textBaseline='middle';
  const lines=text.split('\n'); const px=parseInt(font.match(/(\d+)px/)[1]); const lh=px*1.18;
  lines.forEach((l,i)=>g.fillText(l,w/2,h/2+(i-(lines.length-1)/2)*lh));
  const t=new THREE.CanvasTexture(c); t.anisotropy=4; return t;
}
const labelMat = (text,w,h,bg,fg,font,border) => new THREE.MeshBasicMaterial({map:labelTex(text,w,h,bg,fg,font,border),color:0xcfcfcf});

/* glow sprite */
const glowTex = (()=>{ const c=document.createElement('canvas'); c.width=c.height=64; const g=c.getContext('2d');
  const r=g.createRadialGradient(32,32,2,32,32,32); r.addColorStop(0,'rgba(255,255,255,1)'); r.addColorStop(.35,'rgba(255,255,255,.35)'); r.addColorStop(1,'rgba(255,255,255,0)');
  g.fillStyle=r; g.fillRect(0,0,64,64); return new THREE.CanvasTexture(c); })();
function glow(x,y,z,color,size){
  const s=new THREE.Sprite(new THREE.SpriteMaterial({map:glowTex,color,blending:THREE.AdditiveBlending,transparent:true,depthWrite:false}));
  s.position.set(x,y,z); s.scale.set(size,size,1); scene.add(s); return s;
}

/* ================= MATERIAIS ================= */
const ph  = (c,e) => new THREE.MeshPhongMaterial(Object.assign({color:c,shininess:8,specular:0x101010},e||{}));
const phT = (t,e) => new THREE.MeshPhongMaterial(Object.assign({map:t,shininess:6,specular:0x0c0c0c},e||{}));
const lightMat = new THREE.MeshBasicMaterial({color:0x20232e});
const flickMats = [0,1,2].map(()=>new THREE.MeshBasicMaterial({color:0x20232e}));
const M = {
  wall:phT(texWall), tile:phT(texTile), tileSm:phT(texTileSm), wood:phT(texWood), carpetRed:phT(texCarpetRed), carpetNavy:phT(texCarpetNavy),
  grass:phT(texGrass), books:phT(texBooks),
  ceil:ph(0x3b4150), woodL:ph(0xc08a4b), woodDk:ph(0x5a3a1c), woodM:ph(0x8b5a2b),
  metal:ph(0x8e9aa6,{shininess:40}), dark:ph(0x2b2f3a), chair:ph(0x2f6fb0), white:ph(0xf1f1f1), lintel:ph(0xe8dfc2),
  lockDark:ph(0x1b1f27), skin:ph(0xe0b48a), leather:ph(0x3b1e14), sofa:ph(0x2b5d8a), sofa2:ph(0x9a2b3a),
  fridge:ph(0xdfe6ea), counter:ph(0x6d4c41), plant:ph(0x2e8b3a), pot:ph(0xb5651d), paper:ph(0xfaf6e8),
  stairs:ph(0x7b8794), pit:ph(0x050608), bed:ph(0xe9eef2), blanket:ph(0x4a8fd8), rack:ph(0x15181f),
  screen:new THREE.MeshBasicMaterial({color:0x2a74ff}), win:new THREE.MeshBasicMaterial({color:0x27467f}),
  ledG:new THREE.MeshBasicMaterial({color:0x35ff7a}), ledR:new THREE.MeshBasicMaterial({color:0xff3b3b}),
  locker:[ph(0x2e86de),ph(0xe67e22),ph(0x27ae60),ph(0x8e44ad),ph(0xe74c3c)],
  easel:ph(0xb88a52), paint:[ph(0xe74c3c),ph(0xf1c40f),ph(0x3498db),ph(0x2ecc71)],
  glass:ph(0x7fb3c8,{transparent:true,opacity:.75}), gym:ph(0xd35400)
};

/* ================= BATCHING (junta geometrias estáticas) ================= */
const buckets = new Map();
function addGeo(geo,mat){
  let b=buckets.get(mat.uuid);
  if(!b){ b={mat,pos:[],nor:[],uv:[],idx:[],n:0}; buckets.set(mat.uuid,b); }
  const p=geo.attributes.position.array, nn=geo.attributes.normal.array, u=geo.attributes.uv.array, ix=geo.index.array;
  for(let i=0;i<p.length;i++){ b.pos.push(p[i]); b.nor.push(nn[i]); }
  for(let i=0;i<u.length;i++) b.uv.push(u[i]);
  for(let i=0;i<ix.length;i++) b.idx.push(ix[i]+b.n);
  b.n+=p.length/3;
}
function flushBatches(){
  buckets.forEach(b=>{
    const g=new THREE.BufferGeometry();
    g.setAttribute('position',new THREE.Float32BufferAttribute(b.pos,3));
    g.setAttribute('normal',new THREE.Float32BufferAttribute(b.nor,3));
    g.setAttribute('uv',new THREE.Float32BufferAttribute(b.uv,2));
    g.setIndex(b.n>65000?new THREE.Uint32BufferAttribute(b.idx,1):new THREE.Uint16BufferAttribute(b.idx,1));
    const m=new THREE.Mesh(g,b.mat); m.frustumCulled=false; scene.add(m);
  });
  buckets.clear();
}

/* ================= CONSTRUTORES ================= */
let CF = 0; // andar atual da construção
const walls = [[],[]];  // bloqueiam guarda, jogador e visão
const props = [[],[]];  // bloqueiam só o jogador
function addProp(minX,maxX,minZ,maxZ){ const c={minX,maxX,minZ,maxZ,active:true}; props[CF].push(c); return c; }
function scaleUV(geo,su,sv){
  const uv=geo.attributes.uv.array;
  for(let i=0;i<uv.length;i+=2){ uv[i]*=su; uv[i+1]*=sv; }
}
function box(w,h,d,mat,x,y,z,col,o){
  const g=new THREE.BoxGeometry(w,h,d);
  if(o&&(o.su||o.sv)) scaleUV(g,o.su||1,o.sv||1);
  g.translate(x,y+CF*FH,z);
  addGeo(g,mat); g.dispose();
  if(col){
    const c={minX:x-w/2,maxX:x+w/2,minZ:z-d/2,maxZ:z+d/2,active:true};
    (col==='wall'?walls:props)[CF].push(c);
    return c;
  }
}
function cyl(rt,rb,h,seg,mat,x,y,z,col){
  const g=new THREE.CylinderGeometry(rt,rb,h,seg); g.translate(x,y+CF*FH,z); addGeo(g,mat); g.dispose();
  if(col) addProp(x-Math.max(rt,rb),x+Math.max(rt,rb),z-Math.max(rt,rb),z+Math.max(rt,rb));
}
function sph(r,mat,x,y,z){ const g=new THREE.SphereGeometry(r,10,8); g.translate(x,y+CF*FH,z); addGeo(g,mat); g.dispose(); }
function soloBox(w,h,d,mat,x,y,z){
  const m=new THREE.Mesh(new THREE.BoxGeometry(w,h,d),mat); m.position.set(x,y+CF*FH,z); scene.add(m); return m;
}
function plane(w,h,mat,x,y,z,ry,rx){
  const m=new THREE.Mesh(new THREE.PlaneGeometry(w,h),mat);
  m.position.set(x,y+CF*FH,z); m.rotation.set(rx||0,ry||0,0); scene.add(m); return m;
}
function floorRect(x1,z1,x2,z2,mat,ts){
  const w=x2-x1,d=z2-z1, t=ts||2;
  const g=new THREE.PlaneGeometry(w,d); g.rotateX(-Math.PI/2); scaleUV(g,w/t,d/t);
  g.translate((x1+x2)/2,CF*FH,(z1+z2)/2); addGeo(g,mat); g.dispose();
}
function wallH(z,x1,x2){ const len=Math.abs(x2-x1); if(len<0.01) return; box(len,H,T,M.wall,(x1+x2)/2,H/2,z,'wall',{su:len/4}); }
function wallV(x,z1,z2){ const len=Math.abs(z2-z1); if(len<0.01) return; box(T,H,len,M.wall,x,H/2,(z1+z2)/2,'wall',{su:len/4}); }
function wallX(z,x1,x2,gaps){
  let cur=x1; gaps=gaps.slice().sort((a,b)=>a-b);
  gaps.forEach(g=>{ wallH(z,cur,g-GW/2); cur=g+GW/2; box(GW,1,T,M.lintel,g,3.5,z); });
  wallH(z,cur,x2);
}
function wallZ(x,z1,z2,gaps){
  let cur=z1; gaps=gaps.slice().sort((a,b)=>a-b);
  gaps.forEach(g=>{ wallV(x,cur,g-GW/2); cur=g+GW/2; box(T,1,GW,M.lintel,x,3.5,g); });
  wallV(x,cur,z2);
}

/* ================= SISTEMAS DE JOGO ================= */
const INV = {
  blue:{n:'Chave Azul',c:0x2b8cff}, green:{n:'Chave Verde',c:0x2ecc71}, red:{n:'Chave Vermelha',c:0xff3b3b},
  cutter:{n:'Alicate',c:0xff8c1a}, fuse:{n:'Fusível',c:0xffe066}
};
const inv = {blue:0,green:0,red:0,cutter:0,fuse:0};
const flags = {blue:false,cutter:false,green:false,red:false,fuseZ:false,fuseL:false,fuseP:false};
let fuseIn = 0, power = false, alarm = false;
const resetHooks = [];
const onReset = fn => resetHooks.push(fn);
const I = []; // interativos
function addI(o){
  if(o.f===undefined) o.f=CF;
  o.range=o.range||2.4; o.dot=(o.dot===undefined)?0.3:o.dot;
  I.push(o); return o;
}
function noteI(x,z,title,body,label,range){
  addI({x,z,range:range||2.4,dot:0.3,text:()=>label||('Ler: '+title),act:()=>showNote(title,body)});
}
function paper(x,y,z,withNote){
  box(0.38,0.015,0.27,M.paper,x,y,z);
  glow(x,y+CF*FH+0.25,z,0xfff0b0,0.7);
}

/* ----- itens ----- */
const items = [];
function buildItemMesh(id){
  const g=new THREE.Group(), col=INV[id].c;
  const m=ph(col,{emissive:col,emissiveIntensity:.7});
  if(id==='fuse'){
    const b=new THREE.Mesh(new THREE.CylinderGeometry(.07,.07,.34,10),ph(0xffe066,{emissive:0xffcc22,emissiveIntensity:.8,transparent:true,opacity:.9})); g.add(b);
    [-.18,.18].forEach(y=>{ const c=new THREE.Mesh(new THREE.CylinderGeometry(.09,.09,.06,10),M.metal); c.position.y=y; g.add(c); });
  } else if(id==='cutter'){
    [-1,1].forEach(s=>{ const a=new THREE.Mesh(new THREE.BoxGeometry(.06,.5,.04),M.metal); a.position.set(s*.05,.05,0); a.rotation.z=s*.18; g.add(a);
      const h=new THREE.Mesh(new THREE.BoxGeometry(.07,.28,.06),ph(0xff8c1a,{emissive:0xff6a00,emissiveIntensity:.5})); h.position.set(s*.13,-.3,0); h.rotation.z=s*.18; g.add(h); });
  } else {
    const ring=new THREE.Mesh(new THREE.TorusGeometry(.14,.045,8,18),m); ring.position.y=.3; g.add(ring);
    const shaft=new THREE.Mesh(new THREE.BoxGeometry(.07,.42,.05),m); g.add(shaft);
    [-.06,-.18].forEach(yy=>{ const t=new THREE.Mesh(new THREE.BoxGeometry(.14,.06,.05),m); t.position.set(.09,yy,0); g.add(t); });
  }
  g.scale.setScalar(1.35);
  const s=new THREE.Sprite(new THREE.SpriteMaterial({map:glowTex,color:col,blending:THREE.AdditiveBlending,transparent:true,depthWrite:false}));
  s.scale.set(1.5,1.5,1); s.position.y=.1; g.add(s);
  return g;
}
function spawnItem(id,x,y,z,hidden,tag){
  const o={id,tag:tag||id,f:CF,x,z,baseY:y+CF*FH,got:false,hiddenInit:!!hidden,revealed:!hidden,group:buildItemMesh(id)};
  o.group.position.set(x,o.baseY,z); o.group.visible=o.revealed; scene.add(o.group); items.push(o);
  addI({f:o.f,x,z,range:2.5,dot:0.25,text:()=>(o.revealed&&!o.got)?('Pegar '+INV[id].n):null,act:()=>pickItem(o)});
  onReset(()=>{ o.got=false; o.revealed=!o.hiddenInit; o.group.visible=o.revealed; });
  return o;
}
function reveal(o){ o.revealed=true; o.group.visible=true; }
function pickItem(o){
  o.got=true; o.group.visible=false; inv[o.id]++; flags[o.tag]=true;
  sfx.pick(); showToast('Você pegou: '+INV[o.id].n+'!'); refreshInv();
}

/* ----- portas ----- */
const doors = [], doorAt = {};
function openDoor(d){
  d.open=true; d.col.active=false; sfx.door();
  if(d.screenMat) d.screenMat.color.set(0x35ff7a);
  showToast('Aberta: '+d.label);
}
function makeDoor(f,cx,side,type,o){
  const cz=side*3, saveCF=CF; CF=f;
  const pivot=new THREE.Group(); pivot.position.set(cx-GW/2,f*FH,cz); scene.add(pivot);
  const bc = type==='key' ? INV[o.keyId].c : (type==='chain' ? 0x6b5b4b : 0x4a5568);
  const body=new THREE.Mesh(new THREE.BoxGeometry(GW,3,0.2),ph(bc,{emissive:type==='key'?bc:0x000000,emissiveIntensity:.18}));
  body.position.set(GW/2,1.5,0); pivot.add(body);
  const pan=new THREE.Mesh(new THREE.BoxGeometry(GW-0.5,2.5,0.24),ph(0x1a1f2b)); pan.position.set(GW/2,1.5,0); pivot.add(pan);
  const knob=new THREE.Mesh(new THREE.BoxGeometry(0.28,0.34,0.3),ph(0xffd23f,{emissive:0xffd23f,emissiveIntensity:.4})); knob.position.set(GW-0.35,1.4,0); pivot.add(knob);
  const col={minX:cx-GW/2,maxX:cx+GW/2,minZ:cz-0.2,maxZ:cz+0.2,active:true,door:true}; walls[f].push(col);
  const d={type,f,cx,side,label:o.label,keyId:o.keyId,code:o.code,open:false,progress:0,col,sign:side<0?1:-1,
    apply(p){ pivot.rotation.y=this.sign*Math.PI/2*p; },
    reset(){ this.open=false; this.progress=0; col.active=true; this.apply(0); if(this.chain) this.chain.visible=true; if(this.screenMat) this.screenMat.color.set(0xff3b3b); }};
  if(type==='chain'){
    const chain=new THREE.Group();
    const cm=ph(0x9a9a9a,{shininess:60});
    [1.0,1.9].forEach(y=>{ for(let i=0;i<11;i++){ const l=new THREE.Mesh(new THREE.TorusGeometry(.11,.03,6,10),cm); l.position.set(0.15+i*0.27,y,0.14*(i%2?1:-1)*0.6); l.rotation.y=(i%2)?Math.PI/2:0; chain.add(l); } });
    const lock=new THREE.Mesh(new THREE.BoxGeometry(.3,.38,.16),ph(0xd4af37,{shininess:60})); lock.position.set(GW/2,1.45,0.2); chain.add(lock);
    pivot.add(chain); d.chain=chain;
  }
  if(type==='keypad'||type==='safe'){
    // painel do teclado ao lado da porta
    const px=cx+2.1, pz=side*2.74;
    box(0.4,0.55,0.08,M.dark,px,1.4,pz);
    d.screenMat=new THREE.MeshBasicMaterial({color:0xff3b3b});
    const sc=soloBox(0.28,0.12,0.02,d.screenMat,px,1.55,side*2.69);
    glow(px,1.5+f*FH,side*2.55,0xff6a6a,0.9);
    d.px=px; d.pz=pz;
  }
  d.text=()=>{
    if(d.open) return null;
    if(type==='key') return inv[d.keyId] ? ('Abrir '+d.label) : (d.label+' trancada – precisa da '+INV[d.keyId].n);
    if(type==='chain') return inv.cutter ? 'Cortar a corrente com o alicate' : (d.label+': corrente grossa – falta uma ferramenta');
    return 'Digitar o código – '+d.label;
  };
  d.act=()=>{
    if(type==='key'){ if(inv[d.keyId]) openDoor(d); else { sfx.fail(); showToast('Trancada! Precisa da '+INV[d.keyId].n+'.'); } }
    else if(type==='chain'){ if(inv.cutter){ d.chain.visible=false; sfx.cut(); openDoor(d); } else { sfx.fail(); showToast('A corrente é muito grossa. Procure uma ferramenta.'); } }
    else openPad({title:d.label,len:4,code:d.code,onOk:()=>openDoor(d)});
  };
  addI({f,x:cx,z:cz,range:3.4,dot:0.3,text:()=>d.text(),act:()=>d.act()});
  if(type==='keypad') addI({f,x:cx+2.1,z:side*2.1,range:1.6,dot:0.2,text:()=>d.text(),act:()=>d.act()});
  doors.push(d); onReset(()=>d.reset());
  doorAt[f+':'+cx+':'+side]=d; CF=saveCF;
  return d;
}

/* ----- puzzles de sequência ----- */
function makeSeq(o){
  const S={progress:0,solved:false};
  function visualReset(){ o.nodes.forEach(nd=>{ nd.mesh.position.y=nd.y0; nd.mat.emissive.setHex(0x000000); }); }
  o.nodes.forEach((nd,i)=>{
    addI({f:nd.f,x:nd.x,z:nd.z,range:o.range||2.2,dot:0.35,text:()=>S.solved?null:(o.verb+' '+nd.label),act:()=>{
      if(S.solved) return;
      if(o.order[S.progress]===i){
        S.progress++; nd.mesh.position.y=nd.y0-0.045; nd.mat.emissive.setHex(nd.color); nd.mat.emissiveIntensity=.7;
        tone(o.tones[i],.28,'triangle',.18);
        if(S.progress===o.order.length){ S.solved=true; setTimeout(()=>sfx.solve(),250); o.onSolve(); }
        else showToast('Certo… ('+S.progress+'/'+o.order.length+')',1200);
      } else {
        S.progress=0; visualReset(); sfx.fail(); showToast('Ordem errada! Recomeçando…',1600);
      }
    }});
  });
  onReset(()=>{ S.progress=0; S.solved=false; visualReset(); });
}
/* ----- puzzle de interruptores ----- */
function makeSwitches(o){
  const S={solved:false,state:o.initial.slice()};
  function paint(){ o.nodes.forEach((nd,i)=>{ nd.handle.position.y=nd.y0+(S.state[i]?0.2:-0.2); nd.mat.color.setHex(S.state[i]?0x35ff7a:0xff5555); nd.mat.emissive.setHex(S.state[i]?0x1a8a40:0x8a1a1a); }); }
  o.nodes.forEach((nd,i)=>{
    addI({f:nd.f,x:nd.x,z:nd.z,range:2.0,dot:0.4,text:()=>S.solved?null:('Mover interruptor '+(i+1)),act:()=>{
      if(S.solved) return;
      S.state[i]=S.state[i]?0:1; paint(); sfx.click();
      if(S.state.every((v,k)=>v===o.target[k])){ S.solved=true; sfx.solve(); o.onSolve(); }
    }});
  });
  paint();
  onReset(()=>{ S.solved=false; S.state=o.initial.slice(); paint(); });
}

/* ----- armários ----- */
const lockers = [];
function locker(x,z,nx,nz){
  const w = nz!==0 ? 0.9 : 0.8, d = nz!==0 ? 0.8 : 0.9;
  const mat = M.locker[(Math.abs(Math.round(x*3+z*5)))%M.locker.length];
  box(w,2.2,d,mat,x,1.1,z);
  const fx=x+nx*0.41, fz=z+nz*0.41;
  box(nz!==0?0.7:0.02,1.9,nz!==0?0.02:0.7,M.lockDark,fx,1.1,fz);
  for(let i=0;i<3;i++) box(nz!==0?0.5:0.03,0.04,nz!==0?0.03:0.5,M.metal,fx+nx*0.01,1.75-i*0.08,fz+nz*0.01);
  addProp(x-w/2,x+w/2,z-d/2,z+d/2);
  const l={f:CF,x,z,nx,nz}; lockers.push(l);
  addI({f:CF,x,z,range:2.1,dot:0.4,text:()=>'Esconder-se no armário',act:()=>enterLocker(l)});
}

/* ================= ÁUDIO ================= */
let actx=null, master=null, siren=null;
function initAudio(){
  if(actx) return;
  try{
    actx=new (window.AudioContext||window.webkitAudioContext)();
    master=actx.createGain(); master.gain.value=.55; master.connect(actx.destination);
    const o=actx.createOscillator(), g=actx.createGain();
    o.type='sawtooth'; o.frequency.value=46; g.gain.value=.012; o.connect(g); g.connect(master); o.start();
  }catch(e){ actx=null; }
}
function tone(f,d,type,v,slide){
  if(!actx) return;
  try{
    const t=actx.currentTime, o=actx.createOscillator(), g=actx.createGain();
    o.type=type||'sine'; o.frequency.setValueAtTime(f,t);
    if(slide) o.frequency.exponentialRampToValueAtTime(Math.max(20,f+slide),t+d);
    g.gain.setValueAtTime(v||.15,t); g.gain.exponentialRampToValueAtTime(.001,t+d);
    o.connect(g); g.connect(master); o.start(t); o.stop(t+d+.03);
  }catch(e){}
}
function startSiren(){
  if(!actx||siren) return;
  try{
    const o=actx.createOscillator(), l=actx.createOscillator(), lg=actx.createGain(), g=actx.createGain();
    o.type='sawtooth'; o.frequency.value=720; l.frequency.value=.9; lg.gain.value=220; g.gain.value=.035;
    l.connect(lg); lg.connect(o.frequency); o.connect(g); g.connect(master); o.start(); l.start(); siren={o,l,g};
  }catch(e){}
}
function stopSiren(){ if(siren){ try{siren.o.stop();siren.l.stop();}catch(e){} siren=null; } }
const sfx = {
  pick(){ tone(660,.12,'square',.09); setTimeout(()=>tone(990,.22,'square',.09),110); },
  door(){ tone(130,.5,'sawtooth',.12,-70); },
  cut(){ tone(900,.08,'square',.12,-500); setTimeout(()=>tone(700,.1,'square',.12,-400),90); },
  fail(){ tone(110,.18,'square',.1); },
  click(){ tone(520,.05,'square',.07); },
  key(){ tone(700,.06,'square',.07); },
  solve(){ [523,659,784].forEach((f,i)=>setTimeout(()=>tone(f,.2,'triangle',.16),i*100)); },
  sting(){ tone(260,.6,'sawtooth',.2,-140); tone(277,.6,'sawtooth',.18,-150); },
  hide(){ tone(180,.25,'square',.08,-60); },
  lose(){ tone(120,1.1,'sawtooth',.26,-90); },
  win(){ [523,659,784,1046].forEach((f,i)=>setTimeout(()=>tone(f,.25,'triangle',.18),i*140)); },
  power(){ tone(80,1.2,'sawtooth',.25,300); }
};

/* ================= CONSTRUÇÃO DA ESCOLA ================= */
const lightFix = [];
function ceilLight(x,z,mat){ box(2,0.08,0.5,mat||lightMat,x,H-0.05,z); }
function doorSign(cx,side,text,bg,border){
  const m=labelMat(text,512,144,bg,'#fff',FONT(60),border);
  if(side<0) plane(2.8,0.8,m,cx,3.5,-3+T/2+0.02,0); else plane(2.8,0.8,m,cx,3.5,3-T/2-0.02,Math.PI);
}
function windowAt(x,s){
  plane(2.2,1.4,M.win,x,2.3,s*19.78,s<0?0:Math.PI);
  plane(0.06,1.4,M.white,x,2.3,s*19.77,s<0?0:Math.PI);
}
function poster(x,side,text,bg,fg){
  const m=labelMat(text,256,340,bg,fg,FONT(36),'rgba(0,0,0,.35)');
  if(side<0) plane(1.1,1.45,m,x,1.9,-3+T/2+0.02,0); else plane(1.1,1.45,m,x,1.9,3-T/2-0.02,Math.PI);
}
function roomShell(cx,s,mat,name,bg,border){
  floorRect(cx-10,s<0?-20:3,cx+10,s<0?-3:20,mat);
  doorSign(cx,s,name,bg,border);
  [-6,6].forEach(dx=>windowAt(cx+dx,s));
  [7,15].forEach((zc,i)=>[-5,5].forEach(dx=>ceilLight(cx+dx,s*zc,(i+(dx>0?1:0))%4===1?flickMats[2]:null)));
  locker(cx-9.4,s*6,1,0); locker(cx+9.4,s*6,-1,0);
}
function desk(x,z,face,mat){
  box(1.2,0.06,0.7,mat||M.woodL,x,0.75,z);
  box(0.05,0.72,0.6,M.metal,x-0.55,0.36,z); box(0.05,0.72,0.6,M.metal,x+0.55,0.36,z);
  const cz=z+0.75*face;
  box(0.45,0.05,0.45,M.chair,x,0.45,cz); box(0.45,0.5,0.05,M.chair,x,0.72,cz+0.22*face); box(0.08,0.43,0.08,M.metal,x,0.22,cz);
  addProp(x-0.6,x+0.6,z-0.35,z+0.35);
}
function chalkBoard(cx,s,text){
  box(6.4,1.9,0.06,M.woodDk,cx,2.1,s*19.77);
  plane(6,1.5,labelMat(text,512,128,'#1f4d3a','#eef7ee',FONT(38,600)),cx,2.1,s*19.735,s<0?0:Math.PI);
}
function classroom(cx,s,title){
  chalkBoard(cx,s,title);
  const tz=s*17.2;
  box(2.2,0.08,0.9,M.woodDk,cx,0.92,tz);
  box(0.08,0.9,0.8,M.woodM,cx-1,0.45,tz); box(0.08,0.9,0.8,M.woodM,cx+1,0.45,tz);
  addProp(cx-1.1,cx+1.1,tz-0.45,tz+0.45);
  [-7,-4.5,4.5,7].forEach(dx=>[8,11,14].forEach(dz=>desk(cx+dx,s*dz,-s)));
}
function bookcase(x,z,len){ box(0.6,2.8,len,M.books,x,1.4,z,'prop',{su:Math.max(1,len/2)}); }

/* ---------- CORREDOR + ESTRUTURA DE CADA ANDAR ---------- */
function buildFloor(f){
  CF=f;
  floorRect(-40,-3,40,3,M.tile);
  { const g=new THREE.PlaneGeometry(80.8,40.8); g.rotateX(Math.PI/2); g.translate(0,f*FH+H,0); addGeo(g,M.ceil); g.dispose(); }
  const gaps=[-30,-10,10,30];
  wallX(-3,-40.2,40.2,gaps); wallX(3,-40.2,40.2,gaps);
  wallX(-20,-40.2,40.2,[]); wallX(20,-40.2,40.2,[]);
  wallZ(-40,-20,20,[]); wallZ(40,-20,20,f===0?[0]:[]);
  [-20,0,20].forEach(x=>{ wallZ(x,-20,-3,[]); wallZ(x,3,20,[]); });
  // luzes do corredor
  for(let i=0,x=-36;x<=36;x+=8,i++) ceilLight(x,0,i%4===1?flickMats[0]:(i%7===3?flickMats[1]:null));
  // armários do corredor
  [-26,-16,-6,6,16,26,36].forEach(x=>locker(x,-2.4,0,1));
  [-24,-14,-4,4,14,24,36].forEach(x=>locker(x,2.4,0,-1));
  // escada (extremo oeste)
  if(f===0){
    for(let i=0;i<9;i++){ const h=(i+1)*0.44; box(0.52,h,4.6,M.stairs,-35.2-0.26-i*0.52,h/2,0); }
    addProp(-39.9,-35.0,-2.4,2.4);
    plane(2.4,0.7,labelMat('ESCADA  ▲  1º ANDAR',512,150,'#12306b','#fff',FONT(46),'#8fb4ff'),-34,3.1,-2.78,0);
    addI({f,x:-34.4,z:0,range:2.4,dot:0.2,text:()=>'Subir para o 1º andar',act:()=>changeFloor(1)});
    // saída leste
    plane(2.8,0.8,new THREE.MeshBasicMaterial({map:labelTex('SAÍDA',512,144,'#0b7a3b','#eaffea',FONT(76),'#7dffa8')}),40-T/2-0.02,3.5,0,-Math.PI/2);
    plane(3.4,1.1,labelMat('FUJA DA ESCOLA!\nResolva os enigmas',512,190,'#7a1020','#fff',FONT(50),'#ff8a95'),-33.5,2.4,2.78,Math.PI);
  } else {
    box(4.8,0.05,4.6,M.pit,-37.5,0.03,0);
    box(0.1,0.08,4.6,M.metal,-35.3,1.0,0);
    [-2.2,0,2.2].forEach(z=>box(0.08,1.0,0.08,M.metal,-35.3,0.5,z));
    addProp(-39.9,-35.1,-2.4,2.4);
    plane(2.4,0.7,labelMat('ESCADA  ▼  TÉRREO',512,150,'#12306b','#fff',FONT(46),'#8fb4ff'),-34,3.1,-2.78,0);
    addI({f,x:-34.4,z:0,range:2.4,dot:0.2,text:()=>'Descer para o térreo',act:()=>changeFloor(0)});
    plane(4,2,M.win,40-T/2-0.02,2.3,0,-Math.PI/2);
    plane(0.08,2,M.white,40-T/2-0.03,2.3,0,-Math.PI/2);
  }
}

/* ---------- TÉRREO ---------- */
buildFloor(0);
poster(-22,-1,'NÃO\nCORRA NOS\nCORREDORES','#fff3b0','#7a1020');
poster(-13.5,1,'SILÊNCIO\nPOR FAVOR','#bde0fe','#12306b');
poster(20,-1,'PROVA\nDE QUÍMICA\nAMANHÃ','#d3f9d8','#14532d');
poster(18,1,'FESTA\nJUNINA\nSEXTA!','#ffd6e0','#8a1c4a');
// placa de fundação (pista)
plane(1.7,0.95,labelMat('ESCOLA ESTADUAL\nPROF. ALMEIDA\nFundada em 1987',384,216,'#b08d2e','#fff7d6',FONT(32),'#7a5f12'),3,1.8,-2.78,0);
glow(3,1.9,-2.5,0xffe58a,1.0);
addI({f:0,x:3,z:-1.9,range:2.6,dot:0.3,text:()=>'Ler a placa na parede',act:()=>showNote('Placa da escola','ESCOLA ESTADUAL PROFESSOR ALMEIDA\n\nFundada em 1987.\n\n"Educar é abrir portas."')});

// SALA 1 – Chave Azul
roomShell(-30,-1,M.wood,'SALA 1','#c0392b','#fff');
classroom(-30,-1,'Sala 1 – Aula de Geografia');
spawnItem('blue',-30.6,1.35,-17.2);
paper(-29.2,0.97,-17.2);
noteI(-29.2,-17.2,'Bilhete do professor','Para a turma de detenção:\n\nNa biblioteca, a gaveta secreta só abre se os livros forem empurrados na ordem do ARCO-ÍRIS:\n\nvermelho, amarelo, verde e azul.\n\n(Quem estiver lendo isto, boa sorte.)','Ler o bilhete sobre a mesa');

// BIBLIOTECA – livros -> Alicate
roomShell(-10,-1,M.carpetNavy,'BIBLIOTECA','#1f5fbf','#2b8cff');
[10.5,16].forEach(z=>[-6.5,6.5].forEach(dx=>bookcase(-10+dx,-z,4)));
box(3.4,0.08,1,M.woodDk,-10,0.82,-17.5);
box(0.08,0.8,0.9,M.woodM,-11.6,0.4,-17.5); box(0.08,0.8,0.9,M.woodM,-8.4,0.4,-17.5);
addProp(-11.7,-8.3,-18,-17);
{
  const cutter=spawnItem('cutter',-10,1.5,-17.5,true);
  const defs=[['azul',0x2b66e8],['vermelho',0xe74c3c],['amarelo',0xf1c40f],['verde',0x2ecc71]];
  const nodes=defs.map((dd,i)=>{
    const x=-11.05+i*0.7, z=-17.5, mat=ph(dd[1]);
    const m=soloBox(0.5,0.14,0.36,mat,x,0.94,z);
    return {f:0,x,z,y0:m.position.y,mesh:m,mat,color:dd[1],label:'o livro '+dd[0]};
  });
  makeSeq({nodes,order:[1,2,3,0],verb:'Empurrar',tones:[330,262,392,440],range:2.3,
    onSolve:()=>{ reveal(cutter); showToast('Uma gaveta secreta se abriu: um ALICATE!',3000); }});
}
noteI(-10,-12.5,'Cartaz da biblioteca','"Os livros guardam segredos para quem os lê na ordem certa."\n\nDica: procure um bilhete deixado por um professor.','Ler o cartaz na estante',2.8);

// LABORATÓRIO – código 1987 + interruptores -> Fusível 2
roomShell(10,-1,M.tile,'LABORATÓRIO','#16856b','#fff');
[[5,8],[5,12.5],[-5,8],[-5,12.5]].forEach(([dx,dz])=>{
  const x=10+dx,z=-dz;
  box(4.2,0.9,1,M.white,x,0.45,z,'prop'); box(4.3,0.08,1.1,M.dark,x,0.94,z);
  for(let i=0;i<4;i++) cyl(0.07,0.09,0.25,8,M.paint[i],x-1.4+i*0.9,1.12,z+(i%2?0.2:-0.15));
});
box(3.4,2.2,0.6,M.glass,10-6,1.1,-19.4,'prop'); box(3.4,2.2,0.6,M.glass,10+6,1.1,-19.4,'prop');
{
  const fuse2=spawnItem('fuse',10,1.2,-18.3,true,'fuseL');
  box(3.6,1.7,0.15,M.dark,10,1.5,-19.7);
  const labelY=[];
  const nodes=[0,1,2,3].map(i=>{
    const x=10-1.2+i*0.8, z=-19.55;
    box(0.1,0.7,0.05,M.metal,x,1.5,z);
    plane(0.4,0.4,labelMat(String(i+1),128,128,'#101521','#fff',FONT(80)),x,2.0,-19.6,0);
    const mat=ph(0xff5555,{emissive:0x8a1a1a});
    const h=soloBox(0.22,0.14,0.12,mat,x,1.3,-19.5);
    return {f:0,x,z:-19.1,y0:h.position.y,handle:h,mat};
  });
  makeSwitches({nodes,initial:[0,1,1,0],target:[1,0,1,1],onSolve:()=>{ reveal(fuse2); showToast('O painel abriu um compartimento: FUSÍVEL!',3000); }});
}
doorSign; // (placa já criada)
makeDoor(0,10,-1,'keypad',{label:'Porta do Laboratório',code:'1987'});

// SALA DE MÚSICA – piano -> Chave Verde
roomShell(30,-1,M.wood,'MÚSICA','#8e44ad','#fff');
{
  box(4.2,1.0,1.2,M.dark,30,0.5,-19.1,'prop');
  box(3.8,0.08,0.9,M.white,30,1.02,-18.1);
  addProp(28.1,31.9,-18.55,-17.65);
  const greenKey=spawnItem('green',30,1.7,-16.9,true);
  const cols=[0xe74c3c,0x3498db,0x2ecc71,0xf1c40f], nm=['vermelha','azul','verde','amarela'];
  const nodes=cols.map((c,i)=>{
    const x=30-1.2+i*0.8, z=-18.1, mat=ph(c);
    const m=soloBox(0.62,0.1,0.8,mat,x,1.1,z);
    const np=new THREE.Mesh(new THREE.PlaneGeometry(0.5,0.5),new THREE.MeshBasicMaterial({map:labelTex(String(i+1),128,128,'#'+('000000'+c.toString(16)).slice(-6),'#fff',FONT(90))}));
    np.rotation.x=-Math.PI/2; np.position.y=0.053; m.add(np);
    return {f:0,x,z:-17.0,y0:m.position.y,mesh:m,mat,color:c,label:'a tecla '+(i+1)+' ('+nm[i]+')'};
  });
  makeSeq({nodes,order:[2,0,3,1],verb:'Tocar',tones:[262,330,392,523],range:2.3,
    onSolve:()=>{ reveal(greenKey); showToast('Algo caiu de dentro do piano: a CHAVE VERDE!',3000); }});
  cyl(0.55,0.55,0.9,14,M.sofa2,30-7,0.45,-12,'prop'); cyl(0.4,0.4,0.7,14,M.sofa2,30-8.2,0.35,-14,'prop');
  box(0.6,1.5,0.6,M.dark,30-8.5,0.75,-6.8,'prop'); box(0.6,1.5,0.6,M.dark,30+8.5,0.75,-6.8,'prop');
  [-5,5].forEach(dx=>{ box(0.45,0.05,0.45,M.chair,30+dx,0.45,-12); box(0.45,0.5,0.05,M.chair,30+dx,0.72,-11.8); box(0.04,1.2,0.04,M.metal,30+dx,0.6,-14); box(0.5,0.35,0.04,M.dark,30+dx,1.3,-14); });
  noteI(30,-15.5,'Quadro de avisos','Aula de música: quem tocar as teclas na ordem da partitura ganha uma surpresa.\n\n"A partitura da turma ficou lá em cima, na sala de artes."','Ler o aviso na parede',2.6);
}

// CANTINA – cardápio (pista 4271)
roomShell(-30,1,M.tile,'CANTINA','#d68910','#fff');
[-7,7].forEach(dx=>[8,12,16].forEach(z=>{
  const x=-30+dx;
  box(2.4,0.08,0.9,M.woodL,x,0.78,z); box(2.2,0.7,0.3,M.woodM,x,0.35,z);
  box(2.4,0.06,0.35,M.woodDk,x,0.45,z-0.85); box(2.4,0.06,0.35,M.woodDk,x,0.45,z+0.85);
  addProp(x-1.2,x+1.2,z-1.05,z+1.05);
}));
box(8,1.1,1,M.counter,-30,0.55,19.2,'prop');
box(1,2.1,0.9,M.fridge,-36.5,1.05,19.3,'prop'); box(1,2.1,0.9,M.fridge,-23.5,1.05,19.3,'prop');
plane(4.6,1.5,labelMat('CARDÁPIO\nCombo do dia:\n4 pizzas · 2 sucos\n7 salgados · 1 pudim',512,170,'#2a1d0e','#ffe9a8',FONT(32,600),'#b8860b'),-30,2.9,19.77,Math.PI);
glow(-30,2.9,19.3,0xffe58a,1.2);
noteI(-30,17.7,'Cardápio da cantina','COMBO DO DIA\n• 4 pizzas\n• 2 sucos\n• 7 salgados\n• 1 pudim\n\nRabiscado embaixo: "Prof. Marta — usei o combo como senha da sala dos professores 😅"','Ler o cardápio',3.0);

// DIRETORIA – cofre 0612 -> Chave Vermelha
roomShell(-10,1,M.carpetRed,'DIRETORIA','#1e8449','#2ecc71');
{
  box(3.4,0.1,1.3,M.woodDk,-10,0.95,17); box(0.1,0.95,1.2,M.woodM,-11.6,0.47,17); box(0.1,0.95,1.2,M.woodM,-8.4,0.47,17);
  addProp(-11.7,-8.3,16.35,17.65);
  box(0.9,0.1,0.9,M.leather,-10,0.55,18.5); box(0.9,1.1,0.1,M.leather,-10,1.05,18.95);
  box(0.9,2.8,0.7,M.woodDk,-16.5,1.4,19.4,'prop');
  box(0.9,0.5,2.4,M.sofa,-17.7,0.25,10,'prop'); box(0.2,1,2.4,M.sofa,-18.2,0.8,10);
  [[-18.6,4.6],[-1.4,4.6]].forEach(([x,z])=>{ box(0.5,0.5,0.5,M.pot,x,0.25,z,'prop'); sph(0.5,M.plant,x,1,z); });
  plane(1.4,1,labelMat('DIPLOMA\nDE DIRETOR',256,180,'#f5e6b8','#4a2c14',FONT(30),'#b8860b'),-10,2.3,19.77,Math.PI);
  paper(-9.2,1.01,17);
  noteI(-9.2,17,'Diário do Diretor','"Se alguém invadir a escola de noite, vou caçá-lo pelos corredores. Ele ouve quem corre!\n\nTrancei a chave do portão no cofre, e a senha é a data da formatura (dia e mês, 4 números). Está marcada no calendário da sala dos professores."','Ler o diário sobre a mesa');
  // cofre
  const sx=-3.8, sz=19.5;
  box(1.1,1.1,0.6,M.dark,sx,1.0,sz,'prop');
  plane(0.9,0.9,M.pit,sx,1.0,19.19,Math.PI);
  const red=spawnItem('red',sx,1.0,18.9,true);
  const hinge=new THREE.Group(); hinge.position.set(sx-0.5,1.0,19.15); scene.add(hinge);
  const dm=new THREE.Mesh(new THREE.BoxGeometry(1.0,1.0,0.08),ph(0x4a5260,{shininess:50})); dm.position.set(0.5,0,0); hinge.add(dm);
  const kp=new THREE.Mesh(new THREE.BoxGeometry(0.2,0.3,0.03),new THREE.MeshBasicMaterial({color:0xff3b3b})); kp.position.set(0.8,0,-0.05); hinge.add(kp);
  const safe={open:false};
  addI({f:0,x:sx,z:18.4,range:2.3,dot:0.35,text:()=>safe.open?null:'Digitar o código do cofre',act:()=>
    openPad({title:'Cofre do Diretor',len:4,code:'0612',onOk:()=>{ safe.open=true; hinge.rotation.y=-1.9; kp.material.color.set(0x35ff7a); reveal(red); sfx.door(); showToast('O cofre abriu: CHAVE VERMELHA!',3000); }})});
  onReset(()=>{ safe.open=false; hinge.rotation.y=0; kp.material.color.set(0xff3b3b); });
  glow(sx,1.5,18.2,0xff8888,1.0);
}
makeDoor(0,-10,1,'key',{keyId:'green',label:'Porta da Diretoria'});

// ZELADORIA – corrente + fusível 1 + caixa de fusíveis
roomShell(10,1,M.tile,'ZELADORIA','#566573','#fff');
{
  [-8,8].forEach(dx=>box(0.5,2.4,5,M.woodM,10+dx,1.2,15,'prop'));
  box(1.2,0.9,0.7,M.metal,13.5,0.45,13,'prop'); // carrinho
  spawnItem('fuse',13.5,1.2,13,false,'fuseZ');
  box(1.6,0.9,0.8,M.woodDk,6.5,0.45,17,'prop');
  cyl(0.25,0.2,0.4,10,M.metal,6.2,1.1,17); cyl(0.25,0.2,0.4,10,M.metal,6.9,1.1,17);
  cyl(0.04,0.04,2.2,6,M.woodL,16.4,1.1,10); cyl(0.2,0.25,0.35,8,M.metal,16.4,0.17,10.3);
  paper(7,0.96,17);
  noteI(6.9,17,'Bilhete do zelador','Preciso de 3 fusíveis na caixa para religar a energia da escola.\n\nUm está aqui na Zeladoria. Os outros dois ficaram guardados no laboratório e na sala dos professores.\n\nAVISO: quando a energia voltar, o alarme dispara!','Ler o bilhete do zelador');
  // caixa de fusíveis
  box(1.3,1.6,0.25,M.dark,10,1.7,19.6);
  plane(1.1,0.35,labelMat('CAIXA DE FUSÍVEIS',384,110,'#c0392b','#fff',FONT(34)),10,2.3,19.45,Math.PI);
  const leds=[0,1,2].map(i=>{ const m=new THREE.MeshBasicMaterial({color:0x552222}); soloBox(0.16,0.16,0.04,m,9.6+i*0.4,1.8,19.45); return m; });
  const paintLeds=()=>leds.forEach((m,i)=>m.color.set(i<fuseIn?0x35ff7a:0x552222));
  addI({f:0,x:10,z:18.3,range:2.4,dot:0.35,text:()=>power?null:(inv.fuse>0?('Instalar fusíveis ('+inv.fuse+')'):('Caixa de fusíveis ('+fuseIn+'/3 instalados)')),act:()=>{
    if(inv.fuse>0){ fuseIn+=inv.fuse; inv.fuse=0; paintLeds(); refreshInv(); sfx.pick();
      if(fuseIn>=3) powerOn(); else showToast('Fusíveis instalados: '+fuseIn+'/3'); }
    else { sfx.fail(); showToast('Faltam fusíveis. Instalados: '+fuseIn+'/3'); }
  }});
  onReset(()=>{ paintLeds(); });
  glow(10,1.8,18.6,0xffe066,1.1);
}
makeDoor(0,10,1,'chain',{label:'Porta da Zeladoria'});

// SALA 2
roomShell(30,1,M.wood,'SALA 2','#7d3c98','#fff');
classroom(30,1,'Sala 2 – Aula de História');

// PORTÃO
const gateGroup=new THREE.Group(); scene.add(gateGroup);
{
  const gm=ph(0xc0392b,{emissive:0x5a0f0a,emissiveIntensity:.5});
  for(let z=-1.4;z<=1.41;z+=0.28){ const b=new THREE.Mesh(new THREE.BoxGeometry(0.1,3,0.1),gm); b.position.set(40,1.5,z); gateGroup.add(b); }
  [0.2,1.5,2.9].forEach(y=>{ const b=new THREE.Mesh(new THREE.BoxGeometry(0.14,0.12,GW),gm); b.position.set(40,y,0); gateGroup.add(b); });
}
const gate={type:'gate',f:0,cx:40,cz:0,label:'Portão da saída',open:false,progress:0,
  col:{minX:39.75,maxX:40.25,minZ:-1.5,maxZ:1.5,active:true,door:true},
  apply(p){ gateGroup.position.y=3.3*p; },
  reset(){ this.open=false; this.progress=0; this.col.active=true; this.apply(0); }};
walls[0].push(gate.col); doors.push(gate); onReset(()=>gate.reset());
addI({f:0,x:39,z:0,range:3.2,dot:0.25,text:()=>{
  if(gate.open) return null;
  if(!power) return 'Portão elétrico: sem energia';
  return inv.red ? 'Abrir o portão da saída' : 'Portão trancado – precisa da Chave Vermelha';
},act:()=>{
  if(!power){ sfx.fail(); showToast('O portão é elétrico. Restaure a energia primeiro!'); }
  else if(!inv.red){ sfx.fail(); showToast('Trancado! Falta a Chave Vermelha (está no cofre da Diretoria).'); }
  else { gate.open=true; gate.col.active=false; sfx.door(); showToast('O portão está abrindo! CORRA!'); }
}});

/* ---------- 1º ANDAR ---------- */
buildFloor(1);
poster(-22,-1,'PROIBIDO\nCORRER NA\nESCADA','#ffe0b2','#7a2f00');
poster(20,1,'BEBA\nÁGUA!','#d0f0ff','#0c4a6e');

// SALA DE ARTES – partitura (pista do piano)
roomShell(-30,-1,M.wood,'ARTES','#e67e22','#fff');
{
  [-6,0,6].forEach((dx,i)=>{ const x=-30+dx; box(1.0,1.8,0.08,M.easel,x,1.0,-9); box(0.9,0.7,0.02,M.paint[i%4],x,1.4,-8.95); box(0.05,1.6,0.6,M.easel,x-0.4,0.8,-8.6); box(0.05,1.6,0.6,M.easel,x+0.4,0.8,-8.6); addProp(x-0.55,x+0.55,-9.1,-8.4); });
  [-7,7].forEach(dx=>[13,16].forEach(dz=>{ box(2.2,0.08,1,M.woodL,-30+dx,0.8,-dz,'prop'); box(2,0.7,0.8,M.woodM,-30+dx,0.4,-dz); cyl(0.12,0.12,0.12,8,M.paint[(dz+dx+8)%4],-30+dx+0.5,0.9,-dz); }));
  plane(3.2,2.2,labelMat('PARTITURA DA TURMA\n\nTeclas do piano:\n3  →  1  →  4  →  2',512,320,'#fffdf2','#2a2418',FONT(40,700),'#c9a227'),-30,2.0,-19.78,0);
  glow(-30,2.0,-19.3,0xfff0b0,1.4);
  noteI(-30,-18.2,'Partitura da turma','PARTITURA DA TURMA\n\nTecle as teclas do piano da sala de música nesta ordem:\n\n3 → 1 → 4 → 2\n\n(As teclas são numeradas e coloridas.)','Ler a partitura na parede',3.0);
}
// INFORMÁTICA – lore
roomShell(-10,-1,M.tile,'INFORMÁTICA','#2471a3','#fff');
{
  [-7,-4.5,4.5,7].forEach(dx=>[8,11,14].forEach(dz=>{
    const x=-10+dx,z=-dz;
    box(1.4,0.06,0.7,M.white,x,0.75,z); box(0.05,0.72,0.6,M.metal,x-0.65,0.36,z); box(0.05,0.72,0.6,M.metal,x+0.65,0.36,z);
    box(0.6,0.4,0.05,M.dark,x,1.05,z-0.2); box(0.54,0.34,0.02,M.screen,x,1.05,z-0.17);
    box(0.45,0.05,0.45,M.chair,x,0.45,z+0.8); box(0.45,0.5,0.05,M.chair,x,0.72,z+1.02); box(0.08,0.43,0.08,M.metal,x,0.22,z+0.8);
    addProp(x-0.7,x+0.7,z-0.35,z+0.35);
  }));
  box(5,2.2,0.12,M.dark,-10,2.0,-19.7); box(4.7,1.9,0.02,M.screen,-10,2.0,-19.62);
  plane(4.5,1.7,labelMat('LOG DO SISTEMA\nENERGIA: DESLIGADA\nALARME: ARMADO\nGUARDAS: 2 EM PATRULHA',512,200,'#06122b','#6fe3ff',FONT(34,600)),-10,2.0,-19.6,0);
  noteI(-10,-18.2,'Terminal de segurança','LOG DO SISTEMA\n\n• Energia da escola: DESLIGADA (fusíveis removidos)\n• Alarme: ARMADO. Dispara quando a energia for religada.\n• Portão da saída: elétrico + chave vermelha.\n• Guardas: o Diretor (térreo) e o Inspetor (1º andar).\n• Os dois ouvem quem corre e seguem pela escada.','Ler o terminal',3.0);
}
// SALA DOS PROFESSORES – código 4271 -> Fusível 3 + calendário (0612)
roomShell(10,-1,M.carpetRed,'PROFESSORES','#a04000','#fff');
{
  [5,-5].forEach(dx=>box(1.4,0.8,6,M.woodM,10+dx,0.4,-12,'prop'));
  spawnItem('fuse',15,1.2,-12,false,'fuseP');
  box(0.9,0.5,2.4,M.sofa2,2.8,0.25,-9,'prop'); box(0.2,1,2.4,M.sofa2,2.3,0.8,-9);
  box(1.0,1.1,0.6,M.dark,10,0.55,-19.4,'prop'); cyl(0.12,0.12,0.3,8,M.white,10,1.3,-19.4);
  plane(1.8,2.2,labelMat('DEZEMBRO\n\nDom Seg Ter Qua Qui Sex Sáb\n\nFORMATURA\n06/12\n(círculo vermelho)',384,460,'#fffdf5','#7a1020',FONT(30,600),'#c0392b'),16,1.9,-19.78,0);
  noteI(16,-18.4,'Calendário de dezembro','DEZEMBRO\n\nDia 06 circulado de vermelho com a nota:\n"FORMATURA — não esquecer a câmera!"\n\n(Formatura: 06/12)','Ler o calendário',3.0);
}
makeDoor(1,10,-1,'keypad',{label:'Sala dos Professores',code:'4271'});
// BANHEIRO
roomShell(30,-1,M.tileSm,'BANHEIRO','#148f77','#fff');
{
  [-8,8].forEach(dx=>[7,9.5,12,14.5,17].forEach(dz=>box(1.8,2,0.06,M.woodL,30+dx,1,-dz,'prop')));
  [-6,-3,3,6].forEach(dx=>box(0.6,0.9,0.5,M.white,30+dx,0.45,-19.4,'prop'));
  plane(1.2,1,M.glass,30-6,1.9,-19.78,0); plane(1.2,1,M.glass,30+6,1.9,-19.78,0);
  plane(3.4,1.1,labelMat('ELE OUVE\nVOCÊ CORRER!',512,170,'#4a0f0f','#ff5a5a',FONT(52),'#8a1c1c'),30,2.3,-19.78,0);
}
// SALA 3
roomShell(-30,1,M.wood,'SALA 3','#2e86c1','#fff');
classroom(-30,1,'Sala 3 – Matemática:  2x + 3 = 7');
// SERVIDOR
roomShell(-10,1,M.tile,'SERVIDOR','#17202a','#5dade2');
{
  [-5,-7.5,5,7.5].forEach(dx=>[8,11,14,17].forEach((dz,k)=>{
    const x=-10+dx,z=dz;
    box(0.9,2.2,0.8,M.rack,x,1.1,z,'prop');
    for(let i=0;i<5;i++) box(0.12,0.05,0.02,(i+k+(dx>0?1:0))%3===0?M.ledR:M.ledG,x-0.25+i*0.12,1.4,z+(dx<0?0.41:-0.41));
  }));
  plane(3.2,1.3,labelMat('SERVIDOR DE SEGURANÇA\nCÂMERAS: OFFLINE',512,200,'#06122b','#5dade2',FONT(34,600)),-10,2.0,19.78,Math.PI);
}
// ENFERMARIA – pista dos interruptores
roomShell(10,1,M.tile,'ENFERMARIA','#c0392b','#fff');
{
  [-6,6].forEach(dx=>[9,13,17].forEach(dz=>{ box(1.1,0.45,2.2,M.metal,10+dx,0.3,dz,'prop'); box(1.0,0.2,2.0,M.bed,10+dx,0.62,dz); box(1.0,0.14,1.2,M.blanket,10+dx,0.76,dz+0.4); }));
  box(1.6,2,0.6,M.fridge,10,1,19.4,'prop');
  plane(0.9,0.9,labelMat('+',128,128,'#ffffff','#d62828',FONT(110)),10-4,2.2,19.78,Math.PI);
  plane(2.4,1.3,labelMat('QUADRO DE DISJUNTORES\nLABORATÓRIO\n1 ▲   2 ▼   3 ▲   4 ▲',512,260,'#fffdf5','#12306b',FONT(34,600),'#2b6cb0'),10+4,2.0,19.78,Math.PI);
  glow(14,2.0,19.2,0xfff0b0,1.3);
  noteI(14,18.3,'Quadro de disjuntores','QUADRO DE DISJUNTORES — LABORATÓRIO\n\nPosição correta dos interruptores (▲ = para cima, ▼ = para baixo):\n\n1 ▲   2 ▼   3 ▲   4 ▲','Ler o quadro de disjuntores',3.0);
}
// GRÊMIO
roomShell(30,1,M.carpetNavy,'GRÊMIO','#7d3c98','#fff');
{
  [[-7,10],[-7,14],[7,10],[7,14]].forEach(([dx,dz],i)=>{ box(0.9,0.5,2.4,i<2?M.sofa:M.sofa2,30+dx,0.25,dz,'prop'); box(0.2,1,2.4,i<2?M.sofa:M.sofa2,30+dx+(dx<0?-0.5:0.5),0.8,dz); });
  box(2.7,0.08,1.5,M.gym,30,0.8,16.5,'prop'); box(0.1,0.8,1.4,M.dark,30,0.4,16.5); box(0.02,0.18,1.55,M.white,30,0.9,16.5);
  box(3,1.6,0.1,M.dark,30,2.1,19.7); box(2.8,1.4,0.02,M.screen,30,2.1,19.63);
  box(1,2,0.8,M.dark,36.8,1,18.3,'prop');
}
flushBatches();

/* ================= EXTERIOR ================= */
{
  const g=new THREE.PlaneGeometry(60,100); g.rotateX(-Math.PI/2); scaleUV(g,30,50); g.translate(70,-0.02,0);
  const m=new THREE.Mesh(g,M.grass); scene.add(m);
  const yel=ph(0xf4b400), blk=ph(0x1a1a1a);
  const mk=(w,h,d,mat,x,y,z)=>{ const b=new THREE.Mesh(new THREE.BoxGeometry(w,h,d),mat); b.position.set(x,y,z); scene.add(b); return b; };
  mk(2.8,2.4,9,yel,52,1.6,-8); mk(2.82,0.3,9.02,blk,52,1.35,-8);
  for(let i=0;i<5;i++) mk(2.84,0.7,1.0,ph(0x9fd3ff,{emissive:0x2a4a6a,emissiveIntensity:.4}),52,2.1,-11.5+i*1.7);
  [-10.5,-5.5].forEach(z=>[50.6,53.4].forEach(x=>{ const w=new THREE.Mesh(new THREE.CylinderGeometry(.55,.55,.3,14),blk); w.rotation.z=Math.PI/2; w.position.set(x,.55,z); scene.add(w); }));
  const trunk=ph(0x5a3a1c), leaf=ph(0x1f6b34);
  [[46,12],[56,18],[62,6],[50,-20],[60,-14],[46,25],[66,-26],[58,28],[44,-12],[70,10],[68,-4]].forEach(([x,z])=>{
    const t=new THREE.Mesh(new THREE.CylinderGeometry(.3,.4,2.2,6),trunk); t.position.set(x,1.1,z); scene.add(t);
    const c=new THREE.Mesh(new THREE.ConeGeometry(1.8,5,8),leaf); c.position.set(x,4.2,z); scene.add(c);
  });
  mk(0.15,5,0.15,M.metal,44,2.5,-4);
  const lampM=new THREE.Mesh(new THREE.SphereGeometry(.35,10,8),new THREE.MeshBasicMaterial({color:0xfff2b0})); lampM.position.set(44,5.1,-4); scene.add(lampM);
  const lampL=new THREE.PointLight(0xffe7a0,0.9,24,1); lampL.position.set(44,5,-2); scene.add(lampL);
  const moon=new THREE.Mesh(new THREE.SphereGeometry(4,16,12),new THREE.MeshBasicMaterial({color:0xf5f1d8,fog:false})); moon.position.set(80,40,-30); scene.add(moon);
  const pts=[]; for(let i=0;i<500;i++){ const a=Math.random()*Math.PI*2, b=Math.random()*1.2+0.1, r=85; pts.push(Math.cos(a)*Math.cos(b)*r, Math.sin(b)*r*0.9+5, Math.sin(a)*Math.cos(b)*r); }
  const sg=new THREE.BufferGeometry(); sg.setAttribute('position',new THREE.Float32BufferAttribute(pts,3));
  scene.add(new THREE.Points(sg,new THREE.PointsMaterial({color:0xffffff,size:1.6,sizeAttenuation:false,fog:false})));
}

/* ================= JOGADOR / ESTADO ================= */
const player = { f:0, x:-31, z:0, yaw:-Math.PI/2, pitch:0, stamina:100, exhausted:false, hidden:null, moving:false, running:false, bob:0 };
let hideYaw = 0, state = 'menu', elapsed = 0, time = 0, noteOpen = false, pad = null;
const keys = {}, joy = {x:0,y:0};
let touchRun = false;

function resolve(p,r,lists){
  for(let it=0;it<2;it++){
    for(const list of lists){
      for(const c of list){
        if(!c.active) continue;
        const qx=Math.max(c.minX,Math.min(p.x,c.maxX)), qz=Math.max(c.minZ,Math.min(p.z,c.maxZ));
        const dx=p.x-qx, dz=p.z-qz, d2=dx*dx+dz*dz;
        if(d2<r*r){
          if(d2>1e-8){ const d=Math.sqrt(d2), k=(r-d)/d; p.x+=dx*k; p.z+=dz*k; }
          else {
            const l=p.x-c.minX, rr=c.maxX-p.x, t=p.z-c.minZ, b=c.maxZ-p.z, m=Math.min(l,rr,t,b);
            if(m===l) p.x=c.minX-r; else if(m===rr) p.x=c.maxX+r; else if(m===t) p.z=c.minZ-r; else p.z=c.maxZ+r;
          }
        }
      }
    }
  }
}
function hasLOS(ax,az,bx,bz,f){
  const dx=bx-ax, dz=bz-az, d=Math.hypot(dx,dz), n=Math.ceil(d/0.3), W=walls[f];
  for(let i=1;i<n;i++){
    const t=i/n, x=ax+dx*t, z=az+dz*t;
    for(const c of W){ if(c.active && x>c.minX && x<c.maxX && z>c.minZ && z<c.maxZ) return false; }
  }
  return true;
}

/* ================= GRAFO DE PATRULHA ================= */
const nodes=[], nid={}, edges=[];
function addNode(f,x,z){ nodes.push({f,x,z}); nid[f+':'+x+':'+z]=nodes.length-1; return nodes.length-1; }
[0,1].forEach(f=>{
  const xs=[-34,-30,-20,-10,0,10,20,30,38];
  xs.forEach(x=>addNode(f,x,0));
  for(let i=0;i<xs.length-1;i++) edges.push({a:nid[f+':'+xs[i]+':0'],b:nid[f+':'+xs[i+1]+':0']});
  [-30,-10,10,30].forEach(cx=>[-1,1].forEach(side=>{
    const n=addNode(f,cx,side*9);
    edges.push({a:nid[f+':'+cx+':0'],b:n,door:doorAt[f+':'+cx+':'+side]});
  }));
});
edges.push({a:nid['0:-34:0'],b:nid['1:-34:0']});
function adj(i){
  const r=[];
  for(const e of edges){ if(e.door && !e.door.open) continue; if(e.a===i) r.push(e.b); else if(e.b===i) r.push(e.a); }
  return r;
}
function bfs(s,t){
  if(s===t) return [s];
  const prev=new Array(nodes.length).fill(-1), seen=new Set([s]), q=[s];
  while(q.length){
    const u=q.shift();
    for(const v of adj(u)){
      if(seen.has(v)) continue;
      seen.add(v); prev[v]=u;
      if(v===t){ const p=[t]; let c=t; while(c!==s){ c=prev[c]; p.unshift(c);} return p; }
      q.push(v);
    }
  }
  return null;
}
function nearestNode(x,z,f,los){
  let best=-1,bd=1e9,any=-1,ba=1e9;
  nodes.forEach((n,i)=>{
    if(n.f!==f) return;
    const d=Math.hypot(n.x-x,n.z-z);
    if(d<ba){ba=d;any=i;}
    if(d<bd && (!los || hasLOS(x,z,n.x,n.z,f))){bd=d;best=i;}
  });
  return best>=0?best:any;
}
function planPath(from,to){
  const s=nearestNode(from.x,from.z,from.f,true), e=nearestNode(to.x,to.z,to.f,true);
  const p=bfs(s,e)||[s];
  const pts=p.map(i=>({x:nodes[i].x,z:nodes[i].z,f:nodes[i].f}));
  pts.push({x:to.x,z:to.z,f:to.f});
  return pts;
}

/* ================= GUARDAS ================= */
function buildGuardMesh(sc){
  const g=new THREE.Group();
  const suit=ph(sc.suit), shirt=ph(0xf2f2f2), tie=ph(sc.tie);
  const eye=new THREE.MeshBasicMaterial({color:0xff2a2a});
  const mk=(w,h,d,mat,x,y,z)=>{ const m=new THREE.Mesh(new THREE.BoxGeometry(w,h,d),mat); m.position.set(x,y,z); g.add(m); return m; };
  mk(.85,.95,.46,suit,0,1.4,0); mk(.28,.8,.02,shirt,0,1.45,.235); mk(.1,.62,.02,tie,0,1.42,.25);
  mk(.52,.5,.5,M.skin,0,2.12,0);
  [-.13,.13].forEach(x=>{ mk(.1,.07,.02,eye,x,2.17,.26); const b=mk(.16,.04,.02,suit,x,2.26,.26); b.rotation.z=x>0?-.35:.35; });
  mk(.2,.04,.02,suit,0,2.0,.26);
  if(sc.hat){ mk(.6,.1,.6,ph(sc.hat),0,2.42,0); mk(.6,.05,.3,ph(sc.hat),0,2.4,.38); }
  if(sc.vest) mk(.88,.5,.5,ph(0xf1c40f,{emissive:0x6a5500,emissiveIntensity:.4}),0,1.55,0);
  const arms=[], legs=[];
  [-1,1].forEach(s=>{
    const p=new THREE.Group(); p.position.set(s*.58,1.8,0);
    const a=new THREE.Mesh(new THREE.BoxGeometry(.24,.85,.24),suit); a.position.y=-.42; p.add(a);
    const h=new THREE.Mesh(new THREE.BoxGeometry(.22,.2,.22),M.skin); h.position.y=-.9; p.add(h);
    g.add(p); arms.push(p);
    const lp=new THREE.Group(); lp.position.set(s*.22,.92,0);
    const l=new THREE.Mesh(new THREE.BoxGeometry(.32,.92,.32),suit); l.position.y=-.46; lp.add(l);
    g.add(lp); legs.push(lp);
  });
  const baton=new THREE.Mesh(new THREE.CylinderGeometry(.045,.045,.7,8),ph(0x444444)); baton.position.set(0,-.95,.2); baton.rotation.x=Math.PI/2.4; arms[1].add(baton);
  g.userData={arms,legs}; scene.add(g); return g;
}
function createGuard(o){
  const g={name:o.name,f:o.f,x:o.x,z:o.z,yaw:Math.PI/2,state:'patrol',path:[],wait:2,last:{x:0,z:0,f:0},lost:0,checked:false,
    speedNow:0,swing:0,stepT:0,hb:0,danger:0,patrol:o.patrol,chase:o.chase,start:{f:o.f,x:o.x,z:o.z},
    mesh:buildGuardMesh(o.scheme),light:new THREE.PointLight(0xff2020,0,9,1)};
  scene.add(g.light); return g;
}
const guards = [
  createGuard({name:'Diretor',f:0,x:20,z:0,patrol:2.3,chase:4.7,scheme:{suit:0x15171c,tie:0xd01a2b}}),
  createGuard({name:'Inspetor',f:1,x:20,z:0,patrol:2.1,chase:4.5,scheme:{suit:0x3b4552,tie:0x2d6a4f,hat:0x2d6a4f,vest:true}})
];
function resetGuards(){
  guards.forEach(g=>{ Object.assign(g,{f:g.start.f,x:g.start.x,z:g.start.z,state:'patrol',path:[],wait:2,lost:0,checked:false,danger:0,hb:0}); });
}
function moveG(g,tx,tz,speed,dt){
  const dx=tx-g.x, dz=tz-g.z, d=Math.hypot(dx,dz);
  if(d<0.15) return true;
  const st=Math.min(d,speed*dt);
  g.x+=dx/d*st; g.z+=dz/d*st;
  resolve(g,0.45,[walls[g.f]]);
  const ty=Math.atan2(dx,dz); let da=ty-g.yaw;
  while(da>Math.PI) da-=2*Math.PI; while(da<-Math.PI) da+=2*Math.PI;
  g.yaw+=da*Math.min(1,dt*8);
  g.speedNow=speed;
  return d<=st+0.05;
}
function followPath(g,speed,dt){
  if(!g.path.length) return false;
  const t=g.path[0];
  if(t.f!==g.f){ g.f=t.f; g.x=t.x; g.z=t.z; g.path.shift(); return true; }
  if(moveG(g,t.x,t.z,speed,dt)) g.path.shift();
  return true;
}
function pickPatrol(g){
  const here=nearestNode(g.x,g.z,g.f,true);
  let target=-1;
  if(Math.random()<0.3 && !player.hidden){
    const t=nearestNode(player.x,player.z,player.f,false);
    if(t!==here && bfs(here,t)) target=t;
  }
  if(target<0){
    const opts=[]; for(let i=0;i<nodes.length;i++){ if(i!==here && bfs(here,i)) opts.push(i); }
    target=opts.length?opts[Math.floor(Math.random()*opts.length)]:here;
  }
  g.path=planPath({x:g.x,z:g.z,f:g.f},{x:nodes[target].x,z:nodes[target].z,f:nodes[target].f});
}
function updateGuard(g,dt){
  const sameF=(g.f===player.f);
  const dx=player.x-g.x, dz=player.z-g.z, dist=sameF?Math.hypot(dx,dz):999;
  g.speedNow=0;
  const am=alarm?1.22:1;
  let sees=false;
  if(sameF && !player.hidden && dist<22*(alarm?1.2:1)){
    const fx=Math.sin(g.yaw), fz=Math.cos(g.yaw);
    const dot=(dx*fx+dz*fz)/(dist||1);
    const noisy=player.moving && player.running && dist<11;
    if((dist<4.5 || dot>0.45 || noisy) && hasLOS(g.x,g.z,player.x,player.z,g.f)) sees=true;
  }
  if(sees){
    if(g.state!=='chase'){ g.state='chase'; sfx.sting(); showToast('⚠️ O '+g.name+' viu você! Corra ou se esconda!'); }
    g.last={x:player.x,z:player.z,f:player.f}; g.lost=0;
  }
  if(g.state==='chase'){
    const sp=(g.chase+keysGot()*0.12)*am;
    if(sees) moveG(g,player.x,player.z,sp,dt);
    else{
      g.lost+=dt;
      if(g.lost>0.45){ g.state='search'; g.path=planPath({x:g.x,z:g.z,f:g.f},g.last); g.wait=0; g.checked=false; }
      else if(g.last.f===g.f) moveG(g,g.last.x,g.last.z,sp,dt);
    }
  } else if(g.state==='search'){
    if(!followPath(g,(g.patrol+1.2)*am,dt)){
      g.wait+=dt; g.yaw+=dt*1.6;
      if(player.hidden && sameF && dist<2.7 && g.wait>1.3 && !g.checked){
        g.checked=true;
        if(Math.random()<0.45){ gameOver('O '+g.name+' abriu o armário onde você estava escondido!'); return; }
        showToast('Ele quase abriu o armário… e foi embora.');
      }
      if(g.wait>3.5){ g.state='patrol'; g.wait=0; g.path=[]; }
    }
  } else {
    if(!followPath(g,g.patrol*am,dt)){
      g.wait+=dt; g.yaw+=dt*0.7;
      if(g.wait>1.2){ g.wait=0; pickPatrol(g); }
    }
  }
  if(sameF && !player.hidden && dist<1.1 && state==='playing'){ gameOver('O '+g.name+' te pegou! Você ficou de detenção.'); return; }

  const ud=g.mesh.userData;
  g.swing+=dt*g.speedNow*2.6;
  const amp=g.speedNow>0?Math.min(.9,g.speedNow*.2):0;
  ud.arms[0].rotation.x=Math.sin(g.swing)*amp; ud.arms[1].rotation.x=-Math.sin(g.swing)*amp;
  ud.legs[0].rotation.x=-Math.sin(g.swing)*amp; ud.legs[1].rotation.x=Math.sin(g.swing)*amp;
  if(g.state==='chase') ud.arms[1].rotation.x=-1.2;
  g.mesh.position.set(g.x,g.f*FH,g.z); g.mesh.rotation.y=g.yaw;
  g.light.position.set(g.x,g.f*FH+2.5,g.z);
  g.light.intensity=g.state==='chase'?1.4:0;
  if(g.speedNow>0){
    g.stepT-=dt;
    if(g.stepT<=0){ g.stepT=0.9/(g.speedNow/2.3); if(dist<26) tone(70+Math.random()*12,.09,'triangle',clamp(0.9/(dist*0.55+1),0,.4)); }
  }
  if(g.state==='chase' && sameF){
    g.hb-=dt;
    if(g.hb<=0){ g.hb=.62; tone(58,.11,'sine',.4); setTimeout(()=>tone(50,.1,'sine',.3),160); }
  }
  const tgt=(g.state==='chase'&&sameF)?1:((dist<7&&hasLOS(g.x,g.z,player.x,player.z,g.f)&&!player.hidden)?.3:0);
  g.danger+=(tgt-g.danger)*Math.min(1,dt*4);
}
function keysGot(){ return (flags.blue?1:0)+(flags.green?1:0)+(flags.red?1:0); }

/* ================= ENERGIA / ALARME ================= */
function powerOn(){
  power=true; alarm=true;
  sfx.power(); setTimeout(startSiren,600);
  hemi.intensity=0.95; lightMat.color.set(0xfffbe0);
  flickMats.forEach(m=>m.color.set(0xfffbe0));
  showToast('ENERGIA RESTAURADA! O ALARME DISPAROU! Corra para o portão!',4200);
}
function resetPower(){
  power=false; alarm=false; stopSiren();
  hemi.intensity=0.5; lightMat.color.set(0x20232e);
}
onReset(()=>{ fuseIn=0; Object.keys(inv).forEach(k=>inv[k]=0); Object.keys(flags).forEach(k=>flags[k]=false); resetPower(); });

/* ================= INTERAÇÃO ================= */
function findTarget(){
  if(player.hidden) return {text:'Sair do armário',act:exitLocker};
  const fx=-Math.sin(player.yaw), fz=-Math.cos(player.yaw);
  let best=null, bs=1e9;
  for(const o of I){
    if(o.f!==player.f) continue;
    const dx=o.x-player.x, dz=o.z-player.z, d=Math.hypot(dx,dz);
    if(d>o.range) continue;
    const dot=(dx*fx+dz*fz)/(d||1);
    if(dot<o.dot && d>1.1) continue;
    const t=o.text(); if(!t) continue;
    const sc=d+(1-dot)*4;
    if(sc<bs){ bs=sc; best={text:t,act:o.act}; }
  }
  return best;
}
function interact(){
  if(state!=='playing' || pad) return;
  if(noteOpen){ closeNote(); return; }
  const t=findTarget(); if(t) t.act();
}
function enterLocker(l){
  player.hidden=l; player.x=l.x; player.z=l.z;
  hideYaw=Math.atan2(-l.nx,-l.nz); player.yaw=hideYaw; player.pitch=0;
  hideOv.style.display='block'; sfx.hide();
  showToast('Shhh… você está escondido. Aperte E para sair.',2000);
}
function exitLocker(){
  const l=player.hidden; if(!l) return;
  player.hidden=null; player.x=l.x+l.nx*1.25; player.z=l.z+l.nz*1.25;
  hideOv.style.display='none'; sfx.hide();
}
function changeFloor(nf){
  fadeEl.classList.add('on');
  setTimeout(()=>fadeEl.classList.remove('on'),260);
  player.f=nf; player.x=-33.4; player.z=0; player.yaw=-Math.PI/2; player.pitch=0;
  tone(200,.3,'triangle',.1,100);
  guards.forEach(g=>{ if(g.state==='chase'){ g.state='search'; g.wait=0; g.checked=false; g.path=planPath({x:g.x,z:g.z,f:g.f},{x:-34,z:0,f:nf}); } });
  showToast(FLOORNAME[nf],1200);
}

/* ----- notas e teclado ----- */
const noteEl=$('note');
function showNote(title,body){
  $('noteT').textContent=title; $('noteB').textContent=body;
  $('noteH').textContent=isTouch?'Toque para fechar':'Pressione E para fechar';
  noteEl.classList.remove('hidden'); noteOpen=true; sfx.click();
}
function closeNote(){ noteEl.classList.add('hidden'); noteOpen=false; }
noteEl.addEventListener('click',closeNote);

const padEl=$('pad');
(function buildPad(){
  const grid=$('padGrid');
  ['1','2','3','4','5','6','7','8','9','⌫','0','OK'].forEach(k=>{
    const b=document.createElement('button'); b.textContent=k; if(k==='OK') b.className='ok';
    b.addEventListener('click',()=>padKey(k)); grid.appendChild(b);
  });
})();
function padRender(){
  const disp=$('padDisp'); disp.innerHTML='';
  for(let i=0;i<pad.len;i++){ const s=document.createElement('span'); s.textContent=pad.val[i]||''; disp.appendChild(s); }
}
function openPad(cfg){
  pad={title:cfg.title,len:cfg.len,code:cfg.code,onOk:cfg.onOk,val:''};
  $('padTitle').textContent=cfg.title+' – digite o código'; $('padMsg').textContent='';
  padRender(); padEl.classList.remove('hidden');
  if(document.pointerLockElement) document.exitPointerLock();
}
function closePad(){ padEl.classList.add('hidden'); pad=null; lockPointer(); }
function padKey(k){
  if(!pad) return;
  if(k==='⌫'){ pad.val=pad.val.slice(0,-1); sfx.key(); }
  else if(k==='OK'){ padSubmit(); return; }
  else if(pad.val.length<pad.len){ pad.val+=k; sfx.key(); }
  padRender();
  if(pad.val.length===pad.len) setTimeout(()=>{ if(pad) padSubmit(); },180);
}
function padSubmit(){
  if(pad.val===pad.code){ const ok=pad.onOk; sfx.solve(); closePad(); ok(); }
  else{
    sfx.fail(); $('padMsg').textContent='Código incorreto.'; pad.val=''; padRender();
    padEl.classList.remove('shake'); void padEl.offsetWidth; padEl.classList.add('shake');
  }
}

/* ================= HUD ================= */
const objEl=$('objText'), stepEl=$('objStep'), timerEl=$('timer'), promptEl=$('prompt'), toastEl=$('toast'), stEl=$('st'),
  dangerEl=$('danger'), alarmEl=$('alarmOv'), hideOv=$('hideOv'), hud=$('hud'), touchEl=$('touch'), btnAct=$('btnAct'), fadeEl=$('fade');
let toastT=null, lastPrompt='';
function showToast(msg,ms){
  toastEl.textContent=msg; toastEl.classList.add('show');
  clearTimeout(toastT); toastT=setTimeout(()=>toastEl.classList.remove('show'),ms||2600);
}
const INVUI=[['blue','Chave azul','#2b8cff'],['green','Chave verde','#2ecc71'],['red','Chave vermelha','#ff3b3b'],['cutter','Alicate','#ff8c1a'],['fuse','Fusíveis','#ffe066']];
$('inv').innerHTML=INVUI.map(([id,n,c])=>'<span class="kslot" id="k-'+id+'" style="color:'+c+'"><span class="kdot"></span><span class="kt">'+n+'</span></span>').join('');
function refreshInv(){
  INVUI.forEach(([id])=>{
    const el=$('k-'+id); let on=inv[id]>0;
    if(id==='fuse'){ const t=inv.fuse+fuseIn; on=t>0; el.querySelector('.kt').textContent='Fusíveis '+t+'/3'; }
    el.classList.toggle('on',on);
  });
}
const dBiblio=doorAt['0:-10:-1'], dLab=doorAt['0:10:-1'], dZel=doorAt['0:10:1'], dDir=doorAt['0:-10:1'], dProf=doorAt['1:10:-1'];
const TASKS=[
  {done:()=>flags.blue,   t:'Pegue a Chave Azul na Sala 1 (primeira porta à esquerda do corredor).'},
  {done:()=>flags.cutter, t:'Abra a Biblioteca (porta azul) e resolva o enigma dos livros.'},
  {done:()=>dZel.open,    t:'Corte a corrente da Zeladoria (lado direito do corredor, depois do meio).'},
  {done:()=>flags.fuseZ,  t:'Pegue o fusível dentro da Zeladoria.'},
  {done:()=>flags.green,  t:'Toque o piano da Sala de Música (extremo leste, à esquerda). A partitura está em outro andar.'},
  {done:()=>dLab.open,    t:'Abra o Laboratório. Procure um número importante no corredor.'},
  {done:()=>flags.fuseL,  t:'Acerte os interruptores do Laboratório. A pista está no andar de cima.'},
  {done:()=>dProf.open,   t:'Entre na Sala dos Professores (1º andar). O código está na Cantina.'},
  {done:()=>flags.fuseP,  t:'Pegue o fusível na Sala dos Professores.'},
  {done:()=>flags.red,    t:'Abra o cofre da Diretoria (porta verde). A senha é uma data.'},
  {done:()=>power,        t:'Instale os 3 fusíveis na caixa da Zeladoria.'},
  {done:()=>gate.open,    t:'O alarme tocou! Abra o portão no fim do corredor (leste) e fuja!'}
];
function fmt(t){ const m=Math.floor(t/60), s=Math.floor(t%60); return String(m).padStart(2,'0')+':'+String(s).padStart(2,'0'); }
let promptDirty=true;
function updateHUD(dt){
  timerEl.textContent=FLOORNAME[player.f]+' · '+fmt(elapsed);
  let idx=TASKS.findIndex(t=>!t.done());
  if(idx<0) idx=TASKS.length-1;
  stepEl.textContent='Etapa '+(idx+1)+' de '+TASKS.length;
  const tt=TASKS[idx].t; if(objEl.textContent!==tt) objEl.textContent=tt;
  stEl.style.width=player.stamina+'%'; stEl.classList.toggle('low',player.exhausted||player.stamina<25);
  const t=findTarget(); const txt=t?t.text:'';
  if(txt!==lastPrompt){
    lastPrompt=txt;
    if(txt){ promptEl.innerHTML=(isTouch?'':'<b>E</b>')+txt.replace(/</g,'&lt;'); promptEl.classList.add('show'); }
    else promptEl.classList.remove('show');
  }
  btnAct.classList.toggle('glow',!!t);
  let dg=0; guards.forEach(g=>{ dg=Math.max(dg,g.danger); });
  dangerEl.style.opacity=(dg*(0.75+0.25*Math.sin(time*9))).toFixed(2);
  alarmEl.style.opacity=alarm?(0.35+0.65*Math.max(0,Math.sin(time*5))).toFixed(2):0;
}

/* ----- minimapa ----- */
const mm=$('mm'), mctx=mm.getContext('2d'); let mmT=0;
mm.addEventListener('click',()=>mm.classList.toggle('big'));
function drawMap(){
  const W=mm.width,Hh=mm.height, sx=W/86, sz=Hh/46;
  const X=x=>(x+43)*sx, Zc=z=>(z+23)*sz;
  mctx.clearRect(0,0,W,Hh);
  mctx.fillStyle='rgba(8,12,28,.78)'; mctx.fillRect(0,0,W,Hh);
  walls[player.f].forEach(c=>{
    if(!c.active) return;
    mctx.fillStyle=c.door?'#ff7b7b':'#8fa0e8';
    mctx.fillRect(X(c.minX),Zc(c.minZ),Math.max(2,(c.maxX-c.minX)*sx),Math.max(2,(c.maxZ-c.minZ)*sz));
  });
  mctx.fillStyle='#5dff9b'; mctx.font='bold '+Math.round(Hh*0.075)+'px sans-serif'; mctx.textAlign='center';
  mctx.fillText(player.f===0?'▲ escada':'▼ escada',X(-37.5),Zc(0)+4);
  if(player.f===0) mctx.fillText('SAÍDA',X(41),Zc(0)-8);
  mctx.fillStyle='#fff'; mctx.textAlign='left'; mctx.fillText(FLOORNAME[player.f],6,Hh-8);
  const px=X(player.x), pz=Zc(player.z), a=-player.yaw;
  mctx.save(); mctx.translate(px,pz); mctx.rotate(Math.atan2(-Math.cos(player.yaw),-Math.sin(player.yaw))*-1+Math.PI/2*0);
  mctx.restore();
  const fx=-Math.sin(player.yaw), fz=-Math.cos(player.yaw);
  mctx.fillStyle='#ffd23f'; mctx.beginPath();
  mctx.moveTo(px+fx*sx*2.2,pz+fz*sz*2.2);
  mctx.lineTo(px-fz*sx*1.2-fx*sx*1.2,pz+fx*sz*1.2-fz*sz*1.2);
  mctx.lineTo(px+fz*sx*1.2-fx*sx*1.2,pz-fx*sz*1.2-fz*sz*1.2);
  mctx.closePath(); mctx.fill();
}

/* ================= TELAS ================= */
const screenEl=$('screen');
function showScreen(o){
  $('sTitle').textContent=o.title; $('sText').textContent=o.text||'';
  $('sExtra').innerHTML=o.extra||''; $('sBtn').textContent=o.btn; $('sBtn').onclick=o.action;
  const b2=$('sBtn2');
  if(o.btn2){ b2.textContent=o.btn2; b2.onclick=o.action2; b2.classList.remove('hidden'); } else b2.classList.add('hidden');
  screenEl.classList.remove('hidden');
  hud.classList.add('hidden'); touchEl.classList.add('hidden'); dangerEl.style.opacity=0; alarmEl.style.opacity=0;
}
function hideScreen(){ screenEl.classList.add('hidden'); hud.classList.remove('hidden'); if(isTouch) touchEl.classList.remove('hidden'); }
const CONTROLS = isTouch
 ? '<div class="row"><kbd>Joystick</kbd> esquerda da tela: andar</div><div class="row"><kbd>Arrastar</kbd> direita da tela: olhar</div><div class="row"><kbd>E</kbd> interagir &nbsp;<kbd>Run</kbd> correr &nbsp;<kbd>F</kbd> lanterna &nbsp;<kbd>Mapa</kbd> toque no minimapa</div>'
 : '<div class="row"><kbd>W A S D</kbd> andar &nbsp;<kbd>Mouse</kbd> olhar (clique na tela)</div><div class="row"><kbd>Shift</kbd> correr &nbsp;<kbd>E</kbd> interagir/esconder &nbsp;<kbd>F</kbd> lanterna</div><div class="row"><kbd>M</kbd> mapa grande &nbsp;<kbd>Esc</kbd> pausar</div>';
function bestTime(){ try{ return parseFloat(localStorage.getItem('fuga_best')||'0')||0; }catch(e){ return 0; } }
function menuScreen(){
  state='menu';
  const b=bestTime();
  showScreen({
    title:'Fuga da Escola 3D',
    text:'Você ficou preso na escola à noite. Dois guardas patrulham os dois andares. Resolva os enigmas, ache as chaves, religue a energia e fuja pelo portão.',
    extra:CONTROLS+'<div class="row" style="margin-top:8px;color:var(--card-sub)">Dicas: os guardas ouvem quem corre. Esconda-se nos armários. Leia tudo o que brilhar.'+(b?' Melhor tempo: '+fmt(b)+'.':'')+'</div>',
    btn:'Jogar', action:startGame
  });
}
function startGame(){ initAudio(); resetAll(); state='playing'; hideScreen(); lockPointer(); showToast('Você está preso na escola! Explore e resolva os enigmas.',3500); }
function lockPointer(){ if(isTouch) return; try{ const p=canvas.requestPointerLock(); if(p&&p.catch) p.catch(()=>{}); }catch(e){} }
function gameOver(msg){
  if(state!=='playing') return;
  state='over'; sfx.lose(); closeNote();
  if(pad){ padEl.classList.add('hidden'); pad=null; }
  if(document.pointerLockElement) document.exitPointerLock();
  setTimeout(()=>showScreen({title:'Pego!',text:msg,extra:'Tempo: '+fmt(elapsed)+'. Você mantém chaves, itens e enigmas resolvidos se continuar.',btn:'Continuar daqui',action:respawn,btn2:'Recomeçar do zero',action2:startGame}),450);
}
function winGame(){
  if(state!=='playing') return;
  state='won'; sfx.win(); stopSiren();
  if(document.pointerLockElement) document.exitPointerLock();
  let rec='';
  const b=bestTime(); if(!b||elapsed<b){ try{ localStorage.setItem('fuga_best',String(elapsed)); }catch(e){} rec=' Novo recorde!'; }
  showScreen({title:'Você fugiu!',text:'Você escapou da escola antes de os guardas te alcançarem.',extra:'Tempo: '+fmt(elapsed)+'.'+rec,btn:'Jogar de novo',action:startGame});
}
function pauseGame(){
  if(state!=='playing') return;
  state='paused';
  showScreen({title:'Pausado',text:'Os guardas também pararam. Por enquanto.',extra:CONTROLS,btn:'Continuar',action:()=>{ state='playing'; hideScreen(); lockPointer(); }});
}
let wasLocked=false;
document.addEventListener('pointerlockchange',()=>{
  const now=document.pointerLockElement===canvas;
  if(now) wasLocked=true;
  else if(wasLocked){ wasLocked=false; if(state==='playing' && !pad) pauseGame(); }
});
document.addEventListener('visibilitychange',()=>{ if(document.hidden && state==='playing') pauseGame(); });
canvas.addEventListener('click',()=>{ if(state==='playing' && !pad) lockPointer(); });

function resetAll(){
  resetHooks.forEach(fn=>fn());
  refreshInv(); closeNote();
  respawnPlayer();
  elapsed=0;
}
function respawnPlayer(){
  player.f=0; player.x=-31; player.z=0; player.yaw=-Math.PI/2; player.pitch=0; player.stamina=100; player.exhausted=false; player.hidden=null;
  hideOv.style.display='none'; lastPrompt=''; toastEl.classList.remove('show');
  resetGuards();
}
function respawn(){ respawnPlayer(); state='playing'; hideScreen(); lockPointer(); }

/* ================= ENTRADA ================= */
window.addEventListener('keydown',e=>{
  if(pad){
    if(/^[0-9]$/.test(e.key)) padKey(e.key);
    else if(e.key==='Backspace') padKey('⌫');
    else if(e.key==='Enter') padKey('OK');
    else if(e.key==='Escape') closePad();
    e.preventDefault(); return;
  }
  keys[e.code]=true;
  if(['ArrowUp','ArrowDown','ArrowLeft','ArrowRight','Space'].includes(e.code)) e.preventDefault();
  if(e.repeat) return;
  if(e.code==='KeyE') interact();
  if(e.code==='KeyF') toggleFlash();
  if(e.code==='KeyM') mm.classList.toggle('big');
  if(e.code==='KeyP' && state==='playing') pauseGame();
});
window.addEventListener('keyup',e=>{ keys[e.code]=false; });
window.addEventListener('blur',()=>{ for(const k in keys) keys[k]=false; });
function toggleFlash(){ flashOn=!flashOn; flash.intensity=flashOn?2.0:0; }
function look(dx,dy,s){
  if(player.hidden) player.yaw=clamp(player.yaw-dx*s,hideYaw-.7,hideYaw+.7); else player.yaw-=dx*s;
  player.pitch=clamp(player.pitch-dy*s,-1.3,1.3);
}
window.addEventListener('mousemove',e=>{
  if(state!=='playing' || pad) return;
  if(document.pointerLockElement===canvas || (e.buttons&1)) look(e.movementX||0,e.movementY||0,0.0022);
});
(function setupTouch(){
  const jz=$('joyZone'), base=$('joyBase'), knob=$('joyKnob');
  let jid=null, ox=0, oy=0;
  jz.addEventListener('pointerdown',e=>{
    jid=e.pointerId; try{jz.setPointerCapture(e.pointerId);}catch(_){}
    ox=e.clientX; oy=e.clientY; base.style.display='block';
    base.style.left=(ox-50)+'px'; base.style.top=(oy-50)+'px'; knob.style.transform='translate(0,0)';
  });
  jz.addEventListener('pointermove',e=>{
    if(e.pointerId!==jid) return;
    let dx=e.clientX-ox, dy=e.clientY-oy; const m=Math.hypot(dx,dy), R=50;
    if(m>R){ dx*=R/m; dy*=R/m; }
    joy.x=dx/R; joy.y=dy/R; knob.style.transform='translate('+dx+'px,'+dy+'px)';
  });
  const end=e=>{ if(e.pointerId!==jid) return; jid=null; joy.x=joy.y=0; base.style.display='none'; };
  jz.addEventListener('pointerup',end); jz.addEventListener('pointercancel',end);
  const lz=$('lookZone'); let lid=null, lx=0, ly=0;
  lz.addEventListener('pointerdown',e=>{ lid=e.pointerId; try{lz.setPointerCapture(e.pointerId);}catch(_){} lx=e.clientX; ly=e.clientY; });
  lz.addEventListener('pointermove',e=>{ if(e.pointerId!==lid || state!=='playing') return; look(e.clientX-lx,e.clientY-ly,0.0055); lx=e.clientX; ly=e.clientY; });
  const lend=e=>{ if(e.pointerId===lid) lid=null; };
  lz.addEventListener('pointerup',lend); lz.addEventListener('pointercancel',lend);
  btnAct.addEventListener('pointerdown',e=>{ e.preventDefault(); interact(); });
  const br=$('btnRun');
  br.addEventListener('pointerdown',e=>{ e.preventDefault(); touchRun=true; });
  ['pointerup','pointercancel','pointerleave'].forEach(ev=>br.addEventListener(ev,()=>{ touchRun=false; }));
  $('btnLight').addEventListener('pointerdown',e=>{ e.preventDefault(); toggleFlash(); });
})();

/* ================= ATUALIZAÇÃO ================= */
function updatePlayer(dt){
  if(player.hidden){
    player.moving=false; player.running=false;
    player.stamina=Math.min(100,player.stamina+25*dt); if(player.stamina>30) player.exhausted=false;
    return;
  }
  if(!pad){
    if(keys.ArrowLeft) player.yaw+=1.9*dt;
    if(keys.ArrowRight) player.yaw-=1.9*dt;
  }
  let fwd=0, str=0;
  if(!pad){
    fwd=(keys.KeyW||keys.ArrowUp?1:0)-(keys.KeyS||keys.ArrowDown?1:0)-joy.y;
    str=(keys.KeyD?1:0)-(keys.KeyA?1:0)+joy.x;
  }
  const len=Math.hypot(fwd,str);
  if(len>1){ fwd/=len; str/=len; }
  const moving=len>0.08;
  const wantRun=(keys.ShiftLeft||keys.ShiftRight||touchRun) && moving;
  const canRun=wantRun && player.stamina>0 && !player.exhausted;
  const speed=canRun?6.3:3.8;
  if(canRun){ player.stamina=Math.max(0,player.stamina-22*dt); if(player.stamina<=0) player.exhausted=true; }
  else{ player.stamina=Math.min(100,player.stamina+(moving?10:16)*dt); if(player.exhausted && player.stamina>25) player.exhausted=false; }
  const sy=Math.sin(player.yaw), cy=Math.cos(player.yaw);
  player.x+=(-sy*fwd+cy*str)*speed*dt; player.z+=(-cy*fwd-sy*str)*speed*dt;
  resolve(player,0.4,[walls[player.f],props[player.f]]);
  player.x=Math.min(player.x,44);
  player.moving=moving; player.running=canRun;
  if(moving) player.bob+=dt*(canRun?13:8.5);
  if(player.f===0 && player.x>41.3) winGame();
}
function updateWorld(dt){
  items.forEach(k=>{
    if(k.got||!k.revealed) return;
    k.group.position.y=k.baseY+Math.sin(time*2.2)*0.06; k.group.rotation.y+=dt*2;
  });
  doors.forEach(d=>{ if(d.open && d.progress<1){ d.progress=Math.min(1,d.progress+dt*(d.type==='gate'?0.7:1.6)); d.apply(d.progress); } });
  flickMats.forEach((m,i)=>{
    const on=Math.random()>(i===2?0.04:(i?0.06:0.1));
    if(power){ const v=on?1:0.18; m.color.setRGB(v,0.98*v,0.88*v); }
    else{ const v=Math.random()>0.93?0.55:0.12; m.color.setRGB(v,v,v*0.9); }
  });
}
function updateCamera(dt){
  if(state==='menu'){
    camera.position.set(-31,1.7,0); camera.rotation.set(-0.03,-Math.PI/2+Math.sin(time*0.35)*0.5,0); camera.fov=74; camera.updateProjectionMatrix(); return;
  }
  const bob=(player.hidden?0:Math.sin(player.bob)*0.045*(player.moving?1:0));
  camera.position.set(player.x,player.f*FH+(player.hidden?1.6:1.7)+bob,player.z);
  camera.rotation.set(player.pitch,player.yaw,0);
  const tf=player.running?84:74;
  if(Math.abs(camera.fov-tf)>0.1){ camera.fov+=(tf-camera.fov)*Math.min(1,dt*6); camera.updateProjectionMatrix(); }
}

let last=performance.now();
function loop(now){
  requestAnimationFrame(loop);
  const dt=Math.min(0.05,(now-last)/1000); last=now;
  if(state==='paused'||state==='over'||state==='won'){ renderer.render(scene,camera); return; }
  time+=dt;
  if(state==='playing'){
    elapsed+=dt;
    updatePlayer(dt);
    for(const g of guards){ if(state==='playing') updateGuard(g,dt); }
    if(state==='playing') updateHUD(dt);
    mmT-=dt; if(mmT<=0){ mmT=0.12; drawMap(); }
  }
  updateWorld(dt);
  updateCamera(dt);
  renderer.render(scene,camera);
}

resetAll();
menuScreen();
requestAnimationFrame(loop);
})();
