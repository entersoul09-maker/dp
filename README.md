<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>達譜案場管理系統 (工作日版)</title>
    <script src="https://cdn.sheetjs.com/xlsx-latest/package/dist/xlsx.full.min.js"></script>
    <style>
        :root {
            --bg: #F8F9FA;
            --card-bg: #FFFFFF;
            --text: #333333;
            --accent: #FF9800;
            --border: #E0E0E0;
        }

        * { box-sizing: border-box; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; }
        body { background: var(--bg); color: var(--text); margin: 0; padding: 10px; line-height: 1.6; }

        .container { max-width: 500px; margin: 0 auto; }
        header { display: flex; justify-content: space-between; align-items: center; padding: 10px 5px; }
        
        /* 日曆 */
        .calendar-card { background: var(--card-bg); border-radius: 12px; padding: 15px; box-shadow: 0 4px 12px rgba(0,0,0,0.05); margin-bottom: 15px; }
        .cal-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 12px; font-weight: bold; }
        .cal-grid { display: grid; grid-template-columns: repeat(7, 1fr); gap: 5px; text-align: center; }
        .cal-day-label { font-size: 0.7rem; color: #999; margin-bottom: 5px; }
        .cal-date { padding: 10px 0; font-size: 0.9rem; border-radius: 8px; position: relative; }
        .weekend { color: #ccc; } /* 週末顏色變淡 */
        .has-event::after { content: ''; width: 4px; height: 4px; background: var(--accent); border-radius: 50%; position: absolute; bottom: 4px; left: 50%; transform: translateX(-50%); }

        /* 表單 */
        .input-card { background: var(--card-bg); border-radius: 12px; padding: 18px; box-shadow: 0 2px 8px rgba(0,0,0,0.05); margin-bottom: 15px; }
        .form-row { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; margin-bottom: 12px; }
        label { display: block; font-size: 0.75rem; color: #777; margin-bottom: 4px; font-weight: bold; }
        input { width: 100%; padding: 12px; border: 1px solid var(--border); border-radius: 8px; font-size: 1rem; background: #FAFAFA; }
        
        /* 警告文字 */
        .date-warn { font-size: 0.7rem; color: #E67E22; margin-top: 4px; display: none; }

        .palette-scroll { display: flex; gap: 8px; overflow-x: auto; padding: 5px 0 10px; }
        .palette-btn { flex: 0 0 auto; padding: 8px 15px; border: 1px solid var(--border); border-radius: 20px; font-size: 0.8rem; background: #fff; }
        .palette-btn.selected { background: var(--accent); color: white; border-color: var(--accent); }

        .order-card { background: white; border-radius: 10px; padding: 15px; margin-bottom: 10px; position: relative; border-left: 5px solid var(--accent); }
        .order-card.closed { border-left-color: #ccc; opacity: 0.6; }
        .btn-group { position: absolute; top: 12px; right: 10px; display: flex; gap: 5px; }
        .action-btn { padding: 6px 10px; font-size: 0.7rem; border-radius: 6px; border: 1px solid #ddd; background: white; }
        
        .main-btn { width: 100%; padding: 15px; background: #333; color: white; border: none; border-radius: 8px; font-size: 1rem; font-weight: bold; }
    </style>
</head>
<body>

<div class="container">
    <header>
        <h1>達譜案場管理</h1>
        <button onclick="shareSite()" style="background:none; border:none; font-size:1.5rem;">📤</button>
    </header>

    <div class="calendar-card">
        <div class="cal-header">
            <button onclick="changeMonth(-1)" style="border:none; background:none;">◀</button>
            <span id="calLabel">2023年 10月</span>
            <button onclick="changeMonth(1)" style="border:none; background:none;">▶</button>
        </div>
        <div class="cal-grid" id="calGrid"></div>
        <div id="eventTip" style="margin-top:10px; font-size:0.85rem; color:var(--accent); display:none;"></div>
    </div>

    <div class="input-card">
        <input type="hidden" id="editId">
        <div class="form-row">
            <div><label>案場名稱</label><input type="text" id="siteName" placeholder="案場名稱"></div>
            <div><label>負責人</label><input type="text" id="manager" placeholder="姓名"></div>
        </div>
        <div class="form-row">
            <div><label>下單日</label><input type="date" id="startDate" onchange="autoCalc()"></div>
            <div>
                <label>出貨日 (僅限週一至五)</label>
                <input type="date" id="shipDate" onchange="validateShipDate(this)">
                <div id="dateWarn" class="date-warn">⚠️ 週末不出貨，已自動調整。</div>
            </div>
        </div>
        <label>色板選擇</label>
        <div class="palette-scroll" id="paletteList"></div>
        <button class="main-btn" onclick="saveOrder()">保存訂單</button>
    </div>

    <div id="orderList"></div>
</div>

<script>
    const paletteNames = ["D317A 水藍", "D321A 鐵灰", "D301B 黑織紗", "D302B 灰織紗", "D1183B 原橡", "D2415B 清水模", "D6590C 奶茶米", "ETC 其他"];
    let orders = JSON.parse(localStorage.getItem('dapu_workday_v1')) || [];
    let selectedColors = new Set();
    let viewDate = new Date();

    function init() {
        const pList = document.getElementById('paletteList');
        pList.innerHTML = paletteNames.map(name => `<div class="palette-btn" onclick="toggleColor(this, '${name}')">${name}</div>`).join('');
        document.getElementById('startDate').valueAsDate = new Date();
        autoCalc();
        renderCalendar();
        renderOrders();
    }

    // 核心邏輯：自動計算並避開週末
    function autoCalc() {
        let date = new Date(document.getElementById('startDate').value);
        if(isNaN(date)) return;

        // 預設加 6 天
        date.setDate(date.getDate() + 6);
        
        // 檢查是否落在週末
        adjustIfWeekend(date);

        document.getElementById('shipDate').valueAsDate = date;
    }

    // 核心邏輯：手動選擇時若選到週末則順延
    function validateShipDate(input) {
        let date = new Date(input.value);
        const day = date.getDay(); // 0 是週日, 6 是週六
        if (day === 0 || day === 6) {
            document.getElementById('dateWarn').style.display = 'block';
            adjustIfWeekend(date);
            input.valueAsDate = date;
            setTimeout(() => { document.getElementById('dateWarn').style.display = 'none'; }, 3000);
        }
    }

    function adjustIfWeekend(date) {
        const day = date.getDay();
        if (day === 6) date.setDate(date.getDate() + 2); // 週六移到週一
        else if (day === 0) date.setDate(date.getDate() + 1); // 週日移到週一
    }

    function toggleColor(el, name) {
        if(selectedColors.has(name)) { selectedColors.delete(name); el.classList.remove('selected'); }
        else { selectedColors.add(name); el.classList.add('selected'); }
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
            const dayOfWeek = new Date(y, m, d).getDay();
            const isWeekend = (dayOfWeek === 0 || dayOfWeek === 6);
            const hasEvent = orders.some(o => o.ship === dateStr && !o.isClosed);
            grid.innerHTML += `<div class="cal-date ${isWeekend?'weekend':''} ${hasEvent?'has-event':''}" onclick="showTip('${dateStr}')">${d}</div>`;
        }
    }

    function showTip(date) {
        const dayOrders = orders.filter(o => o.ship === date && !o.isClosed);
        const tip = document.getElementById('eventTip');
        if(dayOrders.length) {
            tip.style.display = 'block';
            tip.innerHTML = `🚚 <strong>${date} 出貨：</strong><br>` + dayOrders.map(o => o.site).join('、');
        } else { tip.style.display = 'none'; }
    }

    function saveOrder() {
        const site = document.getElementById('siteName').value;
        if(!site) return alert("請填寫案場名稱");
        const order = {
            id: document.getElementById('editId').value || Date.now(),
            site: site, manager: document.getElementById('manager').value,
            ship: document.getElementById('shipDate').value,
            colors: Array.from(selectedColors).join(', '),
            isClosed: false
        };
        const idx = orders.findIndex(o => o.id == order.id);
        if(idx > -1) { order.isClosed = orders[idx].isClosed; orders[idx] = order; }
        else { orders.unshift(order); }
        localStorage.setItem('dapu_workday_v1', JSON.stringify(orders));
        location.reload();
    }

    function renderOrders() {
        const container = document.getElementById('orderList');
        container.innerHTML = orders.map(o => `
            <div class="order-card ${o.isClosed?'closed':''}">
                <div class="btn-group">
                    <button class="action-btn" onclick="editOrder(${o.id})">修正</button>
                    <button class="action-btn" onclick="toggleStatus(${o.id})">${o.isClosed?'恢復':'結束'}</button>
                </div>
                <div class="order-info">
                    <h3>${o.site}</h3>
                    <p>🚚 出貨日：${o.ship}</p>
                    <p>🎨 色板：${o.colors || '未選'}</p>
                </div>
            </div>
        `).join('');
    }

    function editOrder(id) {
        const o = orders.find(x => x.id == id);
        document.getElementById('editId').value = o.id;
        document.getElementById('siteName').value = o.site;
        document.getElementById('manager').value = o.manager;
        document.getElementById('shipDate').value = o.ship;
        window.scrollTo({top: 0, behavior: 'smooth'});
    }

    function toggleStatus(id) {
        const idx = orders.findIndex(o => o.id == id);
        orders[idx].isClosed = !orders[idx].isClosed;
        localStorage.setItem('dapu_workday_v1', JSON.stringify(orders));
        renderOrders(); renderCalendar();
    }

    function changeMonth(n) { viewDate.setMonth(viewDate.getMonth() + n); renderCalendar(); }
    function shareSite() { navigator.share({ title: '達譜系統', url: window.location.href }); }

    init();
</script>
</body>
</html>
