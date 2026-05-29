<template>
  <div class="calendar-app cute-theme">
    <h1>🎀 2026年 Eshennn的行事曆 🌸✨</h1>
    
    <div class="toolbar">
      <div class="settings">
        <label>🔗 GAS 部署網址：</label>
        <input v-model="gasUrl" type="text" placeholder="請貼上 https://script.google.com/macros/s/.../exec" @change="saveGasUrl" />
      </div>
      <div class="file-actions">
        <label class="btn file-label">
          匯入 ICS
          <input type="file" @change="importICS" accept=".ics" hidden />
        </label>
        <button class="btn" @click="exportICS">匯出 ICS</button>
      </div>
    </div>

    <div class="main-content">
      <!-- 左側內容區塊 (新增表單 + 事項列表) -->
      <div class="content-left">
        <div class="form-card">
          <h2>新增事項</h2>
          <div class="form-group">
            <label>標題</label>
            <input v-model="newEvent.title" type="text" placeholder="例: 開會" />
          </div>
          <div class="form-group">
            <label>開始時間</label>
            <input v-model="newEvent.start" type="datetime-local" />
          </div>
          <div class="form-group">
            <label>結束時間</label>
            <input v-model="newEvent.end" type="datetime-local" />
          </div>
          <div class="form-group">
            <label>分類</label>
            <input v-model="newEvent.category" type="text" placeholder="例: 工作" />
          </div>
          <div class="form-group">
            <label>描述</label>
            <textarea v-model="newEvent.description" placeholder="事項備註..."></textarea>
          </div>
          <button class="btn btn-primary" @click="addEvent">新增並儲存</button>
        </div>
        
        <div class="events-list">
          <h2>事項列表</h2>
          <div v-if="events.length === 0" class="empty">目前沒有任何事項</div>
          <div v-for="event in events" :key="event.id" class="event-card">
            <div class="event-header">
              <h3>{{ event.title }}</h3>
              <span v-if="event.category" class="badge">{{ event.category }}</span>
            </div>
            <div class="event-time">
              ⏱ {{ formatDisplayDate(event.start) }} ~ {{ formatDisplayDate(event.end) }}
            </div>
            <p class="event-desc">{{ event.description }}</p>
          </div>
        </div>
      </div>

      <!-- 右側純月曆區塊 (真表格) -->
      <div class="content-right">
        <div class="months-grid">
          <div v-for="monthData in yearMonths" :key="monthData.month" class="calendar-view">
            <div class="calendar-header">
              <h2>💖 {{ monthData.month + 1 }} 月 💖</h2>
            </div>
            <!-- 真正的 HTML 表格結構 -->
            <table class="calendar-table">
              <thead>
                <tr>
                  <th class="weekday">日</th>
                  <th class="weekday">一</th>
                  <th class="weekday">二</th>
                  <th class="weekday">三</th>
                  <th class="weekday">四</th>
                  <th class="weekday">五</th>
                  <th class="weekday">六</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="(week, wIndex) in chunkCells(monthData.cells)" :key="wIndex">
                  <td v-for="(cell, cIndex) in week" :key="cIndex"
                      class="calendar-cell" 
                      :class="{ 'empty-cell': !cell.date, 'has-date': cell.date }" 
                      @click="selectDate(monthData.month, cell)">
                    <div v-if="cell.date" class="cell-date">{{ cell.date }}</div>
                    <div class="cell-events">
                      <div v-for="ev in cell.events" :key="ev.id" class="mini-event" :title="ev.title">{{ ev.title }}</div>
                    </div>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue';

// 從 localStorage 讀取之前存過的網址，預設為空字串
const gasUrl = ref(localStorage.getItem('gasUrl') || '');

const saveGasUrl = () => {
  localStorage.setItem('gasUrl', gasUrl.value);
  if (gasUrl.value) {
    loadEvents();
  }
};

const events = ref([]);
const newEvent = ref({
  title: '',
  start: '',
  end: '',
  category: '',
  description: ''
});

// --- 月曆視圖邏輯 ---
const currentYear = 2026; // 固定鎖定在 2026 年

const yearMonths = computed(() => {
  const months = [];
  for (let m = 0; m < 12; m++) {
    const daysInMonth = new Date(currentYear, m + 1, 0).getDate();
    const firstDayOfWeek = new Date(currentYear, m, 1).getDay(); // 0 (日) - 6 (六)
    
    const cells = [];
    // 補齊前面的空白天數
    for (let i = 0; i < firstDayOfWeek; i++) {
      cells.push({ date: null, events: [] });
    }
    // 當月日期
    for (let d = 1; d <= daysInMonth; d++) {
      const dayEvents = events.value.filter(ev => {
        if (!ev.start) return false;
        const evDate = new Date(ev.start);
        return evDate.getFullYear() === currentYear &&
               evDate.getMonth() === m &&
               evDate.getDate() === d;
      });
      cells.push({ date: d, events: dayEvents });
    }
    // 補齊後面的空白天數，讓陣列長度必須是 7 的倍數 (以利表格分週)
    while (cells.length % 7 !== 0) {
      cells.push({ date: null, events: [] });
    }

    months.push({ month: m, cells });
  }
  return months;
});

// 將每個月的格子分切為「週」(每 7 天一組)
const chunkCells = (cells) => {
  const weeks = [];
  for (let i = 0; i < cells.length; i += 7) {
    weeks.push(cells.slice(i, i + 7));
  }
  return weeks;
};

// 點擊日期時，自動將該日期帶入「新增事項」的表單中
const selectDate = (monthIndex, cell) => {
  if (!cell.date) return;
  const y = currentYear;
  const m = String(monthIndex + 1).padStart(2, '0');
  const d = String(cell.date).padStart(2, '0');
  // 預設為選中日期的早上 9:00 ~ 10:00
  newEvent.value.start = `${y}-${m}-${d}T09:00`;
  newEvent.value.end = `${y}-${m}-${d}T10:00`;
};

// 從 Google Sheets 載入資料
const loadEvents = async () => {
  if (!gasUrl.value || gasUrl.value === 'YOUR_GOOGLE_APPS_SCRIPT_WEB_APP_URL') return;
  try {
    const res = await fetch(gasUrl.value);
    const data = await res.json();
    events.value = data;
  } catch (e) {
    console.error('無法載入事項:', e);
  }
};

// 新增單一事項
const addEvent = async () => {
  if (!newEvent.value.title || !newEvent.value.start) {
    return alert('請至少填寫標題與開始時間！');
  }

  const eventToSave = {
    id: Date.now().toString(),
    ...newEvent.value
  };

  // 樂觀更新 (Optimistic update)
  events.value.push(eventToSave);

  try {
    await fetch(gasUrl.value, {
      method: 'POST',
      headers: { 'Content-Type': 'text/plain;charset=utf-8' }, // 避免 CORS preflight
      body: JSON.stringify({ action: 'add', events: [eventToSave] })
    });
    alert('新增成功！');
    newEvent.value = { title: '', start: '', end: '', category: '', description: '' };
  } catch (e) {
    console.error('儲存事項失敗:', e);
  }
};

// 匯出 ICS
const exportICS = () => {
  let icsContent = "BEGIN:VCALENDAR\nVERSION:2.0\nPRODID:-//Vue Calendar//EN\n";
  events.value.forEach(ev => {
    const dtStart = new Date(ev.start).toISOString().replace(/[-:]/g, '').split('.')[0] + 'Z';
    const dtEnd = ev.end ? new Date(ev.end).toISOString().replace(/[-:]/g, '').split('.')[0] + 'Z' : dtStart;

    icsContent += "BEGIN:VEVENT\n";
    icsContent += `UID:${ev.id}@vuecalendar\n`;
    icsContent += `DTSTART:${dtStart}\n`;
    icsContent += `DTEND:${dtEnd}\n`;
    icsContent += `SUMMARY:${ev.title}\n`;
    if (ev.category) icsContent += `CATEGORIES:${ev.category}\n`;
    if (ev.description) icsContent += `DESCRIPTION:${ev.description}\n`;
    icsContent += "END:VEVENT\n";
  });
  icsContent += "END:VCALENDAR";

  const blob = new Blob([icsContent], { type: 'text/calendar' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = 'calendar-events.ics';
  a.click();
  URL.revokeObjectURL(url);
};

// 匯入 ICS
const importICS = (e) => {
  const file = e.target.files[0];
  if (!file) return;

  const reader = new FileReader();
  reader.onload = async (event) => {
    const content = event.target.result;
    const lines = content.split(/\r\n|\n|\r/);
    const imported = [];
    let currentEvent = null;

    // 簡易 ICS 解析
    for (let line of lines) {
      if (line.startsWith('BEGIN:VEVENT')) {
        currentEvent = { id: Date.now().toString() + Math.random().toString(36).substring(2, 7) };
      } else if (line.startsWith('END:VEVENT')) {
        if (currentEvent) imported.push(currentEvent);
        currentEvent = null;
      } else if (currentEvent) {
        if (line.startsWith('SUMMARY:')) currentEvent.title = line.substring(8);
        else if (line.startsWith('DESCRIPTION:')) currentEvent.description = line.substring(12);
        else if (line.startsWith('CATEGORIES:')) currentEvent.category = line.substring(11);
        else if (line.startsWith('DTSTART:')) currentEvent.start = formatICSDate(line.substring(8));
        else if (line.startsWith('DTEND:')) currentEvent.end = formatICSDate(line.substring(6));
      }
    }

    if (imported.length > 0) {
      events.value.push(...imported);
      try {
        await fetch(gasUrl.value, {
          method: 'POST',
          headers: { 'Content-Type': 'text/plain;charset=utf-8' },
          body: JSON.stringify({ action: 'import', events: imported })
        });
        alert(`成功匯入並儲存 ${imported.length} 筆資料至 Google Sheets`);
      } catch (err) {
        console.error('匯入儲存失敗:', err);
      }
    }
    e.target.value = ''; // 重置 file input
  };
  reader.readAsText(file);
};

// 將 ICS 日期格式 (YYYYMMDDTHHmmssZ) 轉為 datetime-local 可讀格式
const formatICSDate = (icsDate) => {
  if (!icsDate) return '';
  const str = icsDate.replace('Z', '');
  if (str.length >= 15) {
    return `${str.slice(0, 4)}-${str.slice(4, 6)}-${str.slice(6, 8)}T${str.slice(9, 11)}:${str.slice(11, 13)}`;
  }
  return icsDate;
};

// 畫面呈現用日期格式
const formatDisplayDate = (dateStr) => {
  if (!dateStr) return '未設定';
  return dateStr.replace('T', ' ');
};

onMounted(() => {
  if (gasUrl.value) {
    loadEvents();
  }
});
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=M+PLUS+Rounded+1c:wght@700;900&display=swap');

/* 淺粉櫻花色可愛主題 CSS */
.cute-theme { background-color: #fff5f7; color: #7a4b56; min-height: 100vh; border-radius: 15px; padding: 25px; font-weight: 900; }
.calendar-app { max-width: 98%; margin: 0 auto; font-family: 'M PLUS Rounded 1c', 'Varela Round', 'Arial Rounded MT Bold', 'cwTeXYen', '微軟正黑體', sans-serif; letter-spacing: 0.5px; }
h1 { text-align: center; color: #e87a90; margin-bottom: 25px; text-shadow: 2px 2px 0px #fff, 4px 4px 0px #ffd1dc; font-size: 2.8em; letter-spacing: 2px; }
.toolbar { margin-bottom: 20px; padding-bottom: 20px; border-bottom: 2px dashed #ffd1dc; display: flex; justify-content: space-between; align-items: center; }
.settings label { color: #e87a90; font-weight: 900; margin-right: 10px; }
.settings input { width: 350px; padding: 8px 15px; border: 2px solid #ffd1dc; border-radius: 20px; background: #fffdfa; color: #7a4b56; outline: none; font-family: inherit; font-weight: bold; transition: all 0.2s; }
.settings input:focus { border-color: #ff9eb5; box-shadow: 0 0 8px rgba(255, 158, 181, 0.4); }

/* 電腦螢幕上下分區，上方為表單與清單，下方為全寬月曆 */
.main-content { display: flex; flex-direction: column; gap: 40px; }
.content-left { display: grid; grid-template-columns: 360px 1fr; gap: 30px; align-items: start; }
.content-right { display: block; }

/* 限制上方清單高度，讓畫面能快速看到下方的月曆 */
.events-list { max-height: 480px; overflow-y: auto; }

/* 12個月大表格排版 - 加大最小寬度，讓單個月曆變成寬寬橫橫的比例 */
.months-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(700px, 1fr)); gap: 40px; }

/* HTML 原生真表格 CSS 樣式 */
.calendar-view { background: #fff0f3; padding: 20px; border-radius: 20px; border: 3px solid #ffb3c6; box-shadow: 6px 6px 0px #ffe4e8; }
.calendar-header { display: flex; justify-content: center; align-items: center; margin-bottom: 15px; background: #fff; padding: 10px; border-radius: 12px; border: 2px dashed #ffb3c6; }
.calendar-header h2 { margin: 0; font-size: 1.5em; color: #e87a90; text-shadow: 1px 1px 0px #ffe4e8; }

.calendar-table { width: 100%; border-collapse: collapse; background: #fffdfa; border-style: hidden; border-radius: 12px; box-shadow: 0 0 0 2px #ffb3c6; overflow: hidden; }
.calendar-table th, .calendar-table td { border: 2px solid #ffb3c6; }
.calendar-table th { background: #ffe4e8; color: #d67085; font-weight: 900; padding: 12px 0; font-size: 1.1em; }
.calendar-table td { height: 85px; vertical-align: top; padding: 8px; transition: all 0.2s; width: 14.28%; }
.calendar-table td.has-date:hover { background: #fff0f3; cursor: pointer; box-shadow: inset 0 0 15px rgba(255, 158, 181, 0.4); }

.cell-date { font-weight: 900; margin-bottom: 6px; font-size: 1.1em; color: #b8687b; text-align: left; padding-left: 4px; }
.cell-events { display: flex; flex-direction: column; gap: 4px; }
.mini-event { background: #ff9eb5; color: white; font-size: 12px; padding: 4px 6px; border-radius: 8px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; box-shadow: 2px 2px 0px rgba(232, 122, 144, 0.3); border: 1px solid #e87a90; }

.form-card, .events-list { background: #fffdfa; padding: 20px; border-radius: 20px; border: 3px solid #ffb3c6; box-shadow: 4px 4px 0px #ffe4e8; }
.form-group { margin-bottom: 15px; }
.form-group label { display: block; margin-bottom: 5px; font-weight: 900; color: #e87a90; font-size: 1.1em; }
.form-group input, .form-group textarea { width: 100%; padding: 10px; border: 2px solid #ffd1dc; border-radius: 12px; background: #fff; color: #7a4b56; outline: none; font-family: inherit; font-weight: bold; }
.form-group input:focus, .form-group textarea:focus { border-color: #ff9eb5; box-shadow: 0 0 5px rgba(255, 158, 181, 0.3); }
.btn { padding: 10px 18px; border: none; border-radius: 20px; cursor: pointer; background: #ffb3c6; color: #fff; font-weight: 900; margin-right: 10px; transition: all 0.2s; box-shadow: 2px 2px 0px rgba(232, 122, 144, 0.3); letter-spacing: 1px; }
.btn-primary { background: #ff9eb5; width: 100%; font-size: 16px; margin-top: 10px; }
.btn:hover { background: #ff7ea5; transform: translate(-2px, -2px); box-shadow: 4px 4px 0px rgba(232, 122, 144, 0.3); }
.file-label { display: inline-block; cursor: pointer; background: #e87a90; }
.event-card { background: #fff; padding: 15px; margin-bottom: 15px; border-radius: 12px; border-left: 5px solid #ff9eb5; box-shadow: 0 2px 6px rgba(255, 209, 220, 0.4); }
.event-header { display: flex; justify-content: space-between; align-items: center; }
.event-header h3 { margin: 0; color: #e87a90; }
.badge { background: #ffb3c6; color: white; padding: 4px 10px; border-radius: 15px; font-size: 12px; font-weight: bold; }
.event-time { color: #b8687b; font-size: 0.9em; margin: 8px 0; background: #fff0f3; display: inline-block; padding: 3px 8px; border-radius: 10px; }
.empty { color: #d67085; text-align: center; padding: 20px; font-style: italic; }

/* 自訂滾動條，配合可愛風格 */
::-webkit-scrollbar { width: 10px; height: 10px; }
::-webkit-scrollbar-track { background: #fff5f7; border-radius: 10px; }
::-webkit-scrollbar-thumb { background: #ffd1dc; border-radius: 10px; }
::-webkit-scrollbar-thumb:hover { background: #ff9eb5; }

/* =========================================
   響應式設計 (RWD) - 平板與手機版面適應
========================================= */
@media (max-width: 1024px) {
  .content-left { grid-template-columns: 1fr; } /* 上方表單與清單改為單欄上下堆疊 */
  .months-grid { grid-template-columns: 1fr; } /* 月曆在小螢幕強制單欄滿版 */
  .events-list { max-height: none; overflow-y: visible; } /* 小螢幕取消高度限制 */
  h1 { font-size: 2.2em; }
}

@media (max-width: 600px) {
  .cute-theme { padding: 15px; }
  h1 { font-size: 1.8em; margin-bottom: 15px; letter-spacing: 1px; }
  .toolbar { flex-direction: column; gap: 15px; align-items: stretch; text-align: center; }
  .settings input { width: 100%; box-sizing: border-box; margin-top: 8px; }
  .calendar-table td { height: 80px; padding: 3px; } /* 縮小手機版格子高度 */
  .cell-date { font-size: 0.95em; }
  .mini-event { font-size: 10px; padding: 2px 4px; box-shadow: 1px 1px 0px rgba(232, 122, 144, 0.3); }
  .weekday { font-size: 0.85em; padding: 6px 0; }
  .calendar-view { padding: 10px; border-width: 2px; }
}

@media (max-width: 400px) {
  .calendar-table td { height: 75px; } /* 極小螢幕再次縮減高度避免變形 */
  .mini-event { font-size: 9px; }
}
</style>