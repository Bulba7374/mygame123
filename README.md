<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="mobile-web-app-capable" content="yes">
    <title>Ultimate Shadow Runner</title>
    
    <!-- PWA Manifest: Делает игру приложением -->
    <link rel="manifest" href='data:application/json,{"name":"Shadow Runner","short_name":"Runner","start_url":".","display":"fullscreen","background_color":"#0f172a","theme_color":"#facc15","orientation":"landscape","icons":[{"src":"data:image/svg+xml,%3Csvg xmlns=%22http://www.w3.org/2000/svg%22 viewBox=%220 0 100 100%22%3E%3Crect width=%22100%22 height=%22100%22 fill=%22%230f172a%22/%3E%3Ccircle cx=%2250%22 cy=%2250%22 r=%2230%22 fill=%22%23facc15%22/%3E%3C/svg%3E","sizes":"192x192","type":"image/svg+xml"}]}' />
    
    <style>
        /* Сброс стилей для идеального полноэкранного режима */
        body { margin: 0; padding: 0; background: #0f172a; font-family: 'Segoe UI', sans-serif; display: flex; justify-content: center; align-items: center; height: 100vh; width: 100vw; overflow: hidden; touch-action: none; }
        
        #menu {
            position: absolute; z-index: 100; background: rgba(15, 23, 42, 0.95); padding: 20px; border-radius: 25px;
            text-align: center; color: white; width: 90%; max-width: 500px; border: 2px solid #facc15;
            box-shadow: 0 20px 40px rgba(0,0,0,0.9); backdrop-filter: blur(10px);
        }
        #menu h1 { color: #facc15; font-size: 26px; margin: 0 0 10px 0; }
        
        /* Сетка выбора костюмов */
        .skin-grid { display: grid; grid-template-columns: repeat(5, 1fr); gap: 8px; margin: 15px 0; }
        .skin-btn {
            background: #1e293b; color: white; padding: 8px; border-radius: 12px; cursor: pointer;
            border: 2px solid transparent; font-size: 12px; text-align: center; transition: 0.2s;
        }
        .skin-btn.equipped { border-color: #facc15; background: #3b82f6; transform: scale(1.05); box-shadow: 0 0 15px rgba(59,130,246,0.5); }
        
        .btn-primary {
            background: #10b981; color: white; font-size: 20px; font-weight: bold; padding: 14px 30px;
            border: none; border-radius: 25px; cursor: pointer; width: 100%; box-shadow: 0 4px 0 #059669;
        }
        .btn-secondary { background: #8b5cf6; color: white; padding: 8px 15px; border: none; border-radius: 15px; cursor: pointer; font-size: 12px; }

        /* Игровой контейнер (занимает ВСЁ пространство) */
        #gameWrapper { display: none; position: absolute; top: 0; left: 0; width: 100vw; height: 100vh; background: #000; }
        canvas { display: block; width: 100%; height: 100%; touch-action: none; }

        #hud {
            position: absolute; top: 2%; left: 0; width: 100%; display: flex; justify-content: space-between;
            padding: 0 3%; box-sizing: border-box; pointer-events: none; z-index: 10;
        }
        #hud span {
            background: rgba(0,0,0,0.7); color: white; padding: 4px 12px; border-radius: 20px; 
            font-weight: bold; font-size: 14px; backdrop-filter: blur(4px);
        }
        .gold { color: #facc15; } .red { color: #ef4444; }

        /* Сенсорные кнопки (ВСЕГДА В НИЗУ, НЕ МЕШАЮТ) */
        #mobile-controls {
            position: absolute; bottom: 3%; left: 0; width: 100%; display: flex; justify-content: space-between;
            padding: 0 5%; box-sizing: border-box; z-index: 20; pointer-events: none;
        }
        .ctrl-btn {
            pointer-events: auto; background: rgba(255,255,255,0.15); color: white; border: 2px solid rgba(255,255,255,0.3);
            border-radius: 50%; width: 18vw; max-width: 75px; height: 18vw; max-height: 75px;
            font-size: 18px; display: flex; align-items: center; justify-content: center; 
            backdrop-filter: blur(8px); font-weight: bold;
        }
        .ctrl-btn:active { transform: scale(0.9); background: rgba(255,255,255,0.3); }
        .ctrl-jump { border-color: #facc15; background: rgba(250, 204, 21, 0.25); }
        @media (pointer: fine) { #mobile-controls { display: none !important; } }
    </style>
</head>
<body>

    <!-- МЕНЮ -->
    <div id="menu">
        <h1>⚡ SHADOW RUNNER ⚡</h1>
        <div style="font-size:14px; margin-bottom:10px;">
            💰 Монет: <span id="menuCoins" class="gold">0</span>
            <button class="btn-secondary" onclick="toggleFullscreen()">⛑️ Экран</button>
        </div>
        
        <div class="skin-grid" id="skinGrid"></div>
        
        <button class="btn-primary" id="startBtn">ИГРАТЬ!</button>
    </div>

    <!-- ИГРА -->
    <div id="gameWrapper">
        <canvas id="gc" width="1200" height="700"></canvas>
        <div id="hud">
            <span>🌍 <span id="lvlNum" class="gold">1</span></span>
            <span>❤️ <span id="hpNum" class="red">5</span></span>
            <span>💰 <span id="coinNum" class="gold">0</span></span>
            <span id="statusText">Беги!</span>
        </div>
        
        <div id="mobile-controls">
            <div style="display:flex; gap:15px; pointer-events:none;">
                <div class="ctrl-btn" id="touch-left">◀</div>
                <div class="ctrl-btn" id="touch-right">▶</div>
            </div>
            <div class="ctrl-btn ctrl-jump" id="touch-jump">⬆</div>
        </div>
    </div>

    <script>
        // --- ПОЛНОЭКРАННЫЙ РЕЖИМ ---
        function toggleFullscreen() {
            const el = document.documentElement;
            if (!document.fullscreenElement) {
                if (el.requestFullscreen) el.requestFullscreen();
                else if (el.webkitRequestFullscreen) el.webkitRequestFullscreen();
            } else {
                if (document.exitFullscreen) document.exitFullscreen();
                else if (document.webkitExitFullscreen) document.webkitExitFullscreen();
            }
        }

        // --- ДАННЫЕ ИГРЫ ---
        const canvas = document.getElementById('gc');
        const ctx = canvas.getContext('2d');
        canvas.width = 1200; canvas.height = 700;
        
        const lvlNum = document.getElementById('lvlNum');
        const hpNum = document.getElementById('hpNum');
        const coinNum = document.getElementById('coinNum');
        const statusText = document.getElementById('statusText');
        const skinGrid = document.getElementById('skinGrid');
        const menuCoins = document.getElementById('menuCoins');

        // --- 5 КОСТЮМОВ С ТЕКСТУРОЙ ---
        const COSTUMES = [
            { id: 0, type: 'Рыцарь', color: '#bdc3c7', desc: '⚔️', ability: 'shield' },
            { id: 1, type: 'Дракон', color: '#e74c3c', desc: '🐉', ability: 'superJump' },
            { id: 2, type: 'Маг', color: '#8e44ad', desc: '🧙', ability: 'none' },
            { id: 3, type: 'Зомби', color: '#27ae60', desc: '🧟', ability: 'none' },
            { id: 4, type: 'Тень', color: '#0f172a', desc: '🌑', ability: 'none' }
        ];

        let playerData = { coins: 0, equippedSkin: 0, maxHealth: 5, hasSword: false, hasShield: false, hasArmor: false, hasBow: false };
        let currentCostume = COSTUMES[0];
        let isShieldActive = false;

        // --- МЕНЮ ---
        function renderSkins() {
            skinGrid.innerHTML = '';
            COSTUMES.forEach((skin, index) => {
                const btn = document.createElement('div');
                btn.className = 'skin-btn';
                if (playerData.equippedSkin === index) btn.classList.add('equipped');
                btn.innerHTML = `<div style="width:30px;height:30px;margin:0 auto 5px;background:${skin.color};border-radius:8px;border:2px solid #555;display:flex;align-items:center;justify-content:center;">${skin.desc}</div>${skin.type}`;
                btn.onclick = () => { playerData.equippedSkin = index; currentCostume = skin; renderSkins(); };
                skinGrid.appendChild(btn);
            });
            menuCoins.innerText = playerData.coins;
        }

        // --- УПРАВЛЕНИЕ (ПК и ТЕЛЕФОН) ---
        const keys = { left: false, right: false };
        let jumpsLeft = 2;

        // Настройка сенсорных кнопок
        const setupTouch = (id, key) => {
            const el = document.getElementById(id);
            if(!el) return;
            el.addEventListener('touchstart', e => { e.preventDefault(); keys[key] = true; });
            el.addEventListener('touchend', e => { e.preventDefault(); keys[key] = false; });
            el.addEventListener('mousedown', e => { e.preventDefault(); keys[key] = true; });
            el.addEventListener('mouseup', e => { e.preventDefault(); keys[key] = false; });
        };
        setupTouch('touch-left', 'left');
        setupTouch('touch-right', 'right');
        document.getElementById('touch-jump').addEventListener('touchstart', e => { e.preventDefault(); jumpAction(); });
        document.getElementById('touch-jump').addEventListener('mousedown', e => { e.preventDefault(); jumpAction(); });

        // ПК управление
        canvas.addEventListener('contextmenu', e => e.preventDefault());
        canvas.addEventListener('mousedown', e => { if(e.button === 0) keys.left = true; if(e.button === 2) keys.right = true; });
        canvas.addEventListener('mouseup', e => { if(e.button === 0) keys.left = false; if(e.button === 2) keys.right = false; });
        document.addEventListener('keydown', e => { if(e.key === ' ') { e.preventDefault(); jumpAction(); } });

        function jumpAction() {
            if(jumpsLeft > 0 && !player.isDead) {
                let force = -14;
                if(currentCostume.ability === 'superJump' && jumpsLeft === 1) force = -20;
                player.vy = force; jumpsLeft--;
            }
        }

        // --- ИГРОК И БОТЫ ---
        const player = { x: 50, y: 550, w: 30, h: 42, vx: 0, vy: 0, speedMod: 1, slowTimer: 0, health: 5, isDead: false, respawnTimer: 0, isGrounded: true };

        // --- ГЕНЕРАЦИЯ УРОВНЯ ---
        let currentLevel = 0;
        let levelData = { spikes: [], trampolines: [], gears: [], coins: [], bots: [], finishX: 1150 };

        function generateLevel(index) {
            player.x = 50; player.y = 550; player.vx = 0; player.vy = 0; 
            player.health = playerData.maxHealth; player.isDead = false; jumpsLeft = 2; isShieldActive = false;
            updateHUD(); statusText.innerText = "Беги!";
            levelData = { spikes: [], trampolines: [], gears: [], coins: [], bots: [], finishX: 1150 };

            // Много шипов (на расстоянии)
            let prevX = 0;
            for(let i=0; i<15; i++) {
                let x = 130 + 40 + Math.random() * 980;
                if(Math.abs(x - prevX) > 60) {
                    levelData.spikes.push({ x: x, y: 665, w: 20, h: 20 });
                    prevX = x;
                }
            }
            // Батуты
            for(let i=0; i<6; i++) {
                let x = 80 + Math.random() * 1000;
                let isSafe = true;
                for(let s of levelData.spikes) if(Math.abs(s.x - x) < 40) isSafe = false;
                if(isSafe) levelData.trampolines.push({ x: x, y: 640, r: 15 });
            }
            // Шестеренки
            for(let i=0; i<4; i++) {
                let x = 150 + Math.random() * 900;
                let isSafe = true;
                for(let s of levelData.spikes) if(Math.abs(s.x - x) < 40) isSafe = false;
                for(let t of levelData.trampolines) if(Math.abs(t.x - x) < 40) isSafe = false;
                if(isSafe) levelData.gears.push({ x: x, y: 640, r: 18, angle: 0 });
            }
            // Много ботов (можно убить прыжком)
            for(let i=0; i<9; i++) {
                let x = 160 + Math.random() * 900;
                let isSafe = true;
                for(let s of levelData.spikes) if(Math.abs(s.x - x) < 40) isSafe = false;
                for(let t of levelData.trampolines) if(Math.abs(t.x - x) < 40) isSafe = false;
                for(let g of levelData.gears) if(Math.abs(g.x - x) < 40) isSafe = false;
                if(isSafe) levelData.bots.push({ x: x, y: 620, w: 30, h: 30, alive: true, vx: 1.5 + Math.random() * 3 });
            }
            // Монетки
            for(let i=0; i<25; i++) {
                let x = 40 + Math.random() * 1080;
                let isSafe = true;
                for(let s of levelData.spikes) if(Math.abs(s.x - x) < 30) isSafe = false;
                for(let t of levelData.trampolines) if(Math.abs(t.x - x) < 30) isSafe = false;
                for(let g of levelData.gears) if(Math.abs(g.x - x) < 30) isSafe = false;
                for(let b of levelData.bots) if(Math.abs(b.x - x) < 30) isSafe = false;
                if(isSafe) levelData.coins.push({ x: x, y: 645, r: 8 });
            }
            lvlNum.innerText = index + 1;
        }

        // --- ФИЗИКА ---
        const GRAVITY = 0.65; const MOVE_SPEED = 7; const TRAMPOLINE_FORCE = -16;

        function update() {
            if(player.isDead) { player.respawnTimer--; if(player.respawnTimer<=0) generateLevel(currentLevel); return; }
            if(player.slowTimer > 0) { player.slowTimer--; if(player.slowTimer===0) player.speedMod = 1; }

            if(keys.left) player.vx = -MOVE_SPEED * player.speedMod;
            else if(keys.right) player.vx = MOVE_SPEED * player.speedMod;
            else player.vx *= 0.85;

            player.vy += GRAVITY; if(player.vy > 20) player.vy = 20;
            player.x += player.vx; if(player.x < 0) player.x = 0;

            if(player.vy > 0 && player.y + player.h > 670) {
                player.y = 670 - player.h; player.vy = 0; player.isGrounded = true; jumpsLeft = 2;
            }
            player.y += player.vy;

            // Шипы (Смерть)
            for(let spike of levelData.spikes) {
                if(player.x + player.w > spike.x && player.x < spike.x+20 && player.y+player.h > spike.y-20 && player.y < spike.y) {
                    if(currentCostume.ability === 'shield' && !isShieldActive) {
                        isShieldActive = true; continue;
                    } else { player.isDead = true; player.respawnTimer=40; return; }
                }
            }
            // Батуты
            for(let tramp of levelData.trampolines) {
                if(Math.sqrt((player.x+15 - tramp.x)**2 + (player.y+20 - tramp.y)**2) < tramp.r+15) player.vy = TRAMPOLINE_FORCE;
            }
            // Шестеренки
            for(let i=levelData.gears.length-1; i>=0; i--) {
                let g = levelData.gears[i]; g.angle += 0.15;
                if(Math.sqrt((player.x+15 - g.x)**2 + (player.y+20 - g.y)**2) < g.r+15) {
                    player.health -= 1; player.speedMod = 0.5; player.slowTimer = 120;
                    if(player.health <= 0) { player.isDead=true; return; }
                }
            }
            // Боты (Убиваются прыжком)
            for(let i=levelData.bots.length-1; i>=0; i--) {
                let bot = levelData.bots[i];
                if(!bot.alive) continue;
                bot.x += bot.vx;
                if(bot.x < 20 || bot.x > 1180) bot.vx *= -1;

                if(Math.abs((player.x+15) - (bot.x+15)) < 30 && Math.abs((player.y+20) - (bot.y+15)) < 30) {
                    if(player.vy > 0 && player.y + player.h < bot.y + 10) {
                        bot.alive = false; playerData.coins += 5; player.vy = -10; updateHUD();
                    } else { player.isDead = true; player.respawnTimer=40; return; }
                }
            }
            // Монетки
            for(let i=levelData.coins.length-1; i>=0; i--) {
                let c = levelData.coins[i];
                if(Math.sqrt((player.x+15 - c.x)**2 + (player.y+20 - c.y)**2) < c.r+15) {
                    playerData.coins += 1; levelData.coins.splice(i,1); updateHUD();
                }
            }
            if(player.x > levelData.finishX - 20) {
                currentLevel++; if(currentLevel>=1000) { alert("1000 уровней пройдено!"); currentLevel=0; } generateLevel(currentLevel);
            }
        }

        function updateHUD() { hpNum.innerText = Math.floor(player.health); coinNum.innerText = playerData.coins; menuCoins.innerText = playerData.coins; }

        // --- РИСОВАНИЕ ГЕРОЯ И БОТОВ ---
        function drawHero(cx, cy, costume) {
            const c = costume.color;
            ctx.strokeStyle = "#0f172a"; ctx.lineWidth = 2;

            if(costume.type === 'Дракон') {
                ctx.fillStyle = c; 
                ctx.beginPath(); ctx.arc(cx, cy+15, 22, 0, Math.PI*2); ctx.fill(); ctx.stroke();
                ctx.fillRect(cx-30, cy+5, 14, 24); ctx.fillRect(cx+16, cy+5, 14, 24);
                ctx.fillRect(cx-14, cy-12, 4, 8); ctx.fillRect(cx+10, cy-12, 4, 8);
                ctx.beginPath(); ctx.arc(cx, cy-2, 16, 0, Math.PI*2); ctx.fill(); ctx.stroke();
                ctx.fillStyle = "white"; ctx.beginPath(); ctx.arc(cx-6, cy-4, 6, 0, Math.PI*2); ctx.fill(); ctx.stroke();
                ctx.beginPath(); ctx.arc(cx+6, cy-4, 6, 0, Math.PI*2); ctx.fill(); ctx.stroke();
                ctx.fillStyle = "#000"; ctx.beginPath(); ctx.arc(cx-5, cy-3, 3, 0, Math.PI*2); ctx.fill();
                ctx.beginPath(); ctx.arc(cx+7, cy-3, 3, 0, Math.PI*2); ctx.fill();
            } else if(costume.type === 'Тень') {
                ctx.shadowColor = "#8b5cf6"; ctx.shadowBlur = 25;
                ctx.fillStyle = "#000"; ctx.strokeStyle = "#000";
                ctx.fillRect(cx-8, cy+10, 16, 22);
                ctx.beginPath(); ctx.arc(cx, cy+4, 14, 0, Math.PI*2); ctx.fill(); ctx.stroke();
                ctx.shadowBlur = 0;
                ctx.fillStyle = "#8b5cf6"; ctx.beginPath(); ctx.arc(cx-4, cy+2, 4, 0, Math.PI*2); ctx.fill();
                ctx.beginPath(); ctx.arc(cx+4, cy+2, 4, 0, Math.PI*2); ctx.fill();
                ctx.fillStyle = "rgba(139, 92, 246, 0.4)"; ctx.beginPath(); ctx.arc(cx-4, cy+35, 8, 0, Math.PI*2); ctx.fill();
                ctx.beginPath(); ctx.arc(cx+4, cy+35, 8, 0, Math.PI*2); ctx.fill();
            } else {
                ctx.fillStyle = c; ctx.fillRect(cx-8, cy+10, 16, 22);
                ctx.beginPath(); ctx.arc(cx, cy+4, 14, 0, Math.PI*2); ctx.fill(); ctx.stroke();
                ctx.fillStyle = "white"; ctx.beginPath(); ctx.arc(cx-5, cy+2, 5, 0, Math.PI*2); ctx.fill(); ctx.stroke();
                ctx.beginPath(); ctx.arc(cx+5, cy+2, 5, 0, Math.PI*2); ctx.fill(); ctx.stroke();
                ctx.fillStyle = "#000"; ctx.beginPath(); ctx.arc(cx-4, cy+3, 2.5, 0, Math.PI*2); ctx.fill();
                ctx.beginPath(); ctx.arc(cx+6, cy+3, 2.5, 0, Math.PI*2); ctx.fill();
                if(costume.type === 'Маг') { ctx.fillStyle = "#8b5cf6"; ctx.fillRect(cx-16, cy+10, 6, 28); ctx.fillRect(cx+10, cy+10, 6, 28); }
                if(costume.type === 'Зомби') { ctx.fillStyle = "#e0e0e0"; ctx.fillRect(cx-6, cy-4, 12, 8); }
                if(costume.type === 'Рыцарь') { ctx.fillStyle = "#94a3b8"; ctx.fillRect(cx-4, cy-6, 8, 4); ctx.fillRect(cx+4, cy+5, 4, 20); }
            }
        }

        function draw() {
            ctx.clearRect(0, 0, 1200, 700);
            ctx.fillStyle = "#87CEEB"; ctx.fillRect(0, 0, 1200, 700);
            ctx.fillStyle = "#3e2723"; ctx.fillRect(0, 670, 1200, 30);

            for(let s of levelData.spikes) { ctx.fillStyle = "#dc2626"; ctx.beginPath(); ctx.moveTo(s.x+10, s.y-20); ctx.lineTo(s.x, s.y); ctx.lineTo(s.x+20, s.y); ctx.closePath(); ctx.fill(); }
            for(let t of levelData.trampolines) { ctx.fillStyle = "#facc15"; ctx.beginPath(); ctx.arc(t.x, t.y, t.r, 0, Math.PI*2); ctx.fill(); }
            for(let g of levelData.gears) { ctx.fillStyle = "#64748b"; ctx.save(); ctx.translate(g.x, g.y); ctx.rotate(g.angle); for(let i=0; i<10; i++){ ctx.rotate(Math.PI/5); ctx.fillRect(g.r-6,-2,12,4); } ctx.beginPath(); ctx.arc(0,0,g.r-6,0,Math.PI*2); ctx.fill(); ctx.restore(); }
            for(let b of levelData.bots) {
                if(!b.alive) continue;
                ctx.fillStyle = "#1e293b"; ctx.fillRect(b.x, b.y, 30, 30);
                ctx.fillStyle = "#e74c3c"; ctx.fillRect(b.x+5, b.y+5, 8, 8); ctx.fillRect(b.x+17, b.y+5, 8, 8);
                ctx.fillStyle = "#fff"; ctx.fillRect(b.x+7, b.y+7, 4, 4); ctx.fillRect(b.x+19, b.y+7, 4, 4);
                ctx.fillStyle = "#facc15"; ctx.fillRect(b.x+12, b.y+22, 6, 3); // Зубы
            }
            for(let c of levelData.coins) { ctx.fillStyle = "#facc15"; ctx.beginPath(); ctx.arc(c.x, c.y, c.r, 0, Math.PI*2); ctx.fill(); }

            if(!player.isDead || player.respawnTimer%20<10) drawHero(player.x+15, player.y+2, currentCostume);
        }

        // --- ЗАПУСК ---
        renderSkins();
        document.getElementById('startBtn').addEventListener('click', () => {
            document.getElementById('menu').style.display = 'none';
            document.getElementById('gameWrapper').style.display = 'block';
            generateLevel(0); updateHUD();
            function loop(){ update(); draw(); requestAnimationFrame(loop); }
            loop();
        });
    </script>
</body>
</html>
