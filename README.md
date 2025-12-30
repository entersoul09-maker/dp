<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>達譜案場管理系統</title>
    <script src="https://cdn.sheetjs.com/xlsx-latest/package/dist/xlsx.full.min.js"></script>
    <style>
        :root { --bg: #F0F2F5; --card-bg: #FFFFFF; --text: #1C1E21; --accent: #FF9800; --border: #DDDFE2; }
        * { box-sizing: border-box; font-family: -apple-system, sans-serif; }
        body { background: var(--bg); color: var(--text); margin: 0; padding: 12px; }
        .container { max-width: 500px; margin: 0 auto; }
        header { display: flex; justify-content: space-between; align-items: center; padding: 5px 0 10px; }
        .cloud-status { font-size: 0.75rem; text-align: center; margin-bottom: 10px; color: #666; }
        .calendar-card { background: var(--card-bg); border-radius: 15px; padding: 15px; box-shadow: 0 2px 10px rgba(0,0,0,0.08); margin-bottom: 15px; }
        .cal-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 15px; font-weight: bold; }
        .cal-grid { display: grid; grid-template-columns: repeat(7, 1fr); gap: 2px; text-align: center; }
        .cal-date { padding: 12px 0; font-size: 0.95rem; border-radius: 10px; position: relative; }
        .has-event::after { content: ''; width: 5px; height: 5px; background: var(--accent); border-radius: 50%; position: absolute; bottom: 6px; left: 50%; transform: translateX(-50%); }
        .input-card { background: var(--card-bg); border-radius: 15px; padding: 20px; box-shadow: 0 2px 10px rgba(0,0,0,0.08); margin-bottom: 15px; border: 2px solid transparent; transition: 0.3s; }
        .edit-mode { border-color: var(--accent); background: #FFF9F0; }
        .form-row { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; margin-bottom: 12px; }
        label { display: block; font-size: 0.8rem; color: #606770; margin-bottom: 6px; font-weight: 600; }
        input { width: 100%; padding: 14px; border: 1px solid var(--border); border-radius: 10px; font-size: 1rem; background: #F5F6F7; }
        .palette-scroll { display: flex; gap: 8px; overflow-x: auto; padding: 10px 0; }
        .palette-btn { flex: 0 0 auto; padding: 10px 18px; border: 1px solid var(--border); border-radius: 25px; font-size: 0.85rem; background: #fff; }
        .palette-btn.selected { background: var(--accent); color: white; border-color: var(--accent); font-weight: bold; }
        .order-card { background: white; border-radius: 12px; padding: 16px; margin-bottom: 12px; position: relative; border-left: 6px solid var(--accent); box-shadow: 0 2px 6px rgba(0,0,0,0.04); }
        .order-card.closed { border-left-color: #B0B3B8; opacity: 0.7; }
        .btn-group { position: absolute; top: 12px; right: 12px; display: flex; gap: 8px; }
        .action-btn { padding: 8px 12px; font-size: 0.75rem; border-radius: 8px; border: 1px solid #ddd; background: #fff; font-weight: 600; }
        .main-btn { width: 100%; padding: 16px; background: #1C1E21; color: white; border: none; border-radius: 10px; font-size: 1rem; font-weight: bold; margin-top: 10px; }
        .footer-section { display: flex; flex-direction: column; align-items: center; padding: 20px 0 40px; border-top: 1px solid #ddd; }
        .stats-number { font-size: 1.8rem; color: var(--accent); font-weight: 800; }
    </style>
</head>
<body>

<div class="container">
    <header><h1>達譜案場管理</h1></header>
    <div id="syncStatus" class="cloud-status">☁️ 雲端狀態：讀取中...</div>

    <div class="calendar-card">
        <div class="cal-header">
            <button onclick="changeMonth(-1)" style="border:none; background:none;">◀</button>
            <span id="calLabel">年 月</span>
            <button onclick="changeMonth(1)" style="border:none; background:none;">▶</button>
        </div>
        <div class="cal-grid" id="calGrid"></div>
    </div>

    <div class="input-card" id="formContainer">
        <input type="hidden" id="editId">
        <div class="form-row">
            <div><label>案場名稱</label><input type="text" id="siteName"></div>
            <div><label>負責人</label><input type="text" id="manager"></div>
        </div>
        <div style="margin-bottom:12px;">
            <label>下單日</label><input type="date" id="orderDate">
        </div>
        <div class="form-row">
            <div><label>大阪到貨日</label><input type="date" id="arrivalDate" onchange="autoCalc()"></div>
            <div><label>最終出貨日</label><input type="date" id="shipDate"></div>
        </div>
        <div style="margin-bottom:12px;">
            <label>訂單備註</label><input type="text" id="orderMemo">
        </div>
        <label>色板款式 (多選)</label>
        <div class="palette-scroll" id="paletteList"></div>
        <button class="main-btn" id="saveBtn" onclick="saveOrder()">保存案場資料</button>
        <button id="cancelBtn" onclick="resetForm()" style="display:none; width:100%; margin-top:10px; border:none; background:none; color:#777; font-weight:bold;">❌ 取消修改 (回歸新增模式)</button>
    </div>

    <div style="display: flex; justify-content: space-between; margin: 15px 5px;">
        <div style="font-weight: 700;">案場清單</div>
        <label style="font-size:0.9rem; color:#666;"><input type="checkbox" id="hideClosedToggle" onchange="toggleHideClosed()"> 隱藏已結束</label>
    </div>
    <div id="orderList"></div>

    <div class="footer-section">
        <div style="font-size:0.9rem; color:#606770;">本月累計出貨訂單</div>
        <div class="stats-number" id="monthlyStats">0 筆</div>
    </div>
</div>

<script>
    const API_URL = "https://script.google.com/macros/s/AKfycbwCMzNtexj_pUwN2o37MF-BY44tR8_Vv05xULQzdEr7Im5m_FWheF1nHErdHHPaKavh-A/exec";
    const paletteData = ["D317A 水藍", "D321A 鐵灰", "D322A 尼羅河綠", "D301B 黑織紗", "D302B 灰織紗", "D395B 布紋棕", "D1060B 波爾多雪松", "D1122B 風化碳木", "D1183B 北美原橡", "D1185B 冰島白橡", "D1187B 凡爾賽橡木", "D1348 洗白橡木", "D1370B 橡木洗白", "D2091B 丹麥櫸木", "D2415B 安藤清水模", "D3183B 瑞典灰榆", "D5007B 摩卡柚木", "D6357B 白雲岩", "D6358B 泥灰岩", "D371B 台灣柚木", "D373B 古典榆木", "D376B 曉灰榆木", "D3381B 札拉淺橡", "D3383B 札拉灰橡", "D6590C 奶茶米", "D9058C 北歐白核桃", "D6000C 珍珠白", "D6000SC 雪白紋", "D702C 象牙灰", "D552C 艾夏櫚木", "D555C 粉朵拉櫚木", "外訂版", "ETC 其他"];

    let orders = [];
    let hideClosed = JSON.parse(localStorage.getItem('dapu_hide_closed')) || false;
    let selectedColors = new Set();
    let viewDate = new Date();

    // 初始化與抓取資料
    async function init() {
        renderPalette();
        resetDates();
        document.getElementById('hideClosedToggle').checked = hideClosed;
        await fetchData();
    }

    function renderPalette() {
        const pList = document.getElementById('paletteList');
        pList.innerHTML = paletteData.map(name => `<div class="palette-btn" data-name="${name}" onclick="toggleColor(this, '${name}')">${name}</div>`).join('');
    }

    async function fetchData() {
        const statusEl = document.getElementById('syncStatus');
        statusEl.innerText = "🔄 同步中...";
        try {
            const resp = await fetch(API_URL);
            orders = await resp.json();
            statusEl.innerText = "✅ 雲端連線正常";
            statusEl.style.color = "#4CAF50";
        } catch (e) {
            statusEl.innerText = "❌ 離線模式";
            orders = JSON.parse(localStorage.getItem('dapu_db_local')) || [];
        }
        renderCalendar();
        renderOrders();
    }

    // 計算邏輯
    function autoCalc() {
        let date = new Date(document.getElementById('arrivalDate').value);
        if (isNaN(date.getTime())) return;
        let count = (date.getDay() !== 0 && date.getDay() !== 6) ? 1 : 0;
        while (count < 6) {
            date.setDate(date.getDate() + 1);
            if (date.getDay() !== 0 && date.getDay() !== 6) count++;
        }
        document.getElementById('shipDate').valueAsDate = date;
    }

    // 修改訂單：將資料填入表單而不清空
    function editOrder(id) {
        const o = orders.find(x => x.id == id);
        if (!o) return;

        // 1. 填充純文字欄位
        document.getElementById('editId').value = o.id;
        document.getElementById('siteName').value = o.site;
        document.getElementById('manager').value = o.manager || "";
        document.getElementById('orderDate').value = formatDate(o.orderDate);
        document.getElementById('arrivalDate').value = formatDate(o.arrival);
        document.getElementById('shipDate').value = formatDate(o.ship);
        document.getElementById('orderMemo').value = o.memo || "";

        // 2. 被動改變色板選擇
        selectedColors.clear();
        const colorsInOrder = o.colors ? o.colors.split(', ') : [];
        colorsInOrder.forEach(c => selectedColors.add(c));
        
        // 更新按鈕樣式
        document.querySelectorAll('.palette-btn').forEach(btn => {
            const name = btn.getAttribute('data-name');
            if (selectedColors.has(name)) btn.classList.add('selected');
            else btn.classList.remove('selected');
        });

        // 3. 切換介面模式
        document.getElementById('formContainer').classList.add('edit-mode');
        document.getElementById('saveBtn').innerText = "確認修改並同步雲端";
        document.getElementById('cancelBtn').style.display = "block";
        window.scrollTo({ top: 0, behavior: 'smooth' });
    }

    async function saveOrder() {
        const site = document.getElementById('siteName').value;
        if(!site) return alert("案場名稱必填");

        const order = {
            id: document.getElementById('editId').value || Date.now(),
            site: site,
            manager: document.getElementById('manager').value,
            orderDate: document.getElementById('orderDate').value,
            arrival: document.getElementById('arrivalDate').value,
            ship: document.getElementById('shipDate').value,
            memo: document.getElementById('orderMemo').value,
            colors: Array.from(selectedColors).join(', '),
            isClosed: false
        };

        const idx = orders.findIndex(o => o.id == order.id);
        if (idx > -1) {
            order.isClosed = orders[idx].isClosed;
            orders[idx] = order;
        } else {
            orders.unshift(order);
        }

        // 局部更新 UI 提高速度
        renderOrders();
        renderCalendar();
        resetForm();

        // 背景同步
        try {
            await fetch(API_URL, { method: "POST", body: JSON.stringify(orders) });
            localStorage.setItem('dapu_db_local', JSON.stringify(orders));
        } catch (e) { console.error("同步失敗"); }
    }

    function resetForm() {
        document.getElementById('editId').value = "";
        document.getElementById('siteName').value = "";
        document.getElementById('manager').value = "";
        document.getElementById('orderMemo').value = "";
        resetDates();
        selectedColors.clear();
        document.querySelectorAll('.palette-btn').forEach(btn => btn.classList.remove('selected'));
        document.getElementById('formContainer').classList.remove('edit-mode');
        document.getElementById('saveBtn').innerText = "保存案場資料";
        document.getElementById('cancelBtn').style.display = "none";
    }

    function resetDates() {
        document.getElementById('orderDate').valueAsDate = new Date();
        document.getElementById('arrivalDate').valueAsDate = new Date();
        autoCalc();
    }

    function toggleColor(el, name) {
        if(selectedColors.has(name)) { selectedColors.delete(name); el.classList.remove('selected'); }
        else { selectedColors.add(name); el.classList.add('selected'); }
    }

    function formatDate(d) {
        if (!d) return "";
        const date = new Date(d);
        return isNaN(date.getTime()) ? d : date.toISOString().split('T')[0];
    }

    function renderOrders() {
        const container = document.getElementById('orderList');
        let list = [...orders].sort((a,b) => new Date(b.ship) - new Date(a.ship));
        if (hideClosed) list = list.filter(o => String(o.isClosed) !== "true");
        container.innerHTML = list.map(o => `
            <div class="order-card ${String(o.isClosed) === "true" ? 'closed' : ''}">
                <div class="btn-group">
                    <button class="action-btn" onclick="editOrder(${o.id})">修改</button>
                    <button class="action-btn" onclick="toggleStatus(${o.id})">${String(o.isClosed) === "true" ? '恢復' : '結束'}</button>
                </div>
                <div style="font-weight:700;">${o.site}</div>
                <div style="font-size:0.85rem; color:#666;">
                    🚚 出貨：${formatDate(o.ship)} | 🎨：${o.colors}<br>
                    ${o.memo ? `✏️：${o.memo}` : ''}
                </div>
            </div>
        `).join('');
        updateStats();
    }

    function renderCalendar() {
        const grid = document.getElementById('calGrid');
        grid.innerHTML = '';
        const y = viewDate.getFullYear(), m = viewDate.getMonth();
        document.getElementById('calLabel').innerText = `${y}年 ${m+1}月`;
        ['日','一','二','三','四','五','六'].forEach(d => grid.innerHTML += `<div class="cal-day-label">${d}</div>`);
        const firstDay = new Date(y, m, 1).getDay();
        const lastDate = new Date(y, m+1, 0).getDate();
        for(let i=0; i<firstDay; i++) grid.innerHTML += '<div></div>';
        for(let d=1; d<=lastDate; d++) {
            const dateStr = `${y}-${String(m+1).padStart(2,'0')}-${String(d).padStart(2,'0')}`;
            const hasEvent = orders.some(o => formatDate(o.ship) === dateStr && String(o.isClosed) !== "true");
            grid.innerHTML += `<div class="cal-date ${hasEvent?'has-event':''}" onclick="alert('${dateStr} 有案場出貨')">${d}</div>`;
        }
    }

    function updateStats() {
        const y = viewDate.getFullYear(), m = viewDate.getMonth();
        const count = orders.filter(o => {
            const d = new Date(o.ship);
            return d.getFullYear() === y && d.getMonth() === m;
        }).length;
        document.getElementById('monthlyStats').innerText = `${count} 筆`;
    }

    async function toggleStatus(id) {
        const idx = orders.findIndex(o => o.id == id);
        orders[idx].isClosed = !(String(orders[idx].isClosed) === "true");
        renderOrders();
        await fetch(API_URL, { method: "POST", body: JSON.stringify(orders) });
    }

    function toggleHideClosed() {
        hideClosed = document.getElementById('hideClosedToggle').checked;
        localStorage.setItem('dapu_hide_closed', JSON.stringify(hideClosed));
        renderOrders();
    }

    function changeMonth(n) { viewDate.setMonth(viewDate.getMonth() + n); renderCalendar(); updateStats(); }

    init();
</script>
</body>
</html>
