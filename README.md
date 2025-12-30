<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>達譜雲端同步系統</title>
    <script src="https://cdn.sheetjs.com/xlsx-latest/package/dist/xlsx.full.min.js"></script>
    <style>
        /* 樣式保持與之前一致，維持美觀 */
        :root { --bg: #F8F9FA; --card-bg: #FFFFFF; --text: #333333; --accent: #FF9800; --border: #E0E0E0; }
        * { box-sizing: border-box; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; }
        body { background: var(--bg); color: var(--text); margin: 0; padding: 10px; line-height: 1.6; }
        .container { max-width: 500px; margin: 0 auto; }
        header { display: flex; justify-content: space-between; align-items: center; padding: 10px 5px; }
        .calendar-card { background: var(--card-bg); border-radius: 12px; padding: 15px; box-shadow: 0 4px 12px rgba(0,0,0,0.05); margin-bottom: 15px; }
        .cal-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 12px; font-weight: bold; }
        .cal-grid { display: grid; grid-template-columns: repeat(7, 1fr); gap: 5px; text-align: center; }
        .cal-date { padding: 10px 0; font-size: 0.9rem; border-radius: 8px; position: relative; cursor: pointer; }
        .has-event::after { content: ''; width: 4px; height: 4px; background: var(--accent); border-radius: 50%; position: absolute; bottom: 4px; left: 50%; transform: translateX(-50%); }
        .input-card { background: var(--card-bg); border-radius: 12px; padding: 18px; box-shadow: 0 2px 8px rgba(0,0,0,0.05); margin-bottom: 15px; }
        .form-row { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; margin-bottom: 12px; }
        input { width: 100%; padding: 12px; border: 1px solid var(--border); border-radius: 8px; font-size: 1rem; background: #FAFAFA; }
        .palette-scroll { display: flex; gap: 8px; overflow-x: auto; padding: 10px 0; }
        .palette-btn { flex: 0 0 auto; padding: 8px 16px; border: 1px solid var(--border); border-radius: 20px; font-size: 0.8rem; background: #fff; }
        .palette-btn.selected { background: var(--accent); color: white; border-color: var(--accent); }
        .order-card { background: white; border-radius: 10px; padding: 15px; margin-bottom: 10px; position: relative; border-left: 5px solid var(--accent); }
        .order-card.closed { border-left-color: #ccc; opacity: 0.6; }
        .main-btn { width: 100%; padding: 15px; background: #333; color: white; border: none; border-radius: 8px; font-weight: bold; margin-top: 10px; }
        .cloud-status { font-size: 0.7rem; text-align: center; color: #4CAF50; margin-bottom: 10px; }
        .footer-section { text-align: center; padding: 20px 0 40px; border-top: 2px solid #EEE; }
        .stats-number { font-size: 1.5rem; color: var(--accent); font-weight: 900; }
    </style>
</head>
<body>

<div class="container">
    <header>
        <h1>達譜案場 (雲端版)</h1>
        <div id="syncStatus" class="cloud-status">☁️ 雲端同步中...</div>
    </header>

    <div class="calendar-card">
        <div class="cal-header">
            <button onclick="changeMonth(-1)">◀</button>
            <span id="calLabel">年 月</span>
            <button onclick="changeMonth(1)">▶</button>
        </div>
        <div class="cal-grid" id="calGrid"></div>
    </div>

    <div class="input-card">
        <input type="hidden" id="editId">
        <div class="form-row">
            <div><label>案場名稱</label><input type="text" id="siteName"></div>
            <div><label>負責人</label><input type="text" id="manager"></div>
        </div>
        <div class="form-row">
            <div><label>下單日</label><input type="date" id="orderDate"></div>
            <div><label>大板到貨日</label><input type="date" id="arrivalDate" onchange="autoCalc()"></div>
        </div>
        <div>
            <label>最終出貨日</label>
            <input type="date" id="shipDate">
        </div>
        <div style="margin-top:10px;">
            <label>備註</label>
            <input type="text" id="orderMemo" placeholder="案場需求...">
        </div>
        <div class="palette-scroll" id="paletteList"></div>
        <button class="main-btn" id="saveBtn" onclick="saveOrder()">保存並同步至雲端</button>
    </div>

    <div style="display: flex; justify-content: space-between; padding: 10px;">
        <strong>案場清單</strong>
        <label style="font-size: 0.8rem;"><input type="checkbox" id="hideClosedToggle" onchange="renderOrders()"> 隱藏已結束</label>
    </div>
    <div id="orderList"></div>

    <div class="footer-section">
        <div id="statsMonthLabel" style="font-size:0.8rem; color:#888;">本月訂單</div>
        <div class="stats-number" id="monthlyStats">0 筆</div><br>
        <button class="main-btn" style="background:#fff; color:#333; border:1px solid #333;" onclick="exportExcel()">📊 匯出本月 Excel</button>
    </div>
</div>

<script>
    // 請將下方的網址替換為你部署 Apps Script 得到的 URL
    const API_URL = "https://script.google.com/macros/s/AKfycbwCMzNtexj_pUwN2o37MF-BY44tR8_Vv05xULQzdEr7Im5m_FWheF1nHErdHHPaKavh-A/exec";

    const paletteData = ["D317A 水藍", "D321A 鐵灰", "D322A 尼羅河綠", "D301B 黑織紗", "D302B 灰織紗", "D395B 布紋棕", "D1060B 波爾多雪松", "D1122B 風化碳木", "D1183B 北美原橡", "D1185B 冰島白橡", "D1187B 凡爾賽橡木", "D1348 洗白橡木", "D1370B 橡木洗白", "D2091B 丹麥櫸木", "D2415B 安藤清水模", "D3183B 瑞典灰榆", "D5007B 摩卡柚木", "D6357B 白雲岩", "D6358B 泥灰岩", "D371B 台灣柚木", "D373B 古典榆木", "D376B 曉灰榆木", "D3381B 札拉淺橡", "D3383B 札拉灰橡", "D6590C 奶茶米", "D9058C 北歐白核桃", "D6000C 珍珠白", "D6000SC 雪白紋", "D702C 象牙灰", "D552C 艾夏櫚木", "D555C 粉朵拉櫚木", "外訂版", "ETC 其他"];

    let orders = [];
    let selectedColors = new Set();
    let viewDate = new Date();

    async function init() {
        const pList = document.getElementById('paletteList');
        pList.innerHTML = paletteData.map(name => `<div class="palette-btn" onclick="toggleColor(this, '${name}')">${name}</div>`).join('');
        document.getElementById('orderDate').valueAsDate = new Date();
        document.getElementById('arrivalDate').valueAsDate = new Date();
        autoCalc();
        await fetchData(); // 從雲端抓取資料
    }

    // 從 Google Sheets 抓取資料
    async function fetchData() {
        document.getElementById('syncStatus').innerText = "🔄 正在從雲端同步...";
        try {
            const resp = await fetch(API_URL);
            orders = await resp.json();
            document.getElementById('syncStatus').innerText = "✅ 雲端同步完成";
            renderCalendar(); renderOrders();
        } catch (e) {
            document.getElementById('syncStatus').innerText = "❌ 同步失敗，目前為離線模式";
            orders = JSON.parse(localStorage.getItem('dapu_backup')) || [];
            renderCalendar(); renderOrders();
        }
    }

    // 將資料傳送到 Google Sheets
    async function syncToCloud() {
        document.getElementById('syncStatus').innerText = "📤 正在上傳資料...";
        try {
            await fetch(API_URL, {
                method: "POST",
                body: JSON.stringify(orders)
            });
            document.getElementById('syncStatus').innerText = "✅ 資料已成功存至雲端";
        } catch (e) {
            alert("雲端儲存失敗，請檢查網路！");
        }
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

    async function saveOrder() {
        const site = document.getElementById('siteName').value;
        if(!site) return alert("案場名稱必填");
        
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
        
        localStorage.setItem('dapu_backup', JSON.stringify(orders)); // 本地備份
        await syncToCloud(); // 雲端同步
        location.reload();
    }

    function renderOrders() {
        const container = document.getElementById('orderList');
        const hide = document.getElementById('hideClosedToggle').checked;
        let list = orders;
        if(hide) list = list.filter(o => !o.isClosed);
        
        container.innerHTML = list.map(o => `
            <div class="order-card ${o.isClosed?'closed':''}">
                <div style="position:absolute; right:10px; top:10px;">
                    <button onclick="editOrder(${o.id})">修</button>
                    <button onclick="toggleStatus(${o.id})">${o.isClosed?'恢復':'結束'}</button>
                </div>
                <strong>${o.site}</strong><br>
                <small>🚚 出貨：${o.ship} | 🎨：${o.colors}</small>
            </div>
        `).join('');
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

    function toggleStatus(id) {
        const idx = orders.findIndex(o => o.id == id);
        orders[idx].isClosed = (orders[idx].isClosed === 'true' || orders[idx].isClosed === true) ? false : true;
        syncToCloud();
        renderOrders();
    }

    function changeMonth(n) { viewDate.setMonth(viewDate.getMonth()+n); renderCalendar(); }
    function renderCalendar() { /* ...日曆繪製代碼... */ }
    function exportExcel() { /* ...匯出邏輯... */ }

    init();
</script>
</body>
</html>
