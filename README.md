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
            --accent: #FF9800; /* 亮橘色更適合戶外辨識 */
            --border: #E0E0E0;
            --safe-area-bottom: env(safe-area-inset-bottom);
        }

        * { box-sizing: border-box; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; }
        
        body { 
            background: var(--bg); 
            color: var(--text); 
            margin: 0; 
            padding: 10px 10px calc(20px + var(--safe-area-bottom)); 
            line-height: 1.6;
        }

        .container { max-width: 500px; margin: 0 auto; }

        header { 
            display: flex; justify-content: space-between; align-items: center;
            padding: 10px 5px; margin-bottom: 10px;
        }
        header h1 { font-size: 1.2rem; margin: 0; font-weight: 700; color: #000; }

        /* 手機版優化日曆 */
        .calendar-card { 
            background: var(--card-bg); 
            border-radius: 12px; 
            padding: 15px; 
            box-shadow: 0 4px 12px rgba(0,0,0,0.05);
            margin-bottom: 15px;
        }
        .cal-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 12px; font-weight: bold; }
        .cal-grid { display: grid; grid-template-columns: repeat(7, 1fr); gap: 5px; text-align: center; }
        .cal-date { 
            padding: 10px 0; font-size: 0.9rem; border-radius: 8px; 
            position: relative; transition: 0.2s; -webkit-tap-highlight-color: transparent;
        }
        .cal-date:active { background: #eee; }
        .has-event { color: var(--accent); font-weight: bold; }
        .has-event::after {
            content: ''; width: 4px; height: 4px; background: var(--accent); 
            border-radius: 50%; position: absolute; bottom: 4px; left: 50%; transform: translateX(-50%);
        }

        /* 浮動式提示 */
        #eventTip { 
            margin-top: 10px; padding: 12px; background: #FFF3E0; 
            border-left: 4px solid var(--accent); border-radius: 4px; 
            font-size: 0.85rem; display: none; animation: fadeIn 0.3s;
        }

        /* 表單優化 */
        .input-card { 
            background: var(--card-bg); border-radius: 12px; padding: 18px; 
            box-shadow: 0 2px 8px rgba(0,0,0,0.05); margin-bottom: 15px;
        }
        .form-row { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; margin-bottom: 12px; }
        label { display: block; font-size: 0.75rem; color: #777; margin-bottom: 4px; font-weight: bold; }
        input { 
            width: 100%; padding: 12px; border: 1px solid var(--border); 
            border-radius: 8px; background: #FAFAFA; font-size: 1rem;
            -webkit-appearance: none; /* 移除 iOS 預設陰影 */
        }
        
        .palette-scroll { 
            display: flex; gap: 8px; overflow-x: auto; padding: 5px 0 10px;
            -webkit-overflow-scrolling: touch;
        }
        .palette-btn { 
            flex: 0 0 auto; padding: 8px 15px; border: 1px solid var(--border);
            border-radius: 20px; font-size: 0.8rem; background: #fff;
        }
        .palette-btn.selected { background: var(--accent); color: white; border-color: var(--accent); }

        /* 訂單列表 */
        .order-card { 
            background: white; border-radius: 10px; padding: 15px; 
            margin-bottom: 10px; position: relative; border-left: 5px solid var(--accent);
            box-shadow: 0 2px 5px rgba(0,0,0,0.03);
        }
        .order-card.closed { border-left-color: #ccc; opacity: 0.6; background: #f9f9f9; }
        .order-info h3 { margin: 0 0 5px 0; font-size: 1rem; }
        .order-info p { margin: 2px 0; font-size: 0.85rem; color: #666; }
        
        .btn-group { position: absolute; top: 12px; right: 10px; display: flex; gap: 5px; }
        .action-btn { 
            padding: 6px 10px; font-size: 0.7rem; border-radius: 6px; 
            border: 1px solid #ddd; background: white; font-weight: bold;
        }

        .main-btn { 
            width: 100%; padding: 15px; background: #333; color: white; 
            border: none; border-radius: 8px; font-size: 1rem; font-weight: bold;
            margin-top: 10px;
        }

        @keyframes fadeIn { from { opacity: 0; transform: translateY(-5px); } to { opacity: 1; transform: translateY(0); } }
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
        <div id="eventTip"></div>
    </div>

    <div class="input-card">
        <input type="hidden" id="editId">
        <div class="form-row">
            <div><label>案場名稱</label><input type="text" id="siteName" placeholder="案場名稱"></div>
            <div><label>負責人</label><input type="text" id="manager" placeholder="姓名"></div>
        </div>
        <div class="form-row">
            <div><label>下單日</label><input type="date" id="startDate" onchange="autoCalc()"></div>
            <div><label>出貨日</label><input type="date" id="shipDate"></div>
        </div>
        <label>色板選擇 (橫向滑動可多選)</label>
        <div class="palette-scroll" id="paletteList"></div>
        <button class="main-btn" id="saveBtn" onclick="saveOrder()">保存訂單</button>
        <button id="cancelBtn" onclick="resetForm()" style="display:none; width:100%; margin-top:10px; border:none; background:none; color:#999; font-size:0.8rem;">取消修正</button>
    </div>

    <div style="padding: 0 5px 15px;">
        <input type="text" id="search" placeholder="🔍 快速搜尋案場或負責人..." 
               style="width:100%; border-radius:25px; text-align:center; border:1px solid #ddd;" oninput="renderOrders()">
    </div>
    
    <div id="orderList"></div>
    
    <div style="text-align:center; margin-top:30px;">
        <button onclick="exportExcel()" style="color:#888; background:none; border:1px solid #ccc; padding:8px 15px; border-radius:5px; font-size:0.8rem;">匯出 Excel 備份</button>
    </div>
</div>

<script>
    const paletteNames = ["D317A 水藍", "D321A 鐵灰", "D301B 黑織紗", "D302B 灰織紗", "D1183B 北美原橡", "D2415B 安藤清水模", "D6590C 奶茶米", "ETC 其他"];
    let orders = JSON.parse(localStorage.getItem('dapu_mobile_v1')) || [];
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

    function toggleColor(el, name) {
        if(selectedColors.has(name)) { selectedColors.delete(name); el.classList.remove('selected'); }
        else { selectedColors.add(name); el.classList.add('selected'); }
    }

    function autoCalc() {
        let date = new Date(document.getElementById('startDate').value);
        if(isNaN(date)) return;
        let count = 0;
        while (count < 6) {
            date.setDate(date.getDate() + 1);
            if (date.getDay() !== 0 && date.getDay() !== 6) count++;
        }
        document.getElementById('shipDate').valueAsDate = date;
    }

    function renderCalendar() {
        const grid = document.getElementById('calGrid');
        grid.innerHTML = '';
        const y = viewDate.getFullYear(), m = viewDate.getMonth();
        document.getElementById('calLabel').innerText = `${y}年 ${m+1}月`;
        
        const firstDay = new Date(y, m, 1).getDay();
        const lastDate = new Date(y, m+1, 0).getDate();
        
        for(let i=0; i<firstDay; i++) grid.innerHTML += '<div></div>';
        for(let d=1; d<=lastDate; d++) {
            const dateStr = `${y}-${String(m+1).padStart(2,'0')}-${String(d).padStart(2,'0')}`;
            const hasEvent = orders.some(o => o.ship === dateStr && !o.isClosed);
            grid.innerHTML += `<div class="cal-date ${hasEvent?'has-event':''}" onclick="showTip('${dateStr}')">${d}</div>`;
        }
    }

    function showTip(date) {
        const dayOrders = orders.filter(o => o.ship === date && !o.isClosed);
        const tip = document.getElementById('eventTip');
        if(dayOrders.length) {
            tip.style.display = 'block';
            tip.innerHTML = `📍 <strong>${date} 出貨：</strong><br>` + dayOrders.map(o => o.site).join('、');
        } else { tip.style.display = 'none'; }
    }

    function saveOrder() {
        const site = document.getElementById('siteName').value;
        if(!site) return alert("請填寫案場名稱");
        
        const order = {
            id: document.getElementById('editId').value || Date.now(),
            site: site,
            manager: document.getElementById('manager').value,
            ship: document.getElementById('shipDate').value,
            colors: Array.from(selectedColors).join(', '),
            isClosed: false
        };

        const idx = orders.findIndex(o => o.id == order.id);
        if(idx > -1) {
            order.isClosed = orders[idx].isClosed;
            orders[idx] = order;
        } else { orders.unshift(order); }
        
        localStorage.setItem('dapu_mobile_v1', JSON.stringify(orders));
        location.reload();
    }

    function renderOrders() {
        const term = document.getElementById('search').value.toLowerCase();
        const container = document.getElementById('orderList');
        container.innerHTML = orders.filter(o => o.site.toLowerCase().includes(term) || o.manager.toLowerCase().includes(term)).map(o => `
            <div class="order-card ${o.isClosed?'closed':''}">
                <div class="btn-group">
                    <button class="action-btn" style="color:orange" onclick="editOrder(${o.id})">修正</button>
                    <button class="action-btn" onclick="toggleStatus(${o.id})">${o.isClosed?'恢復':'結束'}</button>
                    <button class="action-btn" style="color:red" onclick="deleteOrder(${o.id})">刪</button>
                </div>
                <div class="order-info">
                    <h3>${o.site}</h3>
                    <p>👤 負責人：${o.manager || '未填'}</p>
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
        document.getElementById('saveBtn').innerText = "確認更新";
        document.getElementById('cancelBtn').style.display = "block";
        window.scrollTo({top: 0, behavior: 'smooth'});
    }

    function resetForm() { location.reload(); }

    function toggleStatus(id) {
        const idx = orders.findIndex(o => o.id == id);
        orders[idx].isClosed = !orders[idx].isClosed;
        localStorage.setItem('dapu_mobile_v1', JSON.stringify(orders));
        renderOrders(); renderCalendar();
    }

    function deleteOrder(id) {
        if(confirm("確定刪除此紀錄？")) {
            orders = orders.filter(o => o.id != id);
            localStorage.setItem('dapu_mobile_v1', JSON.stringify(orders));
            renderOrders(); renderCalendar();
        }
    }

    function changeMonth(n) { viewDate.setMonth(viewDate.getMonth() + n); renderCalendar(); }
    
    function shareSite() {
        if (navigator.share) {
            navigator.share({ title: '達譜系統', url: window.location.href });
        } else { alert("請手動複製網址分享"); }
    }

    function exportExcel() {
        const ws = XLSX.utils.json_to_sheet(orders);
        const wb = XLSX.utils.book_new();
        XLSX.utils.book_append_sheet(wb, ws, "案場紀錄");
        XLSX.writeFile(wb, `達譜案場報表_${new Date().toLocaleDateString()}.xlsx`);
    }

    init();
</script>
</body>
</html>
