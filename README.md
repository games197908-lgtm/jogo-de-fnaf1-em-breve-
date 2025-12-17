<!DOCTYPE html>
<html lang="pt">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ScottGames</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            background: #000;
            color: #fff;
            font-family: 'Courier New', monospace;
            overflow: hidden;
            height: 100vh;
            position: relative;
        }
        canvas { position: fixed; top: 0; left: 0; width: 100%; height: 100%; opacity: 0.1; z-index: 1; }
        .content {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            text-align: center;
            z-index: 2;
            animation: glitch 2s infinite, piscar 1s infinite;
        }
        .fnaf {
            font-size: 8vw;
            color: #ffff00;
            text-shadow: 0 0 30px #ffff00;
            margin-bottom: 0.2em;
            letter-spacing: 0.1em;
        }
        .texto {
            font-size: 4vw;
            color: #ff0000;
            text-shadow: 0 0 20px #ff0000;
            letter-spacing: 0.05em;
        }
        @keyframes piscar {
            0%, 100% { opacity: 1; }
            50% { opacity: 0.2; }
        }
        @keyframes glitch {
            0%, 100% { transform: translate(0); text-shadow: 0.05em 0 0 rgba(255,0,0,.75), -0.05em -0.025em 0 rgba(0,255,255,.75), 0 -0.05em 0 rgba(255,255,0,.75); }
            15% { transform: translate(-0.02em, 0.02em); text-shadow: -0.05em 0 0 rgba(255,0,0,.75), 0.05em 0.025em 0 rgba(0,255,255,.75), 0 -0.05em 0 rgba(255,255,0,.75); }
            50% { transform: translate(0.02em, -0.02em); text-shadow: 0.05em 0 0 rgba(255,0,0,.75), -0.05em 0 0 rgba(0,255,255,.75), 0 0.05em 0 rgba(255,255,0,.75); }
        }
        @media (max-width: 768px) {
            .fnaf { font-size: 12vw; }
            .texto { font-size: 6vw; }
        }
    </style>
</head>
<body>
    <canvas id="static"></canvas>
    <div class="content">
        <div class="fnaf">FNAF 1</div>
        <div class="texto">EM BREVE SERÁ LANÇADO</div>
    </div>
    <script>
        const canvas = document.getElementById('static');
        const ctx = canvas.getContext('2d');
        canvas.width = window.innerWidth;
        canvas.height = window.innerHeight;
        function noise() {
            const imageData = ctx.createImageData(canvas.width, canvas.height);
            const buffer = new Uint32Array(imageData.data.buffer);
            for (let i = 0; i < buffer.length; i++) buffer[i] = Math.random() * 0xFFFFFFFF;
            ctx.putImageData(imageData, 0, 0);
        }
        setInterval(noise, 50);
        window.addEventListener('resize', () => {
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight;
        });
    </script>
</body>
</html>
