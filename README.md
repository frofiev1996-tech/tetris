<!doctype html>
<html lang="ru">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Tetris — Встроенная версия</title>
<style>
  :root{
    --cell-size: 28px;
    --cols: 10;
    --rows: 20;
    --ui-w: 220px;
    --bg: #0b1020;
    --panel: #0f1724;
    --accent: #00d4ff;
    --muted: #98a0b3;
  }
  *{box-sizing:border-box}
  body{
    margin:0;
    font-family: Inter, Roboto, Arial, sans-serif;
    background: linear-gradient(180deg,#031026 0%, #051426 100%);
    color:#e6eef8;
    display:flex;
    flex-direction:column;
    align-items:center;
    justify-content:flex-start;
    min-height:100vh;
    padding:10px;
  }
  .wrap{
    display:flex;
    gap:20px;
    align-items:flex-start;
    flex-wrap:wrap;
  }
  .board{
    background: #071025;
    padding:12px;
    border-radius:10px;
    box-shadow: 0 8px 30px rgba(2,8,20,0.7);
    position:relative;
  }
  canvas{
    display:block;
    background: linear-gradient(180deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01));
    border-radius:6px;
    image-rendering: pixelated;
  }
  .panel{
    width:var(--ui-w);
    background:var(--panel);
    padding:12px;
    border-radius:10px;
    box-shadow: 0 6px 20px rgba(2,8,20,0.6);
  }
  h1{font-size:18px;margin:0 0 8px 0}
  .meta{color:var(--muted);font-size:13px;margin-bottom:12px}
  .stat{display:flex;justify-content:space-between;padding:8px 0;border-top:1px solid rgba(255,255,255,0.03)}
  .next{
    width:100%;
    height:120px;
    background: rgba(255,255,255,0.02);
    border-radius:6px;
    display:flex;
    align-items:center;
    justify-content:center;
    margin:8px 0 12px 0;
  }
  .controls{font-size:13px;color:var(--muted);line-height:1.5}
  .btn{
    display:inline-block;margin-top:12px;padding:8px 10px;border-radius:6px;background:linear-gradient(180deg,#0a89a8,#007b99);color:#fff;text-decoration:none;cursor:pointer;
  }
  .touch-controls{
    display:flex;
    flex-wrap:wrap;
    justify-content:center;
    gap:8px;
    margin-top:10px;
  }
  .touch-controls button{
    width:50px;
    height:50px;
    font-size:18px;
    border:none;
    border-radius:6px;
    background:var(--accent);
    color:#000;
    font-weight:bold;
  }
  footer{font-size:12px;color:var(--muted);margin-top:10px;text-align:center}
  @media (max-width:700px){
    .wrap{flex-direction:column;align-items:center}
    .panel{width:100%}
  }
</style>
</head>
<body>
<div class="wrap">
  <div class="board">
    <canvas id="stage" width="280" height="560"></canvas>
  </div>

  <div class="panel">
    <h1>Тетрис (браузер)</h1>
    <div class="meta">Управление: стрелки или кнопки на экране</div>

    <div class="stat"><div>Счёт</div><div id="score">0</div></div>
    <div class="stat"><div>Уровень</div><div id="level">1</div></div>
    <div class="stat"><div>Линии</div><div id="lines">0</div></div>

    <div class="next" id="nextBox">
      <canvas id="next" width="112" height="112"></canvas>
    </div>

    <div class="controls">
      <div>Цель: убрать как можно больше линий.</div>
      <div style="margin-top:8px">Результат растёт по сложной формуле: больше линий — больше очков.</div>
      <div style="margin-top:8px">Bag-алгоритм случайности (без долгой засухи фигур).</div>
    </div>

    <button id="restart" class="btn">Рестарт</button>

    <div class="touch-controls">
      <button onclick="playerMove(-1)">←</button>
      <button onclick="playerMove(1)">→</button>
      <button onclick="playerDrop()">↓</button>
      <button onclick="playerRotate(1)">⟳</button>
      <button onclick="hardDrop()">⤓</button>
      <button onclick="togglePause()">⏸</button>
    </div>

    <footer>
      Сделано ChatGPT — теперь работает на телефоне.
    </footer>
  </div>
</div>

<script>
/* ---- Конфигурация ---- */
const COLS = 10;
const ROWS = 20;
const CELL = 28;
const canvas = document.getElementById('stage');
const ctx = canvas.getContext('2d');
canvas.width = COLS * CELL;
canvas.height = ROWS * CELL;
ctx.scale(CELL, CELL);
ctx.imageSmoothingEnabled = false;

const nextCanvas = document.getElementById('next');
const nctx = nextCanvas.getContext('2d');
nctx.scale(CELL / 2.5, CELL / 2.5);
nctx.imageSmoothingEnabled = false;

/* ---- Tetrominoes ---- */
const SHAPES = { /* как было */ I:[[ [0,0,0,0],[1,1,1,1],[0,0,0,0],[0,0,0,0] ],[[0,0,1,0],[0,0,1,0],[0,0,1,0],[0,0,1,0]]],J:[[[1,0,0],[1,1,1],[0,0,0]]],L:[[[0,0,1],[1,1,1],[0,0,0]]],O:[[[1,1],[1,1]]],S:[[[0,1,1],[1,1,0],[0,0,0]]],T:[[[0,1,0],[1,1,1],[0,0,0]]],Z:[[[1,1,0],[0,1,1],[0,0,0]]] };
const COLORS = { I:'#00f0f0', J:'#0000f0', L:'#f0a000', O:'#f0f000', S:'#00f000', T:'#a000f0', Z:'#f00000' };

/* ---- Game state ---- */
let arena = createMatrix(COLS, ROWS);
let score = 0, level = 1, lines = 0;
let dropCounter = 0, dropInterval = 1000, lastTime = 0;
let paused = false, gameOver = false;
let bag = [];

function refillBag(){ bag = ['I','J','L','O','S','T','Z'].sort(()=>Math.random()-0.5); }
function nextPieceFromBag(){ if(bag.length===0) refillBag(); return bag.pop(); }

let player = { pos:{x:0,y:0}, matrix:null, type:null };
function createMatrix(w,h){ return Array.from({length:h},()=>Array(w).fill(0)); }
function rotate(matrix,dir=1){ for(let y=0;y<matrix.length;y++){for(let x=y+1;x<matrix[y].length;x++){[matrix[y][x],matrix[x][y]]=[matrix[x][y],matrix[y][x]];}} if(dir>0) matrix.forEach(r=>r.reverse()); else matrix.reverse(); }
function buildMatrixFromShape(key){ const raw=SHAPES[key][0]; return raw.map(r=>Array.from(r)); }

function spawn(){ const type=nextPieceFromBag(); player.type=type; player.matrix=buildMatrixFromShape(type); player.pos.y=0; player.pos.x=Math.floor(COLS/2)-Math.floor(player.matrix[0].length/2); if(collide(arena,player)){ gameOver=true; paused=true; alert('Game over! Твой счёт: '+score); } }
function collide(arena,player){ const m=player.matrix; for(let y=0;y<m.length;y++){for(let x=0;x<m[y].length;x++){if(m[y][x]&&(arena[y+player.pos.y]&&arena[y+player.pos.y][x+player.pos.x])) return true;}} return false; }
function merge(arena,player){ const m=player.matrix; for(let y=0;y<m.length;y++){for(let x=0;x<m[y].length;x++){if(m[y][x]) arena[y+player.pos.y][x+player.pos.x]=player.type;}} }
function sweep(){ let rowCount=0; outer: for(let y=arena.length-1;y>=0;y--){for(let x=0;x<arena[y].length;x++){if(!arena[y][x]) continue outer;} const row=arena.splice(y,1)[0].fill(0); arena.unshift(row); y++; rowCount++; } if(rowCount>0){ const points=[0,40,100,300,1200]; score+=(points[rowCount]*level); lines+=rowCount; level=Math.floor(lines/10)+1; dropInterval=Math.max(100,1000-(level-1)*80); updateUI(); } }

function playerDrop(){ player.pos.y++; if(collide(arena,player)){ player.pos.y--; merge(arena,player); sweep(); spawn(); } dropCounter=0; }
function playerMove(dir){ player.pos.x+=dir; if(collide(arena,player)) player.pos.x-=dir; }
function playerRotate(dir){ const pos=player.pos.x; rotate(player.matrix,dir); let offset=1; while(collide(arena,player)){ player.pos.x+=offset; offset=-(offset+(offset>0?1:-1)); if(offset>player.matrix[0].length){ rotate(player.matrix,-dir); player.pos.x=pos; return; } } }
function hardDrop(){ while(!collide(arena,player)) player.pos.y++; player.pos.y--; merge(arena,player); sweep(); spawn(); dropCounter=0; }

function drawCell(x,y,val){ if(!val) return; const color=COLORS[val]||'#fff'; ctx.fillStyle=color; ctx.fillRect(x,y,1,1); ctx.strokeStyle='rgba(0,0,0,0.25)'; ctx.lineWidth=0.06; ctx.strokeRect(x,y,1,1); }
function drawMatrix(matrix,offset){ for(let y=0;y<matrix.length;y++){for(let x=0;x<matrix[y].length;x++){if(matrix[y][x]) drawCell(x+offset.x,y+offset.y,matrix[y][x]);}}}
function draw(){ ctx.fillStyle='#071025'; ctx.fillRect(0,0,COLS,ROWS); for(let y=0;y<arena.length;y++){for(let x=0;x<arena[y].length;x++){if(arena[y][x]) drawCell(x,y,arena[y][x]); else {ctx.strokeStyle='rgba(255,255,255,0.02)'; ctx.lineWidth=0.02; ctx.strokeRect(x,y,1,1);}}} drawMatrix(player.matrix,player.pos); }
function drawNext(){ nctx.clearRect(0,0,nextCanvas.width,nextCanvas.height); const nextType=bag[bag.length-1]||'I'; const m=buildMatrixFromShape(nextType); const offs={x:2,y:1}; for(let y=0;y<m.length;y++){for(let x=0;x<m[y].length;x++){if(m[y][x]){ nctx.fillStyle=COLORS[nextType]; nctx.fillRect(x+offs.x,y+offs.y,1,1); nctx.strokeStyle='rgba(0,0,0,0.25)'; nctx.lineWidth=0.06; nctx.strokeRect(x+offs.x,y+offs.y,1,1); }}}}

function updateUI(){ document.getElementById('score').textContent=score; document.getElementById('level').textContent=level; document.getElementById('lines').textContent=lines; }

function update(time=0){ if(paused){ lastTime=time; requestAnimationFrame(update); return; } const delta=time-lastTime; lastTime=time; dropCounter+=delta; if(dropCounter>dropInterval) playerDrop(); draw(); drawNext(); requestAnimationFrame(update); }

document.addEventListener('keydown', ev=>{ if(gameOver&&ev.key.toLowerCase()!=='r') return; switch(ev.key){ case 'ArrowLeft': playerMove(-1); break; case 'ArrowRight': playerMove(1); break; case 'ArrowDown': playerDrop(); break; case 'ArrowUp': playerRotate(1); break; case ' ': hardDrop(); break; case 'p': case 'P': togglePause(); break; case 'r': case 'R': startGame(); break; } });

document.getElementById('restart').addEventListener('click', startGame);

function togglePause(){ paused=!paused; }

function startGame(){ arena=createMatrix(COLS,ROWS); score=0; level=1; lines=0; gameOver=false; paused=false; dropInterval=1000; bag=[]; refillBag(); spawn(); updateUI(); lastTime=0; }

startGame();
update();

/* ---- Тач-события для свайпов ---- */
let startX,startY;
canvas.addEventListener('touchstart',e=>{ const t=e.touches[0]; startX=t.clientX; startY=t.clientY; });
canvas.addEventListener('touchend',e=>{ const t=e.changedTouches[0]; const dx=t.clientX-startX; const dy=t.clientY-startY; if(Math.abs(dx)>Math.abs(dy)){ if(dx>0) playerMove(1); else playerMove(-1); } else { if(dy>0) playerDrop(); else playerRotate(1); } });
</script>
</body>
</html>
