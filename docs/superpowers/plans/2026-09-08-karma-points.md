# Karma Points Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make standing a real quantity derived from the four behaviours §6.2 rewards, let it order the candidate gallery, show Tolu the receipts behind her own, and put §6.4's undecided display question behind the toggle that will settle it.

**Architecture:** One new global, `MMKARMA`, defined beside `MMSCHED` in the head script block. It holds a four-sub-score record per person and derives everything else — score, band, dots, trait phrases, the sort coefficient, the ledger. Screens never compute karma; they declare a host (`data-karma-of`, `data-karma-head`, `data-karma-ledger`, `data-karma-raise`) and `MMKARMA.paint()` fills it on `hashchange`. That keeps every later task to adding markup, and means the §6.4 toggle has exactly one repaint path.

**Tech Stack:** HTML + CSS + vanilla JavaScript, single file, no build step, no libraries. Verification is browser-driven via the Playwright MCP tools — this repo has no test runner, so every "test" below is an assertion evaluated in the page.

**Spec:** `docs/superpowers/specs/2026-09-08-karma-points-design.md`. Read it alongside this plan — it carries the rejected alternatives behind each locked decision, and the derivation behind the fixture's exact values.

## Global Constraints

- **Single file.** All changes in `/mnt/e/TOOLMAKER/PYTHON/MakeaMoveUI_Mockup/index.html`. No new runtime files, no build step, no libraries, no external CSS/JS. Do not touch `index_mobile_mockup.html` or `index_mockup.html`.
- **Reuse existing components:** `.btn`, `.card`, `.row`, `.chip`, `.opt`, `.note` (+ `.info`), `.confrow` + `.cf-*`, `.why` + `li.y|n|o` + `.src`, `.eyebrow`, `.muted`, `.tag` (+ `.pass`), `.link`, `.pbar`, `.body`, `.foot`, `.seg`, `.devrow`. **Invent no new colours** — every colour comes from a `:root` custom property. New *classes* are fine where nothing fits (Phase 5 added `.e9quote`); new hex values are not.
- **`S-B4` must be added to the `FEED` set** in `classify()`. Nothing else is added to `FEED` or `IMMERSIVE`.
- **Nothing derived may be hardcoded.** The gallery order, the trait tags, the ledger, the §6.6 count and the "what would raise it" block are all computed from `MMKARMA`'s records. A literal `3` beside `pursued()` is a spec violation, not a shortcut — §6.6's first hard constraint is that the number be real.
- **`traits()` returns positive traits only.** A low-standing candidate carries fewer tags, or none. This is §6.3's "without ever announcing to anyone that they have been penalised" and it is enforced in the derivation, never trusted to the copy.
- **No conduct events and no live penalties.** §6.2 requires clean-conduct scoring to wait for an operator ruling (Phase 7's `S-H2`), and the prototype offers no interaction for ghosting — you cannot click *not* doing something. Every penalty in the ledger is seeded.
- **Do not break:** `window.pick`, `window.tog`, `window.togCap`, `window.DSHOW`, `window.FSHOW`, or `MMQ` / `MMASYNC` / `MMLADDER` / `MMSCHED` / `MMDECK` / `MMDEV` / `MMCONF` / `MMTRAIL` / `MMNUM` / `MMDECLINEDATE` / `MMATTACH`. `MMDEV` is **extended** with a third row, not replaced — `setModel` and `setDepth` must still behave exactly as they do today.
- **Commit after every task**, on the branch `feat/karma-points` (already created and holding the spec commit).

## Line numbers shift as you go

Every `index.html:NNNN` reference below was accurate when this plan was written. **Task 1 adds ~150 lines in the head script block and ~25 lines of CSS above it**, and **Task 4 inserts a whole `<section>`**, so every citation after those points drifts — by Task 6 the numbers are off by several hundred. Each step quotes the exact text it is replacing. **Locate by that text, not by the number.** The numbers are a hint about where to look, nothing more.

## Verification setup

`file://` is blocked by the Playwright MCP browser. A static server is likely **already running** on port 3300 from earlier phases — check before starting one, because `python3 -m http.server 3300` fails with `EADDRINUSE` if it is:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:3300/index.html   # 200 = already serving
cd /mnt/e/TOOLMAKER/PYTHON/MakeaMoveUI_Mockup && python3 -m http.server 3300  # only if the above failed
```

Assertions run at **1280×800** unless a step says otherwise, against `http://127.0.0.1:3300/index.html#<screen>`.

**Tooling gotchas carried from Phases 1–5 — each cost real time there:**

- Never call `location.reload()` inside an evaluated function; it destroys the execution context and the evaluate fails instead of returning.
- Re-navigating to an identical URL serves a stale cached copy. Append a fresh cache-buster (`?v=2`, `?v=3`…) **before** the `#hash` every time you need new code.
- Resize **before** navigating — desktop archetype classes are applied on load.
- **Never assert against a screen's `textContent`.** The inline `<script>` fixtures sit inside the `<section>`, so `textContent` contains every fixture string and any substring assertion false-positives. Query rendered elements.
- **Measuring overflow:** `.body` and the stage columns are centred flex columns, so content spills in *both* directions and `scrollHeight` counts only part of it. Use the snippet below. A Phase 3 review under-reported a spill by ~115px the other way.

### The overflow snippet — use this one

The scroll container on a `lay-form` screen is the outer `<section>`, not `.body` — `.body` is `flex:0 0 auto`, so asking whether `.body` scrolls always answers `false`. Measure the child rects against `.body` (that is where content spills from) but ask the **section** whether it scrolls. Substitute the screen id and use verbatim wherever a step says "run the overflow measurement":

```js
(id) => {
  const sec = document.getElementById(id);
  const body = sec.querySelector('.body');
  const box = body.getBoundingClientRect();
  let top = Infinity, bottom = -Infinity;
  [...body.children].forEach(c => { const r = c.getBoundingClientRect();
    top = Math.min(top, r.top); bottom = Math.max(bottom, r.bottom); });
  return {
    spillTop: Math.max(0, box.top - top),
    spillBottom: Math.max(0, bottom - box.bottom),
    sectionScrolls: sec.scrollHeight > sec.clientHeight,
    bodyScrolls: body.scrollHeight > body.clientHeight,
    sec: [sec.scrollHeight, sec.clientHeight]
  };
}
```

A healthy result is `spillTop` and `spillBottom` both 0, with `sectionScrolls` true whenever content is taller than the viewport. `bodyScrolls: false` is normal on a `lay-form` screen and is not a defect. Let entry animations settle before measuring.

### Clear `mm_karma` before asserting standing

Live events persist in `sessionStorage` and raise Tolu's sub-scores at read time. Every assertion about her baseline (78 / Reliable) must start with:

```js
sessionStorage.removeItem('mm_karma');
```

Otherwise a run that touched Task 6's hooks reports a number that looks wrong and is not.

---

## Task 1: `MMKARMA` — the module and its styles

**Files:**
- Modify: `index.html` — insert a CSS block after the `.confrow` rules (`~:633`), and the module after the `MMSCHED` IIFE closes (`~:1722`).

**Interfaces:**
- Consumes: nothing. This task depends on no other code in the file.
- Produces on `window.MMKARMA`:
  - `BEHAVIOURS` — `[{ key, name, w, trait }]`, four entries, `w` summing to 100
  - `TRAIT_GATE` = `75`, `MAX_TAGS` = `2`
  - `of(id)` → `{ responds, shows, complete, feedback }` (live deltas applied for `'you'`)
  - `score(id)` → `Number` 0–100, rounded · `band(id)` → `'Reliable'|'Steady'|'Patchy'` · `dots(id)` → `Number` 0–5
  - `traits(id)` → `String[]`, at most `MAX_TAGS`, positive only
  - `rank(fitPct, id)` → `Number`
  - `dir()` → `'A'|'B'` · `setDir(d)`
  - `tagHTML(id)` / `headHTML(id)` / `ledgerHTML()` / `raiseHTML(id)` → `String`
  - `ledger()` → `[{ id, b, d, when, t }]`, newest first
  - `emit(id, behaviourKey, delta, text)` → `Boolean` (false if already emitted)
  - `pursued()` → `[{ who, asked, band }]`
  - `paint()` — fills every `[data-karma-of|head|ledger|raise]` host in the document

Tasks 2–7 all read these names. They do not change.

- [ ] **Step 1: Write the browser assertion**

Run this first, before writing any code, so you see it fail. Navigate to `#S-B2` and evaluate:

```js
() => {
  const K = window.MMKARMA;
  if (!K || !K.score) return { ok:false, reason:'MMKARMA not defined' };
  sessionStorage.removeItem('mm_karma');

  const sc = { you:K.score('you'), samuel:K.score('samuel'),
               marcus:K.score('marcus'), david:K.score('david'),
               unknown:K.score('nobody-by-that-name') };
  const bd = { samuel:K.band('samuel'), marcus:K.band('marcus'), david:K.band('david') };
  const tr = { you:K.traits('you'), samuel:K.traits('samuel'),
               marcus:K.traits('marcus'), david:K.traits('david') };

  // §6.3 — the sort must disagree with the fit column.
  const rk = { samuel:K.rank(68,'samuel'), david:K.rank(76,'david'), marcus:K.rank(61,'marcus') };

  // Direction B leaks the number; Direction A cannot leak anything.
  K.setDir('A'); const aDavid = K.tagHTML('david'), aSam = K.tagHTML('samuel');
  K.setDir('B'); const bDavid = K.tagHTML('david');
  K.setDir('A');

  // emit is idempotent and moves the sub-score it names.
  const before = K.score('you');
  const first  = K.emit('t-probe','complete',6,'probe');
  const second = K.emit('t-probe','complete',6,'probe');
  const after  = K.score('you');
  sessionStorage.removeItem('mm_karma');

  return {
    ok: sc.you === 78 && sc.samuel === 80 && sc.marcus === 69 && sc.david === 33 &&
        sc.unknown === 50 &&                                  // unknown id falls back, never NaN
        bd.samuel === 'Reliable' && bd.marcus === 'Steady' && bd.david === 'Patchy' &&
        K.dots('you') === 4 &&
        tr.you.length === 2 && tr.you[0] === 'Always shows up' &&   // capped at MAX_TAGS, highest first
        tr.samuel.length === 2 &&
        tr.marcus.length === 1 && tr.marcus[0] === 'Always shows up' &&
        tr.david.length === 0 &&                              // never a negative trait
        rk.samuel > rk.david && rk.david > rk.marcus &&        // Samuel 1st despite lower fit
        aDavid === '' && aSam.indexOf('Responds within a day') > -1 &&
        bDavid.indexOf('33') > -1 &&
        first === true && second === false &&                 // idempotent
        after > before &&
        K.pursued().length === 3,
    sc, bd, tr, rk: { samuel:+rk.samuel.toFixed(1), david:+rk.david.toFixed(1),
                      marcus:+rk.marcus.toFixed(1) },
    emit: { before, after, first, second }
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `http://127.0.0.1:3300/index.html#S-B2`, evaluate the above.
Expected: `{ ok:false, reason:'MMKARMA not defined' }`.

- [ ] **Step 3: Add the karma styles**

Find this exact text — the end of the `.confrow` block, immediately above the mirrored-pairs comment (`~:632`):

```css
  .confrow .cf-wrong{margin-top:8px; font-size:11.5px; background:none; border:none;
    padding:0; cursor:pointer; color:var(--move); font-family:var(--body); text-decoration:underline}

  /* ===== async result · the mirrored-pairs reveal (spec 1.5) =====
```

Replace it with the same text plus the karma block wedged between them:

```css
  .confrow .cf-wrong{margin-top:8px; font-size:11.5px; background:none; border:none;
    padding:0; cursor:pointer; color:var(--move); font-family:var(--body); text-decoration:underline}

  /* ===== karma (spec 6.1–6.4) =====
     The ledger reuses .confrow wholesale; these two states are the only
     addition, and both take colours the block above already uses. A gain is
     --pass, a loss is --move. No new colour enters the file. */
  .confrow[data-state="gain"] .cf-label{color:var(--pass)}
  .confrow[data-state="loss"] .cf-label{color:var(--move)}
  /* Standing, on S-B4. Direction A draws dots, Direction B draws the number;
     both sit in the same host so the toggle swaps one subtree. */
  .kdots{font-size:20px; letter-spacing:4px; color:var(--move); line-height:1}
  .kscore{font-family:var(--display); font-weight:900; font-size:44px;
    line-height:1; color:var(--move)}
  .kband{font-family:var(--display); font-size:11px; letter-spacing:.14em;
    text-transform:uppercase; font-weight:700; margin-top:8px; color:var(--t-dim)}
  .dark .kband{color:var(--ink-dim)}
  .ktraits{display:flex; flex-wrap:wrap; gap:6px; justify-content:center; margin-top:12px}

  /* ===== async result · the mirrored-pairs reveal (spec 1.5) =====
```

- [ ] **Step 4: Insert the module**

Find this exact text — the close of the `MMSCHED` IIFE and the start of the screen wrapper (`~:1720–1724`):

```js
  window.MMSCHED = { TZ, SLOTS, fmtLine, suggestions, save, chosen, clear, mount, slotHTML,
                     openOverlay, closeOverlay, closeAllOverlays };
})();
</script>
    <div class="screenwrap" id="screenwrap">
```

Replace it with the same text, with the module inserted after `MMSCHED`'s IIFE closes and before `</script>`:

```js
  window.MMSCHED = { TZ, SLOTS, fmtLine, suggestions, save, chosen, clear, mount, slotHTML,
                     openOverlay, closeOverlay, closeAllOverlays };
})();

/* ===== KARMA · spec 6.1–6.4, 6.6 ==========================================
   Standing is a DERIVATION over the four behaviours §6.2 rewards, never a
   stored number. One data shape, two renderings: Direction A shows the
   sub-scores clearing the trait gate as phrases, Direction B shows their
   weighted mean as a number. Had Direction A's traits been their own fixture
   copy, the A/B test would compare a real number against invented sentences
   and could not settle anything.

   Screens do not compute karma. They declare a host — data-karma-of,
   -head, -ledger, -raise — and paint() fills it on navigation. That is why
   the §6.4 toggle has exactly one repaint path. */
(function(){
  const BEHAVIOURS = [
    { key:'responds', name:'Responsiveness',      w:30, trait:'Responds within a day' },
    { key:'shows',    name:'Showing up',          w:30, trait:'Always shows up' },
    { key:'complete', name:'Profile & questions', w:20, trait:'Profile fully answered' },
    { key:'feedback', name:'Honest feedback',     w:20, trait:'Leaves honest reviews' }
  ];
  const TRAIT_GATE = 75, MAX_TAGS = 2;

  /* These are the state AFTER the seeded ledger below — the seeded rows are
     DISPLAYED, never applied. Applying both would double-count the month and
     drift these away from the numbers the spec's table fixes. */
  const PEOPLE = {
    you:    { responds:82, shows:90, complete:55, feedback:78 },   // 78 · Reliable
    samuel: { responds:88, shows:84, complete:62, feedback:80 },   // 80 · Reliable
    marcus: { responds:70, shows:76, complete:58, feedback:68 },   // 69 · Steady
    /* David is deliberately the strongest fit (76) and the weakest karma. The
       sort has to visibly disagree with the fit column or §6.3 is unprovable.
       rank()'s coefficient spans 0.7–1.0 — a 1.43x swing — and he leads Samuel
       by 8 fit points, so he must sit below ~47 to be overtaken. RETUNING
       EITHER THIS RECORD OR rank() MEANS RE-DERIVING THAT, or the sort quietly
       stops demonstrating anything while still looking correct. */
    david:  { responds:25, shows:35, complete:50, feedback:25 },   // 33 · Patchy
    /* Bench candidates. The S-B2 drawer can restore any of them into the live
       list, and an unranked candidate would sort to the bottom for the wrong
       reason, so they carry real records rather than relying on the fallback. */
    tunde:  { responds:64, shows:70, complete:44, feedback:58 },
    emeka:  { responds:58, shows:62, complete:40, feedback:54 },
    kwame:  { responds:72, shows:66, complete:48, feedback:60 },
    ifeanyi:{ responds:50, shows:58, complete:38, feedback:46 },
    bode:   { responds:66, shows:74, complete:42, feedback:62 },
    chidi:  { responds:60, shows:56, complete:46, feedback:52 }
  };
  const NEUTRAL = { responds:50, shows:50, complete:50, feedback:50 };

  const KEY = 'mm_karma';
  function live(){
    try { return JSON.parse(sessionStorage.getItem(KEY)) || []; }
    catch(e){ return []; }
  }
  /* Live deltas are applied at READ time, not by mutating PEOPLE. A mutation
     would be lost on reload while the ledger rows survive in sessionStorage,
     and the screen would show rows that no longer explain the number. */
  function of(id){
    const base = PEOPLE[id] || NEUTRAL;
    if (id !== 'you') return base;
    const p = Object.assign({}, base);
    live().forEach(r => { p[r.b] = Math.max(0, Math.min(100, (p[r.b] || 0) + r.d)); });
    return p;
  }

  const score = id => { const p = of(id);
    return Math.round(BEHAVIOURS.reduce((s,b) => s + b.w * (p[b.key] || 0), 0) / 100); };
  const band = id => { const n = score(id);
    return n >= 75 ? 'Reliable' : n >= 50 ? 'Steady' : 'Patchy'; };
  const dots = id => Math.round(score(id) / 20);

  /* POSITIVE TRAITS ONLY. A low-standing candidate carries fewer tags, or
     none at all — §6.3's "without ever announcing to anyone that they have
     been penalised", enforced here rather than trusted to the copy. */
  const traits = id => { const p = of(id);
    return BEHAVIOURS
      .filter(b => (p[b.key] || 0) >= TRAIT_GATE)
      .sort((a,b) => (p[b.key] || 0) - (p[a.key] || 0))
      .slice(0, MAX_TAGS)
      .map(b => b.trait); };

  /* §6.3 — karma moves a candidate by up to ~30% and never swamps fit. */
  const rank = (fitPct, id) => fitPct * (0.7 + 0.3 * score(id) / 100);

  /* §6.4 — the undecided display direction. A is the default: it fits the
     existing .tag styling and the decision log lists it first. */
  const dir = () => sessionStorage.getItem('mm_karma_dir') === 'B' ? 'B' : 'A';
  const setDir = d => sessionStorage.setItem('mm_karma_dir', d === 'B' ? 'B' : 'A');

  function tagHTML(id){
    if (dir() === 'B')
      return '<span class="tag" style="margin-left:6px">Karma ' + score(id) + '</span>';
    return traits(id).map(t =>
      '<span class="tag pass" style="margin-left:6px">' + t + '</span>').join('');
  }
  function headHTML(id){
    const lab = '<div class="kband">' + band(id) + '</div>';
    if (dir() === 'B')
      return '<div class="kscore">' + score(id) + '</div>' + lab;
    const d = dots(id);
    return '<div class="kdots">' + '●'.repeat(d) + '○'.repeat(Math.max(0, 5 - d)) + '</div>'
         + lab
         + '<div class="ktraits">' + traits(id).map(t =>
             '<span class="tag pass">' + t + '</span>').join('') + '</div>';
  }

  /* Seeded history. Displayed, never applied. NOBODY IS NAMED: a named past
     partner would be a fourth person the prototype never shows, and the
     fixture world has no answer when a reader asks who they were. The cast is
     Tolu, David and the gallery; the ledger stays inside it. */
  const SEED = [
    { id:'s-review',  b:'feedback', d: 5, when:'4 Sep',  t:'Left a substantive review after your last call' },
    { id:'s-ghost',   b:'responds', d:-8, when:'29 Aug', t:'Left an async board unanswered for 72 hours' },
    { id:'s-show',    b:'shows',    d: 6, when:'22 Aug', t:'Showed up · Friday’s show' },
    { id:'s-profile', b:'complete', d: 6, when:'14 Aug', t:'Completed Marriage & family' },
    { id:'s-invite',  b:'responds', d: 4, when:'11 Aug', t:'Answered an invite in 3 hours' }
  ];
  const ledger = () => live().slice().reverse().concat(SEED);

  /* One row per event, ever. A revisited screen must not append a second. */
  function emit(id, bkey, delta, text){
    const out = live();
    if (out.some(r => r.id === id)) return false;
    out.push({ id:id, b:bkey, d:delta, when:'Today', t:text });
    sessionStorage.setItem(KEY, JSON.stringify(out));
    paint();
    return true;
  }

  const behName = k => (BEHAVIOURS.filter(b => b.key === k)[0] || { name:k }).name;
  /* Every string here is written by this file — no user-typed text reaches the
     ledger, which is why concatenation is safe. S-E9's textContent rule exists
     because the decline reason IS user-typed; do not read this as a licence
     to relax it there. */
  function ledgerHTML(){
    return ledger().map(r =>
      '<div class="confrow" data-state="' + (r.d < 0 ? 'loss' : 'gain') + '">'
      + '<div class="cf-head"><div class="cf-name">' + r.t + '</div>'
      + '<div class="cf-label">' + (r.d > 0 ? '+' : '') + r.d + '</div></div>'
      + '<div class="cf-val">' + r.when + ' · ' + behName(r.b) + '</div>'
      + '</div>').join('');
  }

  /* What would raise it — derived from the weakest behaviour, not written, so
     it still points at the right thing after live events move the scores. */
  const ACTIONS = {
    responds: { t:'Answer invites and boards while they are still open.',
                cta:'Open your async board ›', href:'#S-F1', alt:null },
    shows:    { t:'Turn up to what you schedule. Every session you attend counts.',
                cta:'See your scheduled show ›', href:'#S-C5', alt:null },
    complete: { t:'Two profile sections are still blank, and they are the ones your candidates have already answered.',
                cta:'Answer Personality &amp; lifestyle ›', href:'#S-P3',
                alt:{ cta:'Love &amp; intimacy ›', href:'#S-P4' } },
    feedback: { t:'Leave a real review after your next call, not a star and a shrug.',
                cta:'See your last review ›', href:'#S-E5', alt:null }
  };
  function raiseHTML(id){
    const p = of(id);
    const weakest = BEHAVIOURS.slice()
      .sort((a,b) => (p[a.key] || 0) - (p[b.key] || 0))[0];
    const a = ACTIONS[weakest.key];
    return '<div class="eyebrow">✦ What would raise it</div>'
      + '<p class="muted" style="font-size:12.5px;line-height:1.5;margin:10px 0 0">'
      + weakest.name + ' is your weakest of the four. ' + a.t + '</p>'
      + '<a class="btn primary block" href="' + a.href + '" style="margin-top:12px">' + a.cta + '</a>'
      + (a.alt ? '<a class="link center" href="' + a.alt.href
                 + '" style="margin-top:8px;display:block">' + a.alt.cta + '</a>' : '');
  }

  /* §6.6 — the count is DERIVED from this list's length, never written. An
     inflated or vague number is exactly the dark pattern the decision forbids.
     `band` keys into the gallery's USER_GAPS so the ask can never point at a
     section she has already filled in. */
  const PURSUED = [
    { who:'A teacher in Lewisham',  asked:'how you like to be close to someone', band:'love' },
    { who:'An engineer in Croydon', asked:'what your weekends actually look like', band:'person' },
    { who:'A GP in Reading',        asked:'what you do to unwind', band:'person' }
  ];

  /* Hosts are declarative, so a screen added by a later task needs no wiring
     here. Guarded because the module is defined ABOVE the markup: at parse
     time there is nothing to fill. */
  function paint(){
    document.querySelectorAll('[data-karma-of]').forEach(el => {
      el.innerHTML = tagHTML(el.getAttribute('data-karma-of')); });
    document.querySelectorAll('[data-karma-head]').forEach(el => {
      el.innerHTML = headHTML(el.getAttribute('data-karma-head')); });
    document.querySelectorAll('[data-karma-ledger]').forEach(el => {
      el.innerHTML = ledgerHTML(); });
    document.querySelectorAll('[data-karma-raise]').forEach(el => {
      el.innerHTML = raiseHTML(el.getAttribute('data-karma-raise')); });
  }
  window.addEventListener('hashchange', paint);
  document.addEventListener('DOMContentLoaded', paint);

  window.MMKARMA = { BEHAVIOURS, TRAIT_GATE, MAX_TAGS,
                     of, score, band, dots, traits, rank,
                     dir, setDir, tagHTML, headHTML, ledgerHTML, raiseHTML,
                     ledger, emit, paint,
                     pursued: () => PURSUED.slice() };
})();
</script>
    <div class="screenwrap" id="screenwrap">
```

- [ ] **Step 5: Run the assertion to verify it passes**

Navigate to `http://127.0.0.1:3300/index.html?v=2#S-B2` (fresh cache-buster) and evaluate the Step 1 function.
Expected: `ok: true`, with `sc: {you:78, samuel:80, marcus:69, david:33, unknown:50}` and `rk: {samuel:63.9, david:60.7, marcus:55.3}`.

- [ ] **Step 6: Verify nothing regressed**

Evaluate:

```js
() => ({
  globals: ['MMQ','MMASYNC','MMLADDER','MMSCHED','MMDEV','pick','tog']
    .filter(n => typeof window[n] === 'undefined'),
  errors: window.__mmErr || null
})
```
Expected: `globals: []`. Then check the browser console for errors — a syntax error inside this script block takes `MMQ`, `MMSCHED` and the whole head module set down with it, so an empty `globals` array is the real pass condition.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat(karma): add MMKARMA — standing derived from the four rewarded behaviours"
```

---

## Task 2: `S-B2` — the sort and the card slot

**Files:**
- Modify: `index.html` — `cardHTML()` (`~:2590`), `render()` (`~:2609`), the heading copy (`~:2334`), and the gallery IIFE's exports.

**Interfaces:**
- Consumes: `MMKARMA.rank(fitPct, id)`, `MMKARMA.paint()`; the gallery's own `fit(c)` (`~:2536`) and `byId(id)`.
- Produces: `window.MMGAL = { render }` — Task 5's toggle calls it to repaint the cards.

- [ ] **Step 1: Write the browser assertion**

Navigate to `#S-B2` and evaluate:

```js
() => {
  /* Force Direction A — a previous run may have left 'B' in storage, and the
     tag counts below only mean anything under A. */
  sessionStorage.removeItem('mm_karma_dir');
  if (window.MMGAL) MMGAL.render();
  const cards = [...document.querySelectorAll('#b2list .cand')];
  if (!cards.length) return { ok:false, reason:'no cards rendered' };
  const order = cards.map(el => el.dataset.id);
  const fits  = cards.map(el => parseInt(el.querySelector('.fit .n').textContent, 10));
  const tags  = cards.map(el => el.querySelectorAll('[data-karma-of] .tag').length);
  return {
    ok: order[0] === 'samuel' && order[1] === 'david' && order[2] === 'marcus' &&
        fits[1] > fits[0] &&                       // David outranks Samuel on fit and still sits below
        tags[0] === 2 && tags[1] === 0 && tags[2] === 1 &&
        typeof window.MMGAL === 'object' && typeof MMGAL.render === 'function',
    order, fits, tags
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `http://127.0.0.1:3300/index.html?v=3#S-B2`, evaluate.
Expected: `ok:false` with `order: ['david','samuel','marcus']` and `tags: [0,0,0]` — the stored order, and no karma slot in the markup yet.

- [ ] **Step 3: Add the karma slot to the card**

Find this exact text inside `cardHTML` (`~:2597`):

```js
        <div class="t1">${c.name}, ${c.age}${lowTag}</div>
```

Replace with:

```js
        <div class="t1">${c.name}, ${c.age}${lowTag}<span data-karma-of="${c.id}"></span></div>
```

The span is left empty on purpose. `render()` calls `MMKARMA.paint()` at the end of Step 4, which fills every host in one pass — so the toggle and the gallery share one code path instead of two that can drift.

- [ ] **Step 4: Sort by rank and paint**

Find this exact text at the top of `render()` (`~:2609`):

```js
  function render(){
    const list = document.getElementById('b2list');
    if (!list) return;
    list.innerHTML = state.live.map(id => cardHTML(byId(id))).join('');
```

Replace with:

```js
  function render(){
    const list = document.getElementById('b2list');
    if (!list) return;
    /* §6.3 — low karma surfaces less often. Ordering only: `state.live` keeps
       its stored order so remove/undo/restore are untouched, and the §6.4
       toggle must NOT change this sequence, only how each card draws it. */
    const ordered = state.live.slice().sort((a,b) =>
      MMKARMA.rank(fit(byId(b)), b) - MMKARMA.rank(fit(byId(a)), a));
    list.innerHTML = ordered.map(id => cardHTML(byId(id))).join('');
```

Then find this exact text, at the end of the same function (`~:2622`):

```js
    renderDrawer();   // defined in Task 4
    renderGap();      // defined in Task 5
    renderProfile();  // defined in Task 6
  }
```

Replace with:

```js
    renderDrawer();   // defined in Task 4
    renderGap();      // defined in Task 5
    renderProfile();  // defined in Task 6
    MMKARMA.paint();  // fills the [data-karma-of] slots the cards just declared
  }
```

Those three trailing comments refer to tasks in the *Phase 2 gallery plan*, not this one. Leave them exactly as they are — rewriting them would break a reader who goes looking for that plan.

- [ ] **Step 5: Explain the inversion in the heading**

Find this exact text (`~:2335`):

```html
          <p class="muted" style="font-size:12.5px;margin-top:-6px">Photo and a few details now — full profiles unlock only after they accept your invite. No bios to doom-scroll.</p>
```

Replace with:

```html
          <p class="muted" style="font-size:12.5px;margin-top:-6px">Photo and a few details now — full profiles unlock only after they accept your invite. No bios to doom-scroll.</p>
          <p class="muted" style="font-size:12px;margin-top:-10px">Ordered by fit and by how each of them treats people.</p>
```

One line and no more. The card slot is the real explanation — under Direction A Samuel carries two trait tags and David carries none — so this only has to stop the fit column reading as a bug.

- [ ] **Step 6: Export the renderer under a name worth calling**

The gallery already publishes its internals. Find this exact text at the bottom of the IIFE (`~:2855–2857`):

```js
  Object.assign(window, { CANDIDATES, BANDS, RESTORE_PRICE, UNDO_MS, USER_GAPS,
    fit, conf, scoreLabel, confLabel, bulletCount, PHOTO, cardPhoto, facePhoto, heroPhoto,
    render, state, byId, save, load, remove, restore, confirmRestore, renderProfile });
```

Replace with:

```js
  Object.assign(window, { CANDIDATES, BANDS, RESTORE_PRICE, UNDO_MS, USER_GAPS,
    fit, conf, scoreLabel, confLabel, bulletCount, PHOTO, cardPhoto, facePhoto, heroPhoto,
    render, state, byId, save, load, remove, restore, confirmRestore, renderProfile,
    /* Task 5's §6.4 toggle repaints the cards through this. `render` is
       already global, but a bare `window.render` is too generic a name for
       another module to reach for. */
    MMGAL: { render } });
```

- [ ] **Step 7: Run the assertion to verify it passes**

Navigate to `http://127.0.0.1:3300/index.html?v=4#S-B2`, evaluate Step 1's function.
Expected: `ok:true`, `order: ['samuel','david','marcus']`, `fits: [68,76,61]`, `tags: [2,0,1]`.

- [ ] **Step 8: Verify remove / undo / restore still work**

The sort must not have disturbed the drawer. On `#S-B2`:

1. Click the ✕ on the first card. Confirm the toast appears and the card leaves the list.
2. Click **Undo** inside the 6-second window. Confirm the card returns **and lands back in rank order**, not at the end.
3. Remove a card and let the toast expire. Open the drawer, restore a bench candidate, and confirm it renders with a karma slot and sorts into a sensible position rather than the bottom.

- [ ] **Step 9: Screenshot both viewports**

At 1280×800 and 390×844, screenshot `#S-B2`. Check the trait tags wrap rather than overflow the card at 390px — Samuel carries two and they are long strings. Run the overflow measurement for `S-B2`.

- [ ] **Step 10: Commit**

```bash
git add index.html
git commit -m "feat(karma): order the gallery by karma and show standing on the card"
```

---

## Task 3: `S-C3` and `S-F2A` — Tolu's standing on the other party's screen

**Files:**
- Modify: `index.html` — the Tolu profile card in `S-C3` (`~:2924`) and the identical one in `S-F2A` (`~:4055`).

**Interfaces:**
- Consumes: `MMKARMA.paint()` via the `hashchange` listener registered in Task 1. No new code, only hosts.

This is the mirror. Karma on a candidate's card is Tolu *reading* someone's standing; these two are Tolu's own standing being read *by someone else*, which is what §6.3 is actually about.

- [ ] **Step 1: Write the browser assertion**

Evaluate, after navigating to `#S-C3`:

```js
() => {
  const host = s => document.querySelector('#' + s + ' [data-karma-of="you"]');
  const c3 = host('S-C3'), f2a = host('S-F2A');
  if (!c3 || !f2a) return { ok:false, reason:'karma host missing', c3:!!c3, f2a:!!f2a };
  MMKARMA.setDir('A'); MMKARMA.paint();
  const aC3 = c3.querySelectorAll('.tag').length;
  MMKARMA.setDir('B'); MMKARMA.paint();
  const bC3 = c3.textContent.trim(), bF2a = f2a.textContent.trim();
  MMKARMA.setDir('A'); MMKARMA.paint();
  return {
    ok: aC3 === 2 && bC3.indexOf('78') > -1 && bF2a.indexOf('78') > -1,
    aC3, bC3, bF2a
  };
}
```

Reading `.textContent` of the **host element** is safe — it is a single span this module owns. The rule forbidden elsewhere is asserting against a whole `<section>`'s `textContent`, which sweeps up every inline fixture string.

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `http://127.0.0.1:3300/index.html?v=5#S-C3`, evaluate.
Expected: `{ ok:false, reason:'karma host missing', c3:false, f2a:false }`.

- [ ] **Step 3: Add the host to `S-C3`**

Find this exact text (`~:2924`):

```html
            <div><div style="font-weight:800;font-size:18px">Tolu, 37</div><div class="muted" style="font-size:13px">Project manager · 2 kids · London</div><div style="margin-top:6px"><span class="tag pass">Family fit</span> <span class="tag">AA</span></div></div>
```

**There are two identical copies of this line** — this one inside `S-C3` and one inside `S-F2A`. Confirm you are in `S-C3` by checking that the enclosing `<section>` opens with `id="S-C3"`. Replace with:

```html
            <div><div style="font-weight:800;font-size:18px">Tolu, 37</div><div class="muted" style="font-size:13px">Project manager · 2 kids · London</div><div style="margin-top:6px"><span class="tag pass">Family fit</span> <span class="tag">AA</span><span data-karma-of="you"></span></div></div>
```

- [ ] **Step 4: Add the host to `S-F2A`**

Find the *second* copy of the same line, inside the section opening `id="S-F2A"` (`~:4055`), and apply the identical replacement:

```html
            <div><div style="font-weight:800;font-size:18px">Tolu, 37</div><div class="muted" style="font-size:13px">Project manager · 2 kids · London</div><div style="margin-top:6px"><span class="tag pass">Family fit</span> <span class="tag">AA</span><span data-karma-of="you"></span></div></div>
```

`S-F2A` is beyond the cluster map's listed screens and is included deliberately: it is the same card in the same moment — someone deciding whether to commit to Tolu, here while being asked to pay 50% — and showing her standing on one but not the other would be a visible inconsistency for a one-line saving.

- [ ] **Step 5: Run the assertion to verify it passes**

Navigate to `http://127.0.0.1:3300/index.html?v=6#S-C3`, evaluate Step 1's function.
Expected: `ok:true`, `aC3: 2`, both `bC3` and `bF2a` containing `78`.

- [ ] **Step 6: Screenshot both viewports**

Screenshot `#S-C3` and `#S-F2A` at 1280×800 and 390×844. The tag row now carries four chips at 390px — confirm it wraps inside the card rather than pushing the avatar out of line.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat(karma): show Tolu's standing on S-C3 and S-F2A"
```

---

## Task 4: `S-B4` — her standing and the receipts

**Files:**
- Modify: `index.html` — insert a `<section>` after `S-B3` (`~:2873`), add `S-B4` to `FEED` in `classify()` (`~:4675`), add an entry row to `S-A9` (`~:2036`).

**Interfaces:**
- Consumes: `MMKARMA.paint()` and the four host attributes. No new JavaScript — the screen is markup plus hosts.

- [ ] **Step 1: Write the browser assertion**

Navigate to `#S-B4` and evaluate:

```js
() => {
  sessionStorage.removeItem('mm_karma');
  const sec = document.getElementById('S-B4');
  if (!sec) return { ok:false, reason:'S-B4 not in the document' };
  MMKARMA.paint();
  const rows  = sec.querySelectorAll('[data-karma-ledger] .confrow');
  const loss  = sec.querySelectorAll('[data-karma-ledger] .confrow[data-state="loss"]');
  const gain  = sec.querySelectorAll('[data-karma-ledger] .confrow[data-state="gain"]');
  const raise = sec.querySelector('[data-karma-raise] .btn');
  MMKARMA.setDir('A'); MMKARMA.paint();
  const aDots = !!sec.querySelector('.kdots'), aScore = !!sec.querySelector('.kscore');
  MMKARMA.setDir('B'); MMKARMA.paint();
  const bDots = !!sec.querySelector('.kdots'), bScore = !!sec.querySelector('.kscore');
  const bNum  = (sec.querySelector('.kscore') || {}).textContent;
  MMKARMA.setDir('A'); MMKARMA.paint();
  return {
    ok: rows.length === 5 && loss.length === 1 && gain.length === 4 &&
        loss[0].querySelector('.cf-label').textContent.trim() === '-8' &&
        aDots === true  && aScore === false &&
        bDots === false && bScore === true && bNum === '78' &&
        !!raise && raise.getAttribute('href') === '#S-P3' &&
        sec.classList.contains('lay-feed'),
    rows: rows.length, loss: loss.length, gain: gain.length,
    dir: { aDots, aScore, bDots, bScore, bNum },
    raiseHref: raise && raise.getAttribute('href'),
    lay: [...sec.classList].filter(c => c.indexOf('lay-') === 0)
  };
}
```

The `raiseHref` check is the one that proves `raiseHTML` derives rather than recites: `complete` (55) is Tolu's weakest behaviour, so the CTA must point at `#S-P3`. If someone hardcodes the copy, this still passes — but Step 7 changes her weakest behaviour and re-checks, which a hardcoded block cannot survive.

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `http://127.0.0.1:3300/index.html?v=7#S-B4`, evaluate.
Expected: `{ ok:false, reason:'S-B4 not in the document' }`.

- [ ] **Step 3: Insert `S-B4`**

Find this exact text — the close of `S-B3` and the start of group C (`~:2872–2876`):

```html
        </div>
      </section>

      <!-- ============ GROUP C · INVITE & SCHEDULING ============ -->
      <section class="screen paper" id="S-C1" data-group="C · Invite & schedule" data-title="Compose invite · seed 5 slots">
```

Replace with the same text with `S-B4` inserted between them:

```html
        </div>
      </section>

      <!-- S-B4 · spec 6.2/6.3/6.4. A leaf off the gallery, not a branch out of
           a flow, so Prev/Next reaches it after S-B3 without wedging a dead end
           into the middle of an arc — the same reasoning that kept S-E6A beside
           S-E6 while the decline branch went to S-E8/S-E9.
           Every block here is a host: MMKARMA.paint() fills them on navigation
           and on the §6.4 toggle. Nothing on this screen is written. -->
      <section class="screen paper" id="S-B4" data-group="B · Matching" data-title="Your karma">
        <div class="pbar"><a class="back" href="#S-B2">‹</a><h1>Your karma</h1><a class="act" href="#S-A9">Profile ›</a></div>
        <div class="body">
          <div class="card" style="text-align:center">
            <div class="eyebrow">Your standing</div>
            <div data-karma-head="you" style="margin-top:12px"></div>
          </div>
          <!-- §6.3, stated once and not elaborated into a rank. The consequence
               lands in other people's galleries, which this prototype can never
               show her, so it is said plainly instead of mocked up. -->
          <div class="note info">Your candidates are ordered by fit and by how they
            treat people. The same ordering works on your side — your standing puts
            you in front of more people than it did last month.</div>
          <h2 class="scrn" style="font-size:20px">How you got here</h2>
          <p class="muted" style="font-size:12.5px;margin-top:-6px">Everything that
            moved your standing. Nobody else ever sees this list — what other people
            see is on your card, and it never includes what you lost.</p>
          <div class="card" data-karma-ledger="you"></div>
          <div class="card" data-karma-raise="you"></div>
        </div>
      </section>

      <!-- ============ GROUP C · INVITE & SCHEDULING ============ -->
      <section class="screen paper" id="S-C1" data-group="C · Invite & schedule" data-title="Compose invite · seed 5 slots">
```

- [ ] **Step 4: Register the desktop archetype**

Find this exact text (`~:4675`):

```js
    const FEED = new Set(['S-B2','S-E3','S-E4','S-E6A','S-A8','S-H1','S-G3','S-H5','S-QD1']);
```

Replace with:

```js
    const FEED = new Set(['S-B2','S-B4','S-E3','S-E4','S-E6A','S-A8','S-H1','S-G3','S-H5','S-QD1']);
```

`S-B4` is a long row list. Without this it defaults to a centred form sheet on desktop and the ledger renders in a narrow column.

- [ ] **Step 5: Add the entry point on `S-A9`**

Find this exact text — the last of the five profile rows on `S-A9` (`~:2036`):

```html
          <a class="row" href="#S-P5"><div class="av g5 s40"><svg class="icon" style="width:20px;height:20px"><use href="#i-ring"/></svg></div><div class="grow"><div class="t1">Marriage &amp; family</div><div class="t2">Readiness, kids, wedding, family views</div></div><span class="chev">›</span></a>
```

Replace with the same row plus a karma row beneath it:

```html
          <a class="row" href="#S-P5"><div class="av g5 s40"><svg class="icon" style="width:20px;height:20px"><use href="#i-ring"/></svg></div><div class="grow"><div class="t1">Marriage &amp; family</div><div class="t2">Readiness, kids, wedding, family views</div></div><span class="chev">›</span></a>
          <a class="row" href="#S-B4"><div class="av g1 s40"><svg class="icon" style="width:20px;height:20px"><use href="#i-sparkle"/></svg></div><div class="grow"><div class="t1">Your karma</div><div class="t2">Your standing, and everything that moved it</div></div><span class="chev">›</span></a>
```

- [ ] **Step 6: Run the assertion to verify it passes**

Navigate to `http://127.0.0.1:3300/index.html?v=8#S-B4`, evaluate Step 1's function.
Expected: `ok:true`, `rows:5`, `loss:1`, `gain:4`, `bNum:'78'`, `raiseHref:'#S-P3'`, `lay:['lay-feed']`.

- [ ] **Step 7: Prove the raise block derives rather than recites**

Evaluate:

```js
() => {
  sessionStorage.removeItem('mm_karma');
  // Push `complete` from 55 to 95, making `feedback` (78) the weakest.
  MMKARMA.emit('t-raise','complete',40,'probe');
  MMKARMA.paint();
  const href = document.querySelector('#S-B4 [data-karma-raise] .btn').getAttribute('href');
  sessionStorage.removeItem('mm_karma');
  MMKARMA.paint();
  const back = document.querySelector('#S-B4 [data-karma-raise] .btn').getAttribute('href');
  return { ok: href === '#S-E5' && back === '#S-P3', href, back };
}
```

Expected: `ok:true`. A hardcoded block returns `#S-P3` both times and fails here — which is the point of running it.

- [ ] **Step 8: Screenshot both viewports and measure overflow**

At 1280×800 and 390×844, screenshot `#S-B4` in **both** toggle directions (set `mm_karma_dir` and re-navigate with a fresh cache-buster). Run the overflow measurement for `S-B4` at both sizes. Confirm `lay-feed` gives it the wide column on desktop, and that the five ledger rows plus three cards scroll rather than spill at 390px.

- [ ] **Step 9: Commit**

```bash
git add index.html
git commit -m "feat(karma): add S-B4, her standing and the receipts behind it"
```

---

## Task 5: The §6.4 toggle

**Files:**
- Modify: `index.html` — `MMDEV` (`~:4889`) and the dev panel markup (`~:5001`).

**Interfaces:**
- Consumes: `MMKARMA.dir/setDir/paint`, `MMGAL.render` (Task 2).
- Produces: `MMDEV.setKarmaDir(d)`.

`MMDEV` was built for this. The comment above it names spec 6.4, and so does the `.devpanel` CSS block — this fills a slot the file already reserved.

- [ ] **Step 1: Write the browser assertion**

Navigate to `#S-B2` and evaluate:

```js
() => {
  if (typeof MMDEV.setKarmaDir !== 'function')
    return { ok:false, reason:'MMDEV.setKarmaDir not defined' };
  const ids = () => [...document.querySelectorAll('#b2list .cand')].map(el => el.dataset.id);
  const slot = () => document.querySelector('#b2list .cand [data-karma-of]').innerHTML;

  MMDEV.setKarmaDir('A');
  const orderA = ids(), slotA = slot();
  const btnA = document.getElementById('devka').classList.contains('on');
  MMDEV.setKarmaDir('B');
  const orderB = ids(), slotB = slot();
  const btnB = document.getElementById('devkb').classList.contains('on');
  MMDEV.setKarmaDir('A');

  return {
    ok: orderA.join() === orderB.join() &&        // the toggle must NOT reorder
        orderA[0] === 'samuel' &&
        slotA !== slotB &&                        // but it must change the drawing
        slotA.indexOf('Responds within a day') > -1 &&
        slotB.indexOf('Karma 80') > -1 &&
        btnA === true && btnB === true &&         // each direction lights its own button
        MMQ.model() === MMQ.model(),              // the other toggles still answer
    orderA, orderB, slotA, slotB, btnA, btnB
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `http://127.0.0.1:3300/index.html?v=9#S-B2`, evaluate.
Expected: `{ ok:false, reason:'MMDEV.setKarmaDir not defined' }`.

- [ ] **Step 3: Teach `MMDEV` about the direction**

Find this exact text inside `MMDEV.paint()` (`~:4901`):

```js
      const pt = document.getElementById('devpt'), pf = document.getElementById('devpf');
      if(pt && pf){
        const full = !!sessionStorage.getItem('mm_answers');
        pt.classList.toggle('on', !full);
        pf.classList.toggle('on', full);
      }
    },
```

Replace with:

```js
      const pt = document.getElementById('devpt'), pf = document.getElementById('devpf');
      if(pt && pf){
        const full = !!sessionStorage.getItem('mm_answers');
        pt.classList.toggle('on', !full);
        pf.classList.toggle('on', full);
      }
      const ka = document.getElementById('devka'), kb = document.getElementById('devkb');
      if(ka && kb){
        const d = MMKARMA.dir();
        ka.classList.toggle('on', d === 'A');
        kb.classList.toggle('on', d === 'B');
      }
    },
```

Then find this exact text — the end of `setDepth` and the close of the `MMDEV` object (`~:4913`):

```js
    setDepth(d){
      MMQ.setProfileDepth(d); MMDEV.paint();
      if(window.MMCONF)  MMCONF.paint();
      if(window.MMTRAIL) MMTRAIL.paint();
    }
  };
```

Replace with:

```js
    setDepth(d){
      MMQ.setProfileDepth(d); MMDEV.paint();
      if(window.MMCONF)  MMCONF.paint();
      if(window.MMTRAIL) MMTRAIL.paint();
    },
    /* Spec 6.4 — A/B on how standing is DRAWN, never on how it is ordered.
       The fan-out lives here rather than inside MMKARMA.setDir so it reads
       like setDepth above, and so the gallery stays an optional consumer. */
    setKarmaDir(d){
      MMKARMA.setDir(d); MMDEV.paint();
      MMKARMA.paint();
      if(window.MMGAL) MMGAL.render();
    }
  };
```

- [ ] **Step 4: Add the panel row**

Find this exact text in the dev panel markup (`~:5002–5009`):

```html
  <div class="devrow">
    <div class="lbl2">Profile depth</div>
    <div class="seg">
      <button id="devpt" onclick="MMDEV.setDepth('thin')">Thin</button>
      <button id="devpf" onclick="MMDEV.setDepth('full')">Complete</button>
    </div>
    <div class="note2">Switches the demo answer set. Drives S-E6's confidence meters and how much of S-F4's mirrored-pairs reveal reads "not answered yet".</div>
  </div>
```

Replace with the same block plus the karma row beneath it:

```html
  <div class="devrow">
    <div class="lbl2">Profile depth</div>
    <div class="seg">
      <button id="devpt" onclick="MMDEV.setDepth('thin')">Thin</button>
      <button id="devpf" onclick="MMDEV.setDepth('full')">Complete</button>
    </div>
    <div class="note2">Switches the demo answer set. Drives S-E6's confidence meters and how much of S-F4's mirrored-pairs reveal reads "not answered yet".</div>
  </div>
  <div class="devrow">
    <div class="lbl2">Karma display</div>
    <div class="seg">
      <button id="devka" onclick="MMDEV.setKarmaDir('A')">A · traits</button>
      <button id="devkb" onclick="MMDEV.setKarmaDir('B')">B · score</button>
    </div>
    <div class="note2">A shows what karma means — "Responds within a day". B shows the number. Changes how standing is drawn on S-B2, S-C3, S-F2A and S-B4; never changes the gallery's order. Note B necessarily reveals a low score where A cannot.</div>
  </div>
```

- [ ] **Step 5: Run the assertion to verify it passes**

Navigate to `http://127.0.0.1:3300/index.html?v=10#S-B2`, open the dev panel (the gear in the top bar), evaluate Step 1's function.
Expected: `ok:true`, with `orderA` and `orderB` identical and `slotA`/`slotB` different.

- [ ] **Step 6: Verify the other two toggles still work**

In the panel, click **B · 10 + 5**, then **A · 10 total**, then **Complete**, then **Thin**. Confirm each still highlights and that `#S-E6`'s confidence rows still repaint on the depth switch. A broken `MMDEV.paint()` shows up as buttons that stop lighting.

- [ ] **Step 7: Screenshot the panel**

Screenshot the open dev panel at 1280×800. Confirm three rows fit and the panel has not outgrown its `width:260px`.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "feat(karma): add the 6.4 display-direction toggle to dev settings"
```

---

## Task 6: The live emit points

**Files:**
- Modify: `index.html` — seven call sites: `ffinish()` in `S-F3`, the feet of `S-P3` and `S-P4`, `MMDECK.submit()`, `MMCONF.reject()`, `MMNUM.submit()`, `MMDECLINEDATE.submit()`.

**Interfaces:**
- Consumes: `MMKARMA.emit(id, behaviourKey, delta, text)`.

Every one of these is an action **Tolu** takes on her own phone. `S-C3` and `S-C4` are deliberately excluded: `S-C3` carries a "David's view" tag in its `.pbar`, so the promptness earned there is his, and `S-C4` is reached from it. That is the trap in wiring responsiveness — the screens that most look like a prompt reply are the ones on the other party's phone.

- [ ] **Step 1: Write the browser assertion**

Navigate to `#S-B4` and evaluate:

```js
() => {
  sessionStorage.removeItem('mm_karma');
  const ids = () => MMKARMA.ledger().map(r => r.id);
  const base = MMKARMA.score('you');

  // Fire the two that are reachable as plain function calls.
  MMCONF.reject('core-value');
  const afterReject = ids();
  MMCONF.reject('core-value');                 // second call, same probe
  const afterTwice = ids();

  const rows = MMKARMA.ledger().filter(r => r.when === 'Today');
  const moved = MMKARMA.score('you');
  sessionStorage.removeItem('mm_conf_rejected');
  sessionStorage.removeItem('mm_karma');

  return {
    ok: afterReject.indexOf('ev-conf-reject') > -1 &&
        afterTwice.filter(i => i === 'ev-conf-reject').length === 1 &&   // never twice
        rows.length === 1 && moved > base,
    base, moved, rows: rows.map(r => r.id)
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `http://127.0.0.1:3300/index.html?v=11#S-B4`, evaluate.
Expected: `ok:false`, with `rows: []` — nothing emits yet.

- [ ] **Step 3: Responsiveness — finishing the async board**

Find this exact text inside `ffinish()` in `S-F3` (`~:4299`):

```js
            const s = MMASYNC.scores(verdicts);
            const them = s.them, you = s.you;
            const pass = them>=70 && you>=70;
            MMASYNC.saveResult({ you, them, verdicts });
```

All four lines carry 12 spaces of indentation — match it exactly or the edit will not apply. Replace with:

```js
            const s = MMASYNC.scores(verdicts);
            const them = s.them, you = s.you;
            const pass = them>=70 && you>=70;
            MMASYNC.saveResult({ you, them, verdicts });
            /* §6.2 responsiveness. Hers: this board is Tolu's side of the
               async game — the partner here is Samuel. */
            MMKARMA.emit('ev-board','responds',4,'Finished your async board inside the window');
```

- [ ] **Step 4: Completeness — the two blank profile sections**

Find this exact text — the note and foot at the end of **`S-P3`** (`~:2126–2128`). The `Save · back to profile` line appears five times in the file; this is the one preceded by the *Lifestyle and Values* note:

```html
          <div class="note">Seeds your <b>Lifestyle</b> and <b>Values</b> dimension weights — and gives the show sharper questions.</div>
        </div>
        <div class="foot"><a class="btn primary block" href="#S-A9">Save · back to profile</a></div>
```

Replace with:

```html
          <div class="note">Seeds your <b>Lifestyle</b> and <b>Values</b> dimension weights — and gives the show sharper questions.</div>
        </div>
        <div class="foot"><a class="btn primary block" href="#S-A9" onclick="MMKARMA.emit('ev-p3','complete',6,'Completed Personality &amp; lifestyle')">Save · back to profile</a></div>
```

Then find the same line at the end of **`S-P4`** — the one preceded by the *sex while dating* option group (`~:2143–2145`):

```html
            <div style="display:flex;flex-direction:column;gap:8px"><div class="opt" onclick="pick(this)"><span class="dot"></span>Sex while dating</div><div class="opt" onclick="pick(this)"><span class="dot"></span>Wait till after marriage</div><div class="opt sel" onclick="pick(this)"><span class="dot"></span>Either works</div></div>
          </div>
        </div>
        <div class="foot"><a class="btn primary block" href="#S-A9">Save · back to profile</a></div>
```

Replace with:

```html
            <div style="display:flex;flex-direction:column;gap:8px"><div class="opt" onclick="pick(this)"><span class="dot"></span>Sex while dating</div><div class="opt" onclick="pick(this)"><span class="dot"></span>Wait till after marriage</div><div class="opt sel" onclick="pick(this)"><span class="dot"></span>Either works</div></div>
          </div>
        </div>
        <div class="foot"><a class="btn primary block" href="#S-A9" onclick="MMKARMA.emit('ev-p4','complete',6,'Completed Love &amp; intimacy')">Save · back to profile</a></div>
```

Leave the other three `Save · back to profile` buttons (`S-P1`, `S-P2`, `S-P5`) alone — `USER_GAPS` records only Personality & lifestyle and Love & intimacy as blank, and crediting her for re-saving a section she already completed would put a false row in the ledger.

- [ ] **Step 5: Completeness — authoring the deck**

Find this exact text (`~:2245`):

```js
          function submit(){ sessionStorage.removeItem('mm_deck_used'); render(); }
```

Replace with:

```js
          function submit(){
            sessionStorage.removeItem('mm_deck_used');
            /* §6.2 completeness — authoring custom questions. Guarded on the
               deck being non-empty: submit() also runs on an empty deck, where
               it only clears the attempt budget, and crediting her for
               submitting nothing would be a lie the ledger then displays. */
            if(MMQ.deck().length) MMKARMA.emit('ev-deck','complete',5,'Submitted your question deck');
            render();
          }
```

- [ ] **Step 6: Completeness — correcting a wrong inference**

Find this exact text (`~:3761`):

```js
          function reject(p){
            const out = rejected();
            if(out.indexOf(p) === -1) out.push(p);
            sessionStorage.setItem('mm_conf_rejected', JSON.stringify(out));
            paint();
          }
```

Replace with:

```js
          function reject(p){
            const out = rejected();
            if(out.indexOf(p) === -1) out.push(p);
            sessionStorage.setItem('mm_conf_rejected', JSON.stringify(out));
            /* §6.2 names this one outright — "correcting wrong inferences on
               S-E6". One row however many probes she corrects; the emit id is
               fixed, so the second call is a no-op. */
            MMKARMA.emit('ev-conf-reject','complete',3,'Corrected what we inferred about you');
            paint();
          }
```

- [ ] **Step 7: Honest feedback — the post-date review**

Find this exact text inside `MMNUM.submit` (`~:3648`):

```js
          function submit(){
            if(!ready()) return false;
            sessionStorage.setItem('mm_numbers', JSON.stringify({
              exchanged, reason,
              note: (document.getElementById('numNote') || {}).value || ''
            }));
            return true;
          }
```

Replace with:

```js
          function submit(){
            if(!ready()) return false;
            sessionStorage.setItem('mm_numbers', JSON.stringify({
              exchanged, reason,
              note: (document.getElementById('numNote') || {}).value || ''
            }));
            /* §6.2 honest feedback. After the ready() gate, so an abandoned
               half-filled review earns nothing. */
            MMKARMA.emit('ev-review','feedback',5,'Answered the post-date review in full');
            return true;
          }
```

- [ ] **Step 8: Honest feedback — the decline that clears its gate**

Find this exact text inside `MMDECLINEDATE.submit` (`~:3950`):

```js
            warn().style.display = 'none';
            sessionStorage.setItem('mm_decline_reason',
              JSON.stringify({ text:t, at:Date.now() }));
            return true;
```

Replace with:

```js
            warn().style.display = 'none';
            sessionStorage.setItem('mm_decline_reason',
              JSON.stringify({ text:t, at:Date.now() }));
            /* §6.2 honest feedback. Below the reviewReason gate on purpose —
               mush and cruelty return early above, so only a reason that
               actually reaches David earns anything. The text she wrote is
               NOT passed to emit(); the ledger stores only this fixed label. */
            MMKARMA.emit('ev-decline','feedback',5,'Wrote a real reason when you passed on someone');
            return true;
```

- [ ] **Step 9: Run the assertion to verify it passes**

Navigate to `http://127.0.0.1:3300/index.html?v=12#S-B4`, evaluate Step 1's function.
Expected: `ok:true`, `rows: ['ev-conf-reject']`, `moved` greater than `base`.

- [ ] **Step 10: Walk two of the emits by hand**

Assertions can call functions the UI never reaches, so exercise two through the interface:

1. Navigate to `#S-P3`, click **Save · back to profile**. Navigate to `#S-B4`. Confirm a **Completed Personality & lifestyle · +6** row sits at the top of the ledger and the standing has moved.
2. Go back to `#S-P3` and click **Save** again. Return to `#S-B4`. Confirm there is still **exactly one** such row.
3. Navigate to `#S-E8`, type a reason that fails the gate (`nothing really nothing really nothing really nothing`), submit, and confirm it is blocked. Then check `#S-B4` — **no** `ev-decline` row may exist. Now write a real reason, submit, and confirm the row appears.

- [ ] **Step 11: Commit**

```bash
git add index.html
git commit -m "feat(karma): emit karma from the seven actions Tolu actually takes"
```

---

## Task 7: §6.6 — "3 people pursued you this month"

**Files:**
- Modify: `index.html` — add a host above `#b2list` (`~:2336`), add `renderPursued()` to the gallery IIFE, call it from `render()`, and demote the gap card's CTA in `renderGap()` (`~:2680`).

**Interfaces:**
- Consumes: `MMKARMA.pursued()`; the gallery's own `USER_GAPS` (`~:2626`).

- [ ] **Step 1: Write the browser assertion**

Navigate to `#S-B2` and evaluate:

```js
() => {
  const wrap = document.getElementById('b2pursued');
  if (!wrap) return { ok:false, reason:'#b2pursued host missing' };
  const card = wrap.querySelector('.card');
  if (!card) return { ok:false, reason:'no card rendered' };
  const n = MMKARMA.pursued().length;
  const head = card.querySelector('.eyebrow').textContent;
  const lines = card.querySelectorAll('.why li').length;
  const cta = card.querySelector('.btn');
  const standing = [...card.querySelectorAll('a')]
    .filter(a => a.getAttribute('href') === '#S-B4').length;
  // §6.6 forbids any language implying a gate.
  const banned = /unlock|eligible|locked|qualify|threshold/i.test(card.textContent);
  // The gap card must no longer carry a primary button.
  const gapBtns = document.querySelectorAll('#b2gap .btn.primary').length;
  return {
    ok: head.indexOf(String(n)) > -1 && lines === n &&
        !!cta && cta.getAttribute('href') === '#S-P3' &&
        standing === 1 && banned === false && gapBtns === 0,
    n, head, lines, cta: cta && cta.getAttribute('href'), standing, banned, gapBtns
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `http://127.0.0.1:3300/index.html?v=13#S-B2`, evaluate.
Expected: `{ ok:false, reason:'#b2pursued host missing' }`.

- [ ] **Step 3: Add the host**

Find this exact text (`~:2336`):

```html
          <div id="b2list" style="display:flex;flex-direction:column;gap:12px"></div>
```

Replace with:

```html
          <div id="b2pursued"></div>
          <div id="b2list" style="display:flex;flex-direction:column;gap:12px"></div>
```

- [ ] **Step 4: Add `renderPursued()`**

Find this exact text — the end of `renderGap()` and the declarations that follow it (`~:2683–2688`):

```js
      <div class="note" style="margin-top:12px">Optional, always. You're already
        matched — this only sharpens who we put in front of you next.</div>
    </div>`;
  }

  let undoTimer = null, pendingUndo = null, drawerOpen = true;
```

Replace with:

```js
      <div class="note" style="margin-top:12px">Optional, always. You're already
        matched — this only sharpens who we put in front of you next.</div>
    </div>`;
  }

  /* ---- spec 6.6 · the enrichment invitation ----
     Rewritten by the 2026-08-26 rollback: the original message ended "finish
     your profile to see who", which was an access claim. Nobody is excluded
     from async any more, so the ask is about match QUALITY instead. No
     "unlock", no "eligible", no progress bar — those all describe a rule that
     no longer exists. */
  function renderPursued(){
    const wrap = document.getElementById('b2pursued');
    if (!wrap) return;
    const people = MMKARMA.pursued();
    /* If nobody tried to pursue her the message does not appear at all. A card
       degrading to "0 people pursued you" would be worse than the gate the
       rollback removed. */
    if (!people.length){ wrap.innerHTML = ''; return; }
    const n = people.length;
    /* Each of them is asking about a section SHE has left blank — the band key
       resolves through USER_GAPS, so the ask can never point at something she
       has already answered. */
    const gapOf = b => USER_GAPS.filter(g => g.band === b)[0];
    const lines = people.map(p => {
      const g = gapOf(p.band);
      return `<li class="o"><span>${p.who} asked ${p.asked}
        <span class="src">[${g ? g.name : 'profile'}]</span></span></li>`;
    }).join('');
    const first = USER_GAPS[0];
    wrap.innerHTML = `<div class="card" style="margin-top:4px">
      <div class="eyebrow">✦ ${n} ${n === 1 ? 'person' : 'people'} pursued you this month</div>
      <ul class="why" style="margin-top:10px">${lines}</ul>
      <p class="muted" style="font-size:12.5px;line-height:1.5;margin:10px 0 0">
        We can't yet tell you which of them actually fits you — the things they were
        asking about are the things you haven't answered. Answer them and we can.</p>
      <a class="btn primary block" href="${first.screen}" style="margin-top:12px">Answer ${first.name} ›</a>
      <a class="link center" href="#S-B4" style="margin-top:8px;display:block">See your standing ›</a>
    </div>`;
  }

  let undoTimer = null, pendingUndo = null, drawerOpen = true;
```

- [ ] **Step 5: Call it from `render()`**

Find this exact text (added in Task 2):

```js
    renderDrawer();   // defined in Task 4
    renderGap();      // defined in Task 5
    renderProfile();  // defined in Task 6
    MMKARMA.paint();  // fills the [data-karma-of] slots the cards just declared
  }
```

Replace with:

```js
    renderPursued(); // spec 6.6
    renderDrawer();   // defined in Task 4
    renderGap();      // defined in Task 5
    renderProfile();  // defined in Task 6
    MMKARMA.paint();  // fills the [data-karma-of] slots the cards just declared
  }
```

- [ ] **Step 6: Demote the gap card's CTA**

The invitation and the gap card would otherwise put two primary buttons on one screen pointing at the same place. Find this exact text inside `renderGap()` (`~:2680`):

```js
      <a class="btn primary block" href="${first.screen}" style="margin-top:12px">Answer ${first.name} ›</a>
```

Replace with:

```js
      <a class="link center" href="${first.screen}" style="margin-top:12px;display:block">Answer ${first.name} ›</a>
```

The gap card keeps its per-band blocking evidence and its "~9 questions · about 2 minutes" line — that is the honest detail behind the invitation — and gives up only the button.

- [ ] **Step 7: Run the assertion to verify it passes**

Navigate to `http://127.0.0.1:3300/index.html?v=14#S-B2`, evaluate Step 1's function.
Expected: `ok:true`, `n:3`, `lines:3`, `cta:'#S-P3'`, `banned:false`, `gapBtns:0`.

- [ ] **Step 8: Prove the guard**

Evaluate:

```js
() => {
  const real = MMKARMA.pursued;
  MMKARMA.pursued = () => [];
  MMGAL.render();
  const empty = document.getElementById('b2pursued').innerHTML.trim();
  MMKARMA.pursued = real;
  MMGAL.render();
  const back = document.getElementById('b2pursued').querySelectorAll('.card').length;
  return { ok: empty === '' && back === 1, empty, back };
}
```

Expected: `ok:true`. §6.6's "if nobody tried to pursue them, the message does not appear" is a hard constraint, so it gets its own check rather than being assumed.

- [ ] **Step 9: Screenshot both viewports and measure overflow**

Screenshot `#S-B2` at 1280×800 and 390×844. `S-B2` now carries the pre-match note, the invitation, three cards, the drawer and the gap card. Run the overflow measurement, and confirm the screen has exactly **one** `.btn.primary` in the body.

- [ ] **Step 10: Commit**

```bash
git add index.html
git commit -m "feat(karma): add the 6.6 enrichment invitation and yield the gap card"
```

---

## Task 8: Full-flow verification sweep

**Files:** none — this task changes nothing unless it finds something.

Phase 5's memory of this repo is explicit: ten clean per-task reviews still hid three Critical defects, two of which had been recorded as verified. Per-task checks pass on the task's own surface and miss what the task did to its neighbours. This sweep exists to catch that.

- [ ] **Step 1: Screenshot every touched screen at both viewports**

At 1280×800 and 390×844, with a fresh cache-buster and a cleared `sessionStorage`:

`#S-B2`, `#S-B3`, `#S-B4`, `#S-C3`, `#S-F2A`, `#S-A9`, `#S-P3`, `#S-P4`, `#S-E5`, `#S-E6`, `#S-E8`, `#S-F3`.

Look at each one. A diff review passes clean while a screen is visibly broken — that is what this step is for.

- [ ] **Step 2: Walk the whole thing as a user**

With `sessionStorage` cleared, starting at `#S-B2`:

1. Confirm Samuel is first and David second, and that David's fit reads higher.
2. Open the invitation card's **See your standing ›** → lands on `S-B4` showing **●●●●○ Reliable** and five ledger rows, one of them **−8**.
3. Back to `#S-B2` → **Answer Personality & lifestyle ›** → `S-P3` → **Save** → `#S-A9` → **Your karma** row → `S-B4`. The new row is at the top and the standing has risen.
4. Open the dev panel, switch to **B · score**. Every karma surface becomes a number; the gallery order is unchanged.
5. Switch back to **A · traits**.

- [ ] **Step 3: Verify Prev/Next and the ☰ index**

Open the ☰ screen index. Confirm `S-B4 · Your karma` appears under **B · Matching** and navigates. Then step Prev/Next through `S-B3 → S-B4 → S-C1` and confirm the sequence reads correctly and nothing is skipped.

- [ ] **Step 4: Check the console on every screen**

Step through all twelve screens from Step 1 with the console open. Zero uncaught errors. `MMKARMA.paint()` runs on every `hashchange`, so a bad host attribute anywhere throws on every navigation, not just on the screen that owns it.

- [ ] **Step 5: Re-run every task assertion in one session**

Run the Step 1 assertion from Tasks 1, 2, 3, 4, 5 and 7 in a single page session, in that order, clearing `mm_karma` between them. Passing individually and failing together is the exact failure mode this sweep is for.

- [ ] **Step 6: Confirm the untouched flows still run**

Play `#S-D2` (the show) to a result, and `#S-F3` (the async board) to `#S-F4`. Both mount question sets from `MMQ`, and both are downstream of a head-script block this phase edited.

- [ ] **Step 7: Commit any fixes**

```bash
git add index.html
git commit -m "fix(karma): resolve full-flow sweep findings"
```

If the sweep found nothing, skip the commit and say so explicitly in the report rather than inventing one.

---

## Spec coverage

| Spec section | Task |
|---|---|
| §6.1 no leaderboard | Nothing is built — no ranking surface exists in any task |
| §6.2 four rewarded behaviours | Task 1 (`BEHAVIOURS`, the records), Task 6 (the seven emits) |
| §6.2 conduct only after a ruling | Deliberately absent; Global Constraints and Task 1's comments record why |
| §6.3 match visibility | Task 2 (the sort), Task 4 (the mirror line on `S-B4`) |
| §6.3 never announce a penalty | Task 1 (`traits()` positive-only), asserted in Tasks 1 and 2 |
| §6.4 Direction A / B | Task 1 (`tagHTML`/`headHTML`), Task 5 (the toggle) |
| §6.4 one shared dev surface | Task 5 — a third `.devrow`, not a new panel |
| §6.6 the invitation | Task 7 |
| §6.6 the number must be real | Task 7 Step 1 (count derived) and Step 8 (the empty guard) |
| §6.6 must not imply a gate | Task 7 Step 1's `banned` regex |
| §6.5 badges | Out of scope — no task, by decision |
| New screen `S-B4` | Task 4 |
| `FEED` registration | Task 4 Step 4 |
| Seeded ledger, no live penalties | Task 1 (`SEED`), Global Constraints |
