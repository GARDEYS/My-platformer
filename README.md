<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>Jumpix — платформер</title>
    <link rel="icon" type="image/svg+xml" href="favicon.svg">
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
        canvas {
            display: block;
            position: fixed;
            top: 50%; left: 50%;
            transform: translate(-50%, -50%);
            background: #87CEEB;
            image-rendering: pixelated;
            max-width: 100vw;
            max-height: 100vh;
            width: auto; height: auto;
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
        // ЗВУКОВОЙ ДВИЖОК
        // ============================================
        const Sound = (() => {
            let actx = null, masterGain = null, musicGain = null, sfxGain = null;
            let enabled = true, unlocked = false, musicTimer = null, musicPlaying = false;

            function init() {
                if (actx) return;
                try {
                    actx = new (window.AudioContext || window.webkitAudioContext)();
                    masterGain = actx.createGain(); masterGain.gain.value = 0.6; masterGain.connect(actx.destination);
                    musicGain = actx.createGain(); musicGain.gain.value = 0.15; musicGain.connect(masterGain);
                    sfxGain = actx.createGain(); sfxGain.gain.value = 0.5; sfxGain.connect(masterGain);
                } catch(e) {}
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
                    osc.connect(g); g.connect(dest || sfxGain);
                    osc.start(); osc.stop(actx.currentTime + duration + 0.05);
                } catch(e) {}
            }
            function noise(duration, vol = 0.3) {
                if (!enabled || !unlocked || !actx) return;
                try {
                    const bufferSize = actx.sampleRate * duration;
                    const buffer = actx.createBuffer(1, bufferSize, actx.sampleRate);
                    const data = buffer.getChannelData(0);
                    for (let i = 0; i < bufferSize; i++) data[i] = Math.random() * 2 - 1;
                    const src = actx.createBufferSource(); src.buffer = buffer;
                    const g = actx.createGain();
                    g.gain.setValueAtTime(vol, actx.currentTime);
                    g.gain.exponentialRampToValueAtTime(0.001, actx.currentTime + duration);
                    src.connect(g); g.connect(sfxGain); src.start();
                } catch(e) {}
            }
            function jump() { tone(520, 0.15, 'square', 0.4, null, 880); }
            function doubleJump() { tone(680, 0.18, 'square', 0.4, null, 1100); }
            function coin() { tone(880, 0.08, 'sine', 0.5); setTimeout(() => tone(1320, 0.12, 'sine', 0.5), 60); }
            function bonus() {
                tone(523, 0.1, 'triangle', 0.5);
                setTimeout(() => tone(659, 0.1, 'triangle', 0.5), 80);
                setTimeout(() => tone(784, 0.15, 'triangle', 0.5), 160);
            }
            function hurt() { noise(0.25, 0.4); tone(200, 0.3, 'sawtooth', 0.3, null, 80); }
            function stomp() {
                tone(300, 0.12, 'square', 0.5, null, 150);
                setTimeout(() => tone(600, 0.1, 'square', 0.4), 60);
            }
            function bossShoot() { tone(180, 0.2, 'sawtooth', 0.35, null, 90); }
            function victory() { [523, 659, 784, 1047].forEach((f, i) => setTimeout(() => tone(f, 0.2, 'triangle', 0.6), i * 120)); }
            function defeat() { [440, 349, 262, 196].forEach((f, i) => setTimeout(() => tone(f, 0.25, 'sawtooth', 0.5), i * 150)); }

            const melody = [262, 330, 392, 330, 262, 330, 392, 523, 440, 392, 330, 262, 294, 330, 262, 220];
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
                musicPlaying = true; melodyIdx = 0;
                playMusicNote();
                musicTimer = setInterval(playMusicNote, 380);
            }
            function stopMusic() { musicPlaying = false; if (musicTimer) { clearInterval(musicTimer); musicTimer = null; } }
            function toggle() { enabled = !enabled; if (masterGain) masterGain.gain.value = enabled ? 0.6 : 0; return enabled; }
            function pauseAll() { if (masterGain) masterGain.gain.value = 0; }
            function resumeAll() { if (masterGain) masterGain.gain.value = enabled ? 0.6 : 0; }

            return { init, unlock, toggle, pauseAll, resumeAll, jump, doubleJump, coin, bonus, hurt, stomp, bossShoot, victory, defeat, startMusic, stopMusic };
        })();

        function firstInteraction() {
            Sound.unlock(); Sound.startMusic();
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
                hint: '🔼×2 двойной прыжок · 🧲 магнит · 🐢 замедление · 👹 босс на 3-м уровне',
                boss: 'БОСС', bossDefeated: 'БОСС ПОБЕЖДЁН!',
                gameOver: 'ИГРА ОКОНЧЕНА', restart: 'Перезапуск...',
                shield: 'Щит', speed: 'Ускорение', heart: 'Жизнь', magnet: 'Магнит', slow: 'Замедление',
                phrases: ['Не сдавайся!', 'Попробуй ещё!', 'Ты близко!', 'Ещё один шанс!', 'Всё получится!', 'Не останавливайся!', 'Ты сможешь!', 'Почти получилось!', 'Попробуй снова!', 'Так держать!'],
                statsTitle: 'СТАТИСТИКА',
                statsCoins: 'Монет собрано', statsScore: 'Очков набрано',
                statsDeaths: 'Смертей', statsTime: 'Время', statsLevel: 'Уровень',
                statsRecord: 'Рекорд', newRecord: 'НОВЫЙ РЕКОРД!'
            },
            en: {
                leaders: 'Leaders', records: 'Records', loading: 'Loading...', close: 'Close',
                noRecords: 'No records yet', lbUnavailable: 'Leaderboard unavailable',
                lbError: 'Loading error', player: 'Player', level: 'Level',
                passed: 'PASSED!', allPassed: 'ALL LEVELS PASSED!',
                score: 'Score', record: 'Record', nextLevel: 'Loading next level...',
                hint: '🔼×2 double jump · 🧲 magnet · 🐢 slow-mo · 👹 boss on level 3',
                boss: 'BOSS', bossDefeated: 'BOSS DEFEATED!',
                gameOver: 'GAME OVER', restart: 'Restarting...',
                shield: 'Shield', speed: 'Speed', heart: 'Life', magnet: 'Magnet', slow: 'Slow-mo',
                phrases: ["Don't give up!", 'Try again!', "You're close!", 'One more chance!', 'You can do it!', 'Keep going!', 'Almost there!', 'Try once more!', "Don't stop now!", 'You got this!'],
                statsTitle: 'STATS',
                statsCoins: 'Coins collected', statsScore: 'Score',
                statsDeaths: 'Deaths', statsTime: 'Time', statsLevel: 'Level',
                statsRecord: 'Record', newRecord: 'NEW RECORD!'
            }
        };
        let currentLang = 'ru';
        function t(key) { return i18n[currentLang][key] || i18n.ru[key] || key; }
        function applyLocalization() {
            document.querySelectorAll('[data-i18n]').forEach(el => {
                el.textContent = t(el.getAttribute('data-i18n'));
            });
        }

        // ============================================
        // YANDEX SDK
        // ============================================
        const LB_NAME = 'bestScore';
        let ysdk = null, ysdkPlayer = null, ysdkLB = null;
        let bestScore = 0, canSave = false, paused = false;

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
                paused = true; Sound.pauseAll();
                if (ysdk && ysdk.features && ysdk.features.GameplayAPI) { try { ysdk.features.GameplayAPI.stop(); } catch(e) {} }
            };
            const handleResume = () => {
                paused = false; Sound.resumeAll();
                if (ysdk && ysdk.features && ysdk.features.GameplayAPI) { try { ysdk.features.GameplayAPI.start(); } catch(e) {} }
            };
            try { ysdk.on('game_api_pause', handlePause); ysdk.on('game_api_resume', handleResume); } catch(e) {}
            try { ysdk.features.LoadingAPI.ready(); console.log('✅ Game Ready'); } catch(e) {}
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
        // КАНВАС
        // ============================================
        const canvas = document.getElementById('game');
        const ctx = canvas.getContext('2d');
        const W = 960, H = 540;
        canvas.width = W; canvas.height = H;

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
            // ========== УРОВЕНЬ 1 ==========
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
                    { x: 420, y: 380 }, { x: 620, y: 310 }, { x: 820, y: 240 },
                    { x: 540, y: 160 }, { x: 290, y: 110 },
                ],
                bonuses: [
                    { type: 'shield', x: 340, y: 340 },
                    { type: 'speed', x: 700, y: 220 },
                    { type: 'heart', x: 180, y: 260 },
                    { type: 'magnet', x: 460, y: 460 },
                    { type: 'slow', x: 850, y: 130 },
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
            // ========== УРОВЕНЬ 2 ==========
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
                    { x: 300, y: 430 }, { x: 470, y: 380 }, { x: 640, y: 330 },
                    { x: 860, y: 280 }, { x: 460, y: 140 }, { x: 250, y: 90 },
                ],
                bonuses: [
                    { type: 'magnet', x: 430, y: 380 },
                    { type: 'slow', x: 820, y: 280 },
                    { type: 'shield', x: 470, y: 140 },
                    { type: 'speed', x: 130, y: 250 },
                    { type: 'heart', x: 700, y: 470 },
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
                movingPlatforms: [
                    { x: 180, y: 470, w: 80, h: 18, axis: 'x', startX: 180, endX: 380, speed: 0.8, dir: 1, dx: 0, dy: 0 },
                    { x: 720, y: 380, w: 90, h: 18, axis: 'y', startY: 380, endY: 200, speed: 1.0, dir: 1, dx: 0, dy: 0 },
                    { x: 640, y: 280, w: 80, h: 18, axis: 'x', startX: 640, endX: 860, speed: 1.2, dir: 1, dx: 0, dy: 0 },
                    { x: 220, y: 220, w: 90, h: 18, axis: 'y', startY: 220, endY: 100, speed: 0.6, dir: 1, dx: 0, dy: 0 },
                ],
                disappearingPlatforms: [
                    { x: 350, y: 380, w: 70, h: 18, phase: 'visible', timer: 180, visibleTime: 180, blinkTime: 60, hiddenTime: 120, appearTime: 30, appearance: 1 },
                    { x: 560, y: 200, w: 70, h: 18, phase: 'visible', timer: 240, visibleTime: 240, blinkTime: 60, hiddenTime: 150, appearTime: 30, appearance: 1 },
                    { x: 120, y: 200, w: 70, h: 18, phase: 'hidden', timer: 60, visibleTime: 200, blinkTime: 60, hiddenTime: 90, appearTime: 30, appearance: 0 },
                ],
                flag: { x: 90, y: 20, w: 30, h: 60 },
            },
            // ========== УРОВЕНЬ 3 ==========
            {
                name: 'Уровень 3',
                spawn: { x: 60, y: 400 },
                platforms: [
                    { x: 0, y: 480, w: 220, h: 60 },
                    { x: 280, y: 440, w: 100, h: 20 },
                    { x: 440, y: 400, w: 100, h: 20 },
                    { x: 600, y: 360, w: 100, h: 20 },
                    { x: 760, y: 320, w: 200, h: 20 },
                    { x: 640, y: 240, w: 120, h: 20 },
                    { x: 460, y: 180, w: 120, h: 20 },
                    { x: 260, y: 140, w: 120, h: 20 },
                    { x: 0, y: 100, w: 180, h: 20 },
                    { x: 380, y: 520, w: 580, h: 20 },
                ],
                coins: [
                    { x: 330, y: 400 }, { x: 490, y: 360 }, { x: 650, y: 320 },
                    { x: 860, y: 280 }, { x: 700, y: 200 }, { x: 520, y: 140 }, { x: 320, y: 100 },
                ],
                bonuses: [
                    { type: 'magnet', x: 500, y: 440 },
                    { type: 'slow', x: 880, y: 240 },
                    { type: 'shield', x: 520, y: 100 },
                    { type: 'speed', x: 180, y: 420 },
                    { type: 'heart', x: 700, y: 440 },
                ],
                enemies: [
                    { type: 'chaser', x: 280, y: 410, w: 30, h: 30, platX: 280, platW: 100, dir: 1, speed: 2.0, baseSpeed: 0.9, chaseRadius: 240 },
                    { type: 'chaser', x: 760, y: 290, w: 30, h: 30, platX: 760, platW: 200, dir: -1, speed: 2.2, baseSpeed: 1.1, chaseRadius: 280 },
                    { type: 'jumper', x: 440, y: 490, w: 28, h: 30, baseY: 490, vy: 0, jumpTimer: 80, jumpInterval: 90 },
                    { type: 'jumper', x: 700, y: 490, w: 28, h: 30, baseY: 490, vy: 0, jumpTimer: 30, jumpInterval: 100 },
                    { type: 'shooter', x: 460, y: 140, w: 32, h: 40, shootTimer: 60, shootInterval: 120 },
                    { type: 'shooter', x: 260, y: 100, w: 32, h: 40, shootTimer: 30, shootInterval: 100 },
                    { type: 'shooter', x: 820, y: 280, w: 32, h: 40, shootTimer: 90, shootInterval: 140 },
                    { type: 'bat', x: 400, y: 60, w: 26, h: 20, baseX: 400, baseY: 60, dirX: 1, dirY: 0.6, speed: 2.6, range: 180, wingPhase: 0 },
                    { type: 'bat', x: 700, y: 100, w: 26, h: 20, baseX: 700, baseY: 100, dirX: -1, dirY: 0.7, speed: 3.0, range: 200, wingPhase: 1 },
                    { type: 'bat', x: 550, y: 80, w: 26, h: 20, baseX: 550, baseY: 80, dirX: 1, dirY: -0.5, speed: 2.4, range: 150, wingPhase: 2 },
                ],
                disappearingPlatforms: [
                    { x: 400, y: 340, w: 70, h: 18, phase: 'visible', timer: 200, visibleTime: 200, blinkTime: 50, hiddenTime: 100, appearTime: 30, appearance: 1 },
                    { x: 720, y: 220, w: 70, h: 18, phase: 'blinking', timer: 40, visibleTime: 220, blinkTime: 50, hiddenTime: 120, appearTime: 30, appearance: 1 },
                    { x: 200, y: 260, w: 70, h: 18, phase: 'visible', timer: 260, visibleTime: 260, blinkTime: 60, hiddenTime: 130, appearTime: 30, appearance: 1 },
                ],
                lava: [
                    { x: 220, y: 540, w: 160, h: 20 },
                ],
                flag: { x: 30, y: 40, w: 30, h: 60 },
            },
        ];

        // ============================================
        // ПЕРЕМЕННЫЕ
        // ============================================
        let currentLevel = 0;
        let platforms = [], coins = [], bonuses = [], enemies = [], flag = null, SPAWN = null;
        let lava = [], movingPlatforms = [], disappearingPlatforms = [];
        let TOTAL_COINS = 0;
        let particles = [];
        let score = 0, lives = 3;
        let win = false, winText = '';
        let hurtFlash = 0, jumpEdge = false;

        let attemptCoins = 0, attemptStartTime = Date.now(), attemptDeaths = 0;

        // Игрок
        const player = {
            x: 0, y: 0, w: 32, h: 42,
            vx: 0, vy: 0, onGround: false, facing: 1,
            invuln: 0, jumpsLeft: MAX_JUMPS,
            shieldTime: 0, speedTime: 0,
            magnetTime: 0, slowTime: 0, animFrame: 0
        };

        // Босс
        const boss = {
            x: 0, y: 0, w: 80, h: 80,
            hp: 5, maxHp: 5, vx: 1.8, invuln: 0,
            active: false, shootTimer: 120, hurtTimer: 0
        };
        let bossProjectiles = [], bossDefeated = false, enemyProjectiles = [];

        // Праздник после босса
        let celebration = {
            active: false, timer: 0, duration: 300,
            fireworks: [], confetti: [], fireworkTimer: 0,
            textScale: 1, textAlpha: 0,
            lightRays: [], lightRayAngle: 0, lightRaysTimer: 0,
            bossDebris: []
        };

        // Проигрыш
        let gameOver = {
            active: false, timer: 0, duration: 210,
            debris: [], textAlpha: 0, textScale: 0.3,
            shakeX: 0, shakeY: 0, flashAlpha: 0, darkAlpha: 0,
            phrase: '', phraseAlpha: 0, phraseScale: 0.5,
            phraseTimer: 0, phraseDuration: 90, phraseIndex: 0,
            stats: null, statsAlpha: 0, statsSlide: 0,
            isNewRecord: false, recordGlow: 0
        };

        // Фоновые эффекты
        let bgEffects = {
            clouds: [], birds: [], petals: [], stars: [],
            sunRays: 0, fogOffset: 0,
            rainbow: { active: false, alpha: 0, timer: 0, duration: 480, nextSpawn: 600 },
            sunset: { progress: 0, speed: 0.0004, glowPulse: 0 },
            storm: { state: 'clear', intensity: 0, timer: 1800, rainDrops: [], lightning: 0, lightningTimer: 0 },
            fireworks: [], fireworkTimer: 600
        };

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

        function spawnSkyFirework(x, y) {
            const colors = ['#FFD700', '#FF5252', '#4FC3F7', '#69F0AE', '#E040FB', '#FFAB40', '#FFFFFF', '#FF8A80'];
            const color = colors[Math.floor(Math.random() * colors.length)];
            const isDouble = Math.random() < 0.4;
            const count = 24;
            for (let i = 0; i < count; i++) {
                const angle = (Math.PI * 2 / count) * i + Math.random() * 0.15;
                const speed = 1.8 + Math.random() * 2.5;
                bgEffects.fireworks.push({
                    x, y, vx: Math.cos(angle) * speed, vy: Math.sin(angle) * speed,
                    life: 70, maxLife: 70, color, size: 2 + Math.random() * 2
                });
            }
            if (isDouble) {
                setTimeout(() => {
                    const color2 = colors[Math.floor(Math.random() * colors.length)];
                    for (let i = 0; i < count; i++) {
                        const angle = (Math.PI * 2 / count) * i + Math.random() * 0.15;
                        const speed = 1.5 + Math.random() * 2.5;
                        bgEffects.fireworks.push({
                            x: x + (Math.random() - 0.5) * 60, y: y - 40,
                            vx: Math.cos(angle) * speed, vy: Math.sin(angle) * speed,
                            life: 70, maxLife: 70, color: color2, size: 2 + Math.random() * 2
                        });
                    }
                }, 250);
            }
            try { Sound.coin(); } catch(e) {}
        }

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
            lava = (L.lava || []).map(l => ({ ...l }));
            movingPlatforms = (L.movingPlatforms || []).map(m => ({ ...m }));
            disappearingPlatforms = (L.disappearingPlatforms || []).map(dp => ({ ...dp }));
            TOTAL_COINS = coins.length;

            player.x = SPAWN.x; player.y = SPAWN.y;
            player.vx = 0; player.vy = 0;
            player.invuln = 60;
            player.jumpsLeft = MAX_JUMPS;
            player.shieldTime = 0; player.speedTime = 0;
            player.magnetTime = 0; player.slowTime = 0;
            particles = [];

            boss.active = false; boss.hp = boss.maxHp;
            boss.invuln = 0; boss.hurtTimer = 0; boss.shootTimer = 120;
            bossProjectiles = []; enemyProjectiles = []; bossDefeated = false;

            celebration.active = false; celebration.timer = 0;
            celebration.fireworks = []; celebration.confetti = [];
            celebration.fireworkTimer = 0; celebration.textScale = 0.3;
            celebration.textAlpha = 0;
            celebration.lightRays = []; celebration.lightRayAngle = 0;
            celebration.lightRaysTimer = 0; celebration.bossDebris = [];

            gameOver.active = false; gameOver.timer = 0;
            gameOver.debris = []; gameOver.textAlpha = 0;
            gameOver.textScale = 0.3; gameOver.shakeX = 0; gameOver.shakeY = 0;
            gameOver.flashAlpha = 0; gameOver.darkAlpha = 0;
            gameOver.phrase = ''; gameOver.phraseAlpha = 0;
            gameOver.phraseScale = 0.5; gameOver.phraseTimer = 0;
            gameOver.phraseIndex = 0;
            gameOver.stats = null; gameOver.statsAlpha = 0;
            gameOver.statsSlide = 40;
            gameOver.isNewRecord = false; gameOver.recordGlow = 0;

            // Фоновые эффекты
            bgEffects.clouds = []; bgEffects.birds = []; bgEffects.petals = [];
            bgEffects.stars = []; bgEffects.sunRays = 0; bgEffects.fogOffset = 0;
            bgEffects.rainbow = { active: false, alpha: 0, timer: 0, duration: 480, nextSpawn: 600 + Math.random() * 1200 };
            bgEffects.sunset = { progress: 0, speed: 0.0004, glowPulse: 0 };
            bgEffects.storm = { state: 'clear', intensity: 0, timer: 1800, rainDrops: [], lightning: 0, lightningTimer: 0 };
            bgEffects.fireworks = [];
            bgEffects.fireworkTimer = 600 + Math.random() * 600;

            if (idx === 0) {
                for (let i = 0; i < 6; i++) {
                    bgEffects.clouds.push({
                        x: Math.random() * W, y: 40 + Math.random() * 130,
                        w: 60 + Math.random() * 80, h: 20 + Math.random() * 20,
                        speed: 0.15 + Math.random() * 0.35, alpha: 0.55 + Math.random() * 0.35
                    });
                }
                for (let i = 0; i < 12; i++) {
                    bgEffects.petals.push({
                        x: Math.random() * W, y: Math.random() * H,
                        vy: 0.4 + Math.random() * 0.6, vx: (Math.random() - 0.5) * 0.6,
                        rotation: Math.random() * Math.PI * 2,
                        rotSpeed: (Math.random() - 0.5) * 0.05,
                        size: 5 + Math.random() * 4,
                        color: ['#FF8A80', '#FFAB91', '#F8BBD0', '#FFCCBC'][Math.floor(Math.random() * 4)]
                    });
                }
            } else if (idx === 1) {
                for (let i = 0; i < 3; i++) {
                    bgEffects.birds.push({
                        x: Math.random() * W, y: 60 + Math.random() * 100,
                        speed: 0.6 + Math.random() * 0.5,
                        flapPhase: Math.random() * Math.PI * 2,
                        size: 12 + Math.random() * 6
                    });
                }
                for (let i = 0; i < 40; i++) {
                    bgEffects.stars.push({
                        x: Math.random() * W, y: Math.random() * 200,
                        size: 1 + Math.random() * 1.5,
                        phase: Math.random() * Math.PI * 2,
                        speed: 0.02 + Math.random() * 0.03
                    });
                }
            }
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
        // УРОН / ПОБЕДА
        // ============================================
        function hitPlayer() {
            if (win || paused || gameOver.active) return;
            if (player.shieldTime > 0) {
                player.shieldTime = 0;
                player.invuln = 60;
                Sound.hurt();
                spawnParticles(player.x + player.w/2, player.y + player.h/2, '#2196F3', 20);
                return;
            }
            if (player.invuln > 0) return;
            lives--;
            attemptDeaths++;
            hurtFlash = 20;
            Sound.hurt();
            spawnParticles(player.x + player.w/2, player.y + player.h/2, '#E53935', 15);
            if (lives <= 0) {
                Sound.defeat();
                startGameOver();
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
                    attemptCoins = 0;
                    attemptStartTime = Date.now();
                    attemptDeaths = 0;
                    loadLevel(0);
                    win = false;
                }, 4000);
            }
        }

        // ============================================
        // БОСС
        // ============================================
        function spawnBoss() {
            boss.active = true; boss.hp = boss.maxHp;
            boss.x = W/2 - boss.w/2; boss.y = 60;
            boss.vx = 1.8; boss.shootTimer = 90;
            boss.invuln = 60; boss.hurtTimer = 0;
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
                const pcx = player.x + player.w/2, pcy = player.y + player.h/2;
                const bcx = boss.x + boss.w/2, bcy = boss.y + boss.h/2;
                const dx = pcx - bcx, dy = pcy - bcy;
                const dist = Math.sqrt(dx*dx + dy*dy) || 1;
                const speed = 4.5;
                bossProjectiles.push({ x: bcx, y: bcy, vx: (dx/dist)*speed, vy: (dy/dist)*speed, r: 12, life: 240 });
            }
            if (rectsCollide(player, boss)) {
                const stomp = player.vy > 0 && (player.y + player.h) < (boss.y + boss.h * 0.6);
                if (stomp) {
                    if (boss.invuln <= 0) {
                        boss.hp--; boss.invuln = 40; boss.hurtTimer = 20;
                        player.vy = -11;
                        Sound.stomp();
                        spawnParticles(boss.x + boss.w/2, boss.y, '#FFD700', 25);
                        if (boss.hp <= 0) {
                            bossDefeated = true; boss.active = false;
                            score += 100;
                            Sound.victory();
                            spawnParticles(boss.x + boss.w/2, boss.y + boss.h/2, '#FF5722', 60);
                            startCelebration();
                            try { if (ysdk) ysdk.adv.showFullscreenAdv(); } catch(e) {}
                        }
                    } else { player.vy = -11; }
                } else { hitPlayer(); }
            }
        }

        function updateBossProjectiles() {
            for (let i = bossProjectiles.length - 1; i >= 0; i--) {
                const p = bossProjectiles[i];
                p.x += p.vx; p.y += p.vy; p.life--;
                if (p.life <= 0 || p.x < -50 || p.x > W + 50 || p.y < -50 || p.y > H + 50) {
                    bossProjectiles.splice(i, 1); continue;
                }
                const dx = p.x - (player.x + player.w/2);
                const dy = p.y - (player.y + player.h/2);
                if (Math.sqrt(dx*dx + dy*dy) < p.r + 18) {
                    if (player.shieldTime > 0) {
                        player.shieldTime = 0; player.invuln = 60;
                        Sound.hurt(); spawnParticles(p.x, p.y, '#2196F3', 15);
                    } else { hitPlayer(); }
                    bossProjectiles.splice(i, 1);
                }
            }
        }

        function updateEnemyProjectiles() {
            for (let i = enemyProjectiles.length - 1; i >= 0; i--) {
                const p = enemyProjectiles[i];
                p.x += p.vx; p.y += p.vy; p.life--;
                if (p.life <= 0 || p.x < -50 || p.x > W + 50 || p.y < -50 || p.y > H + 50) {
                    enemyProjectiles.splice(i, 1); continue;
                }
                const dx = p.x - (player.x + player.w/2);
                const dy = p.y - (player.y + player.h/2);
                if (Math.sqrt(dx*dx + dy*dy) < p.r + 18) {
                    if (player.shieldTime > 0) {
                        player.shieldTime = 0; player.invuln = 60;
                        Sound.hurt(); spawnParticles(p.x, p.y, '#2196F3', 15);
                    } else { hitPlayer(); }
                    enemyProjectiles.splice(i, 1);
                }
            }
        }

        // ============================================
        // ПРАЗДНИК (после босса)
        // ============================================
        function startCelebration() {
            celebration.active = true;
            celebration.timer = celebration.duration;
            celebration.fireworks = []; celebration.confetti = [];
            celebration.fireworkTimer = 0;
            celebration.textScale = 0.3; celebration.textAlpha = 0;
            const cx = boss.x + boss.w / 2, cy = boss.y + boss.h / 2;
            for (let i = 0; i < 24; i++) {
                const angle = (Math.PI * 2 / 24) * i + Math.random() * 0.3;
                const speed = 4 + Math.random() * 6;
                celebration.bossDebris.push({
                    x: cx + (Math.random() - 0.5) * 30, y: cy + (Math.random() - 0.5) * 30,
                    w: 12 + Math.random() * 20, h: 12 + Math.random() * 20,
                    vx: Math.cos(angle) * speed, vy: Math.sin(angle) * speed - 2,
                    rotation: Math.random() * Math.PI * 2, rotSpeed: (Math.random() - 0.5) * 0.4,
                    color: Math.random() < 0.5 ? '#C62828' : '#8E0000',
                    life: 90, maxLife: 90
                });
            }
            for (let i = 0; i < 6; i++) {
                const angle = Math.random() * Math.PI * 2;
                const speed = 3 + Math.random() * 4;
                celebration.bossDebris.push({
                    x: cx + (Math.random() - 0.5) * 60, y: cy + (Math.random() - 0.5) * 60,
                    w: 20 + Math.random() * 15, h: 20 + Math.random() * 15,
                    vx: Math.cos(angle) * speed, vy: Math.sin(angle) * speed - 3,
                    rotation: Math.random() * Math.PI * 2, rotSpeed: (Math.random() - 0.5) * 0.5,
                    color: '#4A0000', life: 120, maxLife: 120
                });
            }
            celebration.lightRayAngle = 0; celebration.lightRaysTimer = 0;
            for (let i = 0; i < 8; i++) {
                celebration.lightRays.push({
                    angle: (Math.PI * 2 / 8) * i,
                    length: 0, maxLength: 300 + Math.random() * 200,
                    width: 20 + Math.random() * 20,
                    life: 60, maxLife: 60, speed: 15 + Math.random() * 10
                });
            }
            for (let i = 0; i < 120; i++) {
                celebration.confetti.push({
                    x: Math.random() * W, y: -20 - Math.random() * 200,
                    vx: (Math.random() - 0.5) * 3, vy: 1 + Math.random() * 3,
                    w: 6 + Math.random() * 8, h: 10 + Math.random() * 12,
                    rotation: Math.random() * Math.PI * 2,
                    rotSpeed: (Math.random() - 0.5) * 0.2,
                    color: ['#FFD700', '#FF5252', '#4FC3F7', '#69F0AE', '#E040FB', '#FFAB40', '#FFFFFF'][Math.floor(Math.random() * 7)],
                    life: 240, maxLife: 240
                });
            }
        }

        function spawnFirework(x, y, color) {
            const count = 30;
            for (let i = 0; i < count; i++) {
                const angle = (Math.PI * 2 / count) * i + Math.random() * 0.1;
                const speed = 2 + Math.random() * 3;
                celebration.fireworks.push({
                    x, y, vx: Math.cos(angle) * speed, vy: Math.sin(angle) * speed,
                    life: 60, maxLife: 60, color, size: 3 + Math.random() * 3
                });
            }
        }

        function updateCelebration() {
            if (!celebration.active) return;
            celebration.timer--;
            if (celebration.textScale < 1) { celebration.textScale += 0.05; if (celebration.textScale > 1) celebration.textScale = 1; }
            if (celebration.textAlpha < 1) { celebration.textAlpha += 0.03; if (celebration.textAlpha > 1) celebration.textAlpha = 1; }
            celebration.lightRayAngle += 0.02;
            for (let i = celebration.lightRays.length - 1; i >= 0; i--) {
                const r = celebration.lightRays[i];
                r.length += r.speed; r.life--;
                if (r.life <= 0) celebration.lightRays.splice(i, 1);
            }
            celebration.lightRaysTimer--;
            if (celebration.lightRaysTimer <= 0 && celebration.timer > 120) {
                celebration.lightRaysTimer = 100;
                const rayCount = 6 + Math.floor(Math.random() * 4);
                const offsetAngle = Math.random() * Math.PI * 2;
                for (let i = 0; i < rayCount; i++) {
                    celebration.lightRays.push({
                        angle: offsetAngle + (Math.PI * 2 / rayCount) * i,
                        length: 0, maxLength: 250 + Math.random() * 250,
                        width: 15 + Math.random() * 25,
                        life: 55, maxLife: 55, speed: 14 + Math.random() * 12
                    });
                }
            }
            for (let i = celebration.bossDebris.length - 1; i >= 0; i--) {
                const d = celebration.bossDebris[i];
                d.x += d.vx; d.y += d.vy;
                d.vy += 0.25; d.vx *= 0.98;
                d.rotation += d.rotSpeed; d.life--;
                if (d.y > H - 40 && d.vy > 0) { d.y = H - 40; d.vy *= -0.4; d.vx *= 0.7; }
                if (d.life <= 0) celebration.bossDebris.splice(i, 1);
            }
            celebration.fireworkTimer--;
            if (celebration.fireworkTimer <= 0 && celebration.timer > 60) {
                celebration.fireworkTimer = 25 + Math.random() * 20;
                const colors = ['#FFD700', '#FF5252', '#4FC3F7', '#69F0AE', '#E040FB', '#FFAB40'];
                spawnFirework(100 + Math.random() * (W - 200), 80 + Math.random() * (H / 2), colors[Math.floor(Math.random() * colors.length)]);
                Sound.coin();
            }
            for (let i = celebration.fireworks.length - 1; i >= 0; i--) {
                const f = celebration.fireworks[i];
                f.x += f.vx; f.y += f.vy;
                f.vy += 0.12; f.vx *= 0.98; f.life--;
                if (f.life <= 0) celebration.fireworks.splice(i, 1);
            }
            for (let i = celebration.confetti.length - 1; i >= 0; i--) {
                const c = celebration.confetti[i];
                c.x += c.vx + Math.sin((Date.now() / 300) + i) * 0.5;
                c.y += c.vy; c.rotation += c.rotSpeed; c.life--;
                if (c.life <= 0 || c.y > H + 40) celebration.confetti.splice(i, 1);
            }
            if (celebration.timer <= 0) celebration.active = false;
        }

        // ============================================
        // ПРОИГРЫШ
        // ============================================
        function startGameOver() {
            gameOver.active = true;
            gameOver.timer = gameOver.duration;
            gameOver.debris = [];
            gameOver.textAlpha = 0; gameOver.textScale = 0.3;
            gameOver.flashAlpha = 1; gameOver.darkAlpha = 0;

            const phrases = i18n[currentLang].phrases || i18n.ru.phrases;
            gameOver.phrase = phrases[Math.floor(Math.random() * phrases.length)];
            gameOver.phraseAlpha = 0; gameOver.phraseScale = 0.5;
            gameOver.phraseTimer = 0; gameOver.phraseIndex = 0;

            const elapsed = Math.floor((Date.now() - attemptStartTime) / 1000);
            const mins = Math.floor(elapsed / 60);
            const secs = elapsed % 60;
            gameOver.stats = {
                coins: attemptCoins, score: score, deaths: attemptDeaths,
                time: mins + ':' + (secs < 10 ? '0' : '') + secs,
                level: (currentLevel + 1) + '/' + LEVELS.length
            };
            gameOver.statsAlpha = 0; gameOver.statsSlide = 40;

            gameOver.isNewRecord = score > bestScore && score > 0;
            gameOver.recordGlow = 0;
            if (gameOver.isNewRecord) {
                bestScore = score;
                saveProgress();
                submitLeaderboard(score);
                setTimeout(() => Sound.bonus(), 400);
            }

            const cx = player.x + player.w / 2, cy = player.y + player.h / 2;
            for (let i = 0; i < 28; i++) {
                const angle = (Math.PI * 2 / 28) * i + Math.random() * 0.4;
                const speed = 4 + Math.random() * 6;
                gameOver.debris.push({
                    x: cx + (Math.random() - 0.5) * 20, y: cy + (Math.random() - 0.5) * 20,
                    w: 8 + Math.random() * 14, h: 8 + Math.random() * 14,
                    vx: Math.cos(angle) * speed, vy: Math.sin(angle) * speed - 3,
                    rotation: Math.random() * Math.PI * 2, rotSpeed: (Math.random() - 0.5) * 0.5,
                    color: Math.random() < 0.6 ? '#1E88E5' : '#0D47A1',
                    life: 120, maxLife: 120
                });
            }
            for (let i = 0; i < 4; i++) {
                const angle = Math.random() * Math.PI * 2;
                gameOver.debris.push({
                    x: cx, y: cy, w: 8, h: 8,
                    vx: Math.cos(angle) * (5 + Math.random() * 3),
                    vy: Math.sin(angle) * (5 + Math.random() * 3) - 4,
                    rotation: 0, rotSpeed: (Math.random() - 0.5) * 0.6,
                    color: '#FFFFFF', life: 130, maxLife: 130
                });
            }
            for (let i = 0; i < 20; i++) spawnParticles(cx, cy, '#FF5252', 1);
        }

        function updateGameOver() {
            if (!gameOver.active) return;
            gameOver.timer--;
            if (gameOver.flashAlpha > 0) { gameOver.flashAlpha -= 0.07; if (gameOver.flashAlpha < 0) gameOver.flashAlpha = 0; }
            if (gameOver.darkAlpha < 0.75 && gameOver.timer < gameOver.duration - 20) {
                gameOver.darkAlpha += 0.025; if (gameOver.darkAlpha > 0.75) gameOver.darkAlpha = 0.75;
            }
            const shakeStrength = Math.max(0, (gameOver.timer / gameOver.duration) * 12);
            gameOver.shakeX = (Math.random() - 0.5) * shakeStrength;
            gameOver.shakeY = (Math.random() - 0.5) * shakeStrength;
            if (gameOver.timer < gameOver.duration - 30) {
                if (gameOver.textAlpha < 1) { gameOver.textAlpha += 0.05; if (gameOver.textAlpha > 1) gameOver.textAlpha = 1; }
                if (gameOver.textScale < 1) { gameOver.textScale += 0.06; if (gameOver.textScale > 1) gameOver.textScale = 1; }
            }
            if (gameOver.timer < gameOver.duration - 60) {
                if (gameOver.phraseAlpha < 1) { gameOver.phraseAlpha += 0.06; if (gameOver.phraseAlpha > 1) gameOver.phraseAlpha = 1; }
                if (gameOver.phraseScale < 1) { gameOver.phraseScale += 0.08; if (gameOver.phraseScale > 1) gameOver.phraseScale = 1; }
                gameOver.phraseTimer++;
                if (gameOver.phraseTimer >= gameOver.phraseDuration) {
                    gameOver.phraseTimer = 0;
                    const phrases = i18n[currentLang].phrases || i18n.ru.phrases;
                    gameOver.phraseIndex = (gameOver.phraseIndex + 1) % phrases.length;
                    gameOver.phrase = phrases[gameOver.phraseIndex];
                    gameOver.phraseAlpha = 0; gameOver.phraseScale = 0.6;
                }
            }
            if (gameOver.timer < gameOver.duration - 90) {
                if (gameOver.statsAlpha < 1) { gameOver.statsAlpha += 0.05; if (gameOver.statsAlpha > 1) gameOver.statsAlpha = 1; }
                if (gameOver.statsSlide > 0) { gameOver.statsSlide -= 2; if (gameOver.statsSlide < 0) gameOver.statsSlide = 0; }
            }
            if (gameOver.isNewRecord && gameOver.statsAlpha > 0.5) gameOver.recordGlow += 0.1;
            for (let i = gameOver.debris.length - 1; i >= 0; i--) {
                const d = gameOver.debris[i];
                d.x += d.vx; d.y += d.vy;
                d.vy += 0.28; d.vx *= 0.98;
                d.rotation += d.rotSpeed; d.life--;
                if (d.y > H - 40 && d.vy > 0) { d.y = H - 40; d.vy *= -0.35; d.vx *= 0.7; }
                if (d.life <= 0) gameOver.debris.splice(i, 1);
            }
            if (gameOver.timer <= 0) {
                gameOver.active = false;
                lives = 3; score = 0;
                attemptCoins = 0;
                attemptStartTime = Date.now();
                attemptDeaths = 0;
                loadLevel(currentLevel);
            }
        }

        // ============================================
        // ОБНОВЛЕНИЕ
        // ============================================
        function update() {
            if (paused) return;
            if (gameOver.active) { updateGameOver(); return; }

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

            // Фоновые эффекты
            for (const c of bgEffects.clouds) {
                c.x += c.speed;
                if (c.x > W + c.w) { c.x = -c.w; c.y = 40 + Math.random() * 130; }
            }
            for (const b of bgEffects.birds) {
                b.x += b.speed; b.flapPhase += 0.25;
                if (b.x > W + 60) { b.x = -60; b.y = 60 + Math.random() * 120; }
            }
            for (const p of bgEffects.petals) {
                p.y += p.vy; p.x += p.vx + Math.sin(Date.now() / 800 + p.y) * 0.4;
                p.rotation += p.rotSpeed;
                if (p.y > H + 20) { p.y = -20; p.x = Math.random() * W; }
            }
            for (const s of bgEffects.stars) s.phase += s.speed;
            bgEffects.sunRays += 0.005;
            bgEffects.fogOffset += 0.2;

            // Радуга
            if (currentLevel === 0) {
                const rb = bgEffects.rainbow;
                if (!rb.active) {
                    rb.nextSpawn--;
                    if (rb.nextSpawn <= 0) { rb.active = true; rb.timer = rb.duration; rb.alpha = 0; }
                } else {
                    rb.timer--;
                    if (rb.timer > rb.duration - 60) { rb.alpha += 1/60; if (rb.alpha > 1) rb.alpha = 1; }
                    else if (rb.timer < 60) { rb.alpha -= 1/60; if (rb.alpha < 0) rb.alpha = 0; }
                    if (rb.timer <= 0) { rb.active = false; rb.alpha = 0; rb.nextSpawn = 900 + Math.random() * 1500; }
                }
            }
            // Закат
            if (currentLevel === 1) {
                bgEffects.sunset.progress += bgEffects.sunset.speed;
                if (bgEffects.sunset.progress > 1) bgEffects.sunset.progress = 1;
                bgEffects.sunset.glowPulse += 0.03;
            }
            // Гроза + фейерверки
            if (currentLevel === 0) {
                const st = bgEffects.storm;
                st.timer--;
                if (st.state === 'clear' && st.timer <= 0) {
                    st.state = 'incoming'; st.timer = 180;
                    st.rainDrops = [];
                    for (let i = 0; i < 60; i++) {
                        st.rainDrops.push({
                            x: Math.random() * W, y: Math.random() * H - H,
                            speed: 8 + Math.random() * 6,
                            length: 10 + Math.random() * 12,
                            alpha: 0.4 + Math.random() * 0.4
                        });
                    }
                } else if (st.state === 'incoming') {
                    st.intensity += 1/180;
                    if (st.intensity >= 1) { st.intensity = 1; st.state = 'storm'; st.timer = 600; }
                } else if (st.state === 'storm') {
                    st.lightningTimer--;
                    if (st.lightningTimer <= 0) { st.lightningTimer = 60 + Math.random() * 180; st.lightning = 15; }
                    if (st.lightning > 0) st.lightning--;
                    for (const r of st.rainDrops) {
                        r.y += r.speed; r.x -= 1.5;
                        if (r.y > H) { r.y = -20; r.x = Math.random() * W + 100; }
                        if (r.x < -20) r.x = W + 20;
                    }
                    if (st.timer <= 0) { st.state = 'outgoing'; st.timer = 180; }
                } else if (st.state === 'outgoing') {
                    st.intensity -= 1/180;
                    if (st.intensity <= 0) { st.intensity = 0; st.state = 'clear'; st.timer = 1500 + Math.random() * 1500; }
                }
                if (st.state === 'clear' && st.intensity < 0.1) {
                    bgEffects.fireworkTimer--;
                    if (bgEffects.fireworkTimer <= 0) {
                        bgEffects.fireworkTimer = 400 + Math.random() * 500;
                        spawnSkyFirework(150 + Math.random() * (W - 300), 60 + Math.random() * 120);
                    }
                }
                for (let i = bgEffects.fireworks.length - 1; i >= 0; i--) {
                    const f = bgEffects.fireworks[i];
                    f.x += f.vx; f.y += f.vy;
                    f.vy += 0.08; f.vx *= 0.99; f.life--;
                    if (f.life <= 0) bgEffects.fireworks.splice(i, 1);
                }
            }

            if (win) { jumpEdge = false; return; }

            const enemyMul = player.slowTime > 0 ? SLOW_FACTOR : 1;

            // Враги
            for (const e of enemies) {
                if (e.type === 'patrol') {
                    e.x += e.speed * e.dir * enemyMul;
                    if (e.x <= e.platX) { e.x = e.platX; e.dir = 1; }
                    if (e.x + e.w >= e.platX + e.platW) { e.x = e.platX + e.platW - e.w; e.dir = -1; }
                } else if (e.type === 'flyer') {
                    e.y += e.speed * e.dir * enemyMul;
                    if (e.y <= e.baseY - e.range) { e.y = e.baseY - e.range; e.dir = 1; }
                    if (e.y >= e.baseY + e.range) { e.y = e.baseY + e.range; e.dir = -1; }
                } else if (e.type === 'chaser') {
                    const ecx = e.x + e.w / 2, pcx = player.x + player.w / 2;
                    const distX = Math.abs(pcx - ecx);
                    const dy = Math.abs((player.y + player.h / 2) - (e.y + e.h / 2));
                    const chasing = distX < (e.chaseRadius || 240) && dy < 120;
                    let speed = chasing ? e.speed : (e.baseSpeed || e.speed * 0.5);
                    if (chasing) e.dir = (pcx > ecx) ? 1 : -1;
                    e.x += speed * e.dir * enemyMul;
                    if (e.x <= e.platX) { e.x = e.platX; e.dir = 1; }
                    if (e.x + e.w >= e.platX + e.platW) { e.x = e.platX + e.platW - e.w; e.dir = -1; }
                    e.chasing = chasing;
                } else if (e.type === 'jumper') {
                    e.vy += GRAVITY * 0.9;
                    e.y += e.vy * enemyMul;
                    if (e.y >= e.baseY) {
                        e.y = e.baseY; e.vy = 0;
                        e.jumpTimer -= 1;
                        if (e.jumpTimer <= 0) {
                            e.vy = -9;
                            e.jumpTimer = e.jumpInterval || 90;
                            spawnParticles(e.x + e.w / 2, e.y + e.h, '#69F0AE', 5);
                        }
                    }
                } else if (e.type === 'shooter') {
                    e.shootTimer -= 1;
                    if (e.shootTimer <= 0) {
                        const pcx2 = player.x + player.w / 2, pcy2 = player.y + player.h / 2;
                        const ecx2 = e.x + e.w / 2, ecy2 = e.y + e.h / 2;
                        const dx2 = pcx2 - ecx2, dy2 = pcy2 - ecy2;
                        const dist2 = Math.sqrt(dx2 * dx2 + dy2 * dy2);
                        if (dist2 < 520) {
                            e.shootTimer = e.shootInterval || 120;
                            const speed2 = 3.8;
                            enemyProjectiles.push({
                                x: ecx2, y: ecy2,
                                vx: (dx2 / dist2) * speed2,
                                vy: (dy2 / dist2) * speed2,
                                r: 9, life: 200
                            });
                            Sound.bossShoot();
                        } else { e.shootTimer = 30; }
                    }
                } else if (e.type === 'bat') {
                    e.x += e.speed * e.dirX * enemyMul;
                    e.y += e.speed * e.dirY * enemyMul;
                    if (e.x <= e.baseX - e.range) { e.x = e.baseX - e.range; e.dirX = 1; }
                    if (e.x >= e.baseX + e.range) { e.x = e.baseX + e.range; e.dirX = -1; }
                    if (e.y <= e.baseY - e.range * 0.5) { e.y = e.baseY - e.range * 0.5; e.dirY = Math.abs(e.dirY); }
                    if (e.y >= e.baseY + e.range * 0.5) { e.y = e.baseY + e.range * 0.5; e.dirY = -Math.abs(e.dirY); }
                    e.wingPhase = (e.wingPhase || 0) + 0.4;
                }
            }

            // Магнит
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

            // Управление
            player.vx = 0;
            const spd = getMoveSpeed();
            if (keys.left) { player.vx = -spd; player.facing = -1; }
            if (keys.right) { player.vx = spd; player.facing = 1; }

            if (jumpEdge && player.jumpsLeft > 0) {
                player.vy = JUMP_POWER;
                player.jumpsLeft--;
                player.onGround = false;
                if (player.jumpsLeft === MAX_JUMPS - 1) Sound.jump(); else Sound.doubleJump();
                spawnParticles(player.x + player.w/2, player.y + player.h,
                    player.jumpsLeft === MAX_JUMPS - 1 ? '#fff' : '#FFD700', 6);
            }
            jumpEdge = false;

            player.vy += GRAVITY;
            if (player.vy > 20) player.vy = 20;

            // X
            player.x += player.vx;
            if (player.x < 0) player.x = 0;
            if (player.x + player.w > W) player.x = W - player.w;
            for (const p of platforms) {
                if (rectsCollide(player, p)) {
                    if (player.vx > 0) player.x = p.x - player.w;
                    else if (player.vx < 0) player.x = p.x + p.w;
                }
            }
            for (const mp of movingPlatforms) {
                if (rectsCollide(player, mp)) {
                    if (player.y + player.h > mp.y + 4 && player.y < mp.y + mp.h - 4) {
                        if (player.vx > 0) player.x = mp.x - player.w;
                        else if (player.vx < 0) player.x = mp.x + mp.w;
                    }
                }
            }
            for (const dp of disappearingPlatforms) {
                if (dp.phase === 'hidden') continue;
                if (rectsCollide(player, dp)) {
                    if (player.y + player.h > dp.y + 4 && player.y < dp.y + dp.h - 4) {
                        if (player.vx > 0) player.x = dp.x - player.w;
                        else if (player.vx < 0) player.x = dp.x + dp.w;
                    }
                }
            }

            // Y
            player.y += player.vy;
            player.onGround = false;
            let ridingPlatform = null;
            for (const p of platforms) {
                if (rectsCollide(player, p)) {
                    if (player.vy > 0) { player.y = p.y - player.h; player.vy = 0; player.onGround = true; }
                    else if (player.vy < 0) { player.y = p.y + p.h; player.vy = 0; }
                }
            }
            for (const mp of movingPlatforms) {
                if (rectsCollide(player, mp)) {
                    if (player.vy >= 0 && player.y + player.h - player.vy <= mp.y + 8) {
                        player.y = mp.y - player.h; player.vy = 0;
                        player.onGround = true; ridingPlatform = mp;
                    } else if (player.vy < 0 && player.y >= mp.y + mp.h - 8) {
                        player.y = mp.y + mp.h; player.vy = 0;
                    } else {
                        if (player.x + player.w / 2 < mp.x + mp.w / 2) player.x = mp.x - player.w;
                        else player.x = mp.x + mp.w;
                    }
                }
            }
            for (const dp of disappearingPlatforms) {
                if (dp.phase === 'hidden') continue;
                if (rectsCollide(player, dp)) {
                    if (player.vy > 0 && player.y + player.h - player.vy <= dp.y + 8) {
                        player.y = dp.y - player.h; player.vy = 0; player.onGround = true;
                    } else if (player.vy < 0 && player.y >= dp.y + dp.h - 8) {
                        player.y = dp.y + dp.h; player.vy = 0;
                    }
                }
            }
            if (ridingPlatform) {
                player.x += ridingPlatform.dx || 0;
                player.y += ridingPlatform.dy || 0;
            }
            if (player.onGround) player.jumpsLeft = MAX_JUMPS;

            if (player.y > H + 200) hitPlayer();

            // Лава
            for (const l of lava) {
                if (rectsCollide(player, l)) {
                    player.shieldTime = 0; player.invuln = 0;
                    lives = 1;
                    spawnParticles(player.x + player.w/2, player.y + player.h/2, '#FF5722', 30);
                    spawnParticles(player.x + player.w/2, player.y + player.h/2, '#FFC107', 20);
                    hitPlayer();
                    break;
                }
            }

            // Столкновение с врагами
            for (const e of enemies) if (rectsCollide(player, e)) hitPlayer();

            // Монеты
            for (const c of coins) {
                if (!c.collected && circleRectCollide(c.x, c.y, c.r, player)) {
                    c.collected = true; score += 10; attemptCoins++;
                    Sound.coin();
                    spawnParticles(c.x, c.y, '#FFD700', 12);
                }
            }

            // Бонусы
            for (const b of bonuses) {
                if (!b.active) continue;
                const by = b.y + Math.sin(Date.now() / 300 + b.phase) * 4;
                if (circleRectCollide(b.x, by, b.r, player)) {
                    b.active = false; b.respawnTimer = BONUS_RESPAWN;
                    Sound.bonus();
                    if (b.type === 'shield') { player.shieldTime = EFFECT_DURATION; spawnParticles(b.x, by, '#2196F3', 20); }
                    else if (b.type === 'speed') { player.speedTime = EFFECT_DURATION; spawnParticles(b.x, by, '#FFC107', 20); }
                    else if (b.type === 'heart') { lives = Math.min(lives + 1, MAX_LIVES); spawnParticles(b.x, by, '#E91E63', 20); }
                    else if (b.type === 'magnet') { player.magnetTime = EFFECT_DURATION; spawnParticles(b.x, by, '#9C27B0', 20); }
                    else if (b.type === 'slow') { player.slowTime = EFFECT_DURATION; spawnParticles(b.x, by, '#00BCD4', 20); }
                }
            }

            // Движущиеся платформы — обновление
            for (const mp of movingPlatforms) {
                if (mp.axis === 'x') {
                    const dx = mp.speed * mp.dir;
                    mp.x += dx;
                    if (mp.x <= mp.startX) { mp.x = mp.startX; mp.dir = 1; }
                    if (mp.x + mp.w >= mp.endX) { mp.x = mp.endX - mp.w; mp.dir = -1; }
                    mp.dx = dx; mp.dy = 0;
                } else if (mp.axis === 'y') {
                    const dy = mp.speed * mp.dir;
                    mp.y += dy;
                    if (mp.y <= mp.startY) { mp.y = mp.startY; mp.dir = 1; }
                    if (mp.y + mp.h >= mp.endY) { mp.y = mp.endY - mp.h; mp.dir = -1; }
                    mp.dy = dy; mp.dx = 0;
                }
            }

            // Исчезающие платформы — обновление
            for (const dp of disappearingPlatforms) {
                dp.timer--;
                if (dp.phase === 'visible') {
                    if (dp.timer <= 0) { dp.phase = 'blinking'; dp.timer = dp.blinkTime; }
                } else if (dp.phase === 'blinking') {
                    if (dp.timer <= 0) { dp.phase = 'hidden'; dp.timer = dp.hiddenTime; }
                } else if (dp.phase === 'hidden') {
                    if (dp.timer <= 0) { dp.phase = 'appearing'; dp.timer = dp.appearTime; }
                } else if (dp.phase === 'appearing') {
                    if (dp.timer <= 0) { dp.phase = 'visible'; dp.timer = dp.visibleTime; }
                }
                if (dp.phase === 'visible') dp.appearance = 1;
                else if (dp.phase === 'blinking') dp.appearance = 0.3 + Math.abs(Math.sin(Date.now() / 80)) * 0.7;
                else if (dp.phase === 'hidden') dp.appearance = 0;
                else if (dp.phase === 'appearing') dp.appearance = 1 - (dp.timer / dp.appearTime);
            }

            // Босс появляется на 3-м уровне
            const collected = coins.filter(c => c.collected).length;
            if (currentLevel === 2 && collected === TOTAL_COINS && !boss.active && !bossDefeated) spawnBoss();
            if (boss.active) updateBoss();
            updateBossProjectiles();
            updateEnemyProjectiles();
            updateCelebration();

            const bossOk = currentLevel !== 2 || bossDefeated;
            if (!win && rectsCollide(player, flag) && collected === TOTAL_COINS && bossOk) onWin();
        }

        // ============================================
        // ОТРИСОВКА
        // ============================================
        const BONUS_COLORS = { shield: '#2196F3', speed: '#FFC107', heart: '#E91E63', magnet: '#9C27B0', slow: '#00BCD4' };
        const BONUS_ICONS = { shield: '🛡', speed: '⚡', heart: '♥', magnet: '🧲', slow: '🐢' };

        function draw() {
            ctx.save();
            ctx.translate(gameOver.shakeX, gameOver.shakeY);

            const grad = ctx.createLinearGradient(0, 0, 0, H);
            if (currentLevel === 0) { grad.addColorStop(0, '#87CEEB'); grad.addColorStop(1, '#E0F6FF'); }
            else if (currentLevel === 1) { grad.addColorStop(0, '#5B2C6F'); grad.addColorStop(1, '#F5B7B1'); }
            else { grad.addColorStop(0, '#1A0033'); grad.addColorStop(1, '#4A148C'); }
            ctx.fillStyle = grad;
            ctx.fillRect(-50, -50, W + 100, H + 100);

            // Фоновые эффекты уровня 1
            if (currentLevel === 0) {
                if (bgEffects.storm.intensity > 0) {
                    ctx.fillStyle = `rgba(20, 25, 45, ${bgEffects.storm.intensity * 0.55})`;
                    ctx.fillRect(-50, -50, W + 100, H + 100);
                }
                if (bgEffects.rainbow.active && bgEffects.rainbow.alpha > 0) {
                    const rb = bgEffects.rainbow;
                    const cx = W * 0.5, cy = H + 180, outerR = 420;
                    const colors = ['#FF0000', '#FF7F00', '#FFFF00', '#00FF00', '#0000FF', '#4B0082', '#9400D3'];
                    ctx.save();
                    ctx.globalAlpha = rb.alpha * 0.55;
                    ctx.lineWidth = 18; ctx.lineCap = 'round';
                    for (let i = 0; i < colors.length; i++) {
                        ctx.strokeStyle = colors[i];
                        ctx.beginPath();
                        ctx.arc(cx, cy, outerR - i * 18, Math.PI, Math.PI * 2);
                        ctx.stroke();
                    }
                    ctx.globalAlpha = rb.alpha * 0.15;
                    const glowGrad = ctx.createRadialGradient(cx, cy, outerR - 150, cx, cy, outerR + 80);
                    glowGrad.addColorStop(0, 'rgba(255,255,255,0)');
                    glowGrad.addColorStop(0.5, 'rgba(255,255,200,0.8)');
                    glowGrad.addColorStop(1, 'rgba(255,255,255,0)');
                    ctx.fillStyle = glowGrad;
                    ctx.beginPath(); ctx.arc(cx, cy, outerR + 80, Math.PI, Math.PI * 2); ctx.fill();
                    ctx.restore();
                }
                for (const c of bgEffects.clouds) {
                    ctx.fillStyle = 'rgba(255, 255, 255, ' + c.alpha + ')';
                    ctx.beginPath();
                    ctx.ellipse(c.x, c.y, c.w * 0.5, c.h * 0.5, 0, 0, Math.PI * 2);
                    ctx.ellipse(c.x - c.w * 0.25, c.y + 4, c.w * 0.35, c.h * 0.4, 0, 0, Math.PI * 2);
                    ctx.ellipse(c.x + c.w * 0.25, c.y + 4, c.w * 0.35, c.h * 0.4, 0, 0, Math.PI * 2);
                    ctx.fill();
                }
                if (bgEffects.storm.intensity > 0.1) {
                    const st = bgEffects.storm;
                    const cloudAlpha = Math.min(1, st.intensity) * 0.85;
                    ctx.fillStyle = `rgba(30, 30, 50, ${cloudAlpha})`;
                    const time = Date.now() / 1000;
                    for (let i = 0; i < 4; i++) {
                        const cx2 = ((time * 20 * (0.5 + i * 0.2)) + i * 200) % (W + 300) - 150;
                        const cy2 = 50 + i * 25;
                        const cw2 = 180 + i * 20;
                        const ch2 = 50 + i * 8;
                        ctx.beginPath();
                        ctx.ellipse(cx2, cy2, cw2 * 0.5, ch2 * 0.5, 0, 0, Math.PI * 2);
                        ctx.ellipse(cx2 - cw2 * 0.3, cy2 + 8, cw2 * 0.35, ch2 * 0.45, 0, 0, Math.PI * 2);
                        ctx.ellipse(cx2 + cw2 * 0.3, cy2 + 6, cw2 * 0.4, ch2 * 0.5, 0, 0, Math.PI * 2);
                        ctx.fill();
                    }
                    if (st.lightning > 0) {
                        const flashAlpha = (st.lightning / 15) * 0.7;
                        ctx.fillStyle = `rgba(220, 230, 255, ${flashAlpha})`;
                        ctx.fillRect(-50, -50, W + 100, H + 100);
                        if (st.lightning > 10) {
                            const lx = 200 + Math.random() * (W - 400);
                            ctx.strokeStyle = `rgba(255, 255, 255, ${flashAlpha})`;
                            ctx.lineWidth = 3;
                            ctx.shadowColor = '#AACCFF'; ctx.shadowBlur = 20;
                            ctx.beginPath();
                            ctx.moveTo(lx, 0);
                            let cx3 = lx, cy3 = 0;
                            for (let i = 0; i < 8; i++) {
                                cx3 += (Math.random() - 0.5) * 40;
                                cy3 += 30 + Math.random() * 30;
                                ctx.lineTo(cx3, cy3);
                            }
                            ctx.stroke();
                            ctx.shadowBlur = 0;
                        }
                    }
                }
            }

            // Солнце / луна / закат
            if (currentLevel === 2) {
                ctx.fillStyle = '#E1BEE7';
                ctx.beginPath(); ctx.arc(W - 80, 80, 40, 0, Math.PI * 2); ctx.fill();
                ctx.fillStyle = '#CE93D8';
                ctx.beginPath(); ctx.arc(W - 95, 70, 8, 0, Math.PI * 2); ctx.fill();
                ctx.beginPath(); ctx.arc(W - 65, 95, 6, 0, Math.PI * 2); ctx.fill();
                ctx.beginPath(); ctx.arc(W - 90, 105, 5, 0, Math.PI * 2); ctx.fill();
                for (let i = 0; i < 30; i++) {
                    const sx = (i * 137) % W;
                    const sy = (i * 73) % 200;
                    const twinkle = 0.5 + Math.sin(Date.now() / 500 + i) * 0.5;
                    ctx.fillStyle = 'rgba(255,255,255,' + twinkle + ')';
                    ctx.fillRect(sx, sy, 2, 2);
                }
            } else if (currentLevel === 1) {
                const sp = bgEffects.sunset.progress;
                const sunX = W - 80, sunY = 80 + sp * (H * 0.7);
                const sunR = 45 + Math.sin(bgEffects.sunset.glowPulse) * 3;
                const glowR = 90 + sp * 60;
                const glowGrad = ctx.createRadialGradient(sunX, sunY, 0, sunX, sunY, glowR);
                glowGrad.addColorStop(0, `rgba(255, ${Math.floor(220 - sp * 80)}, ${Math.floor(100 - sp * 80)}, ${0.9 - sp * 0.4})`);
                glowGrad.addColorStop(0.5, `rgba(255, ${Math.floor(140 - sp * 60)}, 60, ${0.4 - sp * 0.2})`);
                glowGrad.addColorStop(1, 'rgba(255, 80, 30, 0)');
                ctx.fillStyle = glowGrad;
                ctx.beginPath(); ctx.arc(sunX, sunY, glowR, 0, Math.PI * 2); ctx.fill();
                const sunGrad = ctx.createRadialGradient(sunX - 10, sunY - 10, 0, sunX, sunY, sunR);
                sunGrad.addColorStop(0, `rgb(${Math.max(255 - sp * 60, 100)}, ${Math.max(230 - sp * 120, 80)}, ${Math.max(100 - sp * 80, 30)})`);
                sunGrad.addColorStop(1, `rgb(${Math.max(195 - sp * 40, 80)}, ${Math.max(140 - sp * 100, 40)}, ${Math.max(50 - sp * 50, 10)})`);
                ctx.fillStyle = sunGrad;
                ctx.beginPath(); ctx.arc(sunX, sunY, sunR, 0, Math.PI * 2); ctx.fill();
                if (sp < 0.85) {
                    ctx.save();
                    ctx.globalCompositeOperation = 'lighter';
                    ctx.globalAlpha = (1 - sp) * 0.6;
                    const rayTime = Date.now() / 3000;
                    for (let i = 0; i < 5; i++) {
                        const a = (Math.PI * 2 / 5) * i + rayTime;
                        ctx.save();
                        ctx.translate(sunX, sunY);
                        ctx.rotate(a);
                        const rayGrad = ctx.createLinearGradient(0, 0, 300, 0);
                        rayGrad.addColorStop(0, 'rgba(255, 200, 80, 0.35)');
                        rayGrad.addColorStop(1, 'rgba(255, 150, 50, 0)');
                        ctx.fillStyle = rayGrad;
                        ctx.beginPath();
                        ctx.moveTo(0, -6); ctx.lineTo(300, -30); ctx.lineTo(300, 30); ctx.lineTo(0, 6);
                        ctx.closePath(); ctx.fill();
                        ctx.restore();
                    }
                    ctx.restore();
                }
            } else {
                ctx.fillStyle = '#FFE066';
                ctx.beginPath(); ctx.arc(W - 80, 80, 45, 0, Math.PI * 2); ctx.fill();
                // Лучи солнца
                ctx.save();
                ctx.globalCompositeOperation = 'lighter';
                const sunX = W - 80, sunY = 80;
                const rayAngle = Math.sin(bgEffects.sunRays) * 0.15;
                for (let i = 0; i < 3; i++) {
                    const a = -0.5 + i * 0.5 + rayAngle;
                    ctx.save();
                    ctx.translate(sunX, sunY);
                    ctx.rotate(a);
                    const rayGrad = ctx.createLinearGradient(0, 0, 400, 0);
                    rayGrad.addColorStop(0, 'rgba(255, 235, 100, 0.18)');
                    rayGrad.addColorStop(1, 'rgba(255, 235, 100, 0)');
                    ctx.fillStyle = rayGrad;
                    ctx.beginPath();
                    ctx.moveTo(0, -10); ctx.lineTo(500, -80); ctx.lineTo(500, 80); ctx.lineTo(0, 10);
                    ctx.closePath(); ctx.fill();
                    ctx.restore();
                }
                ctx.restore();
            }

            // Звёзды на 2 уровне
            if (currentLevel === 1) {
                for (const s of bgEffects.stars) {
                    const twinkle = 0.5 + Math.sin(s.phase) * 0.5;
                    ctx.fillStyle = 'rgba(255, 255, 255, ' + twinkle + ')';
                    ctx.beginPath(); ctx.arc(s.x, s.y, s.size, 0, Math.PI * 2); ctx.fill();
                }
            }

            // Фейерверки в небе (уровень 1)
            for (const f of bgEffects.fireworks) {
                const alpha = f.life / f.maxLife;
                ctx.save();
                ctx.globalCompositeOperation = 'lighter';
                ctx.globalAlpha = alpha;
                ctx.fillStyle = f.color;
                ctx.beginPath(); ctx.arc(f.x, f.y, f.size * alpha, 0, Math.PI * 2); ctx.fill();
                ctx.globalAlpha = alpha * 0.7;
                ctx.fillStyle = '#fff';
                ctx.beginPath(); ctx.arc(f.x - 1, f.y - 1, f.size * alpha * 0.4, 0, Math.PI * 2); ctx.fill();
                ctx.restore();
            }

            // Дождь
            if (currentLevel === 0 && bgEffects.storm.state === 'storm') {
                const st = bgEffects.storm;
                ctx.strokeStyle = `rgba(180, 200, 230, ${0.5 * st.intensity})`;
                ctx.lineWidth = 1.5; ctx.lineCap = 'round';
                ctx.beginPath();
                for (const r of st.rainDrops) {
                    ctx.moveTo(r.x, r.y);
                    ctx.lineTo(r.x - 3, r.y + r.length);
                }
                ctx.stroke();
            }

            // Лепестки (уровень 1)
            for (const p of bgEffects.petals) {
                ctx.save();
                ctx.translate(p.x, p.y);
                ctx.rotate(p.rotation);
                ctx.fillStyle = p.color;
                ctx.beginPath();
                ctx.ellipse(0, 0, p.size * 0.6, p.size * 0.3, 0, 0, Math.PI * 2);
                ctx.fill();
                ctx.restore();
            }

            // Птицы (уровень 2)
            for (const b of bgEffects.birds) {
                const flap = Math.sin(b.flapPhase);
                ctx.strokeStyle = 'rgba(30, 20, 50, 0.75)';
                ctx.lineWidth = 2;
                ctx.beginPath();
                ctx.moveTo(b.x - b.size, b.y + flap * 4);
                ctx.quadraticCurveTo(b.x - b.size * 0.5, b.y - 3, b.x, b.y);
                ctx.quadraticCurveTo(b.x + b.size * 0.5, b.y - 3, b.x + b.size, b.y + flap * 4);
                ctx.stroke();
            }

            // Туман (уровень 2)
            if (currentLevel === 1) {
                const fogGrad = ctx.createLinearGradient(0, H - 200, 0, H);
                fogGrad.addColorStop(0, 'rgba(200, 80, 130, 0)');
                fogGrad.addColorStop(1, 'rgba(200, 80, 130, 0.35)');
                ctx.fillStyle = fogGrad;
                ctx.fillRect(-50, H - 200, W + 100, 250);
            }

            // Платформы
            let grass, dirt, edge;
            if (currentLevel === 0) { grass = '#4CAF50'; dirt = '#8B5A2B'; edge = '#5D3A1A'; }
            else if (currentLevel === 1) { grass = '#7E57C2'; dirt = '#3E2723'; edge = '#1A0F0A'; }
            else { grass = '#6A1B9A'; dirt = '#1A0A0A'; edge = '#000000'; }

            for (const p of platforms) {
                ctx.fillStyle = grass; ctx.fillRect(p.x, p.y, p.w, 8);
                ctx.fillStyle = dirt; ctx.fillRect(p.x, p.y + 8, p.w, p.h - 8);
                ctx.strokeStyle = edge; ctx.lineWidth = 2; ctx.strokeRect(p.x, p.y, p.w, p.h);
            }

            // Лава (уровень 3)
            for (const l of lava) {
                const lavaGrad = ctx.createLinearGradient(0, l.y, 0, l.y + l.h);
                lavaGrad.addColorStop(0, '#FFEB3B');
                lavaGrad.addColorStop(0.4, '#FF9800');
                lavaGrad.addColorStop(1, '#B71C1C');
                ctx.fillStyle = lavaGrad;
                ctx.fillRect(l.x, l.y, l.w, l.h);
                const t = Date.now() / 200;
                ctx.fillStyle = '#FFC107';
                for (let i = 0; i < l.w; i += 8) {
                    const offset = Math.sin(t + i * 0.3) * 3;
                    const size = 3 + Math.sin(t * 1.5 + i * 0.5) * 2;
                    ctx.beginPath(); ctx.arc(l.x + i, l.y + offset, size, 0, Math.PI * 2); ctx.fill();
                }
                const glow = ctx.createRadialGradient(l.x + l.w/2, l.y, 0, l.x + l.w/2, l.y, l.w * 0.7);
                glow.addColorStop(0, 'rgba(255,87,34,0.4)');
                glow.addColorStop(1, 'rgba(255,87,34,0)');
                ctx.fillStyle = glow;
                ctx.beginPath(); ctx.arc(l.x + l.w/2, l.y, l.w * 0.7, 0, Math.PI * 2); ctx.fill();
                if (Math.random() < 0.05) {
                    particles.push({
                        x: l.x + Math.random() * l.w, y: l.y,
                        vx: (Math.random() - 0.5) * 1.5, vy: -2 - Math.random() * 2,
                        life: 40, maxLife: 40,
                        color: Math.random() < 0.5 ? '#FF5722' : '#FFC107',
                        size: 2 + Math.random() * 3
                    });
                }
            }

            // Движущиеся платформы
            for (const mp of movingPlatforms) {
                const glowColor = currentLevel === 1 ? 'rgba(126, 87, 194, 0.4)' : 'rgba(255, 193, 7, 0.4)';
                const gl = ctx.createRadialGradient(mp.x + mp.w/2, mp.y + mp.h/2, 0, mp.x + mp.w/2, mp.y + mp.h/2, mp.w);
                gl.addColorStop(0, glowColor);
                gl.addColorStop(1, 'rgba(0,0,0,0)');
                ctx.fillStyle = gl;
                ctx.beginPath(); ctx.arc(mp.x + mp.w/2, mp.y + mp.h/2, mp.w, 0, Math.PI * 2); ctx.fill();
                ctx.fillStyle = currentLevel === 1 ? '#9575CD' : '#FFD54F';
                ctx.fillRect(mp.x, mp.y, mp.w, mp.h);
                ctx.strokeStyle = currentLevel === 1 ? '#4527A0' : '#E65100';
                ctx.lineWidth = 2; ctx.strokeRect(mp.x, mp.y, mp.w, mp.h);
                ctx.fillStyle = currentLevel === 1 ? '#B39DDB' : '#FFE082';
                ctx.fillRect(mp.x, mp.y, mp.w, 4);
                ctx.fillStyle = 'rgba(255, 255, 255, 0.7)';
                const centerX = mp.x + mp.w / 2;
                const centerY = mp.y + mp.h / 2;
                if (mp.axis === 'x') {
                    ctx.beginPath(); ctx.moveTo(centerX - 12, centerY); ctx.lineTo(centerX - 6, centerY - 4); ctx.lineTo(centerX - 6, centerY + 4); ctx.closePath(); ctx.fill();
                    ctx.beginPath(); ctx.moveTo(centerX + 12, centerY); ctx.lineTo(centerX + 6, centerY - 4); ctx.lineTo(centerX + 6, centerY + 4); ctx.closePath(); ctx.fill();
                } else {
                    ctx.beginPath(); ctx.moveTo(centerX, centerY - 8); ctx.lineTo(centerX - 4, centerY - 3); ctx.lineTo(centerX + 4, centerY - 3); ctx.closePath(); ctx.fill();
                    ctx.beginPath(); ctx.moveTo(centerX, centerY + 8); ctx.lineTo(centerX - 4, centerY + 3); ctx.lineTo(centerX + 4, centerY + 3); ctx.closePath(); ctx.fill();
                }
                ctx.fillStyle = currentLevel === 1 ? '#4527A0' : '#BF360C';
                ctx.beginPath();
                ctx.arc(mp.x + 6, mp.y + mp.h / 2, 2.5, 0, Math.PI * 2);
                ctx.arc(mp.x + mp.w - 6, mp.y + mp.h / 2, 2.5, 0, Math.PI * 2);
                ctx.fill();
            }

            // Исчезающие платформы
            for (const dp of disappearingPlatforms) {
                if (dp.phase === 'hidden') continue;
                ctx.save();
                ctx.globalAlpha = dp.appearance;
                const glowColor = currentLevel === 1 ? 'rgba(255, 183, 77, 0.5)' : currentLevel === 2 ? 'rgba(255, 82, 82, 0.5)' : 'rgba(255, 193, 7, 0.5)';
                const gl = ctx.createRadialGradient(dp.x + dp.w / 2, dp.y + dp.h / 2, 0, dp.x + dp.w / 2, dp.y + dp.h / 2, dp.w);
                gl.addColorStop(0, glowColor);
                gl.addColorStop(1, 'rgba(0,0,0,0)');
                ctx.fillStyle = gl;
                ctx.beginPath(); ctx.arc(dp.x + dp.w / 2, dp.y + dp.h / 2, dp.w, 0, Math.PI * 2); ctx.fill();
                const baseColor = currentLevel === 2 ? '#E53935' : '#FB8C00';
                const edgeColor = currentLevel === 2 ? '#7F0000' : '#BF360C';
                ctx.fillStyle = baseColor;
                ctx.fillRect(dp.x, dp.y, dp.w, dp.h);
                ctx.strokeStyle = edgeColor;
                ctx.lineWidth = 2; ctx.strokeRect(dp.x, dp.y, dp.w, dp.h);
                ctx.fillStyle = 'rgba(255, 255, 255, 0.5)';
                ctx.fillRect(dp.x, dp.y, dp.w, 3);
                ctx.strokeStyle = 'rgba(255, 255, 255, 0.6)';
                ctx.lineWidth = 1;
                ctx.setLineDash([4, 4]);
                ctx.beginPath();
                ctx.moveTo(dp.x + 4, dp.y + dp.h / 2);
                ctx.lineTo(dp.x + dp.w - 4, dp.y + dp.h / 2);
                ctx.stroke();
                ctx.setLineDash([]);
                ctx.fillStyle = 'rgba(255, 255, 255, 0.9)';
                ctx.font = 'bold 11px Arial';
                ctx.textAlign = 'center';
                ctx.textBaseline = 'middle';
                if (dp.phase === 'blinking') {
                    ctx.font = 'bold 14px Arial';
                    ctx.fillText('!', dp.x + dp.w / 2, dp.y + dp.h / 2);
                } else if (dp.phase === 'appearing') {
                    ctx.fillText('⌛', dp.x + dp.w / 2, dp.y + dp.h / 2);
                } else {
                    ctx.fillText('⏱', dp.x + dp.w / 2, dp.y + dp.h / 2);
                }
                ctx.textBaseline = 'alphabetic';
                if (dp.phase === 'blinking') {
                    ctx.strokeStyle = 'rgba(255, 100, 100, 0.9)';
                    ctx.lineWidth = 2; ctx.strokeRect(dp.x - 2, dp.y - 2, dp.w + 4, dp.h + 4);
                }
                ctx.restore();
            }

            // Флаг
            const allCoins = coins.every(c => c.collected);
            ctx.fillStyle = '#666';
            ctx.fillRect(flag.x + flag.w/2 - 2, flag.y, 4, flag.h);
            ctx.fillStyle = allCoins ? '#00C853' : '#E53935';
            ctx.beginPath();
            ctx.moveTo(flag.x + flag.w/2 + 2, flag.y);
            ctx.lineTo(flag.x + flag.w/2 + 30, flag.y + 12);
            ctx.lineTo(flag.x + flag.w/2 + 2, flag.y + 24);
            ctx.closePath(); ctx.fill();

            // Монеты
            for (const c of coins) {
                if (c.collected) continue;
                ctx.fillStyle = '#FFD700';
                ctx.beginPath(); ctx.arc(c.x, c.y, c.r, 0, Math.PI*2); ctx.fill();
                ctx.strokeStyle = '#DAA520'; ctx.lineWidth = 3; ctx.stroke();
                ctx.fillStyle = '#FFF8B0';
                ctx.beginPath(); ctx.arc(c.x - 3, c.y - 3, c.r * 0.35, 0, Math.PI*2); ctx.fill();
            }

            // Бонусы и враги
            for (const b of bonuses) if (b.active) drawBonus(b);
            for (const e of enemies) drawEnemy(e);

            drawBoss();
            drawBossProjectiles();
            drawEnemyProjectiles();

            // Частицы
            for (const p of particles) {
                ctx.globalAlpha = p.life / p.maxLife;
                ctx.fillStyle = p.color;
                ctx.beginPath(); ctx.arc(p.x, p.y, p.size, 0, Math.PI*2); ctx.fill();
            }
            ctx.globalAlpha = 1;

            if (!gameOver.active) drawPlayer();

            // HUD
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

            // Подсказка
            if (score === 0 && !win && lives === 3 && player.invuln < 30 && !gameOver.active) {
                ctx.fillStyle = 'rgba(0,0,0,0.6)';
                ctx.fillRect(W/2 - 280, H - 140, 560, 40);
                ctx.fillStyle = '#fff';
                ctx.font = '15px Arial';
                ctx.textAlign = 'center';
                ctx.fillText(t('hint'), W/2, H - 115);
            }

            if (hurtFlash > 0) {
                ctx.fillStyle = 'rgba(255,0,0,' + (hurtFlash / 40) + ')';
                ctx.fillRect(-50, -50, W + 100, H + 100);
            }
            if (player.slowTime > 0) {
                ctx.fillStyle = 'rgba(0,188,212,0.08)';
                ctx.fillRect(-50, -50, W + 100, H + 100);
            }

            drawCelebration();

            // Победа
            if (win) {
                ctx.fillStyle = 'rgba(0,0,0,0.7)';
                ctx.fillRect(-50, -50, W + 100, H + 100);
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

            drawGameOver();

            ctx.restore();
        }

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
            if (player.speedTime > 0) active.push({ type: 'speed', time: player.speedTime });
            if (player.magnetTime > 0) active.push({ type: 'magnet', time: player.magnetTime });
            if (player.slowTime > 0) active.push({ type: 'slow', time: player.slowTime });
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
                ctx.beginPath(); ctx.arc(x + w/2, y + h/2, (w/2 + 8) * pulse, 0, Math.PI*2); ctx.stroke();
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
                    ctx.beginPath(); ctx.arc(x + w/2, y + h/2, r, 0, Math.PI*2); ctx.stroke();
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
                    ctx.beginPath(); ctx.arc(x + w/2 - 8 + i * 16, y - 12, 4, 0, Math.PI*2); ctx.fill();
                }
            }
        }

        function drawEnemy(e) {
            if (player.slowTime > 0) {
                ctx.fillStyle = 'rgba(0,188,212,0.25)';
                ctx.beginPath(); ctx.arc(e.x + e.w/2, e.y + e.h/2, e.w, 0, Math.PI*2); ctx.fill();
            }
            if (e.type === 'patrol') {
                ctx.fillStyle = '#8E24AA';
                ctx.fillRect(e.x, e.y, e.w, e.h);
                ctx.strokeStyle = '#4A148C'; ctx.lineWidth = 2; ctx.strokeRect(e.x, e.y, e.w, e.h);
                ctx.fillStyle = '#fff';
                ctx.fillRect(e.x + 5, e.y + 8, 7, 7);
                ctx.fillRect(e.x + e.w - 12, e.y + 8, 7, 7);
                ctx.fillStyle = '#000';
                ctx.fillRect(e.x + 7, e.y + 10, 3, 3);
                ctx.fillRect(e.x + e.w - 10, e.y + 10, 3, 3);
                ctx.fillStyle = '#fff';
                for (let i = 0; i < 3; i++) ctx.fillRect(e.x + 5 + i * 7, e.y + e.h - 10, 4, 5);
            } else if (e.type === 'flyer') {
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
            } else if (e.type === 'chaser') {
                if (e.chasing) {
                    ctx.fillStyle = 'rgba(255, 80, 80, 0.25)';
                    ctx.beginPath(); ctx.arc(e.x + e.w / 2, e.y + e.h / 2, e.w * 1.2, 0, Math.PI * 2); ctx.fill();
                }
                ctx.fillStyle = '#7F0000';
                for (let i = 0; i < 4; i++) {
                    ctx.beginPath();
                    ctx.moveTo(e.x + 3 + i * 7, e.y);
                    ctx.lineTo(e.x + 7 + i * 7, e.y - 9);
                    ctx.lineTo(e.x + 11 + i * 7, e.y);
                    ctx.closePath(); ctx.fill();
                }
                ctx.fillStyle = e.chasing ? '#FF1744' : '#D32F2F';
                ctx.fillRect(e.x, e.y, e.w, e.h);
                ctx.strokeStyle = '#5D0000'; ctx.lineWidth = 2; ctx.strokeRect(e.x, e.y, e.w, e.h);
                ctx.fillStyle = '#FFF';
                ctx.fillRect(e.x + 5, e.y + 8, 8, 8);
                ctx.fillRect(e.x + e.w - 13, e.y + 8, 8, 8);
                ctx.fillStyle = '#000';
                ctx.fillRect(e.x + 7, e.y + 10, 4, 4);
                ctx.fillRect(e.x + e.w - 11, e.y + 10, 4, 4);
                ctx.strokeStyle = '#000'; ctx.lineWidth = 2;
                ctx.beginPath();
                ctx.moveTo(e.x + 3, e.y + 5); ctx.lineTo(e.x + 14, e.y + 10);
                ctx.moveTo(e.x + e.w - 3, e.y + 5); ctx.lineTo(e.x + e.w - 14, e.y + 10);
                ctx.stroke();
                ctx.fillStyle = '#FFF';
                ctx.fillRect(e.x + 8, e.y + e.h - 8, 4, 5);
                ctx.fillRect(e.x + e.w - 12, e.y + e.h - 8, 4, 5);
            } else if (e.type === 'jumper') {
                if (e.baseY !== undefined) {
                    const jumpH = Math.min(1, Math.abs(e.baseY - e.y) / 100);
                    const shadowScale = 1 - jumpH * 0.6;
                    ctx.fillStyle = 'rgba(0,0,0,0.25)';
                    ctx.beginPath();
                    ctx.ellipse(e.x + e.w / 2, e.baseY + e.h + 4, 18 * shadowScale, 5 * shadowScale, 0, 0, Math.PI * 2);
                    ctx.fill();
                }
                ctx.fillStyle = '#43A047';
                ctx.fillRect(e.x, e.y, e.w, e.h);
                ctx.strokeStyle = '#1B5E20'; ctx.lineWidth = 2; ctx.strokeRect(e.x, e.y, e.w, e.h);
                ctx.strokeStyle = '#1B5E20'; ctx.lineWidth = 2;
                ctx.beginPath();
                ctx.moveTo(e.x + 6, e.y); ctx.lineTo(e.x + 4, e.y - 8);
                ctx.moveTo(e.x + e.w - 6, e.y); ctx.lineTo(e.x + e.w - 4, e.y - 8);
                ctx.stroke();
                ctx.fillStyle = '#FFEB3B';
                ctx.beginPath();
                ctx.arc(e.x + 4, e.y - 8, 3, 0, Math.PI * 2);
                ctx.arc(e.x + e.w - 4, e.y - 8, 3, 0, Math.PI * 2);
                ctx.fill();
                ctx.fillStyle = '#FFF';
                ctx.beginPath();
                ctx.arc(e.x + 10, e.y + 12, 6, 0, Math.PI * 2);
                ctx.arc(e.x + e.w - 10, e.y + 12, 6, 0, Math.PI * 2);
                ctx.fill();
                ctx.fillStyle = '#000';
                ctx.beginPath();
                ctx.arc(e.x + 10, e.y + 12, 3, 0, Math.PI * 2);
                ctx.arc(e.x + e.w - 10, e.y + 12, 3, 0, Math.PI * 2);
                ctx.fill();
                ctx.fillStyle = '#1B5E20';
                ctx.fillRect(e.x + 8, e.y + e.h - 12, e.w - 16, 4);
            } else if (e.type === 'shooter') {
                ctx.fillStyle = '#546E7A';
                ctx.fillRect(e.x, e.y + 10, e.w, e.h - 10);
                ctx.strokeStyle = '#263238'; ctx.lineWidth = 2; ctx.strokeRect(e.x, e.y + 10, e.w, e.h - 10);
                ctx.fillStyle = '#78909C';
                ctx.fillRect(e.x + 4, e.y, e.w - 8, 14);
                ctx.strokeRect(e.x + 4, e.y, e.w - 8, 14);
                const dirToPlayer = (player.x + player.w / 2) > (e.x + e.w / 2) ? 1 : -1;
                ctx.fillStyle = '#455A64';
                ctx.fillRect(e.x + e.w / 2 - 4 + (dirToPlayer > 0 ? 8 : -16), e.y + 4, 16, 6);
                ctx.fillStyle = '#FF5252';
                ctx.beginPath(); ctx.arc(e.x + e.w / 2, e.y + 22, 5, 0, Math.PI * 2); ctx.fill();
                ctx.fillStyle = '#FFF';
                ctx.beginPath(); ctx.arc(e.x + e.w / 2 - 1, e.y + 21, 2, 0, Math.PI * 2); ctx.fill();
                ctx.fillStyle = '#37474F';
                ctx.fillRect(e.x + 4, e.y + e.h - 4, 4, 4);
                ctx.fillRect(e.x + e.w - 8, e.y + e.h - 4, 4, 4);
            } else if (e.type === 'bat') {
                const wingPhase = e.wingPhase || 0;
                const wingUp = Math.sin(wingPhase) > 0;
                ctx.fillStyle = '#4A148C';
                ctx.beginPath();
                if (wingUp) {
                    ctx.moveTo(e.x, e.y + 8); ctx.lineTo(e.x - 12, e.y - 2); ctx.lineTo(e.x - 6, e.y + 10);
                    ctx.closePath(); ctx.fill();
                    ctx.beginPath();
                    ctx.moveTo(e.x + e.w, e.y + 8); ctx.lineTo(e.x + e.w + 12, e.y - 2); ctx.lineTo(e.x + e.w + 6, e.y + 10);
                    ctx.closePath(); ctx.fill();
                } else {
                    ctx.beginPath();
                    ctx.moveTo(e.x, e.y + 8); ctx.lineTo(e.x - 12, e.y + 18); ctx.lineTo(e.x - 6, e.y + 6);
                    ctx.closePath(); ctx.fill();
                    ctx.beginPath();
                    ctx.moveTo(e.x + e.w, e.y + 8); ctx.lineTo(e.x + e.w + 12, e.y + 18); ctx.lineTo(e.x + e.w + 6, e.y + 6);
                    ctx.closePath(); ctx.fill();
                }
                ctx.fillStyle = '#311B92';
                ctx.beginPath();
                ctx.ellipse(e.x + e.w/2, e.y + e.h/2, e.w/2, e.h/2, 0, 0, Math.PI * 2);
                ctx.fill();
                ctx.strokeStyle = '#000'; ctx.lineWidth = 1.5; ctx.stroke();
                ctx.fillStyle = '#311B92';
                ctx.beginPath();
                ctx.moveTo(e.x + 6, e.y + 4); ctx.lineTo(e.x + 4, e.y - 6); ctx.lineTo(e.x + 12, e.y + 2);
                ctx.closePath(); ctx.fill();
                ctx.beginPath();
                ctx.moveTo(e.x + e.w - 6, e.y + 4); ctx.lineTo(e.x + e.w - 4, e.y - 6); ctx.lineTo(e.x + e.w - 12, e.y + 2);
                ctx.closePath(); ctx.fill();
                ctx.fillStyle = '#FFEB3B';
                ctx.beginPath();
                ctx.arc(e.x + 8, e.y + 9, 3, 0, Math.PI * 2);
                ctx.arc(e.x + e.w - 8, e.y + 9, 3, 0, Math.PI * 2);
                ctx.fill();
                ctx.fillStyle = '#000';
                ctx.beginPath();
                ctx.arc(e.x + 8, e.y + 9, 1.5, 0, Math.PI * 2);
                ctx.arc(e.x + e.w - 8, e.y + 9, 1.5, 0, Math.PI * 2);
                ctx.fill();
                ctx.fillStyle = '#FFF';
                ctx.fillRect(e.x + 9, e.y + e.h - 6, 3, 4);
                ctx.fillRect(e.x + e.w - 12, e.y + e.h - 6, 3, 4);
            }
        }

        function drawBoss() {
            if (!boss.active) return;
            const flash = boss.hurtTimer > 0 && Math.floor(boss.hurtTimer / 4) % 2 === 0;
            ctx.fillStyle = flash ? '#FFFFFF' : '#C62828';
            ctx.fillRect(boss.x, boss.y, boss.w, boss.h);
            ctx.strokeStyle = '#4A0000'; ctx.lineWidth = 3; ctx.strokeRect(boss.x, boss.y, boss.w, boss.h);
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
            ctx.strokeStyle = '#000'; ctx.lineWidth = 3;
            ctx.beginPath();
            ctx.moveTo(boss.x + 10, boss.y + 16); ctx.lineTo(boss.x + 34, boss.y + 24);
            ctx.moveTo(boss.x + boss.w - 10, boss.y + 16); ctx.lineTo(boss.x + boss.w - 34, boss.y + 24);
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
            ctx.strokeStyle = '#FFF'; ctx.lineWidth = 2; ctx.strokeRect(barX, barY, barW, barH);
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
                ctx.beginPath(); ctx.arc(p.x, p.y, p.r * 2, 0, Math.PI*2); ctx.fill();
                ctx.fillStyle = '#FF5722';
                ctx.beginPath(); ctx.arc(p.x, p.y, p.r, 0, Math.PI*2); ctx.fill();
                ctx.fillStyle = '#FFEB3B';
                ctx.beginPath(); ctx.arc(p.x - 2, p.y - 2, p.r * 0.5, 0, Math.PI*2); ctx.fill();
            }
        }

        function drawEnemyProjectiles() {
            for (const p of enemyProjectiles) {
                const g = ctx.createRadialGradient(p.x, p.y, 0, p.x, p.y, p.r * 2);
                g.addColorStop(0, 'rgba(255, 100, 0, 0.9)');
                g.addColorStop(1, 'rgba(200, 0, 0, 0)');
                ctx.fillStyle = g;
                ctx.beginPath(); ctx.arc(p.x, p.y, p.r * 2, 0, Math.PI * 2); ctx.fill();
                ctx.fillStyle = '#E53935';
                ctx.beginPath(); ctx.arc(p.x, p.y, p.r, 0, Math.PI * 2); ctx.fill();
                ctx.fillStyle = '#FFEB3B';
                ctx.beginPath(); ctx.arc(p.x - 1, p.y - 1, p.r * 0.45, 0, Math.PI * 2); ctx.fill();
            }
        }

        function drawCelebration() {
            if (!celebration.active) return;
            const centerX = W / 2, centerY = 120;
            ctx.save();
            ctx.globalCompositeOperation = 'lighter';
            for (const r of celebration.lightRays) {
                const alpha = (r.life / r.maxLife) * 0.55;
                const angle = r.angle + celebration.lightRayAngle;
                ctx.save();
                ctx.translate(centerX, centerY);
                ctx.rotate(angle);
                const grad = ctx.createLinearGradient(0, 0, r.length, 0);
                grad.addColorStop(0, 'rgba(255, 255, 200, ' + alpha + ')');
                grad.addColorStop(0.4, 'rgba(255, 220, 100, ' + (alpha * 0.7) + ')');
                grad.addColorStop(1, 'rgba(255, 180, 50, 0)');
                ctx.fillStyle = grad;
                ctx.beginPath();
                ctx.moveTo(0, -r.width / 2);
                ctx.lineTo(r.length, -r.width * 0.15);
                ctx.lineTo(r.length, r.width * 0.15);
                ctx.lineTo(0, r.width / 2);
                ctx.closePath(); ctx.fill();
                ctx.restore();
            }
            ctx.restore();
            for (const d of celebration.bossDebris) {
                const alpha = Math.min(1, d.life / 30);
                ctx.globalAlpha = alpha;
                ctx.save();
                ctx.translate(d.x, d.y);
                ctx.rotate(d.rotation);
                ctx.fillStyle = 'rgba(0,0,0,0.2)';
                ctx.fillRect(-d.w / 2 + 2, -d.h / 2 + 2, d.w, d.h);
                ctx.fillStyle = d.color;
                ctx.fillRect(-d.w / 2, -d.h / 2, d.w, d.h);
                ctx.strokeStyle = 'rgba(0,0,0,0.4)'; ctx.lineWidth = 2;
                ctx.strokeRect(-d.w / 2, -d.h / 2, d.w, d.h);
                ctx.fillStyle = 'rgba(255,255,255,0.25)';
                ctx.fillRect(-d.w / 2, -d.h / 2, d.w * 0.4, d.h * 0.3);
                ctx.restore();
            }
            ctx.globalAlpha = 1;
            for (const f of celebration.fireworks) {
                const alpha = f.life / f.maxLife;
                ctx.globalAlpha = alpha;
                ctx.fillStyle = f.color;
                ctx.beginPath(); ctx.arc(f.x, f.y, f.size * alpha, 0, Math.PI * 2); ctx.fill();
                ctx.globalAlpha = alpha * 0.6;
                ctx.fillStyle = '#fff';
                ctx.beginPath(); ctx.arc(f.x - 1, f.y - 1, f.size * alpha * 0.4, 0, Math.PI * 2); ctx.fill();
            }
            ctx.globalAlpha = 1;
            for (const c of celebration.confetti) {
                const alpha = Math.min(1, c.life / 60);
                ctx.globalAlpha = alpha;
                ctx.save();
                ctx.translate(c.x, c.y);
                ctx.rotate(c.rotation);
                ctx.fillStyle = c.color;
                ctx.fillRect(-c.w / 2, -c.h / 2, c.w, c.h);
                ctx.strokeStyle = 'rgba(0,0,0,0.15)'; ctx.lineWidth = 1;
                ctx.strokeRect(-c.w / 2, -c.h / 2, c.w, c.h);
                ctx.restore();
            }
            ctx.globalAlpha = 1;
            if (celebration.textAlpha > 0) {
                const txt = t('bossDefeated');
                ctx.save();
                ctx.translate(W / 2, H / 2 - 50);
                ctx.scale(celebration.textScale, celebration.textScale);
                ctx.globalAlpha = celebration.textAlpha;
                ctx.font = 'bold 72px Arial';
                ctx.textAlign = 'center';
                ctx.textBaseline = 'middle';
                ctx.lineWidth = 12;
                ctx.strokeStyle = '#0D47A1';
                ctx.strokeText(txt, 0, 0);
                const grad = ctx.createLinearGradient(0, -40, 0, 40);
                grad.addColorStop(0, '#FFEB3B');
                grad.addColorStop(0.5, '#FFD700');
                grad.addColorStop(1, '#FF9800');
                ctx.fillStyle = grad;
                ctx.fillText(txt, 0, 0);
                ctx.font = 'bold 32px Arial';
                ctx.lineWidth = 8;
                ctx.strokeStyle = '#0D47A1';
                ctx.strokeText('+100 ' + (currentLang === 'en' ? 'POINTS' : 'ОЧКОВ'), 0, 70);
                ctx.fillStyle = '#FFFFFF';
                ctx.fillText('+100 ' + (currentLang === 'en' ? 'POINTS' : 'ОЧКОВ'), 0, 70);
                ctx.restore();
            }
        }

        function drawGameOver() {
            if (!gameOver.active) return;
            for (const d of gameOver.debris) {
                const alpha = Math.min(1, d.life / 30);
                ctx.globalAlpha = alpha;
                ctx.save();
                ctx.translate(d.x, d.y);
                ctx.rotate(d.rotation);
                ctx.fillStyle = 'rgba(0,0,0,0.25)';
                ctx.fillRect(-d.w / 2 + 2, -d.h / 2 + 2, d.w, d.h);
                ctx.fillStyle = d.color;
                ctx.fillRect(-d.w / 2, -d.h / 2, d.w, d.h);
                ctx.strokeStyle = 'rgba(0,0,0,0.4)'; ctx.lineWidth = 1.5;
                ctx.strokeRect(-d.w / 2, -d.h / 2, d.w, d.h);
                ctx.fillStyle = 'rgba(255,255,255,0.3)';
                ctx.fillRect(-d.w / 2, -d.h / 2, d.w * 0.35, d.h * 0.3);
                ctx.restore();
            }
            ctx.globalAlpha = 1;
            if (gameOver.darkAlpha > 0) {
                ctx.fillStyle = 'rgba(0,0,0,' + gameOver.darkAlpha + ')';
                ctx.fillRect(-50, -50, W + 100, H + 100);
            }
            if (gameOver.flashAlpha > 0) {
                ctx.fillStyle = 'rgba(255,30,30,' + gameOver.flashAlpha + ')';
                ctx.fillRect(-50, -50, W + 100, H + 100);
            }
            if (gameOver.textAlpha > 0) {
                ctx.save();
                ctx.translate(W / 2, H / 2);
                ctx.scale(gameOver.textScale, gameOver.textScale);
                ctx.globalAlpha = gameOver.textAlpha;
                const txt = t('gameOver');
                ctx.font = 'bold 88px Arial';
                ctx.textAlign = 'center';
                ctx.textBaseline = 'middle';
                ctx.lineWidth = 14;
                ctx.strokeStyle = '#000';
                ctx.strokeText(txt, 0, 0);
                ctx.lineWidth = 8;
                ctx.strokeStyle = '#7F0000';
                ctx.strokeText(txt, 0, 0);
                const textGrad = ctx.createLinearGradient(0, -50, 0, 50);
                textGrad.addColorStop(0, '#FF5252');
                textGrad.addColorStop(0.5, '#E53935');
                textGrad.addColorStop(1, '#B71C1C');
                ctx.fillStyle = textGrad;
                ctx.fillText(txt, 0, 0);
                ctx.restore();
            }
            if (gameOver.phraseAlpha > 0) {
                ctx.save();
                ctx.translate(W / 2, H / 2 + 80);
                ctx.scale(gameOver.phraseScale, gameOver.phraseScale);
                ctx.globalAlpha = gameOver.phraseAlpha;
                ctx.font = 'bold 32px Arial';
                ctx.textAlign = 'center';
                ctx.textBaseline = 'middle';
                ctx.lineWidth = 8;
                ctx.strokeStyle = 'rgba(0,0,0,0.7)';
                ctx.strokeText(gameOver.phrase, 0, 0);
                const phraseGrad = ctx.createLinearGradient(0, -20, 0, 20);
                phraseGrad.addColorStop(0, '#FFF59D');
                phraseGrad.addColorStop(1, '#FFC107');
                ctx.fillStyle = phraseGrad;
                ctx.fillText(gameOver.phrase, 0, 0);
                ctx.restore();
            }
            if (gameOver.textAlpha > 0 && Math.floor(Date.now() / 400) % 2 === 0) {
                ctx.save();
                ctx.globalAlpha = gameOver.textAlpha * 0.9;
                ctx.font = 'bold 22px Arial';
                ctx.textAlign = 'center';
                ctx.lineWidth = 6;
                ctx.strokeStyle = '#000';
                ctx.strokeText(t('restart'), W / 2, H - 30);
                ctx.fillStyle = '#FFFFFF';
                ctx.fillText(t('restart'), W / 2, H - 30);
                ctx.restore();
            }
            if (gameOver.statsAlpha > 0 && gameOver.stats) {
                const panelW = 340, panelH = 270;
                const panelX = W / 2 - panelW / 2;
                const panelY = 60 - gameOver.statsSlide;
                ctx.save();
                ctx.globalAlpha = gameOver.statsAlpha;
                ctx.fillStyle = 'rgba(15, 20, 40, 0.85)';
                ctx.beginPath();
                if (ctx.roundRect) ctx.roundRect(panelX, panelY, panelW, panelH, 16);
                else ctx.rect(panelX, panelY, panelW, panelH);
                ctx.fill();
                ctx.strokeStyle = '#FFC107'; ctx.lineWidth = 3;
                ctx.beginPath();
                if (ctx.roundRect) ctx.roundRect(panelX, panelY, panelW, panelH, 16);
                else ctx.rect(panelX, panelY, panelW, panelH);
                ctx.stroke();
                ctx.font = 'bold 22px Arial';
                ctx.textAlign = 'center';
                ctx.fillStyle = '#FFC107';
                ctx.fillText('📊 ' + t('statsTitle'), W / 2, panelY + 35);
                ctx.strokeStyle = 'rgba(255,193,7,0.4)'; ctx.lineWidth = 1;
                ctx.beginPath();
                ctx.moveTo(panelX + 20, panelY + 50);
                ctx.lineTo(panelX + panelW - 20, panelY + 50);
                ctx.stroke();
                const rowY = panelY + 80;
                const rowGap = 30;
                const labelX = panelX + 25;
                const valueX = panelX + panelW - 25;
                ctx.font = '17px Arial';
                ctx.textAlign = 'left';
                ctx.fillStyle = '#E0E0E0';
                ctx.fillText('🪙 ' + t('statsCoins'), labelX, rowY);
                ctx.fillText('⭐ ' + t('statsScore'), labelX, rowY + rowGap);
                ctx.fillText('💀 ' + t('statsDeaths'), labelX, rowY + rowGap * 2);
                ctx.fillText('⏱ ' + t('statsTime'), labelX, rowY + rowGap * 3);
                ctx.fillText('📍 ' + t('statsLevel'), labelX, rowY + rowGap * 4);
                ctx.textAlign = 'right';
                ctx.font = 'bold 18px Arial';
                ctx.fillStyle = '#FFD700';
                ctx.fillText(gameOver.stats.coins + ' / ' + TOTAL_COINS, valueX, rowY);
                ctx.fillStyle = '#81D4FA';
                ctx.fillText(String(gameOver.stats.score), valueX, rowY + rowGap);
                ctx.fillStyle = '#FF8A80';
                ctx.fillText(String(gameOver.stats.deaths), valueX, rowY + rowGap * 2);
                ctx.fillStyle = '#A5D6A7';
                ctx.fillText(gameOver.stats.time, valueX, rowY + rowGap * 3);
                ctx.fillStyle = '#CE93D8';
                ctx.fillText(gameOver.stats.level, valueX, rowY + rowGap * 4);
                ctx.strokeStyle = 'rgba(255,193,7,0.3)'; ctx.lineWidth = 1;
                ctx.beginPath();
                ctx.moveTo(panelX + 20, rowY + rowGap * 4 + 18);
                ctx.lineTo(panelX + panelW - 20, rowY + rowGap * 4 + 18);
                ctx.stroke();
                const recordY = rowY + rowGap * 5;
                const pulse = gameOver.isNewRecord ? 1 + Math.sin(gameOver.recordGlow) * 0.15 : 1;
                ctx.font = 'bold 18px Arial';
                ctx.textAlign = 'left';
                if (gameOver.isNewRecord) {
                    const glowAlpha = 0.7 + Math.sin(gameOver.recordGlow * 2) * 0.3;
                    ctx.fillStyle = `rgba(255, 215, 0, ${glowAlpha})`;
                    ctx.fillText('🏅 ' + t('newRecord'), labelX, recordY);
                } else {
                    ctx.fillStyle = '#B0BEC5';
                    ctx.fillText('🏅 ' + t('statsRecord'), labelX, recordY);
                }
                ctx.save();
                ctx.translate(valueX, recordY);
                ctx.scale(pulse, pulse);
                ctx.textAlign = 'right';
                ctx.font = 'bold 20px Arial';
                ctx.fillStyle = gameOver.isNewRecord ? '#FFD700' : '#FFECB3';
                ctx.fillText(String(bestScore), 0, 0);
                if (gameOver.isNewRecord) {
                    const glowAlpha = 0.4 + Math.sin(gameOver.recordGlow * 2) * 0.3;
                    ctx.shadowColor = `rgba(255, 215, 0, ${glowAlpha})`;
                    ctx.shadowBlur = 20;
                    ctx.fillText(String(bestScore), 0, 0);
                    ctx.shadowBlur = 0;
                }
                ctx.restore();
                ctx.restore();
            }
        }

        // ============================================
        // СТАРТ
        // ============================================
        loadLevel(0);
        applyLocalization();

        document.addEventListener('visibilitychange', () => {
            if (document.hidden) Sound.pauseAll();
            else if (!paused) Sound.resumeAll();
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
