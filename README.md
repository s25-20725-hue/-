<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0"/>
<title>Animal Tower (HTML5)</title>
<style>
  * { box-sizing: border-box; }
  html, body { margin:0; padding:0; background:#0e0f13; color:#f2f2f2; font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial;}
  #ui {
    position: absolute; inset: 0; pointer-events: none;
    display: flex; flex-direction: column; align-items: center; justify-content: space-between;
  }
  #topbar {
    width: 100%; display:flex; align-items:center; justify-content:space-between;
    padding: 10px 14px; background: linear-gradient(180deg, rgba(0,0,0,.45), rgba(0,0,0,0));
    pointer-events: none;
  }
  .pill {
    pointer-events: auto;
    background: rgba(255,255,255,.08);
    border: 1px solid rgba(255,255,255,.15);
    border-radius: 999px;
    padding: 8px 12px; margin: 0 6px; font-weight: 700; letter-spacing: .2px;
    backdrop-filter: blur(6px);
  }
  #score { font-size: 18px; }
  #help  { font-size: 14px; opacity:.9;}
  #title { font-weight:900; font-size:18px; letter-spacing:.3px; }
  #bottom {
    width:100%; display:flex; align-items:center; justify-content:center; gap:10px;
    padding: 10px 14px; background: linear-gradient(0deg, rgba(0,0,0,.45), rgba(0,0,0,0));
  }
  button {
    pointer-events:auto; cursor:pointer; border:none; border-radius:10px; padding:10px 14px; font-weight:800;
    background:#42d392; color:#082a1e; box-shadow: 0 6px 18px rgba(66,211,146,.25);
  }
  button.secondary { background:#a8b1ff; color:#0c0c2b; box-shadow:0 6px 18px rgba(168,177,255,.25); }
  #centerTip {
    position:absolute; top:54px; left:50%; transform:translateX(-50%);
    font-size:13px; opacity:.9; background:rgba(255,255,255,.08); border:1px solid rgba(255,255,255,.15);
    padding:6px 10px; border-radius:999px; pointer-events:none;
  }
  canvas { display:block; margin:0 auto; background: linear-gradient(#87c6ff, #bfe1ff 45%, #d9f0ff); }
  #gameover {
    position:absolute; inset:0; display:none; align-items:center; justify-content:center; flex-direction:column; gap:18px;
    background: radial-gradient(1200px 600px at 50% 30%, rgba(0,0,0,.35), rgba(0,0,0,.6));
    backdrop-filter: blur(2px);
  }
  #gameover h1 { margin:0; font-size:36px; }
  #gameover p { margin:0; opacity:.9; }
</style>
</head>
<body>
<div id="ui">
  <div id="topbar">
    <div id="title" class="pill">ANIMAL TOWER</div>
    <div style="display:flex; align-items:center;">
      <div id="score" class="pill">Score: 0</div>
      <div id="help" class="pill">클릭/터치/Space 로 떨어뜨리기</div>
    </div>
  </div>
  <div id="centerTip">균형을 잡아 최대한 높이 쌓아 보세요!</div>
  <div id="bottom">
    <button id="dropBtn" class="secondary">지금 떨어뜨리기</button>
    <button id="resetBtn">다시 시작</button>
  </div>
  <div id="gameover">
    <h1>GAME OVER</h1>
    <p id="finalScore">Score: 0</p>
    <div style="display:flex; gap:10px; margin-top:6px;">
      <button id="retryBtn">다시 도전</button>
      <button id="shareBtn" class="secondary">클립보드로 점수 복사</button>
    </div>
  </div>
</div>

<!-- Matter.js physics (CDN) -->
<script src="https://cdn.jsdelivr.net/npm/matter-js@0.20.0/build/matter.min.js"></script>
<script>
(() => {
  // ----- Basic setup -----
  const W = Math.min(520, Math.floor(window.innerWidth));
  const H = Math.min(900, Math.floor(window.innerHeight));
  const pixRatio = window.devicePixelRatio || 1;

  const engine = Matter.Engine.create();
  const world  = engine.world;
  world.gravity.y = 1.1; // gravity

  const render = Matter.Render.create({
    element: document.body,
    engine,
    options: {
      width: W, height: H, pixelRatio: pixRatio,
      wireframes: false, background: 'transparent',
      showAngleIndicator: false
    }
  });
  const runner = Matter.Runner.create();

  // Move canvas under UI
  document.body.insertBefore(render.canvas, document.querySelector('#ui'));

  // ----- Colors / materials -----
  const COLORS = [
    '#ffb02e','#ff6392','#4cd9e8','#8be28b','#a890fe','#ffd166','#06d6a0','#ef476f'
  ];

  const MATERIALS = {
    bouncy: { restitution: 0.15, friction: 0.9, frictionStatic: 0.7, frictionAir: 0.006, density: 0.0016 },
    light : { restitution: 0.08, friction: 0.85, frictionStatic: 0.75, frictionAir: 0.006, density: 0.0012 },
    heavy : { restitution: 0.05, friction: 0.95, frictionStatic: 0.8, frictionAir: 0.005, density: 0.0020 }
  };

  // ----- Ground & boundaries -----
  const ground = Matter.Bodies.rectangle(W/2, H-20, W*0.72, 24, { isStatic: true, chamfer: { radius: 8 }, render:{ fillStyle:'#2f3b2f' }});
  const base   = Matter.Bodies.rectangle(W/2, H-32, W*0.72, 8, { isStatic: true, render:{ fillStyle:'#4b6f4b' }});
  const leftWall  = Matter.Bodies.rectangle(-40, H/2, 80, H*2, { isStatic: true, render:{ opacity:0 }});
  const rightWall = Matter.Bodies.rectangle(W+40, H/2, 80, H*2, { isStatic: true, render:{ opacity:0 }});
  Matter.World.add(world, [ground, base, leftWall, rightWall]);

  // Lose sensor (if anything falls too low)
  const loseY = H + 80;

  // ----- Crane (moving spawn x) -----
  let craneDir = 1;
  let craneX = W/2;
  const craneY = 90;
  const craneMin = W*0.18, craneMax = W*0.82, craneSpeed = 1.6;

  // ----- State -----
  let currentBody = null;
  let canDrop = true;
  let score = 0;
  let gameOver = false;

  const $score = document.getElementById('score');
  const $dropBtn = document.getElementById('dropBtn');
  const $resetBtn = document.getElementById('resetBtn');
  const $gameover = document.getElementById('gameover');
  const $finalScore = document.getElementById('finalScore');
  const $retryBtn = document.getElementById('retryBtn');
  const $shareBtn = document.getElementById('shareBtn');

  // ----- Animal-ish shapes (stylized polygons) -----
  // We’ll emulate 다양한 동물 실루엣: 거북/코끼리/기린/펭귄/고래 등 (균형이 서로 다르게)
  function makeAnimal(x, y, scale) {
    // Random pick
    const types = ['turtle','elephant','giraffe','penguin','whale','hippo','rhino','panda'];
    const type = types[Math.floor(Math.random()*types.length)];
    const color = COLORS[Math.floor(Math.random()*COLORS.length)];

    let body;
    const mat = (Math.random() < 0.4) ? MATERIALS.heavy : (Math.random() < 0.7 ? MATERIALS.light : MATERIALS.bouncy);

    switch(type) {
      case 'turtle': {
        // rounded hex (low center)
        body = Matter.Bodies.polygon(x, y, 6, 26*scale, { ...mat, chamfer:{ radius: 8*scale }});
        break;
      }
      case 'elephant': {
        // wide rectangle w/ chamfer (heavy, stable)
        body = Matter.Bodies.rectangle(x, y, 70*scale, 40*scale, { ...mat, chamfer:{ radius: 10*scale }});
        break;
      }
      case 'giraffe': {
        // tall rectangle (risky)
        body = Matter.Bodies.rectangle(x, y, 28*scale, 72*scale, { ...mat, chamfer:{ radius: 6*scale }});
        break;
      }
      case 'penguin': {
        // octagon-ish
        body = Matter.Bodies.polygon(x, y, 8, 28*scale, { ...mat });
        break;
      }
      case 'whale': {
        // long capsule-like (low profile, slippery)
        body = Matter.Bodies.rectangle(x, y, 90*scale, 30*scale, { ...mat, chamfer:{ radius: 16*scale }});
        break;
      }
      case 'hippo': {
        // fat square
        body = Matter.Bodies.rectangle(x, y, 56*scale, 56*scale, { ...mat, chamfer:{ radius: 12*scale }});
        break;
      }
      case 'rhino': {
        // triangle (very unstable tip)
        body = Matter.Bodies.polygon(x, y, 3, 32*scale, { ...mat });
        break;
      }
      default: { // panda
        // pentagon
        body = Matter.Bodies.polygon(x, y, 5, 30*scale, { ...mat });
      }
    }

    // Add cute face dots as separate render (simple)
    body.render.fillStyle = color;
    body.render.strokeStyle = 'rgba(0,0,0,.25)';
    body.render.lineWidth = 1.2;

    // Tiny random rotation to increase challenge
    Matter.Body.rotate(body, (Math.random()-0.5) * 0.18);

    // Tag for scoring
    body.plugin = { isAnimal: true, placed: false, type };

    return body;
  }

  function spawnNext() {
    if (gameOver) return;
    const scale = 1 + Math.random()*0.25;
    currentBody = makeAnimal(craneX, craneY, scale);
    Matter.World.add(world, currentBody);
    // make kinematic until drop (no gravity by making it static then switch)
    Matter.Body.setStatic(currentBody, true);
  }

  function dropCurrent() {
    if (!currentBody || !canDrop || gameOver) return;
    canDrop = false;
    Matter.Body.setStatic(currentBody, false);
    // slight downward nudge
    Matter.Body.setVelocity(currentBody, {x:0, y:1});
    // score after a short settle
    setTimeout(() => {
      if (currentBody && !gameOver) {
        currentBody.plugin.placed = true;
        score++;
        $score.textContent = `Score: ${score}`;
      }
    }, 600);
    // schedule next spawn
    setTimeout(() => {
      currentBody = null;
      canDrop = true;
      spawnNext();
    }, 800);
  }

  // ----- Input -----
  function handleDrop() { dropCurrent(); }
  window.addEventListener('pointerdown', () => { handleDrop(); });
  window.addEventListener('keydown', (e) => {
    if (e.code === 'Space') { e.preventDefault(); handleDrop(); }
    if ((e.key === 'r' || e.key === 'R') && gameOver) restart();
  });
  $dropBtn.onclick = handleDrop;
  $resetBtn.onclick = () => restart();

  // ----- Crane movement -----
  Matter.Events.on(engine, 'beforeUpdate', () => {
    if (gameOver) return;

    // crane horizontal bounce
    craneX += craneSpeed * craneDir;
    if (craneX < craneMin) { craneX = craneMin; craneDir = 1; }
    if (craneX > craneMax) { craneX = craneMax; craneDir = -1; }

    // follow current body if static (before drop)
    if (currentBody && currentBody.isStatic) {
      Matter.Body.setPosition(currentBody, { x: craneX, y: craneY });
    }
  });

  // ----- Render crane helper -----
  const renderCrane = () => {
    const ctx = render.context;
    ctx.save();
    ctx.lineWidth = 2;
    ctx.strokeStyle = 'rgba(0,0,0,.25)';
    ctx.fillStyle = '#ffd166';

    // rope
    ctx.beginPath();
    ctx.moveTo(craneX, 0);
    ctx.lineTo(craneX, craneY - 24);
    ctx.stroke();

    // hook
    ctx.beginPath();
    ctx.arc(craneX, craneY - 18, 6, 0, Math.PI*2);
    ctx.fill();

    // top beam
    ctx.fillStyle = '#264653';
    ctx.fillRect(W*0.12, 8, W*0.76, 8);

    ctx.restore();
  };

  // decorate sky (clouds)
  function drawCloud(x, y, s, ctx) {
    ctx.save();
    ctx.globalAlpha = 0.25;
    ctx.fillStyle = '#ffffff';
    ctx.beginPath();
    ctx.arc(x, y, 14*s, 0, Math.PI*2);
    ctx.arc(x+14*s, y+3*s, 18*s, 0, Math.PI*2);
    ctx.arc(x+30*s, y-2*s, 12*s, 0, Math.PI*2);
    ctx.arc(x+42*s, y+2*s, 16*s, 0, Math.PI*2);
    ctx.fill();
    ctx.restore();
  }

  Matter.Events.on(render, 'afterRender', () => {
    // extra HUD drawings on top of Matter render
    const ctx = render.context;
    drawCloud(80, 80, 1, ctx);
    drawCloud(W-120, 110, 1.2, ctx);
    renderCrane();
  });

  // ----- Lose detection -----
  function checkLose() {
    if (gameOver) return;
    const bodies = Matter.Composite.allBodies(world);
    for (const b of bodies) {
      if (!b.isStatic && b.position && b.position.y > loseY && b.plugin && b.plugin.isAnimal) {
        endGame();
        return true;
      }
    }
    return false;
  }

  function endGame() {
    gameOver = true;
    $finalScore.textContent = `Score: ${score}`;
    $gameover.style.display = 'flex';
  }

  function restart() {
    // remove all animals
    const bodies = Matter.Composite.allBodies(world);
    for (const b of bodies) {
      if (b.plugin && b.plugin.isAnimal) Matter.World.remove(world, b);
    }
    score = 0;
    $score.textContent = `Score: ${score}`;
    $gameover.style.display = 'none';
    gameOver = false;
    canDrop = true;
    currentBody = null;
    craneDir = 1; craneX = W/2;
    spawnNext();
  }

  $retryBtn.onclick = restart;
  $shareBtn.onclick = async () => {
    try {
      await navigator.clipboard.writeText(`[Animal Tower] 내 점수: ${score}`);
      $shareBtn.textContent = '복사 완료!';
      setTimeout(()=> $shareBtn.textContent = '클립보드로 점수 복사', 1400);
    } catch(e) {
      alert('클립보드 접근이 허용되지 않았습니다.');
    }
  };

  // ----- Start engine -----
  Matter.Render.run(render);
  Matter.Runner.run(runner, engine);

  // Initial spawn
  spawnNext();

  // Game loop checks
  (function loop(){
    if (!gameOver) checkLose();
    requestAnimationFrame(loop);
  })();

  // Resize (simple: just center canvas via CSS; we keep physics size fixed for consistency)
  window.addEventListener('resize', () => {
    // no-op: fixed canvas for stable gameplay
  });
})();
</script>
</body>
</html>
