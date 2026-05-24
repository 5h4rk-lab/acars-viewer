# Chart Tab + Rotating Plane Icon Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a dedicated Chart tab (altitude/speed/fuel over time via Canvas 2D) and a rotating top-down airplane silhouette at the last known position on the Leaflet map.

**Architecture:** All changes in `index.html`. Chart uses the vanilla Canvas 2D API — no external library. Plane icon uses Leaflet `divIcon` with inline SVG rotated via CSS `transform: rotate(${hdg}deg)`. No new files, no new dependencies.

**Tech Stack:** Vanilla JS, Canvas 2D API, Leaflet 1.9.4 (already loaded), inline SVG

---

### Task 1: Add Chart tab HTML structure + update clearAll

**Files:**
- Modify: `index.html:114-124` (detail panel tabs + panes)
- Modify: `index.html:607` (clearAll tab reset array)

- [ ] **Step 1: Add Chart tab button**

In `index.html`, find the `.detail-tabs` div. Add the Chart tab after the Raw tab:

```html
<!-- Find: -->
    <div class="dtab" data-tab="raw">Raw</div>
<!-- Add immediately after: -->
    <div class="dtab" data-tab="chart">Chart</div>
```

- [ ] **Step 2: Add Chart tab pane**

Find the last tab pane. Add Chart pane after it:

```html
<!-- Find: -->
  <div class="tab-pane" id="tab-raw"><div class="empty">SELECT A FLIGHT</div></div>
<!-- Add immediately after: -->
  <div class="tab-pane" id="tab-chart"><div class="empty">SELECT A FLIGHT</div></div>
```

- [ ] **Step 3: Update clearAll to reset Chart tab**

Find this line in `clearAll()` (around line 607):
```javascript
['tab-info','tab-decoded','tab-timeline','tab-raw'].forEach(id=>document.getElementById(id).innerHTML='<div class="empty">SELECT A FLIGHT</div>');
```

Replace with:
```javascript
['tab-info','tab-decoded','tab-timeline','tab-raw','tab-chart'].forEach(id=>document.getElementById(id).innerHTML='<div class="empty">SELECT A FLIGHT</div>');
```

- [ ] **Step 4: Open in browser and verify**

Open `index.html` in a browser. Verify:
- 5 tabs visible: Info | Decoded | Timeline | Raw | Chart
- Clicking Chart shows "SELECT A FLIGHT" empty state
- All other tabs still work

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add Chart tab HTML structure to detail panel"
```

---

### Task 2: Implement drawChart() function

**Files:**
- Modify: `index.html` — add `drawChart(f)` after the `labelDesc()` function (around line 548)

- [ ] **Step 1: Add drawChart() after labelDesc()**

Find the closing brace of `labelDesc()` (around line 548). Add immediately after:

```javascript
// ── CHART ─────────────────────────────────────────────────────────
function drawChart(f) {
  const container = document.getElementById('tab-chart');
  if (!f.pos.length) {
    container.innerHTML = '<div class="empty">NO POSITION DATA</div>';
    return;
  }
  container.innerHTML = '<canvas id="chart-canvas" style="display:block"></canvas>';
  const canvas = document.getElementById('chart-canvas');
  const W = document.getElementById('detail').clientWidth || 320;
  const H = 220;
  canvas.width = W;
  canvas.height = H;
  const ctx = canvas.getContext('2d');

  const pts = f.pos.map(p => ({
    t: new Date(p.ts.replace(/\//g, '-')).getTime(),
    alt: p.alt, tas: p.tas, fob: p.fob
  })).filter(p => !isNaN(p.t));

  if (!pts.length) { container.innerHTML = '<div class="empty">NO POSITION DATA</div>'; return; }

  const PAD = { top: 28, right: 58, bottom: 22, left: 48 };
  const cW = W - PAD.left - PAD.right;
  const cH = H - PAD.top - PAD.bottom;

  const tMin = pts[0].t, tMax = pts[pts.length - 1].t;
  const tRange = tMax - tMin || 1;

  const alts  = pts.map(p => p.alt).filter(v => v != null);
  const tases = pts.map(p => p.tas).filter(v => v != null);
  const fobs  = pts.map(p => p.fob).filter(v => v != null);

  const altMax   = alts.length  ? Math.max(...alts)  * 1.1 : 40000;
  const speedMax = tases.length ? Math.max(...tases) * 1.1 : 600;
  const fobMax   = fobs.length  ? Math.max(...fobs)  * 1.1 : 100000;

  const xOf  = t => PAD.left + (t - tMin) / tRange * cW;
  const yAlt = v => PAD.top + cH - (v / altMax)   * cH;
  const ySpd = v => PAD.top + cH - (v / speedMax) * cH;
  const yFob = v => PAD.top + cH - (v / fobMax)   * cH;

  // Grid
  ctx.lineWidth = 0.5;
  [0.25, 0.5, 0.75].forEach(frac => {
    ctx.strokeStyle = '#1a2332';
    ctx.setLineDash([]);
    ctx.beginPath();
    ctx.moveTo(PAD.left, PAD.top + cH * (1 - frac));
    ctx.lineTo(PAD.left + cW, PAD.top + cH * (1 - frac));
    ctx.stroke();
  });

  // Altitude filled area
  if (alts.length) {
    const ap = pts.filter(p => p.alt != null);
    ctx.beginPath();
    ctx.moveTo(xOf(ap[0].t), yAlt(ap[0].alt));
    ap.forEach(p => ctx.lineTo(xOf(p.t), yAlt(p.alt)));
    ctx.lineTo(xOf(ap[ap.length - 1].t), PAD.top + cH);
    ctx.lineTo(xOf(ap[0].t), PAD.top + cH);
    ctx.closePath();
    ctx.fillStyle = 'rgba(123,104,238,0.12)';
    ctx.fill();
    ctx.beginPath();
    ctx.moveTo(xOf(ap[0].t), yAlt(ap[0].alt));
    ap.forEach(p => ctx.lineTo(xOf(p.t), yAlt(p.alt)));
    ctx.strokeStyle = '#7b68ee';
    ctx.lineWidth = 2;
    ctx.setLineDash([]);
    ctx.stroke();
  }

  // Speed dashed line
  if (tases.length) {
    const sp = pts.filter(p => p.tas != null);
    ctx.beginPath();
    ctx.moveTo(xOf(sp[0].t), ySpd(sp[0].tas));
    sp.forEach(p => ctx.lineTo(xOf(p.t), ySpd(p.tas)));
    ctx.strokeStyle = '#00d4ff';
    ctx.lineWidth = 1.5;
    ctx.setLineDash([5, 3]);
    ctx.stroke();
  }

  // Fuel dashed line
  if (fobs.length) {
    const fp = pts.filter(p => p.fob != null);
    ctx.beginPath();
    ctx.moveTo(xOf(fp[0].t), yFob(fp[0].fob));
    fp.forEach(p => ctx.lineTo(xOf(p.t), yFob(p.fob)));
    ctx.strokeStyle = '#ffcc00';
    ctx.lineWidth = 1.5;
    ctx.setLineDash([3, 4]);
    ctx.stroke();
  }

  ctx.setLineDash([]);

  // Left Y-axis labels (altitude in FL)
  ctx.font = '9px monospace';
  ctx.textAlign = 'right';
  [0.25, 0.5, 0.75, 1].forEach(frac => {
    ctx.fillStyle = '#7b68ee';
    ctx.fillText('FL' + Math.round(altMax * frac / 100), PAD.left - 4, PAD.top + cH * (1 - frac) + 3);
  });

  // Right Y-axis labels (speed in kts)
  if (tases.length) {
    ctx.fillStyle = '#00d4ff';
    ctx.textAlign = 'left';
    [0.5, 1].forEach(frac => {
      ctx.fillText(Math.round(speedMax * frac) + 'kt', PAD.left + cW + 4, PAD.top + cH * (1 - frac) + 3);
    });
  }

  // X-axis time labels
  const fmt = t => new Date(t).toISOString().slice(11, 16);
  ctx.fillStyle = '#4a6080';
  ctx.font = '8px monospace';
  ctx.textAlign = 'center';
  ctx.fillText(fmt(tMin), PAD.left, H - 4);
  ctx.fillText(fmt((tMin + tMax) / 2), PAD.left + cW / 2, H - 4);
  ctx.fillText(fmt(tMax), PAD.left + cW, H - 4);

  // Legend
  let lx = PAD.left;
  const legends = [];
  if (alts.length)  legends.push({ color: '#7b68ee', label: 'Alt',   dash: false });
  if (tases.length) legends.push({ color: '#00d4ff', label: 'Speed', dash: true  });
  if (fobs.length)  legends.push({ color: '#ffcc00', label: 'Fuel',  dash: true  });
  ctx.font = '9px monospace';
  legends.forEach(leg => {
    ctx.strokeStyle = leg.color;
    ctx.lineWidth = 1.5;
    ctx.setLineDash(leg.dash ? [4, 2] : []);
    ctx.beginPath(); ctx.moveTo(lx, 14); ctx.lineTo(lx + 14, 14); ctx.stroke();
    ctx.setLineDash([]);
    ctx.fillStyle = leg.color;
    ctx.textAlign = 'left';
    ctx.fillText(leg.label, lx + 17, 18);
    lx += 14 + ctx.measureText(leg.label).width + 18;
  });
}
```

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "feat: implement drawChart() — altitude/speed/fuel canvas chart"
```

---

### Task 3: Wire drawChart into renderDetail() and tab click handler

**Files:**
- Modify: `index.html` — end of `renderDetail()` (around line 494)
- Modify: `index.html` — tab click handler (around line 611)

- [ ] **Step 1: Call drawChart at end of renderDetail()**

Find the end of `renderDetail(key)`. The last block sets `#tab-raw`. Add `drawChart(f)` after it:

```javascript
  // RAW TAB
  const raw = f.msgs.slice(0,50).map(m=>`
    <div class="msg-block">
      <div class="msg-time">${m.ts} · ${m.freq}MHz · ${m.lvl}dBm · E:${m.err}</div>
      <span class="msg-label">${m.label}</span>
      <div class="msg-body">${(m.body||'').trim().slice(0,300)}</div>
    </div>`).join('');
  document.getElementById('tab-raw').innerHTML = raw||'<div class="empty">NO MESSAGES</div>';

  // CHART TAB
  drawChart(f);
}
```

- [ ] **Step 2: Re-draw chart when Chart tab is clicked**

The canvas width uses `detail.clientWidth` so it works when the pane is hidden, but re-drawing on tab click ensures correct sizing. Find the tab click handler (around line 611):

```javascript
document.querySelectorAll('.dtab').forEach(t=>t.addEventListener('click',()=>{
  const tab=t.dataset.tab;
  document.querySelectorAll('.dtab').forEach(x=>x.classList.remove('active'));
  document.querySelectorAll('.tab-pane').forEach(x=>x.classList.remove('active'));
  t.classList.add('active');
  document.getElementById('tab-'+tab).classList.add('active');
}));
```

Replace with:

```javascript
document.querySelectorAll('.dtab').forEach(t=>t.addEventListener('click',()=>{
  const tab=t.dataset.tab;
  document.querySelectorAll('.dtab').forEach(x=>x.classList.remove('active'));
  document.querySelectorAll('.tab-pane').forEach(x=>x.classList.remove('active'));
  t.classList.add('active');
  document.getElementById('tab-'+tab).classList.add('active');
  if (tab==='chart' && selected) drawChart(flights[selected]);
}));
```

- [ ] **Step 3: Verify in browser**

Load a log file. Select a flight with position data. Click the Chart tab. Verify:
- Canvas renders with purple altitude area
- Cyan dashed speed line visible (if TAS data present in log)
- Yellow dashed fuel line visible (if FOB data present in log)
- FL labels on left Y-axis, knot labels on right (if speed present)
- Time labels (HH:MM) on X-axis at start, mid, end
- Legend strip at top of canvas
- Select a flight with no position data → "NO POSITION DATA" shown

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: wire Chart tab — drawChart on select and on tab click"
```

---

### Task 4: Add airplaneIcon() helper + update renderMap()

**Files:**
- Modify: `index.html` — add `airplaneIcon()` after `planeIcon()` (around line 315)
- Modify: `index.html` — `renderMap()` position markers forEach (around line 342)

- [ ] **Step 1: Add airplaneIcon() helper after planeIcon()**

Find the `planeIcon()` function (around line 310). Add `airplaneIcon()` immediately after its closing brace:

```javascript
function airplaneIcon(color, hdg) {
  return L.divIcon({
    html: `<div style="transform:rotate(${hdg||0}deg);width:38px;height:38px;display:flex;align-items:center;justify-content:center">
      <svg width="38" height="38" viewBox="0 0 38 38">
        <rect x="17" y="6" width="4" height="26" rx="2" fill="${color}"/>
        <path d="M19 16 L4 24 L4 22 L19 14 L34 22 L34 24 Z" fill="${color}"/>
        <path d="M19 28 L12 34 L12 32 L19 26 L26 32 L26 34 Z" fill="${color}"/>
      </svg>
    </div>`,
    className: '',
    iconAnchor: [19, 19]
  });
}
```

- [ ] **Step 2: Update renderMap() position markers forEach**

Find the `f.pos.forEach` block inside `renderMap()` (around line 342). Replace the entire forEach with:

```javascript
  f.pos.forEach((p,i)=>{
    const isFirst=i===0, isLast=i===f.pos.length-1;
    const color=isFirst?'#39ff14':isLast?altColor(p.alt):'#00d4ff';
    const r=(isFirst||isLast)?9:5;
    const popup=`<div style="font-family:monospace;font-size:11px;background:#0d1117;color:#c8d8e8;padding:8px;min-width:150px">
      <b style="color:${color}">${f.flight} · ${p.ts.split(' ')[1]||''}</b><br>
      ${p.alt?`Alt: <b>${p.alt.toLocaleString()} ft</b><br>`:''}
      ${p.tas?`Speed: ${p.tas} kts<br>`:''}
      ${p.hdg?`Heading: ${p.hdg}°<br>`:''}
      ${p.fob?`Fuel: ${p.fob.toLocaleString()} lbs<br>`:''}
      ${p.eta?`ETA: ${p.eta} UTC<br>`:''}
      <span style="color:#4a6080;font-size:9px">✅ CONFIRMED by your SDR</span>
    </div>`;
    let m;
    if (isLast && p.hdg != null) {
      m=L.marker([p.lat,p.lon],{icon:airplaneIcon(color,p.hdg)})
        .bindPopup(popup,{className:'dark-popup'}).addTo(map);
    } else {
      m=L.circleMarker([p.lat,p.lon],{radius:r,color,fillColor:color,fillOpacity:1,weight:2})
        .bindPopup(popup,{className:'dark-popup'}).addTo(map);
    }
    markers['p'+i]=m;
  });
```

- [ ] **Step 3: Verify in browser**

Load a log file. Select a flight with heading data (labels 80, 21 have heading). Verify:
- Last position shows airplane silhouette rotated to match heading
- Clicking airplane shows popup with all flight data
- Intermediate positions still show blue circles
- First position still shows green circle
- Select a flight with no heading data → last position stays as orange/colored circle

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: rotating airplane silhouette icon at last known position"
```

---

### Task 5: Update renderAllMap() + final verification

**Files:**
- Modify: `index.html` — `renderAllMap()` inner forEach (around line 411)

- [ ] **Step 1: Update renderAllMap() last-position markers**

Find the inner `coords.forEach` inside `renderAllMap()` (around line 411):

```javascript
    coords.forEach((c,i)=>{
      const isLast=i===coords.length-1;
      L.circleMarker(c,{radius:isLast?7:4,color,fillColor:color,fillOpacity:1,weight:2})
        .bindPopup(`<div style="font-family:monospace;font-size:11px;background:#0d1117;color:#c8d8e8;padding:6px"><b style="color:${color}">${f.flight}</b><br>${f.route||''}<br>Alt: ${f.pos[i].alt?f.pos[i].alt.toLocaleString()+' ft':'N/A'}</div>`,{className:'dark-popup'}).addTo(map);
    });
```

Replace with:

```javascript
    coords.forEach((c,i)=>{
      const isLast=i===coords.length-1;
      const p=f.pos[i];
      const popup=`<div style="font-family:monospace;font-size:11px;background:#0d1117;color:#c8d8e8;padding:6px"><b style="color:${color}">${f.flight}</b><br>${f.route||''}<br>Alt: ${p.alt?p.alt.toLocaleString()+' ft':'N/A'}</div>`;
      if (isLast && p.hdg != null) {
        L.marker(c,{icon:airplaneIcon(color,p.hdg)})
          .bindPopup(popup,{className:'dark-popup'}).addTo(map);
      } else {
        L.circleMarker(c,{radius:isLast?7:4,color,fillColor:color,fillOpacity:1,weight:2})
          .bindPopup(popup,{className:'dark-popup'}).addTo(map);
      }
    });
```

- [ ] **Step 2: Final browser verification**

Load a log file with multiple flights. Verify:
- "Show All" mode: each flight's last position shows airplane (if heading available), intermediate positions are small colored circles, callsign labels still appear
- Select individual flights → cycle through all 5 tabs, no console errors
- Chart tab shows correct data per flight
- Clear button resets everything including Chart tab
- Flights with no position data: map shows destination marker, Chart tab shows "NO POSITION DATA"

Open browser dev console. Confirm zero JS errors across all interactions.

- [ ] **Step 3: Final commit**

```bash
git add index.html
git commit -m "feat: airplane icon in Show All mode — chart + icon integration complete"
```
