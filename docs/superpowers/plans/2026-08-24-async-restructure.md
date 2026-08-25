# Async Game Restructure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn the async game symmetric — both parties answer everything up front, each reacts to the other's answers, both are charged before play, a 72-hour window governs completion — gate the whole mode on knowing both people well enough, and render the mirrored-pairs reveal that Phase 1 built the data for.

**Architecture:** A `window.MMASYNC` module holds the shared async state: the 72-hour deadline, a single global countdown tick that paints every `[data-mm-countdown]` element, both sides' answers and verdicts, and the stored result. `S-F3` becomes two boards — your flip-slot verdicts on their answers driving *their* score, and a compact strip of their verdicts on your answers driving *yours* — with both gated at 70%. `S-F4` reads the stored result rather than hardcoding a pass.

**Tech Stack:** HTML + CSS + vanilla JavaScript, single file. Verification is browser-driven via the Playwright MCP tools.

**Spec:** `docs/superpowers/specs/2026-08-24-prompt2-decisions.md` — Cluster 2 (§2.1–2.6) plus §1.5, whose rendering was deferred from Phase 1 to this phase. Read it alongside this plan.

## Global Constraints

- **Single file.** All changes in `/mnt/e/TOOLMAKER/PYTHON/MakeaMoveUI_Mockup/index.html`. No new runtime files, no build step, no libraries.
- **Reuse existing components:** `.card`, `.row`, `.note`, `.opt`, `.btn`, `.field`, `.textarea`, `.tag`, `.chip`, `.av`, `.pbar`, `.body`, `.foot`, `.btnrow`, `.eyebrow`, `.muted`, `.big-num`, `.pill-live` and the `:root` custom properties.
- **Theme both variants.** `S-F1`/`S-F3`/`S-F4`/`S-F5` are `dark`; `S-F2`/`S-F2A`/`S-F6`/`S-G3` are `paper`. Any shared component must work on both.
- **Both parties are charged before play begins, and a failed gate refunds nothing** (§2.3). Every mention of a hold "released on a fail" is now wrong and must go.
- **The pursued is charged at the moment they consent** (§2.3), after seeing the pursuer's full profile and custom questions (§2.4).
- **72-hour window** (§2.5). On expiry the side that has not finished forfeits their charge; the side that completed gets a session credit back.
- **Both parties must reach 70%** — the async gate is now symmetric, exactly like the sync show.
- **Async eligibility** (§2.7): a user may neither pursue nor be pursued unless **3 attributes are Confirmed AND 12 questions are answered**. Checked at invite and consent, then **locked for that game**. **Sync and matching stay completely ungated** — async is what you unlock, never what you are locked out of.
- **Dimension weights come from `MMQ.DIMS`.** Do not redeclare them.
- **Question counts vary 10–15** (Phase 1, §1.2). Nothing may assume a fixed count.
- **Do not break:** `window.FSHOW`'s `{react, reset}` shape (the router calls `FSHOW.reset()` on entering `#S-F3`), or `MMQ`/`MMDECK`/`MMDEV`/`MMATTACH`/`MMCONF`.

## A deliberate non-change

`S-F3` is not in the `IMMERSIVE` set in `classify()`, so it renders as a centred form sheet on desktop. It arguably belongs there — it is a full-bleed interactive game screen. **Leave it alone.** Phase 1's scroll-and-cap fix for the board was tuned against the current archetype, and switching archetypes mid-phase risks re-breaking the layout this plan is already reworking. Revisit it in Phase 3, which rebuilds the board furniture anyway.

## Verification setup

`file://` is blocked by the Playwright MCP browser. Serve the project once, before Task 1:

```bash
cd /mnt/e/TOOLMAKER/PYTHON/MakeaMoveUI_Mockup && python3 -m http.server 3300
```

Assertions run at **1280×800** unless a step says otherwise, against `http://127.0.0.1:3300/index.html#<screen>`.

**Tooling gotchas carried from Phase 1 — all three cost real time there:**
- Never call `location.reload()` inside an evaluated function; it destroys the execution context and the evaluate fails instead of returning.
- Re-navigating to an identical URL serves a stale cached copy. Append a fresh cache-buster (`?v=2`, `?v=3`…) **before** the `#hash` every time you need new code.
- Any loop that plays a board must be `async` and `await` the delays — `freact()` advances the slot behind a 950ms `setTimeout`, so a synchronous loop spins and misleadingly hits its guard.

**Phase 1's hardest-won lesson:** diff review passed clean on eight tasks while `S-F3` was visibly broken on screen. This plan changes how many elements `S-F3` draws, which is exactly the condition that caused it. Task 8 is a mandatory visual sweep at two viewports; do not treat it as optional.

## File Structure

One file, six insertion points:

- **New `<script id="mmasync">`** immediately after the existing `<script id="mmq">` block, before `<div class="screenwrap">`. It depends on `MMQ` and must parse after it.
- **CSS block** (main `<style>`): append an "async" section for `.mirrorrow` and `.minitok`.
- **`S-F1`** (~2581) and **`S-F2`** (~2598): copy rewrites for the new charge model.
- **New `S-F2A`** after `S-F2`: the pursued's consent-and-charge screen.
- **`S-F3`** (~2613–2757): the two-board rebuild, in its own scoped `<style>` and inline script.
- **`S-F4`** (~2759–2772): dynamic result plus the mirrored-pairs block.
- **New `S-F5`** (72h expiry) and **`S-F6`** (declined + reasons) after `S-F4`; **`S-G3`** (~2805) gains the async lifecycle.

---

### Task 1: The MMASYNC state module

**Files:**
- Modify: `index.html` — insert a new `<script id="mmasync">` immediately after the closing `</script>` of `<script id="mmq">` and before `<div class="screenwrap" id="screenwrap">`.

**Interfaces:**
- Consumes: `MMQ.answers()`, `MMQ.DEMO.them`, `MMQ.DEMO.verdict`, `MMQ.buildSet()`, `MMQ.DIMS`.
- Produces on `window`: `MMASYNC` with `WINDOW_MS`, `deadline()`, `remaining()`, `fmt()`, `expire()`, `resetWindow()`, `myAnswer(qid)`, `theirAnswer(qid)`, `theirVerdict(qid)`, `saveResult(o)`, `result()`, `clearResult()`, `scores()`. Every later task calls these by exactly these names.

- [ ] **Step 1: Write the browser assertion**

Run this first, before writing any code, so you see it fail:

```js
() => {
  if (!window.MMASYNC) return { ok:false, reason:'MMASYNC not defined' };
  const A = window.MMASYNC;
  A.resetWindow(); A.clearResult();

  const r = A.remaining();
  const fresh = r.h === 71 || r.h === 72;         // 72h minus a few ms
  const fmtOk = /^\d{2}:\d{2}:\d{2}$/.test(A.fmt());

  // answers resolve for every question in the set, with fallbacks
  const set = MMQ.buildSet();
  const badMine  = set.filter(q => { const a = A.myAnswer(q.id);
                     return !a || typeof a.opt !== 'number' || !q.opts[a.opt] ||
                            typeof a.answered !== 'boolean'; });
  // the fallback must be flagged, not silently presented as an answer
  const known = MMQ.answers().map(a => a.qid);
  const flagWrong = set.filter(q =>
    A.myAnswer(q.id).answered !== (known.indexOf(q.id) !== -1));
  const badTheirs = set.filter(q => { const a = A.theirAnswer(q.id);
                     return !a || typeof a.opt !== 'number' || !q.opts[a.opt]; });
  const badVerdict = set.filter(q => ['move','stay'].indexOf(A.theirVerdict(q.id)) === -1);

  // scores() derives both sides from weights
  const allMove = {}; set.forEach(q => { allMove[q.id] = 'move'; });
  const sAll  = A.scores(allMove);
  const allStay = {}; set.forEach(q => { allStay[q.id] = 'stay'; });
  const sNone = A.scores(allStay);

  // result round-trips
  A.saveResult({ you:81, them:74, verdicts:allMove });
  const back = A.result();

  // expire() zeroes the window
  A.expire();
  const afterExpire = A.remaining();
  A.resetWindow(); A.clearResult();

  return {
    ok: fresh && fmtOk &&
        badMine.length===0 && badTheirs.length===0 && badVerdict.length===0 &&
        flagWrong.length===0 &&
        sAll.them===100 && sNone.them===0 &&
        sAll.you === sNone.you &&              // your score is their fixed verdicts
        back && back.you===81 && back.them===74 &&
        afterExpire.expired === true && afterExpire.ms === 0,
    hours:r.h, fmt:A.fmt(),
    badMine:badMine.map(q=>q.id), badTheirs:badTheirs.map(q=>q.id),
    flagWrong:flagWrong.map(q=>q.id),
    scoresAllMove:sAll, scoresAllStay:sNone,
    roundTrip:back, expired:afterExpire.expired
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `http://127.0.0.1:3300/index.html#S-F3`, resize 1280×800, evaluate the above.
Expected: `{ ok:false, reason:'MMASYNC not defined' }`.

- [ ] **Step 3: Insert the module**

Insert immediately after the `</script>` that closes `<script id="mmq">`:

```html
<script id="mmasync">
/* ===== MMASYNC · shared async-game state (spec Cluster 2) =====
   The async game is symmetric: both parties answer everything before play
   (§2.1), then each reacts to the other's answers. Your verdicts drive THEIR
   score; their verdicts drive YOURS. Both need 70%.

   Declared after MMQ because it reads the question set and the fixtures, and
   before #screenwrap because S-F3's inline script consumes it at parse time. */
(function(){
  const WINDOW_MS = 72*60*60*1000;          // §2.5 — the whole game fits in 72h
  const DL = 'mm_async_deadline', RES = 'mm_async_result';

  function deadline(){
    let d = +sessionStorage.getItem(DL);
    if(!d){ d = Date.now() + WINDOW_MS; sessionStorage.setItem(DL, d); }
    return d;
  }
  function resetWindow(){ sessionStorage.removeItem(DL); deadline(); }
  function expire(){ sessionStorage.setItem(DL, Date.now()); }

  function remaining(){
    const ms = Math.max(0, deadline() - Date.now());
    return { ms, expired: ms === 0,
             h: Math.floor(ms/3600000),
             m: Math.floor(ms/60000) % 60,
             s: Math.floor(ms/1000) % 60 };
  }
  const p2 = n => String(n).padStart(2,'0');
  function fmt(){ const r = remaining(); return p2(r.h)+':'+p2(r.m)+':'+p2(r.s); }

  /* ONE interval paints every countdown on the page, so navigating between
     screens can never stack timers. Elements opt in with data-mm-countdown. */
  function paintAll(){
    const t = fmt();
    document.querySelectorAll('[data-mm-countdown]').forEach(el => { el.textContent = t; });
  }
  setInterval(paintAll, 1000);
  document.addEventListener('DOMContentLoaded', paintAll);
  window.addEventListener('hashchange', paintAll);

  /* Both sides answered every question before play began (§2.1). Demo
     fixtures are sparse, so every lookup falls back to a valid option. */
  /* `answered` distinguishes a real answer from a fallback. A profile fills in
     progressively (§2.7), so a user can legitimately reach a game with nothing
     recorded for some questions — and the result screen must say so rather
     than implying an answer they never gave. */
  function myAnswer(qid){
    const a = MMQ.answers().find(x => x.qid === qid);
    return a ? { qid, opt:a.opt, why:a.why || '', answered:true }
             : { qid, opt:0, why:'', answered:false };
  }
  function theirAnswer(qid){
    return Object.assign({ qid }, MMQ.DEMO.them[qid] || { opt:0, why:'' });
  }
  function theirVerdict(qid){
    return MMQ.DEMO.verdict[qid] === 'stay' ? 'stay' : 'move';
  }

  /* myVerdicts maps question id -> 'move'|'stay' (what YOU decided about
     THEM). Their score is what you gave them; yours is what they gave you. */
  function scores(myVerdicts){
    const set = MMQ.buildSet();
    const totw = set.reduce((s,q) => s + MMQ.DIMS[q.dim], 0);
    let them = 0, you = 0;
    set.forEach(q => {
      const w = MMQ.DIMS[q.dim];
      if((myVerdicts||{})[q.id] === 'move') them += w;
      if(theirVerdict(q.id) === 'move')     you  += w;
    });
    return { you: Math.round(you/totw*100), them: Math.round(them/totw*100) };
  }

  function saveResult(o){ sessionStorage.setItem(RES, JSON.stringify(o)); }
  function result(){ try { return JSON.parse(sessionStorage.getItem(RES)); }
                     catch(e){ return null; } }
  function clearResult(){ sessionStorage.removeItem(RES); }

  window.MMASYNC = { WINDOW_MS, deadline, resetWindow, expire, remaining, fmt,
                     myAnswer, theirAnswer, theirVerdict, scores,
                     saveResult, result, clearResult, paintAll };
})();
</script>
```

- [ ] **Step 4: Run the assertion to verify it passes**

Re-evaluate Step 1 (remember the cache-buster).
Expected: `ok: true`, `hours` 71 or 72, `fmt` matching `HH:MM:SS`, empty `badMine`/`badTheirs`, `scoresAllMove.them === 100`, `scoresAllStay.them === 0`, and `scoresAllMove.you === scoresAllStay.you` (your score depends only on *their* fixed verdicts, never on what you do).

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(async): add MMASYNC state module — 72h window, both-sides fixtures, scoring"
```

---

### Task 2: Rewrite the S-F1 and S-F2 charge model

**Files:**
- Modify: `index.html` — `S-F1` (~2581–2596) and `S-F2` (~2598–2611).

**Interfaces:**
- Consumes: nothing new. `S-F2` keeps its `<div class="card" id="attachF2"></div>`, which `MMATTACH` fills.
- Produces: `S-F2`'s primary CTA now points at `#S-F2A` (the consent screen built in Task 3) rather than `#S-F3`.

- [ ] **Step 1: Write the browser assertion**

```js
() => {
  const f1 = document.getElementById('S-F1').textContent.replace(/\s+/g,' ');
  const f2 = document.getElementById('S-F2').textContent.replace(/\s+/g,' ');
  const cta = document.querySelector('#S-F2 .foot a.btn');
  const dead = /asymmetric|soft-held|soft-hold|released|only if you pass|pursued's weights/i;
  return {
    ok: !dead.test(f1) && !dead.test(f2) &&
        /both/i.test(f1) && /70%/.test(f1) &&
        /before the game|before play/i.test(f1 + f2) &&
        /no refund|not refunded|nothing is refunded/i.test(f1 + f2) &&
        cta && cta.getAttribute('href') === '#S-F2A' &&
        !!document.getElementById('attachF2'),
    f1Dead: (f1.match(dead)||[])[0] || null,
    f2Dead: (f2.match(dead)||[])[0] || null,
    cta: cta && cta.getAttribute('href'),
    attachPresent: !!document.getElementById('attachF2')
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `#S-F1`, evaluate. Expected: `ok:false`, with `f1Dead` reporting `asymmetric` — the current copy describes the old model.

- [ ] **Step 3: Replace the S-F1 body**

Replace the contents of `S-F1`'s `<div class="body">` with:

```html
          <span class="chip" style="align-self:center">Unlocked at 200 users</span>
          <h2 class="scrn" style="font-size:32px;max-width:14ch;margin:0 auto">Can't both be live? Play <span style="color:var(--move)">async.</span></h2>
          <p class="muted" style="max-width:32ch;margin:0 auto;line-height:1.55">Same game, on your own time. You both answer everything up front, then take turns reacting to each other's answers — whenever you each get a moment.</p>
          <div class="card" style="text-align:left">
            <div class="eyebrow" style="color:var(--bulb)">How async works</div>
            <ul class="muted" style="font-size:12.5px;margin:8px 0 0;padding-left:18px;line-height:1.7">
              <li>You both answer the full set <b>before</b> play starts.</li>
              <li>Your Moves build <b>their</b> score; theirs build <b>yours</b>. Both need <b>70%</b>.</li>
              <li><b>Both of you pay before the game begins</b> — they're charged the moment they consent.</li>
              <li>You have <b>72 hours</b> to finish once it starts.</li>
            </ul>
          </div>
          <div class="note" style="text-align:left">The fee buys the game, not the outcome — <b>a result under the gate is not refunded</b>. Whoever leaves a game unfinished past 72 hours forfeits; the other side gets a credit back.</div>
          <a class="btn move block" href="#S-F2">Start an async game</a>
```

- [ ] **Step 4: Replace the S-F2 body and CTA**

Replace the contents of `S-F2`'s `<div class="body">` with:

```html
          <div class="card" style="display:flex;gap:12px;align-items:center"><div class="av g3 s52">S</div><div><div style="font-weight:800">Samuel, 44</div><div class="muted" style="font-size:12px">Architect · wants kids · London</div></div></div>
          <div class="note info">You're the <b>pursuer</b>. You'll both answer the full set, then react to each other's answers — and you each need <b>70%</b> to pass.</div>
          <div class="card" id="attachF2"></div>
          <div class="card">
            <div style="display:flex;justify-content:space-between"><span>Your fee, charged now</span><b>£19.00</b></div>
            <div class="muted" style="font-size:12px;margin-top:4px">Free during launch — covered by your free credit.</div>
            <div style="display:flex;justify-content:space-between;margin-top:10px"><span class="muted" style="font-size:12.5px">Samuel's fee</span><span class="tag warn">On his consent</span></div>
          </div>
          <div class="note">Both fees are taken before the game begins. <b>Neither is refunded if you don't clear the gate</b> — the fee buys the game, not the result.</div>
```

Then change `S-F2`'s footer CTA to point at the consent screen:

```html
        <div class="foot"><a class="btn move block" href="#S-F2A">Send · Samuel consents next</a></div>
```

- [ ] **Step 5: Run the assertion to verify it passes**

Re-evaluate Step 1. Expected: `ok: true`, `f1Dead: null`, `f2Dead: null`, `cta: '#S-F2A'`, `attachPresent: true`.

The assertion will still fail on `cta` until Task 3 creates `S-F2A` — that is expected and fine; the href is correct now. Confirm `cta` reads `'#S-F2A'` even though the target does not exist yet.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(async): rewrite S-F1/S-F2 for the symmetric, charge-up-front model"
```

---

### Task 3: The pursued's consent-and-charge screen (S-F2A)

**Files:**
- Modify: `index.html` — insert a new `<section>` immediately after the closing `</section>` of `S-F2`.

**Interfaces:**
- Consumes: `MMQ.deck()`, `MMQ.MAX_DECK`, `MMASYNC` (for the countdown attribute).
- Produces: screen `S-F2A`. Its accept CTA goes to `#S-F3`; its decline link goes to `#S-F6` (built in Task 4).

- [ ] **Step 1: Write the browser assertion**

```js
() => {
  const s = document.getElementById('S-F2A');
  if (!s) return { ok:false, reason:'S-F2A not found' };
  const txt = s.textContent.replace(/\s+/g,' ');
  const accept = s.querySelector('.foot a.btn, .foot button.btn');
  const decline = s.querySelector('.foot a.link, .foot a[href="#S-F6"]');
  const countdown = s.querySelector('[data-mm-countdown]');
  const qs = s.querySelectorAll('[data-pursuer-q]');
  return {
    ok: !!s && s.classList.contains('paper') &&
        s.getAttribute('data-group') === 'F · Async game' &&
        !!s.getAttribute('data-title') &&
        /charged/i.test(txt) && /72 hours|72h/i.test(txt) &&
        qs.length >= 1 &&                       // pursuer's questions shown BEFORE paying
        !!countdown &&
        accept && accept.getAttribute('href') === '#S-F3' &&
        decline && decline.getAttribute('href') === '#S-F6',
    isPaper: s.classList.contains('paper'),
    questionsShown: qs.length,
    hasCountdown: !!countdown,
    accept: accept && accept.getAttribute('href'),
    decline: decline && decline.getAttribute('href')
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `#S-F2`, evaluate. Expected: `{ ok:false, reason:'S-F2A not found' }`.

- [ ] **Step 3: Insert the screen**

Insert immediately after `S-F2`'s closing `</section>`:

```html
      <section class="screen paper" id="S-F2A" data-group="F · Async game" data-title="Async consent + charge (pursued)">
        <div class="pbar"><span class="brand" style="font-size:16px">Make<span class="a">a</span>Move</span><span class="tag warn">Samuel's view</span></div>
        <div class="body">
          <div class="note info"><svg class="icon"><use href="#i-hourglass"/></svg> Tolu wants to play an async game with you. Once you accept, you both have <b><span data-mm-countdown>72:00:00</span></b> to finish.</div>

          <div class="card" style="display:flex;gap:14px;align-items:center">
            <div class="av g2 s72">T</div>
            <div><div style="font-weight:800;font-size:18px">Tolu, 37</div><div class="muted" style="font-size:13px">Project manager · 2 kids · London</div><div style="margin-top:6px"><span class="tag pass">Family fit</span> <span class="tag">AA</span></div></div>
          </div>

          <div class="card">
            <div class="eyebrow" style="color:var(--move)">What Tolu will ask you</div>
            <div class="muted" style="font-size:11.5px;margin-top:4px">You see her questions before you decide — not after.</div>
            <div id="pursuerQs" style="margin-top:10px;display:flex;flex-direction:column;gap:6px"></div>
          </div>

          <div class="card">
            <div style="display:flex;justify-content:space-between"><span>Your fee, charged on accept</span><b>£19.00</b></div>
            <div class="muted" style="font-size:12px;margin-top:4px">Free during launch — covered by your free credit.</div>
          </div>
          <div class="note">Accepting charges you now. <b>Neither fee is refunded if the game ends under the gate</b> — you're paying for the game, not the result. Leave it unfinished past 72 hours and you forfeit; finish it and you're covered either way.</div>
        </div>
        <div class="foot">
          <a class="btn move block" href="#S-F3">Accept &amp; pay · start playing</a>
          <a class="link center" href="#S-F6">No thanks — decline</a>
        </div>
        <script>
        (function(){
          /* The pursuer's custom questions are shown BEFORE payment (§2.4).
             Falls back to a representative set when the deck is empty, so the
             screen never renders as a blank promise. */
          function paint(){
            const host = document.getElementById('pursuerQs');
            if(!host) return;
            const deck = MMQ.deck().filter(q => q.src !== 'bank');
            const shown = deck.length ? deck.map(q => q.text)
              : ['Would you move abroad for the right person?',
                 'How do you handle a disagreement that lasts more than a day?'];
            host.innerHTML = shown.map(t =>
              '<div class="t2" data-pursuer-q>· ' + t + '</div>').join('')
              + '<div class="muted" style="font-size:11px;margin-top:6px">'
              + 'Plus the standard set — ' + MMQ.buildSet().length + ' questions in total.</div>';
          }
          window.addEventListener('hashchange', paint);
          paint();
        })();
        </script>
      </section>
```

- [ ] **Step 4: Run the assertion to verify it passes**

Re-evaluate Step 1. Expected: `ok: true`, `questionsShown` ≥ 1, `hasCountdown: true`, `accept: '#S-F3'`, `decline: '#S-F6'`.

`decline` points at a screen Task 4 creates; the href is correct now.

- [ ] **Step 5: Screenshot**

Navigate to `#S-F2A` at 1280×800 and again at 390×844. The pursuer's questions must be visible above the fee card on both — the whole point of §2.4 is that they see what they're paying for first.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(async): add S-F2A pursued consent screen with pre-payment disclosure"
```

---

### Task 4: The declined screen with reasons (S-F6)

**Files:**
- Modify: `index.html` — insert a new `<section>` after the closing `</section>` of `S-F4`.

**Interfaces:**
- Consumes: nothing.
- Produces on `window`: `MMDECLINE.pick(el)`, `MMDECLINE.submit()`. Screen `S-F6`.

- [ ] **Step 1: Write the browser assertion**

```js
() => {
  const s = document.getElementById('S-F6');
  if (!s) return { ok:false, reason:'S-F6 not found' };
  if (!window.MMDECLINE) return { ok:false, reason:'MMDECLINE not defined' };
  const opts = s.querySelectorAll('.opt[data-reason]');
  const free = s.querySelector('textarea');
  const submit = s.querySelector('.foot .btn');

  sessionStorage.removeItem('mm_decline');
  const blocked = submit.classList.contains('disabled') || submit.getAttribute('aria-disabled') === 'true';
  MMDECLINE.pick(opts[1]);
  const selected = s.querySelectorAll('.opt.sel[data-reason]').length;
  const unblocked = !submit.classList.contains('disabled');
  MMDECLINE.submit();
  const saved = JSON.parse(sessionStorage.getItem('mm_decline') || 'null');
  sessionStorage.removeItem('mm_decline');

  return {
    ok: opts.length >= 4 && !!free &&
        blocked && selected === 1 && unblocked &&
        saved && saved.reason === opts[1].getAttribute('data-reason') &&
        'note' in saved,
    optionCount: opts.length, hasFreeText: !!free,
    blockedBeforePick: blocked, selectedAfterPick: selected,
    saved
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `#S-F4`, evaluate. Expected: `{ ok:false, reason:'S-F6 not found' }`.

- [ ] **Step 3: Insert the screen**

Insert after `S-F4`'s closing `</section>`:

```html
      <section class="screen paper" id="S-F6" data-group="F · Async game" data-title="⚠ Async declined · reason">
        <div class="pbar"><a class="back" href="#S-F2A">‹</a><h1>Before you go</h1></div>
        <div class="body">
          <div class="note">Nothing has been charged. Telling us why takes a second and genuinely changes who we put in front of you next — this goes to your profile, not to Tolu.</div>
          <label class="muted" style="font-size:12px;font-weight:700">Why are you passing on this one?</label>
          <div style="display:flex;flex-direction:column;gap:8px">
            <div class="opt" data-reason="not-attracted" onclick="MMDECLINE.pick(this)"><span class="dot"></span>Not attracted to them</div>
            <div class="opt" data-reason="life-stage" onclick="MMDECLINE.pick(this)"><span class="dot"></span>Wrong life stage for me</div>
            <div class="opt" data-reason="distance" onclick="MMDECLINE.pick(this)"><span class="dot"></span>Too far away</div>
            <div class="opt" data-reason="values" onclick="MMDECLINE.pick(this)"><span class="dot"></span>Values or faith don't line up</div>
            <div class="opt" data-reason="not-ready" onclick="MMDECLINE.pick(this)"><span class="dot"></span>Not ready to play right now</div>
            <div class="opt" data-reason="other" onclick="MMDECLINE.pick(this)"><span class="dot"></span>Something else</div>
          </div>
          <div class="field"><label>Anything to add? (optional)</label><textarea class="textarea" id="declineNote" placeholder="Only we see this."></textarea></div>
        </div>
        <div class="foot"><a class="btn move block disabled" id="declineSubmit" href="#S-B2" onclick="return MMDECLINE.submit()">Submit &amp; go back to matches</a></div>
        <script>
        (function(){
          /* §2.6 — MECE options with an optional free line. Structured options
             are what the pre-matching system can consume; the free line catches
             what the options miss. Goes to the rejector's profile, never to the
             other party. */
          let chosen = null;
          function pick(el){
            if(!el) return;
            document.querySelectorAll('#S-F6 .opt[data-reason]')
              .forEach(o => o.classList.remove('sel'));
            el.classList.add('sel');
            chosen = el.getAttribute('data-reason');
            document.getElementById('declineSubmit').classList.remove('disabled');
          }
          function submit(){
            if(!chosen) return false;          // a reason is required
            const note = (document.getElementById('declineNote')||{}).value || '';
            sessionStorage.setItem('mm_decline', JSON.stringify({ reason:chosen, note }));
            return true;
          }
          window.MMDECLINE = { pick, submit };
        })();
        </script>
      </section>
```

- [ ] **Step 4: Add the disabled-button style**

Append to the main `<style>` block:

```css
/* a CTA that is deliberately inert until a required choice is made */
.btn.disabled{opacity:.45; pointer-events:none}
```

- [ ] **Step 5: Run the assertion to verify it passes**

Re-evaluate Step 1. Expected: `ok: true`, `optionCount: 6`, `blockedBeforePick: true`, `selectedAfterPick: 1`, and `saved` holding both `reason` and `note`.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(async): add S-F6 decline screen collecting structured reasons"
```

---

### Task 5: Rebuild S-F3 as two boards

**Files:**
- Modify: `index.html` — `S-F3`'s scoped `<style>`, its markup between `<div class="fhead">` and `<div class="actf">`, and its entire inline `<script>` (~2613–2757).

**Interfaces:**
- Consumes: `MMASYNC.scores(myVerdicts)`, `MMASYNC.theirVerdict(qid)`, `MMASYNC.theirAnswer(qid)`, `MMASYNC.saveResult(o)`, `MMASYNC.resetWindow()`, `MMQ.buildSet()`, `MMQ.DIMS`, `MMQ.answerCard(q, ans, label)`.
- Produces: `window.FSHOW` keeping exactly `{react, reset}`. On completion writes `MMASYNC.saveResult({you, them, verdicts})` and routes to `#S-F4`.

- [ ] **Step 1: Write the browser assertion**

```js
async () => {
  const sleep = ms => new Promise(r => setTimeout(r, ms));
  sessionStorage.removeItem('mm_deck');
  MMQ.setModel('A'); MMASYNC.clearResult(); MMASYNC.resetWindow();
  location.hash = 'S-F3'; FSHOW.reset();
  await sleep(60);

  const slots = document.querySelectorAll('#S-F3 .slot').length;
  const mini  = document.querySelectorAll('#S-F3 .minitok').length;
  const hasBoth = !!document.getElementById('fpctThem') && !!document.getElementById('fpctYou');
  const hasCountdown = !!document.querySelector('#S-F3 [data-mm-countdown]');

  // play the whole board with Move every time
  let guard = 0;
  while (location.hash.replace('#','') === 'S-F3' && guard++ < 40) {
    const btn = document.querySelector('#S-F3 .actf .btn.move');
    if (btn) { FSHOW.react('move'); await sleep(1000); } else { await sleep(300); }
  }
  const res = MMASYNC.result();
  const lit = document.querySelectorAll('#S-F3 .minitok.on').length;

  return {
    ok: slots === 10 && mini === 10 && hasBoth && hasCountdown &&
        res && res.them === 100 &&
        typeof res.you === 'number' &&
        Object.keys(res.verdicts).length === 10 &&
        location.hash === '#S-F4' && guard < 40,
    slots, mini, hasBoth, hasCountdown, lit, res, landed: location.hash, guard
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `#S-F3`, evaluate.
Expected: `ok:false` — `mini: 0` and `hasBoth: false`, because the second board and the two-up scorebar do not exist yet.

- [ ] **Step 3: Replace the scoped CSS additions**

In `S-F3`'s `<style>` block, replace the existing `#S-F3 .scorebar` rules (the block from `#S-F3 .scorebar{` through the `.pctf` rule) with:

```css
          /* Two scores now — theirs from your Moves, yours from theirs (§2.1) */
          #S-F3 .scorebar{flex:0 0 auto; margin:4px 14px 0; display:flex; align-items:stretch; gap:8px}
          #S-F3 .half{flex:1; display:flex; align-items:center; gap:8px; padding:8px 12px;
            background:linear-gradient(180deg,var(--stage-700),var(--stage-800)); clip-path:var(--ticket)}
          #S-F3 .half .nm{font-weight:800; font-size:12px; line-height:1.2}
          #S-F3 .half .sub2{font-size:9.5px; color:var(--ink-faint)}
          #S-F3 .half .pctf{font-family:var(--display); font-weight:900; font-size:22px; color:var(--bulb);
            text-shadow:0 0 16px rgba(255,182,39,.45); margin-left:auto}
          #S-F3 .half.pass .pctf{color:var(--pass); text-shadow:0 0 16px rgba(46,204,113,.4)}
          #S-F3 .gateline{flex:0 0 auto; text-align:center; font-size:9.5px; letter-spacing:.14em;
            text-transform:uppercase; color:var(--ink-faint); margin:5px 0 0}
          /* their verdicts on YOUR answers, arriving as you play (§2.2) */
          #S-F3 .mini{flex:0 0 auto; display:flex; gap:4px; justify-content:center;
            padding:6px 14px 0; flex-wrap:wrap}
          #S-F3 .minitok{width:16px; height:16px; border-radius:5px; background:rgba(255,255,255,.07);
            border:1px solid rgba(255,255,255,.1); display:flex; align-items:center; justify-content:center;
            font-size:9px; color:var(--ink-faint); transition:.35s var(--ease-show)}
          #S-F3 .minitok.on.move{background:var(--move); border-color:var(--move); color:#fff}
          #S-F3 .minitok.on.stay{background:rgba(255,255,255,.14); color:var(--ink-dim)}
          /* the board shrinks a little to make room for the second score row */
          #S-F3 .board{max-height:32vh}
          #S-F3 .slots{max-height:calc(32vh - 30px)}
```

- [ ] **Step 4: Replace the head and scorebar markup**

Replace the `<div class="fhead">…</div>` and `<div class="scorebar">…</div>` blocks with:

```html
        <div class="fhead">
          <span class="tag" style="background:rgba(255,255,255,.08);color:var(--ink-dim)">Async · both must clear 70%</span>
          <span class="rec"><span class="rd"></span><span data-mm-countdown>72:00:00</span> left</span>
        </div>
        <div class="scorebar">
          <div class="half" id="halfThem">
            <div class="av g3 s28">S</div>
            <div><div class="nm">Samuel</div><div class="sub2">your Moves</div></div>
            <div class="pctf" id="fpctThem">0%</div>
          </div>
          <div class="half" id="halfYou">
            <div class="av g2 s28">T</div>
            <div><div class="nm">You</div><div class="sub2">his Moves</div></div>
            <div class="pctf" id="fpctYou">0%</div>
          </div>
        </div>
        <div class="gateline">both need 70% to pass</div>
        <div class="mini" id="fmini"></div>
```

- [ ] **Step 5: Replace the inline script**

Replace the entire contents of `S-F3`'s `<script>` (the whole IIFE) with:

```js
        (function(){
          /* Symmetric async (§2.1): you react to Samuel's recorded answers,
             building HIS score; his recorded verdicts on YOUR answers build
             yours, revealed one per turn as you play (§2.2). Both need 70%. */
          let QF, TOTW;
          function fbuild(){
            QF = MMQ.buildSet().map(q => ({
              q, w: MMQ.DIMS[q.dim], ans: MMASYNC.theirAnswer(q.id)
            }));
            TOTW = QF.reduce((s,t) => s + t.w, 0);
          }
          function fslots(){
            document.getElementById('fslots').innerHTML = QF.map((t,n) =>
              '<div class="slot"><div class="in">'
              + '<div class="face front"><b>'+(n+1)+'</b></div>'
              + '<div class="face back"></div></div></div>').join('');
            document.getElementById('fmini').innerHTML = QF.map((t,n) =>
              '<div class="minitok" data-i="'+n+'">'+(n+1)+'</div>').join('');
          }
          let fi=0, gotThem=0, gotYou=0, verdicts={};
          const $=id=>document.getElementById(id);
          function ftween(el,val){ const from=(el._v==null)?0:el._v; el._v=val;
            if(from===val){ el.textContent=val+'%'; return; }
            const t0=performance.now();
            (function step(n){ const p=Math.min(1,(n-t0)/500);
              el.textContent=Math.round(from+(val-from)*(1-Math.pow(1-p,3)))+'%';
              if(p<1) requestAnimationFrame(step); })(t0);
          }
          function paint(){
            const them = Math.round(gotThem/TOTW*100), you = Math.round(gotYou/TOTW*100);
            ftween($('fpctThem'), them); ftween($('fpctYou'), you);
            $('halfThem').classList.toggle('pass', them >= 70);
            $('halfYou').classList.toggle('pass', you >= 70);
          }
          function frender(){
            if(fi>=QF.length) return ffinish();
            const t=QF[fi];
            $('fstage').innerHTML='<span class="chip" style="animation:rise .4s var(--ease-show) both">'
              + t.q.dim + ' · slot ' + (fi+1) + ' of ' + QF.length + '</span>'
              + '<div class="qbar">' + t.q.text + '</div>'
              + MMQ.answerCard(t.q, t.ans, 'Samuel answered (before play)')
              + '<div class="wnote">Weight ' + t.w.toFixed(1) + ' — your call moves <b>his</b> score.</div>';
            $('fact').innerHTML='<button class="btn ghost" onclick="FSHOW.react(\'stay\')">Stay<span class="sub">not feeling it</span></button>'
              +'<button class="btn move" onclick="FSHOW.react(\'move\')"><svg class="icon"><use href="#i-heart"/></svg> Move<span class="sub">I want this</span></button>';
            const cur = document.querySelectorAll('#S-F3 .slot')[fi];
            if(cur && cur.scrollIntoView) cur.scrollIntoView({block:'nearest'});
          }
          function freact(kind){
            const t=QF[fi];
            verdicts[t.q.id] = kind;
            const slot=document.querySelectorAll('#S-F3 .slot')[fi];
            const back=slot.querySelector('.face.back');
            back.innerHTML = kind==='move'
              ? '<span><span class="hh">♥</span>'+t.q.dim+'</span><span class="pts">'+Math.round(t.w*10)+'</span>'
              : '<span style="letter-spacing:.12em">'+t.q.dim+' · stay</span><span class="pts">—</span>';
            slot.classList.add('flip', kind);
            if(kind==='move') gotThem += t.w;

            /* his verdict on YOUR answer to this question lands at the same
               beat — that is the reciprocity §2.2 wants visible during play */
            const hv = MMASYNC.theirVerdict(t.q.id);
            const tok = document.querySelector('#S-F3 .minitok[data-i="'+fi+'"]');
            if(tok){ tok.classList.add('on', hv); tok.textContent = hv==='move' ? '♥' : '·'; }
            if(hv==='move') gotYou += t.w;

            paint();
            $('fact').innerHTML='<div class="wait">Slot '+(fi+1)+' — you said '
              +(kind==='move'?'<b style="color:var(--move)">Move</b>':'<b>Stay</b>')
              +'; Samuel said '+(hv==='move'?'<b style="color:var(--move)">Move</b>':'<b>Stay</b>')+' on yours</div>';
            setTimeout(()=>{ fi++; frender(); }, 950);
          }
          function ffinish(){
            /* One canonical derivation for the recorded result — the running
               gotThem/gotYou totals drive the live display only, so the number
               shown and the number stored cannot drift apart. */
            const s = MMASYNC.scores(verdicts);
            const them = s.them, you = s.you;
            const pass = them>=70 && you>=70;
            MMASYNC.saveResult({ you, them, verdicts });
            $('fboard').classList.add('done');
            $('fstage').innerHTML='<div class="qbar" style="text-align:center">Board complete.</div>'
              +'<div class="wnote">'+(pass
                 ? 'You both cleared 70% — taking you to the result…'
                 : 'One of you came in under the gate — here\'s where it landed…')+'</div>';
            $('fact').innerHTML='<div class="wait">Samuel '+them+'% · you '+you+'% · gate 70%</div>';
            setTimeout(()=>{ location.hash='S-F4'; }, 1700);
          }
          window.FSHOW={react:freact, reset(){
            fi=0; gotThem=0; gotYou=0; verdicts={};
            $('fpctThem')._v=null; $('fpctThem').textContent='0%';
            $('fpctYou')._v=null;  $('fpctYou').textContent='0%';
            $('halfThem').classList.remove('pass'); $('halfYou').classList.remove('pass');
            $('fboard').classList.remove('done');
            MMASYNC.clearResult();
            fbuild(); fslots(); frender(); }};
          fbuild(); fslots(); frender();
        })();
```

- [ ] **Step 6: Run the assertion to verify it passes**

Re-evaluate Step 1. Expected: `ok: true`, `slots: 10`, `mini: 10`, `hasBoth: true`, `res.them: 100`, ten verdicts recorded, landed `#S-F4`, guard under 40.

- [ ] **Step 7: Screenshot both viewports**

Screenshot `#S-F3` at **1280×800 and 390×844**, under Model A and again under Model B with a 5-question deck (`MMDEV.setModel('B')` after `MMQ.saveDeck(...)`). In all four, confirm the answer card and the Stay/Move buttons do not overlap, and that both score halves and the mini strip are fully visible. This is the exact bug class that broke this screen in Phase 1.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "feat(async): rebuild S-F3 as two symmetric boards with both scores"
```

---

### Task 6: Rebuild S-F4 with the result and the mirror reveal

**Files:**
- Modify: `index.html` — `S-F4` (~2759–2772); append CSS to the main `<style>`; guard the confetti in the router `<script>`.

**Interfaces:**
- Consumes: `MMASYNC.result()`, `MMASYNC.myAnswer(qid)`, `MMASYNC.theirAnswer(qid)`, `MMASYNC.theirVerdict(qid)`, `MMQ.buildSet()`, `MMQ.qById(id)`.
- Produces on `window`: `MMRESULT.paint()`. This is where spec §1.5 finally renders.

- [ ] **Step 1: Write the browser assertion**

```js
() => {
  if (!window.MMRESULT) return { ok:false, reason:'MMRESULT not defined' };
  const set = MMQ.buildSet();

  // a passing result
  const allMove = {}; set.forEach(q => { allMove[q.id]='move'; });
  MMASYNC.saveResult({ you:88, them:100, verdicts:allMove });
  location.hash='S-F4'; MMRESULT.paint();
  const passTxt = document.getElementById('S-F4').textContent.replace(/\s+/g,' ');
  const passRows = document.querySelectorAll('#S-F4 .mirrorrow').length;
  const passHead = document.getElementById('f4head').textContent;
  // rows with no recorded answer must say so, not show a default option
  const known = MMQ.answers().map(a => a.qid);
  const blanks = document.querySelectorAll('#S-F4 .mr-side.mr-none').length;
  const expectedBlanks = [...new Set(set.map(q => q.probe))]
    .filter(p => { const q = set.find(x => x.probe === p);
                   return known.indexOf(q.id) === -1; }).length;

  // a failing result
  const allStay = {}; set.forEach(q => { allStay[q.id]='stay'; });
  MMASYNC.saveResult({ you:40, them:0, verdicts:allStay });
  MMRESULT.paint();
  const failTxt = document.getElementById('S-F4').textContent.replace(/\s+/g,' ');
  const failHead = document.getElementById('f4head').textContent;
  const failCta = document.querySelector('#S-F4 .foot a, #S-F4 a.btn');

  MMASYNC.clearResult();
  return {
    ok: /pass/i.test(passHead) && /100%/.test(passTxt) && /88%/.test(passTxt) &&
        passRows >= 5 && blanks === expectedBlanks &&
        !/pass(ed)?\b/i.test(failHead) && /40%/.test(failTxt) &&
        /not refunded|no refund/i.test(failTxt) &&
        !/released|soft-held|soft-hold/i.test(passTxt + failTxt) &&
        failCta && failCta.getAttribute('href') !== '#S-E1',
    passHead: passHead.trim(), failHead: failHead.trim(),
    mirrorRows: passRows, blanks, expectedBlanks,
    failCta: failCta && failCta.getAttribute('href')
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `#S-F4`, evaluate. Expected: `{ ok:false, reason:'MMRESULT not defined' }`.

- [ ] **Step 3: Add the mirror-row CSS**

Append to the main `<style>` block:

```css
/* ===== async result · the mirrored-pairs reveal (spec 1.5) =====
   Each attribute both parties were asked about, what each said, and how the
   other reacted. Divergences are the interesting part, so they are marked. */
.mirrorrow{border-top:1px solid rgba(255,255,255,.08); padding:10px 0; text-align:left}
.paper .mirrorrow{border-top-color:var(--line)}
.mirrorrow:first-child{border-top:none}
.mirrorrow .mr-q{font-weight:700; font-size:12.5px; line-height:1.35}
.mirrorrow .mr-side{display:flex; justify-content:space-between; align-items:baseline;
  gap:10px; margin-top:5px; font-size:11.5px; color:var(--ink-dim)}
.paper .mirrorrow .mr-side{color:var(--t-dim)}
.mirrorrow .mr-who{font-weight:700; color:var(--ink-faint)}
.paper .mirrorrow .mr-who{color:var(--t-faint)}
.mirrorrow .mr-v{font-family:var(--display); font-weight:700; font-size:10px;
  letter-spacing:.1em; text-transform:uppercase; white-space:nowrap}
.mirrorrow .mr-v.move{color:var(--move)} .mirrorrow .mr-v.stay{color:var(--ink-faint)}
/* a profile fills in progressively — an unanswered side is stated plainly,
   never filled with a default that implies an answer nobody gave */
.mirrorrow .mr-side.mr-none{opacity:.55; font-style:italic}
.mirrorrow .mr-flag{display:inline-block; margin-top:6px; font-size:10px; letter-spacing:.1em;
  text-transform:uppercase; font-family:var(--display); font-weight:700; color:var(--bulb)}
```

- [ ] **Step 4: Replace the S-F4 body**

Replace `S-F4`'s `<div class="body">…</div>` with:

```html
        <div class="body" style="justify-content:flex-start;text-align:center;gap:14px;padding:26px 22px">
          <div id="f4icon" style="font-size:36px"></div>
          <h2 class="scrn" id="f4head" style="font-size:32px"></h2>
          <p class="muted" id="f4sub" style="max-width:32ch;margin:0 auto;line-height:1.5"></p>
          <div class="card" style="text-align:left" id="f4money"></div>
          <div class="card" style="text-align:left">
            <div class="eyebrow" style="color:var(--bulb)">Where you two actually landed</div>
            <div class="muted" style="font-size:11.5px;margin-top:4px">You were both asked about the same things. Here's each side, and how the other reacted.</div>
            <div id="f4mirror" style="margin-top:8px"></div>
          </div>
          <div id="f4cta"></div>
        </div>
        <script>
        (function(){
          /* Spec §1.5 renders here rather than in Phase 1, because this screen
             was rebuilt wholesale for the async restructure. */
          function rows(res){
            const set = MMQ.buildSet(), byProbe = {};
            set.forEach(q => { (byProbe[q.probe] = byProbe[q.probe] || []).push(q); });
            return Object.keys(byProbe).map(p => {
              const qs = byProbe[p], q = qs[0];
              const mine = MMASYNC.myAnswer(q.id), theirs = MMASYNC.theirAnswer(q.id);
              const myV = (res.verdicts||{})[q.id] || 'stay';
              const theirV = MMASYNC.theirVerdict(q.id);
              /* An unanswered row cannot diverge from anything — suppress both
                 flags rather than comparing against a fallback (§2.7). */
              return { q, qs, mine, theirs, myV, theirV,
                       answersDiffer: mine.answered &&
                         q.opts[mine.opt].v !== q.opts[theirs.opt].v,
                       verdictsDiffer: mine.answered && myV !== theirV };
            });
          }
          function paint(){
            const res = MMASYNC.result();
            if(!res) return;
            const pass = res.you >= 70 && res.them >= 70;
            document.getElementById('f4icon').innerHTML = pass
              ? '<svg class="icon" style="width:36px;height:36px;color:var(--bulb)"><use href="#i-party"/></svg>'
              : '<svg class="icon" style="width:36px;height:36px;color:var(--ink-faint)"><use href="#i-hourglass"/></svg>';
            const head = document.getElementById('f4head');
            head.textContent = pass ? 'You both passed' : 'Not this time';
            head.style.color = pass ? 'var(--pass)' : 'var(--ink)';
            document.getElementById('f4sub').innerHTML = pass
              ? 'Samuel <b>'+res.them+'%</b> · you <b>'+res.you+'%</b> — both clear of the 70% gate. It\'s a mutual match.'
              : 'Samuel <b>'+res.them+'%</b> · you <b>'+res.you+'%</b> — the gate is 70% for both of you, and it takes both.';

            document.getElementById('f4money').innerHTML =
                '<div style="display:flex;justify-content:space-between;font-size:13px"><span>Your fee</span><span class="tag pass">Charged</span></div>'
              + '<div style="display:flex;justify-content:space-between;font-size:13px;margin-top:8px"><span>Samuel\'s fee</span><span class="tag pass">Charged</span></div>'
              + '<div class="muted" style="font-size:11px;margin-top:8px">Both were taken before play. The fee buys the game, not the result — '
              + (pass ? 'and this one paid off.' : 'so a result under the gate is <b>not refunded</b>.') + '</div>';

            document.getElementById('f4mirror').innerHTML = rows(res).map(r =>
                '<div class="mirrorrow">'
              + '<div class="mr-q">' + r.q.text + '</div>'
              + '<div class="mr-side"><span><span class="mr-who">Samuel:</span> ' + r.q.opts[r.theirs.opt].t + '</span>'
              + '<span class="mr-v ' + r.myV + '">you ' + r.myV + '</span></div>'
              + (r.mine.answered
                  ? '<div class="mr-side"><span><span class="mr-who">You:</span> ' + r.q.opts[r.mine.opt].t + '</span>'
                    + '<span class="mr-v ' + r.theirV + '">he ' + r.theirV + 's</span></div>'
                  : '<div class="mr-side mr-none"><span><span class="mr-who">You:</span> not answered yet</span>'
                    + '<span class="mr-v">—</span></div>')
              + (r.answersDiffer ? '<span class="mr-flag">↯ you answered differently</span>' : '')
              + (!r.answersDiffer && r.verdictsDiffer ? '<span class="mr-flag">↯ same answer, different verdict</span>' : '')
              + (r.qs.length > 1 ? ' <span class="mr-flag" style="color:var(--ink-faint)">asked ' + r.qs.length + ' ways</span>' : '')
              + '</div>').join('');

            document.getElementById('f4cta').innerHTML = pass
              ? '<a class="btn pass block" href="#S-E1">Schedule the video call</a>'
              : '<a class="btn primary block" href="#S-B2">Back to your matches</a>';
          }
          window.MMRESULT = { paint };
          window.addEventListener('hashchange', () => {
            if(location.hash === '#S-F4') paint();
          });
          paint();
        })();
        </script>
```

- [ ] **Step 5: Guard the confetti**

The router fires confetti on entering `#S-F4`. A failed result must not get a celebration. In the main router `<script>`, change the `onEnter` confetti line from:

```js
    if(CELEBRATE.includes(location.hash)) confettiBurst(location.hash.slice(1));
```

to:

```js
    /* S-F4 now renders both outcomes — only celebrate an actual pass */
    if(CELEBRATE.includes(location.hash)){
      const r = location.hash === '#S-F4' && window.MMASYNC ? MMASYNC.result() : null;
      if(location.hash !== '#S-F4' || (r && r.you >= 70 && r.them >= 70))
        confettiBurst(location.hash.slice(1));
    }
```

- [ ] **Step 6: Run the assertion to verify it passes**

Re-evaluate Step 1. Expected: `ok: true`, `passHead: 'You both passed'`, `failHead: 'Not this time'`, `mirrorRows` ≥ 5, and the fail CTA pointing at `#S-B2` rather than `#S-E1`.

- [ ] **Step 7: Screenshot both outcomes**

With a passing result then a failing one, screenshot `#S-F4` at 1280×800 and 390×844. Confirm the mirror rows are readable, divergence flags are visible, and the fail state shows no confetti.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "feat(async): rebuild S-F4 with dynamic result and the mirrored-pairs reveal"
```

---

### Task 7: The 72-hour expiry screen and the async hold lifecycle

**Files:**
- Modify: `index.html` — insert `S-F5` after `S-F4`'s closing `</section>` (before `S-F6`); modify `S-G3` (~2805–2815); add a demo branch link to `S-F3`.

**Interfaces:**
- Consumes: `MMASYNC.expire()`, `MMASYNC.resetWindow()`.
- Produces: screen `S-F5`.

- [ ] **Step 1: Write the browser assertion**

```js
() => {
  const s = document.getElementById('S-F5');
  if (!s) return { ok:false, reason:'S-F5 not found' };
  const txt = s.textContent.replace(/\s+/g,' ');
  const g3 = document.getElementById('S-G3').textContent.replace(/\s+/g,' ');
  const rows = document.querySelectorAll('#S-G3 .row').length;
  return {
    ok: /72/.test(txt) && /forfeit/i.test(txt) &&
        /credit/i.test(txt) &&
        /async/i.test(g3) && /consent/i.test(g3) && rows >= 5 &&
        !/Nothing is captured until you actually play/i.test(g3),
    f5HasForfeit: /forfeit/i.test(txt),
    f5HasCredit: /credit/i.test(txt),
    g3Rows: rows, g3MentionsAsync: /async/i.test(g3)
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `#S-G3`, evaluate. Expected: `{ ok:false, reason:'S-F5 not found' }`.

- [ ] **Step 3: Insert S-F5**

Insert after `S-F4`'s closing `</section>` and before `S-F6`:

```html
      <section class="screen dark" id="S-F5" data-group="F · Async game" data-title="⚠ Async expired (72h forfeit)">
        <div class="body" style="justify-content:center;text-align:center;gap:18px;padding:40px 26px">
          <div class="av g5 s72" style="margin:0 auto"><svg class="icon" style="width:30px;height:30px"><use href="#i-hourglass"/></svg></div>
          <h2 class="scrn" style="font-size:28px">The 72 hours ran out.</h2>
          <p class="muted" style="max-width:32ch;margin:0 auto;line-height:1.55">Samuel never finished his board. An async game has to close inside 72 hours so nobody is left waiting indefinitely.</p>
          <div class="card" style="text-align:left">
            <div style="display:flex;justify-content:space-between;font-size:13px"><span>Samuel's fee</span><span class="tag danger">Forfeited</span></div>
            <div style="display:flex;justify-content:space-between;font-size:13px;margin-top:8px"><span>Your fee</span><span class="tag pass">Credit returned</span></div>
            <div class="muted" style="font-size:11px;margin-top:8px">You finished your side, so you're made whole — a session credit is back in your account. The side that left it unfinished forfeits.</div>
          </div>
          <div class="btnrow"><a class="btn ghost block" href="#S-G1">See your credits</a><a class="btn move block" href="#S-B2">Find someone else</a></div>
        </div>
      </section>
```

- [ ] **Step 4: Add the demo branch link to S-F3**

So the expiry branch is reachable the way every other failure branch in this prototype is, add to `S-F3`'s `<div class="fhead">`, immediately after the countdown `<span class="rec">…</span>`:

```html
          <a class="tag" href="#S-F5" style="background:rgba(255,255,255,.08);color:var(--ink-dim);text-decoration:none">demo · expire ›</a>
```

- [ ] **Step 5: Rewrite the S-G3 lifecycle**

Replace `S-G3`'s `<div class="body">…</div>` with:

```html
        <div class="body">
          <div class="note">A sync session and an async game charge at different moments. Here's every state either can be in.</div>
          <div class="eyebrow" style="margin-top:4px">Sync session</div>
          <div class="row"><span class="tag" style="background:var(--paper-2)">1</span><div class="grow"><div class="t1">Soft-held</div><div class="t2">Placed when the session is scheduled</div></div><span class="tag warn">Pending</span></div>
          <div class="row"><span class="tag" style="background:var(--paper-2)">2</span><div class="grow"><div class="t1">Captured</div><div class="t2">Both showed up and played</div></div><span class="tag pass">Charged</span></div>
          <div class="row" style="border-color:var(--danger)"><span class="tag" style="background:var(--paper-2)">3</span><div class="grow"><div class="t1">Forfeited</div><div class="t2">You no-showed (100%, &lt;6h to reschedule)</div></div><span class="tag danger">Lost</span></div>
          <div class="row" style="border-color:var(--pass)"><span class="tag" style="background:var(--paper-2)">4</span><div class="grow"><div class="t1">Released</div><div class="t2">Partner no-showed — reuse your hold</div></div><span class="tag pass">Returned</span></div>
          <div class="eyebrow" style="margin-top:14px">Async game</div>
          <div class="row"><span class="tag" style="background:var(--paper-2)">1</span><div class="grow"><div class="t1">Charged at consent</div><div class="t2">Pursuer on send, pursued the moment they accept — before play</div></div><span class="tag pass">Charged</span></div>
          <div class="row"><span class="tag" style="background:var(--paper-2)">2</span><div class="grow"><div class="t1">Kept on a fail</div><div class="t2">Under the gate is still a game played — the fee buys the game, not the result</div></div><span class="tag">No refund</span></div>
          <div class="row" style="border-color:var(--danger)"><span class="tag" style="background:var(--paper-2)">3</span><div class="grow"><div class="t1">Forfeited at 72h</div><div class="t2">You left your board unfinished inside the window</div></div><span class="tag danger">Lost</span></div>
          <div class="row" style="border-color:var(--pass)"><span class="tag" style="background:var(--paper-2)">4</span><div class="grow"><div class="t1">Credit returned</div><div class="t2">You finished, they didn't — you're made whole</div></div><span class="tag pass">Returned</span></div>
        </div>
```

- [ ] **Step 6: Run the assertion to verify it passes**

Re-evaluate Step 1. Expected: `ok: true`, `g3Rows: 8`, `g3MentionsAsync: true`.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat(async): add S-F5 expiry screen and split the S-G3 hold lifecycle"
```

---

### Task 8: Expand the question bank so attributes can actually be confirmed

**Files:**
- Modify: `index.html` — the `BANK` array inside `<script id="mmq">`.

**Interfaces:**
- Consumes: the `o(t, v)` option helper already in that closure.
- Produces: `MMQ.BANK` grows from 15 to 20 questions. Five probes become
  confirmable instead of one. No signature changes.

**Why this task exists.** Confidence is damped by `min(1, hits/3)`, so an
attribute needs **three** differently-worded questions before consistent
answers can clear the 70 gate. Verified against the live bank on 2026-08-24:
only `wants-children` has three variations. Nine of ten probes cap at 67 or 33
and can *never* be confirmed. Task 9's eligibility gate would be
mathematically unsatisfiable without this.

- [ ] **Step 1: Write the browser assertion**

```js
() => {
  const M = window.MMQ;
  const byProbe = {};
  M.BANK.forEach(q => { (byProbe[q.probe] = byProbe[q.probe] || []).push(q.id); });
  const confirmable = Object.keys(byProbe)
    .filter(p => Math.round(Math.min(1, byProbe[p].length/3) * 100) >= M.CONF_GATE);
  const malformed = M.BANK.filter(q =>
    !q.id || !q.dim || !q.probe || !q.text ||
    !Array.isArray(q.opts) || q.opts.length !== 4 ||
    q.opts.some(o => !o.t || !o.v) || !(q.dim in M.DIMS));
  const ids = M.BANK.map(q => q.id);
  // Phase 1's pinned confidence fixtures must not move
  const conf = { children:M.confidence('wants-children', M.DEMO.answers),
                 value:M.confidence('core-value', M.DEMO.answers),
                 weekend:M.confidence('weekend', M.DEMO.answers),
                 money:M.confidence('money-model', M.DEMO.answers) };
  return {
    ok: M.BANK.length === 20 && malformed.length === 0 &&
        ids.length === new Set(ids).size &&
        confirmable.length === 5 &&
        ['wants-children','core-value','weekend','money-model','closeness']
          .every(p => confirmable.includes(p)) &&
        conf.children===100 && conf.value===67 && conf.weekend===33 && conf.money===33,
    bankSize: M.BANK.length, confirmable, malformed: malformed.map(q=>q.id), conf
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `#S-E6`, evaluate. Expected: `bankSize: 15`, `confirmable: ['wants-children']` — one, not five.

- [ ] **Step 3: Add five questions to BANK**

Insert these into the `BANK` array, each next to its existing probe-mates so
the array stays readable. Keep the `o(text, value)` helper and reuse the
**exact** `v` values already used by that probe — confidence compares `v`, so a
new spelling would silently never agree with the existing questions:

```js
    { id:'q-val-04', dim:'Values', depth:'Exploratory', probe:'core-value',
      text:'A partner does something generous nobody will ever know about. What does that tell you?',
      opts:[o('That they are honest to the core','honesty'), o('That they are genuinely kind','kindness'),
            o('That they are building something','ambition'), o('That their faith is real','faith')] },

    { id:'q-lif-03', dim:'Lifestyle', depth:'Exploratory', probe:'weekend',
      text:'A free Sunday appears with nothing planned. What happens to it?',
      opts:[o('It fills up with people I love','family'), o('It stays gloriously empty','home'),
            o('I get in the car and go somewhere','adventure'), o('I finally finish that thing','work')] },

    { id:'q-fin-03', dim:'Finances', depth:'Deep dive', probe:'money-model',
      text:'One of you out-earns the other three to one. How do the bills work?',
      opts:[o('One pot — it is all ours','joint'), o('Proportional, separate accounts','hybrid'),
            o('Straight down the middle regardless','separate'), o('The higher earner carries it','earner')] },

    { id:'q-int-02', dim:'Intimacy', depth:'Exploratory', probe:'closeness',
      text:'After a hard day, what do you actually want from a partner?',
      opts:[o('The usual comforts, without asking','rituals'), o('To talk it all the way through','talk'),
            o('To be held','touch'), o('Just company, no talking','presence')] },

    { id:'q-int-03', dim:'Intimacy', depth:'Deep dive', probe:'closeness',
      text:'What would make you feel most distant from someone you live with?',
      opts:[o('The small rituals stopping','rituals'), o('No real conversations any more','talk'),
            o('No physical affection','touch'), o('Never being in the same room','presence')] }
```

Do **not** add matching entries to `DEMO.answers` — Phase 1's confidence
fixtures are pinned by the assertion above and must stay exactly where they are.

- [ ] **Step 4: Run the assertion to verify it passes**

Re-evaluate Step 1. Expected: `ok: true`, `bankSize: 20`, `confirmable` listing all five probes, and the four Phase 1 confidence values unchanged at 100 / 67 / 33 / 33.

- [ ] **Step 5: Confirm the games still build correctly**

```js
() => {
  sessionStorage.removeItem('mm_deck'); MMQ.setModel('A');
  const a = MMQ.buildSet().map(q=>q.id);
  MMQ.setModel('B');
  const b = MMQ.buildSet().map(q=>q.id);
  MMQ.setModel('A');
  return { ok: a.length===10 && b.length===10 &&
               a.length===new Set(a).size && b.length===new Set(b).size,
           lenA:a.length, lenB:b.length };
}
```

Expected: `ok: true`. `buildSet` slices the first ten bank questions, so a larger bank must not change the game length.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(questions): expand bank to 20 so five attributes can be confirmed"
```

---

### Task 9: Async eligibility gate

**Files:**
- Modify: `index.html` — add eligibility to `<script id="mmq">`; add a lock to `<script id="mmasync">`; gate `S-F1`; add the eligibility panel to `S-F2A`; add a dev-panel row.

**Interfaces:**
- Consumes: `MMQ.confidence`, `MMQ.probes`, `MMQ.answers`, `MMQ.CONF_GATE`, `MMQ.buildSet`, `MMASYNC`.
- Produces on `window`: `MMQ.ELIG_CONFIRMED` (3), `MMQ.ELIG_ANSWERED` (12), `MMQ.eligibility(answers)`, `MMQ.setProfileDepth(d)`, `MMQ.DEMO.answersFull`; `MMASYNC.lockEligibility()`, `MMASYNC.lockedEligibility()`; `MMELIG.paint()`.

**The rule (spec §2.7).** A user may neither initiate an async pursuit nor be
pursued unless **3 attributes are Confirmed AND 12 questions are answered**.
Eligibility is checked at invite and consent, then **locked for the duration of
that game** — otherwise correcting a wrong inference on `S-E6` mid-game could
evaporate a match in progress. Sync stays completely ungated.

- [ ] **Step 1: Write the browser assertion**

```js
() => {
  const M = window.MMQ;
  if (!M.eligibility) return { ok:false, reason:'MMQ.eligibility not defined' };

  M.setProfileDepth('thin');
  const thin = M.eligibility();
  M.setProfileDepth('full');
  const full = M.eligibility();

  // the gate is visible on S-F1
  location.hash = 'S-F1'; MMELIG.paint();
  const fullCta = document.querySelector('#S-F1 .foot a.btn, #S-F1 a.btn.move');
  const fullLocked = !!document.querySelector('#S-F1 .btn.disabled');
  M.setProfileDepth('thin'); MMELIG.paint();
  const thinLocked = !!document.querySelector('#S-F1 .btn.disabled');
  const thinTxt = document.getElementById('S-F1').textContent.replace(/\s+/g,' ');

  // locking freezes the verdict even if the profile later thins out
  M.setProfileDepth('full');
  MMASYNC.lockEligibility();
  M.setProfileDepth('thin');
  const locked = MMASYNC.lockedEligibility();

  M.setProfileDepth('full');
  return {
    ok: thin.ok === false && full.ok === true &&
        full.confirmed >= M.ELIG_CONFIRMED && full.answered >= M.ELIG_ANSWERED &&
        thin.needConfirmed > 0 &&
        thinLocked === true && fullLocked === false &&
        /\d+ more/.test(thinTxt) &&
        locked && locked.ok === true,
    thin, full, thinLocked, fullLocked, lockedVerdict: locked
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `#S-F1`, evaluate. Expected: `{ ok:false, reason:'MMQ.eligibility not defined' }`.

- [ ] **Step 3: Add eligibility to MMQ**

Inside the `<script id="mmq">` IIFE, immediately before the `window.MMQ = {...}` line:

```js
  /* ===== async eligibility (spec 2.7) =====
     You must be *known* before you can pursue or be pursued. Measured by
     confirmed attributes rather than raw answer count, because confidence
     only rises when variations of the same probe agree — it cannot be
     reached by answering fast. The answered floor is a secondary guard
     against someone consistent on very few questions. */
  const ELIG_CONFIRMED = 3, ELIG_ANSWERED = 12;
  function eligibility(ans){
    ans = ans || answers();
    const confirmedProbes = probes().filter(p => {
      const c = confidence(p, ans);
      return c !== null && c >= CONF_GATE;
    });
    const confirmed = confirmedProbes.length, answered = ans.length;
    return { confirmed, confirmedProbes, answered,
             ok: confirmed >= ELIG_CONFIRMED && answered >= ELIG_ANSWERED,
             needConfirmed: Math.max(0, ELIG_CONFIRMED - confirmed),
             needAnswered:  Math.max(0, ELIG_ANSWERED - answered) };
  }
  /* Prototype switch between a thin profile and a well-answered one, so both
     sides of the gate are demoable. Writing mm_answers is also what makes
     answers() return something other than the sparse demo fixture. */
  function setProfileDepth(d){
    if(d === 'full') sessionStorage.setItem('mm_answers', JSON.stringify(DEMO.answersFull));
    else sessionStorage.removeItem('mm_answers');
  }
```

Add `ELIG_CONFIRMED, ELIG_ANSWERED, eligibility, setProfileDepth` to the exported object, **preserving every name already there**.

- [ ] **Step 4: Add the full-profile fixture**

Inside `DEMO`, immediately after the existing `answers: [...]` array, add:

```js
    /* Twelve answers, consistent across four probes — clears both thresholds
       with one to spare. wants-children, core-value, closeness and
       money-model each get their three variations answered the same way. */
    answersFull: [
      { qid:'q-fam-01', opt:0, why:'I always pictured a bigger, noisier house.' },
      { qid:'q-fam-03', opt:1, why:'' },
      { qid:'q-fam-06', opt:0, why:'' },
      { qid:'q-val-01', opt:0, why:'Honesty. And kindness when no one is watching.' },
      { qid:'q-val-03', opt:0, why:'' },
      { qid:'q-val-04', opt:0, why:'' },
      { qid:'q-int-01', opt:0, why:'Tea made without asking.' },
      { qid:'q-int-02', opt:0, why:'' },
      { qid:'q-int-03', opt:0, why:'' },
      { qid:'q-fin-01', opt:1, why:'Shared goals, honest numbers, some fun money.' },
      { qid:'q-fin-02', opt:1, why:'' },
      { qid:'q-fin-03', opt:1, why:'' }
    ],
```

- [ ] **Step 5: Add the lock to MMASYNC**

Inside the `<script id="mmasync">` IIFE, before its `window.MMASYNC = {...}` line:

```js
  /* Eligibility is frozen when a game starts (§2.7). Without this, correcting
     a wrong inference on S-E6 mid-game could drop you under the threshold and
     evaporate a match already in progress. */
  const ELG = 'mm_async_elig';
  function lockEligibility(){
    const e = MMQ.eligibility();
    sessionStorage.setItem(ELG, JSON.stringify(e));
    return e;
  }
  function lockedEligibility(){
    try { return JSON.parse(sessionStorage.getItem(ELG)) || null; }
    catch(e){ return null; }
  }
  function clearEligibility(){ sessionStorage.removeItem(ELG); }
```

Add `lockEligibility, lockedEligibility, clearEligibility` to the exported object, preserving every existing name.

- [ ] **Step 6: Gate S-F1**

Task 2 rewrote `S-F1`'s body, ending it with this line:

```html
          <a class="btn move block" href="#S-F2">Start an async game</a>
```

Replace **that exact line** (it is inside `.body`, not a `.foot`) with a mount point:

```html
          <div id="f1gate"></div>
```

Then add this `<script>` immediately before `S-F1`'s closing `</section>`:

```html
        <script>
        (function(){
          /* Async is what you UNLOCK, never what you are locked out of —
             matching and the sync show stay open to everyone (spec 2.7). */
          function paint(){
            const host = document.getElementById('f1gate');
            if(!host) return;
            const e = MMQ.eligibility();
            if(e.ok){
              host.innerHTML = '<a class="btn move block" href="#S-F2" '
                + 'onclick="MMASYNC.lockEligibility()">Start an async game</a>';
              return;
            }
            const need = [];
            if(e.needConfirmed) need.push(e.needConfirmed + ' more things confirmed about you');
            if(e.needAnswered)  need.push(e.needAnswered + ' more questions');
            host.innerHTML =
                '<div class="card" style="text-align:left">'
              + '<div class="eyebrow" style="color:var(--bulb)">Not unlocked yet</div>'
              + '<div class="muted" style="font-size:12.5px;margin-top:6px;line-height:1.55">'
              + 'Async only works when we know you both well enough to be sure a match is real. '
              + 'You need ' + need.join(' and ') + '.</div>'
              + '<div class="prog" style="margin-top:10px"><i style="width:'
              + Math.round(Math.min(1, e.confirmed / MMQ.ELIG_CONFIRMED) * 100)
              + '%;background:var(--bulb)"></i></div>'
              + '<div class="muted" style="font-size:11px;margin-top:6px">'
              + e.confirmed + ' of ' + MMQ.ELIG_CONFIRMED + ' things confirmed · '
              + e.answered + ' of ' + MMQ.ELIG_ANSWERED + ' questions answered</div>'
              + '</div>'
              + '<a class="btn disabled block" style="margin-top:10px">Start an async game</a>'
              + '<a class="btn move block" style="margin-top:8px" href="#S-QD1">Answer a few more ›</a>';
          }
          window.MMELIG = { paint };
          window.addEventListener('hashchange', paint);
          paint();
        })();
        </script>
```

- [ ] **Step 7: Show the pursued what they actually have to do**

In `S-F2A`, add a mount point on the line immediately **after** the closing of
the `<div id="pursuerQs" …></div>` element, still inside the same `.card`:

```html
            <div id="f2aKnown" class="muted" style="font-size:11.5px;margin-top:8px"></div>
```

Then inside `S-F2A`'s existing `<script>`, at the end of its `paint()` function, add:

```js
            /* The ask is only small because eligibility guaranteed answers are
               already on file (§2.7) — say so, or it reads as 15 cold questions. */
            const known = document.getElementById('f2aKnown');
            if(known){
              const set = MMQ.buildSet(), mine = MMQ.answers();
              const have = set.filter(q => mine.some(a => a.qid === q.id)).length;
              const fresh = set.length - have;
              known.innerHTML = have
                ? 'You\'ve already answered <b>' + have + '</b> of these. Only <b>'
                  + fresh + '</b> are new.'
                : 'All <b>' + set.length + '</b> are new to you.';
            }
```

- [ ] **Step 8: Add the dev-panel row**

In the dev panel markup, immediately after the existing question-model `.devrow`:

```html
  <div class="devrow">
    <div class="lbl2">Profile depth</div>
    <div class="seg">
      <button id="devpt" onclick="MMDEV.setDepth('thin')">Thin</button>
      <button id="devpf" onclick="MMDEV.setDepth('full')">Complete</button>
    </div>
    <div class="note2">Switches the demo answer set, so you can see both sides of the async eligibility gate.</div>
  </div>
```

And in `MMDEV`, add the method and extend `paint()`:

```js
    setDepth(d){
      MMQ.setProfileDepth(d); MMDEV.paint();
      if(window.MMELIG)  MMELIG.paint();
      if(window.MMCONF)  MMCONF.paint();
    },
```

Inside `MMDEV.paint()`, after the existing model toggle lines, add:

```js
      const pt = document.getElementById('devpt'), pf = document.getElementById('devpf');
      if(pt && pf){
        const full = !!sessionStorage.getItem('mm_answers');
        pt.classList.toggle('on', !full);
        pf.classList.toggle('on', full);
      }
```

- [ ] **Step 9: Make sync play feed the answer pool**

Eligibility is only fair if playing the flagship mode counts toward it (§2.7).
Answers given in a sync show must accumulate, and confirmation draws on sync
responses and the profile together. Right now `mm_answers` is read by
`MMQ.answers()` and **written by nothing** — that gap is why the gate would
otherwise be a separate grind.

Add to `<script id="mmq">`, immediately before the `window.MMQ = {...}` line:

```js
  /* Answers accumulate across modes (§2.7): what you answer in a sync show
     counts toward async eligibility and toward confirming attributes. Answering
     the same question again overwrites — people change their minds, and the
     profile is progressively reconciled rather than fixed at onboarding. */
  function recordAnswer(ans){
    if(!ans || !ans.qid) return null;
    const all = answers().slice();
    const rec = { qid:ans.qid, opt:ans.opt, why:ans.why || '' };
    const at = all.findIndex(a => a.qid === ans.qid);
    if(at === -1) all.push(rec); else all[at] = rec;
    sessionStorage.setItem('mm_answers', JSON.stringify(all));
    return all.length;
  }
```

Add `recordAnswer` to the exported object, preserving every existing name.

Then in `S-D2`'s inline script, inside `lock()`, record the answer as the first
statement after `t` and `v` are declared:

```js
            MMQ.recordAnswer(t.ans);   // sync play feeds the profile (§2.7)
```

**This is correct even though `t.ans` may have started as a fallback.** For a
`'you'` turn, `t.ans` is what the arena displays as *your answer* and what you
then lock in — so in the fiction of the show you are answering that question
right now. Locking it is precisely the moment it becomes a real answer, which
is what turns a fallback into a recorded one and makes the accumulation story
work. Do not add a guard against it.

- [ ] **Step 10: Assert that sync play moves eligibility**

```js
async () => {
  const sleep = ms => new Promise(r => setTimeout(r, ms));
  MMQ.setProfileDepth('thin');
  const before = MMQ.eligibility();
  location.hash = 'S-D2'; DSHOW.reset();
  await sleep(60);
  // play four turns; the 'you' turns are the ones that record an answer
  /* Six turns, not four: the thin demo fixture already answers the two
     questions that fall on the player's turns in the first four, and
     recordAnswer dedupes by id — so no growth is observable until turn 5. */
  let guard = 0;
  while (guard++ < 6 && location.hash === '#S-D2') {
    const lockBtn = document.getElementById('act').querySelector('button[onclick*="lock"]');
    if (lockBtn) { DSHOW.lock(); await sleep(700); document.getElementById('rnext').click(); }
    else { DSHOW.react('move'); await sleep(850); }
    await sleep(30);
  }
  const after = MMQ.eligibility();
  MMQ.setProfileDepth('thin');
  return { ok: after.answered > before.answered,
           before:{answered:before.answered, confirmed:before.confirmed},
           after:{answered:after.answered, confirmed:after.confirmed} };
}
```

Expected: `ok: true` — `after.answered` is strictly greater than `before.answered`. Playing the sync show measurably moves you toward unlocking async, which is the whole point of the cross-mode rule.

- [ ] **Step 11: Run the eligibility assertion to verify it passes**

Re-evaluate Step 1. Expected: `ok: true` — `thin.ok: false` with `needConfirmed > 0`, `full.ok: true`, the locked CTA present only on a thin profile, and `lockedVerdict.ok: true` even after the profile is thinned afterwards.

- [ ] **Step 12: Screenshot both sides of the gate**

Screenshot `#S-F1` at 1280×800 and 390×844, under **both** profile depths. The locked state must read as an invitation with visible progress, never as a rejection. Also screenshot `#S-F2A` and confirm the answered-vs-new line appears.

- [ ] **Step 13: Commit**

```bash
git add index.html
git commit -m "feat(async): gate async on profile depth, locked at game start"
```

---

### Task 10: Full-flow regression and visual sweep

**Files:**
- None expected — verification only. Commit any fix you are asked to make.

- [ ] **Step 1: Walk every screen and check for errors**

```js
() => {
  const ids = [...document.querySelectorAll('.screen')].map(s => s.id);
  const errs = [];
  window.onerror = m => errs.push(m);
  ids.forEach(id => { location.hash = id; });
  return { screens: ids.length,
           newOnes: ['S-F2A','S-F5','S-F6'].filter(i => ids.includes(i)),
           errs };
}
```

Expected: `screens` is 69 (66 from Phase 1 plus three), `newOnes` lists all three, `errs: []`.

- [ ] **Step 2: Check the console**

Run `browser_console_messages` at level `error`. Expected: zero errors. The favicon 404 and any Unsplash image 404s are pre-existing and acceptable; anything naming `MMQ`, `MMASYNC`, `MMRESULT`, `MMDECLINE`, `FSHOW` or `MMDECK` is not.

- [ ] **Step 3: Play the async game end to end under both count models**

```js
async () => {
  const sleep = ms => new Promise(r => setTimeout(r, ms));
  const out = {};
  for (const m of ['A','B']) {
    sessionStorage.removeItem('mm_deck');
    MMQ.setModel(m);
    if (m === 'B') MMQ.saveDeck(MMQ.BANK.slice(0,4).map((q,n)=>({...q,id:'cq'+n,src:'custom',author:'You',review:{ok:true,reason:'Cleared'}})));
    MMASYNC.clearResult(); MMASYNC.resetWindow();
    location.hash = 'S-F3'; FSHOW.reset();
    await sleep(80);
    const slots = document.querySelectorAll('#S-F3 .slot').length;
    let guard = 0;
    while (location.hash.replace('#','') === 'S-F3' && guard++ < 40) {
      if (document.querySelector('#S-F3 .actf .btn.move')) { FSHOW.react('move'); await sleep(1000); }
      else await sleep(300);
    }
    out[m] = { slots, landed: location.hash, res: MMASYNC.result(), guard };
  }
  MMQ.setModel('A'); sessionStorage.removeItem('mm_deck');
  return out;
}
```

Expected: Model A gives 10 slots, Model B gives 14 (10 bank + 4 deck); both land on `#S-F4` without hitting the guard, and both store a result with `them: 100`.

- [ ] **Step 4: Confirm the countdown ticks and is shared**

```js
async () => {
  const sleep = ms => new Promise(r => setTimeout(r, ms));
  MMASYNC.resetWindow();
  location.hash = 'S-F2A';
  const a = document.querySelector('#S-F2A [data-mm-countdown]').textContent;
  await sleep(2200);
  const b = document.querySelector('#S-F2A [data-mm-countdown]').textContent;
  location.hash = 'S-F3';
  const c = document.querySelector('#S-F3 [data-mm-countdown]').textContent;
  MMASYNC.expire();
  await sleep(1200);
  const d = document.querySelector('#S-F3 [data-mm-countdown]').textContent;
  MMASYNC.resetWindow();
  return { ok: a !== b && c !== '' && d === '00:00:00', first:a, later:b, onF3:c, expired:d };
}
```

Expected: `ok: true` — the countdown moves, appears on both screens, and reaches `00:00:00` when expired.

- [ ] **Step 5: The visual sweep — this is not optional**

Phase 1's worst defect was invisible to eight diff reviews and obvious in a screenshot. This phase changes how many elements `S-F3` draws, which is the same condition.

Screenshot every one of these at **1280×800 and again at 390×844**:

`#S-F1` (**both profile depths** — locked and unlocked), `#S-F2`, `#S-F2A`, `#S-F3` (Model A and Model B), `#S-F4` (passing and failing), `#S-F5`, `#S-F6`, `#S-G3`

Set the profile depth with `MMDEV.setDepth('thin')` / `MMDEV.setDepth('full')`. The locked `S-F1` must read as an invitation with visible progress — if it reads as a rejection, say so.

For each, check and report: does any element overlap another; is any text clipped or truncated; is anything unreadable against its background; does the layout look consistent with the rest of the prototype. **Do not fix what you find — list it.**

- [ ] **Step 6: Report**

Write up the sweep with one line per screen per viewport. Flag anything that looks wrong even if you are unsure.

- [ ] **Step 7: Fix the ambiguous fee label on S-F2**

`S-F2`'s fee card reads **"Your fee, charged now"**, but it sits above a Send
button the user has not pressed yet — read literally it says the charge already
fired. On a payment screen that ambiguity is not acceptable. Change that label
to:

```html
            <div style="display:flex;justify-content:space-between"><span>Your fee, charged when you send</span><b>£19.00</b></div>
```

Leave the contrasting "Samuel's fee · On his consent" row exactly as it is —
the pairing is what makes the two triggers legible.

- [ ] **Step 8: Fix the out-of-scope `.t2` class on S-F2A**

`S-F2A`'s rendered pursuer-questions use `class="t2"`, but the only `.t2` rule
in the file is scoped `.row .grow .t2` — these divs are not inside a `.row`, so
they render at inherited size instead of the intended small muted style.

Fix at the call site rather than by adding a bare `.t2` base rule: that class is
used widely and changing its specificity this late risks shifting layouts the
phase has already verified. In `S-F2A`'s inline script, change the question line
to use an existing global class:

```js
            host.innerHTML = shown.map(t =>
              '<div class="muted" style="font-size:12px" data-pursuer-q>· ' + t + '</div>').join('')
```

Keep the `data-pursuer-q` attribute — an assertion counts those elements.

- [ ] **Step 9: Merge the duplicate `#S-F3 .board` sizing rules**

Task 5 revealed that `S-F3`'s scoped style block ended up with **two**
same-selector, same-specificity `.board`/`.slots` max-height rules. They were
made to agree at 32vh, which removed the bug — but not the trap. Two rules that
match only by construction is exactly how the silent-override returns, and
Phase 3 rebuilds this very CSS into a windowed ladder.

Delete the **earlier, redundant** block (currently around line 2775) — these
three lines:

```css
          /* the board shrinks a little to make room for the second score row */
          #S-F3 .board{max-height:32vh}
          #S-F3 .slots{max-height:calc(32vh - 30px)}
```

The later block already sets the same two properties plus `overflow`, so it
fully supersedes them. Keep the useful comment by folding it into the surviving
block, which becomes:

```css
          /* The board holds one slot per question — 10 under Model A, up to 15
             under Model B — so it must scroll rather than push the stage into
             the button row, and it is capped to leave room for the second
             score row. Phase 3 replaces this with a windowed ladder. */
          #S-F3 .board{max-height:32vh; overflow:hidden}
```

Leave the `.slots` rules in that block untouched.

**Then re-measure.** Deleting the wrong rule would silently restore the overlap
this phase exists to prevent, so confirm at 1280×800 and 390×844 under both
question-count models that the answer card's bottom edge is still above the
action bar's top edge. Report the four numbers.

- [ ] **Step 10: Fix S-F4's stale title and complete the mirror-row theming**

Two leftovers from the `S-F4` rebuild.

**(a) The screen's title still describes the old payment model.** `S-F4` carries
`data-title="Async result + buy-in soft-hold"` — but there is no soft-hold any
more, and this string is user-visible: the breadcrumb and the ☰ screen index
both render from `dataset.title`. Change it to:

```html
      <section class="screen dark" id="S-F4" data-group="F · Async game" data-title="Async result + the reveal">
```

While you are there, check the other `F · Async game` titles for the same
staleness and report anything you find — do not fix those without saying so.

**(b) Two mirror-row colours have no `paper` variant.** Every other faint colour
in that component gets a `.paper` override; these two were missed, which breaks
the standing convention that shared components work on both surfaces. Add
alongside the existing overrides in the main `<style>` block:

```css
.paper .mirrorrow .mr-v.stay{color:var(--t-faint)}
```

And in `S-F4`'s inline script, the "asked N ways" flag hardcodes
`style="color:var(--ink-faint)"`. Replace that inline colour with a class so it
themes with everything else — add to the main `<style>`:

```css
.mirrorrow .mr-flag.soft{color:var(--ink-faint)}
.paper .mirrorrow .mr-flag.soft{color:var(--t-faint)}
```

and change the flag's markup in the script from
`'<span class="mr-flag" style="color:var(--ink-faint)">'` to
`'<span class="mr-flag soft">'`.

- [ ] **Step 11: Commit any fixes you were asked to make**

```bash
git add index.html
git commit -m "fix(async): resolve full-flow regression findings"
```

---

## Self-review

**Spec coverage.** §2.7 → Tasks 8, 9 (bank expansion is a hard prerequisite — without it only one attribute is confirmable and the gate is unsatisfiable; then the gate itself, locked at game start). §2.1 → Tasks 1, 5 (symmetric play, both sides' fixtures, two scores). §2.2 → Task 5 (main board plus the mini strip filling in as you play). §2.3 → Tasks 2, 6, 7 (charge-up-front copy on `S-F1`/`S-F2`, "Charged"/no-refund on `S-F4`, the async lifecycle on `S-G3`). §2.4 → Task 3 (`S-F2A`, questions shown before the fee card). §2.5 → Tasks 1, 5, 7 (72h deadline, countdown on `S-F2A` and `S-F3`, `S-F5` forfeit screen). §2.6 → Task 4 (`S-F6`, MECE options plus optional free text, required before submit). §1.5 → Task 6 (the mirror block, with divergence flags). No gaps.

**Type consistency.** `MMASYNC.scores(myVerdicts)` is defined in Task 1 and its `{you, them}` shape is consumed in Task 5's `ffinish()` reasoning and Task 6's `paint()`. `MMASYNC.saveResult({you, them, verdicts})` in Task 5 matches `MMASYNC.result()` reads in Task 6. `theirVerdict(qid)` returns exactly `'move'|'stay'` and is used as a CSS class in both Task 5 and Task 6. `FSHOW` keeps `{react, reset}` throughout.

§2.7's cross-mode rule → Task 9 Steps 9–10 (`recordAnswer`, wired into `S-D2`'s
`lock()`, with an assertion that playing the sync show measurably moves
eligibility). §2.7's progressive-reconciliation consequence → Task 1's
`answered` flag and Task 6's `.mr-none` rendering, so an unanswered row says so
instead of showing a default option.

**Task 9 is the largest task in the plan** at 13 steps. It is kept whole rather
than split because everything in it serves one deliverable — an eligibility
gate that is real and fair — and a reviewer benefits from seeing the gate and
the mechanism that makes it reachable together.

**Known sharp edges.**
1. Task 5 replaces the whole `S-F3` script. The old one declared `QF`/`TOTW` via `fbuild()` — keep that name, since Phase 1's reviewers verified the rebuild-before-render ordering and Task 5's `reset()` preserves it.
2. Task 5's assertion depends on `MMQ.DEMO.verdict`, which marks two questions `stay`. Your own score therefore lands below 100 even on an all-Move run — that is correct and intended, and is why the assertion checks `res.them === 100` but only checks that `res.you` is a number.
3. Task 6's `rows()` reads `q.opts[mine.opt]`. Task 1's `myAnswer()` fallback guarantees a valid index, and Task 1's assertion checks exactly that for every question in the set. If Task 1 was weakened, Task 6 throws.
