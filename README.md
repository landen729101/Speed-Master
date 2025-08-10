<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no" />
  <title>Traffic Racer</title>
  <style>
    html,body{height:100%;margin:0;background:#0b1020;overflow:hidden;-webkit-user-select:none}
    #gameWrap{position:fixed;inset:0;display:flex;align-items:center;justify-content:center}
    canvas{touch-action:none;background:linear-gradient(#263142,#0b1020);border-radius:10px}
    #hud{position:fixed;top:12px;left:12px;color:#fff;font-family:sans-serif;font-weight:700;z-index:20}
    #hud .score{font-size:18px}
    #hud .high{font-size:12px;opacity:.8}
    #controls{position:fixed;inset:auto 12px 18px 12px;display:flex;justify-content:space-between;z-index:20}
    .btn{width:48%;height:86px;border-radius:14px;background:rgba(255,255,255,0.06);backdrop-filter:blur(2px);display:flex;align-items:center;justify-content:center;font-weight:800;color:#fff;font-size:18px;user-select:none}
    .btn:active{background:rgba(255,255,255,0.12)}
    #restart{position:fixed;right:12px;bottom:120px;background:#ff5757;color:#fff;padding:10px 14px;border-radius:10px;font-weight:800;border:none;z-index:20}
    #pause{position:fixed;left:12px;bottom:120px;background:#3aa0ff;color:#fff;padding:10px 14px;border-radius:10px;font-weight:800;border:none;z-index:20}
  </style>
</head>
<body>
  <div id="gameWrap"><canvas id="c"></canvas></div>
  <div id="hud"><div class="score">Score: <span id="score">0</span></div><div class="high">High: <span id="high">0</span></div></div>
  <div id="controls">
    <div id="leftBtn" class="btn">◀</div>
    <div id="rightBtn" class="btn">▶</div>
  </div>
  <button id="pause">Pause</button>
  <button id="restart">Restart</button>

<script>
(function(){
  const canvas = document.getElementById('c');
  const ctx = canvas.getContext('2d');

  function resize(){
    const w = window.innerWidth, h = window.innerHeight;
    const ratio = window.devicePixelRatio || 1;
    canvas.style.width = w + 'px';
    canvas.style.height = h + 'px';
    canvas.width = w * ratio;
    canvas.height = h * ratio;
    ctx.setTransform(ratio,0,0,ratio,0,0);
  }
  window.addEventListener('resize', resize);
  resize();

  let running = true, gameOver = false;
  let score = 0, high = Number(localStorage.getItem('racer_high')||0);
  document.getElementById('high').textContent = high;

  const track = { lanes: 3, lanePadding: 14 };
  const player = { w: 46, h: 80, lane: 1, x:0, y:0, color: '#4de27f' };
  let traffic = [];
  let baseSpeed = 3.2, speedIncrease = 0.0025, spawnTimer = 0, spawnInterval = 70, difficultyMult = 1;
  const particles = [];
  let moveInput = 0;

  function reset(){
    score = 0; baseSpeed = 3.2; difficultyMult = 1; spawnInterval = 70;
    traffic = []; particles.length = 0; gameOver = false; player.lane = 1;
    updatePlayerPos();
  }
  function updatePlayerPos(){
    const pw = canvas.width / (window.devicePixelRatio||1);
    const ph = canvas.height / (window.devicePixelRatio||1);
    const trackW = Math.min(420, pw - 40);
    const laneW = (trackW - track.lanePadding*2) / track.lanes;
    const left = (pw - trackW)/2 + track.lanePadding;
    player.x = left + player.lane * laneW + laneW/2 - player.w/2;
    player.y = ph - player.h - 18;
  }
  function spawnTraffic(){
    const car = {
      lane: Math.floor(Math.random()*track.lanes),
      w: 40 + Math.round(Math.random()*20),
      h: 70 + Math.round(Math.random()*40),
      y: -120 - Math.random()*200,
      speed: baseSpeed * (0.6 + Math.random()*1.1),
      color: pickTrafficColor()
    };
    traffic.push(car);
  }
  function pickTrafficColor(){
    const colors = ['#ff6b6b','#ffd93d','#5bc0eb','#9b5de5','#ff7b00','#00b894'];
    return colors[Math.floor(Math.random()*colors.length)];
  }
  function rectsIntersect(a,b){
    return !(a.x + a.w < b.x || a.x > b.x + b.w || a.y + a.h < b.y || a.y > b.y + b.h);
  }
  function triggerExplosion(cx,cy){
    const num = 26 + Math.floor(Math.random()*26);
    for(let i=0;i<num;i++){
      particles.push({
        x:cx + (Math.random()-0.5)*18,
        y:cy + (Math.random()-0.5)*18,
        vx:(Math.random()-0.5)*6,
        vy:(Math.random()-0.5)*6,
        r:2 + Math.random()*4,
        color:pickTrafficColor(),
        life:40 + Math.floor(Math.random()*30)
      });
    }
  }
  function gameTick(){
    baseSpeed += speedIncrease * difficultyMult;
    spawnTimer++;
    if(spawnTimer >= spawnInterval){
      spawnTimer = 0; spawnTraffic();
      spawnInterval = Math.max(18, 70 - Math.floor(score/5));
    }
    const pw = canvas.width / (window.devicePixelRatio||1);
    const ph = canvas.height / (window.devicePixelRatio||1);
    const trackW = Math.min(420, pw - 40);
    const laneW = (trackW - track.lanePadding*2) / track.lanes;
    const left = (pw - trackW)/2 + track.lanePadding;

    for(let i=traffic.length-1;i>=0;i--){
      const t = traffic[i];
      t.y += baseSpeed + t.speed * 0.2;
      t.x = left + t.lane*laneW + laneW/2 - t.w/2;
      if(t.y > ph + 40){
        traffic.splice(i,1);
        score++; difficultyMult += 0.02;
        if(score % 7 === 0) baseSpeed += 0.6;
        document.getElementById('score').textContent = score;
        if(score > high){ high = score; localStorage.setItem('racer_high', high); document.getElementById('high').textContent = high; }
      }
    }
    if(moveInput !== 0 && !gameOver){
      const target = player.lane + moveInput;
      if(target >= 0 && target < track.lanes){
        player.lane = target; moveInput = 0; updatePlayerPos();
      }
    }
    const playerRect = {x:player.x, y:player.y, w:player.w, h:player.h};
    for(const t of traffic){
      const tRect = {x:t.x, y:t.y, w:t.w, h:t.h};
      if(rectsIntersect(playerRect,tRect)){
        triggerExplosion(player.x + player.w/2, player.y + player.h/2);
        gameOver = true; running = false; break;
      }
    }
    for(let i=particles.length-1;i>=0;i--){
      const p = particles[i];
      p.x += p.vx; p.y += p.vy; p.life--; p.vy += 0.06;
      if(p.life <= 0) particles.splice(i,1);
    }
  }
  function draw(){
    const pw = canvas.width / (window.devicePixelRatio||1);
    const ph = canvas.height / (window.devicePixelRatio||1);
    ctx.clearRect(0,0,pw,ph);
    const trackW = Math.min(420, pw - 40);
    const left = (pw - trackW)/2;
    ctx.fillStyle = '#222831'; ctx.fillRect(left,8,trackW,ph-16);
    const laneW = (trackW - track.lanePadding*2) / track.lanes;
    ctx.strokeStyle = 'rgba(255,255,255,0.12)'; ctx.lineWidth = 2;
    for(let i=1;i<track.lanes;i++){
      const lx = left + track.lanePadding + i*laneW;
      for(let y=-40;y<ph+80;y+=40){
        ctx.beginPath(); ctx.moveTo(lx,y); ctx.lineTo(lx,y+20); ctx.stroke();
      }
    }
    for(const t of traffic){
      ctx.fillStyle = t.color;
      ctx.fillRect(t.x,t.y,t.w,t.h);
    }
    if(!gameOver){
      ctx.fillStyle = player.color;
      ctx.fillRect(player.x,player.y,player.w,player.h);
    }
    for(const p of particles){
      ctx.beginPath(); ctx.fillStyle = p.color; ctx.globalAlpha = Math.max(0, p.life/60);
      ctx.arc(p.x,p.y,p.r,0,Math.PI*2); ctx.fill(); ctx.globalAlpha = 1;
    }
    if(gameOver){
      ctx.fillStyle = 'rgba(0,0,0,0.45)'; ctx.fillRect(0,0,pw,ph);
      ctx.fillStyle = '#fff'; ctx.font = 'bold 28px sans-serif'; ctx.textAlign='center';
      ctx.fillText('BOOM! You crashed', pw/2, ph/2 - 22);
      ctx.font = '16px sans-serif'; ctx.fillText('Tap Restart to try again', pw/2, ph/2 + 6);
    }
  }
  let last = performance.now();
  function loop(ts){
    last = ts;
    if(running) gameTick();
    draw();
    requestAnimationFrame(loop);
  }
  requestAnimationFrame(loop);

  const leftBtn = document.getElementById('leftBtn');
  const rightBtn = document.getElementById('rightBtn');
  const pauseBtn = document.getElementById('pause');
  const restartBtn = document.getElementById('restart');

  leftBtn.addEventListener('touchstart', e=>{e.preventDefault(); if(!gameOver) moveInput=-1;});
  leftBtn.addEventListener('touchend', e=>{e.preventDefault(); moveInput=0;});
  rightBtn.addEventListener('touchstart', e=>{e.preventDefault(); if(!gameOver) moveInput=1;});
  rightBtn.addEventListener('touchend', e=>{e.preventDefault(); moveInput=0;});

  pauseBtn.addEventListener('click', ()=>{running=!running; pauseBtn.textContent=running?'Pause':'Resume';});
  restartBtn.addEventListener('click', ()=>{running=true; reset(); pauseBtn.textContent='Pause'; document.getElementById('score').textContent=score; document.getElementById('high').textContent=high;});

  reset(); updatePlayerPos();
})();
</script>
</body>
</html>
