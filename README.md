<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>ゼビウス風 縦スクロールシューティング - XEVUS</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
            user-select: none;
            -webkit-user-select: none;
        }
        body {
            background-color: #0d0f12;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            overflow: hidden;
            font-family: 'Courier New', monospace;
            touch-action: none;
        }
        canvas {
            background-color: #1a2636;
            max-width: 100%;
            max-height: 100vh;
            aspect-ratio: 3 / 4;
            box-shadow: 0 0 30px rgba(0, 230, 255, 0.2);
            border: 2px solid #2a3a4e;
        }
    </style>
</head>
<body>

<canvas id="gameCanvas" width="480" height="640"></canvas>

<script>
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

// --- Web Audio API (BGM & SFX 統合エンジン) ---
const AudioCtx = window.AudioContext || window.webkitAudioContext;
let audioCtx = null;
let bgmInterval = null;

function initAudio() {
    if (!audioCtx) audioCtx = new AudioCtx();
    if (audioCtx.state === 'suspended') audioCtx.resume();
}

// BGM（8ビット風レトロループ調）
const bgmNotes = [
    110, 110, 220, 110, 147, 110, 165, 110,
    110, 110, 220, 110, 196, 175, 165, 147,
    130, 130, 261, 130, 175, 130, 196, 130,
    130, 130, 261, 130, 220, 196, 175, 165
];
let bgmStep = 0;

function startBGM() {
    if (bgmInterval) clearInterval(bgmInterval);
    bgmStep = 0;
    bgmInterval = setInterval(() => {
        if (!audioCtx || gameState !== 'PLAYING') return;
        const now = audioCtx.currentTime;
        
        // ベースライン
        const freq = bgmNotes[bgmStep % bgmNotes.length];
        const osc = audioCtx.createOscillator();
        const gain = audioCtx.createGain();
        osc.type = 'sawtooth';
        osc.frequency.setValueAtTime(freq, now);
        
        const filter = audioCtx.createBiquadFilter();
        filter.type = 'lowpass';
        filter.frequency.setValueAtTime(600, now);

        gain.gain.setValueAtTime(0.04, now);
        gain.gain.exponentialRampToValueAtTime(0.001, now + 0.12);

        osc.connect(filter);
        filter.connect(gain);
        gain.connect(audioCtx.destination);

        osc.start(now);
        osc.stop(now + 0.12);

        // 高音アルペジオ
        if (bgmStep % 4 === 2) {
            const mOsc = audioCtx.createOscillator();
            const mGain = audioCtx.createGain();
            mOsc.type = 'square';
            mOsc.frequency.setValueAtTime(freq * 3, now);
            mGain.gain.setValueAtTime(0.02, now);
            mGain.gain.exponentialRampToValueAtTime(0.001, now + 0.08);
            mOsc.connect(mGain);
            mGain.connect(audioCtx.destination);
            mOsc.start(now);
            mOsc.stop(now + 0.08);
        }

        bgmStep++;
    }, 130);
}

function stopBGM() {
    if (bgmInterval) {
        clearInterval(bgmInterval);
        bgmInterval = null;
    }
}

// 効果音 (SFX)
function playSFX(type) {
    if (!audioCtx) return;
    const now = audioCtx.currentTime;

    if (type === 'laser') { // 対空ショット
        const osc = audioCtx.createOscillator();
        const gain = audioCtx.createGain();
        osc.type = 'square';
        osc.frequency.setValueAtTime(880, now);
        osc.frequency.exponentialRampToValueAtTime(220, now + 0.06);
        gain.gain.setValueAtTime(0.05, now);
        gain.gain.exponentialRampToValueAtTime(0.001, now + 0.06);
        osc.connect(gain);
        gain.connect(audioCtx.destination);
        osc.start(now);
        osc.stop(now + 0.06);

    } else if (type === 'bomb') { // 対地ブラスター投下
        const osc = audioCtx.createOscillator();
        const gain = audioCtx.createGain();
        osc.type = 'sine';
        osc.frequency.setValueAtTime(400, now);
        osc.frequency.exponentialRampToValueAtTime(100, now + 0.15);
        gain.gain.setValueAtTime(0.08, now);
        gain.gain.exponentialRampToValueAtTime(0.001, now + 0.15);
        osc.connect(gain);
        gain.connect(audioCtx.destination);
        osc.start(now);
        osc.stop(now + 0.15);

    } else if (type === 'air_exp') { // 空中敵爆発
        const bufferSize = audioCtx.sampleRate * 0.15;
        const buffer = audioCtx.createBuffer(1, bufferSize, audioCtx.sampleRate);
        const data = buffer.getChannelData(0);
        for (let i = 0; i < bufferSize; i++) data[i] = Math.random() * 2 - 1;

        const noise = audioCtx.createBufferSource();
        noise.buffer = buffer;
        const gain = audioCtx.createGain();
        gain.gain.setValueAtTime(0.1, now);
        gain.gain.exponentialRampToValueAtTime(0.001, now + 0.15);
        noise.connect(gain);
        gain.connect(audioCtx.destination);
        noise.start(now);

    } else if (type === 'ground_exp') { // 地上爆発（重低音）
        const osc = audioCtx.createOscillator();
        const gain = audioCtx.createGain();
        osc.type = 'triangle';
        osc.frequency.setValueAtTime(150, now);
        osc.frequency.exponentialRampToValueAtTime(30, now + 0.3);
        gain.gain.setValueAtTime(0.2, now);
        gain.gain.exponentialRampToValueAtTime(0.001, now + 0.3);
        osc.connect(gain);
        gain.connect(audioCtx.destination);
        osc.start(now);
        osc.stop(now + 0.3);

    } else if (type === 'lockon') { // ロックオン音
        const osc = audioCtx.createOscillator();
        const gain = audioCtx.createGain();
        osc.type = 'sine';
        osc.frequency.setValueAtTime(1200, now);
        gain.gain.setValueAtTime(0.03, now);
        gain.gain.exponentialRampToValueAtTime(0.001, now + 0.04);
        osc.connect(gain);
        gain.connect(audioCtx.destination);
        osc.start(now);
        osc.stop(now + 0.04);
    }
}

// --- ゲーム状態 ---
let gameState = 'START';
let score = 0;
let highScore = 0;
let frameCount = 0;
let scrollY = 0;

// 自機
const player = {
    x: 240,
    y: 520,
    width: 32,
    height: 36,
    speed: 5,
    sightDistance: 110,
    lives: 3
};

let playerShots = [];
let playerBombs = [];
let airEnemies = [];
let groundEnemies = [];
let enemyBullets = [];
let particles = [];

let touchTargetX = null;
let touchTargetY = null;

// --- 入力処理 ---
function setupControls() {
    const updateTouchPos = (e) => {
        initAudio();
        const rect = canvas.getBoundingClientRect();
        const touch = e.touches ? e.touches[0] : e;
        const scaleX = canvas.width / rect.width;
        const scaleY = canvas.height / rect.height;
        touchTargetX = (touch.clientX - rect.left) * scaleX;
        touchTargetY = (touch.clientY - rect.top) * scaleY - 30;

        if (gameState === 'START' || gameState === 'GAMEOVER') {
            startGame();
        }
    };

    canvas.addEventListener('touchstart', (e) => { e.preventDefault(); updateTouchPos(e); }, { passive: false });
    canvas.addEventListener('touchmove', (e) => { e.preventDefault(); updateTouchPos(e); }, { passive: false });
    canvas.addEventListener('touchend', () => { touchTargetX = null; touchTargetY = null; });

    canvas.addEventListener('mousedown', (e) => { updateTouchPos(e); });
    canvas.addEventListener('mousemove', (e) => { if (e.buttons === 1) updateTouchPos(e); });

    window.addEventListener('keydown', (e) => {
        initAudio();
        if ((gameState === 'START' || gameState === 'GAMEOVER') && (e.code === 'Space' || e.code === 'KeyZ')) {
            startGame();
        }
    });
}

const keys = {};
window.addEventListener('keydown', e => keys[e.code] = true);
window.addEventListener('keyup', e => keys[e.code] = false);

function startGame() {
    score = 0;
    frameCount = 0;
    player.x = canvas.width / 2;
    player.y = 520;
    player.lives = 3;
    playerShots = [];
    playerBombs = [];
    airEnemies = [];
    groundEnemies = [];
    enemyBullets = [];
    particles = [];
    gameState = 'PLAYING';
    startBGM();
}

// --- ゲーム更新 ---
function update() {
    if (gameState !== 'PLAYING') return;

    frameCount++;
    scrollY += 2;

    // 自機移動
    if (touchTargetX !== null && touchTargetY !== null) {
        player.x += (touchTargetX - player.x) * 0.25;
        player.y += (touchTargetY - player.y) * 0.25;
    } else {
        if (keys['ArrowLeft'] || keys['KeyA']) player.x -= player.speed;
        if (keys['ArrowRight'] || keys['KeyD']) player.x += player.speed;
        if (keys['ArrowUp'] || keys['KeyW']) player.y -= player.speed;
        if (keys['ArrowDown'] || keys['KeyS']) player.y += player.speed;
    }

    player.x = Math.max(20, Math.min(canvas.width - 20, player.x));
    player.y = Math.max(60, Math.min(canvas.height - 40, player.y));

    const sightX = player.x;
    const sightY = player.y - player.sightDistance;

    // 自動射撃
    if (frameCount % 8 === 0) {
        playerShots.push({ x: player.x - 8, y: player.y - 15 });
        playerShots.push({ x: player.x + 8, y: player.y - 15 });
        playSFX('laser');
    }

    if (frameCount % 35 === 0) {
        playerBombs.push({
            startX: player.x, startY: player.y,
            targetX: sightX, targetY: sightY,
            x: player.x, y: player.y,
            progress: 0, radius: 12
        });
        playSFX('bomb');
    }

    // ショット＆ボム移動
    for (let i = playerShots.length - 1; i >= 0; i--) {
        const s = playerShots[i];
        s.y -= 12;
        if (s.y < -10) playerShots.splice(i, 1);
    }

    for (let i = playerBombs.length - 1; i >= 0; i--) {
        const b = playerBombs[i];
        b.progress += 0.06;
        b.x = b.startX + (b.targetX - b.startX) * b.progress;
        b.y = b.startY + (b.targetY - b.startY) * b.progress;
        b.radius = 12 * (1 - b.progress * 0.5);

        if (b.progress >= 1) {
            createExplosion(b.targetX, b.targetY, '#ffaa00', 12);
            checkGroundHit(b.targetX, b.targetY);
            playerBombs.splice(i, 1);
        }
    }

    // 敵生成
    if (frameCount % 50 === 0) {
        airEnemies.push({
            x: Math.random() * (canvas.width - 60) + 30,
            y: -20,
            type: Math.random() > 0.5 ? 'zigzag' : 'straight',
            angle: 0, radius: 14
        });
    }

    if (frameCount % 110 === 0) {
        groundEnemies.push({
            x: Math.floor(Math.random() * 8) * 50 + 60,
            y: -40,
            type: Math.random() > 0.3 ? 'base' : 'sol',
            width: 32, height: 32
        });
    }

    // 空中敵処理
    for (let i = airEnemies.length - 1; i >= 0; i--) {
        const e = airEnemies[i];
        if (e.type === 'straight') {
            e.y += 3.5;
        } else {
            e.y += 2.5;
            e.angle += 0.08;
            e.x += Math.sin(e.angle) * 3;
        }

        if (Math.random() < 0.015 && e.y > 50 && e.y < 350) {
            const angle = Math.atan2(player.y - e.y, player.x - e.x);
            enemyBullets.push({ x: e.x, y: e.y, vx: Math.cos(angle) * 3.5, vy: Math.sin(angle) * 3.5 });
        }

        for (let j = playerShots.length - 1; j >= 0; j--) {
            const s = playerShots[j];
            if (Math.hypot(s.x - e.x, s.y - e.y) < e.radius + 6) {
                createExplosion(e.x, e.y, '#00ffff', 15);
                playSFX('air_exp');
                score += 100;
                airEnemies.splice(i, 1);
                playerShots.splice(j, 1);
                break;
            }
        }

        if (Math.hypot(player.x - e.x, player.y - e.y) < e.radius + 12) playerHit();
        if (e.y > canvas.height + 30) airEnemies.splice(i, 1);
    }

    // 地上物＆照準ロックオン
    let hasLockOn = false;
    for (let i = groundEnemies.length - 1; i >= 0; i--) {
        const g = groundEnemies[i];
        g.y += 2;
        if (Math.abs(g.x - sightX) < 25 && Math.abs(g.y - sightY) < 25) hasLockOn = true;
        if (g.y > canvas.height + 50) groundEnemies.splice(i, 1);
    }
    if (hasLockOn && frameCount % 6 === 0) playSFX('lockon');

    // 敵弾処理
    for (let i = enemyBullets.length - 1; i >= 0; i--) {
        const b = enemyBullets[i];
        b.x += b.vx;
        b.y += b.vy;
        if (Math.hypot(player.x - b.x, player.y - b.y) < 10) {
            playerHit();
            enemyBullets.splice(i, 1);
        }
        if (b.x < 0 || b.x > canvas.width || b.y < 0 || b.y > canvas.height) enemyBullets.splice(i, 1);
    }

    // パーティクル
    for (let i = particles.length - 1; i >= 0; i--) {
        const p = particles[i];
        p.x += p.vx;
        p.y += p.vy;
        p.alpha -= 0.03;
        if (p.alpha <= 0) particles.splice(i, 1);
    }
}

function checkGroundHit(tx, ty) {
    for (let i = groundEnemies.length - 1; i >= 0; i--) {
        const g = groundEnemies[i];
        if (Math.hypot(g.x - tx, g.y - ty) < 28) {
            createExplosion(g.x, g.y, '#ff4400', 25);
            playSFX('ground_exp');
            score += (g.type === 'sol') ? 500 : 300;
            groundEnemies.splice(i, 1);
        }
    }
}

function playerHit() {
    createExplosion(player.x, player.y, '#ffffff', 30);
    playSFX('ground_exp');
    player.lives--;
    if (player.lives <= 0) {
        gameState = 'GAMEOVER';
        stopBGM();
        if (score > highScore) highScore = score;
    } else {
        player.x = canvas.width / 2;
        player.y = 520;
    }
}

function createExplosion(x, y, color, count) {
    for (let i = 0; i < count; i++) {
        const angle = Math.random() * Math.PI * 2;
        const speed = Math.random() * 4 + 1;
        particles.push({
            x: x, y: y,
            vx: Math.cos(angle) * speed,
            vy: Math.sin(angle) * speed,
            color: color,
            radius: Math.random() * 3 + 2,
            alpha: 1
        });
    }
}

// --- 描画 ---
function draw() {
    // 地上背景
    ctx.fillStyle = '#1e381e';
    ctx.fillRect(0, 0, canvas.width, canvas.height);

    ctx.strokeStyle = '#274727';
    ctx.lineWidth = 1;
    const gridOffset = scrollY % 40;
    for (let y = gridOffset; y < canvas.height; y += 40) {
        ctx.beginPath();
        ctx.moveTo(0, y);
        ctx.lineTo(canvas.width, y);
        ctx.stroke();
    }

    // 道路
    ctx.fillStyle = '#2c353f';
    ctx.fillRect(180, 0, 120, canvas.height);

    // 地上物
    groundEnemies.forEach(g => {
        ctx.save();
        ctx.translate(g.x, g.y);
        if (g.type === 'sol') {
            ctx.fillStyle = '#e6b800';
            ctx.beginPath();
            ctx.moveTo(0, -16);
            ctx.lineTo(16, 16);
            ctx.lineTo(-16, 16);
            ctx.closePath();
            ctx.fill();
            ctx.fillStyle = '#ffeb80';
            ctx.fillRect(-4, -4, 8, 8);
        } else {
            ctx.fillStyle = '#5a6b7c';
            ctx.fillRect(-16, -16, 32, 32);
            ctx.fillStyle = '#8b9bb0';
            ctx.fillRect(-10, -10, 20, 20);
            ctx.fillStyle = '#ff3333';
            ctx.beginPath();
            ctx.arc(0, 0, 5, 0, Math.PI * 2);
            ctx.fill();
        }
        ctx.restore();
    });

    // 照準
    if (gameState === 'PLAYING') {
        const sightX = player.x;
        const sightY = player.y - player.sightDistance;

        ctx.strokeStyle = '#00ffff';
        ctx.lineWidth = 1.5;
        ctx.beginPath();
        ctx.moveTo(sightX, sightY - 12);
        ctx.lineTo(sightX + 12, sightY);
        ctx.lineTo(sightX, sightY + 12);
        ctx.lineTo(sightX - 12, sightY);
        ctx.closePath();
        ctx.stroke();

        ctx.fillStyle = '#ff0055';
        ctx.fillRect(sightX - 1.5, sightY - 1.5, 3, 3);
    }

    // ボム
    playerBombs.forEach(b => {
        ctx.fillStyle = '#ffff00';
        ctx.beginPath();
        ctx.arc(b.x, b.y, b.radius, 0, Math.PI * 2);
        ctx.fill();
    });

    // ショット
    ctx.fillStyle = '#00ffff';
    playerShots.forEach(s => {
        ctx.fillRect(s.x - 2, s.y - 8, 4, 12);
    });

    // 空中敵
    airEnemies.forEach(e => {
        ctx.save();
        ctx.translate(e.x, e.y);
        ctx.fillStyle = '#e63946';
        ctx.beginPath();
        ctx.arc(0, 0, e.radius, 0, Math.PI * 2);
        ctx.fill();
        ctx.fillStyle = '#ffffff';
        ctx.fillRect(-4, -4, 8, 8);
        ctx.restore();
    });

    // 敵弾
    ctx.fillStyle = '#ff0055';
    enemyBullets.forEach(b => {
        ctx.beginPath();
        ctx.arc(b.x, b.y, 4, 0, Math.PI * 2);
        ctx.fill();
    });

    // 爆発
    particles.forEach(p => {
        ctx.fillStyle = p.color;
        ctx.globalAlpha = p.alpha;
        ctx.beginPath();
        ctx.arc(p.x, p.y, p.radius, 0, Math.PI * 2);
        ctx.fill();
        ctx.globalAlpha = 1;
    });

    // 自機
    if (gameState === 'PLAYING') {
        ctx.save();
        ctx.translate(player.x, player.y);
        ctx.fillStyle = '#d1d5db';
        ctx.beginPath();
        ctx.moveTo(0, -18);
        ctx.lineTo(14, 12);
        ctx.lineTo(0, 6);
        ctx.lineTo(-14, 12);
        ctx.closePath();
        ctx.fill();
        ctx.fillStyle = '#2563eb';
        ctx.fillRect(-10, 2, 20, 4);
        ctx.fillStyle = '#00f0ff';
        ctx.beginPath();
        ctx.arc(0, -4, 3, 0, Math.PI * 2);
        ctx.fill();
        ctx.restore();
    }

    // UI
    ctx.fillStyle = '#ffffff';
    ctx.font = 'bold 16px "Courier New", monospace';
    ctx.fillText(`SCORE: ${score}`, 20, 30);
    ctx.fillText(`HIGH: ${highScore}`, 180, 30);
    ctx.fillText(`LIVES: ${'★'.repeat(Math.max(0, player.lives))}`, 340, 30);

    if (gameState === 'START') {
        drawOverlay('XEVUS - ゼビウス風シューティング', '指で画面をスライドして操作！\n自動連射＆自動爆撃\n\n【タップでスタート】');
    } else if (gameState === 'GAMEOVER') {
        drawOverlay('GAME OVER', `SCORE: ${score}\n\n【タップでリトライ】`);
    }
}

function drawOverlay(title, subtitle) {
    ctx.fillStyle = 'rgba(10, 15, 25, 0.75)';
    ctx.fillRect(0, 0, canvas.width, canvas.height);
    
    ctx.fillStyle = '#00ffff';
    ctx.textAlign = 'center';
    ctx.font = 'bold 22px sans-serif';
    ctx.fillText(title, canvas.width / 2, canvas.height / 2 - 50);
    
    ctx.fillStyle = '#ffffff';
    ctx.font = '16px sans-serif';
    const lines = subtitle.split('\n');
    lines.forEach((line, index) => {
        ctx.fillText(line, canvas.width / 2, canvas.height / 2 + 10 + (index * 26));
    });
    ctx.textAlign = 'left';
}

function gameLoop() {
    update();
    draw();
    requestAnimationFrame(gameLoop);
}

setupControls();
gameLoop();
</script>

</body>
</html>