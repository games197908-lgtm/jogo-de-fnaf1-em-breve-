// [PLAYABLE_GAME_CODE]
const canvas = document.createElement('canvas');
const ctx = canvas.getContext('2d');
document.body.appendChild(canvas);
document.body.style.margin = "0";
document.body.style.overflow = "hidden";
document.body.style.backgroundColor = "#000";

const V_WIDTH = 1280, V_HEIGHT = 720;
let scale = 1, offsetX = 0, offsetY = 0;
function resize() {
    scale = Math.min(window.innerWidth / V_WIDTH, window.innerHeight / V_HEIGHT);
    canvas.width = window.innerWidth; canvas.height = window.innerHeight;
    offsetX = (window.innerWidth - V_WIDTH * scale) / 2;
    offsetY = (window.innerHeight - V_HEIGHT * scale) / 2;
}
window.addEventListener('resize', resize); resize();

const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
function sfx(f, t, d, v) {
    try { const o = audioCtx.createOscillator(); const g = audioCtx.createGain();
    o.type = t; o.frequency.setValueAtTime(f, audioCtx.currentTime);
    g.gain.setValueAtTime(v, audioCtx.currentTime); g.gain.exponentialRampToValueAtTime(0.0001, audioCtx.currentTime + d);
    o.connect(g); g.connect(audioCtx.destination); o.start(); o.stop(audioCtx.currentTime + d); } catch(e){}
}

let state = "MAIN_MENU", power = 100, hour = 0, gameTime = 0, camOpen = false, curCam = "1A";
let dL = false, dR = false, lL = false, lR = false, staticA = 0;
let jumpScareTarget = "freddy";
let levels = { freddy: 1, bonnie: 1, chica: 1, foxy: 1, golden: 1, shadow: 1 };

const anims = {
    freddy: { pos: "1A", timer: 0, color: "#4d2a0a" },
    bonnie: { pos: "1A", timer: 0, color: "#3b3b8c" },
    chica:  { pos: "1A", timer: 0, color: "#d4af37" },
    foxy:   { pos: "1C", timer: 0, color: "#962d2d", stage: 0 },
    golden: { active: false, timer: 0, color: "#e3c512" },
    shadow: { pos: "OFF", timer: 0, color: "#222" }
};

const CAM_POS = {
    "1A": {x: 1000, y: 350}, "1B": {x: 930, y: 410}, "1C": {x: 860, y: 470},
    "2A": {x: 1000, y: 550}, "2B": {x: 1000, y: 620}, "4A": {x: 1120, y: 550}, "4B": {x: 1120, y: 620}
};

function update() {
    if (state !== "PLAY") return;
    const dt = 1/60; gameTime += dt; staticA = Math.random() * 0.3;
    if (gameTime > 60) { gameTime = 0; hour++; if(hour >= 6) state = "WIN"; }
    power -= (0.15 + (dL?0.4:0) + (dR?0.4:0) + (camOpen?0.2:0) + (lL||lR?0.2:0)) * dt;
    if (power <= 0) { state = "JUMP"; jumpScareTarget = "freddy"; }

    // IA LOGIC
    anims.freddy.timer += dt * (levels.freddy * 0.4);
    if (anims.freddy.timer > 12) {
        const p = {"1A":"1B", "1B":"4A", "4A":"4B", "4B":"4B"};
        if (anims.freddy.pos === "4B" && camOpen && !dR) { state = "JUMP"; jumpScareTarget = "freddy"; }
        anims.freddy.pos = (anims.freddy.pos === "4B" && dR) ? "4A" : p[anims.freddy.pos];
        anims.freddy.timer = 0;
    }

    [anims.bonnie, anims.chica].forEach((a, i) => {
        a.timer += dt * (i === 0 ? levels.bonnie : levels.chica) * 0.5;
        if (a.timer > 10) {
            if (i === 0) {
                const p = {"1A":"1B", "1B":"2A", "2A":"2B", "2B":"1A"};
                if (a.pos === "2B" && !dL && !camOpen) { state = "JUMP"; jumpScareTarget = "bonnie"; }
                a.pos = p[a.pos];
            } else {
                const p = {"1A":"1B", "1B":"4A", "4A":"4B", "4B":"1A"};
                if (a.pos === "4B" && !dR && !camOpen) { state = "JUMP"; jumpScareTarget = "chica"; }
                a.pos = p[a.pos];
            }
            a.timer = 0;
        }
    });

    if (!camOpen) {
        anims.foxy.timer += dt * (levels.foxy * 0.3);
        if (anims.foxy.timer > 15) {
            anims.foxy.stage++;
            if (anims.foxy.stage > 3) {
                if (dL) { anims.foxy.stage = 0; power -= 10; sfx(50, 'square', 0.5, 0.5); }
                else { state = "JUMP"; jumpScareTarget = "foxy"; }
            }
            anims.foxy.timer = 0;
        }
    }

    // GOLDEN FREDDY SPAWN
    if (camOpen && Math.random() < (levels.golden * 0.0005)) { anims.golden.active = true; sfx(30, 'sawtooth', 2, 0.2); }
    if (anims.golden.active && !camOpen) { state = "JUMP"; jumpScareTarget = "golden"; }

    // SHADOW BONNIE SPAWN
    anims.shadow.timer += dt * (levels.shadow * 0.2);
    if (anims.shadow.timer > 20) {
        anims.shadow.pos = (anims.shadow.pos === "OFF") ? "2B" : "OFF";
        if (anims.shadow.pos === "2B" && !dL) { power -= 10; sfx(800, 'sine', 0.1, 0.2); }
        anims.shadow.timer = 0;
    }
}

function drawEntity(x, y, color, size, isFreddy, isFoxy) {
    ctx.fillStyle = color; ctx.beginPath(); ctx.arc(x, y, size, 0, 7); ctx.fill();
    ctx.fillStyle = isFreddy ? "red" : (isFoxy ? "yellow" : "white");
    ctx.beginPath(); ctx.arc(x-size/3, y-size/4, size/4, 0, 7); ctx.fill();
    ctx.beginPath(); ctx.arc(x+size/3, y-size/4, size/4, 0, 7); ctx.fill();
}

function draw() {
    ctx.fillStyle = "#000"; ctx.fillRect(0, 0, canvas.width, canvas.height);
    ctx.save(); ctx.translate(offsetX, offsetY); ctx.scale(scale, scale);

    if (state === "MAIN_MENU") {
        ctx.fillStyle = "white"; ctx.font = "80px Impact"; ctx.fillText("FIVE NIGHTS AT TURBO", 300, 200);
        ctx.fillStyle = "#222"; ctx.fillRect(440, 300, 400, 80); ctx.fillRect(440, 420, 400, 80);
        ctx.fillStyle = "white"; ctx.font = "30px Arial";
        ctx.fillText("NEW GAME (NIGHT 1)", 500, 350); ctx.fillText("CUSTOM NIGHT", 530, 470);
    } else if (state === "CUSTOM_MENU") {
        ctx.fillStyle = "white"; ctx.font = "40px Impact"; ctx.fillText("CUSTOM NIGHT CONFIG", 450, 80);
        let i = 0;
        for (let k in levels) {
            ctx.fillStyle = anims[k].color; ctx.fillRect(150 + i*170, 200, 120, 120);
            ctx.fillStyle = "white"; ctx.font = "20px Arial"; ctx.fillText(k.toUpperCase(), 150 + i*170, 350);
            ctx.font = "40px Impact"; ctx.fillText(levels[k], 190 + i*170, 400);
            i++;
        }
        ctx.fillStyle = "green"; ctx.fillRect(540, 550, 200, 80);
        ctx.fillStyle = "white"; ctx.font = "30px Arial"; ctx.fillText("START", 600, 600);
    } else if (state === "PLAY") {
        ctx.fillStyle = "#080808"; ctx.fillRect(200, 50, 880, 600);
        if (lL) {
            ctx.fillStyle = "rgba(255,255,255,0.1)"; ctx.fillRect(200, 50, 300, 600);
            if (anims.bonnie.pos === "2B") drawEntity(350, 350, anims.bonnie.color, 80, false, false);
            if (anims.shadow.pos === "2B") drawEntity(450, 380, "#111", 90, false, false);
        }
        if (lR) {
            ctx.fillStyle = "rgba(255,255,255,0.1)"; ctx.fillRect(780, 50, 300, 600);
            if (anims.chica.pos === "4B") drawEntity(930, 350, anims.chica.color, 80, false, false);
        }
        if (anims.golden.active) drawEntity(640, 400, anims.golden.color, 160, true, false);
        ctx.fillStyle = dL ? "#333" : "#000"; ctx.fillRect(200, 50, 100, 600);
        ctx.fillStyle = dR ? "#333" : "#000"; ctx.fillRect(980, 50, 100, 600);

        const bX = [30, 1130];
        [[dL, lL], [dR, lR]].forEach((b, i) => {
            ctx.fillStyle = b[0] ? "#ff0000" : "#444"; ctx.fillRect(bX[i], 150, 120, 120);
            ctx.fillStyle = b[1] ? "#ffff00" : "#444"; ctx.fillRect(bX[i], 300, 120, 120);
        });

        if (camOpen) {
            ctx.fillStyle = "#000"; ctx.fillRect(200, 50, 880, 600);
            if (curCam === "1C") {
                if (anims.foxy.stage === 1) drawEntity(640, 350, anims.foxy.color, 40, false, true);
                if (anims.foxy.stage === 2) drawEntity(640, 350, anims.foxy.color, 90, false, true);
            } else if (curCam === "1A") {
                if (anims.freddy.pos === "1A") drawEntity(640, 250, anims.freddy.color, 60, true, false);
                if (anims.bonnie.pos === "1A") drawEntity(580, 280, anims.bonnie.color, 50, false, false);
                if (anims.chica.pos === "1A") drawEntity(700, 280, anims.chica.color, 50, false, false);
            } else {
                if (anims.freddy.pos === curCam) drawEntity(640, 320, anims.freddy.color, 120, true, false);
                if (anims.bonnie.pos === curCam) drawEntity(450, 400, anims.bonnie.color, 90, false, false);
                if (anims.chica.pos === curCam) drawEntity(830, 400, anims.chica.color, 90, false, false);
            }
            ctx.fillStyle = `rgba(255,255,255,${staticA})`; ctx.fillRect(200, 50, 880, 600);
            for (let id in CAM_POS) {
                ctx.fillStyle = curCam === id ? "#0f0" : "#333"; ctx.fillRect(CAM_POS[id].x, CAM_POS[id].y, 70, 50);
                ctx.fillStyle="white"; ctx.font="16px Arial"; ctx.fillText(id, CAM_POS[id].x+20, CAM_POS[id].y+32);
            }
        }
        ctx.fillStyle = "white"; ctx.font = "30px Courier"; ctx.fillText(`${Math.ceil(power)}%`, 30, 50); ctx.fillText(`${hour} AM`, 1150, 50);
        ctx.fillStyle = "rgba(255,255,255,0.3)"; ctx.fillRect(350, 660, 580, 50);
    } else if (state === "JUMP") {
        ctx.fillStyle = "red"; ctx.font = "100px Impact"; ctx.fillText("GAME OVER", 420, 350);
        drawEntity(640, 450, anims[jumpScareTarget].color, 250, true, false);
    } else if (state === "WIN") {
        ctx.fillStyle = "white"; ctx.font = "120px Courier"; ctx.fillText("6 AM", 500, 350);
    }
    ctx.restore(); requestAnimationFrame(draw);
}

canvas.addEventListener('mousedown', e => {
    const mx = (e.clientX - offsetX) / scale, my = (e.clientY - offsetY) / scale;
    if (state === "MAIN_MENU") {
        if (mx > 440 && mx < 840) {
            if (my > 300 && my < 380) { levels = {freddy:2, bonnie:3, chica:2, foxy:1, golden:1, shadow:1}; state = "PLAY"; }
            if (my > 420 && my < 500) state = "CUSTOM_MENU";
            if (audioCtx.state === 'suspended') audioCtx.resume();
        }
    } else if (state === "CUSTOM_MENU") {
        let i = 0;
        for (let k in levels) {
            if (mx > 150 + i*170 && mx < 270 + i*170 && my > 200 && my < 320) levels[k] = (levels[k] + 1) % 21;
            i++;
        }
        if (mx > 540 && mx < 740 && my > 550 && my < 630) state = "PLAY";
    } else if (state === "PLAY") {
        if (mx > 350 && mx < 930 && my > 660) { camOpen = !camOpen; anims.golden.active = false; sfx(400, 'sine', 0.1, 0.1); }
        else if (!camOpen) {
            if (mx > 30 && mx < 150) { if (my > 150 && my < 270) dL = !dL; if (my > 300 && my < 420) lL = !lL; }
            if (mx > 1130 && mx < 1250) { if (my > 150 && my < 270) dR = !dR; if (my > 300 && my < 420) lR = !lR; }
        } else {
            for(let id in CAM_POS) if(mx > CAM_POS[id].x && mx < CAM_POS[id].x+70 && my > CAM_POS[id].y && my < CAM_POS[id].y+50) curCam = id;
        }
    } else location.reload();
});
setInterval(update, 1000/60); draw();
