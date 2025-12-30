<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>達譜案場管理系統</title>
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
        header h1 { font-size: 1.2rem; margin: 0; font-weight: 700; }

        /* 日曆 */
        .calendar-card { background: var(--card-bg); border-radius: 12px; padding: 15px; box-shadow: 0 4px 12px rgba(0,0,0,0.05); margin-bottom: 15px; }
        .cal-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 12px; font-weight: bold; }
        .cal-grid { display: grid; grid-template-columns: repeat(7, 1fr); gap: 5px; text-align: center; }
        .cal-day-label { font-size: 0.7rem; color: #999; margin-bottom: 5px; }
        .cal-date { padding: 10px 0; font-size: 0.9rem; border-radius: 8px; position: relative; cursor: pointer; }
        .weekend { color: #ccc; }
        .has-event::after { content: ''; width: 4px; height: 4px; background: var(--accent); border-radius: 50%; position: absolute; bottom: 4px; left: 50%; transform: translateX(-50%); }

        /* 表單 */
        .input-card { background: var(--card-bg); border-radius: 12px; padding: 18px; box-shadow: 0 2px 8px rgba(0,0,0,0.05); margin-bottom: 15px; }
        .form-row { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; margin-bottom: 12px; }
        label { display: block; font-size: 0.75rem; color: #777; margin-bottom: 4px; font-weight: bold; }
        input { width: 100%; padding: 12px; border: 1px solid var(--border); border-radius: 8px; font-size: 1rem; background: #FAFAFA; -webkit-appearance: none; }
        
        /* 色板選擇區 */
        .palette-label { display: flex; justify-content: space-between; align-items: center; margin-top: 10px; }
        .palette-scroll { display: flex; gap: 8px; overflow-x: auto; padding: 10px 0; -webkit-overflow-scrolling: touch; }
        .palette-btn { flex: 0 0 auto; padding: 8px 16px; border: 1px solid var(--border); border-radius: 20px; font-size: 0.8rem; background: #fff; text-align: center; }
        .palette-btn.selected { background: var(--accent); color: white; border-color: var(--accent); }

        /* 列表與統計 */
        .order-card { background: white; border-radius: 10px; padding: 15px; margin-bottom: 10px; position: relative; border-left: 5px solid var(--accent); box-shadow: 0 2px 5px rgba(0,0,0,0.03); }
        .order-card.closed { border-left-color: #ccc; opacity: 0.6; }
        .btn-group { position: absolute; top: 12px; right: 10px; display: flex; gap: 5px; }
        .action-btn { padding: 6px 10px; font-size: 0.7rem; border-radius: 6px; border: 1px solid #ddd; background: white; font-weight: bold; }
        
        .main-btn { width: 100%; padding: 15px; background: #333; color: white; border: none; border-radius: 8px; font-size: 1rem; font-weight: bold; margin-top: 10px; }
        .date-warn { font-size: 0.7rem; color: #E67E22; margin-top: 4px; display: none; }
        .memo-tag { color: #E67E22; font-weight: bold; }

        /* 頁尾統計區塊 */
        .footer-section { 
            display: flex; flex-direction: column; align-items: center; 
            padding: 20px 5px 40px; margin-top: 20px; border-top: 2px solid #EEE;
        }
        .stats-display { 
            text-align: center; margin-bottom: 15px; width: 100%;
        }
        .stats-label { font-size: 0.8rem; color: #888; margin-bottom: 5px; }
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
            <div><label>大阪到貨日</label><input type="date" id="arrivalDate" onchange="autoCalc()"></div>
            <div>
                <label>最終出貨日 (不含六日)</label>
                <input type="date" id="shipDate" onchange="validateShipDate(this)">
                <div id="dateWarn" class="date-warn">⚠️ 週末不出貨，已自動跳至週一。</div>
            </div>
        </div>
        <div style="margin-bottom:12px;">
            <label>訂單備註</label>
            <input type="text" id="orderMemo" placeholder="案場特殊需求紀錄...">
        </div>
        <div class="palette-label">
            <label>色板款式 (橫滑多選)</label>
            <span id="colorCount" style="font-size:0.7rem; color:#999;">已選 0 項</span>
        </div>
        <div class="palette-scroll" id="paletteList"></div>
        <button class="main-btn" id="saveBtn" onclick="saveOrder()">保存紀錄</button>
        <button id="cancelBtn" onclick="resetForm()" style="display:none; width:100%; margin-top:10px; border:none; background:none; color:#999; font-size:0.8rem;">取消修正</button>
    </div>

    <div id="orderList"></div>

    <div class="footer-section">
        <div class="stats-display">
            <div class="stats-label" id="statsMonthLabel">本月出貨訂單總數</div>
            <div class="stats-number" id="monthlyStats">0 筆</div>
        </div>
        <button class="export-btn" onclick="exportExcel()">📊 輸出本月 Excel 報表</button>
    </div>
</div>

<script>
    const paletteData = [
        "D317A 水藍", "D321A 鐵灰", "D322A 尼羅河綠", "D301B 黑織紗", "D302B 灰織紗", "D395B 布紋棕",
        "D1060B 波爾多雪松", "D1122B 風化碳木", "D1183B 北美原橡", "D1185B 冰島白橡", "D1187B 凡爾賽橡木", "D1348 洗白橡木",
        "D1370B 橡木洗白", "D2091B 丹麥櫸木", "D2415B 安藤清水模", "D3183B 瑞典灰榆", "D5007B 摩卡柚木", "D6357B 白雲岩",
        "D6358B 泥灰岩", "D371B 台灣柚木", "D373B 古典榆木", "D376B 曉灰榆木", "D3381B 札拉淺橡", "D3383B 札拉灰橡",
        "D6590C 奶茶米", "D9058C 北歐白核桃", "D6000C 珍珠白", "D6000SC 雪白紋", "D702C 象牙灰", "D552C 艾夏櫚木",
        "D555C 粉朵拉櫚木", "外訂版", "ETC 其他"
    ];

    let orders = JSON.parse(localStorage.getItem('dapu_v5_final')) || [];
    let selectedColors = new Set();
    let viewDate = new Date();

    function init() {
        const pList = document.getElementById('paletteList');
        pList.innerHTML = paletteData.map(name => `<div class="palette-btn" onclick="toggleColor(this, '${name}')">${name}</div>`).join('');
        document.getElementById('orderDate').valueAsDate = new Date();
        document.getElementById('arrivalDate').valueAsDate = new Date();
        autoCalc();
        renderCalendar();
        renderOrders();
    }

    function toggleColor(el, name) {
        if(selectedColors.has(name)) { selectedColors.delete(name); el.classList.remove('selected'); }
        else { selectedColors.add(name); el.classList.add('selected'); }
        document.getElementById('colorCount').innerText = `已選 ${selectedColors.size} 項`;
    }

    function autoCalc() {
        let date = new Date(document.getElementById('arrivalDate').value);
        if(isNaN(date)) return;
        date.setDate(date.getDate() + 6);
        adjustIfWeekend(date);
        document.getElementById('shipDate').valueAsDate = date;
    }

    function validateShipDate(input) {
        let date = new Date(input.value);
        if (date.getDay() === 0 || date.getDay() === 6) {
            document.getElementById('dateWarn').style.display = 'block';
            adjustIfWeekend(date);
            input.valueAsDate = date;
            setTimeout(() => { document.getElementById('dateWarn').style.display = 'none'; }, 3000);
        }
    }

    function adjustIfWeekend(date) {
        const day = date.getDay();
        if (day === 6) date.setDate(date.getDate() + 2);
        else if (day === 0) date.setDate(date.getDate() + 1);
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
            const isWeekend = ([0,6].includes(new Date(y, m, d).getDay()));
            const hasEvent = orders.some(o => o.ship === dateStr && !o.isClosed);
            grid.innerHTML += `<div class="cal-date ${isWeekend?'weekend':''} ${hasEvent?'has-event':''}" onclick="showTip('${dateStr}')">${d}</div>`;
        }
        updateStats();
    }

    // 重點：依照顯示月份的出貨日計算總量
    function updateStats() {
        const y = viewDate.getFullYear(), m = viewDate.getMonth();
        const monthlyCount = orders.filter(o => {
            const shipD = new Date(o.ship);
            return shipD.getFullYear() === y && shipD.getMonth() === m;
        }).length;
        document.getElementById('monthlyStats').innerText = `${monthlyCount} 筆`;
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
        localStorage.setItem('dapu_v5_final', JSON.stringify(orders));
        location.reload();
    }

    function renderOrders() {
        const container = document.getElementById('orderList');
        // 依照出貨日排序（最近的在上面）
        const sortedOrders = [...orders].sort((a,b) => new Date(b.ship) - new Date(a.ship));
        container.innerHTML = sortedOrders.map(o => `
            <div class="order-card ${o.isClosed?'closed':''}">
                <div class="btn-group">
                    <button class="action-btn" style="color:orange" onclick="editOrder(${o.id})">修正</button>
                    <button class="action-btn" onclick="toggleStatus(${o.id})">${o.isClosed?'恢復':'結束'}</button>
                </div>
                <div class="order-info">
                    <h3 style="margin:0;">${o.site}</h3>
                    <p style="margin:5px 0; font-size:0.85rem; color:#666;">
                        📝 下單日：${o.orderDate || '未填'}<br>
                        📦 大阪到貨日：${o.arrival || '未填'}<br>
                        🚚 最終出貨日：${o.ship}<br>
                        🎨 色板：${o.colors || '未選'}<br>
                        ${o.memo ? `✏️ 備註：<span class="memo-tag">${o.memo}</span>` : ''}
                    </p>
                </div>
            </div>
        `).join('');
    }

    function editOrder(id) {
        const o = orders.find(x => x.id == id);
        document.getElementById('editId').value = o.id;
        document.getElementById('siteName').value = o.site;
        document.getElementById('manager').value = o.manager;
        document.getElementById('orderDate').value = o.orderDate || '';
        document.getElementById('arrivalDate').value = o.arrival || '';
        document.getElementById('shipDate').value = o.ship;
        document.getElementById('orderMemo').value = o.memo || '';
        selectedColors.clear();
        document.querySelectorAll('.palette-btn').forEach(btn => {
            btn.classList.remove('selected');
            if(o.colors.includes(btn.innerText)) { selectedColors.add(btn.innerText); btn.classList.add('selected'); }
        });
        document.getElementById('colorCount').innerText = `已選 ${selectedColors.size} 項`;
        document.getElementById('saveBtn').innerText = "確認更新紀錄";
        document.getElementById('cancelBtn').style.display = "block";
        window.scrollTo({top: 0, behavior: 'smooth'});
    }

    function resetForm() { location.reload(); }
    function toggleStatus(id) {
        const idx = orders.findIndex(o => o.id == id);
        orders[idx].isClosed = !orders[idx].isClosed;
        localStorage.setItem('dapu_v5_final', JSON.stringify(orders));
        renderOrders(); renderCalendar();
    }
    function changeMonth(n) { viewDate.setMonth(viewDate.getMonth() + n); renderCalendar(); }
    function shareSite() { if(navigator.share) navigator.share({ title: '達譜系統', url: window.location.href }); }

    function exportExcel() {
        const y = viewDate.getFullYear(), m = viewDate.getMonth();
        const currentMonthOrders = orders.filter(o => {
            const shipD = new Date(o.ship);
            return shipD.getFullYear() === y && shipD.getMonth() === m;
        });

        if (currentMonthOrders.length === 0) return alert("本月份無出貨資料可匯出");
        
        const dataForExcel = currentMonthOrders.map(o => ({
            "案場名稱": o.site,
            "負責人": o.manager,
            "下單日": o.orderDate || "",
            "大阪到貨日": o.arrival || "",
            "最終出貨日": o.ship,
            "狀態": o.isClosed ? "已結束" : "進行中",
            "色板款式": o.colors,
            "備註": o.memo || ""
        }));
        
        const worksheet = XLSX.utils.json_to_sheet(dataForExcel);
        const workbook = XLSX.utils.book_new();
        XLSX.utils.book_append_sheet(workbook, worksheet, "本月出貨單");
        XLSX.writeFile(workbook, `達譜_${y}年${m+1}月報表.xlsx`);
    }

    init();
</script>
</body>
</html>
