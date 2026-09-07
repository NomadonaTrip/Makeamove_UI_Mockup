# Scheduling Core Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build one scheduler component and use it at every scheduling moment in the journey, add the two screens the flow is missing, and put both parties' timezones on every proposed slot.

**Architecture:** A `window.MMSCHED` module splits in two. A **pure core** holds the timezone offset map, the slot fixtures, the 12-hour formatter and a per-purpose `sessionStorage` accessor — all assertable without a DOM. A thin **`mount()`** renders that core into one of two modes: `propose` (pick up to 5 of your own times, `S-C1`) or `choose` (three suggestions plus a "see all times" week-grid overlay, used at `S-C4`, `S-E0` and `S-E2A`). Two new screens carry the flow: `S-E0`, the post-pass call screen reached from both the sync and async passes, and `S-E2A`, the physical date's time.

**Tech Stack:** HTML + CSS + vanilla JavaScript, single file. Verification is browser-driven via the Playwright MCP tools.

**Spec:** `docs/superpowers/specs/2026-09-06-scheduling-core-design.md`. Read it alongside this plan — it carries the rejected alternatives behind each locked decision.

## Global Constraints

- **Single file.** All changes in `/mnt/e/TOOLMAKER/PYTHON/MakeaMoveUI_Mockup/index.html`. No new runtime files, no build step, no libraries, no external CSS/JS. Do not touch `index_mobile_mockup.html` or `index_mockup.html`.
- **Reuse existing components:** `.opt`, `.card`, `.note`, `.btn`, `.tag`, `.chip`, `.av`, `.row`, `.body`, `.foot`, `.btnrow`, `.muted`, `.eyebrow`, `.big-num`, and the `:root` custom properties. Invent no new colours.
- **Both timezone lines render on every slot, always** — even when the two zones are identical. This is decided; do not "optimise" the duplicate away.
- **One storage key per scheduling purpose.** The session time (`S-C4`) and the date time (`S-E2A`) are different facts. They must not overwrite each other — see "The storage-key trap" below.
- **Nothing may hardcode a time that a screen claims was chosen.** `S-C5` and `S-E4` must read what was actually selected.
- **Do not break:** `window.pick`, `window.tog`, `window.togCap` (global option helpers used across many screens), `window.DSHOW`, `window.FSHOW`, or `MMQ` / `MMASYNC` / `MMLADDER` / `MMDECK` / `MMDEV`.
- **New screens are `paper` and take the default desktop archetype.** Do NOT add `S-E0` or `S-E2A` to the `FEED` or `IMMERSIVE` sets in `classify()` — a centred form sheet is correct for both.

## Line numbers shift as you go

Every `index.html:NNNN` reference below was accurate when this plan was
written. **Task 3 inserts a candidate (~+8 lines) and Task 6 inserts a whole
section (~+60 lines)**, so every citation after those points drifts. Each step
quotes the exact markup it is replacing — **locate by that text, not by the
number.** The numbers are a hint about where to look, nothing more.

## The storage-key trap

The single most likely defect in this phase. Two screens schedule two different things:

| Screen | Schedules | Key |
|---|---|---|
| `S-C4` | the game-show session | `mm_slot_session` |
| `S-E0` | the video call | `mm_slot_call` |
| `S-E2A` | the physical date | `mm_slot_date` |

If they share one key, picking a date time silently rewrites the session time and `S-C5` starts announcing the wrong thing. `mount()` therefore takes a `key` and every read is `chosen(key)`. Task 1 asserts that two keys do not collide.

## Verification setup

`file://` is blocked by the Playwright MCP browser. A static server is likely **already running** on port 3300 from earlier phases — check before starting one, because `python3 -m http.server 3300` fails with `EADDRINUSE` if it is:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:3300/index.html   # 200 = already serving
cd /mnt/e/TOOLMAKER/PYTHON/MakeaMoveUI_Mockup && python3 -m http.server 3300  # only if the above failed
```

Assertions run at **1280×800** unless a step says otherwise, against `http://127.0.0.1:3300/index.html#<screen>`.

**Tooling gotchas carried from Phases 1–3 — each cost real time there:**

- Never call `location.reload()` inside an evaluated function; it destroys the execution context and the evaluate fails instead of returning.
- Re-navigating to an identical URL serves a stale cached copy. Append a fresh cache-buster (`?v=2`, `?v=3`…) **before** the `#hash` every time you need new code.
- Resize **before** navigating — desktop archetype classes are applied on load.
- Any loop that plays a board must be `async` and `await` the delays.
- **Measuring overflow on these screens:** `.body` and the stage columns are centred flex columns, so content spills in *both* directions and `scrollHeight` counts only part of it. Compute overflow from child rects against the container's box, and let entry animations settle first. A Phase 3 review under-reported a spill by ~115px this way.

**The lesson this phase is most exposed to:** Phase 1 shipped eight tasks whose diffs all reviewed clean while a screen was visibly broken; Phase 3's five clean per-task reviews still missed a Critical that only the whole-branch pass caught. This phase adds two screens and a full-screen overlay to fixed-height flex columns — the same condition. **Task 8 is mandatory.**

## File Structure

One file, five insertion points:

- **New `<script id="mmsched">`** immediately after the closing `</script>` of `<script id="mmladder">`, before `<div class="screenwrap" id="screenwrap">`.
- **CSS block** (main `<style>`): a "scheduler" section appended to the shared components.
- **`CANDIDATES` fixture** (inside the gallery script): one new bench entry.
- **`S-C1`, `S-C3`, `S-C4`, `S-C5`** in the C group; **`S-E3`, `S-E4`** in the E group.
- **Two new sections:** `S-E0` immediately before `S-E1`, and `S-E2A` immediately after `S-E2`.

---

### Task 1: MMSCHED's pure core

**Files:**
- Modify: `index.html` — insert a new `<script id="mmsched">` immediately after the closing `</script>` of `<script id="mmladder">` and before `<div class="screenwrap" id="screenwrap">`.

**Interfaces:**
- Consumes: nothing. The core is pure apart from `sessionStorage`.
- Produces on `window`: `MMSCHED` with `TZ`, `SLOTS`, `fmtLine(slot, fromTz, toTz)`, `suggestions(n)`, `save(key, slot)`, `chosen(key)`, `clear(key)`. Task 2 adds `mount` to this same export object.

- [ ] **Step 1: Write the browser assertion**

Run this first, before writing any code, so you see it fail. Navigate to `#S-C1` and evaluate:

```js
() => {
  if (!window.MMSCHED) return { ok:false, reason:'MMSCHED not defined' };
  const S = window.MMSCHED;

  const at = t => ({ id:'x', day:'Sat 21 Jun', time:t, mutual:true });

  const f = {
    same:      S.fmtLine(at('20:00'), 'GMT', 'GMT'),   // 8:00 pm GMT
    ahead:     S.fmtLine(at('20:00'), 'GMT', 'WAT'),   // 9:00 pm WAT
    rollFwd:   S.fmtLine(at('23:30'), 'GMT', 'WAT'),   // 12:30 am WAT (+1d)
    rollBack:  S.fmtLine(at('00:30'), 'WAT', 'GMT'),   // 11:30 pm GMT (-1d)
    noonEdge:  S.fmtLine(at('12:00'), 'GMT', 'GMT'),   // 12:00 pm GMT
    midnight:  S.fmtLine(at('00:00'), 'GMT', 'GMT')    // 12:00 am GMT
  };

  const sug = S.suggestions(3);

  // The storage-key trap: two purposes must not overwrite each other.
  S.clear('session'); S.clear('date');
  S.save('session', S.SLOTS[0]);
  S.save('date',    S.SLOTS[1]);
  const keys = { session: S.chosen('session'), date: S.chosen('date') };
  S.clear('session');
  const afterClear = { session: S.chosen('session'), date: S.chosen('date') };

  return {
    ok: f.same === '8:00 pm GMT' &&
        f.ahead === '9:00 pm WAT' &&
        /^12:30 am WAT/.test(f.rollFwd) && /\+1d/.test(f.rollFwd) &&
        /^11:30 pm GMT/.test(f.rollBack) && /1d/.test(f.rollBack) &&
        f.noonEdge === '12:00 pm GMT' &&
        f.midnight === '12:00 am GMT' &&
        sug.length === 3 && sug.every(s => s.mutual === true) &&
        S.SLOTS.length >= 7 &&
        S.SLOTS.every(s => s.id && s.day && /^\d{2}:\d{2}$/.test(s.time)) &&
        keys.session.id !== keys.date.id &&
        afterClear.session === null && afterClear.date && afterClear.date.id === keys.date.id,
    f, sugIds: sug.map(s => s.id), slotCount: S.SLOTS.length, keys, afterClear
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `http://127.0.0.1:3300/index.html#S-C1`, evaluate the above.
Expected: `{ ok:false, reason:'MMSCHED not defined' }`.

- [ ] **Step 3: Write the module**

Insert after the `mmladder` script block:

```html
<script id="mmsched">
/* ===== MMSCHED — one scheduler, reused everywhere
   Spec: docs/superpowers/specs/2026-09-06-scheduling-core-design.md (Cluster 4).
   Fixtures, not an availability engine: this is a frontend mockup and every
   "system-computed" time here is a fixture presented as if computed. */
(function(){
  const TZ = { GMT: 0, WAT: 1 };

  /* A week of candidate times. `mutual` marks the ones choose mode surfaces
     as suggestions — the top three are the "best mutual times" of spec 4.1. */
  const SLOTS = [
    { id:'sat-2000', day:'Sat 21 Jun', time:'20:00', mutual:true  },
    { id:'sun-1930', day:'Sun 22 Jun', time:'19:30', mutual:true  },
    { id:'tue-2100', day:'Tue 24 Jun', time:'21:00', mutual:true  },
    { id:'sun-1600', day:'Sun 22 Jun', time:'16:00', mutual:false },
    { id:'thu-2030', day:'Thu 26 Jun', time:'20:30', mutual:false },
    { id:'fri-1800', day:'Fri 27 Jun', time:'18:00', mutual:false },
    { id:'sat-1300', day:'Sat 28 Jun', time:'13:00', mutual:false },
    { id:'sat-2330', day:'Sat 28 Jun', time:'23:30', mutual:false }
  ];

  const pad = n => (n < 10 ? '0' : '') + n;

  /* 24h -> 12h display. 00:00 reads "12:00 am", 12:00 reads "12:00 pm". */
  function clock(h, m){
    const ap = h < 12 ? 'am' : 'pm';
    let hr = h % 12; if (hr === 0) hr = 12;
    return hr + ':' + pad(m) + ' ' + ap;
  }

  /* A slot's `time` is stated in `fromTz`. Render it in `toTz`, flagging the
     day roll — a 23:30 GMT slot is half past midnight the NEXT day in Lagos,
     and hiding that is exactly the missed-session class of bug 4.4 exists
     to prevent. */
  function fmtLine(slot, fromTz, toTz){
    const delta = (TZ[toTz] || 0) - (TZ[fromTz] || 0);
    const parts = slot.time.split(':');
    let h = +parts[0] + delta; const m = +parts[1];
    let roll = 0;
    if (h >= 24){ h -= 24; roll = 1; }
    if (h < 0)  { h += 24; roll = -1; }
    return clock(h, m) + ' ' + toTz
         + (roll > 0 ? ' (+1d)' : roll < 0 ? ' (−1d)' : '');
  }

  const suggestions = n => SLOTS.filter(s => s.mutual).slice(0, n || 3);

  /* One key per scheduling purpose. The session, the call and the date are
     different facts; a shared key makes picking one silently rewrite another. */
  const K = k => 'mm_slot_' + (k || 'session');
  function save(key, slot){ sessionStorage.setItem(K(key), JSON.stringify(slot)); }
  function chosen(key){
    try { return JSON.parse(sessionStorage.getItem(K(key))); }
    catch(e){ return null; }
  }
  function clear(key){ sessionStorage.removeItem(K(key)); }

  window.MMSCHED = { TZ, SLOTS, fmtLine, suggestions, save, chosen, clear };
})();
</script>
```

- [ ] **Step 4: Run the assertion again**

Re-navigate with a cache-buster (`?v=2#S-C1`) and evaluate.
Expected: `ok: true`, with `f.rollFwd` reading `12:30 am WAT (+1d)`.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(sched): add MMSCHED's pure core

Timezone offsets, slot fixtures, a 12-hour formatter that flags day
rolls, and a per-purpose storage accessor so the session, the call and
the date cannot overwrite one another."
```

---

### Task 2: MMSCHED rendering — propose, choose, and the week grid

**Files:**
- Modify: `index.html` — the `mmsched` script (add `mount`), and the main `<style>` block (append a scheduler section after the existing shared components).

**Interfaces:**
- Consumes: `MMSCHED.SLOTS`, `fmtLine`, `suggestions`, `save`, `chosen`, `clear` from Task 1.
- Produces: `MMSCHED.mount(el, opts) → handle`.
  - `opts`: `{ mode:'propose'|'choose', key:'session'|'call'|'date', self:{name,tz}, other:{name,tz}, max:5 }`
  - `handle`: `{ selected(), reset(), openGrid(), closeGrid() }`. `selected()` returns the chosen slot object in choose mode, or an array of chosen slots in propose mode. Tasks 4–7 call exactly these names.

- [ ] **Step 1: Write the browser assertion**

Navigate to `#S-C1` and evaluate. This mounts onto a scratch element so the live screen is undisturbed:

```js
() => {
  const S = window.MMSCHED;
  if (!S || !S.mount) return { ok:false, reason:'MMSCHED.mount not defined' };
  S.clear('t1'); S.clear('t2');

  const host = document.createElement('div');
  document.body.appendChild(host);
  const ch = S.mount(host, { mode:'choose', key:'t1',
    self:{name:'Tolu',tz:'GMT'}, other:{name:'Chidi',tz:'WAT'} });

  const rows = () => [...host.querySelectorAll('.sched-slot')];
  const suggestedCount = rows().length;
  // Every row carries BOTH zone lines, always.
  const firstLines = rows()[0].querySelectorAll('.tzline span').length;
  const firstText  = rows()[0].querySelector('.tzline').textContent;

  rows()[1].click();
  const picked = ch.selected();
  const stored = S.chosen('t1');
  const selMarked = rows().filter(r => r.classList.contains('sel')).length;

  // The grid overlay
  ch.openGrid();
  const gridOpen = !!host.querySelector('.schedgrid.on');
  const gridRows = host.querySelectorAll('.schedgrid .sched-slot').length;
  ch.closeGrid();
  const gridClosed = !host.querySelector('.schedgrid.on');

  // Propose mode caps at max and does not write storage on every tap
  const host2 = document.createElement('div');
  document.body.appendChild(host2);
  const pr = S.mount(host2, { mode:'propose', key:'t2', max:3,
    self:{name:'Tolu',tz:'GMT'}, other:{name:'David',tz:'GMT'} });
  const prRows = [...host2.querySelectorAll('.sched-slot')];
  prRows.slice(0, 5).forEach(r => r.click());       // try to pick 5 with max 3
  const capped = pr.selected().length;
  const capNotice = !!host2.querySelector('.sched-cap');

  const out = {
    ok: suggestedCount === 3 &&
        firstLines === 2 && /GMT/.test(firstText) && /WAT/.test(firstText) &&
        picked && stored && picked.id === stored.id && selMarked === 1 &&
        gridOpen === true && gridRows === S.SLOTS.length && gridClosed === true &&
        capped === 3 && capNotice === true,
    suggestedCount, firstLines, firstText, picked, stored, selMarked,
    gridOpen, gridRows, gridClosed, capped, capNotice
  };
  host.remove(); host2.remove(); S.clear('t1'); S.clear('t2');
  return out;
}
```

- [ ] **Step 2: Run it to confirm it fails**

Expected: `{ ok:false, reason:'MMSCHED.mount not defined' }`.

- [ ] **Step 3: Add the CSS**

Append to the main `<style>` block, after the existing shared components:

```css
  /* ===== scheduler — one component, two modes (spec 2026-09-06, Cluster 4)
     Mounted by S-C1 (propose), S-C4 / S-E0 / S-E2A (choose). Paper surfaces
     only: every consuming screen is `paper`. */
  .sched{display:flex; flex-direction:column; gap:8px}
  .sched .sched-slot{align-items:center}
  .sched .sgrow{display:flex; flex-direction:column; gap:2px; flex:1; min-width:0}
  .sched .sgrow b{font-size:13.5px; font-weight:700}
  .sched .tzline{display:flex; flex-direction:column; gap:1px}
  .sched .tzline span{font-size:11px; color:var(--t-dim); line-height:1.35}
  .sched .tzline span + span{color:var(--t-faint)}
  .sched .sched-more{align-self:flex-start; font-size:12.5px; font-weight:700;
    color:var(--spotlight); cursor:pointer; text-decoration:underline; background:none;
    border:0; padding:4px 0}
  .sched .sched-cap{font-size:11.5px; color:var(--t-faint)}
  .sched .sched-cap.hit{color:var(--warn); font-weight:700}
  /* the full week grid, as an overlay within its screen — same pattern as
     S-D2's .reveal, so it never pushes the screen's flex column */
  .schedgrid{position:absolute; inset:0; z-index:70; display:none;
    flex-direction:column; background:var(--paper); padding:18px 16px}
  .schedgrid.on{display:flex; animation:scin .28s var(--ease-show)}
  .schedgrid .sgh{display:flex; align-items:center; justify-content:space-between;
    gap:10px; margin-bottom:10px}
  .schedgrid .sgh h3{font-family:var(--display); font-size:17px; margin:0}
  .schedgrid .sgclose{border:0; background:var(--paper-2); border-radius:9px;
    width:32px; height:32px; font-size:16px; cursor:pointer; color:var(--t-dim)}
  .schedgrid .sgbody{flex:1; overflow-y:auto; display:flex; flex-direction:column; gap:8px}
  .schedgrid .sgday{font-size:11px; font-weight:700; letter-spacing:.08em;
    text-transform:uppercase; color:var(--t-faint); margin-top:6px}
```

- [ ] **Step 4: Add `mount` to the module**

Inside the `mmsched` IIFE, before the `window.MMSCHED = …` line:

```js
  const esc = s => String(s).replace(/[&<>"]/g, c =>
    ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;'}[c]));

  function slotHTML(s, self, other, sel){
    return '<div class="opt sched-slot' + (sel ? ' sel' : '') + '" data-id="' + s.id + '">'
         + '<span class="dot"></span>'
         + '<span class="sgrow"><b>' + esc(s.day) + '</b>'
         + '<span class="tzline">'
         + '<span>' + esc(fmtLine(s, self.tz, self.tz)) + '</span>'
         + '<span>' + esc(fmtLine(s, self.tz, other.tz)) + '</span>'
         + '</span></span>'
         + (s.mutual ? '<span class="tag pass">both free</span>' : '')
         + '</div>';
  }

  function mount(el, opts){
    opts = opts || {};
    const mode  = opts.mode === 'propose' ? 'propose' : 'choose';
    const key   = opts.key || 'session';
    const self  = opts.self  || { name:'You',  tz:'GMT' };
    const other = opts.other || { name:'Them', tz:'GMT' };
    const max   = opts.max || 5;
    let picks = [];                       // slot ids

    el.innerHTML =
        '<div class="sched">'
      + '<div class="sched-list"></div>'
      + (mode === 'propose'
          ? '<div class="sched-cap"></div>'
          : '<button type="button" class="sched-more">see all times ›</button>')
      + '</div>'
      + '<div class="schedgrid"><div class="sgh"><h3>All times</h3>'
      + '<button type="button" class="sgclose">✕</button></div>'
      + '<div class="sgbody"></div></div>';

    const listEl = el.querySelector('.sched-list');
    const gridEl = el.querySelector('.schedgrid');
    const gridBody = el.querySelector('.sgbody');
    const capEl  = el.querySelector('.sched-cap');

    function shown(){ return mode === 'propose' ? SLOTS : suggestions(3); }

    function paintCap(){
      if (!capEl) return;
      capEl.textContent = picks.length + ' of ' + max + ' times chosen';
      capEl.classList.toggle('hit', picks.length >= max);
    }

    function draw(){
      listEl.innerHTML = shown()
        .map(s => slotHTML(s, self, other, picks.indexOf(s.id) > -1)).join('');
      /* The grid shows every slot, grouped by day, and reflects the same
         selection — it is the same list, not a second one. */
      const byDay = [];
      SLOTS.forEach(s => {
        let g = byDay.filter(x => x.day === s.day)[0];
        if (!g){ g = { day:s.day, rows:[] }; byDay.push(g); }
        g.rows.push(s);
      });
      gridBody.innerHTML = byDay.map(g =>
        '<div class="sgday">' + esc(g.day) + '</div>'
        + g.rows.map(s => slotHTML(s, self, other, picks.indexOf(s.id) > -1)).join('')
      ).join('');
      paintCap();
    }

    function toggle(id){
      const at = picks.indexOf(id);
      if (mode === 'choose'){
        picks = [id];
        save(key, SLOTS.filter(s => s.id === id)[0]);
      } else if (at > -1){
        picks.splice(at, 1);
      } else if (picks.length < max){
        picks.push(id);
      }
      draw();
    }

    el.addEventListener('click', e => {
      const row = e.target.closest && e.target.closest('.sched-slot');
      if (row){ toggle(row.getAttribute('data-id')); return; }
      if (e.target.closest && e.target.closest('.sched-more')) api.openGrid();
      if (e.target.closest && e.target.closest('.sgclose'))    api.closeGrid();
    });

    const api = {
      selected(){
        const got = picks.map(id => SLOTS.filter(s => s.id === id)[0]);
        return mode === 'choose' ? (got[0] || null) : got;
      },
      reset(){ picks = []; clear(key); draw(); },
      openGrid(){ gridEl.classList.add('on'); },
      closeGrid(){ gridEl.classList.remove('on'); }
    };

    draw();
    return api;
  }
```

and extend the export to
`window.MMSCHED = { TZ, SLOTS, fmtLine, suggestions, save, chosen, clear, mount };`

- [ ] **Step 5: Run the assertion again**

Re-navigate with a fresh cache-buster and evaluate.
Expected: `ok: true`, `suggestedCount: 3`, `firstLines: 2`, `capped: 3`.

If `gridRows` is 0, the grid body is being drawn before `gridBody` is resolved — check that `draw()` runs after the `innerHTML` assignment.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(sched): render propose and choose modes plus the week grid

Both timezone lines on every row, always. The grid is an overlay inside
its screen, like S-D2's .reveal, so it cannot push a fixed flex column."
```

---

### Task 3: The Lagos candidate

**Files:**
- Modify: `index.html` — the `CANDIDATES` array, after the `bode` entry.

**Interfaces:**
- Consumes: the `benchBands(...)` and `B(...)` helpers already in that closure.
- Produces: `CANDIDATES` grows from 8 to 9. No signature changes.

**Why this task exists.** Every persona in the prototype resides in the UK or Dublin — same offset. Without a non-UK resident, 4.4's dual-timezone line renders "8:00 pm GMT · 8:00 pm GMT" everywhere and demonstrates nothing. Lagos and Abuja appear only as *state-of-origin* options in onboarding (`index.html:1354`–`1357`), never as residences.

- [ ] **Step 1: Write the browser assertion**

Navigate to `#S-B2` and evaluate:

```js
() => {
  const rows = [...document.querySelectorAll('#S-B2 [data-cand]')];
  const txt = document.getElementById('S-B2').textContent;
  const imgs = [...document.querySelectorAll('#S-B2 img')];
  const broken = imgs.filter(i => i.complete && i.naturalWidth === 0).length;
  return {
    ok: /Lagos/.test(txt) && broken === 0,
    hasLagos: /Lagos/.test(txt), cardCount: rows.length,
    imgCount: imgs.length, broken
  };
}
```

If `[data-cand]` matches nothing, fall back to counting `#S-B2 .row` — the assertion's load-bearing checks are `hasLagos` and `broken`.

- [ ] **Step 2: Run it to confirm it fails**

Expected: `hasLagos: false`.

- [ ] **Step 3: Add the candidate**

Insert into `CANDIDATES` immediately after the `bode` entry:

```js
    { id:'chidi', name:'Chidi', age:44, city:'Lagos',
      blurb:'Civil engineer · 1 kid · Lagos',
      /* Placeholder portrait: this reuses bode's Unsplash id rather than
         inventing one, because PHOTO() builds the URL directly and a wrong
         id renders a broken image. Swap for a hand-picked photo. */
      pid:'1617244147299-5ef406921c35',
      bands:benchBands(34,'n','You live in London · he lives in Lagos — different country, and an hour ahead',
                       56,'You are 40 · he is 44 — inside your stated 38–48 range',
                       B('y','You both hold a postgraduate qualification','profile')) },
```

- [ ] **Step 4: Run the assertion again**

Re-navigate with a fresh cache-buster to `#S-B2`.
Expected: `ok: true`, `hasLagos: true`, `broken: 0`, and `cardCount` one higher than before.

- [ ] **Step 5: Check the profile screen still renders**

Navigate to `#S-B3` and evaluate:

```js
() => {
  const s = document.getElementById('S-B3');
  const err = [...s.querySelectorAll('img')].filter(i => i.complete && i.naturalWidth === 0);
  return { ok: err.length === 0 && s.textContent.length > 200,
           brokenImgs: err.length, len: s.textContent.length };
}
```

Expected: `ok: true`.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(matching): add a Lagos-resident candidate

Every persona resided in the UK or Dublin, so a cross-timezone pair was
unreachable and 4.4's dual-zone line would have demoed nothing. Portrait
is a reused placeholder pending a hand-picked photo."
```

---

### Task 4: S-C1 proposes, S-C3 reads the same fixture

**Files:**
- Modify: `index.html` — `S-C1`'s slot block (currently five hardcoded `.opt` rows) and `S-C3`'s slot list.

**Interfaces:**
- Consumes: `MMSCHED.mount()`, `MMSCHED.SLOTS`, `MMSCHED.fmtLine()`.
- Produces: no new globals. `S-C1` mounts with `key:'session'`, `mode:'propose'`, `max:5`.

- [ ] **Step 1: Write the browser assertion**

Navigate to `#S-C1` and evaluate:

```js
() => {
  const host = document.querySelector('#S-C1 #c1sched .sched');
  if (!host) return { ok:false, reason:'no scheduler mounted in S-C1' };
  const rows = [...document.querySelectorAll('#S-C1 .sched-slot')];
  rows.slice(0, 6).forEach(r => r.click());          // try 6 against a cap of 5
  const sel = document.querySelectorAll('#S-C1 .sched-slot.sel').length;
  const cap = document.querySelector('#S-C1 .sched-cap');

  // S-C3's stated count must match what it actually lists.
  const c3 = document.getElementById('S-C3');
  const c3rows = c3.querySelectorAll('.sched-slot, .opt').length;
  const c3claim = (c3.textContent.match(/proposed\s+(\d+)\s+times/) || [])[1];
  const c3tz = c3.querySelectorAll('.tzline').length;

  return {
    ok: sel === 5 && !!cap && rows.length >= 7 &&
        c3claim && +c3claim === c3rows && c3tz === c3rows,
    sel, rowCount: rows.length, capText: cap && cap.textContent,
    c3claim, c3rows, c3tz
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Expected: `{ ok:false, reason:'no scheduler mounted in S-C1' }`.

- [ ] **Step 3: Replace S-C1's slot block**

Replace the label and the five hardcoded rows (`index.html:2403`–`2410`, the
`<label>` through the closing `</div>` of the slot column) with:

```html
          <label class="muted" style="font-size:12px;font-weight:700">Propose up to 5 times that work for you</label>
          <div id="c1sched" style="position:relative"></div>
```

`position:relative` is required — the grid overlay is `position:absolute; inset:0` and must anchor to this block, not to the screen.

Then add, immediately before `S-C1`'s closing `</section>`:

```html
        <script>
        (function(){
          /* Propose mode (spec 4.1): the invitee has not engaged yet, so there
             is nothing to overlap — you state your own times and S-C3 receives
             them. Choose mode takes over from S-C4 onward. */
          MMSCHED.mount(document.getElementById('c1sched'), {
            mode:'propose', key:'session', max:5,
            self:{ name:'Tolu', tz:'GMT' }, other:{ name:'David', tz:'GMT' }
          });
        })();
        </script>
```

- [ ] **Step 4: Re-point S-C3's list at the fixture**

Replace `S-C3`'s hardcoded heading and slot rows with a mount point:

```html
          <h2 class="scrn" style="font-size:22px" id="c3claim">She proposed 4 times</h2>
          <p class="muted" style="font-size:12.5px;margin-top:-6px">Pick one, or offer your own — you'll see when you both overlap.</p>
          <div id="c3sched" style="position:relative"></div>
```

and add before `S-C3`'s closing `</section>`:

```html
        <script>
        (function(){
          /* David's side. He picks from what Tolu proposed, so the count in the
             heading is derived — a hardcoded "4" would drift the moment the
             fixture changes. */
          const shown = MMSCHED.SLOTS.slice(0, 4);
          document.getElementById('c3claim').textContent =
            'She proposed ' + shown.length + ' times';
          document.getElementById('c3sched').innerHTML =
            '<div class="sched"><div class="sched-list">'
            + shown.map(s =>
                '<div class="opt sched-slot" data-id="' + s.id + '" onclick="pick(this)">'
              + '<span class="dot"></span><span class="sgrow"><b>' + s.day + '</b>'
              + '<span class="tzline"><span>' + MMSCHED.fmtLine(s,'GMT','GMT') + '</span>'
              + '<span>' + MMSCHED.fmtLine(s,'GMT','WAT') + '</span></span></span></div>'
              ).join('')
            + '</div></div>';
        })();
        </script>
```

- [ ] **Step 5: Run the assertion again**

Re-navigate with a fresh cache-buster.
Expected: `ok: true`, `sel: 5` (the sixth click refused), `c3claim` equal to `c3rows`.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(sched): S-C1 proposes via MMSCHED; S-C3 reads the fixture

S-C3's 'she proposed N times' is now derived, so it cannot drift from
the list beneath it."
```

---

### Task 5: S-C4 chooses, S-C5 shows what was chosen

**Files:**
- Modify: `index.html` — `S-C4`'s overlap cards, and `S-C5`'s confirmation note.

**Interfaces:**
- Consumes: `MMSCHED.mount()`, `MMSCHED.chosen('session')`, `MMSCHED.fmtLine()`.
- Produces: no new globals. `S-C4` mounts with `key:'session'`, `mode:'choose'`.

- [ ] **Step 1: Write the browser assertion**

Navigate to `#S-C4` and evaluate:

```js
async () => {
  const wait = ms => new Promise(r => setTimeout(r, ms));
  MMSCHED.clear('session');
  const host = document.querySelector('#S-C4 #c4sched .sched');
  if (!host) return { ok:false, reason:'no scheduler mounted in S-C4' };

  const rows = [...document.querySelectorAll('#S-C4 .sched-list .sched-slot')];
  const more = document.querySelector('#S-C4 .sched-more');
  rows[0].click();
  const stored = MMSCHED.chosen('session');

  more.click();
  const gridOpen = !!document.querySelector('#S-C4 .schedgrid.on');
  document.querySelector('#S-C4 .sgclose').click();
  const gridShut = !document.querySelector('#S-C4 .schedgrid.on');

  /* S-C5 must announce the slot actually chosen, not a hardcoded string.
     hashchange fires asynchronously, and S-C5 paints on that event — read
     the text without waiting and you get the pre-paint copy. */
  location.hash = 'S-C5';
  await wait(150);
  const c5 = document.getElementById('S-C5').textContent;
  const hasChosenDay = stored && c5.indexOf(stored.day) > -1;
  const hardcoded = /Saturday 21 Jun · 8:00 pm agreed/.test(c5);

  return {
    ok: rows.length === 3 && !!more && !!stored &&
        gridOpen && gridShut && hasChosenDay && !hardcoded,
    rowCount: rows.length, stored, gridOpen, gridShut, hasChosenDay, hardcoded
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Expected: `{ ok:false, reason:'no scheduler mounted in S-C4' }`.

- [ ] **Step 3: Replace S-C4's cards**

Replace the two `.card` blocks (`index.html:2457`–`2465`) with:

```html
          <div id="c4sched" style="position:relative"></div>
          <div class="note">A session is scheduled when you both land on the same time. The suggestions are the times you are both free; <b>see all times</b> opens the full week.</div>
```

Keep the legend row above it — the grid uses the same key.

Add before `S-C4`'s closing `</section>`:

```html
        <script>
        (function(){
          /* Choose mode (spec 4.1): three best mutual times, one tap, with the
             full week behind "see all times". */
          MMSCHED.mount(document.getElementById('c4sched'), {
            mode:'choose', key:'session',
            self:{ name:'Tolu', tz:'GMT' }, other:{ name:'David', tz:'GMT' }
          });
        })();
        </script>
```

- [ ] **Step 4: Make S-C5 read the chosen slot**

Replace `S-C5`'s hardcoded note (`index.html:2474`) with:

```html
          <div class="note pass" id="c5slot">✓ Agreed by both. This is the moment commitment bites.</div>
```

and add before `S-C5`'s closing `</section>`:

```html
        <script>
        (function(){
          function paint(){
            if (location.hash !== '#S-C5') return;
            const s = MMSCHED.chosen('session');
            const el = document.getElementById('c5slot');
            if (!el) return;
            el.innerHTML = s
              ? '✓ <b>' + s.day + ' · ' + MMSCHED.fmtLine(s,'GMT','GMT') + '</b> ('
                + MMSCHED.fmtLine(s,'GMT','WAT') + ') agreed by both. This is the moment commitment bites.'
              : '✓ Agreed by both. This is the moment commitment bites.';
          }
          window.addEventListener('hashchange', paint);
          paint();
        })();
        </script>
```

- [ ] **Step 5: Run the assertion again**

Re-navigate with a fresh cache-buster.
Expected: `ok: true`, `rowCount: 3`, `hardcoded: false`.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(sched): S-C4 chooses from three mutual times; S-C5 reads it

S-C5 stops announcing a hardcoded Saturday and reports the slot that was
actually selected, in both zones."
```

---

### Task 6: S-E0, the call screen, and the 60-second ring

**Files:**
- Create: a new `<section id="S-E0">` in `index.html`, immediately **before** `<section … id="S-E1">`.
- Modify: `index.html` — `S-D5`'s CTA, and `S-F4`'s pass-branch CTA.

**Interfaces:**
- Consumes: `MMSCHED.mount()` with `key:'call'`.
- Produces: `window.MMCALL` with `{ ring(), answer(), noAnswer() }`, used by the screen's own buttons and by the demo link.

- [ ] **Step 1: Write the browser assertion**

Navigate to `#S-E0` and evaluate:

```js
async () => {
  const wait = ms => new Promise(r => setTimeout(r, ms));
  const s = document.getElementById('S-E0');
  if (!s) return { ok:false, reason:'S-E0 does not exist' };

  const sched = s.querySelector('.sched');
  const callBtn = s.querySelector('[data-call-now]');

  /* Both passes must route here. S-F4's CTA is written by its own inline
     script when that screen is entered, so it must be visited before its
     innerHTML contains the link at all. */
  const d5 = document.getElementById('S-D5').innerHTML;
  location.hash = 'S-F4'; await wait(400);
  const f4src = document.getElementById('S-F4').innerHTML;
  location.hash = 'S-E0'; await wait(150);

  // Ring overlay: fires, then the no-answer path returns to the scheduler.
  MMCALL.ring();
  await wait(80);
  const ringing = !!s.querySelector('.ringwrap.on');
  const hasCountdown = /\d+/.test((s.querySelector('[data-ring-count]')||{}).textContent || '');
  MMCALL.noAnswer();
  await wait(80);
  const closed = !s.querySelector('.ringwrap.on');
  const stillOnE0 = location.hash === '#S-E0';

  return {
    ok: !!sched && !!callBtn && ringing && hasCountdown && closed && stillOnE0 &&
        /#S-E0/.test(d5) && /#S-E0/.test(f4src),
    hasSched: !!sched, hasCallBtn: !!callBtn, ringing, hasCountdown,
    closed, stillOnE0, d5LinksE0: /#S-E0/.test(d5), f4LinksE0: /#S-E0/.test(f4src)
  };
}
```

`S-F4`'s CTA is written by its inline script, so `innerHTML` is the reliable place to look for the link.

- [ ] **Step 2: Run it to confirm it fails**

Expected: `{ ok:false, reason:'S-E0 does not exist' }`.

- [ ] **Step 3: Add the S-E0 section**

Insert immediately before `<section class="screen dark" id="S-E1" …>`:

```html
      <section class="screen paper" id="S-E0" data-group="E · Post-show" data-title="Set up the video call">
        <style>
          #S-E0 .ringwrap{position:absolute; inset:0; z-index:80; display:none;
            flex-direction:column; align-items:center; justify-content:center; gap:16px;
            background:radial-gradient(70% 50% at 50% 40%, #1b2440, #0d1226); color:var(--ink); text-align:center; padding:30px}
          #S-E0 .ringwrap.on{display:flex; animation:scin .3s ease}
          #S-E0 .ringwrap .rnum{font-family:var(--display); font-weight:900; font-size:54px; color:var(--bulb)}
          #S-E0 .ringwrap .rsub{font-size:13px; color:var(--ink-dim); max-width:28ch}
          #S-E0 .ringwrap .pulse{width:96px; height:96px; border-radius:50%;
            background:rgba(255,45,111,.18); display:flex; align-items:center; justify-content:center;
            animation:ringpulse 1.1s ease-in-out infinite}
          @keyframes ringpulse{50%{transform:scale(1.12); background:rgba(255,45,111,.32)}}
        </style>
        <div class="pbar"><h1>Set up the call</h1></div>
        <div class="body" style="position:relative">
          <div class="note pass">You both cleared 70%. Next: a 30-minute video call.</div>
          <div class="card" style="display:flex;gap:12px;align-items:center">
            <div class="av g1 s52">D</div>
            <div><div style="font-weight:800">Call David now</div>
              <div class="muted" style="font-size:12px">Rings for 60 seconds. No answer, and you'll pick a time instead.</div></div>
          </div>
          <a class="btn move block" data-call-now onclick="MMCALL.ring()">Call now</a>
          <div class="muted" style="font-size:12px;font-weight:700">Or agree a time</div>
          <div id="e0sched" style="position:relative"></div>
          <div class="note">Nothing is charged for the call. <a class="link" onclick="MMCALL.ring()">demo · ring ›</a></div>
          <div class="ringwrap">
            <div class="pulse"><svg class="icon" style="width:38px;height:38px;color:var(--move)"><use href="#i-heart"/></svg></div>
            <div class="rnum" data-ring-count>60</div>
            <div class="rsub">Ringing David… we'll drop you back to scheduling if he doesn't pick up.</div>
            <div class="btnrow">
              <a class="btn ghost sm" onclick="MMCALL.noAnswer()">Cancel</a>
              <a class="btn move sm" onclick="MMCALL.answer()">Demo: he answers ›</a>
            </div>
          </div>
        </div>
        <script>
        (function(){
          /* The 60s ring is an OVERLAY, not a screen (spec 4.2). Unanswered,
             it closes and the caller is already standing in the scheduler —
             which is exactly the "no awkward dead end" the decision asks for. */
          MMSCHED.mount(document.getElementById('e0sched'), {
            mode:'choose', key:'call',
            self:{ name:'Tolu', tz:'GMT' }, other:{ name:'David', tz:'GMT' }
          });
          const wrap = document.querySelector('#S-E0 .ringwrap');
          const num  = document.querySelector('#S-E0 [data-ring-count]');
          let t = null, left = 60;
          function stop(){ if(t){ clearInterval(t); t = null; } wrap.classList.remove('on'); }
          window.MMCALL = {
            ring(){
              left = 60; num.textContent = left;
              wrap.classList.add('on');
              if (t) clearInterval(t);
              t = setInterval(() => {
                left--; num.textContent = left;
                if (left <= 0) MMCALL.noAnswer();
              }, 1000);
            },
            answer(){ stop(); location.hash = 'S-E1'; },
            noAnswer(){ stop(); }
          };
          /* Leaving the screen must kill the timer, or it keeps counting and
             yanks the user back here from wherever they navigated to. */
          window.addEventListener('hashchange', () => {
            if (location.hash !== '#S-E0') stop();
          });
        })();
        </script>
      </section>
```

- [ ] **Step 4: Route both passes to S-E0**

In `S-D5`, change the CTA from `href="#S-E1"` to `href="#S-E0"` and its label from
"Start the video call" to "Set up the video call".

In `S-F4`'s inline script, change the pass branch from
`'<a class="btn pass block" href="#S-E1">Schedule the video call</a>'`
to
`'<a class="btn pass block" href="#S-E0">Schedule the video call</a>'`
— the label was already truthful; only the destination was wrong.

- [ ] **Step 5: Run the assertion again**

Re-navigate with a fresh cache-buster.
Expected: `ok: true`, with `d5LinksE0` and `f4LinksE0` both true.

- [ ] **Step 6: Confirm the timer cannot outlive the screen**

Evaluate:

```js
async () => {
  const wait = ms => new Promise(r => setTimeout(r, ms));
  location.hash = 'S-E0'; await wait(100);
  MMCALL.ring(); await wait(100);
  location.hash = 'S-B2'; await wait(1400);
  return { ok: location.hash === '#S-B2',
           hash: location.hash,
           ringGone: !document.querySelector('#S-E0 .ringwrap.on') };
}
```

Expected: `ok: true`, `ringGone: true`. A failure here means the interval survived navigation and will drag the user back.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat(sched): add S-E0, the post-pass call screen

Reached from both S-D5 and S-F4 — both previously jumped straight into
S-E1 with nothing scheduling it, and S-F4's button promised scheduling
it did not do. The 60s ring is an overlay; unanswered, it closes onto
the scheduler the caller is already standing in."
```

---

### Task 7: S-E2A, the midpoint venue, and a date S-E4 can trust

**Files:**
- Create: a new `<section id="S-E2A">` in `index.html`, immediately after `S-E2`'s closing `</section>`.
- Modify: `index.html` — `S-E2`'s CTA, `S-E3`'s location note, `S-E4`'s date line.

**Interfaces:**
- Consumes: `MMSCHED.mount()` with `key:'date'`, `MMSCHED.chosen('date')`, `MMSCHED.fmtLine()`.
- Produces: no new globals.

- [ ] **Step 1: Write the browser assertion**

Navigate to `#S-E2A` and evaluate:

```js
async () => {
  const wait = ms => new Promise(r => setTimeout(r, ms));
  MMSCHED.clear('date'); MMSCHED.clear('session');
  const s = document.getElementById('S-E2A');
  if (!s) return { ok:false, reason:'S-E2A does not exist' };

  const rows = [...s.querySelectorAll('.sched-list .sched-slot')];
  rows[1].click();
  const date = MMSCHED.chosen('date');
  const session = MMSCHED.chosen('session');   // must NOT have been written

  location.hash = 'S-E4';
  await wait(150);                              // S-E4 paints on hashchange
  const e4 = document.getElementById('S-E4').textContent;
  const e3 = document.getElementById('S-E3').textContent;

  return {
    ok: rows.length === 3 && !!date && session === null &&
        e4.indexOf(date.day) > -1 &&
        !/Saturday 28 Jun · 7:30 pm/.test(e4) &&
        !/Canary Wharf/.test(e3.split('Boisdale')[0]) &&
        /midpoint/i.test(e3),
    rowCount: rows.length, date, session,
    e4HasDate: e4.indexOf(date && date.day) > -1,
    e3Midpoint: /midpoint/i.test(e3)
  };
}
```

The `S-E3` check deliberately looks only at text *before* the venue rows: "Boisdale of Canary Wharf" is a venue name and stays.

- [ ] **Step 2: Run it to confirm it fails**

Expected: `{ ok:false, reason:'S-E2A does not exist' }`.

- [ ] **Step 3: Add S-E2A**

Insert immediately after `S-E2`'s closing `</section>`:

```html
      <section class="screen paper" id="S-E2A" data-group="E · Post-show" data-title="Date scheduling · time">
        <div class="pbar"><a class="back" href="#S-E2">‹</a><h1>When suits you both?</h1></div>
        <div class="body" style="position:relative">
          <div class="note info">Pick a time first — the venue list is filtered to places open then.</div>
          <div id="e2asched" style="position:relative"></div>
        </div>
        <div class="foot"><a class="btn move block" href="#S-E3">Choose a venue ›</a></div>
        <script>
        (function(){
          /* The physical date's time. Its own key: overwriting the session
             slot here would make S-C5 announce the date instead. */
          MMSCHED.mount(document.getElementById('e2asched'), {
            mode:'choose', key:'date',
            self:{ name:'Tolu', tz:'GMT' }, other:{ name:'David', tz:'GMT' }
          });
        })();
        </script>
      </section>
```

- [ ] **Step 4: Point S-E2 at it**

In `S-E2`'s foot, change `href="#S-E3"` to `href="#S-E2A"` on the
"We're a match — plan a date" button.

- [ ] **Step 5: Make S-E3's location a midpoint**

Replace `index.html:2890`:

```html
          <div class="note info"><svg class="icon"><use href="#i-pin"/></svg> Google Places · near Canary Wharf · 2-week window to meet.</div>
```

with:

```html
          <div class="note info"><svg class="icon"><use href="#i-pin"/></svg> <b>Midpoint of you both — near Stratford.</b> Computed from your two areas; neither of you sees where the other lives or works. 2-week window to meet.</div>
```

- [ ] **Step 6: Make S-E4 read the chosen date**

Replace `S-E4`'s hardcoded line (`index.html:2904`):

```html
            <div class="muted">Saturday 28 Jun · 7:30 pm · with David</div>
```

with:

```html
            <div class="muted" id="e4when">with David</div>
```

and add before `S-E4`'s closing `</section>`:

```html
        <script>
        (function(){
          function paint(){
            if (location.hash !== '#S-E4') return;
            const s = MMSCHED.chosen('date');
            const el = document.getElementById('e4when');
            if (!el) return;
            el.innerHTML = s
              ? s.day + ' · ' + MMSCHED.fmtLine(s,'GMT','GMT')
                + ' <span style="color:var(--t-faint)">(' + MMSCHED.fmtLine(s,'GMT','WAT') + ')</span> · with David'
              : 'with David';
          }
          window.addEventListener('hashchange', paint);
          paint();
        })();
        </script>
```

- [ ] **Step 7: Run the assertion again**

Re-navigate with a fresh cache-buster.
Expected: `ok: true`, `session: null` — the key separation holds.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "feat(sched): add S-E2A, midpoint venue copy, and a real S-E4 date

S-E4 announced a time no screen ever chose. S-E2A chooses it, under its
own storage key so the session slot is untouched, and S-E3's hardcoded
Canary Wharf becomes a computed midpoint that discloses neither address."
```

---

### Task 8: Full-flow regression and visual sweep

**Files:**
- Modify: `index.html` — only if this task finds defects.

**Interfaces:**
- Consumes: everything from Tasks 1–7.
- Produces: no code. A pass/fail gate on the branch.

**Why this task exists.** Phase 1 shipped eight tasks whose diffs all reviewed clean while a screen was visibly broken. Phase 3's five clean per-task reviews still missed a Critical that only the whole-branch pass caught — an overflow nobody saw because every viewport tested was 844px or taller. This phase adds two screens and a full-screen overlay to fixed-height flex columns. **Do not substitute reading the diff for looking at the screen.**

- [ ] **Step 1: Assert the three storage keys stay independent**

Navigate to `#S-C4` and evaluate:

```js
async () => {
  const wait = ms => new Promise(r => setTimeout(r, ms));
  ['session','call','date'].forEach(k => MMSCHED.clear(k));

  location.hash = 'S-C4'; await wait(120);
  document.querySelectorAll('#S-C4 .sched-list .sched-slot')[0].click();
  location.hash = 'S-E0'; await wait(120);
  document.querySelectorAll('#S-E0 .sched-list .sched-slot')[1].click();
  location.hash = 'S-E2A'; await wait(120);
  document.querySelectorAll('#S-E2A .sched-list .sched-slot')[2].click();

  const k = { session:MMSCHED.chosen('session'), call:MMSCHED.chosen('call'), date:MMSCHED.chosen('date') };
  const ids = [k.session.id, k.call.id, k.date.id];
  return { ok: new Set(ids).size === 3, ids, k };
}
```

Expected: `ok: true`. A failure means two screens share a key and are overwriting each other.

- [ ] **Step 2: Walk the whole scheduling journey at 390×844**

Resize first, then navigate. Screenshot and **look at** each of:
`S-C1`, `S-C3`, `S-C4`, `S-C5`, `S-E0` (both resting and mid-ring), `S-E2A`, `S-E3`, `S-E4`.

Confirm by eye:

- every slot row shows **two** timezone lines and they are not clipped;
- the "see all times" overlay covers its screen without spilling, and its close button works;
- the propose cap message on `S-C1` is visible when 5 are chosen;
- `S-E0`'s Move/Stay-style CTA and the scheduler both fit without the foot being pushed off-screen;
- `S-C5` and `S-E4` name the slot you actually picked.

- [ ] **Step 3: Repeat at 1440×900**

Same screens. Confirm the centred form sheets are not stretched, and that the
grid overlay is bounded by its screen rather than the viewport.

- [ ] **Step 4: Check the tightest phone**

Resize to **390×667** and re-check `S-E0` and `S-E2A` — the two new screens,
which are the ones with no prior sizing history. For each, measure overflow
from child rects rather than `scrollHeight`:

```js
() => {
  const id = location.hash.slice(1);
  const b = document.querySelector('#' + id + ' .body');
  const box = b.getBoundingClientRect();
  let top = Infinity, bot = -Infinity;
  [...b.children].forEach(c => { const r = c.getBoundingClientRect();
    if (r.height) { top = Math.min(top, r.top); bot = Math.max(bot, r.bottom); } });
  const foot = document.querySelector('#' + id + ' .foot');
  return { id, spill: Math.max(0, Math.round((box.top - top) + (bot - box.bottom))),
           footBottom: foot ? Math.round(foot.getBoundingClientRect().bottom) : null,
           vh: window.innerHeight };
}
```

Expected: `spill: 0`, and `footBottom <= vh`.

- [ ] **Step 5: Regression-check the neighbours**

- `S-B2` and `S-B3` still render with 9 candidates and no broken images.
- `S-D5` → `S-E0` → `S-E1` and `S-F4` → `S-E0` both walk cleanly.
- `S-E1`, `S-D2` and `S-F3` are unchanged — screenshot `S-D2` at 1440×900 to
  confirm Phase 3's ladder grid still holds.
- The ☰ screen index lists every screen, now including `S-E0` and `S-E2A`.
- Zero console errors across the walk.

- [ ] **Step 6: Fix anything found, then commit**

```bash
git add index.html
git commit -m "fix(sched): resolve full-flow regression findings"
```

If nothing needed fixing, say so in the PR description rather than committing
an empty change.

- [ ] **Step 7: Stop here**

Do **not** push or open a PR. Branch integration is the user's decision.

---

## Self-review

**Spec coverage.** Component and its two modes → Tasks 1–2. Timezone rendering
and the day-roll flag → Tasks 1–2. The Lagos fixture prerequisite → Task 3.
`S-C1` propose and `S-C3`'s derived count → Task 4. `S-C4` choose and `S-C5`'s
real slot → Task 5. `S-E0`, the instant call and the ring overlay, plus routing
from both passes → Task 6. `S-E2A`, `S-E3`'s midpoint and `S-E4`'s real date →
Task 7. The verification table → Task 8.

**Type consistency.** `mount(el, opts)` returns `{ selected, reset, openGrid,
closeGrid }` in Task 2 and is called by exactly those names in Tasks 4–7.
`save`/`chosen`/`clear` all take `key` first, and the three keys are
`'session'`, `'call'`, `'date'` in every task that names one. `fmtLine(slot,
fromTz, toTz)` keeps its argument order everywhere. `selected()` returns a
single slot in choose mode and an array in propose mode — Task 4 is the only
caller that relies on the array form, and it uses `.length`.

**Known judgement call, flagged rather than hidden.** Task 3 reuses `bode`'s
Unsplash id for the new candidate. `PHOTO()` builds the URL directly with no
fallback, so an invented id renders a broken image; a duplicated portrait is
the lesser evil in a prototype and is commented as a placeholder.
