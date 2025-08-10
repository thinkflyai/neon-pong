<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>Neon Pong — Mobile Ready</title>
<style>
  :root {
    --bg1:#0f1021; --bg2:#1b1e3b; --fg:#eaf2ff;
    --neon:#7cf4ff; --neon2:#ff72e1; --accent:#8cff69;
  }
  html,body{
    height:100%; margin:0;
    background:radial-gradient(1200px 800px at 20% 10%,rgba(124,244,255,.08),transparent 60%),
               radial-gradient(1000px 700px at 80% 90%,rgba(255,114,225,.08),transparent 60%),
               linear-gradient(160deg,var(--bg1),var(--bg2));
    color:var(--fg);
    font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial;
  }
  .wrap{display:grid;place-items:center;height:100%;}
  canvas{background:rgba(0,0,0,.25);backdrop-filter: blur(8px);
    border-radius:16px; box-shadow:0 20px 60px rgba(0,0,0,.55);}
  .hud{position:fixed;top:14px;left:0;right:0;text-align:center;
    font-weight:800;letter-spacing:.06em;user-select:none}
  .sub{font-weight:600;opacity:.75;margin-top:6px;font-size:14px}
  .touchHint{position:fixed;bottom:12px;left:12px;opacity:.7;font-size:13px}
</style>
</head>
<body>
<div class="hud">
  <div id="score">0 : 0</div>
  <div class="sub">W/S • ↑/↓ • Space = Start • P = Pause • R = Reset • F = Fullscreen</div>
</div>
<div class="wrap">
  <canvas id="game" width="960" height="540" aria-label="Neon Pong"></canvas>
</div>
<div class="touchHint">Touch & drag left/right halves to move paddles</div>

<script>
(() => {
  const canvas = document.getElementById('game');
  const c = canvas.getContext('2d');
  const scoreEl = document.getElementById('score');

  let audioCtx = null; let muted = false;
  function beep(type='square', freq=440, dur=0.06, gain=0.06){
    if (muted) return;
    if (!audioCtx) try { audioCtx = new (window.AudioContext||window.webkitAudioContext)(); } catch(_){ return; }
    const t = audioCtx.currentTime;
    const o = audioCtx.createOscillator(); const g = audioCtx.createGain();
    o.type = type; o.frequency.value = freq; g.gain.setValueAtTime(gain, t);
    g.gain.exponentialRampToValueAtTime(0.0001, t+dur);
    o.connect(g).connect(audioCtx.destination); o.start(t); o.stop(t+dur);
  }

  const W = canvas.width, H = canvas.height;
  const PADDLE = { w: 14, h: 100, speed: 540 };
  const BALL = { r: 9, speed: 460, maxSpeed: 920 };

  let paused = true, last = 0, serveCountdown = 0;
  const left = { x: 42, y: H/2 - PADDLE.h/2, vy: 0, score: 0, scale:1 };
  const right= { x: W-42-PADDLE.w, y: H/2 - PADDLE.h/2, vy: 0, score: 0, scale:1 };
  const ball = { x: W/2, y: H/2, vx: BALL.speed, vy: 0, r: BALL.r };

  function clamp(v,min,max){return Math.max(min, Math.min(max, v));}
  function resetBall(toLeft){
    ball.x=W/2; ball.y=H/2;
    const a=(Math.random()*0.6-0.3);
    const d=toLeft?-1:1, s=BALL.speed;
    ball.vx=Math.cos(a)*s*d; ball.vy=Math.sin(a)*s;
    serveCountdown=0.7; paused=false; // tap/space starts
  }
  function resetAll(){
    left.y=H/2-PADDLE.h/2; right.y=H/2-PADDLE.h/2;
    left.score=0; right.score=0; updateScore(); resetBall(false);
  }
  function updateScore(){ scoreEl.textContent = `${left.score} : ${right.score}`; }

  const keys={};
  addEventListener('keydown',e=>{
    keys[e.key]=true;
    if(e.key===' '){ paused=false; }
    if(e.key.toLowerCase()==='p') paused=!paused;
    if(e.key.toLowerCase()==='r') { resetAll(); paused=true; }
  });
  addEventListener('keyup',e=>keys[e.key]=false);

  // 🆕 Tap to start + unlock audio on mobile
  canvas.addEventListener('touchstart', () => {
    if (paused || serveCountdown > 0) paused = false;
    try { beep('square', 1, 0.001, 0.001); } catch {}
  }, { passive: false });

  // Touch move controls
  canvas.addEventListener('touchmove',e=>{
    for(const t of e.changedTouches){
      const rect=canvas.getBoundingClientRect();
      const rx=t.clientX-rect.left, ry=t.clientY-rect.top;
      if(rx < W/2) left.y = clamp(ry - PADDLE.h/2, 0, H - PADDLE.h);
      else right.y = clamp(ry - PADDLE.h/2, 0, H - PADDLE.h);
    }
    e.preventDefault();
  }, {passive:false});

  function update(dt){
    if (serveCountdown>0){
      serveCountdown -= dt;
      if (serveCountdown>0) return;
    }
    const upL = keys['w']||keys['W']; const dnL = keys['s']||keys['S'];
    left.vy = (upL? -1:0) + (dnL?1:0); left.y += left.vy * PADDLE.speed * dt;
    left.y = clamp(left.y, 0, H - PADDLE.h);

    const upR=keys['ArrowUp'], dnR=keys['ArrowDown'];
    right.vy = (upR?-1:0)+(dnR?1:0); right.y += right.vy * PADDLE.speed * dt;
    right.y = clamp(right.y, 0, H - PADDLE.h);

    ball.x += ball.vx * dt; ball.y += ball.vy * dt;
    if (ball.y - ball.r <= 0 && ball.vy < 0){ ball.y = ball.r; ball.vy *= -1; }
    if (ball.y + ball.r >= H && ball.vy > 0){ ball.y = H - ball.r; ball.vy *= -1; }

    // Paddle collisions
    function collide(p){
      if (ball.y + ball.r < p.y || ball.y - ball.r > p.y + PADDLE.h) return false;
      if (p===left && ball.x - ball.r <= p.x + PADDLE.w && ball.vx < 0 && ball.x > p.x){
        ball.x = p.x + PADDLE.w + ball.r;
      } else if (p===right && ball.x + ball.r >= p.x && ball.vx > 0 && ball.x < p.x + PADDLE.w){
        ball.x = p.x - ball.r;
      } else return false;
      const hitPos = ((ball.y - p.y) - PADDLE.h/2)/(PADDLE.h/2);
      const speed = Math.min(Math.hypot(ball.vx, ball.vy)*1.05, BALL.maxSpeed);
      const angle = hitPos * Math.PI/3.8, dir = p===left?1:-1;
      ball.vx = Math.cos(angle)*speed*dir; ball.vy = Math.sin(angle)*speed;
      beep('square', dir>0?520:480, .05, .05);
      return true;
    }
    collide(left); collide(right);

    if (ball.x < -ball.r){ right.score++; updateScore(); resetBall(true); }
    if (ball.x > W + ball.r){ left.score++; updateScore(); resetBall(false); }
  }

  function draw(){
    c.clearRect(0,0,W,H);
    c.fillStyle='#eaf2ff'; c.beginPath(); c.arc(ball.x, ball.y, ball.r, 0, Math.PI*2); c.fill();
    c.fillRect(left.x, left.y, PADDLE.w, PADDLE.h);
    c.fillRect(right.x, right.y, PADDLE.w, PADDLE.h);
    if (serveCountdown>0){
      c.fillStyle='rgba(0,0,0,.45)'; c.fillRect(0,0,W,H);
      c.fillStyle='#eaf2ff'; c.font='700 26px system-ui'; c.textAlign='center';
      c.fillText('Get Ready...', W/2, H/2);
    }
  }

  function loop(ts){
    const t=ts/1000; let dt=t-last; last=t;
    dt=Math.min(dt,1/30);
    if(!paused) update(dt);
    draw();
    requestAnimationFrame(loop);
  }

  resetAll();
  requestAnimationFrame(loop);
})();
</script>
</body>
</html>

