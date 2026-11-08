# Swarm Rebuild Effect — Prompt สำหรับใส่ในเว็บไหนก็ได้

> วิธีใช้: copy ตั้งแต่ `## PROMPT` ลงไป แล้วส่งให้ AI (Claude / Cursor / Copilot) พร้อมโปรเจกต์
> แก้ค่าใน `[วงเล็บเหลี่ยม]` ก่อนส่ง ถ้าไม่แก้ AI จะใช้ค่า default
> โค้ดอ้างอิง (vanilla JS) อยู่ท้ายไฟล์ เอาไปใช้ตรงๆ ได้เลย หรือให้ AI port เป็น React/Vue ก็ได้

---

## PROMPT

Add a **"Swarm Rebuild" intro effect** to this website: on first load the page is wiped blank, then a flock of colorful labeled cursors ("AI agents") fly in and redraw the page section by section, following the visitor as they scroll. The real page must stay fully intact underneath. This is a pure visual effect: it never changes layout, content or behavior.

### Project settings
- Framework: `[Next.js App Router / React / Vue / plain HTML]`
- Where to apply: `[whole site / home page only / specific route]`
- Accent color: `[#C8F25A]`
- Agent names (one per cursor, themed to the site): `[Mapper, Modeler, Router, Rules, Approver, Connector, Notifier, Auditor, Miner, Scheduler, Validator, Optimizer, Escalator, Simulator, Archivist, Reporter]`
- Max agents on screen: `[14]`
- Play: `[once per session (sessionStorage) / every visit]`
- Show control bar: `[yes]`

### 1. Mark the blocks
- Tag each visual "piece" the swarm should draw with `data-swarm="<unique-id>"`: nav logo, nav links, headline, sub-headline, CTA row, each card, each table row, each diagram node, footer columns. Aim for 30–70 blocks per page.
- **Never nest blocks.** A tagged element must not contain another tagged element (the parent's clip would hide the child). Tag leaves; leave section wrappers untagged.
- If an element needs its own `clip-path`, `opacity` or `outline` already, wrap it in a plain `<div data-swarm>` instead of tagging it directly.
- Keep content inside each block within ~12px of its box (the reveal clip has a 12px margin).

### 2. Block states (CSS only, no layout change)
Each block is in one of three states, applied as inline style or classes:

| State | Style |
|---|---|
| `hidden` | `opacity:0; clip-path: inset(-12px calc(100% + 12px) -12px -12px); outline:1.5px dashed transparent; outline-offset:5px; transition: clip-path 420ms cubic-bezier(.6,0,.4,1), opacity 420ms` |
| `building` | `opacity:1; clip-path: inset(-12px); outline:1.5px dashed <agent color>; outline-offset:5px; transition: clip-path <drawMs>ms linear, opacity 140ms` |
| `done` | `opacity:1; clip-path: inset(-12px); outline-color: transparent; transition: outline-color 600ms` |

- Reveal is a left-to-right clip wipe, synced with the cursor sweeping across the block. That reads as "typing / drawing".
- Wipe-out at start: first set every block to `done` (with the explicit `clip-path: inset(-12px)` so it can animate), wait one frame, then set all to `hidden`. That gives a sweeping wipe instead of a jump.
- After the effect ends, remove all inline styles so the page is 100% normal.

### 3. Cursor layer
- One absolutely positioned layer covering the whole document: `pointer-events:none; overflow:hidden; z-index:9999; aria-hidden="true"`. Cursor coordinates are **document coordinates** (`rect.left + scrollX`, `rect.top + scrollY`), so cursors scroll with the page.
- Each cursor: an SVG arrow (filled with the agent's color, dark 1.4px stroke, drop-shadow) plus a pill label with the agent name, positioned at `left:14px; top:19px` relative to the arrow tip.
- Move cursors only with `transform: translate3d(x,y,0)` + a CSS `transition` whose duration equals the job duration. Never animate per frame in JS.
- Easing: travel `cubic-bezier(.5,.05,.2,1)`, drawing `cubic-bezier(.4,.15,.6,.85)`.

### 4. Scheduler (the swarm brain)
Run a `setInterval` tick every ~130ms:
1. **Pending** = blocks not yet assigned whose `getBoundingClientRect().top < innerHeight * 1.04`, in DOM order (top first).
2. **Wanted agents** = `clamp(4, maxAgents, ceil(pending * 0.7) + scrollBoost)`. Spawn new cursors from just outside the viewport (top edge 50%, left 25%, right 25%). Mark a new cursor `fresh` and don't give it a job until the next tick, so it renders at its spawn point first and the transition is visible.
3. Give each idle cursor the next pending block:
   - **Travel** to the block's left edge: vertical middle, or 22% down for blocks taller than 110px. `travelMs = min(1200, 280 + distance * 0.5) / speed`.
   - Then **draw**: set the block to `building`, and move the cursor to the right edge (78% down for tall blocks). `drawMs = clamp(450, 1600, width * 0.9 + height * 0.8) / speed`.
   - After `drawMs`, set the block to `done` and the cursor to idle.
4. Idle cursors with nothing to do wander lazily near the mouse, or near the bottom of the viewport ("waiting for you to scroll"), every ~1.6s.
5. **Scroll boost:** on scroll, compute velocity in px/ms. Set `boost = min(10, max(boost, v * 5))` and decay it each tick (`boost = boost * 0.88 - 0.05`). Faster scrolling means more agents.
6. When every block is `done`, stop the interval and enter the **done** phase. Cursors flock in a ring around the mouse (radius 34–74px, staggered durations of `380 + i*45` ms) and keep following it on `mousemove`/`scroll`, throttled to about 70ms.

### 5. Control bar (fixed, bottom-center)
- While building: "`N working · M agents`", "`built / total`", a thin progress bar, speed buttons `1× 2× 3×` (`aria-pressed`), **Tour** (auto-scrolls 60% of the viewport whenever nothing is pending, and stops at the bottom), and **Skip** (everything done immediately).
- When done: **Go live** (fade out the cursors, shrink the bar to a small "Replay build" pill) and **Replay** (scroll to top, wipe, rebuild).
- Real `<button type="button">` elements, at least 44px tall, visible focus ring, no global keyboard shortcuts.

### 6. Robustness rules
- **State:** keep the swarm state in a mutable object, not in React state per tick. Batch DOM/state updates into one `requestAnimationFrame`. In React, put the whole effect in one `useEffect` + refs and render the layer through a portal, or run it with vanilla DOM.
- **Run token:** every async timer captures a `runId` and bails if it changed. Skip, Replay and unmount increment it. That way no stale timer can mark blocks after a replay.
- **Accessibility:** respect `prefers-reduced-motion: reduce` by not running at all. The page must be fully usable while building: the clip only affects painting, so links and focus still work.
- **SEO / no-JS:** blocks are visible by default in HTML/SSR. Only JS hides them. No flash of hidden content without JS.
- **Mobile:** allowed, but cap agents at 8 when `innerWidth < 768`.
- **Cleanup:** on unmount or route change, clear intervals and timeouts, remove listeners and the layer, and strip inline styles from the blocks.
- **Next.js:** a client component (`"use client"`) mounted once in the layout or page. Query blocks after mount, and re-run on route change only if "every visit" is set.

### Deliverables
1. The effect module (TS if the project is TS) with an `initSwarm(options)` / `<SwarmRebuild />` API returning `destroy()`.
2. `data-swarm` attributes added across the chosen pages, following rule 1.
3. Short notes: how to disable it, change agents or speed, and add blocks to new sections.

Use the reference implementation below as the source of truth for behavior. Adapt it to the framework, don't redesign it.

---

## Reference implementation (vanilla JS, ~190 lines)

```js
// swarm-rebuild.js — usage: import { initSwarm } from './swarm-rebuild.js'; const s = initSwarm({ accent: '#C8F25A' });
export function initSwarm(opts = {}) {
  const o = {
    selector: '[data-swarm]', maxAgents: 14, accent: '#C8F25A', controls: true, once: false,
    agents: [['Mapper','#FF8787'],['Modeler','#FFB347'],['Router','#FFE066'],['Rules','#A9E34B'],
      ['Approver','#63E6BE'],['Connector','#66D9E8'],['Notifier','#74C0FC'],['Auditor','#91A7FF'],
      ['Miner','#B197FC'],['Scheduler','#E599F7'],['Validator','#FAA2C1'],['Optimizer','#8CE99A'],
      ['Escalator','#FFA94D'],['Simulator','#3BC9DB'],['Archivist','#DA77F2'],['Reporter','#FFD43B']],
    ...opts,
  };
  const noop = { destroy() {}, replay() {}, skip() {} };
  if (matchMedia('(prefers-reduced-motion: reduce)').matches) return noop;
  if (o.once && sessionStorage.getItem('swarm-played')) return noop;
  if (innerWidth < 768) o.maxAgents = Math.min(o.maxAgents, 8);

  const els = [...document.querySelectorAll(o.selector)];
  if (!els.length) return noop;
  const S = { phase: 'build', speed: 1, tour: false, built: 0, total: els.length, run: 0 };
  const timers = new Set(); let loop = 0, boost = 0, lastY = scrollY, lastT = performance.now();
  let mouse = null, lastTour = 0, lastFlock = 0, startAt = 0;
  const assigned = new Set(); const cursors = [];

  const later = (fn, ms) => { const t = setTimeout(() => { timers.delete(t); fn(); }, ms); timers.add(t); };
  const stop = () => { S.run++; timers.forEach(clearTimeout); timers.clear(); clearInterval(loop); };

  // ---- block styles
  const BASE = 'outline-offset:5px;';
  const st = {
    hidden: () => BASE + 'outline:1.5px dashed transparent;opacity:0;clip-path:inset(-12px calc(100% + 12px) -12px -12px);transition:clip-path 420ms cubic-bezier(.6,0,.4,1),opacity 420ms ease;',
    building: (c, d) => BASE + `outline:1.5px dashed ${c};opacity:1;clip-path:inset(-12px);transition:clip-path ${d}ms linear,opacity 140ms ease;`,
    done: () => BASE + 'outline:1.5px dashed transparent;opacity:1;clip-path:inset(-12px);transition:clip-path 420ms cubic-bezier(.6,0,.4,1),opacity 300ms ease,outline-color 600ms ease;',
  };
  const orig = els.map((el) => el.getAttribute('style') || '');
  const setBlock = (i, css) => { els[i].style.cssText = orig[i] + ';' + css; };
  const restore = () => els.forEach((el, i) => el.setAttribute('style', orig[i]));

  // ---- cursor layer
  const layer = document.createElement('div');
  layer.setAttribute('aria-hidden', 'true');
  layer.style.cssText = 'position:absolute;left:0;top:0;width:100%;pointer-events:none;overflow:hidden;z-index:9999;transition:opacity .5s;';
  const sizeLayer = () => { layer.style.height = document.documentElement.scrollHeight + 'px'; };
  document.body.appendChild(layer); sizeLayer();

  function spawn(i) {
    const [name, color] = o.agents[i % o.agents.length];
    const el = document.createElement('div');
    el.style.cssText = 'position:absolute;left:0;top:0;will-change:transform;';
    el.innerHTML = `<svg width="20" height="22" viewBox="0 0 20 22" style="display:block;filter:drop-shadow(0 2px 4px rgba(0,0,0,.5))"><path d="M2 1.5L2 18L6.6 13.8L9.6 20.4L12.6 19.1L9.7 12.6L16 12.6Z" fill="${color}" stroke="#0A0C10" stroke-width="1.4" stroke-linejoin="round"/></svg><div style="position:absolute;left:14px;top:19px;padding:4px 8px;border-radius:999px;white-space:nowrap;font:600 11px/1 ui-monospace,monospace;color:#0A0C10;background:${color};box-shadow:0 4px 12px rgba(0,0,0,.35)">${name}${i >= o.agents.length ? '-' + (Math.floor(i / o.agents.length) + 1) : ''}</div>`;
    layer.appendChild(el);
    const r = Math.random();
    const x = r < 0.5 ? Math.random() * innerWidth : r < 0.75 ? -30 : innerWidth + 10;
    const y = r < 0.5 ? scrollY - 40 : scrollY + Math.random() * innerHeight;
    const c = { el, color, x, y, busy: false, fresh: true, idleAt: 0 };
    move(c, x, y, 0);
    return c;
  }
  function move(c, x, y, ms, draw) {
    c.x = x; c.y = y;
    c.el.style.transition = `transform ${Math.round(ms)}ms ${draw ? 'cubic-bezier(.4,.15,.6,.85)' : 'cubic-bezier(.5,.05,.2,1)'}`;
    c.el.style.transform = `translate3d(${x.toFixed(1)}px,${y.toFixed(1)}px,0)`;
  }

  function assign(c, i) {
    const run = S.run, sp = S.speed, r = els[i].getBoundingClientRect();
    const L = r.left + scrollX, T = r.top + scrollY, W = r.width, H = r.height, tall = H > 110, pad = Math.min(8, W * 0.08);
    const sx = L + pad, sy = T + (tall ? H * 0.22 : H / 2), ex = L + W - pad, ey = T + (tall ? H * 0.78 : H / 2);
    const travel = Math.min(1200, 280 + Math.hypot(sx - c.x, sy - c.y) * 0.5) / sp;
    const draw = Math.min(1600, Math.max(450, W * 0.9 + H * 0.8)) / sp;
    assigned.add(i); c.busy = true; move(c, sx, sy, travel);
    later(() => {
      if (run !== S.run) return;
      setBlock(i, st.building(c.color, Math.round(draw))); move(c, ex, ey, draw, true);
      later(() => {
        if (run !== S.run) return;
        setBlock(i, st.done()); c.busy = false; c.idleAt = Date.now() - 900; S.built++; ui();
        if (S.built >= S.total) finish();
      }, draw + 40);
    }, travel);
  }

  function tick() {
    if (S.phase !== 'build' || Date.now() < startAt) return;
    const pending = [];
    els.forEach((el, i) => { if (!assigned.has(i) && el.getBoundingClientRect().top < innerHeight * 1.04) pending.push(i); });
    boost = Math.max(0, boost * 0.88 - 0.05);
    cursors.forEach((c) => (c.fresh = false));
    const want = Math.min(o.maxAgents, Math.max(4, Math.ceil(pending.length * 0.7) + Math.round(boost)));
    while (cursors.length < want) cursors.push(spawn(cursors.length));
    const now = Date.now(); let p = 0;
    for (const c of cursors) {
      if (c.busy || c.fresh) continue;
      if (p < pending.length) { assign(c, pending[p++]); continue; }
      if (now - c.idleAt > 1600) {
        const inView = mouse && mouse.y > scrollY && mouse.y < scrollY + innerHeight;
        const b = inView ? mouse : { x: innerWidth / 2, y: scrollY + innerHeight * 0.8 };
        const a = Math.random() * Math.PI * 2, d = 50 + Math.random() * 140;
        move(c, Math.max(10, Math.min(innerWidth - 90, b.x + Math.cos(a) * d * 1.8)), b.y + Math.sin(a) * d * 0.5, 1400);
        c.idleAt = now;
      }
    }
    if (S.tour && !pending.length && now - lastTour > 1000 && cursors.filter((c) => c.busy).length <= Math.ceil(cursors.length / 3)) {
      lastTour = now;
      const se = document.scrollingElement;
      if (se.scrollTop + innerHeight >= se.scrollHeight - 4) { S.tour = false; ui(); }
      else scrollBy({ top: Math.round(innerHeight * 0.6), behavior: 'smooth' });
    }
    ui();
  }

  function flock() {
    const p = mouse && mouse.y > scrollY - 50 && mouse.y < scrollY + innerHeight + 50 ? mouse : { x: innerWidth / 2, y: scrollY + innerHeight / 2 };
    cursors.forEach((c, i) => {
      const a = (i / cursors.length) * Math.PI * 2 + 0.4, rad = 34 + (i % 3) * 20;
      move(c, p.x + Math.cos(a) * rad, p.y + Math.sin(a) * rad, 380 + i * 45);
    });
  }
  function finish() {
    clearInterval(loop); S.phase = 'done'; S.tour = false;
    cursors.forEach((c) => (c.busy = false));
    if (o.once) sessionStorage.setItem('swarm-played', '1');
    flock(); ui();
  }
  function start() {
    stop(); const run = S.run;
    assigned.clear(); S.built = 0; S.phase = 'build'; layer.style.opacity = '1'; sizeLayer();
    cursors.forEach((c) => { c.busy = false; c.idleAt = 0; });
    els.forEach((_, i) => setBlock(i, st.done()));
    requestAnimationFrame(() => requestAnimationFrame(() => {
      if (run !== S.run) return;
      els.forEach((_, i) => setBlock(i, st.hidden()));
      startAt = Date.now() + 520; loop = setInterval(tick, 130); ui();
    }));
  }
  const skip = () => { stop(); els.forEach((_, i) => setBlock(i, st.done())); S.built = S.total; finish(); };
  const goLive = () => { S.phase = 'live'; layer.style.opacity = '0'; later(restore, 700); ui(); };
  const replay = () => { stop(); scrollTo({ top: 0, behavior: 'smooth' }); later(start, 550); };

  // ---- listeners
  const onMove = (e) => {
    mouse = { x: e.clientX + scrollX, y: e.clientY + scrollY };
    if (S.phase === 'done' && Date.now() - lastFlock > 70) { lastFlock = Date.now(); flock(); }
  };
  const onScroll = () => {
    const t = performance.now(), dy = scrollY - lastY;
    boost = Math.min(10, Math.max(boost, (Math.abs(dy) / Math.max(16, t - lastT)) * 5));
    if (mouse) mouse.y += dy;
    lastY = scrollY; lastT = t;
    if (S.phase === 'done' && Date.now() - lastFlock > 90) { lastFlock = Date.now(); flock(); }
  };
  addEventListener('mousemove', onMove, { passive: true });
  addEventListener('scroll', onScroll, { passive: true });
  addEventListener('resize', sizeLayer);

  // ---- control bar
  let bar = null;
  const btn = 'min-height:44px;min-width:44px;padding:0 12px;border:0;border-radius:10px;font:600 12px/1 ui-monospace,monospace;cursor:pointer;';
  function ui() {
    if (!o.controls) return;
    if (!bar) {
      bar = document.createElement('div');
      bar.setAttribute('role', 'region'); bar.setAttribute('aria-label', 'Swarm rebuild controls');
      bar.style.cssText = 'position:fixed;left:50%;bottom:20px;transform:translateX(-50%);z-index:10000;display:flex;gap:4px;align-items:center;flex-wrap:wrap;justify-content:center;padding:6px;max-width:calc(100% - 24px);background:rgba(17,20,27,.94);border:1px solid #2A3040;border-radius:16px;box-shadow:0 16px 48px rgba(0,0,0,.55);font:600 12px/1 ui-monospace,monospace;color:#EEF0F4;';
      bar.addEventListener('click', (e) => {
        const a = e.target.closest('button')?.dataset.a; if (!a) return;
        if (a.startsWith('speed')) S.speed = +a.slice(5);
        if (a === 'tour') { S.tour = !S.tour; lastTour = 0; }
        if (a === 'skip') skip(); if (a === 'live') goLive(); if (a === 'replay') replay();
        ui();
      });
      document.body.appendChild(bar);
    }
    const pct = Math.round((S.built / S.total) * 100);
    const busy = cursors.filter((c) => c.busy).length;
    const on = (x) => (x ? `background:${o.accent};color:#0A0C10;` : 'background:transparent;color:#C9CFDA;');
    if (S.phase === 'live') { bar.innerHTML = `<button type="button" data-a="replay" style="${btn}${on(false)}">Replay build</button>`; return; }
    bar.innerHTML =
      `<div style="display:flex;flex-direction:column;gap:7px;min-width:210px;padding:6px 12px"><div style="display:flex;justify-content:space-between;gap:14px"><span>${S.phase === 'done' ? 'Built by ' + cursors.length + ' agents' : busy + ' working · ' + cursors.length + ' agents'}</span><span style="color:#A3ACBB">${S.built} / ${S.total}</span></div><div style="height:4px;border-radius:9px;background:#232836;overflow:hidden"><div style="height:100%;width:${pct}%;background:${o.accent};transition:width .3s"></div></div></div>` +
      (S.phase === 'build'
        ? [1, 2, 3].map((n) => `<button type="button" data-a="speed${n}" aria-pressed="${S.speed === n}" style="${btn}${on(S.speed === n)}">${n}×</button>`).join('') +
          `<button type="button" data-a="tour" aria-pressed="${S.tour}" style="${btn}${S.tour ? 'background:#2A3040;color:#fff;' : on(false)}">Tour</button><button type="button" data-a="skip" style="${btn}${on(false)}">Skip</button>`
        : `<button type="button" data-a="live" style="${btn}${on(true)}">Go live</button><button type="button" data-a="replay" style="${btn}${on(false)}">Replay</button>`);
  }

  start();
  return {
    replay, skip,
    destroy() {
      stop(); restore(); layer.remove(); bar?.remove();
      removeEventListener('mousemove', onMove); removeEventListener('scroll', onScroll); removeEventListener('resize', sizeLayer);
    },
  };
}
```

### React / Next.js wrapper

```tsx
'use client';
import { useEffect } from 'react';
import { initSwarm } from '@/lib/swarm-rebuild';

export function SwarmRebuild(props: { accent?: string; maxAgents?: number; once?: boolean }) {
  useEffect(() => {
    const s = initSwarm(props);
    return () => s.destroy();
  }, []); // eslint-disable-line react-hooks/exhaustive-deps
  return null;
}

// app/page.tsx → <SwarmRebuild accent="#C8F25A" once />  (วางท้าย page หลัง content ที่มี data-swarm)
```

### Markup example

```html
<nav>
  <a data-swarm="logo" href="/">Brand</a>
  <div data-swarm="nav-links">…</div>
</nav>
<section>                       <!-- section ไม่ต้องติด -->
  <h1 data-swarm="hero-title">…</h1>
  <p data-swarm="hero-sub">…</p>
  <div data-swarm="hero-cta">…</div>
  <div class="grid">
    <article data-swarm="card-1">…</article>   <!-- ติดที่การ์ด ไม่ใช่ข้างในการ์ด -->
    <article data-swarm="card-2">…</article>
  </div>
</section>
```
