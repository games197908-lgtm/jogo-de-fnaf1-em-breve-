<!DOCTYPE html>
<html lang="pt">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>FNAF 1 - Versão Completa e Corrigida</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            background: black;
            overflow: hidden;
            font-family: 'Courier New', monospace;
            color: #0f0;
            cursor: pointer;
        }
        #canvas {
            display: block;
            width: 100vw;
            height: 100vh;
            image-rendering: optimizeSpeed;
            image-rendering: -moz-crisp-edges;
            image-rendering: -webkit-optimize-contrast;
            image-rendering: crisp-edges;
            image-rendering: -o-crisp-edges;
            image-rendering: pixelated;
        }
    </style>
</head>
<body>
    <!-- INSTRUÇÕES PARA GITHUB PAGES:
         1. Salve como index.html na raiz do repositório.
         2. Settings > Pages > Branch: main > /(root) > Save.
         3. Se der 404, crie arquivo vazio .nojekyll na raiz.
         Tudo inline → sem erro 404 de recursos. -->
    <canvas id="canvas"></canvas>
    <script>
        let canvas, ctx;
        let state = 'menu';
        let power = 100;
        let minute = 0;
        let timeAccum = 0;
        let aiAccum = 0;
        let powerAccum = 0;
        let poweroutAccum = 0;
        let shake = 0;
        let lastTime = 0;
        let nowTime = 0;
        let audioCtx = null;
        let leftDoorClosed = false;
        let rightDoorClosed = false;
        let leftLightOn = false;
        let rightLightOn = false;
        let selectedCam = '1A';
        let lastSeen = {};

        // Animatronics
        let bonnie = {pos: 0}; // 0: stage, 1: dining, 2: west hall, 3: left door
        let chica = {pos: 0}; // 0: stage, 1: dining, 2: east hall, 3: right door
        let foxy = {progress: 0}; // 0-5
        let freddy = {pos: 0};
        let freddyInOffice = false;

        const CAM_LIST = ['1A', '1B', '2', '3', '4', '5', '6', '7', '8', '9', '10', '11'];

        const BUTTONS = {
            leftLight: {x: 250, y: 430, w: 80, h: 40},
            leftDoor: {x: 250, y: 480, w: 80, h: 40},
            rightLight: {x: 950, y: 430, w: 80, h: 40},
            rightDoor: {x: 950, y: 480, w: 80, h: 40},
            camera: {x: 590, y: 530, w: 120, h: 40},
            closeCam: {x: 1100, y: 600, w: 120, h: 50},
            restart: {x: 500, y: 450, w: 280, h: 60},
            start: {x: 400, y: 450, w: 480, h: 80}
        };

        let camRects = [];

        function init() {
            canvas = document.getElementById('canvas');
            if (!canvas) return;
            ctx = canvas.getContext('2d');
            resizeCanvas();
            window.addEventListener('resize', resizeCanvas);
            canvas.addEventListener('click', onClick);
            canvas.addEventListener('mousemove', onMouseMove);
            updateCamRects();
            requestAnimationFrame(gameLoop);
        }

        function resizeCanvas() {
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight;
        }

        function updateCamRects() {
            camRects = [];
            let mapX = 50, mapY = 200, camSize = 50;
            for (let i = 0; i < CAM_LIST.length; i++) {
                let cx = mapX + (i % 4) * 55;
                let cy = mapY + Math.floor(i / 4) * 55;
                camRects.push({cam: CAM_LIST[i], x: cx, y: cy, w: camSize, h: camSize / 2});
            }
        }

        function onMouseMove(e) {
            const rect = canvas.getBoundingClientRect();
            const scaleX = 1280 / rect.width;
            const scaleY = 720 / rect.height;
            // Não usado diretamente, mas mantido
        }

        function onClick(e) {
            const rect = canvas.getBoundingClientRect();
            const scaleX = 1280 / rect.width;
            const scaleY = 720 / rect.height;
            let x = (e.clientX - rect.left) * scaleX;
            let y = (e.clientY - rect.top) * scaleY;

            if (!audioCtx) {
                audioCtx = new (window.AudioContext || window.webkitAudioContext)();
            }

            if (state === 'menu') {
                if (inRect(x, y, BUTTONS.start)) startGame();
            } else if (state === 'office') {
                if (inRect(x, y, BUTTONS.leftLight)) toggleLeftLight();
                if (inRect(x, y, BUTTONS.leftDoor)) toggleLeftDoor();
                if (inRect(x, y, BUTTONS.rightLight)) toggleRightLight();
                if (inRect(x, y, BUTTONS.rightDoor)) toggleRightDoor();
                if (inRect(x, y, BUTTONS.camera)) openCams();
            } else if (state === 'cams') {
                if (inRect(x, y, BUTTONS.closeCam)) closeCams();
                for (let r of camRects) {
                    if (inRect(x, y, r)) {
                        selectedCam = r.cam;
                        playSound('click');
                    }
                }
            } else if (state === 'gameover' || state === 'win') {
                if (inRect(x, y, BUTTONS.restart)) startGame();
            }
        }

        function inRect(x, y, r) {
            return x > r.x && x < r.x + r.w && y > r.y && y < r.y + r.h;
        }

        function startGame() {
            power = 100;
            minute = 0;
            timeAccum = aiAccum = powerAccum = poweroutAccum = shake = 0;
            leftDoorClosed = rightDoorClosed = leftLightOn = rightLightOn = false;
            bonnie.pos = chica.pos = foxy.progress = freddy.pos = 0;
            freddyInOffice = false;
            lastSeen = {};
            state = 'office';
            playSound('click');
        }

        function gameLoop(time) {
            if (!lastTime) lastTime = time;
            const dt = time - lastTime;
            lastTime = time;
            nowTime = time;
            update(dt);
            draw();
            requestAnimationFrame(gameLoop);
        }

        function update(dt) {
            if (state !== 'office' && state !== 'cams') return;

            timeAccum += dt;
            while (timeAccum > 2000) {
                timeAccum -= 2000;
                minute++;
                if (minute >= 360) {
                    state = 'win';
                    return;
                }
            }

            aiAccum += dt;
            if (aiAccum > 3000) {
                aiAccum = 0;
                moveAI();
            }

            let drainRate = 0.25;
            if (leftLightOn) drainRate += 0.75;
            if (rightLightOn) drainRate += 0.75;
            if (leftDoorClosed) drainRate += 1;
            if (rightDoorClosed) drainRate += 1;
            if (state === 'cams') drainRate += 1;
            powerAccum += drainRate * (dt / 1000);
            if (powerAccum >= 1) {
                powerAccum -= 1;
                power = Math.max(0, power - 1);
                if (power <= 0) {
                    state = 'powerout';
                    playSound('powerout');
                }
            }

            shake *= 0.92;
        }

        function moveAI() {
            const hour = Math.floor(minute / 60);
            const agg = 0.015 + hour * 0.004;

            // Bonnie
            if (Math.random() < agg) {
                if (bonnie.pos < 3) bonnie.pos++;
                else if (leftLightOn && Math.random() < 0.6) bonnie.pos--;
                else if (!leftDoorClosed) jumpscare('bonnie');
                else if (Math.random() < 0.3) bonnie.pos--;
            }

            // Chica
            if (Math.random() < agg) {
                if (chica.pos < 3) chica.pos++;
                else if (rightLightOn && Math.random() < 0.6) chica.pos--;
                else if (!rightDoorClosed) jumpscare('chica');
                else if (Math.random() < 0.3) chica.pos--;
            }

            // Foxy
            if (Math.random() < agg * 1.5) foxy.progress = Math.min(5, foxy.progress + 1);
            if (foxy.progress >= 5) {
                if (!leftDoorClosed) jumpscare('foxy');
                else {
                    shake = 30;
                    power = Math.max(0, power - 8);
                    playSound('bang');
                    foxy.progress = 0;
                }
            }
            if (lastSeen.foxy && (nowTime - lastSeen.foxy) < 15000) foxy.progress = Math.max(0, foxy.progress - 1);

            // Freddy
            if (hour >= 1 && Math.random() < agg * 0.6) {
                if (freddy.pos < 2) freddy.pos++;
                else if (!rightDoorClosed) jumpscare('freddy');
                else if (Math.random() < 0.25) freddy.pos--;
            }
            if (hour >= 4 && power < 40 && Math.random() < 0.002) {
                freddyInOffice = true;
                setTimeout(() => freddyInOffice = false, 3000);
            }
        }

        function jumpscare(anim) {
            playSound('jumpscare');
            shake = 60;
            state = 'gameover';
        }

        function toggleLeftLight() { leftLightOn = !leftLightOn; playSound('click'); }
        function toggleLeftDoor() { leftDoorClosed = !leftDoorClosed; playSound('click'); }
        function toggleRightLight() { rightLightOn = !rightLightOn; playSound('click'); }
        function toggleRightDoor() { rightDoorClosed = !rightDoorClosed; playSound('click'); }
        function openCams() { state = 'cams'; playSound('click'); }
        function closeCams() { state = 'office'; playSound('click'); }

        function getWhiteNoise(dur) {
            const bufferSize = audioCtx.sampleRate * dur;
            const buffer = audioCtx.createBuffer(1, bufferSize, audioCtx.sampleRate);
            const data = buffer.getChannelData(0);
            for (let i = 0; i < bufferSize; i++) data[i] = Math.random() * 2 - 1;
            return buffer;
        }

        function playSound(type) {
            if (!audioCtx) return;
            const now = audioCtx.currentTime;

            switch (type) {
                case 'click':
                    const osc1 = audioCtx.createOscillator();
                    osc1.frequency.value = 800;
                    const gain1 = audioCtx.createGain();
                    gain1.gain.setValueAtTime(0.2, now);
                    gain1.gain.exponentialRampToValueAtTime(0.01, now + 0.05);
                    osc1.connect(gain1); gain1.connect(audioCtx.destination);
                    osc1.start(now); osc1.stop(now + 0.05);
                    break;
                case 'jumpscare':
                    playTone(1200, 100, 0.4, 0.7, 'sawtooth');
                    playTone(900, 150, 0.5, 0.6, 'square');
                    playTone(1400, 200, 0.4, 0.5, 'triangle');
                    const noise = audioCtx.createBufferSource();
                    noise.buffer = getWhiteNoise(1.2);
                    const ng = audioCtx.createGain();
                    ng.gain.setValueAtTime(0.4, now);
                    ng.gain.exponentialRampToValueAtTime(0.01, now + 1.2);
                    const lp = audioCtx.createBiquadFilter();
                    lp.type = 'lowpass'; lp.frequency.value = 1500;
                    noise.connect(lp); lp.connect(ng); ng.connect(audioCtx.destination);
                    noise.start(now);
                    break;
                case 'bang':
                    playTone(120, 40, 0.15, 1, 'sine');
                    playTone(80, 30, 0.3, 0.8, 'triangle');
                    const bangNoise = audioCtx.createBufferSource();
                    bangNoise.buffer = getWhiteNoise(0.5);
                    const bf = audioCtx.createBiquadFilter();
                    bf.type = 'lowpass'; bf.frequency.value = 300;
                    const bg = audioCtx.createGain();
                    bg.gain.setValueAtTime(0.9, now);
                    bg.gain.exponentialRampToValueAtTime(0.01, now + 0.5);
                    bangNoise.connect(bf); bf.connect(bg); bg.connect(audioCtx.destination);
                    bangNoise.start(now);
                    break;
                case 'powerout':
                    const melody = [233.08,233.08,233.08,233.08,233.08,233.08,220,220,196,196,174.61,174.61,155.56,155.56,146.83,146.83,130.81];
                    let t = now;
                    melody.forEach((f, i) => {
                        const d = i === 16 ? 2 : 1.2;
                        const osc = audioCtx.createOscillator();
                        osc.type = 'triangle';
                        osc.frequency.value = f * 0.8;
                        const g = audioCtx.createGain();
                        g.gain.value = 0.25;
                        osc.connect(g); g.connect(audioCtx.destination);
                        osc.start(t); osc.stop(t + d);
                        g.gain.exponentialRampToValueAtTime(0.01, t + d);
                        t += d;
                    });
                    break;
            }

            function playTone(f1, f2, dur, vol, type) {
                const osc = audioCtx.createOscillator();
                osc.type = type || 'sine';
                osc.frequency.setValueAtTime(f1, now);
                if (f2) osc.frequency.exponentialRampToValueAtTime(f2, now + dur);
                const g = audioCtx.createGain();
                g.gain.setValueAtTime(vol, now);
                g.gain.exponentialRampToValueAtTime(0.01, now + dur);
                osc.connect(g); g.connect(audioCtx.destination);
                osc.start(now); osc.stop(now + dur);
            }
        }

        function drawButton(r, bg, text, fg) {
            ctx.fillStyle = bg;
            ctx.fillRect(r.x, r.y, r.w, r.h);
            ctx.strokeStyle = '#666';
            ctx.lineWidth = 2;
            ctx.strokeRect(r.x, r.y, r.w, r.h);
            ctx.fillStyle = fg;
            ctx.font = 'bold 20px Courier New';
            ctx.textAlign = 'center';
            ctx.textBaseline = 'middle';
            ctx.fillText(text, r.x + r.w / 2, r.y + r.h / 2);
        }

        function drawAnim(type, x, y, s) {
            ctx.save();
            ctx.translate(x, y);
            ctx.scale(s, s);
            let bodyColor = type === 'bonnie' ? '#7744cc' : type === 'chica' ? '#ffaa44' : type === 'foxy' ? '#ff8800' : '#aa6633';
            ctx.fillStyle = bodyColor;
            ctx.beginPath();
            ctx.ellipse(0, 60, 45, 70, 0, 0, Math.PI * 2);
            ctx.fill();
            ctx.beginPath();
            ctx.ellipse(0, 5, 40, 45, 0, 0, Math.PI * 2);
            ctx.fill();
            ctx.fillStyle = 'white';
            ctx.beginPath();
            ctx.arc(-18, -8, 14, 0, Math.PI * 2);
            ctx.arc(18, -8, 14, 0, Math.PI * 2);
            ctx.fill();
            ctx.fillStyle = 'black';
            ctx.beginPath();
            ctx.arc(-18, -8, 7, 0, Math.PI * 2);
            ctx.arc(18, -8, 7, 0, Math.PI * 2);
            ctx.fill();
            if (type === 'freddy') {
                ctx.fillStyle = 'black';
                ctx.fillRect(-45, -70, 90, 25);
                ctx.fillRect(-35, -85, 70, 15);
            }
            ctx.restore();
        }

        function drawEyes(x, y, size) {
            ctx.save();
            ctx.translate(x, y);
            ctx.scale(size, size);
            ctx.fillStyle = 'white';
            ctx.beginPath();
            ctx.arc(-18, -8, 14, 0, Math.PI * 2);
            ctx.arc(18, -8, 14, 0, Math.PI * 2);
            ctx.fill();
            ctx.fillStyle = 'black';
            ctx.beginPath();
            ctx.arc(-18, -8, 7, 0, Math.PI * 2);
            ctx.arc(18, -8, 7, 0, Math.PI * 2);
            ctx.fill();
            ctx.restore();
        }

        function drawCamView(cam) {
            // Implementação simplificada - apenas palco e pirate cove para teste
            if (cam === '1A' || cam === '1B') {
                if (bonnie.pos === 0) drawAnim('bonnie', 420, 380, 0.85);
                if (chica.pos === 0) drawAnim('chica', 720, 380, 0.85);
                if (freddy.pos === 0) drawAnim('freddy', 570, 370, 1.1);
            } else if (cam === '3') {
                if (foxy.progress > 0) {
                    let fx = 500 + foxy.progress * 30;
                    drawAnim('foxy', fx, 420, 0.5 + foxy.progress * 0.1);
                    lastSeen.foxy = nowTime;
                }
            }
        }

        function draw() {
            ctx.fillStyle = 'black';
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            const scaleX = canvas.width / 1280;
            const scaleY = canvas.height / 720;
            ctx.save();
            ctx.scale(scaleX, scaleY);

            const sx = shake * (Math.random() - 0.5) * 2;
            const sy = shake * (Math.random() - 0.5) * 2;
            ctx.translate(sx, sy);

            const hour = Math.floor(minute / 60);
            const dispHour = hour === 0 ? 12 : hour;
            const timeStr = `${dispHour} AM`;

            // Power bar
            ctx.fillStyle = '#333';
            ctx.fillRect(50, 100, 30, 400);
            ctx.fillStyle = power > 20 ? '#0f0' : power > 5 ? '#f80' : '#f00';
            ctx.fillRect(50, 500 - (power * 4), 30, power * 4);

            // Time
            ctx.fillStyle = '#fff';
            ctx.font = 'bold 60px Courier New';
            ctx.textAlign = 'right';
            ctx.fillText(timeStr, 1230, 80);

            if (state === 'menu') {
                ctx.fillStyle = '#f00';
                ctx.font = 'bold 120px Arial';
                ctx.textAlign = 'center';
                ctx.fillText('FNAF 1', 640, 320);
                ctx.fillStyle = '#ff0';
                ctx.font = 'bold 50px Courier New';
                ctx.fillText('SOBREVIVA ATÉ 6AM', 640, 420);
                drawButton(BUTTONS.start, '#555', 'COMEÇAR', '#fff');
            } else if (state === 'powerout') {
                poweroutAccum += dt || 16;
                ctx.fillStyle = 'rgba(0,0,0,0.95)';
                ctx.fillRect(0, 0, 1280, 720);
                if (Math.floor(nowTime / 150) % 2) drawEyes(640, 360, 3);
                if (poweroutAccum > 15000) jumpscare('freddy');
            } else if (state === 'office') {
                ctx.fillStyle = '#444';
                ctx.fillRect(0, 0, 1280, 150);
                ctx.fillRect(0, 150, 200, 420);
                ctx.fillRect(1080, 150, 200, 420);
                ctx.fillStyle = '#663300';
                ctx.fillRect(300, 400, 680, 200);
                ctx.fillStyle = '#552200';
                ctx.fillRect(300, 400, 680, 30);

                // Monitores laterais
                let monX = 360, monY = 430;
                ctx.fillStyle = '#111';
                ctx.fillRect(monX, monY, 100, 70);
                ctx.strokeStyle = '#333';
                ctx.lineWidth = 3;
                ctx.strokeRect(monX, monY, 100, 70);
                if (leftLightOn && bonnie.pos === 3) drawAnim('bonnie', monX + 75, monY + 35, 0.35);

                let rmonX = 820;
                ctx.fillRect(rmonX, monY, 100, 70);
                ctx.strokeRect(rmonX, monY, 100, 70);
                if (rightLightOn && chica.pos === 3) drawAnim('chica', rmonX + 25, monY + 35, 0.35);

                drawButton(BUTTONS.leftLight, leftLightOn ? '#ff0' : '#555', 'LIGHT', leftLightOn ? '#000' : '#fff');
                drawButton(BUTTONS.leftDoor, leftDoorClosed ? '#f00' : '#555', leftDoorClosed ? 'CLOSE' : 'OPEN', '#fff');
                drawButton(BUTTONS.rightLight, rightLightOn ? '#ff0' : '#555', 'LIGHT', rightLightOn ? '#000' : '#fff');
                drawButton(BUTTONS.rightDoor, rightDoorClosed ? '#f00' : '#555', rightDoorClosed ? 'CLOSE' : 'OPEN', '#fff');
                drawButton(BUTTONS.camera, '#555', 'CAMERA', '#fff');
            } else if (state === 'cams') {
                ctx.fillStyle = '#0a0';
                ctx.fillRect(200, 150, 880, 420);
                for (let i 
