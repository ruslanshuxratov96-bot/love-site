<!doctype html>

<html lang="ru">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Барабан — спиннер валют и чисел</title>
  <style>
    :root{--size:520px}
    body{font-family:Inter, system-ui, Arial;display:flex;align-items:center;justify-content:center;height:100vh;margin:0;background:linear-gradient(180deg,#0f172a,#071033);color:#fff}
    .wrap{width:calc(var(--size) + 40px);text-align:center}
    canvas{width:var(--size);height:var(--size);background:transparent;border-radius:50%;box-shadow:0 10px 30px rgba(0,0,0,.6);display:block;margin:0 auto}
    .pointer{position:relative;height:18px;margin:10px auto 20px;width:6px}
    .pointer::before{content:"";position:absolute;left:50%;transform:translateX(-50%);top:0;width:0;height:0;border-left:10px solid transparent;border-right:10px solid transparent;border-bottom:18px solid #ffdd57;border-radius:2px}
    .controls{display:flex;gap:10px;justify-content:center;align-items:center}
    button{background:#0ea5a1;border:none;padding:10px 16px;border-radius:10px;color:#02111a;font-weight:700;cursor:pointer;box-shadow:0 6px 18px rgba(14,165,161,.12)}
    button:active{transform:translateY(1px)}
    .result{margin-top:14px;font-size:18px;min-height:28px}
    .small{font-size:13px;color:#9aa7b2}
    .muted{opacity:.8}
  </style>
</head>
<body>
  <div class="wrap">
    <canvas id="wheel" width="1024" height="1024"></canvas>
    <div class="pointer" aria-hidden></div>
    <div class="controls">
      <button id="spinBtn">Крутить</button>
      <button id="resetBtn">Сброс (вернуть валюты)</button>
    </div>
    <div class="result" id="result">Нажмите «Крутить», чтобы запустить барабан.</div>
    <div class="small muted">После первого спина барабан автоматически заменится на числа от 1 до 50.</div>
  </div>  <script>
    const canvas = document.getElementById('wheel');
    const ctx = canvas.getContext('2d');
    const spinBtn = document.getElementById('spinBtn');
    const resetBtn = document.getElementById('resetBtn');
    const resultEl = document.getElementById('result');
    // initial segments: валюты
    const initialSegments = [
      'Bitcoin (BTC)', 'Ethereum (ETH)', 'TON', 'USDT', 'USDC', 'Litecoin (LTC)', 'Dogecoin (DOGE)', 'Ripple (XRP)'
    ];
    let segments = initialSegments.slice();
    let isNumbersMode = false; // когда true - на барабане числа 1..50
    const cx = canvas.width/2; // 512
    const cy = canvas.height/2;
    const radius = Math.min(cx,cy) - 20;
    let rotation = 0; // radians
    let angularVelocity = 0;
    let spinning = false;
    function drawWheel(highlightIndex = -1) {
      ctx.clearRect(0,0,canvas.width,canvas.height);
      const n = segments.length;
      const slice = (Math.PI*2)/n;
      ctx.save();
      ctx.translate(cx,cy);
      ctx.rotate(rotation);
      for (let i=0;i<n;i++){
        const start = i*slice;
        const end = start + slice;
        // color by HSL
        const hue = Math.round((i/n)*360);
        ctx.beginPath();
        ctx.moveTo(0,0);
        ctx.arc(0,0,radius,start,end);
        ctx.closePath();
        ctx.fillStyle = `hsl(${hue} 60% 45%)`;
        ctx.fill();
      // highlight selected
        if (i === highlightIndex) {
          ctx.save();
          ctx.shadowColor = 'rgba(255,255,200,0.9)';
          ctx.shadowBlur = 40;
          ctx.lineWidth = 8;
          ctx.strokeStyle = 'rgba(255,255,255,0.95)';
          ctx.stroke();
          ctx.restore();
        } else {
          ctx.lineWidth = 1;
          ctx.strokeStyle = 'rgba(0,0,0,0.2)';
          ctx.stroke();
        }
        // text
        ctx.save();
        const mid = start + slice/2;
        ctx.rotate(mid);
        ctx.translate(radius*0.6,0);
        ctx.rotate(Math.PI/2);
        ctx.fillStyle = 'rgba(255,255,255,0.96)';
        ctx.font = `${Math.max(18, Math.floor(radius/12))}px sans-serif`;
        ctx.textAlign = 'center';
        ctx.textBaseline = 'middle';
        const label = segments[i];
        // wrap long labels
        wrapText(ctx,label, 0, 0, radius*0.9, 20);
        ctx.restore();
      }

      ctx.restore();

      // center circle
      ctx.beginPath();
      ctx.arc(cx,cy,60,0,Math.PI*2);
      ctx.fillStyle = '#071733';
      ctx.fill();
      ctx.lineWidth = 4;
      ctx.strokeStyle = 'rgba(255,255,255,0.06)';
      ctx.stroke();
    }

    function wrapText(ctx, text, x, y, maxWidth, lineHeight) {
      const words = text.split(' ');
      let line = '';
      let lines = [];
      for (let n=0;n<words.length;n++){
        const testLine = line + words[n] + ' ';
        const metrics = ctx.measureText(testLine);
        if (metrics.width > maxWidth && n>0) {
          lines.push(line.trim());
          line = words[n] + ' ';
        } else {
          line = testLine;
        }
      }
      lines.push(line.trim());
      const offset = -((lines.length-1) * lineHeight)/2;
      for (let i=0;i<lines.length;i++){
        ctx.fillText(lines[i], x, y + offset + i*lineHeight);
      }
    }

    // determine index under pointer (pointer at top)
    function getSelectedIndex() {
      const n = segments.length;
      // angle where pointer is pointing relative to wheel rotation
      // pointer at top -> angle = -PI/2 in canvas coordinates
      let angle = -rotation + Math.PI/2; // map rotation to pointer reference
      angle = (angle % (Math.PI*2) + Math.PI*2) % (Math.PI*2);
      const slice = (Math.PI*2)/n;
      const index = Math.floor(angle / slice);
      // because drawing order starts at angle 0 (right side), we need to invert index
      // adjust because our segments are drawn from 0..n-1
      return (n - index) % n;
    }

    // spin physics
    function startSpin() {
      if (spinning) return;
      spinning = true;
      // random spin: between 4 and 8 full rotations + small offset
      const rotations = Math.random()*4 + 4;
      const target = rotations * Math.PI*2 + (Math.random()*Math.PI*2);
      // set initial angular velocity via simple easing
      const duration = 3000 + Math.random()*2000; // ms
      const start = performance.now();
      const initialRot = rotation;

      function animate(now) {
        const t = Math.min(1,(now - start)/duration);
        // easeOutCubic
        const eased = 1 - Math.pow(1-t,3);
        rotation = initialRot + target*eased;
        drawWheel();
        if (t < 1) {
          requestAnimationFrame(animate);
        } else {
          spinning = false;
          const idx = getSelectedIndex();
          highlightAndShow(idx);
        }
      }
      requestAnimationFrame(animate);
    }

    function highlightAndShow(idx) {
      // flash highlight a few times
      let flashes = 6;
      let on = true;
      const flashInterval = 300;
      const timer = setInterval(()=>{
        drawWheel(on ? idx : -1);
        on = !on;
        flashes--;
        if (flashes<=0){
          clearInterval(timer);
          drawWheel(idx);
          const value = segments[idx];
          resultEl.innerHTML = `<strong>Выпало:</strong> ${value}`;

          // after first spin (if currently in initial currency mode), switch to numbers 1..50
          if (!isNumbersMode) {
            setTimeout(()=>{
              switchToNumbers();
            }, 800);
          }
        }
      }, flashInterval);
    }

    function switchToNumbers() {
      // build 1..50
      const nums = Array.from({length:50},(_,i)=>String(i+1));
      segments = nums;
      isNumbersMode = true;
      rotation = 0; // reset rotation for clarity
      resultEl.innerHTML += ' <span class="small">(барабан обновлён — теперь числа 1–50)</span>';
      drawWheel();
    }

    spinBtn.addEventListener('click', startSpin);
    resetBtn.addEventListener('click', ()=>{
      segments = initialSegments.slice();
      isNumbersMode = false;
      rotation = 0;
      resultEl.textContent = 'Вернулись валюты. Нажмите «Крутить». ';
      drawWheel();
    });

    // initial draw
    drawWheel();

    // make canvas look crisp on high-DPI
    function fixDPI(){
      const dpr = Math.max(1, window.devicePixelRatio || 1);
      const w = canvas.clientWidth * dpr;
      const h = canvas.clientHeight * dpr;
      if (canvas.width !== w || canvas.height !== h) {
        canvas.width = w;
        canvas.height = h;
      }
    }
    function resize(){
      fixDPI();
      // update center & radius
      const min = Math.min(canvas.width, canvas.height);
      // keep cx,cy as half of current width/height
      // (we already used cx,cy constants based on initial 1024 — that's fine for our drawing scale)
      drawWheel();
    }
    window.addEventListener('resize', resize);

    // optional: allow spacebar to spin
    window.addEventListener('keydown', (e)=>{ if (e.code==='Space') { e.preventDefault(); startSpin(); } });
  </script></body>
</html>
