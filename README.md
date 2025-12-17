<!DOCTYPE html>
<html>
<head>
    <title>Aviso FNAF 1</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            background: #000;
            color: #ff0000;
            font-family: 'Courier New', monospace;
            overflow: hidden;
        }
        .aviso {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            font-size: 4em;
            text-align: center;
            text-shadow: 0 0 20px #ff0000;
            animation: piscar 1s infinite;
        }
        @keyframes piscar {
            0%, 50% { opacity: 1; }
            51%, 100% { opacity: 0.3; }
        }
        .fnaf {
            font-size: 6em;
            color: #ffff00;
            margin-bottom: 0.2em;
        }
    </style>
</head>
<body>
    <div class="aviso">
        <div class="fnaf">FNAF 1</div>
        EM BREVE SERÁ LANÇADO
    </div>
</body>
</html>
