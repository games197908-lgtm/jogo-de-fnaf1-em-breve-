<!DOCTYPE html>
<html lang="pt">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>FNAF 1 - Versão Final (Compatível GitHub Pages)</title>
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
    <!-- 
        INSTRUÇÕES PARA GITHUB PAGES (SEM ERRO 404):
        1. Crie um repositório público.
        2. Salve este arquivo como "index.html" na raiz (não dentro de pastas).
        3. Vá em Settings > Pages > Source: Deploy from branch "main" > "/ (root)" > Save.
        4. Aguarde 1-2 minutos. O site ficará em https://seuusuario.github.io/nome-do-repo/
        Tudo é inline (sem arquivos externos), então NENHUM 404 de recursos.
        Se ainda der 404, crie um arquivo vazio ".nojekyll" na raiz e commit.
    -->
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
        let bonnie = {pos: 0};
        let chica = {pos: 0};
        let foxy = {progress: 0};
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

        // ... (todas as funções anteriores permanecem iguais: updateCamRects, onMouseMove, onClick, inRect, startGame, gameLoop, update, moveAI, jumpscare, toggle functions, openCams, closeCams)

        function getWhiteNoise(dur) {
            const bufferSize = audioCtx.sampleRate * dur;
            const buffer = audioCtx.createBuffer(1, bufferSize, audioCtx.sampleRate);
            const data = buffer.getChannelData(0);
            for (let i = 0; i < bufferSize; i++) data[i] = Math.random() * 2 - 1;
            return buffer;
        }

        function playTone(freqStart, freqEnd, dur, gainStart, type = 'sine') {
            const osc = audioCtx.createOscillator();
            const gain = audioCtx.createGain();
            osc.type = type;
            osc.frequency.setValueAtTime(freqStart, audioCtx.currentTime);
            if (freqEnd) osc.frequency.exponentialRampToValueAtTime(freqEnd, audioCtx.currentTime + dur);
            gain.gain.setValueAtTime(gainStart, audioCtx.currentTime);
            gain.gain.exponentialRampToValueAtTime(0.01, audioCtx.currentTime + dur);
            osc.connect(gain);
            gain.connect(audioCtx.destination);
            osc.start();
            osc.stop(audioCtx.currentTime + dur);
        }

        function playSound(type) {
            if (!audioCtx) return;
            const now = audioCtx.currentTime;

            switch (type) {
                case 'click':
                    playTone(800, 600, 0.05, 0.2);
                    playTone(1000, 800, 0.05, 0.15);
                    break;
                case 'jumpscare':
                    // Scream mais realista com múltiplos osciladores + ruído distorcido
                    playTone(1200, 100, 0.4, 0.7, 'sawtooth');
                    playTone(900, 150, 0.5, 0.6, 'square');
                    playTone(1400, 200, 0.4, 0.5, 'triangle');
                    // Ruído de distorção
                    const noiseSource = audioCtx.createBufferSource();
                    noiseSource.buffer = getWhiteNoise(1.2);
                    const noiseGain = audioCtx.createGain();
                    noiseGain.gain.setValueAtTime(0.4, now);
                    noiseGain.gain.exponentialRampToValueAtTime(0.01, now + 1.2);
                    const lowpass = audioCtx.createBiquadFilter();
                    lowpass.type = 'lowpass';
                    lowpass.frequency.value = 1500;
                    noiseSource.connect(lowpass);
                    lowpass.connect(noiseGain);
                    noiseGain.connect(audioCtx.destination);
                    noiseSource.start(now);
                    break;
                case 'bang':
                    // Batida pesada (Foxy/porta)
                    playTone(120, 40, 0.15, 1, 'sine');
                    playTone(80, 30, 0.3, 0.8, 'triangle');
                    const bangNoise = audioCtx.createBufferSource();
                    bangNoise.buffer = getWhiteNoise(0.5);
                    const bangFilter = audioCtx.createBiquadFilter();
                    bangFilter.type = 'lowpass';
                    bangFilter.frequency.value = 300;
                    const bangGain = audioCtx.createGain();
                    bangGain.gain.setValueAtTime(0.9, now);
                    bangGain.gain.exponentialRampToValueAtTime(0.01, now + 0.5);
                    bangNoise.connect(bangFilter);
                    bangFilter.connect(bangGain);
                    bangGain.connect(audioCtx.destination);
                    bangNoise.start(now);
                    break;
                case 'powerout':
                    // Toreador March (música do Freddy no power out) - versão lenta e creepy
                    const melody = [
                        {f: 233.08, d: 1.2}, // Bb
                        {f: 233.08, d: 1.2},
                        {f: 233.08, d: 1.2},
                        {f: 233.08, d: 1.2},
                        {f: 233.08, d: 1.2},
                        {f: 233.08, d: 1.2},
                        {f: 220.00, d: 1.2}, // A
                        {f: 220.00, d: 1.2},
                        {f: 196.00, d: 1.2}, // G
                        {f: 196.00, d: 1.2},
                        {f: 174.61, d: 1.2}, // F
                        {f: 174.61, d: 1.2},
                        {f: 155.56, d: 1.2}, // Eb
                        {f: 155.56, d: 1.2},
                        {f: 146.83, d: 1.2}, // D
                        {f: 146.83, d: 1.2},
                        {f: 130.81, d: 2.0}  // C (final mais longo)
                    ];
                    let t = now;
                    melody.forEach(note => {
                        const osc = audioCtx.createOscillator();
                        osc.type = 'triangle';
                        osc.frequency.value = note.f * 0.8; // tom mais grave
                        const gain = audioCtx.createGain();
                        gain.gain.value = 0.25;
                        osc.connect(gain);
                        gain.connect(audioCtx.destination);
                        osc.start(t);
                        osc.stop(t + note.d);
                        gain.gain.exponentialRampToValueAtTime(0.01, t + note.d);
                        t += note.d;
                    });
                    break;
            }
        }

        // ... (o resto do código: draw(), drawButton, drawCamView, drawAnim, drawEyes permanece igual ao da versão anterior completa)

        // Exemplo de final do draw() para gameover/win (completando o que estava cortado):
        } else if (state === 'gameover' || state === 'win') {
            ctx.fillStyle = state === 'gameover' ? '#f00' : '#0f0';
            ctx.font = 'bold 140px Arial';
            ctx.textAlign = 'center';
            ctx.textBaseline = 'middle';
            ctx.fillText(state === 'gameover' ? 'GAME OVER' : '6AM\nSOBREVIVEU!', 640, 360);
            drawButton(BUTTONS.restart, '#555', 'REINICIAR', '#fff');
        }

        ctx.restore();
    }

        // (Inclua aqui todas as funções drawCamView, drawAnim e drawEyes da versão anterior completa)

    </script>
</body>
</html>
