<!DOCTYPE html>
<html lang="pt">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
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
            display: flex;
            justify-content: center;
            align-items: center;
        }
        .aviso {
            text-align: center;
            animation: glitch 1.5s infinite, piscar 1.2s infinite;
        }
        .fnaf {
            font-size: 7em;
            color: #ffff00;
            text-shadow: 0 0 25px #ffff00, 0 0 40px #ffff00;
            margin-bottom: 0.1em;
        }
        .texto {
            font-size: 4em;
            text-shadow: 0 0 20px #ff0000, 0 0 35px #ff0000;
        }
        @keyframes piscar {
            0%, 100% { opacity: 1; }
            50% { opacity: 0.3; }
        }
        @keyframes glitch {
            0% { 
                text-shadow: 0.05em 0 0 #ff0000, -0.05em -0.05em 0 #00ffff; 
                transform: translate(0);
            }
            15% { 
                text-shadow: -0.05em 0 0 #ff0000, 0.05em 0.05em 0 #00ffff; 
            }
            30% { 
                transform: translate(-0.02em, 0.02em);
            }
            50% { 
                text-shadow: 0.05em -0.05em 0 #ff0000, -0.05em 0 0 #00ffff; 
            }
            70% { 
                transform: translate(0.02em, -0.02em);
            }
            100% { 
                text-shadow: 0.05em 0 0 #ff0000, -0.05em -0.05em 0 #00ffff; 
                transform: translate(0);
            }
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
