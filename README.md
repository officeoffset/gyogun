# gyogun<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>極・海物語 魚群シミュレーター</title>
    <style>
        * {
            box-sizing: border-box;
            margin: 0;
            padding: 0;
            user-select: none;
            -webkit-user-select: none;
        }
        body, html {
            width: 100%;
            height: 100%;
            overflow: hidden;
            background: linear-gradient(to bottom, #005ca5, #001f4d);
            font-family: 'Helvetica Neue', Arial, sans-serif;
        }
        /* 海背景の泡アニメーション */
        .bubble {
            position: absolute;
            background: rgba(255, 255, 255, 0.3);
            border-radius: 50%;
            bottom: -50px;
            animation: rise 8s infinite linear;
            pointer-events: none;
        }
        @keyframes rise {
            0% { transform: translateY(0) translateX(0); opacity: 0; }
            10% { opacity: 0.6; }
            90% { opacity: 0.6; }
            100% { transform: translateY(-110vh) translateX(50px); opacity: 0; }
        }
        /* 魚群のコンテナ */
        #fish-field {
            position: absolute;
            width: 100%;
            height: 100%;
            top: 0;
            left: 0;
            overflow: hidden;
            pointer-events: none;
            z-index: 1;
        }
        /* 魚1匹のスタイル（CSSアニメーションで動かす） */
        .fish {
            position: absolute;
            font-size: 38px; /* 少し大きくして迫力を出す */
            line-height: 1;
            white-space: nowrap;
            display: inline-block;
            transform: scaleX(-1); /* 左向き */
            pointer-events: none;
            will-change: transform; /* GPUネイティブ加速 */
            animation: swimLeft 1.6s linear forwards; /* 1.6秒で高速通過 */
        }

        @keyframes swimLeft {
            0% {
                transform: translateX(0) scaleX(-1);
            }
            100% {
                /* 画面左外（画面幅 + 魚群全体の長さ分）へ一気に移動 */
                transform: translateX(-2500px) scaleX(-1);
            }
        }

        /* コントロールパネル */
        .controls {
            position: absolute;
            top: 20px;
            left: 50%;
            transform: translateX(-50%);
            display: flex;
            flex-direction: column;
            gap: 12px;
            width: 85%;
            max-width: 340px;
            z-index: 10;
            background: rgba(0, 0, 0, 0.6);
            padding: 16px;
            border-radius: 16px;
            box-shadow: 0 8px 32px rgba(0, 0, 0, 0.5);
            border: 1px solid rgba(255, 255, 255, 0.2);
            backdrop-filter: blur(8px);
            -webkit-backdrop-filter: blur(8px);
        }
        button {
            width: 100%;
            padding: 14px;
            border: none;
            border-radius: 12px;
            font-size: 16px;
            font-weight: bold;
            color: white;
            cursor: pointer;
            transition: all 0.2s ease;
            box-shadow: 0 4px 12px rgba(0,0,0,0.2);
        }
        #btn-trigger {
            background: linear-gradient(135deg, #ff3b30, #ff9500);
            border-bottom: 4px solid #cc2d24;
            text-shadow: 1px 1px 2px rgba(0,0,0,0.4);
        }
        #btn-trigger:active {
            transform: translateY(2px);
            border-bottom-width: 2px;
            box-shadow: 0 2px 6px rgba(0,0,0,0.3);
        }
        #btn-auto {
            background: linear-gradient(135deg, #444, #222);
            border-bottom: 4px solid #111;
        }
        #btn-auto.active {
            background: linear-gradient(135deg, #4cd964, #5ac8fa);
            border-bottom: 4px solid #38b04a;
        }
        #btn-auto:active {
            transform: translateY(2px);
            border-bottom-width: 2px;
        }
        .status-text {
            color: #fff;
            font-size: 13px;
            font-weight: bold;
            text-align: center;
            margin-top: 4px;
            letter-spacing: 0.5px;
        }
    </style>
</head>
<body>

    <div id="bubble-bg"></div>
    <div id="fish-field"></div>

    <div class="controls">
        <button id="btn-trigger">🐋 魚群を流す (手動)</button>
        <button id="btn-auto">🔄 自動モード: OFF</button>
        <div class="status-text" id="status-display">待機中...</div>
    </div>

<script>
    const fishField = document.getElementById('fish-field');
    const bubbleBg = document.getElementById('bubble-bg');
    const btnTrigger = document.getElementById('btn-trigger');
    const btnAuto = document.getElementById('btn-auto');
    const statusDisplay = document.getElementById('status-display');
    
    // パチンコ海物語でおなじみの主要なメンツ
    const fishTypes = ['🐟', '🐠', '🐡', '🐙', '🦑', '🦐', '🦀', '🦈', '🐬', '🐋'];
    
    let isRunning = false;
    let isAutoMode = false;
    let autoTimeout = null;

    // 背景の泡生成
    function createBubbles() {
        for (let i = 0; i < 20; i++) {
            const bubble = document.createElement('div');
            bubble.classList.add('bubble');
            const size = Math.random() * 25 + 5;
            bubble.style.width = `${size}px`;
            bubble.style.height = `${size}px`;
            bubble.style.left = `${Math.random() * 100}%`;
            bubble.style.animationDelay = `${Math.random() * 8}s`;
            bubble.style.animationDuration = `${Math.random() * 5 + 5}s`;
            bubbleBg.appendChild(bubble);
        }
    }
    createBubbles();

    // 魚群生成・アニメーションロジック
    function spawnFishSwarm() {
        if (isRunning) return;
        isRunning = true;
        statusDisplay.innerText = "🚨 魚群通過中（激熱）！";

        const totalRows = 8;     // 列数を増やして画面全体をカバー
        const fishPerRow = 28;   // 1列あたりの数を大幅に増やして「超密集」にする
        const screenWidth = window.innerWidth;
        const screenHeight = window.innerHeight;

        for (let row = 0; row < totalRows; row++) {
            const baseY = (screenHeight / (totalRows + 1)) * (row + 1);
            
            for (let col = 0; col < fishPerRow; col++) {
                const fish = document.createElement('div');
                fish.classList.add('fish');
                
                // 【再現ポイント】単一の種類ではなく、複数の魚を混ぜてカラフルな魚群にする
                fish.innerText = fishTypes[Math.floor(Math.random() * fishTypes.length)];

                // 本物っぽいウェーブのうねり（サイン波）を計算
                const waveOffset = Math.sin(col * 0.5) * 30;
                const yPos = baseY + waveOffset + (Math.random() * 16 - 8);
                
                // 初期位置：画面の右外側にギッチリ詰めて配置（間隔を28pxに縮小）
                const startX = screenWidth + (col * 28) + (Math.random() * 10);
                
                fish.style.top = `${yPos}px`;
                fish.style.left = `${startX}px`;
                
                // アニメーションの開始タイミングをわずかにずらして、群れのリアリティを出す
                fish.style.animationDelay = `${col * 0.01}s`;

                fishField.appendChild(fish);

                // アニメーション終了時に要素を削除
                fish.addEventListener('animationend', () => {
                    fish.remove();
                });
            }
        }

        // 魚群が完全にハケるタイミング（アニメーション1.6秒 + α）でロック解除
        setTimeout(() => {
            isRunning = false;
            statusDisplay.innerText = isAutoMode ? "🔄 自動巡回中..." : "待機中...";
        }, 2200);
    }

    function autoLoop() {
        if (!isAutoMode) return;
        const randomDelay = Math.random() * 3000 + 3000; // 3〜6秒
        autoTimeout = setTimeout(() => {
            if (!isRunning) {
                spawnFishSwarm();
            }
            autoLoop();
        }, randomDelay);
    }

    btnTrigger.addEventListener('click', () => {
        if (!isRunning) spawnFishSwarm();
    });

    btnAuto.addEventListener('click', () => {
        isAutoMode = !isAutoMode;
        if (isAutoMode) {
            btnAuto.classList.add('active');
            btnAuto.innerText = "🔄 自動モード: ON";
            statusDisplay.innerText = "🔄 自動巡回中...";
            autoLoop();
        } else {
            btnAuto.classList.remove('active');
            btnAuto.innerText = "🔄 自動モード: OFF";
            statusDisplay.innerText = "待機中...";
            clearTimeout(autoTimeout);
        }
    });
</script>
</body>
</html>
