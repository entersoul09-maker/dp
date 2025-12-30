<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>達譜雲端管理系統</title>
    <script src="https://cdn.sheetjs.com/xlsx-latest/package/dist/xlsx.full.min.js"></script>
    <style>
        :root {
            --bg: #F0F2F5;
            --card-bg: #FFFFFF;
            --text: #1C1E21;
            --accent: #FF9800;
            --border: #DDDFE2;
            --input-focus: #4267B2;
        }

        * { box-sizing: border-box; font-family: -apple-system, "Helvetica Neue", Helvetica, Arial, sans-serif; }
        body { background: var(--bg); color: var(--text); margin: 0; padding: 12px; line-height: 1.5; -webkit-tap-highlight-color: transparent; }

        .container { max-width: 500px; margin: 0 auto; }
        header { display: flex; justify-content: space-between; align-items: center; padding: 5px 0 15px; }
        header h1 { font-size: 1.3rem; margin: 0; color: #333; }

        .cloud-status { font-size: 0.75rem; text-align: center; margin-bottom: 10px; color: #666; font-weight: 500; }

        /* 日曆區塊 */
        .calendar-card { background: var(--card-bg); border-radius: 15px; padding: 15px; box-shadow: 0 2px 10px rgba(0,0,0,0.08); margin-bottom: 15px; }
        .cal-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 15px; font-weight: bold; font-size: 1.1rem; }
        .cal-grid { display: grid; grid-template-columns: repeat(7, 1fr); gap: 2px; text-align: center; }
        .cal-day-label { font-size: 0.75rem; color: #888; padding-bottom: 10px; }
        .cal-date { padding: 12px 0; font-size: 0.95rem; border-radius: 10px; position: relative; cursor: pointer; transition: 0.2s; }
        .cal-date:active { background: #eee; }
        .has-event::after { content: ''; width: 5px; height: 5px; background: var(--accent); border-radius: 50%; position: absolute; bottom: 6px; left: 50%; transform: translateX(-50%); }

        /* 表單佈局優化 */
        .input-card { background: var(--card-bg); border-radius: 15px; padding: 20px; box-shadow: 0 2px 10px rgba(0,0,0,0.08); margin-bottom: 15px; }
        .form-section { margin-bottom: 15px; }
        .form-row { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; margin-bottom: 12px; }
        label { display: block; font-size: 0.8rem; color: #606770; margin-bottom: 6px; font-weight: 600; }
        input { 
            width: 100%; padding: 14px; border: 1px solid var(--border); border-radius: 10px; 
            font-size: 1rem; background: #F5F6F7; -webkit-appearance: none; transition: 0.3s;
        }
        input:focus { outline: none; border-color: var(--accent); background: #fff; }

        /* 色板款式滾動 */
        .palette-scroll { display: flex; gap: 8px; overflow-x: auto; padding: 10px 0; margin-top: 5px; }
        .palette-btn { 
            flex: 0 0 auto; padding: 10px 18px; border: 1px solid var(--border); 
            border-radius: 25px; font-size: 0.85rem; background: #fff; color: #4B4F56;
        }
        .palette-btn.selected { background: var(--accent); color: white; border-color: var(--accent); font-weight: bold; }

        /* 列表工具欄 */
        .list-tools { display: flex; justify-content: space-between; align-items: center; margin: 20px 5px 10px; }
        .toggle-label { font-size: 0.9rem; color: #606770; display: flex; align-items: center; gap: 8px; }

        /* 訂單卡片手機優化 */
        .order-card { 
            background: white; border-radius: 12px; padding: 16px; margin-bottom: 12px; 
            position: relative; border-left: 6px solid var(--accent); box-shadow: 0 2px 6px rgba(0,0,0,0.04);
        }
        .order-card.closed { border-left-color: #B0B3B8; opacity: 0.7; }
        .btn-group { position: absolute; top: 12px; right: 12px; display: flex; gap: 8px; }
        .action-btn { padding: 8px 12px; font-size: 0.75rem; border-radius: 8px; border: 1px solid #ddd; background: #fff; font-weight: 600; }

        .main-btn { 
            width: 100%; padding: 16px; background: #1C1E21; color: white; border: none; 
            border-radius: 10px; font-size: 1.05rem; font-weight: bold; margin-top: 15px;
        }
        .main-btn:active { opacity: 0.8; transform: scale(0.98); }

        .footer-section { 
            display: flex; flex-direction: column; align-items: center; 
            padding: 30px 10px 50px; margin-top: 20px; border-top: 1px solid #ddd;
        }
        .stats-number { font-size: 1.8rem; color: var(--accent); font-weight: 800; margin: 5px 0; }
        .export-btn { background: #fff; border: 1.5px solid #1C1E21; color: #1C1E21; padding: 12px 30px; border-radius: 10px; font-size: 0.95rem; font-weight: bold; margin-top: 10px; width: 100%; }
    </style>
</head>
<body>

<div class="container">
    <header>
        <h1>達譜案場管理 (雲端版)</h1>
        <button onclick="shareSite()" style="background:none; border:none; font-size:1.5rem;">📤</button>
    </header>

    <div id="syncStatus" class="cloud-status">☁️ 雲端狀態：讀取中...</div>

    <div class="calendar-card">
        <div class="cal-header">
            <button onclick="changeMonth(-1)" style="border:none; background:none; font-size:1.2rem;">◀</button>
            <span id="calLabel">2024年 11月</span>
            <button onclick="changeMonth(1)" style="border:none; background:none; font-size:1.2rem;">▶</button>
        </div>
        <div class="cal-grid" id="calGrid"></div>
        <div id="eventTip" style="margin-top:10px; font-size:0.85rem; color:var(--accent); display:none; padding:12px; background:#FFF3E0; border-radius:10px;"></div>
    </div>

    <div class="input-card">
        <input type="hidden" id="editId">
        
        <div class="form-row">
            <div><label>案場名稱</label><input type="text" id="siteName" placeholder="案場名稱"></div>
            <div><label>負責人</label><input type="text" id="manager" placeholder="姓名"></div>
        </div>

        <div class="form-section">
            <label>下單日</label>
            <input type="date" id="orderDate">
        </div>

        <div class="form-row">
            <div><label>大阪到貨日</label><input type="date" id="arrivalDate" onchange="autoCalc()"></div>
            <div><label>最終出貨日</label><input type="date" id="shipDate"></div>
        </div>

        <div class="form-section">
            <label>訂單備註</label>
            <input type="text" id="orderMemo" placeholder="案場特殊需求紀錄...">
        </div>

        <label>色板款式 (橫滑多選)</label>
        <div class="palette-scroll" id="paletteList"></div>
        
        <button class="main-btn" id="saveBtn" onclick="saveOrder()">保存並同步雲端資料</button>
        <button id="cancelBtn" onclick="resetForm()" style="display:none; width:100%; margin-top:15px; border:none; background:none; color:#777; font-weight:bold;">取消修正</button>
    </div>

    <div class="list-tools">
        <div style="font-weight: 700;">案場清單</div>
        <label class="toggle-label">
            <input type="checkbox" id="hideClosedToggle" onchange="toggleHideClosed()"> 隱藏已結束
        </label>
    </div>

    <div id="orderList"></div>

    <div class="footer-section">
        <div id="statsMonthLabel" style="font-size:0.9rem; color:#606770;">本月累計出貨訂單</div>
        <div class="stats-number" id="monthlyStats">0 筆</div>
        <button class="export-btn" onclick="exportExcel()">📊 匯出本月 Excel 報表</button>
    </div>
</div>

<script>
    // 1. 已更新為您提供的 URL
    const API_URL = "https://script.google.com/macros/s/AKfycbxe-qH7E_rVz5H6v9S30m_k9m8j2xV_3Y9k9n_z6b-7/exec";

    const paletteData = ["D317A 水藍", "D321A 鐵灰", "D322A 尼羅河綠", "D301B 黑織紗", "D302B 灰織紗", "D395B 布紋棕", "D1060B 波爾多雪松", "D1122B 風化碳木", "D1183B 北美原橡", "D1185B 冰島白橡", "D1187B 凡爾賽橡木", "D1348 洗白橡木", "D1370B 橡木洗白", "D2091B 丹麥櫸木", "D2415B 安藤清水模", "D3183B 瑞典灰榆", "D5007B 摩卡柚木", "D6357B 白雲岩", "D6358B 泥灰岩", "D371B 台灣柚木", "D373B 古典榆木", "D376B 曉灰榆木", "D3381B 札拉淺橡", "D3383B 札拉灰橡", "D6590C 奶茶米", "D9058C 北歐白核桃", "D6000C 珍珠白", "D6000SC 雪白紋", "D702C 象牙灰", "D552C 艾夏櫚木", "D555C 粉朵拉櫚木", "外訂版", "ETC 其他"];

    let orders = [];
    let hideClosed = JSON.parse(localStorage.getItem('dapu_hide_closed')) || false;
    let selectedColors = new Set();
    let viewDate = new Date();

    async function init() {
        const pList = document.getElementById('paletteList');
        pList.innerHTML = paletteData.map(name => `<div class="palette-btn" onclick="toggleColor(this, '${name}')">${name}</div>`).join('');
        
        document.getElementById('orderDate').valueAsDate = new Date();
        document.getElementById('arrivalDate').valueAsDate = new Date();
        document.getElementById('hideClosedToggle').checked = hideClosed;
        autoCalc();
        
        await fetchData();
    }

    async function fetchData() {
        const statusEl = document.getElementById('syncStatus');
        statusEl.innerText = "🔄 雲端同步中...";
        try {
            const resp = await fetch(API_URL);
            orders = await resp.json();
            statusEl.innerText = "✅ 雲端連線正常 (多人模式)";
            statusEl.style.color = "#4CAF50";
        } catch (e) {
            statusEl.innerText = "❌ 離線模式 (資料暫存本地)";
            statusEl.style.color = "#f44336";
            orders = JSON.parse(localStorage.getItem('dapu_db_local')) || [];
        }
        renderCalendar();
        renderOrders();
    }

    async function syncToCloud() {
        try {
            await fetch(API_URL, { method: "POST", body: JSON.stringify(orders) });
            localStorage.setItem('dapu_db_local', JSON.stringify(orders));
        } catch (e) { console.error("Sync failed"); }
    }

    function toggleColor(el, name) {
        if(selectedColors.has(name)) { selectedColors.delete(name); el.classList.remove('selected'); }
        else { selectedColors.add(name); el.classList.add('selected'); }
    }

    function autoCalc() {
        let date = new Date(document.getElementById('arrivalDate').value);
        date.setDate(date.getDate() + 6);
        const day = date.getDay();
        if (day === 6) date.setDate(date.getDate() + 2);
        else if (day === 0) date.setDate(date.getDate() + 1);
        document.getElementById('shipDate').valueAsDate = date;
    }

    function renderCalendar() {
        const grid = document.getElementById('calGrid');
        grid.innerHTML = '';
        const y = viewDate.getFullYear(), m = viewDate.getMonth();
        document.getElementById('calLabel').innerText = `${y}年 ${m+1}月`;
        document.getElementById('statsMonthLabel').innerText = `${m+1}月 出貨訂單總數`;
        
        ['日','一','二','三','四','五','六'].forEach(d => grid.innerHTML += `<div class="cal-day-label">${d}</div>`);
        const firstDay = new Date(y, m, 1).getDay();
        const lastDate = new Date(y, m+1, 0).getDate();
        for(let i=0; i<firstDay; i++) grid.innerHTML += '<div></div>';
        for(let d=1; d<=lastDate; d++) {
            const dateStr = `${y}-${String(m+1).padStart(2,'0')}-${String(d).padStart(2,'0')}`;
            // 判斷該日期是否有出貨
            const hasEvent = orders.some(o => o.ship === dateStr && (o.isClosed === false || o.isClosed === "false"));
            grid.innerHTML += `<div class="cal-date ${hasEvent?'has-event':''}" onclick="showTip('${dateStr}')">${d}</div>`;
        }
        updateStats();
    }

    function updateStats() {
        const y = viewDate.getFullYear(), m = viewDate.getMonth();
        const count = orders.filter(o => {
            const d = new Date(o.ship);
            return d.getFullYear() === y && d.getMonth() === m;
        }).length;
        document.getElementById('monthlyStats').innerText = `${count} 筆`;
    }

    function showTip(date) {
        const dayOrders = orders.filter(o => o.ship === date && (o.isClosed === false || o.isClosed === "false"));
        const tip = document.getElementById('eventTip');
        if(dayOrders.length) {
            tip.style.display = 'block';
            tip.innerHTML = `🚚 <strong>${date} 出貨案場：</strong><br>` + dayOrders.map(o => o.site).join('、');
        } else { tip.style.display = 'none'; }
    }

    async function saveOrder() {
        const site = document.getElementById('siteName').value;
        if(!site) return alert("請填寫案場名稱");
        const order = {
            id: document.getElementById('editId').value || Date.now(),
            site: site, manager: document.getElementById('manager').value,
            orderDate: document.getElementById('orderDate').value,
            arrival: document.getElementById('arrivalDate').value,
            ship: document.getElementById('shipDate').value,
            memo: document.getElementById('orderMemo').value,
            colors: Array.from(selectedColors).join(', '),
            isClosed: false
        };
        const idx = orders.findIndex(o => o.id == order.id);
        if(idx > -1) { order.isClosed = orders[idx].isClosed; orders[idx] = order; }
        else { orders.unshift(order); }
        await syncToCloud();
        location.reload();
    }

    function renderOrders() {
        const container = document.getElementById('orderList');
        let list = [...orders].sort((a,b) => new Date(b.ship) - new Date(a.ship));
        if (hideClosed) list = list.filter(o => o.isClosed === false || o.isClosed === "false");
        
        container.innerHTML = list.map(o => `
            <div class="order-card ${o.isClosed === true || o.isClosed === "true" ? 'closed' : ''}">
                <div class="btn-group">
                    <button class="action-btn" onclick="editOrder(${o.id})">修改</button>
                    <button class="action-btn" onclick="toggleStatus(${o.id})">${o.isClosed === true || o.isClosed === "true" ? '恢復' : '結束'}</button>
                </div>
                <div style="font-weight:700; font-size:1.05rem; margin-bottom:5px;">${o.site}</div>
                <div style="font-size:0.85rem; color:#606770; line-height:1.6;">
                    📝 下單：${o.orderDate}<br>
                    📦 到貨：${o.arrival} | 🚚 出貨：${o.ship}<br>
                    🎨 色板：${o.colors}<br>
                    ${o.memo ? `✏️ 備註：<span style="color:var(--accent); font-weight:bold;">${o.memo}</span>` : ''}
                </div>
            </div>
        `).join('');
    }

    function editOrder(id) {
        const o = orders.find(x => x.id == id);
        document.getElementById('editId').value = o.id;
        document.getElementById('siteName').value = o.site;
        document.getElementById('manager').value = o.manager;
        document.getElementById('orderDate').value = o.orderDate;
        document.getElementById('arrivalDate').value = o.arrival;
        document.getElementById('shipDate').value = o.ship;
        document.getElementById('orderMemo').value = o.memo;
        document.getElementById('saveBtn').innerText = "更新雲端資料";
        document.getElementById('cancelBtn').style.display = "block";
        window.scrollTo({top: 0, behavior: 'smooth'});
    }

    async function toggleStatus(id) {
        const idx = orders.findIndex(o => o.id == id);
        const current = (orders[idx].isClosed === true || orders[idx].isClosed === "true");
        orders[idx].isClosed = !current;
        await syncToCloud();
        renderOrders(); 
        renderCalendar();
    }

    function toggleHideClosed() {
        hideClosed = document.getElementById('hideClosedToggle').checked;
        localStorage.setItem('dapu_hide_closed', JSON.stringify(hideClosed));
        renderOrders();
    }

    function changeMonth(n) { viewDate.setMonth(viewDate.getMonth() + n); renderCalendar(); }
    function resetForm() { location.reload(); }
    function shareSite() { if(navigator.share) navigator.share({ title: '達譜管理系統', url: window.location.href }); }

    function exportExcel() {
        const y = viewDate.getFullYear(), m = viewDate.getMonth();
        const currentMonthOrders = orders.filter(o => {
            const shipD = new Date(o.ship);
            return shipD.getFullYear() === y && shipD.getMonth() === m;
        });
        if (currentMonthOrders.length === 0) return alert("本月份無資料可匯出");
        const worksheet = XLSX.utils.json_to_sheet(currentMonthOrders);
        const workbook = XLSX.utils.book_new();
        XLSX.utils.book_append_sheet(workbook, worksheet, "案場報表");
        XLSX.writeFile(workbook, `達譜_${y}_${m+1}月份報表.xlsx`);
    }

    init();
</script>
</body>
</html>
