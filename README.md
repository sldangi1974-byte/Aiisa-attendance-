<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Attendance — Export PDF</title>
<style>
  body{font-family: Arial, Helvetica, sans-serif; margin:12px; background:#f7f7f7; color:#222;}
  h1{font-size:18px; margin:6px 0 12px;}
  .controls{display:flex;gap:8px;flex-wrap:wrap;align-items:center;margin-bottom:10px;}
  select,input[type=number]{padding:6px;border-radius:4px;border:1px solid #bbb;}
  button{padding:8px 10px;border-radius:6px;border:0;background:#4CAF50;color:#fff;cursor:pointer;}
  button.secondary{background:#1976d2;}
  button.warn{background:#f44336;}
  button.grey{background:#777;}
  .calendar{width:100%;overflow:auto;background:#fff;border-radius:6px;padding:8px;box-shadow:0 1px 4px rgba(0,0,0,0.08);}
  table{width:100%; border-collapse:collapse;}
  th,td{border:1px solid #e0e0e0;padding:6px;text-align:left;vertical-align:top;font-size:13px;}
  th{background:#f0f0f0;position:sticky;top:0;z-index:2;}
  .row-date{min-width:90px;font-weight:600;background:#fcfcfc;}
  .presence-btn{display:inline-block;padding:4px 8px;border-radius:4px;border:1px solid #999;cursor:pointer;background:#e6f7e6;color:#0a5;}
  .absent{background:#ffe6e6;color:#b00;border-color:#d66;}
  input[type=text]{width:100%;padding:4px;border:1px solid #ccc;border-radius:4px;box-sizing:border-box;}
  .time-input{width:90px;}
  .small{font-size:12px;color:#666;}
  @media (max-width:700px){
    th,td{font-size:12px;padding:5px;}
    .time-input{width:66px;}
  }
</style>
</head>
<body>

<h1>Attendance — Monthly sheet (PDF export included)</h1>

<div class="controls">
  <label for="month">Month</label>
  <select id="month"></select>

  <label for="year">Year</label>
  <select id="year"></select>

  <button id="generate" class="secondary">Generate Calendar</button>
  <button id="save">Save (local)</button>
  <button id="clear" class="grey">Clear Month</button>
  <button id="exportCsv">Export CSV</button>
  <button id="exportPdf">Export PDF</button>
  <button id="resetAll" class="warn">Reset ALL Data</button>
</div>

<div class="calendar" id="calendarArea"></div>

<!-- jsPDF and autoTable -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf-autotable/3.5.28/jspdf.plugin.autotable.min.js"></script>

<script>
const monthNames = ["January","February","March","April","May","June","July","August","September","October","November","December"];
const dayNames = ["Sunday","Monday","Tuesday","Wednesday","Thursday","Friday","Saturday"];

function getStorageKey(month, year){
  return `attendance-${year}-${String(month).padStart(2,'0')}`;
}

const monthSel = document.getElementById('month');
const yearSel = document.getElementById('year');
const now = new Date();
for(let i=0;i<12;i++){ const opt=document.createElement('option'); opt.value=i; opt.textContent=monthNames[i]; monthSel.appendChild(opt); }
for(let y= now.getFullYear()-3; y<= now.getFullYear()+3; y++){ const opt=document.createElement('option'); opt.value=y; opt.textContent=y; yearSel.appendChild(opt); }
monthSel.value = now.getMonth(); yearSel.value = now.getFullYear();

function daysInMonth(m,y){ return new Date(y,m+1,0).getDate(); }
function loadMonthData(m,y){ const raw = localStorage.getItem(getStorageKey(m+1,y)); return raw ? JSON.parse(raw) : {}; }
function saveMonthData(m,y,data){ localStorage.setItem(getStorageKey(m+1,y), JSON.stringify(data)); }

function buildCalendar(){
  const m = parseInt(monthSel.value,10);
  const y = parseInt(yearSel.value,10);
  const total = daysInMonth(m,y);
  const stored = loadMonthData(m,y);
  const container = document.getElementById('calendarArea');
  container.innerHTML = '';

  const tbl = document.createElement('table');
  tbl.innerHTML = `<thead><tr>
    <th class="row-date">Date</th>
    <th>Day</th>
    <th>Presence</th>
    <th>Time In</th>
    <th>Time Out</th>
    <th>Location</th>
    <th>Work</th>
    <th>Remarks</th>
  </tr></thead>`;
  const tbody = document.createElement('tbody');

  for(let d=1; d<=total; d++){
    const dt = new Date(y,m,d);
    const dayName = dayNames[dt.getDay()];
    const key = `d${d}`;
    const rowData = stored[key] || {presence:'', timeIn:'', timeOut:'', location:'', work:'', remarks:''};
    const tr = document.createElement('tr');

    tr.innerHTML = `
      <td class="row-date"><div>${d}-${String(m+1).padStart(2,'0')}-${y}</div><div class="small">${dayName}</div></td>
      <td>${dayName}</td>
      <td><button class="presence-btn ${rowData.presence==='Absent'?'absent':''}">${rowData.presence || 'Mark'}</button></td>
      <td><input type="text" class="time-input" value="${rowData.timeIn}" placeholder="timeIn"></td>
      <td><input type="text" class="time-input" value="${rowData.timeOut}" placeholder="timeOut"></td>
      <td><input type="text" value="${rowData.location}" placeholder="location"></td>
      <td><input type="text" value="${rowData.work}" placeholder="work"></td>
      <td><input type="text" value="${rowData.remarks}" placeholder="remarks"></td>
    `;

    const btn = tr.querySelector('.presence-btn');
    btn.addEventListener('click', ()=>{
      if(rowData.presence === '') rowData.presence = 'Present';
      else if(rowData.presence === 'Present') rowData.presence = 'Absent';
      else rowData.presence = '';
      btn.textContent = rowData.presence || 'Mark';
      btn.classList.toggle('absent', rowData.presence === 'Absent');
      stored[key] = rowData;
      saveMonthData(m,y,stored);
    });

    tr.querySelectorAll('input').forEach(inp=>{
      inp.addEventListener('input', ()=>{
        rowData[inp.placeholder] = inp.value;
        stored[key] = rowData;
        saveMonthData(m,y,stored);
      });
    });

    tbody.appendChild(tr);
  }
  tbl.appendChild(tbody);
  container.appendChild(tbl);
}

document.getElementById('generate').addEventListener('click', buildCalendar);
document.getElementById('save').addEventListener('click', ()=>alert('Saved to browser storage.'));
document.getElementById('clear').addEventListener('click', ()=>{
  if(confirm('Clear this month?')){
    const m = parseInt(monthSel.value,10), y = parseInt(yearSel.value,10);
    localStorage.removeItem(getStorageKey(m+1,y));
    buildCalendar();
  }
});
document.getElementById('resetAll').addEventListener('click', ()=>{
  if(confirm('Reset ALL data?')){
    Object.keys(localStorage).forEach(k=>{ if(k.startsWith('attendance-')) localStorage.removeItem(k); });
    buildCalendar();
  }
});

/* PDF Export — Single Page Optimized */
document.getElementById('exportPdf').addEventListener('click', ()=>{
  const { jsPDF } = window.jspdf;
  const doc = new jsPDF('l','pt','a4');
  const m = parseInt(monthSel.value,10), y = parseInt(yearSel.value,10);
  const stored = loadMonthData(m,y);
  const total = daysInMonth(m,y);

  const body = [];
  for(let d=1; d<=total; d++){
    const dt = new Date(y,m,d);
    const r = stored[`d${d}`] || {};
    body.push([
      `${d}-${String(m+1).padStart(2,'0')}-${y}`,
      dayNames[dt.getDay()],
      r.presence || '',
      r.timeIn || '',
      r.timeOut || '',
      r.location || '',
      r.work || '',
      r.remarks || ''
    ]);
  }

  doc.setFontSize(14);
  doc.text(`Attendance — ${monthNames[m]} ${y}`, 40, 30);

  doc.autoTable({
    head: [['Date','Day','Presence','In','Out','Location','Work','Remarks']],
    body: body,
    startY: 50,
    styles: { fontSize: 8, cellPadding: 2 },
    theme: 'grid',
    headStyles: { fillColor: [240,240,240] },
    columnStyles: {
      0: { cellWidth: 55 },
      1: { cellWidth: 50 },
      2: { cellWidth: 45 },
      3: { cellWidth: 35 },
      4: { cellWidth: 35 },
      5: { cellWidth: 100 },
      6: { cellWidth: 120 },
      7: { cellWidth: 160 }
    },
    tableWidth: 'wrap',
    didParseCell: function(data){
      if(data.column.index === 2){
        const val = (data.cell.raw || '').toString().toLowerCase();
        if(val.includes('present')) data.cell.styles.fillColor = [217,255,217];
        if(val.includes('absent')) data.cell.styles.fillColor = [255,220,220];
      }
    }
  });

  doc.save(`attendance-${String(m+1).padStart(2,'0')}-${y}.pdf`);
});

buildCalendar();
</script>
</body>
</html>
