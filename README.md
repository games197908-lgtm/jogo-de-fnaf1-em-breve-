<!-- ===========================================================
     ARQUIVO COMPLETO
     SITE + BACKEND PARA CRIAR SERVIDOR MINECRAFT PE
     =========================================================== -->

<!-- ======================= FRONT-END ========================= -->
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Criar Servidor Minecraft PE</title>
    <style>
        body {
            background:#0d0d0d;
            color:white;
            font-family:Arial;
            text-align:center;
        }
        .box {
            margin-top:80px;
            background:#111;
            padding:40px;
            width:400px;
            margin-left:auto;
            margin-right:auto;
            border-radius:12px;
            box-shadow:0 0 20px #00ffea;
        }
        input, select {
            width:90%;
            padding:12px;
            margin-top:12px;
            border-radius:6px;
            border:none;
        }
        button {
            margin-top:20px;
            padding:12px 20px;
            background:#00ffee;
            color:black;
            font-size:18px;
            border:none;
            border-radius:6px;
            cursor:pointer;
        }
    </style>
</head>
<body>

<div class="box">
    <h1>Minecraft PE Server Creator</h1>
    <h3>Crie seu servidor grátis • 24 horas • PocketMine ou Bedrock</h3>

    <input id="nome" type="text" placeholder="Nome do Servidor">

    <select id="tipo">
        <option>PocketMine-MP</option>
        <option>Bedrock Dedicated Server</option>
    </select>

    <select id="versao">
        <option>1.21</option>
        <option>1.20.80</option>
        <option>1.20</option>
    </select>

    <button onclick="criar()">Criar Servidor</button>

    <p id="status"></p>
</div>

<script>
async function criar() {
    const nome = document.getElementById('nome').value;
    const tipo = document.getElementById('tipo').value;
    const versao = document.getElementById('versao').value;

    const req = await fetch("/criar", {
        method:"POST",
        headers:{"Content-Type":"application/json"},
        body: JSON.stringify({ nome, tipo, versao })
    });

    const res = await req.json();
    document.getElementById("status").innerText = res.msg;
}
</script>

</body>
</html>

<!-- ======================= BACK-END ========================= -->

<!--  
 Salve isso como "server.js"
 Execute:
   npm init -y
   npm install express
   node server.js
 O site acima precisa estar na mesma pasta.
-->

<script type="text/plain">
const express = require("express");
const { exec } = require("child_process");
const fs = require("fs");
const app = express();

app.use(express.json());
app.use(express.static(".")); // Serve o index.html automaticamente

app.post("/criar", async (req, res) => {
    const { nome, tipo, versao } = req.body;

    const pasta = `/home/servers/${nome}`;
    if (!fs.existsSync(pasta)) fs.mkdirSync(pasta, { recursive: true });

    let comando = "";

    // ================= PocketMine =================
    if (tipo === "PocketMine-MP") {
        comando = `
            cd ${pasta} &&
            wget https://get.pmmp.io/${versao} -O pocketmine.phar &&
            echo "php pocketmine.phar" > start.sh &&
            chmod +x start.sh
        `;
    }

    // ================= Bedrock Official =================
    else {
        comando = `
            cd ${pasta} &&
            wget https://minecraft.net/bedrock-server-${versao}.zip -O bedrock.zip &&
            unzip bedrock.zip &&
            chmod +x bedrock_server
        `;
    }

    exec(comando, (erro) => {
        if (erro) return res.json({ msg: "Erro ao criar servidor." });
        return res.json({ msg: "Servidor criado com sucesso! Inicie pelo start.sh" });
    });
});

app.listen(3000, () => console.log("Servidor do painel rodando na porta 3000"));
</script>
