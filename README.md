<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>地理位置測驗系統 (教室版)</title>
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <link href="https://fonts.googleapis.com/css2?family=Cinzel:wght@700&family=Lato:wght@400;700&display=swap" rel="stylesheet">
    
    <style>
        /* --- 淺色系/現代風格 Design System --- */
        :root {
            --bg-light: #f9f9f9;
            --primary-dark: #1f3355; /* 深藍色 (主要文字/邊框) */
            --accent-cyan: #00aaff; /* 亮青色 (強調/互動) */
            --accent-gold: #c8aa6e; /* 淺金色 */
            --text-dark: #333;
        }

        body, html { margin: 0; padding: 0; height: 100%; overflow: hidden; background: var(--bg-light); font-family: 'Lato', sans-serif; color: var(--text-dark); }
        h1, h2, h3, button { font-family: 'Cinzel', serif; letter-spacing: 0.5px; }

        /* --- 登入畫面 --- */
        #login-screen {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            background: linear-gradient(135deg, #f0f0f5 0%, #fff 100%);
            display: flex; justify-content: center; align-items: center; z-index: 3000;
        }
        .login-card {
            width: 90%; max-width: 400px; padding: 40px;
            background: #fff; border: 2px solid var(--primary-dark);
            box-shadow: 0 10px 30px rgba(0,0,0,0.1);
            text-align: center; border-radius: 8px;
        }
        .input-group label { color: var(--primary-dark); font-weight: 700; }
        .input-group input {
            border: 1px solid #ccc; color: var(--primary-dark);
            background: #fff; border-radius: 4px;
        }
        .input-group input:focus { border-color: var(--accent-cyan); box-shadow: 0 0 5px rgba(0, 170, 255, 0.3); }

        /* --- 主要佈局 --- */
        #game-container { display: flex; width: 100vw; height: 100vh; transition: filter 1s; }
        #map { flex: 1; height: 100%; background: #e0f7ff; }

        /* 右側選單/側邊欄 (Sidebar) */
        #sidebar {
            width: 280px;
            background: #fff;
            border-left: 1px solid #ddd;
            display: flex; flex-direction: column;
            box-shadow: -3px 0 15px rgba(0,0,0,0.05);
            z-index: 2000;
            overflow-y: auto;
        }
        .user-info-display {
            padding: 15px; border-bottom: 1px solid #eee;
            color: var(--primary-dark); font-size: 14px;
            background: var(--bg-light);
        }
        .q-btn {
            width: 100%; padding: 12px; margin-bottom: 5px;
            background: #f0f0f0; border: none;
            color: var(--text-dark); text-align: left; cursor: pointer;
            transition: 0.3s; border-radius: 4px;
        }
        .q-btn:hover { background: #e0e0e0; }
        .q-btn.active { background: var(--accent-cyan); color: #fff; font-weight: 700; }
        .q-btn.locked { background: #d0f0d0; color: #555; } /* 鎖定後變淺綠 */
        .q-btn.locked::after { content: '✓'; right: 10px; font-weight: bold; }

        /* 底部題目面板 (Question Panel) */
        #question-panel {
            position: absolute; bottom: 0; left: 0; right: 280px; 
            background: #fff; border-top: 1px solid #ddd;
            padding: 15px 20px;
            color: var(--text-dark); z-index: 1000;
            display: flex; justify-content: space-between; align-items: center;
        }
        
        /* 通用按鈕樣式 */
        .hex-btn {
            background: var(--primary-dark); color: #fff;
            border: none; padding: 10px 20px;
            font-size: 14px; cursor: pointer; border-radius: 4px;
            transition: 0.3s;
        }
        .hex-btn:hover { background: var(--accent-cyan); box-shadow: 0 2px 10px rgba(0, 170, 255, 0.5); }
        .hex-btn.gold { background: var(--accent-gold); color: var(--primary-dark); }
        .hex-btn.gold:hover { background: #dcd0a0; }
        .hex-btn:disabled { background: #ccc; color: #888; cursor: not-allowed; }

        /* --- 行動裝置優化 (小於 768px) --- */
        @media (max-width: 768px) {
            #sidebar {
                position: absolute; top: 0; left: 0; right: 0;
                width: 100%; height: 200px; 
                border-left: none; border-bottom: 1px solid #ddd;
                order: -1; /* 將 sidebar 放在最上面 */
                overflow: hidden;
            }
            #game-container { flex-direction: column; }
            #question-panel {
                position: static; 
                width: 100%;
                right: 0;
                order: 1; /* 將 question panel 放在最下面 */
                border-top: none;
            }
            #map { flex: 1; height: calc(100vh - 270px); } /* 扣除上下面板高度 */
        }
    </style>
</head>
<body>

    <div id="loading-overlay" style="display: none;">正在載入地圖資料...</div>

    <div id="login-screen">
        <div class="login-card">
            <h1 style="color: var(--primary-dark); margin-bottom: 30px;">地理位置測驗</h1>
            <div class="input-group">
                <label>班級</label><input type="text" id="in-class" placeholder="Ex: 801">
            </div>
            <div class="input-group">
                <label>座號</label><input type="text" id="in-seat" placeholder="Ex: 15">
            </div>
            <div class="input-group">
                <label>姓名</label><input type="text" id="in-name" placeholder="請輸入姓名">
            </div>
            <button class="hex-btn" style="width:100%; margin-top:10px;" onclick="startGame()">開始測驗</button>
        </div>
    </div>

    <div id="game-container">
        <div id="map"></div>

        <div id="sidebar">
            <div class="user-info-display">
                學員：<span id="display-class"></span> <span id="display-name"></span>
            </div>
            <div class="question-list" id="q-list-container">
                </div>
            <div class="action-area">
                <button class="hex-btn gold" onclick="calculateFinalScore()">計算總成績</button>
            </div>
        </div>

        <div id="question-panel">
            <div style="flex: 1;">
                <h3 id="q-title" style="margin:0 0 5px 0; color: var(--accent-cyan);">請選擇題目</h3>
                <div id="q-desc" style="font-size: 16px;">點擊右側選單或上方按鈕開始作答。</div>
                <div id="q-status" style="font-size: 12px; color: #888; margin-top: 5px;"></div>
            </div>
            <button id="lock-btn" class="hex-btn" onclick="lockCurrentAnswer()" disabled>鎖定答案</button>
        </div>
    </div>

    <div id="score-modal" style="display: none; position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.85); z-index: 4000; justify-content: center; align-items: center;">
        <div style="width: 90%; max-width: 500px; background: #fff; border: 4px solid var(--accent-cyan); padding: 40px; text-align: center; border-radius: 8px;">
            <div style="font-size: 32px; color: var(--primary-dark); margin-bottom: 20px;">測驗結果</div>
            <div style="font-size: 14px;">學員: <span id="score-name"></span></div>
            <div style="font-size: 64px; color: var(--accent-cyan); margin: 20px 0; font-weight: bold;" id="final-score">0</div>
            <p id="score-detail" style="color:var(--text-dark);">答對題數: 0/4</p>
            <button class="hex-btn" onclick="location.reload()">重新開始</button>
        </div>
    </div>

    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
    <script>
        // --- 繁體中文名稱對照表 (使用 ISO 3166-1 alpha-3 代碼) ---
        const ZH_NAMES = {
            "TWN": "臺灣", "USA": "美國", "CAN": "加拿大", "JPN": "日本",
            "IND": "印度", "AUS": "澳洲", "MDG": "馬達加斯加",
            // 更多國家請依需求補充
            "CHN": "中國", "RUS": "俄羅斯", "FRA": "法國", "DEU": "德國", "GBR": "英國",
            "BRA": "巴西", "SAU": "沙烏地阿拉伯", "KOR": "韓國"
        };
        
        // --- 遊戲資料與狀態 ---
        let userInfo = { class: "", seat: "", name: "" };
        const questions = [
            { id: 1, title: "題號 01", text: "請在地圖上找出【臺灣】的位置。", answers: ["TWN"], maxSelect: 1, userSelection: new Set(), locked: false },
            { id: 2, title: "題號 02", text: "【四方安全會談 (Quad)】是哪四個國家？", answers: ["USA", "JPN", "IND", "AUS"], maxSelect: 4, userSelection: new Set(), locked: false },
            { id: 3, title: "題號 03", text: "請找出兩個不在歐洲的【北約 (NATO)】創始國。", answers: ["USA", "CAN"], maxSelect: 2, userSelection: new Set(), locked: false },
            { id: 4, title: "題號 04", text: "請找出【馬達加斯加】。", answers: ["MDG"], maxSelect: 1, userSelection: new Set(), locked: false }
        ];

        let currentQIndex = -1; 
        let map, geoJsonLayer;

        // --- 系統初始化與登入 ---
        function startGame() {
            const c = document.getElementById('in-class').value;
            const s = document.getElementById('in-seat').value;
            const n = document.getElementById('in-name').value;

            if(!c || !s || !n) { alert("請完整填寫資料！"); return; }
            userInfo = { class: c, seat: s, name: n };
            
            document.getElementById('display-class').innerText = `${c}-${s}`;
            document.getElementById('display-name').innerText = n;
            
            document.getElementById('login-screen').style.display = 'none';
            document.getElementById('game-container').classList.add('active');

            renderSidebar();
            initMap();
        }

        // --- 地圖核心與互動 ---
        function initMap() {
            map = L.map('map', { center: [20, 120], zoom: 3, zoomControl: true, attributionControl: false, worldCopyJump: true });

            // 使用標準 OpenStreetMap Tiles，因為淺色背景更易讀
            L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', { maxZoom: 19 }).addTo(map);

            // 載入 GeoJSON
            fetch('https://raw.githubusercontent.com/johan/world.geo.json/master/countries.geo.json')
                .then(res => res.json())
                .then(data => {
                    geoJsonLayer = L.geoJSON(data, { style: defaultStyle, onEachFeature: onEachFeature }).addTo(map);
                    map.invalidateSize(); // 確保地圖尺寸正確
                });
        }

        // 未選取狀態的樣式
        function defaultStyle(feature) {
            return { fillColor: '#e0e0e0', weight: 1, opacity: 1, color: '#999', fillOpacity: 0.8 };
        }

        // 綁定點擊與滑鼠事件
        function onEachFeature(feature, layer) {
            // 新增 Tooltip 顯示中文名稱
            const isoId = feature.id;
            const chineseName = ZH_NAMES[isoId] || feature.properties.name;
            layer.bindTooltip(chineseName, { permanent: false, direction: 'auto', className: 'chinese-tooltip' });

            layer.on('mouseover', function(e) {
                if(currentQIndex === -1 || questions[currentQIndex].locked) return;
                
                if(!questions[currentQIndex].userSelection.has(isoId)) {
                    layer.setStyle({ weight: 3, color: 'var(--accent-cyan)', fillOpacity: 0.95 });
                }
            });

            layer.on('mouseout', function(e) {
                if(currentQIndex === -1 || questions[currentQIndex].locked) return;
                if(!questions[currentQIndex].userSelection.has(isoId)) {
                    geoJsonLayer.resetStyle(e.target);
                }
            });

            layer.on('click', function(e) {
                if(currentQIndex === -1) { alert("請先選擇題目！"); return; }
                const q = questions[currentQIndex];
                if(q.locked) { alert("此題已鎖定，無法修改。"); return; }

                if(q.userSelection.has(isoId)) {
                    q.userSelection.delete(isoId);
                } else {
                    if(q.userSelection.size < q.maxSelect) {
                        q.userSelection.add(isoId);
                    } else {
                        alert("已達選擇上限 (Max: " + q.maxSelect + ")"); return;
                    }
                }
                
                updateMapState(); 
                updatePanelUI();
            });
        }

        // 根據當前題目的選擇狀態重繪地圖
        function updateMapState() {
            if(currentQIndex === -1 || !geoJsonLayer) return;
            const q = questions[currentQIndex];

            geoJsonLayer.eachLayer(layer => {
                const id = layer.feature.id;
                if(q.userSelection.has(id)) {
                    layer.setStyle({
                        fillColor: 'var(--accent-cyan)', 
                        color: 'var(--primary-dark)',
                        weight: 3,
                        fillOpacity: 0.95
                    });
                } else {
                    geoJsonLayer.resetStyle(layer);
                }
            });
        }

        // --- 題目與選單邏輯 ---
        function renderSidebar() {
            const container = document.getElementById('q-list-container');
            container.innerHTML = ""; 
            questions.forEach((q, index) => {
                const btn = document.createElement('button');
                btn.className = `q-btn ${index === currentQIndex ? 'active' : ''} ${q.locked ? 'locked' : ''}`;
                btn.innerText = `${q.title}`;
                btn.onclick = () => selectQuestion(index);
                container.appendChild(btn);
            });
        }

        function selectQuestion(index) {
            currentQIndex = index;
            const q = questions[index];

            document.getElementById('q-title').innerText = q.title;
            document.getElementById('q-desc').innerText = q.text;
            document.getElementById('lock-btn').innerText = q.locked ? "✓ 已鎖定" : "鎖定答案";
            document.getElementById('lock-btn').disabled = q.locked;
            
            updateMapState(); 
            updatePanelUI();
            renderSidebar(); 
        }

        function updatePanelUI() {
            if(currentQIndex === -1) return;
            const q = questions[currentQIndex];
            document.getElementById('q-status').innerText = `已選: ${q.userSelection.size} / ${q.maxSelect}`;
        }

        function lockCurrentAnswer() {
            if(currentQIndex === -1) return;
            const q = questions[currentQIndex];

            if(q.userSelection.size === 0) {
                if(!confirm("尚未選擇任何國家，確定要鎖定嗎？")) return;
            }

            q.locked = true;
            selectQuestion(currentQIndex); 
            alert(`答案已鎖定，題號 ${q.id} 完成。`);
        }

        // --- 成績計算 ---
        function calculateFinalScore() {
            const unlocked = questions.filter(q => !q.locked);
            if(unlocked.length > 0) {
                if(!confirm(`尚有 ${unlocked.length} 題未鎖定，確定要強制提交嗎？未鎖定題目將不計分。`)) return;
            }

            let correctCount = 0;
            const scorePerQ = 100 / questions.length;

            questions.forEach(q => {
                if (!q.locked) return; // 未鎖定題目不計分

                const correctSet = new Set(q.answers);
                let isCorrect = true;

                if(q.userSelection.size !== q.answers.length) {
                    isCorrect = false;
                } else {
                    q.userSelection.forEach(id => {
                        if(!correctSet.has(id)) isCorrect = false;
                    });
                }

                if(isCorrect) {
                    correctCount++;
                }
            });

            // 顯示成績單
            const finalScore = Math.round(correctCount * scorePerQ);
            document.getElementById('score-name').innerText = userInfo.name;
            document.getElementById('final-score').innerText = finalScore;
            document.getElementById('score-detail').innerText = `答對題數: ${correctCount} / ${questions.length}`;
            document.getElementById('score-modal').style.display = 'flex';
        }
    </script>
</body>
</html>
