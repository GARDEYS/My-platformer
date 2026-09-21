
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>Платформер</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        html, body {
            width: 100%; height: 100%;
            overflow: hidden;
            background: #000;
            font-family: Arial, sans-serif;
            touch-action: none;
            -webkit-user-select: none;
            -moz-user-select: none;
            -ms-user-select: none;
            user-select: none;
        }
        /* Canvas растягивается на весь экран, сохраняя пропорции */
        canvas {
            display: block;
            position: fixed;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background: #87CEEB;
            image-rendering: pixelated;
            max-width: 100vw;
            max-height: 100vh;
            width: auto;
            height: auto;
        }
        #controls {
            position: fixed; bottom: 20px; left: 0; right: 0;
            display: flex; justify-content: space-between;
            padding: 0 20px; pointer-events: none; z-index: 10;
        }
        .btn {
            width: 80px; height: 80px;
            background: rgba(255,255,255,0.4);
            border: 3px solid rgba(255,255,255,0.7);
            border-radius: 50%;
            display: flex; align-items: center; justify-content: center;
            font-size: 40px; color: #fff;
            user-select: none; pointer-events: all; touch-action: none;
        }
        .btn:active { background: rgba(255,255,255,0.7); }
        #left-btns { display: flex; gap: 15px; }
        #lb-btn {
            position: fixed; top: 15px; right: 15px;
            padding: 10px 18px; background: rgba(0,0,0,0.5);
            color: #fff; border: 2px solid #fff; border-radius: 8px;
            cursor: pointer; font-size: 15px; z-index: 20;
        }
        #lb-btn:hover { background: rgba(0,0,0,0.8); }
        #sound-btn {
            position: fixed; top: 15px; left: 15px;
            width: 50px; height: 50px;
            padding: 0; background: rgba(0,0,0,0.5);
            color: #fff; border: 2px solid #fff; border-radius: 50%;
            cursor: pointer; font-size: 22px; z-index: 20;
            display: flex; align-items: center; justify-content: center;
        }
        #sound-btn:hover { background: rgba(0,0,0,0.8); }
        #lb-modal {
            position: fixed; inset: 0; display: none;
            background: rgba(0,0,0,0.85); z-index: 30;
            justify-content: center; align-items: center; padding: 20px;
        }
        #lb-modal.active { display: flex; }
        #lb-box {
            background: #1a1a2e; color: #fff; padding: 25px;
            border-radius: 15px; max-width: 400px; width: 100%;
            max-height: 80vh; overflow-y: auto; border: 2px solid #444;
        }
        #lb-box h2 { margin-bottom: 15px; text-align: center; color: #FFD700; }
        .lb-row {
            display: flex; justify-content: space-between;
            padding: 10px; border-bottom: 1px solid #333; font-size: 16px;
        }
        .lb-row.self { background: rgba(255,215,0,0.15); border-radius: 5px; }
        .lb-row .rank { font-weight: bold; color: #FFD700; min-width: 40px; }
        #lb-close {
            margin-top: 15px; width: 100%; padding: 12px;
            background: #E53935; color: #fff; border: none;
            border-radius: 8px; font-size: 16px; cursor: pointer;
        }
        #lb-loading { text-align: center; padding: 20px; color: #aaa; }
    </style>
</head>
<body>
    <canvas id="game" width="960" height="540"></canvas>
    <button id="sound-btn" title="Звук">🔊</button>
    <button id="lb-btn">🏆 <span data-i18n="leaders">Лидеры</span></button>

    <div id="lb-modal">
        <div id="lb-box">
            <h2>🏆 <span data-i18n="records">Рекорды</span></h2>
            <div id="lb-content"><div id="lb-loading" data-i18n="loading">Загрузка...</div></div>
            <button id="lb-close" data-i18n="close">Закрыть</button>
        </div>
    </div>

    <div id="controls">
        <div id="left-btns">
            <div class="btn" id="btn-left">◀</div>
            <div class="btn" id="btn-right">▶</div>
        </div>
        <div class="btn" id="btn-jump">▲</div>
    </div>

    <script src="https://yandex.ru/games/sdk/v2"></script>
    <script>
        // ============================================
        // ЗВУКОВОЙ ДВИЖОК (Web Audio API)
        // ============================================
        const Sound = (() => {
            let actx = null;
            let masterGain = null;
            let musicGain = null;
            let sfxGain = null;
            let enabled = true;
            let unlocked = false;
            let musicTimer = null;
            let musicPlaying = false;

            function init() {
                if (actx) return;
                try {
                    actx = new (window.AudioContext || window.webkitAudioContext)();
                    masterGain = actx.createGain();
                    masterGain.gain.value = 0.6;
                    masterGain.connect(actx.destination);

                    musicGain = actx.createGain();
                    musicGain.gain.value = 0.15;
                    musicGain.connect(masterGain);

                    sfxGain = actx.createGain();
                    sfxGain.gain.value = 0.5;
                    sfxGain.connect(masterGain);
                } catch(e) { console.warn('AudioContext error:', e); }
            }

            function unlock() {
                init();
                if (!actx) return;
                if (actx.state === 'suspended') actx.resume();
                unlocked = true;
            }

            function tone(freq, duration, type = 'sine', vol = 1, dest = null, slideTo = null) {
                if (!enabled || !unlocked || !actx) return;
                try {
                    const osc = actx.createOscillator();
                    const g = actx.createGain();
                    osc.type = type;
                    osc.frequency.value = freq;
                    if (slideTo) osc.frequency.exponentialRampToValueAtTime(slideTo, actx.currentTime + duration);
                    g.gain.setValueAtTime(0, actx.currentTime);
                    g.gain.linearRampToValueAtTime(vol, actx.currentTime + 0.01);
                    g.gain.exponentialRampToValueAtTime(0.001, actx.currentTime + duration);
                    osc.connect(g);
                    g.connect(dest || sfxGain);
                    osc.start();
                    osc.stop(actx.currentTime + duration + 0.05);
                } catch(e) {}
            }

            function noise(duration, vol = 0.3) {
                if (!enabled || !unlocked || !actx) return;
                try {
                    const bufferSize = actx.sampleRate * duration;
                    const buffer = actx.createBuffer(1, bufferSize, actx.sampleRate);
                    const data = buffer.getChannelData(0);
                    for (let i = 0; i < bufferSize; i++) data[i] = Math.random() * 2 - 1;
                    const src = actx.createBufferSource();
                    src.buffer = buffer;
                    const g = actx.createGain();
                    g.gain.setValueAtTime(vol, actx.currentTime);
                    g.gain.exponentialRampToValueAtTime(0.001, actx.currentTime + duration);
                    src.connect(g);
                    g.connect(sfxGain);
                    src.start();
                } catch(e) {}
            }

            function jump() { tone(520, 0.15, 'square', 0.4, null, 880); }
            function doubleJump() { tone(680, 0.18, 'square', 0.4, null, 1100); }
            function coin() {
                tone(880, 0.08, 'sine', 0.5);
                setTimeout(() => tone(1320, 0.12, 'sine', 0.5), 60);
            }
            function bonus() {
                tone(523, 0.1, 'triangle', 0.5);
                setTimeout(() => tone(659, 0.1, 'triangle', 0.5), 80);
                setTimeout(() => tone(784, 0.15, 'triangle', 0.5), 160);
            }
            function hurt() {
                noise(0.25, 0.4);
                tone(200, 0.3, 'sawtooth', 0.3, null, 80);
            }
            function stomp() {
                tone(300, 0.12, 'square', 0.5, null, 150);
                setTimeout(() => tone(600, 0.1, 'square', 0.4), 60);
            }
            function bossShoot() { tone(180, 0.2, 'sawtooth', 0.35, null, 90); }
            function victory() {
                [523, 659, 784, 1047].forEach((f, i) => setTimeout(() => tone(f, 0.2, 'triangle', 0.6), i * 120));
            }
            function defeat() {
                [440, 349, 262, 196].forEach((f, i) => setTimeout(() => tone(f, 0.25, 'sawtooth', 0.5), i * 150));
            }

            const melody = [
                262, 330, 392, 330, 262, 330, 392, 523,
                440, 392, 330, 262, 294, 330, 262, 220
            ];
            let melodyIdx = 0;
            function playMusicNote() {
                if (!enabled || !unlocked || !actx) return;
                const f = melody[melodyIdx % melody.length];
                melodyIdx++;
                tone(f, 0.35, 'triangle', 0.4, musicGain);
                if (melodyIdx % 4 === 1) tone(f / 2, 0.6, 'sine', 0.5, musicGain);
            }
            function startMusic() {
                if (musicPlaying) return;
                musicPlaying = true;
                melodyIdx = 0;
                playMusicNote();
                musicTimer = setInterval(playMusicNote, 380);
            }
            function stopMusic() {
                musicPlaying = false;
                if (musicTimer) { clearInterval(musicTimer); musicTimer = null; }
            }
            function toggle() {
                enabled = !enabled;
                if (masterGain) masterGain.gain.value = enabled ? 0.6 : 0;
                return enabled;
            }
            function pauseAll() { if (masterGain) masterGain.gain.value = 0; }
            function resumeAll() { if (masterGain) masterGain.gain.value = enabled ? 0.6 : 0; }

            return {
                init, unlock, toggle, pauseAll, resumeAll,
                jump, doubleJump, coin, bonus, hurt, stomp, bossShoot,
                victory, defeat, startMusic, stopMusic
            };
        })();

        function firstInteraction() {
            Sound.unlock();
            Sound.startMusic();
            document.removeEventListener('pointerdown', firstInteraction);
            document.removeEventListener('keydown', firstInteraction);
            document.removeEventListener('touchstart', firstInteraction);
        }
        document.addEventListener('pointerdown', firstInteraction);
        document.addEventListener('keydown', firstInteraction);
        document.addEventListener('touchstart', firstInteraction);

        document.getElementById('sound-btn').addEventListener('click', () => {
            Sound.unlock();
            const on = Sound.toggle();
            document.getElementById('sound-btn').textContent = on ? '🔊' : '🔇';
            if (on) Sound.startMusic(); else Sound.stopMusic();
        });

        // ============================================
        // ЛОКАЛИЗАЦИЯ
        // ============================================
        const i18n = {
            ru: {
                leaders: 'Лидеры', records: 'Рекорды', loading: 'Загрузка...', close: 'Закрыть',
                noRecords: 'Пока нет рекордов', lbUnavailable: 'Лидерборд недоступен',
                lbError: 'Ошибка загрузки', player: 'Игрок', level: 'Уровень',
                passed: 'ПРОЙДЕН!', allPassed: 'ВСЕ УРОВНИ ПРОЙДЕНЫ!',
                score: 'Очки', record: 'Рекорд', nextLevel: 'Загрузка следующего уровня...',
                hint: '🔼×2 двойной прыжок · 🧲 магнит · 🐢 замедление · 👹 босс на 2-м уровне',
                boss: 'БОСС',
                shield: 'Щит', speed: 'Ускорение', heart: 'Жизнь', magnet: 'Магнит', slow: 'Замедление'
            },
            en: {
                leaders: 'Leaders', records: 'Records', loading: 'Loading...', close: 'Close',
                noRecords: 'No records yet', lbUnavailable: 'Leaderboard unavailable',
                lbError: 'Loading error', player: 'Player', level: 'Level',
                passed: 'PASSED!', allPassed: 'ALL LEVELS PASSED!',
                score: 'Score', record: 'Record', nextLevel: 'Loading next level...',
                hint: '🔼×2 double jump · 🧲 magnet · 🐢 slow-mo · 👹 boss on level 2',
                boss: 'BOSS',
                shield: 'Shield', speed: 'Speed', heart: 'Life', magnet: 'Magnet', slow: 'Slow-mo'
            }
        };
        let currentLang = 'ru';
        function t(key) { return i18n[currentLang][key] || i18n.ru[key] || key; }
        function applyLocalization() {
            document.querySelectorAll('[data-i18n]').forEach(el => {
                const key = el.getAttribute('data-i18n');
                el.textContent = t(key);
            });
        }

        // ============================================
        // YANDEX SDK
        // ============================================
        const LB_NAME = 'bestScore';
        let ysdk = null, ysdkPlayer = null, ysdkLB = null;
        let bestScore = 0, canSave = false;
        let paused = false;

        function loadLocalBest() { try { return parseInt(localStorage.getItem('bestScore')) || 0; } catch(e){ return 0; } }
        function saveLocalBest(v) { try { localStorage.setItem('bestScore', v); } catch(e){} }
        bestScore = loadLocalBest();

        YaGames.init().then(async sdk => {
            ysdk = sdk;

            try {
                const lang = ysdk.environment.i18n.lang;
                currentLang = (lang === 'en') ? 'en' : 'ru';
                applyLocalization();
            } catch(e) {}

            try {
                ysdkPlayer = await ysdk.getPlayer({ scopes: false });
                canSave = true;
                const data = await ysdkPlayer.getData(['bestScore']);
                if (data && typeof data.bestScore === 'number') bestScore = Math.max(bestScore, data.bestScore);
            } catch(e) {}

            try { ysdkLB = await ysdk.getLeaderboards(); } catch(e) {}

            const handlePause = () => {
                paused = true;
                Sound.pauseAll();
                if (ysdk && ysdk.features && ysdk.features.GameplayAPI) {
                    try { ysdk.features.GameplayAPI.stop(); } catch(e) {}
                }
            };
            const handleResume = () => {
                paused = false;
                Sound.resumeAll();
                if (ysdk && ysdk.features && ysdk.features.GameplayAPI) {
                    try { ysdk.features.GameplayAPI.start(); } catch(e) {}
                }
            };
            try {
                ysdk.on('game_api_pause', handlePause);
                ysdk.on('game_api_resume', handleResume);
            } catch(e) {}

            try {
                ysdk.features.LoadingAPI.ready();
                console.log('✅ Game Ready вызван');
            } catch(e) {}

            try { ysdk.features.GameplayAPI?.start(); } catch(e) {}
            try { ysdk.adv.showFullscreenAdv(); } catch(e) {}
        }).catch(e => console.log('SDK err:', e));

        async function saveProgress() {
            saveLocalBest(bestScore);
            if (canSave && ysdkPlayer) { try { await ysdkPlayer.setData({ bestScore }, false); } catch(e){} }
        }
        async function submitLeaderboard(s) {
            if (!ysdkLB) return;
            try { await ysdkLB.setLeaderboardScore(LB_NAME, s); } catch(e){}
        }
        async function showLeaderboard() {
            const modal = document.getElementById('lb-modal');
            const content = document.getElementById('lb-content');
            modal.classList.add('active');
            if (!ysdkLB) { content.innerHTML = `<div id="lb-loading">${t('lbUnavailable')}</div>`; return; }
            content.innerHTML = `<div id="lb-loading">${t('loading')}</div>`;
            try {
                const r = await ysdkLB.getLeaderboardEntries(LB_NAME, { quantityTop: 10, includeUser: true, quantityAround: 3 });
                let html = '';
                if (!r.entries || r.entries.length === 0) html = `<div id="lb-loading">${t('noRecords')}</div>`;
                else r.entries.forEach(e => {
                    const self = r.userRank && e.rank === r.userRank;
                    const name = (e.player && e.player.publicName) || t('player');
                    html += `<div class="lb-row ${self ? 'self' : ''}">
                        <span class="rank">#${e.rank}</span>
                        <span>${name}</span>
                        <span><b>${e.score}</b></span></div>`;
                });
                content.innerHTML = html;
            } catch(e) { content.innerHTML = `<div id="lb-loading">${t('lbError')}</div>`; }
        }
        document.getElementById('lb-btn').addEventListener('click', showLeaderboard);
        document.getElementById('lb-close').addEventListener('click', () => document.getElementById('lb-modal').classList.remove('active'));

        window.addEventListener('contextmenu', e => e.preventDefault());

        // ============================================
        // КАНВАС — ФИКСИРОВАННЫЙ 960×540, растягивается через CSS
        // ============================================
        const canvas = document.getElementById('game');
        const ctx = canvas.getContext('2d');
        const W = 960;
        const H = 540;

        // Ничего не делаем при ресайзе — CSS сам масштабирует canvas
        function resize() {
            canvas.width = W;
            canvas.height = H;
        }
        resize();

        // ============================================
        // КОНСТАНТЫ
        // ============================================
        const GRAVITY = 0.6;
        const JUMP_POWER = -13;
        const MOVE_SPEED = 4.5;
        const MAX_JUMPS = 2;
        const MAX_LIVES = 5;
        const EFFECT_DURATION = 300;
        const BONUS_RESPAWN = 600;
        const MAGNET_RADIUS = 240;
        const SLOW_FACTOR = 0.35;

        // ============================================
        // УРОВНИ
        // ============================================
        const LEVELS = [
            {
                name: 'Уровень 1',
                spawn: { x: 60, y: 200 },
                platforms: [
                    { x: 0, y: 480, w: 300, h: 60 },
                    { x: 380, y: 420, w: 120, h: 20 },
                    { x: 580, y: 350, w: 120, h: 20 },
                    { x: 780, y: 280, w: 120, h: 20 },
                    { x: 500, y: 200, w: 120, h: 20 },
                    { x: 250, y: 150, w: 120, h: 20 },
                    { x: 0, y: 100, w: 100, h: 20 },
                ],
                coins: [
                    { x: 420, y: 380 }, { x: 620, y: 310 },
                    { x: 820, y: 240 }, { x: 540, y: 160 },
                    { x: 290, y: 110 },
                ],
                bonuses: [
                    { type: 'shield', x: 340, y: 340 },
                    { type: 'speed',  x: 700, y: 220 },
                    { type: 'heart',  x: 180, y: 260 },
                    { type: 'magnet', x: 460, y: 460 },
                    { type: 'slow',   x: 850, y: 130 },
                ],
                enemies: [
                    { type: 'patrol', x: 400, y: 388, w: 28, h: 30, platX: 380, platW: 120, dir: 1, speed: 1.4 },
                    { type: 'patrol', x: 600, y: 318, w: 28, h: 30, platX: 580, platW: 120, dir: -1, speed: 1.6 },
                    { type: 'patrol', x: 520, y: 168, w: 28, h: 30, platX: 500, platW: 120, dir: 1, speed: 1.8 },
                    { type: 'flyer', x: 700, y: 200, w: 30, h: 30, baseY: 200, dir: 1, speed: 1.2, range: 60 },
                    { type: 'flyer', x: 180, y: 300, w: 30, h: 30, baseY: 300, dir: -1, speed: 1.0, range: 80 },
                ],
                flag: { x: 30, y: 40, w: 30, h: 60 },
            },
            {
                name: 'Уровень 2',
                spawn: { x: 40, y: 400 },
                platforms: [
                    { x: 0, y: 500, w: 180, h: 40 },
                    { x: 260, y: 470, w: 90, h: 18 },
                    { x: 430, y: 420, w: 90, h: 18 },
                    { x: 600, y: 370, w: 90, h: 18 },
                    { x: 780, y: 320, w: 180, h: 20 },
                    { x: 620, y: 240, w: 100, h: 18 },
                    { x: 420, y: 180, w: 100, h: 18 },
                    { x: 200, y: 130, w: 100, h: 18 },
                    { x: 0, y: 80, w: 140, h: 20 },
                    { x: 300, y: 520, w: 300, h: 20 },
                    { x: 680, y: 520, w: 280, h: 20 },
                ],
                coins: [
                    { x: 300, y: 430 }, { x: 470, y: 380 },
                    { x: 640, y: 330 }, { x: 860, y: 280 },
                    { x: 460, y: 140 }, { x: 250, y: 90 },
                ],
                bonuses: [
                    { type: 'magnet', x: 430, y: 380 },
                    { type: 'slow',   x: 820, y: 280 },
                    { type: 'shield', x: 470, y: 140 },
                    { type: 'speed',  x: 130, y: 250 },
                    { type: 'heart',  x: 700, y: 470 },
                ],
                enemies: [
                    { type: 'patrol', x: 260, y: 440, w: 28, h: 30, platX: 260, platW: 90, dir: 1, speed: 1.5 },
                    { type: 'patrol', x: 600, y: 340, w: 28, h: 30, platX: 600, platW: 90, dir: -1, speed: 2.0 },
                    { type: 'patrol', x: 300, y: 490, w: 28, h: 30, platX: 300, platW: 300, dir: 1, speed: 2.4 },
                    { type: 'patrol', x: 420, y: 150, w: 28, h: 30, platX: 420, platW: 100, dir: 1, speed: 1.7 },
                    { type: 'flyer', x: 550, y: 200, w: 30, h: 30, baseY: 220, dir: 1, speed: 1.4, range: 90 },
                    { type: 'flyer', x: 900, y: 420, w: 30, h: 30, baseY: 420, dir: -1, speed: 1.6, range: 70 },
                    { type: 'flyer', x: 120, y: 200, w: 30, h: 30, baseY: 220, dir: 1, speed: 1.5, range: 100 },
                ],
                flag: { x: 90, y: 20, w: 30, h: 60 },
            },
        ];

        // ============================================
        // ПЕРЕМЕННЫЕ
        // ============================================
        let currentLevel = 0;
        let platforms = [], coins = [], bonuses = [], enemies = [], flag = null, SPAWN = null;
        let TOTAL_COINS = 0;
        let particles = [];
        let score = 0;
        let lives = 3;
        let win = false;
        let winText = '';
        let hurtFlash = 0;
        let jumpEdge = false;

        // ============================================
        // ИГРОК
        // ============================================
        const player = {
            x: 0, y: 0, w: 32, h: 42,
            vx: 0, vy: 0, onGround: false, facing: 1,
            invuln: 0, jumpsLeft: MAX_JUMPS,
            shieldTime: 0, speedTime: 0,
            magnetTime: 0, slowTime: 0,
            animFrame: 0
        };

        // ============================================
        // БОСС
        // ============================================
        const boss = {
            x: 0, y: 0, w: 80, h: 80,
            hp: 5, maxHp: 5,
            vx: 1.8, invuln: 0,
            active: false, shootTimer: 120, hurtTimer: 0
        };
        let bossProjectiles = [];
        let bossDefeated = false;

        // ============================================
        // ЗАГРУЗКА УРОВНЯ
        // ============================================
        function loadLevel(idx) {
            const L = LEVELS[idx];
            SPAWN = { ...L.spawn };
            platforms = L.platforms.map(p => ({ ...p }));
            coins = L.coins.map(c => ({ ...c, r: 12, collected: false }));
            bonuses = L.bonuses.map(b => ({ ...b, r: 16, active: true, respawnTimer: 0, phase: Math.random() * Math.PI * 2 }));
            enemies = L.enemies.map(e => ({ ...e }));
            flag = { ...L.flag };
            TOTAL_COINS = coins.length;

            player.x = SPAWN.x; player.y = SPAWN.y;
            player.vx = 0; player.vy = 0;
            player.invuln = 60;
            player.jumpsLeft = MAX_JUMPS;
            player.shieldTime = 0; player.speedTime = 0;
            player.magnetTime = 0; player.slowTime = 0;
            particles = [];

            boss.active = false;
            boss.hp = boss.maxHp;
            boss.invuln = 0;
            boss.hurtTimer = 0;
            boss.shootTimer = 120;
            bossProjectiles = [];
            bossDefeated = false;
        }

        // ============================================
        // УПРАВЛЕНИЕ
        // ============================================
        const keys = { left: false, right: false, jump: false };
        document.addEventListener('keydown', e => {
            if (e.key === 'ArrowLeft' || e.key === 'a' || e.key === 'A') keys.left = true;
            if (e.key === 'ArrowRight' || e.key === 'd' || e.key === 'D') keys.right = true;
            if (e.key === 'ArrowUp' || e.key === 'w' || e.key === 'W' || e.key === ' ') {
                if (!keys.jump) jumpEdge = true;
                keys.jump = true;
            }
        });
        document.addEventListener('keyup', e => {
            if (e.key === 'ArrowLeft' || e.key === 'a' || e.key === 'A') keys.left = false;
            if (e.key === 'ArrowRight' || e.key === 'd' || e.key === 'D') keys.right = false;
            if (e.key === 'ArrowUp' || e.key === 'w' || e.key === 'W' || e.key === ' ') keys.jump = false;
        });

        function bindTouch(id, key) {
            const el = document.getElementById(id);
            const press = () => { if (key === 'jump' && !keys.jump) jumpEdge = true; keys[key] = true; };
            el.addEventListener('touchstart', e => { e.preventDefault(); press(); }, { passive: false });
            el.addEventListener('touchend', e => { e.preventDefault(); keys[key] = false; }, { passive: false });
            el.addEventListener('touchcancel', e => { e.preventDefault(); keys[key] = false; }, { passive: false });
            el.addEventListener('mousedown', press);
            el.addEventListener('mouseup', () => keys[key] = false);
            el.addEventListener('mouseleave', () => keys[key] = false);
        }
        bindTouch('btn-left', 'left'); bindTouch('btn-right', 'right'); bindTouch('btn-jump', 'jump');

        // ============================================
        // ХЕЛПЕРЫ
        // ============================================
        function rectsCollide(a, b) {
            return a.x < b.x + b.w && a.x + a.w > b.x && a.y < b.y + b.h && a.y + a.h > b.y;
        }
        function circleRectCollide(cx, cy, r, rect) {
            const cX = Math.max(rect.x, Math.min(cx, rect.x + rect.w));
            const cY = Math.max(rect.y, Math.min(cy, rect.y + rect.h));
            const dx = cx - cX, dy = cy - cY;
            return dx*dx + dy*dy < r*r;
        }
        function spawnParticles(x, y, color, count = 10) {
            for (let i = 0; i < count; i++) {
                const a = Math.random() * Math.PI * 2;
                const s = 1 + Math.random() * 3;
                particles.push({
                    x, y, vx: Math.cos(a)*s, vy: Math.sin(a)*s - 1,
                    life: 30, maxLife: 30, color, size: 2 + Math.random()*3
                });
            }
        }
        function getMoveSpeed() { return player.speedTime > 0 ? MOVE_SPEED * 1.6 : MOVE_SPEED; }

        function hitPlayer() {
            if (win || paused) return;
            if (player.shieldTime > 0) {
                player.shieldTime = 0;
                player.invuln = 60;
                Sound.hurt();
                spawnParticles(player.x + player.w/2, player.y + player.h/2, '#2196F3', 20);
                return;
            }
            if (player.invuln > 0) return;
            lives--;
            hurtFlash = 20;
            Sound.hurt();
            spawnParticles(player.x + player.w/2, player.y + player.h/2, '#E53935', 15);
            if (lives <= 0) {
                Sound.defeat();
                lives = 3;
                score = 0;
                loadLevel(currentLevel);
            } else {
                player.x = SPAWN.x; player.y = SPAWN.y;
                player.vx = 0; player.vy = 0;
                player.invuln = 60;
                player.jumpsLeft = MAX_JUMPS;
            }
        }

        async function onWin() {
            win = true;
            Sound.victory();
            try { if (ysdk) ysdk.adv.showFullscreenAdv(); } catch(e){}

            if (currentLevel < LEVELS.length - 1) {
                winText = t('level') + ' ' + (currentLevel + 1) + ' ' + t('passed');
                setTimeout(() => {
                    currentLevel++;
                    loadLevel(currentLevel);
                    win = false;
                }, 2200);
            } else {
                winText = t('allPassed');
                if (score > bestScore) {
                    bestScore = score;
                    await saveProgress();
                    await submitLeaderboard(score);
                }
                setTimeout(() => {
                    currentLevel = 0;
                    score = 0; lives = 3;
                    loadLevel(0);
                    win = false;
                }, 4000);
            }
        }

        // ============================================
        // БОСС
        // ============================================
        function spawnBoss() {
            boss.active = true;
            boss.hp = boss.maxHp;
            boss.x = W/2 - boss.w/2;
            boss.y = 60;
            boss.vx = 1.8;
            boss.shootTimer = 90;
            boss.invuln = 60;
            boss.hurtTimer = 0;
            Sound.bonus();
            spawnParticles(boss.x + boss.w/2, boss.y + boss.h/2, '#FF0000', 30);
        }

        function updateBoss() {
            if (boss.invuln > 0) boss.invuln--;
            if (boss.hurtTimer > 0) boss.hurtTimer--;

            boss.x += boss.vx;
            if (boss.x < 60) { boss.x = 60; boss.vx = Math.abs(boss.vx); }
            if (boss.x + boss.w > W - 60) { boss.x = W - 60 - boss.w; boss.vx = -Math.abs(boss.vx); }

            boss.shootTimer--;
            if (boss.shootTimer <= 0) {
                boss.shootTimer = 80 + Math.random() * 60;
                Sound.bossShoot();
                const pcx = player.x + player.w/2;
                const pcy = player.y + player.h/2;
                const bcx = boss.x + boss.w/2;
                const bcy = boss.y + boss.h/2;
                const dx = pcx - bcx, dy = pcy - bcy;
                const dist = Math.sqrt(dx*dx + dy*dy) || 1;
                const speed = 4.5;
                bossProjectiles.push({
                    x: bcx, y: bcy,
                    vx: (dx/dist)*speed, vy: (dy/dist)*speed,
                    r: 12, life: 240
                });
            }

            if (rectsCollide(player, boss)) {
                const stomp = player.vy > 0 && (player.y + player.h) < (boss.y + boss.h * 0.6);
                if (stomp) {
                    if (boss.invuln <= 0) {
                        boss.hp--;
                        boss.invuln = 40;
                        boss.hurtTimer = 20;
                        player.vy = -11;
                        Sound.stomp();
                        spawnParticles(boss.x + boss.w/2, boss.y, '#FFD700', 25);
                        if (boss.hp <= 0) {
                            bossDefeated = true;
                            boss.active = false;
                            score += 100;
                            Sound.victory();
                            spawnParticles(boss.x + boss.w/2, boss.y + boss.h/2, '#FF5722', 60);
                            try { if (ysdk) ysdk.adv.showFullscreenAdv(); } catch(e) {}
                        }
                    } else {
                        player.vy = -11;
                    }
                } else {
                    hitPlayer();
                }
            }
        }

        function updateBossProjectiles() {
            for (let i = bossProjectiles.length - 1; i >= 0; i--) {
                const p = bossProjectiles[i];
                p.x += p.vx; p.y += p.vy; p.life--;
                if (p.life <= 0 || p.x < -50 || p.x > W + 50 || p.y < -50 || p.y > H + 50) {
                    bossProjectiles.splice(i, 1);
                    continue;
                }
                const dx = p.x - (player.x + player.w/2);
                const dy = p.y - (player.y + player.h/2);
                if (Math.sqrt(dx*dx + dy*dy) < p.r + 18) {
                    if (player.shieldTime > 0) {
                        player.shieldTime = 0;
                        player.invuln = 60;
                        Sound.hurt();
                        spawnParticles(p.x, p.y, '#2196F3', 15);
                    } else {
                        hitPlayer();
                    }
                    bossProjectiles.splice(i, 1);
                }
            }
        }

        // ============================================
        // ОБНОВЛЕНИЕ
        // ============================================
        function update() {
            if (paused) return;

            if (player.invuln > 0) player.invuln--;
            if (hurtFlash > 0) hurtFlash--;
            if (player.shieldTime > 0) player.shieldTime--;
            if (player.speedTime > 0) player.speedTime--;
            if (player.magnetTime > 0) player.magnetTime--;
            if (player.slowTime > 0) player.slowTime--;
            player.animFrame++;

            for (let i = particles.length - 1; i >= 0; i--) {
                const p = particles[i];
                p.x += p.vx; p.y += p.vy; p.vy += 0.15; p.life--;
                if (p.life <= 0) particles.splice(i, 1);
            }

            for (const b of bonuses) {
                if (!b.active) { b.respawnTimer--; if (b.respawnTimer <= 0) b.active = true; }
            }

            if (win) { jumpEdge = false; return; }

            const enemyMul = player.slowTime > 0 ? SLOW_FACTOR : 1;

            for (const e of enemies) {
                if (e.type === 'patrol') {
                    e.x += e.speed * e.dir * enemyMul;
                    if (e.x <= e.platX) { e.x = e.platX; e.dir = 1; }
                    if (e.x + e.w >= e.platX + e.platW) { e.x = e.platX + e.platW - e.w; e.dir = -1; }
                } else if (e.type === 'flyer') {
                    e.y += e.speed * e.dir * enemyMul;
                    if (e.y <= e.baseY - e.range) { e.y = e.baseY - e.range; e.dir = 1; }
                    if (e.y >= e.baseY + e.range) { e.y = e.baseY + e.range; e.dir = -1; }
                }
            }

            if (player.magnetTime > 0) {
                const pcx = player.x + player.w/2, pcy = player.y + player.h/2;
                for (const c of coins) {
                    if (c.collected) continue;
                    const dx = pcx - c.x, dy = pcy - c.y;
                    const dist = Math.sqrt(dx*dx + dy*dy);
                    if (dist < MAGNET_RADIUS && dist > 1) {
                        const force = (MAGNET_RADIUS - dist) / MAGNET_RADIUS * 7;
                        c.x += (dx / dist) * force;
                        c.y += (dy / dist) * force;
                    }
                }
            }

            player.vx = 0;
            const spd = getMoveSpeed();
            if (keys.left) { player.vx = -spd; player.facing = -1; }
            if (keys.right) { player.vx = spd; player.facing = 1; }

            if (jumpEdge && player.jumpsLeft > 0) {
                player.vy = JUMP_POWER;
                player.jumpsLeft--;
                player.onGround = false;
                if (player.jumpsLeft === MAX_JUMPS - 1) Sound.jump();
                else Sound.doubleJump();
                spawnParticles(
                    player.x + player.w/2, player.y + player.h,
                    player.jumpsLeft === MAX_JUMPS - 1 ? '#fff' : '#FFD700', 6
                );
            }
            jumpEdge = false;

            player.vy += GRAVITY;
            if (player.vy > 20) player.vy = 20;

            player.x += player.vx;
            if (player.x < 0) player.x = 0;
            if (player.x + player.w > W) player.x = W - player.w;
            for (const p of platforms) {
                if (rectsCollide(player, p)) {
                    if (player.vx > 0) player.x = p.x - player.w;
                    else if (player.vx < 0) player.x = p.x + p.w;
                }
            }

            player.y += player.vy;
            player.onGround = false;
            for (const p of platforms) {
                if (rectsCollide(player, p)) {
                    if (player.vy > 0) {
                        player.y = p.y - player.h; player.vy = 0; player.onGround = true;
                    } else if (player.vy < 0) {
                        player.y = p.y + p.h; player.vy = 0;
                    }
                }
            }
            if (player.onGround) player.jumpsLeft = MAX_JUMPS;

            if (player.y > H + 200) hitPlayer();

            for (const e of enemies) if (rectsCollide(player, e)) hitPlayer();

            for (const c of coins) {
                if (!c.collected && circleRectCollide(c.x, c.y, c.r, player)) {
                    c.collected = true;
                    score += 10;
                    Sound.coin();
                    spawnParticles(c.x, c.y, '#FFD700', 12);
                }
            }

            for (const b of bonuses) {
                if (!b.active) continue;
                const by = b.y + Math.sin(Date.now() / 300 + b.phase) * 4;
                if (circleRectCollide(b.x, by, b.r, player)) {
                    b.active = false;
                    b.respawnTimer = BONUS_RESPAWN;
                    Sound.bonus();
                    if (b.type === 'shield') { player.shieldTime = EFFECT_DURATION; spawnParticles(b.x, by, '#2196F3', 20); }
                    else if (b.type === 'speed') { player.speedTime = EFFECT_DURATION; spawnParticles(b.x, by, '#FFC107', 20); }
                    else if (b.type === 'heart') { lives = Math.min(lives + 1, MAX_LIVES); spawnParticles(b.x, by, '#E91E63', 20); }
                    else if (b.type === 'magnet') { player.magnetTime = EFFECT_DURATION; spawnParticles(b.x, by, '#9C27B0', 20); }
                    else if (b.type === 'slow') { player.slowTime = EFFECT_DURATION; spawnParticles(b.x, by, '#00BCD4', 20); }
                }
            }

            const collected = coins.filter(c => c.collected).length;

            if (currentLevel === 1 && collected === TOTAL_COINS && !boss.active && !bossDefeated) {
                spawnBoss();
            }

            if (boss.active) updateBoss();
            updateBossProjectiles();

            const bossOk = currentLevel !== 1 || bossDefeated;
            if (!win && rectsCollide(player, flag) && collected === TOTAL_COINS && bossOk) onWin();
        }

        // ============================================
        // ОТРИСОВКА
        // ============================================
        function draw() {
            const grad = ctx.createLinearGradient(0, 0, 0, H);
            if (currentLevel === 0) { grad.addColorStop(0, '#87CEEB'); grad.addColorStop(1, '#E0F6FF'); }
            else { grad.addColorStop(0, '#5B2C6F'); grad.addColorStop(1, '#F5B7B1'); }
            ctx.fillStyle = grad;
            ctx.fillRect(0, 0, W, H);

            ctx.fillStyle = currentLevel === 0 ? '#FFE066' : '#FFB74D';
            ctx.beginPath(); ctx.arc(W - 80, 80, 45, 0, Math.PI * 2); ctx.fill();

            const grass = currentLevel === 0 ? '#4CAF50' : '#7E57C2';
            const dirt = currentLevel === 0 ? '#8B5A2B' : '#3E2723';
            const edge = currentLevel === 0 ? '#5D3A1A' : '#1A0F0A';
            for (const p of platforms) {
                ctx.fillStyle = grass; ctx.fillRect(p.x, p.y, p.w, 8);
                ctx.fillStyle = dirt; ctx.fillRect(p.x, p.y + 8, p.w, p.h - 8);
                ctx.strokeStyle = edge; ctx.lineWidth = 2; ctx.strokeRect(p.x, p.y, p.w, p.h);
            }

            const allCoins = coins.every(c => c.collected);
            ctx.fillStyle = '#666';
            ctx.fillRect(flag.x + flag.w/2 - 2, flag.y, 4, flag.h);
            ctx.fillStyle = allCoins ? '#00C853' : '#E53935';
            ctx.beginPath();
            ctx.moveTo(flag.x + flag.w/2 + 2, flag.y);
            ctx.lineTo(flag.x + flag.w/2 + 30, flag.y + 12);
            ctx.lineTo(flag.x + flag.w/2 + 2, flag.y + 24);
            ctx.closePath(); ctx.fill();

            for (const c of coins) {
                if (c.collected) continue;
                ctx.fillStyle = '#FFD700';
                ctx.beginPath(); ctx.arc(c.x, c.y, c.r, 0, Math.PI*2); ctx.fill();
                ctx.strokeStyle = '#DAA520'; ctx.lineWidth = 3; ctx.stroke();
                ctx.fillStyle = '#FFF8B0';
                ctx.beginPath(); ctx.arc(c.x - 3, c.y - 3, c.r * 0.35, 0, Math.PI*2); ctx.fill();
            }

            for (const b of bonuses) if (b.active) drawBonus(b);
            for (const e of enemies) drawEnemy(e);

            drawBoss();
            drawBossProjectiles();

            for (const p of particles) {
                ctx.globalAlpha = p.life / p.maxLife;
                ctx.fillStyle = p.color;
                ctx.beginPath(); ctx.arc(p.x, p.y, p.size, 0, Math.PI*2); ctx.fill();
            }
            ctx.globalAlpha = 1;

            drawPlayer();

            ctx.fillStyle = 'rgba(0,0,0,0.5)';
            ctx.fillRect(10, 10, 340, 40);
            ctx.fillStyle = '#fff';
            ctx.font = 'bold 20px Arial';
            ctx.textAlign = 'left';
            ctx.fillText('⭐ ' + score, 25, 38);
            ctx.fillText('❤️ ' + lives, 120, 38);
            ctx.fillText('🏆 ' + bestScore, 185, 38);
            ctx.fillStyle = '#FFD700';
            ctx.font = 'bold 16px Arial';
            ctx.fillText('📍 ' + (currentLevel + 1) + '/' + LEVELS.length, 275, 37);

            drawEffects();

            if (score === 0 && !win && lives === 3 && player.invuln < 30) {
                ctx.fillStyle = 'rgba(0,0,0,0.6)';
                ctx.fillRect(W/2 - 280, H - 140, 560, 40);
                ctx.fillStyle = '#fff';
                ctx.font = '15px Arial';
                ctx.textAlign = 'center';
                ctx.fillText(t('hint'), W/2, H - 115);
            }

            if (hurtFlash > 0) {
                ctx.fillStyle = 'rgba(255,0,0,' + (hurtFlash / 40) + ')';
                ctx.fillRect(0, 0, W, H);
            }

            if (player.slowTime > 0) {
                ctx.fillStyle = 'rgba(0,188,212,0.08)';
                ctx.fillRect(0, 0, W, H);
            }

            if (win) {
                ctx.fillStyle = 'rgba(0,0,0,0.7)';
                ctx.fillRect(0, 0, W, H);
                ctx.fillStyle = '#FFD700';
                ctx.font = 'bold 52px Arial';
                ctx.textAlign = 'center';
                ctx.fillText('🏆 ' + winText, W/2, H/2 - 20);
                if (currentLevel === LEVELS.length - 1) {
                    ctx.fillStyle = '#fff';
                    ctx.font = '24px Arial';
                    ctx.fillText(t('score') + ': ' + score, W/2, H/2 + 30);
                    ctx.font = '18px Arial';
                    ctx.fillStyle = '#FFD700';
                    ctx.fillText(t('record') + ': ' + bestScore, W/2, H/2 + 65);
                } else {
                    ctx.fillStyle = '#fff';
                    ctx.font = '20px Arial';
                    ctx.fillText(t('nextLevel'), W/2, H/2 + 40);
                }
            }
        }

        const BONUS_COLORS = { shield: '#2196F3', speed: '#FFC107', heart: '#E91E63', magnet: '#9C27B0', slow: '#00BCD4' };
        const BONUS_ICONS  = { shield: '🛡', speed: '⚡', heart: '♥', magnet: '🧲', slow: '🐢' };

        function drawBonus(b) {
            const by = b.y + Math.sin(Date.now() / 300 + b.phase) * 4;
            const color = BONUS_COLORS[b.type];
            const glow = ctx.createRadialGradient(b.x, by, 0, b.x, by, b.r * 2);
            glow.addColorStop(0, color + '88');
            glow.addColorStop(1, color + '00');
            ctx.fillStyle = glow;
            ctx.beginPath(); ctx.arc(b.x, by, b.r * 2, 0, Math.PI*2); ctx.fill();

            ctx.fillStyle = color;
            ctx.beginPath(); ctx.arc(b.x, by, b.r, 0, Math.PI*2); ctx.fill();
            ctx.strokeStyle = '#fff'; ctx.lineWidth = 3; ctx.stroke();

            ctx.fillStyle = '#fff';
            ctx.font = 'bold 18px Arial';
            ctx.textAlign = 'center';
            ctx.textBaseline = 'middle';
            ctx.fillText(BONUS_ICONS[b.type], b.x, by + 1);
            ctx.textBaseline = 'alphabetic';
        }

        function drawEffects() {
            const active = [];
            if (player.shieldTime > 0) active.push({ type: 'shield', time: player.shieldTime });
            if (player.speedTime > 0)  active.push({ type: 'speed',  time: player.speedTime });
            if (player.magnetTime > 0) active.push({ type: 'magnet', time: player.magnetTime });
            if (player.slowTime > 0)   active.push({ type: 'slow',   time: player.slowTime });

            let x = 10;
            const y = H - 40;
            for (const ef of active) {
                const w = 135, h = 24;
                ctx.fillStyle = 'rgba(0,0,0,0.5)';
                ctx.fillRect(x, y, w, h);
                const pct = ef.time / EFFECT_DURATION;
                ctx.fillStyle = BONUS_COLORS[ef.type];
                ctx.fillRect(x + 2, y + 2, (w - 4) * pct, h - 4);
                ctx.fillStyle = ef.type === 'speed' ? '#000' : '#fff';
                ctx.font = 'bold 14px Arial';
                ctx.textAlign = 'left';
                ctx.fillText(BONUS_ICONS[ef.type] + ' ' + t(ef.type), x + 8, y + 17);
                x += w + 8;
            }
        }

        function drawPlayer() {
            const { x, y, w, h, facing } = player;
            if (player.invuln > 0 && Math.floor(player.invuln / 4) % 2 === 0) return;

            if (player.shieldTime > 0) {
                const pulse = 1 + Math.sin(Date.now() / 100) * 0.1;
                ctx.strokeStyle = 'rgba(33,150,243,' + (0.5 + Math.sin(Date.now()/150) * 0.3) + ')';
                ctx.lineWidth = 3;
                ctx.beginPath();
                ctx.arc(x + w/2, y + h/2, (w/2 + 8) * pulse, 0, Math.PI*2);
                ctx.stroke();
            }

            if (player.speedTime > 0) {
                ctx.strokeStyle = 'rgba(255,193,7,0.5)';
                ctx.lineWidth = 2;
                for (let i = 0; i < 3; i++) {
                    const off = (Date.now() / 30 + i * 40) % 100;
                    ctx.beginPath();
                    ctx.arc(x + w/2, y + h/2, w/2 + 15 + i * 6 - off * 0.15, 0, Math.PI*2);
                    ctx.stroke();
                }
            }

            if (player.magnetTime > 0) {
                ctx.strokeStyle = 'rgba(156,39,176,0.35)';
                ctx.lineWidth = 2;
                for (let i = 0; i < 2; i++) {
                    const r = (Date.now() / 12 + i * 120) % MAGNET_RADIUS;
                    ctx.globalAlpha = 1 - r / MAGNET_RADIUS;
                    ctx.beginPath();
                    ctx.arc(x + w/2, y + h/2, r, 0, Math.PI*2);
                    ctx.stroke();
                }
                ctx.globalAlpha = 1;
            }

            ctx.fillStyle = '#1E88E5';
            ctx.fillRect(x, y, w, h);
            ctx.strokeStyle = '#0D47A1';
            ctx.lineWidth = 2;
            ctx.strokeRect(x, y, w, h);

            ctx.fillStyle = '#fff';
            const eyeX = facing > 0 ? x + w - 14 : x + 6;
            ctx.fillRect(eyeX, y + 8, 8, 8);
            ctx.fillRect(eyeX + (facing > 0 ? 8 : -8), y + 8, 8, 8);
            ctx.fillStyle = '#000';
            ctx.fillRect(eyeX + (facing > 0 ? 2 : 0), y + 10, 4, 4);
            ctx.fillRect(eyeX + (facing > 0 ? 10 : -8), y + 10, 4, 4);

            ctx.fillStyle = '#fff';
            ctx.fillRect(x + w/2 - 6, y + h - 12, 12, 4);

            if (!player.onGround) {
                for (let i = 0; i < player.jumpsLeft; i++) {
                    ctx.fillStyle = '#FFD700';
                    ctx.beginPath();
                    ctx.arc(x + w/2 - 8 + i * 16, y - 12, 4, 0, Math.PI*2);
                    ctx.fill();
                }
            }
        }

        function drawEnemy(e) {
            if (player.slowTime > 0) {
                ctx.fillStyle = 'rgba(0,188,212,0.25)';
                ctx.beginPath();
                ctx.arc(e.x + e.w/2, e.y + e.h/2, e.w, 0, Math.PI*2);
                ctx.fill();
            }

            if (e.type === 'patrol') {
                ctx.fillStyle = '#8E24AA';
                ctx.fillRect(e.x, e.y, e.w, e.h);
                ctx.strokeStyle = '#4A148C'; ctx.lineWidth = 2;
                ctx.strokeRect(e.x, e.y, e.w, e.h);
                ctx.fillStyle = '#fff';
                ctx.fillRect(e.x + 5, e.y + 8, 7, 7);
                ctx.fillRect(e.x + e.w - 12, e.y + 8, 7, 7);
                ctx.fillStyle = '#000';
                ctx.fillRect(e.x + 7, e.y + 10, 3, 3);
                ctx.fillRect(e.x + e.w - 10, e.y + 10, 3, 3);
                ctx.fillStyle = '#fff';
                for (let i = 0; i < 3; i++) ctx.fillRect(e.x + 5 + i * 7, e.y + e.h - 10, 4, 5);
            } else {
                ctx.fillStyle = 'rgba(255, 100, 100, 0.85)';
                ctx.beginPath();
                ctx.arc(e.x + e.w/2, e.y + e.h/2 - 2, e.w/2, Math.PI, 0);
                ctx.lineTo(e.x + e.w, e.y + e.h);
                const seg = e.w / 3;
                for (let i = 0; i < 3; i++) {
                    const cx = e.x + e.w - seg * (i + 0.5);
                    const cy = e.y + e.h - 4;
                    ctx.arc(cx, cy, seg/2, 0, Math.PI);
                }
                ctx.lineTo(e.x, e.y + e.h);
                ctx.closePath(); ctx.fill();
                ctx.fillStyle = '#fff';
                ctx.beginPath(); ctx.arc(e.x + 10, e.y + 13, 4, 0, Math.PI*2); ctx.fill();
                ctx.beginPath(); ctx.arc(e.x + e.w - 10, e.y + 13, 4, 0, Math.PI*2); ctx.fill();
                ctx.fillStyle = '#000';
                ctx.beginPath(); ctx.arc(e.x + 10, e.y + 13, 2, 0, Math.PI*2); ctx.fill();
                ctx.beginPath(); ctx.arc(e.x + e.w - 10, e.y + 13, 2, 0, Math.PI*2); ctx.fill();
            }
        }

        function drawBoss() {
            if (!boss.active) return;
            const flash = boss.hurtTimer > 0 && Math.floor(boss.hurtTimer / 4) % 2 === 0;

            ctx.fillStyle = flash ? '#FFFFFF' : '#C62828';
            ctx.fillRect(boss.x, boss.y, boss.w, boss.h);
            ctx.strokeStyle = '#4A0000';
            ctx.lineWidth = 3;
            ctx.strokeRect(boss.x, boss.y, boss.w, boss.h);

            ctx.fillStyle = '#FFF';
            ctx.beginPath();
            ctx.arc(boss.x + 22, boss.y + 28, 10, 0, Math.PI*2);
            ctx.arc(boss.x + boss.w - 22, boss.y + 28, 10, 0, Math.PI*2);
            ctx.fill();
            ctx.fillStyle = '#000';
            ctx.beginPath();
            ctx.arc(boss.x + 22, boss.y + 30, 5, 0, Math.PI*2);
            ctx.arc(boss.x + boss.w - 22, boss.y + 30, 5, 0, Math.PI*2);
            ctx.fill();

            ctx.strokeStyle = '#000';
            ctx.lineWidth = 3;
            ctx.beginPath();
            ctx.moveTo(boss.x + 10, boss.y + 16);
            ctx.lineTo(boss.x + 34, boss.y + 24);
            ctx.moveTo(boss.x + boss.w - 10, boss.y + 16);
            ctx.lineTo(boss.x + boss.w - 34, boss.y + 24);
            ctx.stroke();

            ctx.fillStyle = '#000';
            ctx.fillRect(boss.x + 18, boss.y + boss.h - 22, boss.w - 36, 12);
            ctx.fillStyle = '#FFF';
            for (let i = 0; i < 4; i++) ctx.fillRect(boss.x + 22 + i * 10, boss.y + boss.h - 22, 5, 6);

            const barW = Math.min(400, W - 80);
            const barH = 20;
            const barX = W/2 - barW/2;
            const barY = 60;
            ctx.fillStyle = 'rgba(0,0,0,0.7)';
            ctx.fillRect(barX - 4, barY - 4, barW + 8, barH + 8);
            ctx.fillStyle = '#333';
            ctx.fillRect(barX, barY, barW, barH);
            const hpPct = boss.hp / boss.maxHp;
            ctx.fillStyle = hpPct > 0.5 ? '#4CAF50' : hpPct > 0.25 ? '#FFC107' : '#F44336';
            ctx.fillRect(barX, barY, barW * hpPct, barH);
            ctx.strokeStyle = '#FFF';
            ctx.lineWidth = 2;
            ctx.strokeRect(barX, barY, barW, barH);

            ctx.fillStyle = '#FFF';
            ctx.font = 'bold 16px Arial';
            ctx.textAlign = 'center';
            ctx.fillText('👹 ' + t('boss') + '  ' + boss.hp + '/' + boss.maxHp, W/2, barY - 10);
        }

        function drawBossProjectiles() {
            for (const p of bossProjectiles) {
                const g = ctx.createRadialGradient(p.x, p.y, 0, p.x, p.y, p.r * 2);
                g.addColorStop(0, 'rgba(255,100,0,0.9)');
                g.addColorStop(1, 'rgba(255,0,0,0)');
                ctx.fillStyle = g;
                ctx.beginPath();
                ctx.arc(p.x, p.y, p.r * 2, 0, Math.PI*2);
                ctx.fill();

                ctx.fillStyle = '#FF5722';
                ctx.beginPath();
                ctx.arc(p.x, p.y, p.r, 0, Math.PI*2);
                ctx.fill();
                ctx.fillStyle = '#FFEB3B';
                ctx.beginPath();
                ctx.arc(p.x - 2, p.y - 2, p.r * 0.5, 0, Math.PI*2);
                ctx.fill();
            }
        }

        // ============================================
        // СТАРТ
        // ============================================
        loadLevel(0);
        applyLocalization();

        document.addEventListener('visibilitychange', () => {
            if (document.hidden) {
                Sound.pauseAll();
            } else if (!paused) {
                Sound.resumeAll();
            }
        });

        function loop() {
            update();
            draw();
            requestAnimationFrame(loop);
        }
        loop();

        window.addEventListener('beforeunload', () => saveLocalBest(bestScore));
    </script>
</body>
</html>
