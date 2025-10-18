# Ajedrez 
<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>Ajedrez - Juego simple</title>
<style>
  :root{
    --light:#f0d9b5;
    --dark:#b58863;
    --sel:#ffd54d;
    --mark:#8fd3ff;
    --bg:#0b1220;
    --panel:#111827;
    --text:#e6edf3;
  }
  *{box-sizing:border-box}
  body{margin:0;min-height:100vh;display:flex;align-items:center;justify-content:center;background:linear-gradient(180deg,#061021,#071428);font-family:system-ui,Segoe UI,Roboto,Arial;color:var(--text)}
  .container{width:900px;max-width:96%;display:flex;gap:18px;padding:18px;background:var(--panel);border-radius:12px;box-shadow:0 12px 30px rgba(0,0,0,.6)}
  .left{display:flex;flex-direction:column;gap:12px}
  canvas{background:linear-gradient(#fff,#eee);border-radius:8px;display:block}
  .info{min-width:220px;padding:12px;background:rgba(255,255,255,0.03);border-radius:8px}
  .info h2{margin:0 0 8px 0;font-size:18px}
  .btn{display:inline-block;padding:8px 10px;border-radius:8px;background:#0b1220;border:0;color:var(--text);cursor:pointer;margin-right:6px}
  .row{display:flex;gap:8px;margin-bottom:8px}
  .log{height:320px;overflow:auto;background:rgba(0,0,0,0.18);padding:8px;border-radius:6px;font-size:13px}
  .score{font-weight:600}
  footer{font-size:12px;margin-top:8px;color:#b9c4d6}
</style>
</head>
<body>
  <div class="container">
    <div class="left">
      <canvas id="board" width="640" height="640"></canvas>
      <div style="display:flex;justify-content:space-between;align-items:center">
        <div>
          <button class="btn" id="reset">Reiniciar</button>
          <button class="btn" id="flip">Girar tablero</button>
        </div>
        <div>Turno: <span id="turn">Blancas</span></div>
      </div>
    </div>

    <div class="info">
      <h2>Ajedrez simple</h2>
      <div class="row">
        <div>Jugadas:</div>
        <div class="score" id="movesCount">0</div>
      </div>
      <div class="row">
        <div>Capturas blancas:</div><div id="capW">-</div>
      </div>
      <div class="row">
        <div>Capturas negras:</div><div id="capB">-</div>
      </div>
      <div style="margin-top:8px;font-weight:600">Registro de movimientos</div>
      <div class="log" id="log"></div>
      <footer>Arrastra o haz click en una pieza para moverla. La promoción es automática a reina.</footer>
    </div>
  </div>

<script>
// ----- Configuración -----
const canvas = document.getElementById('board');
const ctx = canvas.getContext('2d');
const size = canvas.width;
const cells = 8;
const cellSize = size / cells;
let flipBoard = false;

// estado del juego
let board = []; // matriz 8x8 de piezas o null
// pieza: {type: 'p','r','n','b','q','k', color: 'w'|'b'}
let selected = null;
let legalMoves = [];
let turn = 'w';
let moveCount = 0;
let captures = {w:[], b:[]};
const logEl = document.getElementById('log');
const movesCountEl = document.getElementById('movesCount');
const capW = document.getElementById('capW');
const capB = document.getElementById('capB');
const turnEl = document.getElementById('turn');

// ----- Inicialización -----
function setupBoard(){
  // vaciar
  board = Array.from({length:8}, ()=>Array(8).fill(null));
  // filas principales
  const back = ['r','n','b','q','k','b','n','r'];
  for(let i=0;i<8;i++){
    board[0][i] = {type:back[i], color:'b'};
    board[7][i] = {type:back[i], color:'w'};
    board[1][i] = {type:'p', color:'b'};
    board[6][i] = {type:'p', color:'w'};
  }
  selected = null;
  legalMoves = [];
  turn = 'w';
  moveCount = 0;
  captures = {w:[], b:[]};
  updateUI();
  draw();
}

// ----- Dibujado -----
function draw(){
  // tablero
  for(let r=0;r<8;r++){
    for(let c=0;c<8;c++){
      let x = c*cellSize, y = r*cellSize;
      if(flipBoard){ x = (7-c)*cellSize; y = (7-r)*cellSize; }
      const light = (r+c)%2===0;
      ctx.fillStyle = light ? getComputedStyle(document.documentElement).getPropertyValue('--light').trim() : getComputedStyle(document.documentElement).getPropertyValue('--dark').trim();
      ctx.fillRect(x,y,cellSize,cellSize);
    }
  }

  // marcar legal moves
  ctx.globalAlpha = 0.9;
  for(const m of legalMoves){
    const r = flipBoard? 7-m.r : m.r;
    const c = flipBoard? 7-m.c : m.c;
    const x = c*cellSize, y = r*cellSize;
    ctx.fillStyle = getComputedStyle(document.documentElement).getPropertyValue('--mark').trim();
    ctx.fillRect(x+4,y+4,cellSize-8,cellSize-8);
  }

  // marcar seleccionado
  if(selected){
    const {r,c} = selected;
    const rr = flipBoard? 7-r : r;
    const cc = flipBoard? 7-c : c;
    ctx.fillStyle = getComputedStyle(document.documentElement).getPropertyValue('--sel').trim();
    ctx.fillRect(cc*cellSize+2, rr*cellSize+2, cellSize-4, cellSize-4);
  }

  // piezas (dibujar letras simples)
  ctx.globalAlpha = 1;
  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';
  ctx.font = `${cellSize*0.52}px system-ui`;

  for(let r=0;r<8;r++){
    for(let c=0;c<8;c++){
      const piece = board[r][c];
      if(!piece) continue;
      const drawR = flipBoard? 7-r : r;
      const drawC = flipBoard? 7-c : c;
      const x = drawC*cellSize + cellSize/2;
      const y = drawR*cellSize + cellSize/2;
      ctx.fillStyle = piece.color === 'w' ? '#111' : '#fff';
      const glyph = glyphFor(piece);
      ctx.fillText(glyph, x, y + (piece.type === 'p' ? 2 : 0));
    }
  }
}

// devuelve un símbolo unicode para la pieza
function glyphFor(p){
  // usamos letras simples para claridad (puedes cambiar a piezas Unicode)
  const map = {
    'p':'♟', 'r':'♜','n':'♞','b':'♝','q':'♛','k':'♚'
  };
  if(!p) return '';
  const g = map[p.type] || p.type;
  // si blanca, usar versión blanca (invertimos color con css)
  if(p.color === 'w'){
    // estilos: dibujaremos el mismo glyph pero lo pintamos oscuro arriba
    return g;
  }
  return g;
}

// ----- Utilitarios de coordenadas -----
function posFromMouse(e){
  const rect = canvas.getBoundingClientRect();
  const cx = e.clientX - rect.left, cy = e.clientY - rect.top;
  let c = Math.floor(cx / cellSize), r = Math.floor(cy / cellSize);
  if(flipBoard){ c = 7 - c; r = 7 - r; }
  c = clamp(c,0,7); r = clamp(r,0,7);
  return {r,c};
}
function clamp(v,a,b){ return Math.max(a,Math.min(b,v)); }

// ----- Lógica de movimientos (básica, sin enroque/en passant) -----
function inBounds(r,c){ return r>=0 && r<8 && c>=0 && c<8; }

function generateMoves(r,c){
  const piece = board[r][c];
  if(!piece) return [];
  const color = piece.color;
  const moves = [];
  const addIfEmptyOrCapture = (rr,cc)=>{
    if(!inBounds(rr,cc)) return;
    const target = board[rr][cc];
    if(!target) moves.push({r:rr,c:cc});
    else if(target.color !== color) moves.push({r:rr,c:cc});
  };

  switch(piece.type){
    case 'p': {
      const dir = color==='w' ? -1 : 1;
      // un paso
      const r1 = r + dir;
      if(inBounds(r1,c) && !board[r1][c]) moves.push({r:r1,c:c});
      // doble paso desde fila inicial
      const startRow = color==='w' ? 6 : 1;
      const r2 = r + 2*dir;
      if(r===startRow && inBounds(r2,c) && !board[r1][c] && !board[r2][c]) moves.push({r:r2,c:c});
      // capturas diagonales
      for(const dc of [-1,1]){
        const rr = r + dir, cc = c + dc;
        if(inBounds(rr,cc) && board[rr][cc] && board[rr][cc].color !== color) moves.push({r:rr,c:cc});
      }
      break;
    }
    case 'r': {
      const dirs = [[1,0],[-1,0],[0,1],[0,-1]];
      for(const [dr,dc] of dirs){
        let rr=r+dr, cc=c+dc;
        while(inBounds(rr,cc)){
          if(!board[rr][cc]) { moves.push({r:rr,c:cc}); }
          else { if(board[rr][cc].color!==color) moves.push({r:rr,c:cc}); break; }
          rr += dr; cc += dc;
        }
      }
      break;
    }
    case 'b': {
      const dirs = [[1,1],[1,-1],[-1,1],[-1,-1]];
      for(const [dr,dc] of dirs){
        let rr=r+dr, cc=c+dc;
        while(inBounds(rr,cc)){
          if(!board[rr][cc]) { moves.push({r:rr,c:cc}); }
          else { if(board[rr][cc].color!==color) moves.push({r:rr,c:cc}); break; }
          rr += dr; cc += dc;
        }
      }
      break;
    }
    case 'q': {
      const dirs = [[1,0],[-1,0],[0,1],[0,-1],[1,1],[1,-1],[-1,1],[-1,-1]];
      for(const [dr,dc] of dirs){
        let rr=r+dr, cc=c+dc;
        while(inBounds(rr,cc)){
          if(!board[rr][cc]) { moves.push({r:rr,c:cc}); }
          else { if(board[rr][cc].color!==color) moves.push({r:rr,c:cc}); break; }
          rr += dr; cc += dc;
        }
      }
      break;
    }
    case 'n': {
      const steps = [[2,1],[2,-1],[-2,1],[-2,-1],[1,2],[1,-2],[-1,2],[-1,-2]];
      for(const [dr,dc] of steps){ const rr=r+dr, cc=c+dc; if(inBounds(rr,cc)) addIfEmptyOrCapture(rr,cc); }
      break;
    }
    case 'k': {
      for(let dr=-1;dr<=1;dr++) for(let dc=-1;dc<=1;dc++){
        if(dr===0 && dc===0) continue;
        const rr=r+dr, cc=c+dc; if(inBounds(rr,cc)) addIfEmptyOrCapture(rr,cc);
      }
      break;
    }
  }
  return moves;
}

// ----- Interacción -----
canvas.addEventListener('mousedown', e=>{
  const {r,c} = posFromMouse(e);
  const piece = board[r][c];
  if(piece && piece.color === turn){
    selected = {r,c};
    legalMoves = generateMoves(r,c);
    draw();
  } else if(selected){
    // intentar mover a la casilla clickeada
    attemptMove(selected.r, selected.c, r, c);
  }
});

canvas.addEventListener('mouseup', e=>{
  // para soporte táctil/drag simple mantenemos click-release
});
canvas.addEventListener('dblclick', e=>{
  // deseleccionar
  selected = null; legalMoves = []; draw();
});

// para arrastrar (soporte básico)
let dragging = false;
canvas.addEventListener('mousemove', e=>{
  if(!selected) return;
  // highlight se mantiene por legalMoves; opcional se puede dibujar pieza flotante
});

function attemptMove(r0,c0,r1,c1){
  // comprobar si el destino está dentro de legalMoves
  const legal = legalMoves.some(m=>m.r===r1 && m.c===c1);
  if(!legal){ selected = null; legalMoves = []; draw(); return; }
  const moving = board[r0][c0];
  const target = board[r1][c1];
  // aplicar movimiento
  board[r1][c1] = moving;
  board[r0][c0] = null;
  // promoción automática a reina
  if(moving.type === 'p'){
    if(moving.color==='w' && r1===0){ board[r1][c1] = {type:'q', color:'w'}; }
    if(moving.color==='b' && r1===7){ board[r1][c1] = {type:'q', color:'b'}; }
  }
  // registro de captura
  if(target){
    captures[moving.color].push(target.type);
  }
  // actualizar turno
  turn = turn === 'w' ? 'b' : 'w';
  moveCount++;
  logMove(moving, r0,c0, r1,c1, target);
  selected = null; legalMoves = [];
  updateUI();
  draw();
}

// ----- Registro y UI -----
function coordToAlgebraic(r,c){
  const file = 'abcdefgh'[c];
  const rank = 8 - r;
  return `${file}${rank}`;
}
function pieceName(p){
  if(!p) return '';
  const names = {p:'P', r:'R', n:'N', b:'B', q:'Q', k:'K'};
  return names[p.type] || p.type.toUpperCase();
}
function logMove(piece, r0,c0, r1,c1, captured){
  const from = coordToAlgebraic(r0,c0);
  const to = coordToAlgebraic(r1,c1);
  const p = pieceName(piece);
  const cap = captured ? 'x' : '-';
  const txt = `${moveCount}. ${piece.color === 'w' ? '' : ''}${p}${from}${captured ? 'x' : '-'}${to}`;
  const entry = document.createElement('div');
  entry.textContent = `${piece.color === 'w' ? 'Blancas':'Negras'}: ${p}${from}${captured ? 'x':''}${to}`;
  logEl.prepend(entry);
}
function updateUI(){
  movesCountEl.textContent = moveCount;
  capW.textContent = captures['w'].length ? captures['w'].join(',') : '-';
  capB.textContent = captures['b'].length ? captures['b'].join(',') : '-';
  turnEl.textContent = turn === 'w' ? 'Blancas' : 'Negras';
}

// ----- Botones -----
document.getElementById('reset').addEventListener('click', ()=>{ setupBoard(); logEl.innerHTML=''; });
document.getElementById('flip').addEventListener('click', ()=>{ flipBoard = !flipBoard; draw(); });

// ----- Inicio -----
setupBoard();

// redibujar si la ventana cambia tamaño (mantener escala)
window.addEventListener('resize', ()=>{ draw(); });

</script>
</body>
</html>
