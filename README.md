<!DOCTYPE html>
<html lang="pt">
<head>
    <meta charset="UTF-8">
    <title>FNAF 1 - Em Breve</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            background: #000;
            color: #ff0000;
            font-family: 'Courier New', Courier, monospace;
            overflow: hidden;
            height: 100vh;
        }
        .aviso {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            text-align: center;
            animation: glitch 2s infinite, piscar 1.5s infinite;
        }
        .fnaf {
            font-size: 6em;
            color: #ffff00;
            text-shadow: 0 0 20px #ffff00;
            margin-bottom: 0.1em;
        }
        .texto {
            font-size: 3.5em;
            text-shadow: 0 0 15px #ff0000;
        }
        @keyframes piscar {
            0%, 100% { opacity: 1; }
            50% { opacity: 0.4; }
        }
        @keyframes glitch {
            0% { text-shadow: 0.05em 0 0 #ff0000, -0.05em 0 0 #00ffff; }
            14% { text-shadow: 0.05em 0 0 #ff0000, -0.05em 0 0 #00ffff; }
            15% { text-shadow: -0.05em 0 0 #ff0000, 0.05em 0 0 #00ffff; }
            49% { text-shadow: -0.05em 0 0 #ff0000, 0.05em 0 0 #00ffff; }
            50% { text-shadow: 0.05em 0 0 #ff0000, -0.05em 0 0 #00ffff; }
            99% { text-shadow: 0.05em 0 0 #ff0000, -0.05em 0 0 #00ffff; }
            100% { text-shadow: -0.05em 0 0 #ff0000, 0.05em 0 0 #00ffff; }
        }
    </style>
</head>
<body>
    <div class="aviso">
        <div class="fnaf">FNAF 1</div>
        <div class="texto">EM BREVE SERÁ LANÇADO</div>
    </div>
</body>
</html>
