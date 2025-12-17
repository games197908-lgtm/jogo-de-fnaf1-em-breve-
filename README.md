<!DOCTYPE html>
<html lang="pt">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>FNAF 1 - Versão Melhorada</title>
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
    <canvas id="canvas"></canvas>
    <script>
        let canvas, ctx;
        let state = 'menu';
        let power = 100;
        let minute = 0;
        let gameStartTime = 0;
        let timeAccum = 0;
        let aiAccum = 0;
        let powerAccum = 0;
        let poweroutAccum = 0;
        let shake = 0;
        let lastTime = 0;
        let mouseX = 0, mouseY = 0;
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
        let freddy = {pos: 0}; // 0: stage, 1: dining/kitchen, 2: right door
        let freddyInOffice = false;

        const CAM_LIST = ['1A', '1B', '2', '3', '4', '5', '6', '7', '8', '9', '10', '11'];

        // Button rects (logical canvas coords 1280x720)
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

        // Cam map rects
        let camRects = [];

        init();

        function init() {
            canvas = document.getElementById('canvas');
            ctx = canvas.getContext('2d');
            canvas.width = 1280;
            canvas.height = 720;
            canvas.addEventListener('click', onClick);
            canvas.addEventListener('mousemove', onMouseMove);
            updateCamRects();
            requestAnimationFrame(gameLoop);
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
            const scaleX = canvas.width / rect.width;
            const scaleY = canvas.height / rect.height;
            mouseX = (e.clientX - rect.left) * scaleX;
            mouseY = (e.clientY - rect.top) * scaleY;
        }

        function onClick(e) {
            const rect = canvas.getBoundingClientRect();
            const scaleX = canvas.width / rect.width;
            const scaleY = canvas.height / rect.height;
            let x = (e.clientX - rect.left) * scaleX;
            let y = (e.clientY - rect.top) * scaleY;

            if (state === 'menu') {
                if (x > BUTTONS.start.x && x < BUTTONS.start.x + BUTTONS.start.w &&
                    y > BUTTONS.start.y && y < BUTTONS.start.y + BUTTONS.start.h) {
                    startGame();
                }
            } else if (state === 'office' || state === 'cams') {
                if (state === 'office') {
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
                            break;
                        }
                    }
                }
            } else if (state === 'gameover' || state === 'win') {
                if (inRect(x, y, BUTTONS.restart)) startGame();
            }

            // Init audio on first interaction
            if (!audioCtx) {
                audioCtx = new (window.AudioContext || window.webkitAudioContext)();
            }
        }

        function inRect(x, y, r) {
            return x > r.x && x < r.x + r.w && y > r.y && y < r.y + r.h;
        }

        function startGame() {
            power = 100;
            minute = 0;
            gameStartTime = performance.now();
            timeAccum = 0;
            aiAccum = 0;
            powerAccum = 0;
            poweroutAccum = 0;
            shake = 0;
            leftDoorClosed = false;
            rightDoorClosed = false;
            leftLightOn = false;
            rightLightOn = false;
            bonnie.pos = 0;
            chica.pos = 0;
            foxy.progress = 0;
            freddy.pos = 0;
            freddyInOffice = false;
            state = 'office';
        }

        function gameLoop(time) {
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
            while (timeAccum > 2000) { // ~2s per minute
                timeAccum -= 2000;
                minute++;
                if (minute >= 360) {
                    state = 'win';
                    return;
                }
            }

            aiAccum += dt;
            if (aiAccum > 3000) { // AI tick every ~3s
                aiAccum = 0;
                moveAI();
            }

            // Power drain (% per sec base 0.25)
            let drainRate = 0.25;
            if (leftLightOn) drainRate += 0.75;
            if (rightLightOn) drainRate += 0.75;
            if (leftDoorClosed) drainRate += 1;
            if (rightDoorClosed) drainRate += 1;
            if (state === 'cams') drainRate += 1;
            powerAccum += drainRate * dt;
            while (powerAccum >= 1) {
                powerAccum -= 1;
                power = Math.max(0, power - 1);
                if (power <= 0) {
                    state = 'powerout';
                    playSound('powerout');
                    return;
                }
            }

            shake *= 0.92;
        }

        function moveAI() {
            const hour = Math.floor(minute / 60);
            const agg = 0.015 + hour * 0.004;

            // Bonnie
            if (Math.random() < agg) {
                if (bonnie.pos < 3) {
                    bonnie.pos++;
                } else {
                    if (leftLightOn && Math.random() < 0.6) {
                        bonnie.pos--;
                    } else if (!leftDoorClosed) {
                        jumpscare('bonnie');
                    } else if (Math.random() < 0.3) {
                        bonnie.pos--;
                    }
                }
            }

            // Chica
            if (Math.random() < agg) {
                if (chica.pos < 3) {
                    chica.pos++;
                } else {
                    if (rightLightOn && Math.random() < 0.6) {
                        chica.pos--;
                    } else if (!rightDoorClosed) {
                        jumpscare('chica');
                    } else if (Math.random() < 0.3) {
                        chica.pos--;
                    }
                }
            }

            // Foxy
            const foxyAgg = agg * 1.5;
            if (Math.random() < foxyAgg) {
                foxy.progress = Math.min(5, foxy.progress + 1);
            }
            if (foxy.progress >= 5) {
                if (!leftDoorClosed) {
                    jumpscare('foxy');
                } else {
                    shake = 30;
                    power = Math.max(0, power - 8);
                    playSound('bang');
                    foxy.progress = 0;
                }
            }
            // Reset if recently seen
            if (lastSeen.foxy && (nowTime - lastSeen.foxy) < 15000) {
                foxy.progress = Math.max(0, foxy.progress - 1);
            }

            // Freddy (slower, after 1AM)
            const fredAgg = agg * 0.6;
            if (hour >= 1 && Math.random() < fredAgg) {
                if (freddy.pos < 2) {
                    freddy.pos++;
                } else {
                    if (!rightDoorClosed) {
                        jumpscare('freddy');
                    } else if (Math.random() < 0.25) {
                        freddy.pos--;
                    }
                }
            }
            // Hallucination
            if (hour >= 4 && power < 40 && Math.random() < 0.002) {
                freddyInOffice = true;
                setTimeout(() => { freddyInOffice = false; }, 3000);
            }
        }

        function jumpscare(anim) {
            playSound('jumpscare');
            shake = 60;
            state = 'gameover';
        }

        function toggleLeftLight() {
            leftLightOn = !leftLightOn;
            playSound('click');
        }

        function toggleLeftDoor() {
            leftDoorClosed = !leftDoorClosed;
            playSound('click');
        }

        function toggleRightLight() {
            rightLightOn = !rightLightOn;
            playSound('click');
        }

        function toggleRightDoor() {
            rightDoorClosed = !rightDoorClosed;
            playSound('click');
        }

        function openCams() {
            state = 'cams';
            playSound('click');
        }

        function closeCams() {
            state = 'office';
            playSound('click');
        }

        function playSound(type) {
            if (!audioCtx) return;
            const audioNow = audioCtx.currentTime;

            function playTone(freqStart, freqEnd, dur, gainStart, type = 'sine') {
                const osc = audioCtx.createOscillator();
                const gain = audioCtx.createGain();
                osc.connect(gain);
                gain.connect(audioCtx.destination);
                osc.type = type;
                osc.frequency.setValueAtTime(freqStart, audioNow);
                osc.frequency.exponentialRampToValueAtTime(freqEnd, audioNow + dur);
                gain.gain.setValueAtTime(gainStart, audioNow);
                gain.gain.exponentialRampToValueAtTime(0.01, audioNow + dur);
                osc.start(audioNow);
                osc.stop(audioNow + dur);
            }

            switch (type) {
                case 'click':
                    playTone(1200, 300, 0.08, 0.15);
                    break;
                case 'jumpscare':
                    playTone(2000, 80, 1.8, 0.45, 'sawtooth');
                    break;
                case 'bang':
                    playTone(60, 40, 0.4, 0.6, 'square');
                    break;
                case 'powerout':
                    playTone(523, 392, 0.3, 0.12);
                    setTimeout(() => playTone(440, 330, 0.3, 0.18), 400);
                    setTimeout(() => playTone(392, 262, 0.3, 0.25), 800);
                    break;
            }
        }

        function draw() {
            ctx.fillStyle = 'black';
            ctx.fillRect(0, 0, 1280, 720);

            const sx = shake * (Math.random() - 0.5) * 2;
            const sy = shake * (Math.random() - 0.5) * 2;
            ctx.save();
            ctx.translate(sx, sy);

            const hour = Math.floor(minute / 60);
            const dispHour = hour === 0 ? 12 : hour;
            const timeStr = `${dispHour} AM`;
            const progress = Math.min(1, minute / 360);

            // Power bar (left)
            const barX = 50, barY = 100, barW = 30, barH = 400;
            ctx.fillStyle = '#333';
            ctx.fillRect(barX, barY, barW, barH);
            ctx.fillStyle = power > 20 ? '#0f0' : power > 5 ? '#f80' : '#f00';
            ctx.fillRect(barX, barY + barH - (power / 100 * barH), barW, power / 100 * barH);

            // Time (top right)
            ctx.fillStyle = '#fff';
            ctx.font = 'bold 60px Courier New';
            ctx.textAlign = 'right';
            ctx.textBaseline = 'middle';
            ctx.fillText(timeStr, 1230, 80);

            // Progress bar (bottom)
            ctx.fillStyle = '#333';
            ctx.fillRect(0, 620, 1280, 100);
            const grad = ctx.createLinearGradient(0, 620, 1280 * progress, 620);
            grad.addColorStop(0, '#0a0');
            grad.addColorStop(1, '#080');
            ctx.fillStyle = grad;
            ctx.fillRect(0, 620, 1280 * progress, 100);

            if (state === 'menu') {
                ctx.fillStyle = '#f00';
                ctx.font = 'bold 120px Arial';
                ctx.textAlign = 'center';
                ctx.textBaseline = 'middle';
                ctx.fillText('FNAF 1', 640, 320);
                ctx.fillStyle = '#ff0';
                ctx.font = 'bold 50px Courier New';
                ctx.fillText('SOBREVIVA ATÉ 6AM', 640, 420);
                ctx.fillStyle = '#555';
                ctx.fillRect(BUTTONS.start.x, BUTTONS.start.y, BUTTONS.start.w, BUTTONS.start.h);
                ctx.fillStyle = '#fff';
                ctx.font = 'bold 40px Courier New';
                ctx.fillText('COMEÇAR', 640, BUTTONS.start.y + 45);
            } else if (state === 'powerout') {
                poweroutAccum += (nowTime - lastTime);
                ctx.fillStyle = 'rgba(0,0,0,0.95)';
                ctx.fillRect(0, 0, 1280, 720);
                if (Math.floor(nowTime / 150) % 2) {
                    drawEyes(640, 360, 3); // big eyes
                }
                if (poweroutAccum > 15000) {
                    jumpscare('freddy');
                }
            } else if (state === 'office') {
                // Office background
                ctx.fillStyle = '#444';
                ctx.fillRect(0, 0, 1280, 150); // ceiling
                ctx.fillRect(0, 150, 200, 420); // left wall
                ctx.fillRect(1080, 150, 200, 420); // right wall
                ctx.fillRect(0, 570, 1280, 150); // floor?

                // Desk
                ctx.fillStyle = '#663300';
                ctx.fillRect(300, 400, 680, 200);
                ctx.fillStyle = '#552200';
                ctx.fillRect(300, 400, 680, 30); // front

                // Left monitor
                let monX = 360, monY = 430, monW = 100, monH = 70;
                ctx.fillStyle = '#111';
                ctx.fillRect(monX, monY, monW, monH);
                ctx.strokeStyle = '#333';
                ctx.lineWidth = 3;
                ctx.strokeRect(monX, monY, monW, monH);
                if (leftLightOn) {
                    ctx.fillStyle = '#222';
                    ctx.fillRect(monX + 5, monY + 5, monW - 10, monH - 10);
                    // Hallway perspective
                    ctx.strokeStyle = '#555';
                    ctx.lineWidth = 2;
                    ctx.beginPath();
                    ctx.moveTo(monX + 5, monY + monH - 5);
                    ctx.lineTo(monX + monW - 5, monY + 20);
                    ctx.moveTo(monX + 5, monY + 20);
                    ctx.lineTo(monX + monW - 5, monY + monH - 5);
                    ctx.stroke();
                    if (bonnie.pos === 3) {
                        drawAnim('bonnie', monX + monW - 25, monY + 35, 0.35);
                    }
                }

                // Right monitor
                let rmonX = 820, rmonY = 430;
                ctx.fillStyle = '#111';
                ctx.fillRect(rmonX, rmonY, monW, monH);
                ctx.strokeStyle = '#333';
                ctx.lineWidth = 3;
                ctx.strokeRect(rmonX, rmonY, monW, monH);
                if (rightLightOn) {
                    ctx.fillStyle = '#222';
                    ctx.fillRect(rmonX + 5, rmonY + 5, monW - 10, monH - 10);
                    ctx.strokeStyle = '#555';
                    ctx.lineWidth = 2;
                    ctx.beginPath();
                    ctx.moveTo(rmonX + 5, rmonY + monH - 5);
                    ctx.lineTo(rmonX + monW - 5, rmonY + 20);
                    ctx.moveTo(rmonX + 5, rmonY + 20);
                    ctx.lineTo(rmonX + monW - 5, rmonY + monH - 5);
                    ctx.stroke();
                    if (chica.pos === 3) {
                        drawAnim('chica', rmonX + 25, rmonY + 35, 0.35);
                    }
                }

                // Buttons
                drawButton(BUTTONS.leftLight, leftLightOn ? '#ff0' : '#555', 'LIGHT', leftLightOn ? '#000' : '#fff');
                drawButton(BUTTONS.leftDoor, leftDoorClosed ? '#f00' : '#555', leftDoorClosed ? 'CLOSE' : 'OPEN', '#fff');
                drawButton(BUTTONS.rightLight, rightLightOn ? '#ff0' : '#555', 'LIGHT', rightLightOn ? '#000' : '#fff');
                drawButton(BUTTONS.rightDoor, rightDoorClosed ? '#f00' : '#555', rightDoorClosed ? 'CLOSE' : 'OPEN', '#fff');
                drawButton(BUTTONS.camera, state === 'cams' ? '#00f' : '#555', 'CAMERA', '#fff');
            } else if (state === 'cams') {
                // Camera view
                ctx.fillStyle = '#0a0';
                ctx.fillRect(200, 150, 880, 420);
                // Static noise
                for (let i = 0; i < 100; i++) {
                    let nx = 200 + Math.random() * 880;
                    let ny = 150 + Math.random() * 420;
                    ctx.fillStyle = `rgba(0, 200, 0, ${Math.random() * 0.4 + 0.1})`;
                    ctx.fillRect(nx, ny, 3, 3);
                }

                drawCamView(selectedCam);

                // Cam map
                let mapX = 50, mapY = 200, camSize = 50;
                for (let i = 0; i < CAM_LIST.length; i++) {
                    let cx = mapX + (i % 4) * 55;
                    let cy = mapY + Math.floor(i / 4) * 55;
                    let isSel = CAM_LIST[i] === selectedCam;
                    ctx.fillStyle = isSel ? '#f0f' : '#333';
                    ctx.fillRect(cx, cy, camSize, camSize / 2);
                    ctx.strokeStyle = '#666';
                    ctx.lineWidth = 2;
                    ctx.strokeRect(cx, cy, camSize, camSize / 2);
                    ctx.fillStyle = '#fff';
                    ctx.font = 'bold 22px Courier New';
                    ctx.textAlign = 'center';
                    ctx.textBaseline = 'middle';
                    ctx.fillText(CAM_LIST[i], cx + 25, cy + 12.5);
                }

                drawButton(BUTTONS.closeCam, '#555', 'CLOSE', '#fff');
            } else if (state === 'gameover' || state === 'win') {
                ctx.fillStyle = state === 'gameover' ? '#f00' : '#0f0';
                ctx.font = 'bold 140px Arial';
  
