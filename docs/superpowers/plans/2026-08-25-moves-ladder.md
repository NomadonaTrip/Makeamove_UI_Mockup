# Moves Ladder Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace `S-D2`'s flat token strip with a vertical points ladder whose rungs light as the other party's verdicts land, with the 70% gate drawn across the rail — and promote `S-F3`'s `.mini` strip to a compact variant of the same component.

**Architecture:** A `window.MMLADDER` module splits cleanly in two. A **pure `model()`** derives every number the rail shows — points per rung, the gate threshold, which rung the gate line sits above, and the won/to-go/left readout — so the arithmetic is assertable without a DOM. A thin **`mount()`** renders that model into either a vertical `rail` (the arena) or a horizontal `compact` strip (`S-F3`), and returns a handle the screens drive with `set()`, `focus()` and `reset()`. The ladder holds no game state: the screens own their turn loops and tell it what happened.

**Tech Stack:** HTML + CSS + vanilla JavaScript, single file. Verification is browser-driven via the Playwright MCP tools.

**Spec:** `docs/superpowers/specs/2026-08-25-moves-ladder-design.md`. Read it alongside this plan — it carries the rejected alternatives and the reasoning behind each locked decision.

## Global Constraints

- **Single file.** All changes in `/mnt/e/TOOLMAKER/PYTHON/MakeaMoveUI_Mockup/index.html`. No new runtime files, no build step, no libraries.
- **Reuse the `:root` custom properties** — `--move`, `--stay`, `--stay-soft`, `--pass`, `--bulb`, `--ink`, `--ink-dim`, `--ink-faint`, `--stage-700/800/850`, `--display`, `--body`, `--ease-show`, `--glow-bulb`. Invent no new colours.
- **One rung = one verdict on you.** Arena rung count is `ceil(questions/2)`; async rung count is `questions`. Neither may be hardcoded — question counts vary 10–15 under Model B.
- **Points are `Math.round(w * 10)`**, matching the convention already live at `index.html:3134`.
- **Dimension weights come from `MMQ.DIMS`.** Do not redeclare them.
- **The gate must agree with the screens.** Both `S-D2` (`index.html:2544`) and `MMASYNC.scores()` (`index.html:1048`) pass on `Math.round(pct) >= 70`. See "The gate threshold" below — a naive `0.70 × total` drifts by a point and would print "1 to go" while the score head already reads 70%.
- **Do not break:** `window.DSHOW`'s `{lock, react, reset}` shape or `window.FSHOW`'s `{react, reset}` shape — the router calls `reset()` on entering those screens. Nor `MMQ` / `MMASYNC` / `MMDECK` / `MMDEV` / `MMATTACH` / `MMCONF`.
- **Out of scope, by decision:** rebuilding `S-F3`'s flip board, weight-proportional rung heights, any change to `S-D2`'s two score heads, and `index_mobile_mockup.html` / `index_mockup.html`.

## The gate threshold

This is the one piece of arithmetic in the phase that is easy to get subtly wrong, so it is stated once here and every task inherits it.

The screens pass a player when `Math.round(won / total * 100) >= 70`. That is true from `won / total >= 0.695`, **not** from `won / total >= 0.70`. So:

```js
const gatePts = Math.ceil(0.695 * totalPts);
```

Worked: `totalPts = 105` gives `ceil(72.975) = 73`. A player on 73 scores `round(73/105*100) = 70` and passes — a `0.70 × 105 = 74` threshold would have told them they were still a point short. At `totalPts = 120` both formulas agree on 84, which is why the default set alone will not expose the bug.

**Note for the reader of the spec:** the spec's deck-attached worked example quotes `G = 73.5`. The gate *rung* is unaffected (still rung 4), but the readout figure is 73, per the formula above.

## A deliberate scope note

`S-D2` and `S-F3` are both `dark` screens, so the ladder CSS is written for the stage palette only. `CLAUDE.md` asks that shared components theme both surfaces; a `paper` variant is deliberately **not** written here because no paper screen mounts a ladder. Add one when a consumer exists, not before.

## Verification setup

`file://` is blocked by the Playwright MCP browser. Serve the project once, before Task 1:

```bash
cd /mnt/e/TOOLMAKER/PYTHON/MakeaMoveUI_Mockup && python3 -m http.server 3300
```

Assertions run at **1280×800** unless a step says otherwise, against `http://127.0.0.1:3300/index.html#<screen>`.

**Tooling gotchas carried from Phases 1 and 2 — all three cost real time there:**

- Never call `location.reload()` inside an evaluated function; it destroys the execution context and the evaluate fails instead of returning.
- Re-navigating to an identical URL serves a stale cached copy. Append a fresh cache-buster (`?v=2`, `?v=3`…) **before** the `#hash` every time you need new code.
- Any loop that plays a board must be `async` and `await` the delays — `S-D2`'s reveal holds 600ms and `S-F3`'s `freact()` advances behind a 950ms `setTimeout`, so a synchronous loop spins and misleadingly hits its guard.

**The lesson this phase is most exposed to:** in Phase 1, diff review passed clean on eight tasks while `S-F3` was visibly broken on screen; in Phase 2, ten clean per-task reviews still hid three Critical defects found only by a whole-branch pass. This plan changes how much vertical space two fixed-height flex columns consume, which is exactly the condition that produces invisible breakage. **Task 6 is mandatory.**

## File Structure

One file, four insertion points:

- **New `<script id="mmladder">`** immediately after the closing `</script>` of `<script id="mmasync">`, before `<div class="screenwrap">`. It reads `MMQ.DIMS` only through its callers, so ordering against `MMQ` is not strictly required — but keep it with its siblings.
- **CSS block** (main `<style>`): a new "ladder" section appended to the shared components, plus one rule inside the existing `@media (min-width:768px)` block for the `S-D2` desktop grid.
- **`S-D2`** (~2357–2552): `.base` markup swap, and wiring inside its inline script.
- **`S-F3`** (~2980–3179): `#fmini` becomes a mount point; `freact()` and `FSHOW.reset()` rewired; the stale comment at `index.html:3016` corrected.

---

### Task 1: MMLADDER's pure model

**Files:**
- Modify: `index.html` — insert a new `<script id="mmladder">` immediately after the closing `</script>` of `<script id="mmasync">` and before `<div class="screenwrap" id="screenwrap">`.

**Interfaces:**
- Consumes: nothing. The model is pure — callers pass rung descriptors in.
- Produces on `window`: `MMLADDER` with `PTS(w)`, `GATE_FRAC`, and `model(rungs, verdicts)`. `model` returns:

```
{ rungs:   [{ n, dim, w, pts, state }],   // state: 'pending' | 'move' | 'stay'
  totalPts, gatePts, gateIndex,           // gateIndex is 1-based
  won, left, toGo, cleared, reachable }
```

Task 2 renders this object. Tasks 3 and 5 never call `model` directly — they go through `mount`.

- [ ] **Step 1: Write the browser assertion**

Run this first, before writing any code, so you see it fail. Navigate to `#S-D2` and evaluate:

```js
() => {
  if (!window.MMLADDER) return { ok:false, reason:'MMLADDER not defined' };
  const L = window.MMLADDER;

  // The default Model A arena set: 3 Family + 2 Values on your turns.
  const arena = [{dim:'Family',w:3.0},{dim:'Family',w:3.0},{dim:'Family',w:3.0},
                 {dim:'Values',w:1.5},{dim:'Values',w:1.5}];
  const a = L.model(arena, []);

  // Deck-attached Model A: five custom Values questions displace bank ones,
  // so your rungs become 15/15/15/30/30 and the gate rung MOVES to 4.
  const decked = [{dim:'Values',w:1.5},{dim:'Values',w:1.5},{dim:'Values',w:1.5},
                  {dim:'Family',w:3.0},{dim:'Family',w:3.0}];
  const d = L.model(decked, []);

  // Mid-game: rungs 1 and 3 won, rung 2 stayed, 4 and 5 pending.
  const mid = L.model(arena, ['move','stay','move']);

  // Unreachable: everything decided against you except one 15-pt rung.
  const dead = L.model(arena, ['stay','stay','stay','stay']);

  // The gate must agree with the screens' own rule, at every plausible total.
  const drift = [];
  for (let n = 1; n <= 15; n++){
    for (const w of [0.6, 0.9, 1.0, 1.4, 1.5, 3.0]){
      const m = L.model(Array(n).fill({dim:'Values',w}), []);
      const passAtGate  = Math.round(m.gatePts / m.totalPts * 100) >= 70;
      const failBelow   = Math.round((m.gatePts - 1) / m.totalPts * 100) < 70;
      if (!passAtGate || !failBelow) drift.push({ n, w, gatePts:m.gatePts, totalPts:m.totalPts });
    }
  }

  return {
    ok: a.totalPts === 120 && a.gatePts === 84 && a.gateIndex === 3 &&
        d.totalPts === 105 && d.gatePts === 73 && d.gateIndex === 4 &&
        mid.won === 60 && mid.left === 30 && mid.toGo === 24 &&
        mid.cleared === false && mid.reachable === true &&
        dead.reachable === false && dead.left === 15 &&
        a.rungs[0].pts === 30 && a.rungs[0].n === 1 &&
        a.rungs[0].state === 'pending' && mid.rungs[1].state === 'stay' &&
        drift.length === 0,
    arena:a, decked:d, mid:{won:mid.won,left:mid.left,toGo:mid.toGo}, drift
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `http://127.0.0.1:3300/index.html#S-D2`, evaluate the above.
Expected: `{ ok:false, reason:'MMLADDER not defined' }`.

- [ ] **Step 3: Write the module**

Insert after the `mmasync` script block:

```html
<script id="mmladder">
/* ===== MMLADDER — the Millionaire ladder (spec 2026-08-25, from decision 3.1)
   One rung = ONE VERDICT ON YOU. In the arena that is your answering turns
   only (S-D2 alternates); on the async board every question carries one.
   model() is pure so the arithmetic can be asserted without a DOM. */
(function(){
  const PTS = w => Math.round(w * 10);

  /* The screens pass on Math.round(won/total*100) >= 70, which is true from
     0.695 upward — NOT 0.70. Deriving the threshold any other way makes the
     foot readout say "1 to go" while the score head already reads 70%. */
  const GATE_FRAC = 0.695;

  function model(rungs, verdicts){
    verdicts = verdicts || [];
    const r = rungs.map((x, i) => ({
      n: i + 1, dim: x.dim, w: x.w, pts: PTS(x.w),
      state: verdicts[i] || 'pending'
    }));
    const totalPts = r.reduce((s, x) => s + x.pts, 0);
    const gatePts  = Math.ceil(GATE_FRAC * totalPts);

    /* The gate line sits above the first rung at which the points available
       so far could carry you over. Statement: win everything below this line
       and you have cleared. */
    let cum = 0, gateIndex = r.length;
    for (let i = 0; i < r.length; i++){
      cum += r[i].pts;
      if (cum >= gatePts){ gateIndex = i + 1; break; }
    }

    const won  = r.filter(x => x.state === 'move').reduce((s, x) => s + x.pts, 0);
    const left = r.filter(x => x.state === 'pending').reduce((s, x) => s + x.pts, 0);
    return { rungs:r, totalPts, gatePts, gateIndex, won, left,
             toGo: Math.max(0, gatePts - won),
             cleared: won >= gatePts,
             reachable: (won + left) >= gatePts };
  }

  window.MMLADDER = { PTS, GATE_FRAC, model };
})();
</script>
```

- [ ] **Step 4: Run the assertion again**

Re-navigate with a cache-buster (`?v=2#S-D2`) and evaluate.
Expected: `ok: true`, `drift: []`.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(ladder): add MMLADDER's pure rung and gate model

One rung = one verdict on you. The gate threshold derives from 0.695,
not 0.70, so it agrees with the Math.round(pct) >= 70 rule both S-D2
and MMASYNC.scores() already use."
```

---

### Task 2: Rendering — the rail and compact layouts

**Files:**
- Modify: `index.html` — the `mmladder` script (add `mount`), and the main `<style>` block (append a ladder section after the existing shared components).

**Interfaces:**
- Consumes: `MMLADDER.model()` from Task 1.
- Produces: `MMLADDER.mount(el, opts) → handle`.
  - `opts`: `{ rungs, subject, layout }` where `layout` is `'rail'` or `'compact'`.
  - `handle`: `{ set(i, verdict), focus(i), reset(rungs), state() }`. `i` is **0-based**; `state()` returns the current model object (used by assertions in later tasks). Tasks 3 and 5 call exactly these names.

- [ ] **Step 1: Write the browser assertion**

Navigate to `#S-D2` and evaluate. This mounts onto a scratch element, so it does not disturb the screen:

```js
() => {
  const L = window.MMLADDER;
  if (!L || !L.mount) return { ok:false, reason:'MMLADDER.mount not defined' };

  const host = document.createElement('div');
  document.body.appendChild(host);
  const arena = [{dim:'Family',w:3.0},{dim:'Family',w:3.0},{dim:'Family',w:3.0},
                 {dim:'Values',w:1.5},{dim:'Values',w:1.5}];
  const h = L.mount(host, { rungs:arena, subject:"David's reactions to you", layout:'rail' });

  const rungEls = () => [...host.querySelectorAll('.lrung')];
  // Reversed DOM order: the first rung element in the DOM is the TOP rung.
  const domOrder = rungEls().map(e => +e.dataset.n);

  // The gate divider must sit immediately before rung 3 in DOM order,
  // i.e. visually above it.
  const kids = [...host.querySelector('.lrungs').children];
  const gateAt = kids.findIndex(e => e.classList.contains('lgate'));
  const afterGate = kids[gateAt + 1];

  h.set(0, 'move'); h.set(1, 'stay'); h.set(2, 'move');
  const s = h.state();
  const foot = host.querySelector('.lfoot').textContent;
  /* Capture NOW: the unreachable-path block below resets the same host, and a
     live re-query afterwards would read the reset state instead of this one. */
  const rung1Move = rungEls()[4].classList.contains('move');

  // Unreachable path: the gate line desaturates and NOTHING is announced.
  const h2State = (() => { h.reset(arena);
    h.set(0,'stay'); h.set(1,'stay'); h.set(2,'stay'); h.set(3,'stay');
    return { dim: host.querySelector('.lgate').classList.contains('dim'),
             text: host.querySelector('.lfoot').textContent,
             reachable: h.state().reachable }; })();

  const compactHost = document.createElement('div');
  document.body.appendChild(compactHost);
  const c = L.mount(compactHost, { rungs:arena, subject:'x', layout:'compact' });
  const compactTagged = compactHost.querySelector('.ladder').dataset.layout === 'compact';

  const out = {
    ok: domOrder.join(',') === '5,4,3,2,1' &&
        gateAt > -1 && afterGate && +afterGate.dataset.n === 3 &&
        s.won === 60 && s.toGo === 24 &&
        /60/.test(foot) && /24/.test(foot) && /30/.test(foot) &&
        rung1Move &&
        h2State.dim === true && h2State.reachable === false &&
        !/cannot|can't|impossible|out of reach/i.test(h2State.text) &&
        compactTagged && compactHost.querySelectorAll('.lrung').length === 5,
    domOrder, gateAt, afterGateN: afterGate && +afterGate.dataset.n, foot, h2State
  };
  host.remove(); compactHost.remove();
  return out;
}
```

- [ ] **Step 2: Run it to confirm it fails**

Expected: `{ ok:false, reason:'MMLADDER.mount not defined' }`.

- [ ] **Step 3: Add the CSS**

Append to the main `<style>` block, after the existing shared components:

```css
  /* ===== ladder — the Millionaire rail (spec 2026-08-25 / decision 3.1)
     Mounted by S-D2 (layout:rail) and S-F3 (layout:compact). Stage palette
     only: no paper screen mounts one. */
  .ladder{--lrung:30px; --lgap:5px; display:flex; flex-direction:column; gap:6px; min-height:0}
  .ladder .lhead{font-size:10px; letter-spacing:.1em; text-transform:uppercase;
    color:var(--ink-faint); font-weight:700}
  .ladder .lrungs{display:flex; flex-direction:column; gap:var(--lgap);
    overflow-y:auto; scrollbar-width:none; min-height:0}
  .ladder .lrungs::-webkit-scrollbar{display:none}
  .ladder .lrung{flex:0 0 auto; height:var(--lrung); display:flex; align-items:center;
    gap:8px; padding:0 10px; border-radius:8px; background:rgba(255,255,255,.04);
    border:1px solid rgba(255,255,255,.08); transition:.35s var(--ease-show)}
  .ladder .lrung .ln{font-family:var(--display); font-weight:900; font-size:11px;
    color:var(--ink-faint); min-width:12px}
  .ladder .lrung .ld{font-size:11.5px; color:var(--ink-dim); flex:1;
    overflow:hidden; text-overflow:ellipsis; white-space:nowrap}
  .ladder .lrung .lp{font-family:var(--display); font-weight:900; font-size:12.5px;
    color:var(--ink-faint); min-width:38px; text-align:right}
  .ladder .lrung.move{background:rgba(255,45,111,.14); border-color:rgba(255,45,111,.45)}
  .ladder .lrung.move .ld{color:var(--ink)}
  .ladder .lrung.move .lp{color:var(--bulb); text-shadow:0 0 12px rgba(255,182,39,.5)}
  .ladder .lrung.stay{background:rgba(255,255,255,.03); border-color:rgba(255,255,255,.06)}
  .ladder .lrung.stay .ld, .ladder .lrung.stay .lp{color:var(--stay-soft)}
  .ladder .lrung.on{animation:rungpop .4s var(--ease-show) both}
  @keyframes rungpop{from{transform:scale(.9) translateY(6px); opacity:0}}
  .ladder .lgate{flex:0 0 auto; display:flex; align-items:center; gap:8px;
    font-family:var(--display); font-weight:800; font-size:9.5px; letter-spacing:.14em;
    text-transform:uppercase; color:var(--pass); transition:.4s}
  .ladder .lgate::before, .ladder .lgate::after{content:""; flex:1; height:2px;
    background:repeating-linear-gradient(90deg,var(--pass) 0 5px,transparent 5px 9px)}
  .ladder .lgate.dim{color:var(--ink-faint); filter:grayscale(1); opacity:.55}
  .ladder .lgate.dim::before, .ladder .lgate.dim::after{
    background:repeating-linear-gradient(90deg,var(--ink-faint) 0 5px,transparent 5px 9px)}
  .ladder .lfoot{font-size:10.5px; color:var(--ink-dim); font-weight:600}
  .ladder .lfoot b{color:var(--ink)}

  /* compact — S-F3 has no room for a vertical window and carries up to 15
     rungs, so the same model lays out along the axis its strip already used.
     Header and foot share one line to buy back the vertical space. */
  .ladder[data-layout="compact"]{--lrung:26px; gap:4px}
  .ladder[data-layout="compact"] .lrungs{flex-direction:row; flex-wrap:wrap;
    gap:3px; justify-content:center; overflow:visible}
  .ladder[data-layout="compact"] .lrung{width:var(--lrung); height:var(--lrung);
    flex-direction:column; gap:0; padding:0; justify-content:center}
  .ladder[data-layout="compact"] .lrung .ln,
  .ladder[data-layout="compact"] .lrung .ld{display:none}
  .ladder[data-layout="compact"] .lrung .lp{min-width:0; font-size:10px; text-align:center}
  .ladder[data-layout="compact"] .lgate{flex:0 0 auto; width:0; height:var(--lrung);
    padding:0; border-left:2px dashed var(--pass); font-size:0; gap:0}
  .ladder[data-layout="compact"] .lgate::before,
  .ladder[data-layout="compact"] .lgate::after{display:none}
  .ladder[data-layout="compact"] .lgate.dim{border-left-color:var(--ink-faint)}
  .ladder[data-layout="compact"] .lhead{display:none}
  .ladder[data-layout="compact"] .lfoot{text-align:center; font-size:10px}
```

- [ ] **Step 4: Add `mount` to the module**

Inside the `mmladder` IIFE, before the `window.MMLADDER = …` line:

```js
  const esc = s => String(s).replace(/[&<>"]/g, c =>
    ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;'}[c]));

  function rungHTML(r){
    const body = r.state === 'move' ? '<span class="lp">&hearts; ' + r.pts + '</span>'
               : r.state === 'stay' ? '<span class="lp">&#10005; &mdash;</span>'
               :                      '<span class="lp">' + r.pts + '</span>';
    return '<div class="lrung ' + r.state + '" data-n="' + r.n + '">'
         + '<span class="ln">' + r.n + '</span>'
         + '<span class="ld">' + esc(r.dim) + '</span>' + body + '</div>';
  }

  function mount(el, opts){
    let rungs = opts.rungs.slice(), verdicts = [];

    el.innerHTML = '<div class="ladder" data-layout="' + (opts.layout || 'rail') + '">'
      + '<div class="lhead"></div><div class="lrungs"></div><div class="lfoot"></div></div>';
    const root  = el.firstChild;
    const rowsEl = root.querySelector('.lrungs');
    const footEl = root.querySelector('.lfoot');
    root.querySelector('.lhead').textContent = opts.subject || '';

    function draw(){
      const m = window.MMLADDER.model(rungs, verdicts);
      /* Reversed DOM order, not flex column-reverse: the rail climbs upward
         and a screen reader must announce it in the order it is seen. */
      const parts = [];
      for (let i = m.rungs.length - 1; i >= 0; i--){
        /* Iterating top-down, the divider is pushed just BEFORE its rung, so
           it renders above it: [5, 4, GATE, 3, 2, 1]. */
        if (m.rungs[i].n === m.gateIndex)
          parts.push('<div class="lgate' + (m.reachable ? '' : ' dim') + '">'
            + '<span>70% gate &middot; ' + m.gatePts + ' pts</span></div>');
        parts.push(rungHTML(m.rungs[i]));
      }
      rowsEl.innerHTML = parts.join('');
      footEl.innerHTML = '<b>' + m.won + '</b> won &middot; <b>' + m.toGo
        + '</b> to go &middot; ' + m.left + ' left';
      return m;
    }

    draw();

    return {
      set(i, verdict){
        verdicts[i] = verdict;
        draw();
        const cell = rowsEl.querySelector('.lrung[data-n="' + (i + 1) + '"]');
        if (cell) cell.classList.add('on');
        this.focus(i);
      },
      focus(i){
        const cell = rowsEl.querySelector('.lrung[data-n="' + (i + 1) + '"]');
        if (cell && cell.scrollIntoView) cell.scrollIntoView({ block:'nearest' });
      },
      reset(next){ rungs = (next || rungs).slice(); verdicts = []; draw(); },
      state(){ return window.MMLADDER.model(rungs, verdicts); }
    };
  }
```

and extend the export to `window.MMLADDER = { PTS, GATE_FRAC, model, mount };`.

- [ ] **Step 5: Run the assertion again**

Re-navigate with a fresh cache-buster and evaluate.
Expected: `ok: true`, `domOrder: "5,4,3,2,1"`, `afterGateN: 3`.

If `afterGateN` is wrong, the divider is being spliced on the wrong side — in reversed DOM order the divider must be emitted **before** the gate rung's element, so it renders above it.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(ladder): render the rail and compact layouts

Reversed DOM order so visual and reading order agree. The gate divider
desaturates when clearing becomes unreachable and says nothing — the
foot readout already states the position, and S-D6 announces failure."
```

---

### Task 3: Wire the arena's phone rail

**Files:**
- Modify: `index.html:2451` — the `.base` markup.
- Modify: `index.html:2384–2388` — delete the `.tokens` / `.tok` rules and size the rail's window.
- Modify: `index.html:2412–2413` — delete the `tokpop` keyframe and its rule.
- Modify: `index.html:2455–2550` — the inline `DSHOW` script.

**Interfaces:**
- Consumes: `MMLADDER.mount()` from Task 2; `MMQ.DIMS` (already aliased as `W`).
- Produces: no new globals. `window.DSHOW` keeps its `{lock, react, reset}` shape.

- [ ] **Step 1: Write the browser assertion**

Navigate to `#S-D2` and evaluate. Note the `async` wrapper — the reveal holds 600ms:

```js
async () => {
  const wait = ms => new Promise(r => setTimeout(r, ms));
  DSHOW.reset();
  await wait(60);

  const rail = document.querySelector('#S-D2 .base .ladder');
  if (!rail) return { ok:false, reason:'no ladder mounted in S-D2 .base' };
  if (document.getElementById('tokens')) return { ok:false, reason:'#tokens still present' };

  const rungCount = rail.querySelectorAll('.lrung').length;
  const qCount = MMQ.buildSet().length;

  // Play through, locking on your turns and reacting on David's.
  let guard = 0;
  while (guard++ < 40){
    const act = document.getElementById('act');
    const lock = act.querySelector('button[onclick*="lock"]');
    if (lock){ lock.click(); await wait(700);
               const next = document.getElementById('rnext'); if (next) next.click();
               await wait(60); }
    else {
      const move = act.querySelector('button[onclick*="react"]');
      if (!move) break;
      move.click(); await wait(850);
    }
    if (location.hash === '#S-D5' || location.hash === '#S-D6') break;
  }

  const py = +sessionStorage.getItem('mm_py');
  return {
    ok: rungCount === Math.ceil(qCount / 2) &&
        rail.querySelector('.lgate') !== null &&
        (location.hash === '#S-D5' || location.hash === '#S-D6') &&
        Number.isFinite(py),
    rungCount, expected: Math.ceil(qCount / 2), qCount, hash: location.hash, py
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Expected: `{ ok:false, reason:'no ladder mounted in S-D2 .base' }`.

- [ ] **Step 3: Swap the markup**

Replace `index.html:2451`:

```html
        <div class="base"><div class="blbl">David's reactions to you ↓</div><div class="tokens" id="tokens"></div></div>
```

with:

```html
        <div class="base"><div id="dladder"></div></div>
```

- [ ] **Step 4: Replace the token CSS with the rail's window**

Delete these three rules from `S-D2`'s scoped `<style>`:

```css
          #S-D2 .base .blbl{…}
          #S-D2 .tokens{…}
          #S-D2 .tok{…} #S-D2 .tok.m{…} #S-D2 .tok.s{…}
```

and the two motion rules `#S-D2 .tok{animation:tokpop …}` plus `@keyframes tokpop{…}`.

Add in their place:

```css
          /* ~5-rung window, anchored on the current rung (decision 3.1).
             The +26px is the gate divider, which lives in the same flow. */
          #S-D2 .base .lrungs{max-height:calc(5 * var(--lrung) + 4 * var(--lgap) + 26px)}
```

- [ ] **Step 5: Wire the script**

Inside `S-D2`'s inline IIFE, add after the `dbuild()` function:

```js
          /* One rung per verdict on you — only even turns produce one, so the
             turn index has to be mapped onto the rung index. */
          let RUNGOF = {}, LADDER = null;
          function lbuild(){
            const rungs = []; RUNGOF = {};
            TURNS.forEach((t, n) => {
              if (t.mode !== 'you') return;
              RUNGOF[n] = rungs.length;
              rungs.push({ dim: t.q.dim, w: W[t.q.dim] });
            });
            if (LADDER) LADDER.reset(rungs);
            else LADDER = MMLADDER.mount(document.getElementById('dladder'),
              { rungs, subject: "David's reactions to you", layout: 'rail' });
          }
```

Then:

1. Change the bare `dbuild();` call (`index.html:2478`) to `dbuild(); lbuild();`.
2. Delete `addTok()` entirely (`index.html:2520`).
3. In `lock()`, replace `addTok(v);` with `LADDER.set(RUNGOF[i], v);`.
4. At the end of `render()`'s `t.mode==='you'` branch, before `paint()`, add:
   `if (LADDER) LADDER.focus(RUNGOF[i]);`
5. Change the `window.DSHOW` export (`index.html:2548`) to:

```js
          window.DSHOW={ lock, react,
            /* exposed so the regression pass in Task 6 can compare the rail's
               arithmetic against the score head's without replaying the board */
            ladder:()=>LADDER,
            reset(){ dbuild(); lbuild(); i=0; yA=yE=tA=tE=sd=0;
                     $('reveal').className='reveal'; render(); } };
```

Note `$('tokens').innerHTML=''` is gone — `lbuild()` now does that work — and
`lock` / `react` / `reset` keep their names, so the router's `DSHOW.reset()`
call is unaffected.

- [ ] **Step 6: Run the assertion again**

Re-navigate with a fresh cache-buster and evaluate.
Expected: `ok: true`, `rungCount: 5`, `expected: 5`, `qCount: 10`.

- [ ] **Step 7: Check Model B**

Evaluate `MMQ.setModel('B')`, then re-navigate and re-run the same assertion.
Expected: `ok: true` with `qCount: 15`, `rungCount: 8`, `expected: 8`.
Then restore with `MMQ.setModel('A')`.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "feat(arena): replace S-D2's token strip with the ladder rail

Rung count is ceil(questions/2) because David only judges you on your
answering turns — the same set of events the 70% gate scores."
```

---

### Task 4: The arena's desktop right rail

**Files:**
- Modify: `index.html` — inside the existing `@media (min-width:768px)` block, after the `.lay-immersive` rules at `index.html:539–541`.

**Interfaces:**
- Consumes: the `.base`/`#dladder` structure from Task 3.
- Produces: no JS. Pure CSS reflow, no markup change — the contract in `CLAUDE.md`.

- [ ] **Step 1: Write the browser assertion**

Resize to 1280×800 first, then navigate to `#S-D2` and evaluate:

```js
() => {
  const s = document.getElementById('S-D2');
  const base = s.querySelector('.base');
  const stage = s.querySelector('.stage');
  const rows = s.querySelector('.base .lrungs');
  const cs = getComputedStyle(s);
  const b = base.getBoundingClientRect(), st = stage.getBoundingClientRect();

  // S-E1 is the other IMMERSIVE screen and must NOT have been reflowed.
  const e1 = getComputedStyle(document.getElementById('S-E1')).display;

  return {
    ok: cs.display === 'grid' &&
        b.left > st.right - 2 &&                    // the rail is to the RIGHT
        getComputedStyle(rows).maxHeight === 'none' &&  // full ladder, no window
        b.bottom <= window.innerHeight + 1 &&       // nothing clipped off-screen
        rows.scrollHeight <= rows.clientHeight + 1 &&
        e1 !== 'grid',
    display: cs.display, baseLeft: b.left, stageRight: st.right,
    maxHeight: getComputedStyle(rows).maxHeight,
    overflowing: rows.scrollHeight - rows.clientHeight, e1
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Expected: `display: 'flex'` — the rail is still stacked below the stage.

- [ ] **Step 3: Add the desktop grid**

Inside the `@media (min-width:768px)` block, immediately after the existing
`.lay-immersive` rules:

```css
    /* S-D2 only — the full ladder as a right rail (decision 3.1). S-E1 is the
       other IMMERSIVE screen and must keep the stacked layout, so this is
       scoped by id, not by archetype.
       `.on` is REQUIRED: without it, #S-D2.lay-immersive would outrank
       `.screen{display:none}` and paint a hidden screen over the live one. */
    #S-D2.lay-immersive.on{
      display:grid; column-gap:28px; justify-content:center; padding:0 24px;
      grid-template-columns:minmax(0,620px) 240px;
      grid-template-rows:auto auto 1fr auto;
      grid-template-areas:"heads ladder" "flag ladder" "stage ladder" "act ladder"}
    #S-D2.lay-immersive.on .heads{grid-area:heads}
    #S-D2.lay-immersive.on .turnflag{grid-area:flag}
    #S-D2.lay-immersive.on .stage{grid-area:stage}
    #S-D2.lay-immersive.on .act{grid-area:act}
    #S-D2.lay-immersive.on .base{grid-area:ladder; align-self:center; max-width:none;
      border-top:0; padding:38px 0 16px; min-height:0}
    #S-D2.lay-immersive.on .base .lrungs{max-height:none; overflow:visible}
    #S-D2.lay-immersive.on .base .ladder{--lrung:34px; --lgap:6px}
    /* .esc is absolutely positioned at left:50% and would centre on the
       viewport, not on the stage column. The ladder column is a fixed
       240px + 28px gap, so the stage column's centre is always 134px left
       of the grid's centre — exact at every width from 768px up. */
    #S-D2.lay-immersive.on .esc{left:calc(50% - 134px)}
```

- [ ] **Step 4: Run the assertion again**

Re-navigate at 1280×800 with a fresh cache-buster.
Expected: `ok: true`, `display: 'grid'`, `overflowing: 0`.

- [ ] **Step 5: Check the escalation banner lands on the stage**

Evaluate:

```js
() => {
  const esc = document.getElementById('esc');
  esc.classList.add('on');
  const e = esc.getBoundingClientRect();
  const st = document.querySelector('#S-D2 .stage').getBoundingClientRect();
  const escMid = e.left + e.width / 2, stMid = st.left + st.width / 2;
  esc.classList.remove('on');
  return { ok: Math.abs(escMid - stMid) < 3, escMid, stMid };
}
```

Expected: `ok: true`.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(arena): reflow S-D2 into a stage + ladder grid at >=768px

Scoped by id so S-E1, the other IMMERSIVE screen, keeps its stacked
layout, and gated on .on so the rule cannot paint a hidden screen."
```

---

### Task 5: The compact rail on S-F3

**Files:**
- Modify: `index.html:3013–3019` — correct the stale Phase 3 comment.
- Modify: `index.html:2999–3005` — delete the `.mini` / `.minitok` rules.
- Modify: `index.html:3076` — `#fmini` becomes a mount point.
- Modify: `index.html:3092–3099, 3128–3151, 3169–3176` — `fslots()`, `freact()`, `FSHOW.reset()`.

**Interfaces:**
- Consumes: `MMLADDER.mount()` from Task 2; `MMASYNC.theirVerdict()`.
- Produces: no new globals. `window.FSHOW` keeps its `{react, reset}` shape.

- [ ] **Step 1: Write the browser assertion**

Navigate to `#S-F3` and evaluate:

```js
async () => {
  const wait = ms => new Promise(r => setTimeout(r, ms));
  FSHOW.reset();
  await wait(60);

  const rail = document.querySelector('#S-F3 #fmini .ladder');
  if (!rail) return { ok:false, reason:'no ladder mounted at #fmini' };
  if (document.querySelector('#S-F3 .minitok')) return { ok:false, reason:'.minitok still present' };

  const qCount = MMQ.buildSet().length;
  const rungCount = rail.querySelectorAll('.lrung').length;
  const compact = rail.dataset.layout === 'compact';

  // Play the whole board.
  let guard = 0;
  while (guard++ < 20){
    const btn = document.querySelector('#S-F3 #fact button.btn.move');
    if (!btn) break;
    btn.click(); await wait(1000);
  }
  await wait(1900);

  const res = MMASYNC.result();
  const railYou = rail.querySelector('.lfoot').textContent;
  return {
    ok: rungCount === qCount && compact &&
        rail.querySelector('.lgate') !== null &&
        res && Number.isFinite(res.you),
    rungCount, qCount, compact, railYou, result: res, hash: location.hash
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Expected: `{ ok:false, reason:'no ladder mounted at #fmini' }`.

- [ ] **Step 3: Correct the stale comment**

Replace the comment block at `index.html:3013–3016`:

```js
          /* The board holds one slot per question — 10 under Model A, up to 15
             under Model B — so it must scroll rather than push the stage into
             the button row, and it is capped to leave room for the second
             score row. Phase 3 replaces this with a windowed ladder. */
```

with:

```js
          /* The board holds one slot per question — 10 under Model A, up to 15
             under Model B — so it must scroll rather than push the stage into
             the button row, and it is capped to leave room for the second
             score row. Phase 3 deliberately left this board alone: it tracks
             SAMUEL's score, while the ladder tracks yours, so converting it
             would have changed what the object means. The ladder took over
             the .mini strip above instead. */
```

- [ ] **Step 4: Swap the mini strip for the mount point**

Delete the `.mini` and `.minitok` rules (`index.html:2999–3005`) and the
comment above them, and replace `index.html:3076`:

```html
        <div class="mini" id="fmini"></div>
```

with:

```html
        <div id="fmini" style="flex:0 0 auto; padding:6px 14px 0"></div>
```

- [ ] **Step 5: Wire the script**

In `fslots()`, delete the `#fmini` block:

```js
            document.getElementById('fmini').innerHTML = QF.map((t,n) =>
              '<div class="minitok" data-i="'+n+'">'+(n+1)+'</div>').join('');
```

and add, after the `#fslots` assignment:

```js
            /* Samuel's verdicts on YOUR answers — the same ladder the arena
               draws, laid out along the axis this strip already used. */
            const rungs = QF.map(t => ({ dim: t.q.dim, w: t.w }));
            if (FLADDER) FLADDER.reset(rungs);
            else FLADDER = MMLADDER.mount(document.getElementById('fmini'),
              { rungs, subject: "Samuel's reactions to you", layout: 'compact' });
```

Declare `let FLADDER = null;` alongside `let QF, TOTW;`.

In `freact()`, replace:

```js
            const tok = document.querySelector('#S-F3 .minitok[data-i="'+fi+'"]');
            if(tok){ tok.classList.add('on', hv); tok.textContent = hv==='move' ? '♥' : '·'; }
```

with:

```js
            FLADDER.set(fi, hv);
```

`FSHOW.reset()` needs no change beyond what `fbuild(); fslots();` already does — `fslots()` now calls `FLADDER.reset()`.

- [ ] **Step 6: Run the assertion again**

Re-navigate with a fresh cache-buster.
Expected: `ok: true`, `rungCount: 10`, `qCount: 10`, `compact: true`.

- [ ] **Step 7: Check the strip does not push the stage off-screen**

Resize to **390×667** — the tightest realistic phone — navigate to `#S-F3` and evaluate:

```js
() => {
  const act = document.querySelector('#S-F3 .actf').getBoundingClientRect();
  const mini = document.getElementById('fmini').getBoundingClientRect();
  return { ok: act.bottom <= window.innerHeight + 1, actBottom: act.bottom,
           vh: window.innerHeight, miniHeight: mini.height };
}
```

Expected: `ok: true`. If it fails, reduce `--lrung` in the compact block from
26px to 22px and re-run — do **not** reintroduce a scroll on `#fmini`, which
would hide rungs behind an invisible overflow.

- [ ] **Step 8: Check Model B**

Evaluate `MMQ.setModel('B')`, re-navigate, re-run Steps 6 and 7.
Expected: `rungCount: 15`, and `ok: true` on the overflow check — 15 tiles wrap
to two rows. Restore with `MMQ.setModel('A')`.

- [ ] **Step 9: Commit**

```bash
git add index.html
git commit -m "feat(async): promote S-F3's mini strip to the compact ladder

Same model as the arena rail, laid out horizontally. The flip board is
deliberately untouched — it tracks Samuel's score, not yours — and the
comment that promised otherwise is corrected."
```

---

### Task 6: Full-flow regression and visual sweep

**Files:**
- Modify: `index.html` — only if this task finds defects.

**Interfaces:**
- Consumes: everything from Tasks 1–5.
- Produces: no code. A pass/fail gate on the branch.

**Why this task exists.** Phase 1 shipped eight tasks whose diffs all reviewed
clean while `S-F3` was visibly broken; Phase 2's ten clean per-task reviews
still hid three Critical defects that only a whole-branch pass caught. This
phase changes how much vertical space two fixed-height flex columns consume.
**Do not skip it, and do not substitute reading the diff for looking at the
screen.**

- [ ] **Step 1: Assert the ladder and the score head agree**

The foot readout and the score head derive the same fact by different routes.
Navigate to `#S-D2`, evaluate:

```js
async () => {
  const wait = ms => new Promise(r => setTimeout(r, ms));
  DSHOW.reset(); await wait(60);
  let guard = 0;
  while (guard++ < 40){
    const act = document.getElementById('act');
    const lock = act.querySelector('button[onclick*="lock"]');
    if (lock){ lock.click(); await wait(700);
               const n = document.getElementById('rnext'); if (n) n.click(); await wait(60); }
    else { const m = act.querySelector('button[onclick*="react"]');
           if (!m) break; m.click(); await wait(850); }
    if (location.hash === '#S-D5' || location.hash === '#S-D6') break;
  }
  const py = +sessionStorage.getItem('mm_py');
  /* The rail survives the jump to S-D5/S-D6 — nothing resets it until the
     router re-enters S-D2 — so its final state is still readable here. */
  const s = DSHOW.ladder().state();
  const railPct = Math.round(s.won / s.totalPts * 100);
  return {
    ok: railPct === py && s.cleared === (py >= 70) && s.left === 0,
    py, railPct, won: s.won, totalPts: s.totalPts, gatePts: s.gatePts,
    cleared: s.cleared, left: s.left
  };
}
```

Expected: `ok: true`. The invariants are **`round(won / totalPts * 100) === py`**
and **`cleared === (py >= 70)`**. If they disagree the gate formula in Task 1
has drifted from `MMASYNC.scores()` — fix the module, not the screen.

- [ ] **Step 2: Screenshot the arena at 390×844**

Navigate to `#S-D2`, screenshot at the start, after two verdicts, and once the
gate line has been crossed. Confirm by eye:

- the question ticket and answer card are not clipped or overlapping the rail
- the window shows ~5 rungs and follows the current rung as play advances
- the gate line reads `70% GATE · 84 PTS` and sits above rung 3
- won rungs show `♥ 30`, stayed rungs show `✕ —`, pending rungs show a bare number

- [ ] **Step 3: Screenshot the arena at 1440×900**

Confirm the rail is a full-height right column with every rung visible, the
Move/Stay tickets still sit under the stage, and the `.esc` banner — force it
with `document.getElementById('esc').classList.add('on')` — centres on the
stage column rather than the viewport.

- [ ] **Step 4: Screenshot `S-F3` at 390×844 and 390×667**

Confirm the compact strip wraps cleanly, the gate divider is visible between
tiles, and the Move/Stay row is fully on screen at both heights.

- [ ] **Step 5: Repeat Steps 2–4 under Model B**

Open the dev panel, switch the question model to B, and repeat. This is the
run that matters: 15 questions give 8 arena rungs and 15 async tiles, and any
hardcoded 5 or 10 surfaces here.

- [ ] **Step 6: Regression-check the neighbours**

Screenshot `S-E1` at 1440×900 — it shares the `IMMERSIVE` archetype and must
be unchanged. Walk `S-D5`, `S-D6` and `S-F4` to confirm the results they read
from `sessionStorage` / `MMASYNC.result()` are still populated. Confirm the ☰
screen index still lists every screen.

- [ ] **Step 7: Fix anything found, then commit**

```bash
git add index.html
git commit -m "fix(ladder): resolve full-flow regression findings"
```

If nothing needed fixing, record that explicitly in the PR description rather
than committing an empty change.

- [ ] **Step 8: Open the PR**

```bash
git push -u origin feat/moves-ladder
gh pr create --title "Phase 3 · Moves stacking — the Millionaire ladder" \
  --body "Implements Cluster 3 (decision 3.1) per docs/superpowers/specs/2026-08-25-moves-ladder-design.md."
```

---

## Self-review

**Spec coverage.** Every section of the design doc maps to a task: the
component and its interface → Tasks 1–2; gate arithmetic → Task 1 (with the
0.695 correction); rendering, order and the unreachable-gate treatment →
Task 2; `S-D2` phone → Task 3; `S-D2` desktop, including the `.esc` and
`S-E1` hazards → Task 4; `S-F3` compact and the stale comment → Task 5; the
verification table → Task 6.

**Type consistency.** `model()` returns `gateIndex` (1-based) and rungs
carrying `n` (1-based); `set()` and `focus()` take a **0-based** index and
convert internally via `data-n="i+1"`. Tasks 3 and 5 both pass 0-based indices
(`RUNGOF[i]`, `fi`). The handle members are `set` / `focus` / `reset` /
`state` in Task 2 and are called by exactly those names in Tasks 3, 5 and 6.

**Deviation from the spec, recorded.** The spec's deck-attached example quotes
a gate of 73.5 points rounded to 74; the plan derives 73 from
`ceil(0.695 × total)` so the ladder cannot disagree with the score head. The
gate *rung* is unchanged. Fold this back into the spec if it survives review.
