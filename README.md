# index.html<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Mastermind: Online Battle</title>
    <script src="https://unpkg.com/peerjs@1.5.2/dist/peerjs.min.js"></script>
    <style>
        :root {
            --primary: #6366f1;
            --strike: #ef4444;
            --ball: #3b82f6;
            --bg: #f8fafc;
            --enemy: #f59e0b;
        }

        body { font-family: 'Segoe UI', sans-serif; background: var(--bg); display: flex; justify-content: center; padding: 20px; margin: 0; }
        .app-container { background: white; padding: 24px; border-radius: 20px; box-shadow: 0 10px 25px rgba(0,0,0,0.05); width: 100%; max-width: 450px; text-align: center; }
        
        .status-badge { padding: 5px 10px; border-radius: 10px; font-size: 0.8rem; margin-bottom: 15px; display: inline-block; }
        .online { background: #dcfce7; color: #166534; }
        .offline { background: #fee2e2; color: #991b1b; }

        .connection-area { background: #f1f5f9; padding: 15px; border-radius: 12px; margin-bottom: 20px; font-size: 0.9rem; }
        .my-id { font-weight: bold; color: var(--primary); cursor: pointer; text-decoration: underline; }

        .turn-indicator { font-size: 1.2rem; font-weight: bold; margin: 15px 0; padding: 10px; border-radius: 10px; }
        .your-turn { background: var(--primary); color: white; }
        .enemy-turn { background: #e2e8f0; color: #64748b; }

        .input-group { display: flex; gap: 10px; margin-bottom: 20px; }
        input[type="text"] { flex: 1; padding: 12px; font-size: 1.5rem; letter-spacing: 0.3rem; text-align: center; border: 2px solid #e2e8f0; border-radius: 12px; outline: none; }
        button { padding: 12px 24px; font-weight: bold; background: var(--primary); color: white; border: none; border-radius: 12px; cursor: pointer; transition: 0.2s; }
        button:disabled { background: #cbd5e1; cursor: not-allowed; }

        .log { max-height: 200px; overflow-y: auto; border-top: 2px solid #f1f5f9; margin-top: 10px; }
        .log-item { display: flex; justify-content: space-between; align-items: center; padding: 8px; border-bottom: 1px solid #f1f5f9; font-size: 0.9rem; }
        .badge { padding: 4px 8px; border-radius: 6px; color: white; font-size: 0.7rem; font-weight: bold; margin-left: 5px; }
        .bg-strike { background: var(--strike); }
        .bg-ball { background: var(--ball); }
        .tag { font-size: 0.6rem; padding: 2px 5px; border-radius: 4px; margin-right: 5px; color: white; }
        .tag-me { background: var(--primary); }
        .tag-enemy { background: var(--enemy); }
    </style>
</head>
<body>

<div class="app-container">
    <h2>Strike Battle Online</h2>
    
    <div id="connectionArea" class="connection-area">
        <p>あなたのID: <span id="myId" class="my-id">読み込み中...</span></p>
        <div style="display: flex; gap:5px; margin-top:10px;">
            <input type="text" id="peerIdInput" placeholder="相手のIDを入力" style="font-size: 0.9rem; letter-spacing: 0;">
            <button onclick="connectToPeer()" id="connectBtn" style="padding: 8px 15px;">接続</button>
        </div>
    </div>

    <div id="status" class="status-badge offline">未接続</div>
    <div id="turnBox" class="turn-indicator" style="display:none;">待機中...</div>

    <div id="gameArea" style="display:none;">
        <div class="input-group">
            <input type="text" id="guessInput" inputmode="numeric" placeholder="????" maxlength="4">
            <button onclick="makeGuess()" id="guessBtn">予想</button>
        </div>
        <div class="log" id="gameLog"></div>
    </div>
</div>

<script>
    let peer;
    let conn;
    let myAnswer = [];
    let isMyTurn = false;
    const digitLength = 4;

    // --- 通信初期化 ---
    function initPeer() {
        peer = new Peer();
        
        peer.on('open', (id) => {
            document.getElementById('myId').innerText = id;
            document.getElementById('myId').onclick = () => {
                navigator.clipboard.writeText(id);
                alert("IDをコピーしました！友達に送ってください。");
            };
        });

        peer.on('connection', (c) => {
            if (conn) return; // 既に接続済みなら無視
            conn = c;
            setupChat();
            alert("対戦相手が接続しました！");
        });
    }

    function connectToPeer() {
        const remoteId = document.getElementById('peerIdInput').value;
        if (!remoteId) return;
        conn = peer.connect(remoteId);
        setupChat();
    }

    function setupChat() {
        conn.on('open', () => {
            document.getElementById('status').innerText = "接続済み";
            document.getElementById('status').className = "status-badge online";
            document.getElementById('connectionArea').style.display = "none";
            document.getElementById('gameArea').style.display = "block";
            document.getElementById('turnBox').style.display = "block";
            
            // 最初に接続を仕掛けた方が後攻、受けた方が先攻とする
            startGame(conn.serialization === 'json'); 
        });

        conn.on('data', (data) => {
            if (data.type === 'guess') {
                handleEnemyGuess(data.val);
            } else if (data.type === 'result') {
                handleResult(data);
            }
        });
    }

    // --- ゲームロジック ---
    function startGame(isHost) {
        myAnswer = generateAnswer();
        isMyTurn = isHost;
        updateTurnUI();
    }

    function generateAnswer() {
        const nums = [1, 2, 3, 4, 5, 6, 7, 8, 9];
        const res = [];
        for (let i = 0; i < digitLength; i++) {
            const idx = Math.floor(Math.random() * nums.length);
            res.push(nums.splice(idx, 1)[0]);
        }
        console.log("あなたの守る数字:", res.join(""));
        return res;
    }

    function updateTurnUI() {
        const box = document.getElementById('turnBox');
        const btn = document.getElementById('guessBtn');
        if (isMyTurn) {
            box.innerText = "あなたのターン";
            box.className = "turn-indicator your-turn";
            btn.disabled = false;
        } else {
            box.innerText = "相手のターン...";
            box.className = "turn-indicator enemy-turn";
            btn.disabled = true;
        }
    }

    function makeGuess() {
        const input = document.getElementById('guessInput');
        const val = input.value;
        if (val.length !== digitLength) return;

        // 相手に予想を送る
        conn.send({ type: 'guess', val: val });
        isMyTurn = false;
        updateTurnUI();
        input.value = "";
    }

    function handleEnemyGuess(enemyVal) {
        const guess = enemyVal.split('').map(Number);
        let s = 0, b = 0;
        
        guess.forEach((num, i) => {
            if (num === myAnswer[i]) s++;
            else if (myAnswer.includes(num)) b++;
        });

        // 結果を自分に表示し、相手にも送る
        displayLog(enemyVal, s, b, false);
        conn.send({ type: 'result', val: enemyVal, s: s, b: b });

        if (s === digitLength) {
            alert("負けました...！ 相手が正解しました。");
            location.reload();
        } else {
            isMyTurn = true;
            updateTurnUI();
        }
    }

    function handleResult(data) {
        displayLog(data.val, data.s, data.b, true);
        if (data.s === digitLength) {
            alert("勝利！！ おめでとうございます！");
            location.reload();
        }
    }

    function displayLog(guess, s, b, isMe) {
        const log = document.getElementById('gameLog');
        const item = document.createElement('div');
        item.className = 'log-item';
        const tag = isMe ? '<span class="tag tag-me">YOU</span>' : '<span class="tag tag-enemy">ENEMY</span>';
        item.innerHTML = `
            <div>${tag}<strong>${guess}</strong></div>
            <div>
                <span class="badge bg-strike">${s} S</span>
                <span class="badge bg-ball">${b} B</span>
            </div>
        `;
        log.prepend(item);
    }

    initPeer();
</script>

</body>
</html>
