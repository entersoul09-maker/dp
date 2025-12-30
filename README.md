<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>達譜案場雲端管理系統</title>
    <script src="https://cdn.sheetjs.com/xlsx-latest/package/dist/xlsx.full.min.js"></script>
    <style>
        :root { --bg: #F8F9FA; --card-bg: #FFFFFF; --text: #333333; --accent: #FF9800; --border: #E0E0E0; }
        * { box-sizing: border-box; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; }
        body { background: var(--bg); color: var(--text); margin: 0; padding: 10px; line-height: 1.6; }
        .container { max-width: 500px; margin: 0 auto; }
        header { display: flex; justify-content: space-between; align-items: center; padding: 10px 5px; }
        header h1 { font-size: 1.2rem; margin: 0; font-weight: 700; }
        
        /* 狀態標籤 */
        .cloud-status { font-size: 0.75rem; text-align: center; color: #666; margin-bottom: 10px; padding: 4px; border-radius: 4px; }

        /* 日曆看板樣式 */
        .calendar-card { background: var(--card-bg); border-radius: 12px; padding: 15px; box-shadow: 0 4px 12px rgba(0,0,0,0.05); margin-bottom: 15px; }
        .cal-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 12px; font-weight: bold; }
        .cal-grid { display: grid; grid-template-columns: repeat(7, 1fr); gap: 5px; text-align: center; }
        .cal-day-label { font-size: 0.7rem; color: #999; margin-bottom: 5px; }
        .cal-date { padding: 10px 0; font-size: 0.9rem; border-radius: 8px; position: relative; cursor: pointer; }
        .weekend { color: #ccc; }
        .has-event::after { content: ''; width: 4px; height: 4px; background: var(--accent); border-radius: 50%; position: absolute; bottom: 4px; left: 50%; transform: translateX(-50%); }

        /* 表單樣式 */
        .input-card { background: var(--card-bg); border-radius: 12px; padding: 18px; box-shadow: 0 2px 8px rgba(0,0,0,0.05); margin-bottom: 15px; }
        .form-row { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; margin-bottom: 12px; }
        label { display: block; font-size: 0.75rem; color: #777; margin-bottom: 4px; font-weight: bold; }
        input { width: 100%; padding: 12px; border: 1px solid var(--border); border-radius: 8px; font-size: 1rem; background: #FAFAFA; }
        
        /* 色板款式 */
        .palette-scroll { display: flex; gap: 8px; overflow-x: auto; padding: 10px 0; -webkit-overflow-scrolling: touch; }
        .palette-btn { flex: 0 0 auto; padding: 8px 16px; border: 1px solid var(--border); border-radius: 20px; font-size: 0.8rem; background: #fff; }
        .palette-btn.selected { background: var(--accent); color: white; border-color: var(--accent); }

        /* 清單工具 */
        .list-tools { display: flex; justify-content: space-between; align-items: center; margin: 15px 5px 10px; }
        .toggle-label { font-size: 0.85rem; color: #666; display: flex; align-items: center; gap: 5px; cursor: pointer; }

        /* 訂單卡片 */
        .order-card { background: white; border-radius: 10px; padding: 15px; margin-bottom: 10px; position: relative; border-left: 5px solid var(--accent); box-shadow: 0 2px 5px rgba(0,0,0,0.03); }
        .order-card.closed { border-left-color: #ccc; opacity: 0.6; }
        .btn-group { position: absolute; top: 12px; right: 10px; display: flex; gap: 5px; }
        .action-btn { padding: 6px 10px; font-size: 0.7rem; border-radius: 6px; border: 1px solid #ddd; background: white; font-weight: bold; }
        
        .main-btn { width: 100%; padding: 15px; background: #333; color: white; border: none; border-radius: 8px; font-size: 1rem; font-weight: bold; margin-top: 10px; }
        .footer-section { display: flex; flex-direction: column; align-items: center; padding: 20px 5px 40px; margin-top: 20px; border-top: 2px solid #EEE; }
        .stats-number { font-size: 1.5rem; color: var(--accent); font-weight: 900; }
        .export-btn { background: white; border: 1px solid #333; color: #333; padding: 10px 25px; border-radius: 8px; font-size: 0.9rem; cursor: pointer; font-weight: bold; }
    </style>
</head>
<body>

<div class="container">
    <header>
        <h1>達譜案場管理</h1>
        <button onclick="shareSite()" style="background:none; border:none; font-size:1.5rem;">📤</button>
    </header>
    
    <div id="syncStatus" class="cloud-status">☁️ 雲端狀態：載入中...</div>

    <div class="calendar-card">
        <div class="cal-header">
            <button onclick="changeMonth(-1)" style="border:none; background:none;">◀</button>
            <span id="calLabel">年 月</span>
            <button onclick="changeMonth(1)" style="border:none; background:none;">▶</button>
        </div>
        <div class="cal-grid" id="calGrid"></div>
        <div id="eventTip" style="margin-top:10px; font-size:0.85rem; color:var(--accent); display:none; padding:10px; background:#FFF3E0; border-radius:5px;"></div>
    </div>

    <div class="input-card">
        <input type="hidden" id="editId">
        <div class="form-row">
            <div><label>案場名稱</label><input type="text" id="siteName" placeholder="案場名稱"></div>
            <div><label>負責人</label><input type="text" id="manager" placeholder="姓名"></div>
        </div>
        <div style="margin-bottom:12px;">
            <label>下單日</label>
            <input type="date" id="orderDate">
        </div>
        <div class="form-row">
            <div><label>大板到貨日</label><input type="date" id="arrivalDate" onchange="autoCalc()"></div>
            <div><label>最終出貨日</label><input type="date" id="shipDate"></div>
        </div>
        <div style="margin-bottom:12px;">
            <label>訂單備註</label>
            <input type="text" id="orderMemo" placeholder="案場特殊需求紀錄...">
        </div>
        <label style="font-size:0.75rem; color:#777;">色板款式 (橫滑多選)</label>
        <div class="palette-scroll" id="paletteList"></div>
        <button class="main-btn" id="saveBtn" onclick="saveOrder()">保存並同步雲端</button>
        <button id="cancelBtn" onclick="resetForm()" style="display:none; width:100%; margin-top:10px; border:none; background:none; color:#999;">取消修正</button>
    </div>

    <div class="list-tools">
        <div style="font-weight: bold; font-size: 0.9rem;">案場清單</div>
        <label class="toggle-label">
            <input type="checkbox" id="hideClosedToggle" onchange="toggleHideClosed()"> 隱藏已結束案場
        </label>
    </div>

    <div id="orderList"></div>

    <div class="footer-section">
        <div id="statsMonthLabel" style="font-size:0.8rem; color:#888;">本月出貨訂單總數</div>
        <div class="stats-number" id="monthlyStats">0 筆</div><br>
        <button class="export-btn" onclick="exportExcel()">📊 輸出本月 Excel 報表</button>
    </div>
</div>

<script>
    // 💡 請在此處填入您的 Google Script URL
    const API_URL = "https://script.google.com/macros/library/d/1P_GQPn8DwAdL_TAqcYaF_4GppLuJkQjRopmFqsHvBlvqGvJyawfliX6V/2";

    const paletteData = ["D317A 水藍", "D321A 鐵灰", "D322A 尼羅河綠", "D301B 黑織紗", "D302B 灰織紗", "D395B 布紋棕", "D1060B 波爾多雪松", "D1122B 風化碳木", "D1183B 北美原橡", "D1185B 冰島白橡", "D1187B 凡爾賽橡木", "D1348 洗白橡木", "D1370B 橡木洗白", "D2091B 丹麥櫸木", "D2415B 安藤清水模", "D3183B 瑞典灰榆", "D5007B 摩卡柚木", "D6357B 白雲岩", "D6358B 泥灰岩", "D371B 台灣柚木", "D373B 古典榆木", "D376B 曉灰榆木", "D3381B 札拉淺橡", "D3383B 札拉灰橡", "D6590C 奶茶米", "D9058C 北歐白核桃", "D6000C 珍珠白", "D6000SC 雪白紋", "D702C 象牙灰", "D552C 艾夏櫚木", "D555C 粉朵拉櫚木", "外訂版", "ETC 其他"];

    let orders = [];
    let hideClosed = JSON.parse(localStorage.getItem('dapu_hide_closed')) || false;
    let selectedColors = new Set();
    let viewDate = new Date();

    async function init() {
        // 初始化色板
        const pList = document.getElementById('paletteList');
        pList.innerHTML = paletteData.map(name => `<div class="palette-btn" onclick="toggleColor(this, '${name}')">${name}</div>`).join('');
        
        // 預設日期
        document.getElementById('orderDate').valueAsDate = new Date();
        document.getElementById('arrivalDate').valueAsDate = new Date();
        document.getElementById('hideClosedToggle').checked = hideClosed;
        autoCalc();
        
        // 載入資料
        await fetchData();
    }

    async function fetchData() {
        const statusEl = document.getElementById('syncStatus');
        statusEl.innerText = "🔄 雲端同步中...";
        try {
            const resp = await fetch(API_URL);
            orders = await resp.json();
            statusEl.innerText = "✅ 雲端連線正常";
            statusEl.style.color = "#4CAF50";
        } catch (e) {
            statusEl.innerText = "❌ 離線模式 (請檢查 URL 設定)";
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
            tip.innerHTML = `🚚 <strong>${date} 出貨：</strong><br>` + dayOrders.map(o => o.site).join('、');
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
                    <button class="action-btn" onclick="editOrder(${o.id})">修</button>
                    <button class="action-btn" onclick="toggleStatus(${o.id})">${o.isClosed === true || o.isClosed === "true" ? '恢復' : '結束'}</button>
                </div>
                <strong>${o.site}</strong><br>
                <small>下單:${o.orderDate} | 到貨:${o.arrival} | 出貨:${o.ship}</small><br>
                <small>🎨:${o.colors}</small>
            </div>
        `).join('');
    }

    function editOrder(id) {
        const o = orders.find(x => x.id == id);
        document.getElementById('editId').value = o.id;
        document.getElementById('siteName').value = o.site;
        document.getElementById('orderDate').value = o.orderDate;
        document.getElementById('arrivalDate').value = o.arrival;
        document.getElementById('shipDate').value = o.ship;
        document.getElementById('orderMemo').value = o.memo;
        document.getElementById('saveBtn').innerText = "更新紀錄";
        document.getElementById('cancelBtn').style.display = "block";
        window.scrollTo(0,0);
    }

    async function toggleStatus(id) {
        const idx = orders.findIndex(o => o.id == id);
        const current = (orders[idx].isClosed === true || orders[idx].isClosed === "true");
        orders[idx].isClosed = !current;
        await syncToCloud();
        renderOrders(); renderCalendar();
    }

    function toggleHideClosed() {
        hideClosed = document.getElementById('hideClosedToggle').checked;
        localStorage.setItem('dapu_hide_closed', JSON.stringify(hideClosed));
        renderOrders();
    }

    function changeMonth(n) { viewDate.setMonth(viewDate.getMonth() + n); renderCalendar(); }
    function resetForm() { location.reload(); }
    function shareSite() { if(navigator.share) navigator.share({ title: '達譜管理', url: window.location.href }); }

    function exportExcel() {
        const y = viewDate.getFullYear(), m = viewDate.getMonth();
        const currentMonthOrders = orders.filter(o => {
            const shipD = new Date(o.ship);
            return shipD.getFullYear() === y && shipD.getMonth() === m;
        });
        if (currentMonthOrders.length === 0) return alert("本月無資料");
        const worksheet = XLSX.utils.json_to_sheet(currentMonthOrders);
        const workbook = XLSX.utils.book_new();
        XLSX.utils.book_append_sheet(workbook, worksheet, "報表");
        XLSX.writeFile(workbook, `達譜_${y}_${m+1}.xlsx`);
    }

    init();
</script>
</body>
</html>
