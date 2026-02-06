<!DOCTYPE html>
<html lang="hi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Subway Surfer - High Score Edition</title>
    <style>
        body { margin: 0; padding: 0; overflow: hidden; background: #222; font-family: 'Arial Black', sans-serif; }
        #gameContainer { position: relative; width: 100vw; height: 100vh; display: flex; justify-content: center; align-items: center; }
        canvas { background: #444; display: block; box-shadow: 0 0 50px rgba(0,0,0,0.8); }
        
        #ui { position: absolute; top: 10px; width: 100%; display: flex; justify-content: space-between; padding: 0 15px; box-sizing: border-box; pointer-events: none; z-index: 5; }
        .stat-box { background: rgba(0,0,0,0.7); padding: 8px 12px; border-radius: 12px; color: white; border: 2px solid #fff; font-size: 13px; }
        #highScoreBox { color: #00ff00; border-color: #00ff00; position: absolute; top: 60px; left: 15px; }
        #powerDisplay { color: #00f2ff; display: none; }

        #gameOverScreen { 
            position: absolute; width: 100%; height: 100%; 
            background: rgba(0,0,0,0.9); display: none; 
            flex-direction: column; justify-content: center; 
            align-items: center; color: white; z-index: 20; text-align: center;
        }
        .restart-btn { 
            padding: 15px 40px; font-size: 24px; background: #FFD700; 
            border: none; border-radius: 50px; cursor: pointer; 
            color: #222; font-weight: bold; margin-top: 20px; 
        }
        .result-text { font-size: 20px; margin: 5px 0; }
        .new-record { color: #00ff00; font-weight: bold; animation: blink 0.5s infinite; }
        @keyframes blink { 50% { opacity: 0; } }

        .controls { position: absolute; bottom: 30px; width: 100%; display: flex; justify-content: space-evenly; z-index: 10; }
        .btn { width: 75px; height: 75px; background: rgba(255, 255, 255, 0.2); border: 4px solid #fff; border-radius: 50%; display: flex; justify-content: center; align-items: center; font-size: 35px; cursor: pointer; user-select: none; }
        .jump-btn { background: rgba(255, 165, 0, 0.4); width: 85px; height: 85px; position: relative; top: -15px; }
    </style>
</head>
<body>

    <div id="gameContainer">
        <div id="gameOverScreen">
            <h1 id="overTitle" style="color: #ff3333; font-size: 40px;">CRASHED!</h1>
            <div id="newRecordMsg" class="new-record" style="display:none;">🎉 NEW HIGH SCORE! 🎉</div>
            <div class="result-text">⭐ SCORE: <span id="finalScore">0</span></div>
            <div class="result-text">🏆 BEST: <span id="finalBest">0</span></div>
            <div class="result-text" style="color: #FFD700;">🪙 COINS: <span id="finalCoins">0</span></div>
            <button class="restart-btn" onclick="resetGame()">START AGAIN</button>
        </div>

        <div id="ui">
            <div class="stat-box">⭐ <span id="scoreVal">0</span></div>
            <div id="highScoreBox" class="stat-box">🏆 <span id="highScoreVal">0</span></div>
            <div id="powerDisplay" class="stat-box">POWER ACTIVE</div>
            <div class="stat-box">🪙 <span id="coinVal">0</span></div>
        </div>

        <canvas id="game"></canvas>

        <div class="controls">
            <div class="btn" id="leftBtn">◀️</div>
            <div class="btn jump-btn" id="upBtn">⬆️</div>
            <div class="btn" id="rightBtn">▶️</div>
        </div>
    </div>

<script>
    const canvas = document.getElementById('game');
    const ctx = canvas.getContext('2d');
    const scoreVal = document.getElementById('scoreVal');
    const coinVal = document.getElementById('coinVal');
    const highScoreVal = document.getElementById('highScoreVal');
    const powerDisplay = document.getElementById('powerDisplay');
    const gameOverScreen = document.getElementById('gameOverScreen');

    canvas.width = window.innerWidth > 450 ? 450 : window.innerWidth;
    canvas.height = window.innerHeight;

    const laneWidth = canvas.width / 3;
    const laneX = [laneWidth / 2, laneWidth * 1.5, laneWidth * 2.5];

    let score, coinsCollected, speed, gameRunning, frame, currentLane, obstacles, coins, magnets, boards, jetpacks;
    let magnetT, boardT, jetpackT, isM, isB, isJ;
    
    // High Score logic
    let highScore = localStorage.getItem('subwayHighScore') || 0;
    highScoreVal.innerText = highScore;

    const player = { y: 0, baseY: canvas.height - 180, isJ: false, jH: 0, jS: 20, g: 1.1 };
    const police = { y: canvas.height - 60, lane: 1 };

    function resetGame() {
        score = 0; coinsCollected = 0; speed = 5; gameRunning = true; frame = 0; currentLane = 1;
        isM = isB = isJ = false; magnetT = boardT = jetpackT = 0;
        obstacles = []; coins = []; magnets = []; boards = []; jetpacks = [];
        player.y = player.baseY;
        gameOverScreen.style.display = 'none';
        document.getElementById('newRecordMsg').style.display = 'none';
        powerDisplay.style.display = 'none';
        spawnLoop(); gameLoop();
    }

    document.getElementById('leftBtn').onclick = () => { if (currentLane > 0) currentLane--; };
    document.getElementById('rightBtn').onclick = () => { if (currentLane < 2) currentLane++; };
    document.getElementById('upBtn').onclick = () => { if (!player.isJ && gameRunning && !isJ) { player.isJ = true; player.jH = player.jS; } };

    function spawnLoop() {
        if (!gameRunning) return;
        if (!isJ && frame % 100 === 0) obstacles.push({ x: laneX[Math.floor(Math.random() * 3)], y: -100, type: Math.random() > 0.4 ? '🚆' : '🚧' });
        if (frame % 45 === 0) coins.push({ x: laneX[Math.floor(Math.random() * 3)], y: isJ ? 100 : -100 });
        if (frame % 800 === 0) magnets.push({ x: laneX[Math.floor(Math.random() * 3)], y: -100 });
        if (frame % 1200 === 0) boards.push({ x: laneX[Math.floor(Math.random() * 3)], y: -100 });
        if (frame % 2000 === 0) jetpacks.push({ x: laneX[Math.floor(Math.random() * 3)], y: -100 });
        frame++;
        requestAnimationFrame(spawnLoop);
    }

    function update() {
        score += 0.05; speed += 0.0006;
        scoreVal.innerText = Math.floor(score); coinVal.innerText = coinsCollected;

        if (isM && --magnetT <= 0) isM = false;
        if (isB && --boardT <= 0) { isB = false; powerDisplay.style.display = 'none'; }
        if (isJ) {
            player.y = 100; powerDisplay.innerText = "🚀 FLYING"; powerDisplay.style.display = 'block';
            if (--jetpackT <= 0) { isJ = false; player.y = player.baseY; powerDisplay.style.display = 'none'; }
        }

        if (player.isJ && !isJ) {
            player.y -= player.jH; player.jH -= player.g;
            if (player.y >= player.baseY) { player.y = player.baseY; player.isJ = false; }
        }

        [jetpacks, magnets, boards].forEach((arr, idx) => {
            arr.forEach((p, i) => {
                p.y += speed;
                if (Math.abs(p.x - laneX[currentLane]) < 35 && Math.abs(p.y - player.y) < 60) {
                    if(idx===0) { isJ=true; jetpackT=600; }
                    else if(idx===1) { isM=true; magnetT=600; }
                    else { isB=true; boardT=600; powerDisplay.innerText="🛹 BOARD"; powerDisplay.style.display='block'; }
                    arr.splice(i, 1);
                }
            });
        });

        coins.forEach((c, i) => {
            c.y += speed;
            if (isM) { c.x += (laneX[currentLane] - c.x) * 0.2; c.y += (player.y - c.y) * 0.2; }
            if (Math.abs(c.x - laneX[currentLane]) < 40 && Math.abs(c.y - player.y) < 60) { coinsCollected++; coins.splice(i, 1); }
        });

        obstacles.forEach((obs, i) => {
            obs.y += speed;
            if (!isJ && Math.abs(obs.x - laneX[currentLane]) < 35 && Math.abs(obs.y - player.y) < 50) {
                if (isB) { obstacles.splice(i, 1); score += 50; }
                else if (!(obs.type === '🚧' && player.isJ)) endGame();
            }
            if (obs.y > canvas.height) obstacles.splice(i, 1);
        });
        police.lane = currentLane;
    }

    function endGame() { 
        gameRunning = false;
        let finalS = Math.floor(score);
        document.getElementById('finalScore').innerText = finalS;
        document.getElementById('finalCoins').innerText = coinsCollected;

        if (finalS > highScore) {
            highScore = finalS;
            localStorage.setItem('subwayHighScore', highScore);
            highScoreVal.innerText = highScore;
            document.getElementById('newRecordMsg').style.display = 'block';
        }
        document.getElementById('finalBest').innerText = highScore;
        gameOverScreen.style.display = 'flex'; 
    }

    function gameLoop() {
        if (!gameRunning) return;
        ctx.fillStyle = "#333"; ctx.fillRect(0, 0, canvas.width, canvas.height);
        ctx.strokeStyle = "#555"; ctx.setLineDash([20, 40]); ctx.lineDashOffset = -frame * speed;
        ctx.strokeRect(laneWidth, 0, laneWidth, canvas.height);

        let runO = (player.isJ || isJ) ? 0 : Math.sin(frame * 0.2) * 5;
        let pE = isJ ? "🚀" : (isB ? "🏄" : "🏃");

        jetpacks.forEach(j => { ctx.font = "40px Arial"; ctx.fillText("🚀", j.x - 20, j.y); });
        boards.forEach(b => { ctx.font = "40px Arial"; ctx.fillText("🛹", b.x - 20, b.y); });
        magnets.forEach(m => { ctx.font = "40px Arial"; ctx.fillText("🧲", m.x - 20, m.y); });
        coins.forEach(c => { ctx.font = "30px Arial"; ctx.fillText("🪙", c.x - 15, c.y); });
        obstacles.forEach(obs => { ctx.font = "70px Arial"; ctx.fillText(obs.type, obs.x - 35, obs.y); });

        ctx.font = isJ ? "100px Arial" : "80px Arial"; ctx.textAlign = "center";
        ctx.fillText(pE, laneX[currentLane], player.y + runO);
        
        if(!isJ) {
            ctx.font = "60px Arial"; ctx.fillText("👮", laneX[police.lane], police.y + 10 + runO);
            ctx.font = "40px Arial"; ctx.fillText("🐶", laneX[police.lane], police.y + 55 + runO);
        }

        update(); requestAnimationFrame(gameLoop);
    }
    resetGame();
</script>
</body>
</html>
