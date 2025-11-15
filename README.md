<!doctype html>
<html lang="pt-BR">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>Tia Sil — Resgate da Floresta (versão completa)</title>
<style>
  :root{--bg:#08120b}
  html,body{height:100%;margin:0;background:var(--bg);display:flex;align-items:center;justify-content:center;font-family:Inter,Arial,Helvetica,sans-serif;color:#e7f5ea}
  #wrap{width:960px;max-width:98vw;padding:12px;background:#072015;border-radius:10px;box-shadow:0 8px 30px rgba(0,0,0,.6)}
  header{text-align:center;margin-bottom:8px}
  canvas{display:block;width:100%;background:#0b2615;border-radius:6px;image-rendering:pixelated}
  #hud{display:flex;justify-content:space-between;padding:6px 10px;font-size:14px;color:#dff3e6}
  #msg{margin-top:6px;text-align:center;min-height:28px}
  button{background:#1d6a3f;color:white;border:0;padding:6px 10px;border-radius:6px;cursor:pointer}
  .small{font-size:12px;color:#9fcfae}
</style>
</head>
<body>
  <div id="wrap">
    <header>
      <h1>Tia Sil — Resgate da Floresta</h1>
      <div class="small">Fase 1: apagar o fogo (5 baldes). Cutscene. Fase 2: replantar (5 sementes). Final: jardim.</div>
    </header>

    <div id="hud">
      <div>Fase: <span id="faseLabel">1 (Água)</span></div>
      <div>Baldes: <span id="countWater">0</span>/5</div>
      <div>Sementes: <span id="countSeed">0</span>/5</div>
    </div>

    <canvas id="game" width="900" height="500"></canvas>
    <div id="msg">Controles: ← → mover • ↑ / Espaço pular • M alterna música • R reiniciar</div>
  </div>

<script>
/*
  Jogo completo:
  - usa imagens externas: tia.png, gringo.png, arvore.png
  - animações simples (walk blink/pulse + jump)
  - fogo animado, fumaça, luzes
  - música via WebAudio
  - gringo segue a Tia Sil
  - cutscenes com fade
  - frases sobre justiça socioambiental (mostradas em cutscenes/HUD)
*/

// ---------- CARREGAR IMAGENS (coloque os arquivos na mesma pasta) ----------
const SPR_TIA = new Image(); SPR_TIA.src = 'tia.png';
const SPR_GRINGO = new Image(); SPR_GRINGO.src = 'gringo.png';
const SPR_TREE = new Image(); SPR_TREE.src = 'arvore.png';

// ---------- CANVAS ----------
const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');
ctx.imageSmoothingEnabled = false;
const W = canvas.width, H = canvas.height;

// ---------- HUD ----------
const faseLabel = document.getElementById('faseLabel');
const countWaterEl = document.getElementById('countWater');
const countSeedEl = document.getElementById('countSeed');
const msgEl = document.getElementById('msg');

// ---------- ESTADO ----------
let state = {
  fase: 1, // 1 = água, 1.5 = cutscene encontro, 2 = sementes, 3 = final/cutscene
  water: 0,
  seed: 0,
  musicOn: true,
  fade: {val:0,mode:null}, // mode: 'in' | 'out'
  showPhrase: null
};

// frases sobre justiça socioambiental
const frases = [
  "Justiça socioambiental = cuidado com pessoas + proteção dos territórios.",
  "Salvar a floresta é garantir vidas, modos de subsistência e futuro.",
  "Quem protege a natureza protege também comunidades vulneráveis.",
  "A reconstrução precisa ser justa: participação, cuidado e respeito."
];

// ---------- PLAYER (Tia Sil) ----------
const player = {
  x: 80, y: 320, w: 72, h: 100,
  vx: 0, vy: 0,
  speed: 3.4,
  onGround: false,
  facing: 1, // 1 right, -1 left
  animTime:0,
  state:'idle' // idle | walk | jump
};

// ---------- GRINGO ----------
const gringo = {
  x: 700, y: 320, w:72, h:100,
  vx:0, vy:0, follow:true
};

// ---------- MUNDO / ITENS ----------
let platforms = [];
let items = []; // each item: {x,y,w,h,type}
let trees = []; // render positions
let fires = []; // fire sprites positions

// ---------- CONTROLES ----------
let keys = {};
window.addEventListener('keydown', e=>{ keys[e.key]=true;
  if(e.key==='m' || e.key==='M'){ toggleMusic(); }
  if(e.key==='r' || e.key==='R'){ init(); }
});
window.addEventListener('keyup', e=> keys[e.key]=false);

// ---------- AUDIO (WebAudio simples 8-bit loop) ----------
let audioCtx, masterGain, musicOsc;
function startMusic(){
  if(audioCtx) return;
  audioCtx = new (window.AudioContext || window.webkitAudioContext)();
  masterGain = audioCtx.createGain(); masterGain.gain.value = 0.07; masterGain.connect(audioCtx.destination);

  // cria um pequeno arpejo com três osciladores programados (loop via setInterval)
  const pattern = [440, 523, 659, 880];
  let idx = 0;
  window.musicInterval = setInterval(()=>{
    const note = pattern[idx % pattern.length];
    const o = audioCtx.createOscillator();
    o.type = 'square';
    o.frequency.value = note;
    const g = audioCtx.createGain(); g.gain.value = 0.08;
    o.connect(g); g.connect(masterGain);
    o.start();
    setTimeout(()=>{ o.stop(); }, 220);
    idx++;
  }, 300);
}
function stopMusic(){
  if(window.musicInterval) clearInterval(window.musicInterval);
  window.musicInterval = null;
  if(masterGain) masterGain.disconnect();
  audioCtx = null;
}
function toggleMusic(){ state.musicOn = !state.musicOn; if(state.musicOn) startMusic(); else stopMusic(); }

// ---------- INICIALIZAÇÃO ----------
function init(){
  // estado
  state.fase = 1; state.water = 0; state.seed = 0; state.fade={val:0,mode:'in'}; state.showPhrase=null;
  player.x = 80; player.y = 320; player.vx=0; player.vy=0; player.animTime=0;
  gringo.x = 700; gringo.y = 320;
  // plataformas simples (chão e alguns blocos)
  platforms = [{x:0,y:420,w:900,h:80}];
  platforms.push({x:200,y:340,w:120,h:16});
  platforms.push({x:380,y:300,w:120,h:16});
  platforms.push({x:560,y:260,w:120,h:16});
  // árvores espalhadas
  trees = [];
  for(let i=0;i<6;i++){
    trees.push({x:40+i*140,y:180});
  }
  // fogo = em fase 1, metade das árvores pegam fogo
  fires = [];
  for(let i=0;i<trees.length;i++){
    fires.push({x:trees[i].x+16,y:trees[i].y+10, on: true});
  }
  // itens fase 1: baldes
  items = [];
  for(let i=0;i<5;i++){
    items.push({x:180 + i*120, y:380, w:28, h:28, type:'balde', taken:false});
  }
  // não tocar música automaticamente em dispositivos que bloqueiam autoplay; ativar ao pressionar M
  if(state.musicOn) startMusic();
}
init();

// ---------- FUNÇÕES DE DESENHO ----------
function drawRoundedRect(x,y,w,h,r,col){
  ctx.fillStyle=col; ctx.beginPath();
  ctx.moveTo(x+r,y); ctx.arcTo(x+w,y,x+w,y+h,r);
  ctx.arcTo(x+w,y+h,x,y+h,r); ctx.arcTo(x,y+h,x,y,r); ctx.arcTo(x,y,x+w,y,r);
  ctx.fill();
}

function drawScene(){
  // background base (escuro) com iluminação suave
  ctx.fillStyle = '#07150c'; ctx.fillRect(0,0,W,H);

  // se fase 1: luz de fogo (pulsante)
  if(state.fase === 1){
    const t = performance.now()*0.004;
    const glow = 0.12 + 0.08*Math.sin(t*2);
    // overlay laranja sutil
    ctx.fillStyle = rgba(255,120,40,${glow}); ctx.fillRect(0,0,W,H);
  } else if(state.fase === 2){
    // cinza-sujo para floresta destruída
    ctx.fillStyle = 'rgba(60,60,60,0.6)'; ctx.fillRect(0,0,W,H);
  } else {
    // fase 3 final -> verde vivo
    ctx.fillStyle = '#123f18'; ctx.fillRect(0,0,W,H);
  }

  // desenhar árvores (ou árvores queimadas)
  for(let i=0;i<trees.length;i++){
    const tr = trees[i];
    if(state.fase === 1 && fires[i] && fires[i].on){
      // árvore em chamas: desenhar árvore, escurecer e desenhar fogo animado
      ctx.globalAlpha = 0.9;
      ctx.drawImage(SPR_TREE, tr.x, tr.y, 80, 120);
      ctx.globalAlpha = 1;
      drawFire(tr.x+20, tr.y+10, i);
      // queimado: uma camada escura
      ctx.fillStyle = 'rgba(0,0,0,0.35)'; ctx.fillRect(tr.x, tr.y+60, 80, 60);
    } else if(state.fase === 2){
      // árvore destruída = tronco só (desenho simplificado)
      ctx.fillStyle = '#4b3a2a'; ctx.fillRect(tr.x+28, tr.y+50, 24, 60);
      // solo seco em volta
      ctx.fillStyle = 'rgba(80,60,50,0.6)'; ctx.fillRect(tr.x-6, tr.y+110, 92, 12);
    } else {
      // árvore normal viva (fase 3 ou depois de replantar)
      ctx.drawImage(SPR_TREE, tr.x, tr.y, 80, 120);
    }
  }

  // plataformas (chão)
  platforms.forEach(p=>{
    ctx.fillStyle = '#2f4d2f'; ctx.fillRect(p.x,p.y,p.w,p.h);
  });

  // itens (baldes ou sementes)
  items.forEach(it=>{
    if(it.taken) return;
    if(it.type === 'balde'){
      // desenhar balde azul com luz
      ctx.fillStyle = '#2f8cff'; ctx.fillRect(it.x, it.y, it.w, it.h);
      ctx.fillStyle = '#bfe7ff'; ctx.fillRect(it.x+4, it.y-6, it.w-8, 4);
    } else {
      // semente: marrom + folha
      ctx.fillStyle = '#6b4a2b'; ctx.fillRect(it.x, it.y, it.w, it.h);
      ctx.fillStyle = '#5bd66b'; ctx.fillRect(it.x-4, it.y-8, it.w+8, 6);
    }
  });

  // personagem (Tia Sil) — animação simples: pequeno bob/shift para simular frames
  drawCharacter(SPR_TIA, player, 'tia');

  // se fase >=2, desenhar gringo e animação
  if(state.fase >= 1.5){
    drawCharacter(SPR_GRINGO, gringo, 'gringo');
  }

  // overlay de fade para cutscene
  if(state.fade.mode){
    ctx.fillStyle = rgba(0,0,0,${state.fade.val});
    ctx.fillRect(0,0,W,H);
  }

  // HUD adicional / frases
  if(state.showPhrase){
    ctx.fillStyle = 'rgba(0,0,0,0.55)'; ctx.fillRect(60,80,820,56);
    ctx.fillStyle = '#dff7e7'; ctx.font = '20px Arial';
    wrapText(state.showPhrase, 70, 110, 800, 22);
  }
}

// desenha personagem com "simulação" de frames: oscila posição + pequeno tilt
function drawCharacter(img, ent, tag){
  const t = performance.now()*0.004;
  let bob = 0;
  if(ent.vy !== 0) bob = Math.sin(t*6)*2; // no ar leve
  else bob = Math.sin(t*6)3 (Math.abs(ent.vx)>0.3 ? 1 : 0.4);
  // flip se necessário
  ctx.save();
  if(ent === player && player.facing < 0){
    ctx.translate(ent.x+ent.w/2, 0); ctx.scale(-1,1); ctx.translate(-(ent.x+ent.w/2),0);
  }
  ctx.drawImage(img, ent.x, ent.y + bob, ent.w, ent.h);
  ctx.restore();
}

// desenhar fogo simples com partículas/burn
function drawFire(x,y,seed){
  const t = performance.now()*0.006 + seed*0.7;
  for(let i=0;i<6;i++){
    const px = x + Math.sin(t+i)*10 + (Math.random()*2-1)*2;
    const py = y + Math.cos(t*1.3 + i)*6 + i*6;
    const a = 0.8 - i*0.12;
    ctx.fillStyle = rgba(${255 - i*20},${80 + i*10},0,${a});
    ctx.beginPath(); ctx.arc(px,py,6-i*0.8,0,Math.PI*2); ctx.fill();
  }
  // fumaça
  ctx.fillStyle = 'rgba(60,60,60,0.22)';
  ctx.beginPath(); ctx.ellipse(x+10, y-8, 16, 8, 0, 0, Math.PI*2); ctx.fill();
}

// ---------- FÍSICA E LÓGICA ----------
const GRAV = 0.6;
function update(dt){
  // >>> player control
  const left = keys['ArrowLeft'] || keys['a'] || keys['A'];
  const right = keys['ArrowRight'] || keys['d'] || keys['D'];
  const jump = (keys['ArrowUp'] || keys[' ']);

  if(left){ player.vx = -player.speed; player.facing = -1; player.state='walk'; }
  else if(right){ player.vx = player.speed; player.facing = 1; player.state='walk'; }
  else { player.vx *= 0.8; if(Math.abs(player.vx) < 0.2) player.vx = 0; if(player.onGround) player.state='idle'; }

  // pulo (simples)
  if(jump && player.onGround){ player.vy = -11; player.onGround = false; player.state='jump'; }

  // aplicar física
  player.vy += GRAV;
  player.x += player.vx;
  player.y += player.vy;

  // colisão com plataformas
  player.onGround = false;
  platforms.forEach(p=>{
    if(player.x + player.w > p.x && player.x < p.x + p.w && player.y + player.h > p.y && player.y + player.h < p.y + p.h + 20 && player.vy >= 0){
      player.y = p.y - player.h;
      player.vy = 0;
      player.onGround = true;
    }
  });

  // limites mundo
  if(player.x < 0) player.x = 0;
  if(player.x + player.w > W) player.x = W - player.w;
  if(player.y > H) { player.y = 320; player.vy = 0; }

  // >>> gringo: segue a Tia Sil com velocidade limitada
  if(state.fase >= 1.5){
    const dx = player.x - gringo.x;
    if(Math.abs(dx) > 50){
      gringo.vx = Math.sign(dx) * 1.6; // velocidade de seguir
      gringo.x += gringo.vx;
    } else gringo.vx = 0;
    // gringo doesn't jump; keep him on ground
  }

  // >>> checar coleta de items
  items.forEach(it=>{
    if(it.taken) return;
    if(rectsOverlap(player, it)){
      it.taken = true;
      if(it.type === 'balde'){ state.water++; countWaterEl.textContent = state.water; }
      if(it.type === 'semente'){ state.seed++; countSeedEl.textContent = state.seed; }
      // se coletou balde e era fogo, extingue um fogo com prioridade
      if(it.type === 'balde'){
        // apaga primeiro fire true
        for(let f of fires){ if(f.on){ f.on = false; break; } }
      }
    }
  });

  // quando pegar 5 baldes -> cutscene de encontro
  if(state.fase === 1 && state.water >= 5){
    state.fase = 1.5; faseLabel.textContent = 'Cutscene: encontro';
    state.fade.mode = 'in'; state.fade.val = 0;
    // mostrar frase de justiça socioambiental aleatória
    state.showPhrase = frases[Math.floor(Math.random()*frases.length)];
    setTimeout(()=>{
      // transição para fase 2 (destruída)
      state.fade.mode = 'out';
      setTimeout(()=>{
        state.showPhrase = null;
        state.fase = 2; faseLabel.textContent = '2 (Sementes)';
        // preparar itens sementes
        items = [];
        for(let i=0;i<5;i++) items.push({x:120 + i*140, y:360, w:28, h:28, type:'semente', taken:false});
        // apagar fogo visível
        fires.forEach(f=>f.on=false);
        state.fade.mode = 'in';
      }, 800);
    }, 1800);
  }

  // quando pegar 5 sementes -> final
  if(state.fase === 2 && state.seed >= 5){
    state.fase = 3; faseLabel.textContent = '3 (Final)';
    state.fade.mode='in'; state.fade.val=0;
    state.showPhrase = 'Com justiça socioambiental, a floresta e as pessoas florescem juntas.';
    setTimeout(()=>{ state.showPhrase = null; state.fade.mode='out'; }, 2400);
  }

  // fade handling
  if(state.fade.mode === 'in'){ state.fade.val += 0.03; if(state.fade.val >= 0.98){ state.fade.val = 0.98; state.fade.mode = null; } }
  else if(state.fade.mode === 'out'){ state.fade.val -= 0.04; if(state.fade.val <= 0){ state.fade.val=0; state.fade.mode=null; } }
}

// ---------- UTILIDADES ----------
function rectsOverlap(a,b){ return a.x < b.x + b.w && a.x + a.w > b.x && a.y < b.y + b.h && a.y + a.h > b.y; }
function wrapText(text,x,y,maxWidth,lineHeight){
  const words = text.split(' '); let line=''; let yy=y;
  ctx.font = '18px Arial'; ctx.fillStyle = '#e9f9ea';
  for(let n=0;n<words.length;n++){
    const testLine = line + words[n] + ' ';
    const metrics = ctx.measureText(testLine);
    if(metrics.width > maxWidth && n>0){ ctx.fillText(line, x, yy); line = words[n] + ' '; yy += lineHeight; }
    else line = testLine;
  }
  ctx.fillText(line, x, yy);
}

// ---------- LOOP ----------
let last = 0;
function frame(ts){
  const dt = (ts - last)/1000; last = ts;
  update(dt);
  drawScene();
  requestAnimationFrame(frame);
}
requestAnimationFrame(frame);

// ---------- instruções finais (exibir counts) ----------
setInterval(()=>{
  countWaterEl.textContent = state.water;
  countSeedEl.textContent = state.seed;
}, 200);

// ---------- instruções para o usuário ----------
console.log("Jogo carregado. Use ← → ↑/Espaço para mover/pular, M para música, R para reiniciar.");

// ---------- dica: comece música com M caso não toque automaticamente ----------
</script>
</body>
</html>
