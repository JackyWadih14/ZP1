# ZP1 幸运大转盘 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single-file spinning wheel web app with SVG-based stationary wheel, rotating pointer, customizable awards with weights, and localStorage persistence.

**Architecture:** Single `index.html` with embedded CSS/JS. SVG renders the wheel segments and pointer. A slide-out side panel manages award config. JS drives pointer rotation via requestAnimationFrame with ease-out cubic easing, and uses cumulative-weight random selection for fair probability-based results.

**Tech Stack:** HTML5, CSS3, SVG, vanilla JavaScript (no dependencies)

---

### Task 1: HTML scaffold and layout

**Files:**
- Create: `E:\AIproject\ZP1\index.html`

- [ ] **Step 1: Write HTML scaffold**

Replace the existing index.html with the new structure:

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>幸运大转盘</title>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }

  body {
    min-height: 100vh;
    display: flex;
    align-items: center;
    justify-content: center;
    background: linear-gradient(135deg, #1a1a2e 0%, #16213e 50%, #0f3460 100%);
    font-family: 'Microsoft YaHei', 'PingFang SC', sans-serif;
    user-select: none;
    overflow-x: hidden;
  }

  .app {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 20px;
  }

  h1 {
    color: #ffd700;
    font-size: 2rem;
    letter-spacing: 4px;
    text-shadow: 0 0 20px rgba(255, 215, 0, 0.4);
  }

  .main-area {
    display: flex;
    align-items: center;
    gap: 24px;
  }

  .wheel-wrapper {
    position: relative;
    width: 420px;
    height: 420px;
  }

  .result {
    min-height: 40px;
    font-size: 1.2rem;
    color: #ffd700;
    letter-spacing: 2px;
  }
  .result.show {
    animation: popIn 0.4s ease;
  }
  @keyframes popIn {
    0%   { transform: scale(0.5); opacity: 0; }
    60%  { transform: scale(1.15); }
    100% { transform: scale(1); opacity: 1; }
  }

  .btn-gear {
    position: absolute;
    right: -48px;
    top: 12px;
    width: 36px;
    height: 36px;
    background: rgba(255,255,255,0.1);
    border: 1px solid rgba(255,255,255,0.2);
    border-radius: 50%;
    color: #ffd700;
    font-size: 20px;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: background 0.2s;
    z-index: 5;
  }
  .btn-gear:hover { background: rgba(255,255,255,0.2); }
</style>
</head>
<body>
<div class="app">
  <h1>🎯 幸运大转盘</h1>
  <div class="main-area">
    <div class="wheel-wrapper">
      <button class="btn-gear" id="btnGear" title="设置">⚙</button>
      <svg id="wheelSvg" viewBox="0 0 400 400" width="420" height="420"></svg>
    </div>
    <div class="panel" id="panel"></div>
  </div>
  <div class="result" id="result"></div>
</div>
<script>
// --- Data ---

// --- Rendering ---

// --- Animation ---

// --- Panel ---

// --- Init ---
</script>
</body>
</html>
```

- [ ] **Step 2: Open in browser to verify layout**

Open `E:\AIproject\ZP1\index.html` in browser. You should see: dark gradient background, gold title, empty SVG area, gear button, and result zone.

---

### Task 2: SVG wheel rendering

**Files:**
- Modify: `E:\AIproject\ZP1\index.html`

- [ ] **Step 1: Add default data and SVG drawing logic**

Replace the `// --- Data ---` section with:

```js
const DEFAULT_SEGMENTS = [
  { label: '一等奖',   color: '#ff6b6b', weight: 1 },
  { label: '二等奖',   color: '#ffa502', weight: 1 },
  { label: '三等奖',   color: '#ffd700', weight: 1 },
  { label: '参与奖',   color: '#7bed9f', weight: 1 },
  { label: '谢谢参与', color: '#70a1ff', weight: 1 },
  { label: '幸运奖',   color: '#a29bfe', weight: 1 },
  { label: '特等奖',   color: '#ff6348', weight: 1 },
  { label: '再接再厉', color: '#1e90ff', weight: 1 },
];

const STORAGE_KEY = 'zp1_wheel_segments';

function loadSegments() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (raw) return JSON.parse(raw);
  } catch (_) {}
  return DEFAULT_SEGMENTS.map(s => ({ ...s }));
}

function saveSegments(segs) {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(segs));
}
```

Replace the `// --- Rendering ---` section with:

```js
const svg = document.getElementById('wheelSvg');
const NS = 'http://www.w3.org/2000/svg';

const CX = 200, CY = 200, R = 190;

function polarToCartesian(cx, cy, r, angle) {
  return {
    x: cx + r * Math.cos(angle),
    y: cy + r * Math.sin(angle),
  };
}

function describeArc(cx, cy, r, startAngle, endAngle) {
  const s = polarToCartesian(cx, cy, r, endAngle);
  const e = polarToCartesian(cx, cy, r, startAngle);
  const large = endAngle - startAngle > Math.PI ? 1 : 0;
  return `M${cx},${cy} L${e.x},${e.y} A${r},${r} 0 ${large} 0 ${s.x},${s.y} Z`;
}

function renderWheel(segments) {
  svg.innerHTML = '';
  const count = segments.length;
  const arc = (2 * Math.PI) / count;

  // Draw segments
  for (let i = 0; i < count; i++) {
    const startAngle = i * arc - Math.PI / 2;
    const endAngle = startAngle + arc;

    const path = document.createElementNS(NS, 'path');
    path.setAttribute('d', describeArc(CX, CY, R, startAngle, endAngle));
    path.setAttribute('fill', segments[i].color);
    path.setAttribute('stroke', 'rgba(255,255,255,0.25)');
    path.setAttribute('stroke-width', '2');
    svg.appendChild(path);

    // Label
    const midAngle = startAngle + arc / 2;
    const labelR = R * 0.62;
    const tx = CX + labelR * Math.cos(midAngle);
    const ty = CY + labelR * Math.sin(midAngle);

    const text = document.createElementNS(NS, 'text');
    text.setAttribute('x', tx);
    text.setAttribute('y', ty);
    text.setAttribute('text-anchor', 'middle');
    text.setAttribute('dominant-baseline', 'central');
    text.setAttribute('fill', '#fff');
    text.setAttribute('font-size', '14');
    text.setAttribute('font-weight', 'bold');
    text.setAttribute('font-family', 'Microsoft YaHei, PingFang SC, sans-serif');
    text.textContent = segments[i].label;
    svg.appendChild(text);
  }

  // Outer ring
  const ring = document.createElementNS(NS, 'circle');
  ring.setAttribute('cx', CX);
  ring.setAttribute('cy', CY);
  ring.setAttribute('r', R);
  ring.setAttribute('fill', 'none');
  ring.setAttribute('stroke', 'rgba(255,215,0,0.5)');
  ring.setAttribute('stroke-width', '4');
  svg.appendChild(ring);

  // Center button
  const btnCircle = document.createElementNS(NS, 'circle');
  btnCircle.setAttribute('cx', CX);
  btnCircle.setAttribute('cy', CY);
  btnCircle.setAttribute('r', '36');
  btnCircle.setAttribute('fill', '#fff');
  btnCircle.setAttribute('stroke', '#ccc');
  btnCircle.setAttribute('stroke-width', '3');
  btnCircle.setAttribute('cursor', 'pointer');
  btnCircle.setAttribute('id', 'btnCenter');
  svg.appendChild(btnCircle);

  const btnText = document.createElementNS(NS, 'text');
  btnText.setAttribute('x', CX);
  btnText.setAttribute('y', CY + 2);
  btnText.setAttribute('text-anchor', 'middle');
  btnText.setAttribute('dominant-baseline', 'central');
  btnText.setAttribute('fill', '#e74c3c');
  btnText.setAttribute('font-size', '18');
  btnText.setAttribute('font-weight', 'bold');
  btnText.setAttribute('font-family', 'Microsoft YaHei, PingFang SC, sans-serif');
  btnText.setAttribute('cursor', 'pointer');
  btnText.setAttribute('id', 'btnCenterText');
  btnText.textContent = '抽奖';
  btnCircle.addEventListener('click', spin);
  btnText.addEventListener('click', spin);
  svg.appendChild(btnText);

  // Pointer group (rotates around center)
  const pointerG = document.createElementNS(NS, 'g');
  pointerG.setAttribute('id', 'pointer');
  pointerG.setAttribute('style', 'transform-origin: 200px 200px;');

  const pointer = document.createElementNS(NS, 'polygon');
  pointer.setAttribute('points', '200,22 185,58 215,58');
  pointer.setAttribute('fill', '#ff4757');
  pointer.setAttribute('stroke', '#c0392b');
  pointer.setAttribute('stroke-width', '2');
  pointerG.appendChild(pointer);

  const dot = document.createElementNS(NS, 'circle');
  dot.setAttribute('cx', CX);
  dot.setAttribute('cy', CY);
  dot.setAttribute('r', '8');
  dot.setAttribute('fill', '#ff4757');
  pointerG.appendChild(dot);

  svg.appendChild(pointerG);
}
```

- [ ] **Step 2: Add init code**

Replace `// --- Init ---` with:

```js
let segments = loadSegments();
renderWheel(segments);
```

- [ ] **Step 3: Open in browser**

The SVG wheel should render with 8 colored segments, labels, a white center button reading "抽奖", and a red pointer pointing upward.

---

### Task 3: Spin animation and weight-based result

**Files:**
- Modify: `E:\AIproject\ZP1\index.html`

- [ ] **Step 1: Add animation and result logic**

Replace `// --- Animation ---` with:

```js
let currentAngle = 0;
let spinning = false;

function pickSegment() {
  const totalWeight = segments.reduce((s, seg) => s + seg.weight, 0);
  let rand = Math.random() * totalWeight;
  for (let i = 0; i < segments.length; i++) {
    rand -= segments[i].weight;
    if (rand <= 0) return i;
  }
  return segments.length - 1;
}

function getTargetAngle(winIndex) {
  const count = segments.length;
  const arc = (2 * Math.PI) / count;
  // Pointer at top (12 o'clock = -PI/2). Sector i center is at i * arc.
  // When pointer points at -PI/2, segment 0 is at top.
  // The pointer angle that aligns with sector center i is: i * arc
  const sectorCenter = winIndex * arc;
  const fullRotations = (5 + Math.floor(Math.random() * 6)) * 2 * Math.PI;
  const jitter = (Math.random() - 0.5) * arc * 0.7; // stay within sector
  return fullRotations + sectorCenter + jitter;
}

function spin() {
  if (spinning) return;
  spinning = true;

  const resultEl = document.getElementById('result');
  resultEl.classList.remove('show');
  resultEl.textContent = '';

  const winIndex = pickSegment();
  const targetDelta = getTargetAngle(winIndex);
  const startAngle = currentAngle;
  const targetAngle = startAngle + targetDelta;
  const duration = 4000 + Math.random() * 1000;
  const startTime = performance.now();
  const pointer = document.getElementById('pointer');

  function animate(now) {
    const elapsed = now - startTime;
    const progress = Math.min(elapsed / duration, 1);
    const eased = 1 - Math.pow(1 - progress, 3);
    currentAngle = startAngle + targetDelta * eased;
    pointer.style.transform = `rotate(${(currentAngle * 180 / Math.PI).toFixed(2)}deg)`;

    if (progress < 1) {
      requestAnimationFrame(animate);
    } else {
      currentAngle = targetAngle;
      pointer.style.transform = `rotate(${(currentAngle * 180 / Math.PI).toFixed(2)}deg)`;
      spinning = false;
      resultEl.textContent = `🎉 恭喜获得：${segments[winIndex].label} ！`;
      resultEl.classList.add('show');
    }
  }

  requestAnimationFrame(animate);
}
```

- [ ] **Step 2: Open in browser and test spin**

Click "抽奖" button. Pointer should rotate 5-10 full rotations with smooth deceleration, stop at a random segment, and display result text below.

---

### Task 4: Settings panel — HTML structure and CSS

**Files:**
- Modify: `E:\AIproject\ZP1\index.html`

- [ ] **Step 1: Add panel CSS**

Add these styles inside the `<style>` block, before `</style>`:

```css
  .panel {
    width: 0;
    overflow: hidden;
    background: rgba(255,255,255,0.06);
    border-radius: 16px;
    border: 1px solid rgba(255,255,255,0.1);
    transition: width 0.3s ease, padding 0.3s ease;
    align-self: stretch;
    display: flex;
    flex-direction: column;
  }
  .panel.open {
    width: 280px;
    padding: 16px;
  }

  .panel h3 {
    color: #ffd700;
    font-size: 1rem;
    margin-bottom: 12px;
    white-space: nowrap;
    letter-spacing: 2px;
  }

  .panel-row {
    display: flex;
    align-items: center;
    gap: 8px;
    margin-bottom: 8px;
    white-space: nowrap;
  }
  .panel-row input[type="text"] {
    width: 80px;
    padding: 4px 8px;
    border: 1px solid rgba(255,255,255,0.2);
    border-radius: 6px;
    background: rgba(255,255,255,0.08);
    color: #fff;
    font-size: 13px;
    outline: none;
  }
  .panel-row input[type="text"]:focus {
    border-color: #ffd700;
  }
  .panel-row input[type="number"] {
    width: 50px;
    padding: 4px 6px;
    border: 1px solid rgba(255,255,255,0.2);
    border-radius: 6px;
    background: rgba(255,255,255,0.08);
    color: #fff;
    font-size: 13px;
    outline: none;
    text-align: center;
  }
  .panel-row input[type="number"]:focus {
    border-color: #ffd700;
  }
  .panel-row input[type="color"] {
    width: 28px;
    height: 28px;
    border: none;
    border-radius: 6px;
    cursor: pointer;
    background: transparent;
    padding: 0;
  }
  .panel-row .btn-del {
    width: 24px;
    height: 24px;
    border: none;
    border-radius: 6px;
    background: rgba(255,71,87,0.3);
    color: #ff4757;
    cursor: pointer;
    font-size: 14px;
    line-height: 1;
    display: flex;
    align-items: center;
    justify-content: center;
  }
  .panel-row .btn-del:hover { background: rgba(255,71,87,0.6); }

  .btn-add {
    width: 100%;
    padding: 8px;
    border: 1px dashed rgba(255,255,255,0.3);
    border-radius: 8px;
    background: transparent;
    color: rgba(255,255,255,0.6);
    cursor: pointer;
    font-size: 14px;
    transition: all 0.2s;
    white-space: nowrap;
  }
  .btn-add:hover {
    border-color: #ffd700;
    color: #ffd700;
  }
  .btn-add:disabled {
    display: none;
  }
```

- [ ] **Step 2: Add panel JS logic**

Replace `// --- Panel ---` with:

```js
const panel = document.getElementById('panel');
const btnGear = document.getElementById('btnGear');

function renderPanel() {
  panel.innerHTML = '<h3>⚙ 奖项设置</h3>';
  const count = segments.length;

  segments.forEach((seg, i) => {
    const row = document.createElement('div');
    row.className = 'panel-row';

    const colorInput = document.createElement('input');
    colorInput.type = 'color';
    colorInput.value = seg.color;
    colorInput.addEventListener('input', () => {
      segments[i].color = colorInput.value;
      saveSegments(segments);
      renderWheel(segments);
    });

    const textInput = document.createElement('input');
    textInput.type = 'text';
    textInput.value = seg.label;
    textInput.maxLength = 8;
    textInput.addEventListener('input', () => {
      segments[i].label = textInput.value || seg.label;
      saveSegments(segments);
      renderWheel(segments);
    });

    const weightInput = document.createElement('input');
    weightInput.type = 'number';
    weightInput.value = seg.weight;
    weightInput.min = 1;
    weightInput.max = 100;
    weightInput.addEventListener('input', () => {
      segments[i].weight = Math.max(1, Math.min(100, parseInt(weightInput.value) || 1));
      weightInput.value = segments[i].weight;
      saveSegments(segments);
    });

    const delBtn = document.createElement('button');
    delBtn.className = 'btn-del';
    delBtn.textContent = '✕';
    delBtn.title = '删除';
    if (count <= 2) delBtn.style.display = 'none';
    delBtn.addEventListener('click', () => {
      segments.splice(i, 1);
      saveSegments(segments);
      renderWheel(segments);
      renderPanel();
    });

    row.appendChild(colorInput);
    row.appendChild(textInput);
    row.appendChild(weightInput);
    row.appendChild(delBtn);
    panel.appendChild(row);
  });

  const addBtn = document.createElement('button');
  addBtn.className = 'btn-add';
  addBtn.textContent = '+ 添加奖项';
  if (segments.length >= 12) addBtn.disabled = true;
  addBtn.addEventListener('click', () => {
    const colors = ['#ff6b6b','#ffa502','#ffd700','#7bed9f','#70a1ff','#a29bfe','#ff6348','#1e90ff','#ff9ff3','#54a0ff','#5f27cd','#01a3a4'];
    segments.push({
      label: '新奖项',
      color: colors[segments.length % colors.length],
      weight: 1,
    });
    saveSegments(segments);
    renderWheel(segments);
    renderPanel();
  });
  panel.appendChild(addBtn);
}

btnGear.addEventListener('click', () => {
  panel.classList.toggle('open');
  if (panel.classList.contains('open')) renderPanel();
});
```

- [ ] **Step 3: Open in browser and test panel**

Click gear button → panel slides in from the right. Edit name, color, weight → wheel updates live. Add/delete awards → wheel redraws. Changes persist across page reload.

---

### Task 5: Polish and edge cases

**Files:**
- Modify: `E:\AIproject\ZP1\index.html`

- [ ] **Step 1: Disable editing during spin**

Add spinner check to panel inputs. Wrap each input event handler with:

```js
if (spinning) return;
```

at the start of each handler in `renderPanel()`. Also add an overlay or CSS `pointer-events: none` to the panel when spinning. Add this inside `spin()` after `spinning = true;`:

```js
panel.style.pointerEvents = 'none';
panel.style.opacity = '0.5';
```

And in the animation completion, after `spinning = false;`:

```js
panel.style.pointerEvents = '';
panel.style.opacity = '';
```

- [ ] **Step 2: Add glow animation on win**

Add this CSS:

```css
  .wheel-wrapper.win {
    animation: glow 0.6s ease;
  }
  @keyframes glow {
    0%, 100% { filter: drop-shadow(0 0 10px rgba(255,215,0,0.3)); }
    50%      { filter: drop-shadow(0 0 40px rgba(255,215,0,0.8)); }
  }
```

Add `.win` class to wheel-wrapper on result, remove on next spin. In `spin()`, add before spinning:

```js
document.querySelector('.wheel-wrapper').classList.remove('win');
```

And on completion:

```js
document.querySelector('.wheel-wrapper').classList.add('win');
```

- [ ] **Step 3: Full test in browser**

1. Spin the wheel → pointer rotates, panel dims
2. Result shows with glow and pop animation
3. Edit awards → save and redraw
4. Add/delete awards → works within [2, 12] range
5. Refresh page → settings persist from localStorage
6. Clear localStorage → defaults load correctly
