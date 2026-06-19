# MakeaMove Game-Show UI Refinement Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Refine the existing `index.html` MakeaMove game-show mockup to satisfy the four `prompt.md` requirements it does not yet meet: an 80%-width content rail, a pink "correct answer" reveal on Move, flat 5-point/70%-of-total scoring, and end-screen copy rewritten in points/percentage terms.

**Architecture:** A single self-contained `index.html` (no build step, no framework). Decorative backdrop layers fill 100% of the screen; a new centered `.frame` wrapper holds all foreground content within the middle 80%. A vanilla-JS engine drives state from a `QUESTIONS` bank. Verification is browser-driven via the Playwright MCP tools against the local file.

**Tech Stack:** HTML + CSS + vanilla JavaScript (single file). Playwright MCP for verification.

## Global Constraints

- **Single file only.** All changes live in `index.html`. No new runtime files, no build step, no dependencies.
- **Backdrop fills 100%, foreground within middle 80%.** Decorative layers (`.bulbwall`, `.beam`, `.rail` amber line, `.grain`, `.vignette`) stay full-screen; all foreground content sits inside `.frame` (80% width, 10% gutters each side).
- **Button colors fixed:** Stay = `--stay` gray (`#5B6685`), Move = `--move` pink (`#FF2D6F`). Buttons pinned at the bottom.
- **Scoring:** flat 5 points per question; `TOTAL_POINTS = QUESTIONS.length * 5`; win when final `earned / TOTAL_POINTS >= 70%`. Pass line drawn at 70%.
- **Do not break existing behavior:** keep animations, host voice, keyboard shortcuts (`←`/`S` = Stay, `→`/`M` = Move), `prefers-reduced-motion` handling, the `≤760px` responsive layout, and "Play again" reset.
- **Naming:** the new wrapper class is `.frame`. Do NOT reuse `.rail` — that class already styles the decorative amber neon line.
- **Verification base URL:** `file:///mnt/e/TOOLMAKER/PYTHON/MakeaMoveUI_Mockup/index.html`, viewport resized to **1280×800** before assertions.

---

## File Structure

- Modify: `/mnt/e/TOOLMAKER/PYTHON/MakeaMoveUI_Mockup/index.html` (the only file touched)
  - CSS block (`<style>`, lines ~11–195): add `.frame`, reposition `.ladder`, restyle `.panel.flip-move`, add points-readout styling.
  - HTML body (lines ~198–254): wrap foreground in `.frame`, add a points readout, make the question counter total dynamic.
  - `<script>` (lines ~256–388): rewrite the scoring logic (`react`, `updateScore`, `end`), strip the `w` weighting from `QUESTIONS`, set the correct-answer label on Move, reset it on render.

Because everything is one file, tasks are sequenced so each ends at an independently browser-verifiable state and is committed separately.

---

### Task 1: 80%-width content rail

**Files:**
- Modify: `/mnt/e/TOOLMAKER/PYTHON/MakeaMoveUI_Mockup/index.html` (CSS: add `.frame`, change `.ladder`; HTML: wrap foreground)

**Interfaces:**
- Produces: a `<div class="frame">` element wrapping `.hud`, `.toast`, `.center`, `.host`, `.ladder`, `.controls`. `.frame` is `position:relative` so the score ladder positions against it. Later tasks rely on the same element IDs (`#panel`, `#pct`, `#fill`, `#btnMove`, `#btnStay`, `#overlay`) — wrapping does not change IDs.

- [ ] **Step 1: Define the browser assertion (the "failing test")**

The check, to be run via Playwright `browser_evaluate` after navigating to the file at 1280×800:

```js
() => {
  const f = document.querySelector('.frame');
  if (!f) return { ok:false, reason:'no .frame element' };
  const r = f.getBoundingClientRect();
  const vw = window.innerWidth;
  const ladder = document.querySelector('.ladder').getBoundingClientRect();
  return {
    ok: Math.abs(r.left/vw - 0.10) < 0.02 &&
        Math.abs(r.width/vw - 0.80) < 0.02 &&
        ladder.left >= 0.10*vw - 1 && ladder.right <= 0.90*vw + 1,
    frameLeftPct: +(r.left/vw).toFixed(3),
    frameWidthPct: +(r.width/vw).toFixed(3),
    ladderLeftPct: +(ladder.left/vw).toFixed(3),
    ladderRightPct: +(ladder.right/vw).toFixed(3)
  };
}
```

- [ ] **Step 2: Run it to confirm it currently fails**

Via Playwright MCP: `browser_navigate` to `file:///mnt/e/TOOLMAKER/PYTHON/MakeaMoveUI_Mockup/index.html`, `browser_resize` to 1280×800, then `browser_evaluate` with the snippet above.
Expected: `{ ok:false, reason:'no .frame element' }` — there is no `.frame` yet.

- [ ] **Step 3: Add the `.frame` CSS rule and reposition the ladder**

In the `<style>` block, add a `.frame` rule. Insert it just before the `/* ---------- HUD ---------- */` comment:

```css
  /* ---------- 80% CONTENT FRAME ---------- */
  .frame{position:relative; z-index:5; width:80%; max-width:1200px; margin:0 auto;
    height:100%; display:flex; flex-direction:column}
```

Change the existing `.ladder` rule. Replace:

```css
  .ladder{position:absolute; left:22px; top:50%; transform:translateY(-50%); z-index:5;
    width:74px; display:flex; flex-direction:column; align-items:center; gap:8px}
```

with (anchors to the right edge of `.frame` instead of `left:22px`, keeping it inside the middle 80%):

```css
  .ladder{position:absolute; right:0; top:50%; transform:translateY(-50%); z-index:5;
    width:74px; display:flex; flex-direction:column; align-items:center; gap:8px}
```

- [ ] **Step 4: Wrap the foreground content in `.frame`**

In the HTML, the foreground elements (`.hud` … `.controls`) are currently direct children of `.stage`. Wrap them. Find this line (just after the decorative `<div class="rail"></div>`):

```html
  <div class="rail"></div>

  <!-- HUD -->
```

Replace it with:

```html
  <div class="rail"></div>

  <div class="frame">
  <!-- HUD -->
```

Then find the end of the controls block followed by the end overlay:

```html
  </div>

  <!-- END OVERLAY -->
```

Replace it with (closing the `.frame`; the `.overlay` stays full-screen, outside the frame):

```html
  </div>
  </div>

  <!-- END OVERLAY -->
```

(Note: the second `</div>` closes `.frame`; the existing first `</div>` closes `.controls`.)

- [ ] **Step 5: Run the assertion to confirm it passes**

Repeat the Playwright steps from Step 2 (navigate, resize 1280×800, evaluate).
Expected: `ok: true`, `frameLeftPct ≈ 0.10`, `frameWidthPct ≈ 0.80`, ladder within `[0.10, 0.90]`.
Also `browser_take_screenshot` and visually confirm the backdrop still fills the whole screen while content is centered in the middle 80%.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: lay foreground content within centered 80% frame"
```

---

### Task 2: Pink "correct answer" reveal on Move

**Files:**
- Modify: `/mnt/e/TOOLMAKER/PYTHON/MakeaMoveUI_Mockup/index.html` (CSS: `.panel.flip-move`; JS: set/reset the answer label)

**Interfaces:**
- Consumes: the `.frame` wrapper from Task 1 (no signature dependency).
- Produces: on Move, `#panel` gains class `flip-move` with a solid pink background, and the `.ans-label` inside `#panel` reads `✓ CORRECT`. `render()` resets that label to `David's answer`. Later tasks (3, 4) call `react('move'|'stay', ev)` and `render()` and rely on this behavior remaining intact.

- [ ] **Step 1: Define the browser assertion (the "failing test")**

After clicking Move, the answer panel's resolved background must contain the `--move` pink (`rgb(255, 45, 111)`), not the original blue panel gradient. Assertion snippet:

```js
() => {
  const p = document.getElementById('panel');
  const bg = getComputedStyle(p).backgroundImage;
  const label = p.querySelector('.ans-label').textContent.trim();
  return {
    isPink: bg.includes('255, 45, 111'),
    label,
    bg
  };
}
```

- [ ] **Step 2: Run it to confirm it currently fails**

Via Playwright: navigate + resize, then `browser_click` the Move button (`#btnMove`), then `browser_evaluate` the snippet.
Expected (current build): `isPink: false` (the panel keeps its blue gradient; `flip-move` only adds a glow), `label: "David's answer"`.

- [ ] **Step 3: Restyle `.panel.flip-move` to fill pink**

Replace the existing rule:

```css
  .panel.flip-move{transform:translateY(-4px) scale(1.015); border-color:var(--move); box-shadow:0 0 0 2px rgba(255,45,111,.5), 0 0 60px rgba(255,45,111,.45)}
```

with (adds a solid pink background and keeps the label/answer text readable on pink):

```css
  .panel.flip-move{transform:translatey(-4px) scale(1.015); border-color:var(--move);
    background:linear-gradient(180deg, var(--move-soft), var(--move));
    box-shadow:0 0 0 2px rgba(255,45,111,.5), 0 0 60px rgba(255,45,111,.45)}
  .panel.flip-move .ans-label{color:rgba(255,255,255,.9)}
  .panel.flip-move .ans{color:#fff}
```

(Fix the typo if pasted: the transform value must be `translateY`, capital Y. Use exactly: `transform:translateY(-4px) scale(1.015);`.)

- [ ] **Step 4: Mark the answer "correct" on Move, reset on render**

In the `<script>`, the `react()` function handles Move. Find the Move branch:

```js
  if(kind==='move'){
    earnedW += item.w;
    $('panel').classList.add('flip-move');
    pip.classList.add('move');
    const r=ev.currentTarget.getBoundingClientRect();
    hearts(r.left+r.width/2, r.top+r.height/2);
  }else{
```

Add a line setting the label after adding the class (the `earnedW += item.w;` line is replaced in Task 3 — leave it for now):

```js
  if(kind==='move'){
    earnedW += item.w;
    $('panel').classList.add('flip-move');
    $('panel').querySelector('.ans-label').textContent='✓ CORRECT';
    pip.classList.add('move');
    const r=ev.currentTarget.getBoundingClientRect();
    hearts(r.left+r.width/2, r.top+r.height/2);
  }else{
```

In `render()`, the panel is reset each question. Find:

```js
  const panel=$('panel'); panel.className='panel'; void panel.offsetWidth; panel.classList.add('pop');
```

Immediately after it, reset the label back:

```js
  const panel=$('panel'); panel.className='panel'; void panel.offsetWidth; panel.classList.add('pop');
  panel.querySelector('.ans-label').textContent="David's answer";
```

- [ ] **Step 5: Run the assertion to confirm it passes**

Repeat Step 2's Playwright flow (navigate, resize, click `#btnMove`, evaluate).
Expected: `isPink: true`, `label: "✓ CORRECT"`. `browser_take_screenshot` to confirm the panel reads as a pink "correct" reveal with legible text.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: reveal answer as pink correct panel on Move"
```

---

### Task 3: Flat 5-point scoring, 70%-of-total win line

**Files:**
- Modify: `/mnt/e/TOOLMAKER/PYTHON/MakeaMoveUI_Mockup/index.html` (JS: `QUESTIONS`, scoring constants, `react`, `updateScore`, reset handler; HTML: points readout + dynamic counter total; CSS: points readout style)

**Interfaces:**
- Consumes: `react('move'|'stay', ev)` and the `#panel` reveal from Task 2.
- Produces: globals `POINTS_PER_Q = 5`, `TOTAL_POINTS = QUESTIONS.length * 5`, and `earned` (points). `updateScore()` computes `pct = Math.round(earned / TOTAL_POINTS * 100)` and writes `#pct`, `#fill`, `#pts`. `end()` (rewritten in Task 4) reads `earned`, `pct`, `TOTAL_POINTS`. Removes `askedW`/`earnedW`/`item.w` usage.

- [ ] **Step 1: Define the browser assertion (the "failing test")**

With 7 questions (35 total points), one Move then one Stay should leave the meter at `round(5/35*100) = 14%`, points readout `5 / 35 PTS`, and the headline `14%`. Assertion snippet (runs after two reactions):

```js
() => {
  const pct = document.getElementById('pct').textContent.replace(/\s+/g,' ').trim();
  const fill = document.getElementById('fill').style.height;
  const ptsEl = document.getElementById('pts');
  return {
    hasPtsEl: !!ptsEl,
    pts: ptsEl ? ptsEl.textContent.trim() : null,
    pctText: pct,
    fillHeight: fill
  };
}
```

- [ ] **Step 2: Run it to confirm it currently fails**

Via Playwright: navigate + resize, `browser_click` `#btnMove`, wait ~1s for the auto-advance, `browser_click` `#btnStay`, then `browser_evaluate`.
Expected (current build): `hasPtsEl: false` (no `#pts` element exists), and `pctText` reflects the old weighted math, not `14%`.

- [ ] **Step 3: Strip the `w` weighting from the question bank**

Replace the entire `QUESTIONS` array:

```js
const QUESTIONS = [
  {dim:"Family",    depth:"Screening",  w:3.0, q:"Do you want children of your own someday?",
   a:"\"Yes — two, ideally. I'm a deputy head; honestly, kids are my whole world already.\""},
  {dim:"Family",    depth:"Exploratory", w:3.0, q:"What role should extended family play in raising kids?",
   a:"\"Close, but boundaried. Sunday dinners — not daily oversight of how we parent.\""},
  {dim:"Values",    depth:"Screening",  w:1.5, q:"What matters most to you in a partner?",
   a:"\"Honesty. And someone who's kind when absolutely no one is watching.\""},
  {dim:"Family",    depth:"Exploratory", w:3.0, q:"How would you blend two families together?",
   a:"\"Slowly. The kids set the pace, not the adults. You can't rush trust.\""},
  {dim:"Family",    depth:"Deep dive",  w:3.0, l3:true, q:"Your partner's teenage child rejects you outright. What happens?",
   a:"\"I stay consistent and patient. I was a stepkid — rejection is just fear. I'd earn it over years, not force it in weeks.\""},
  {dim:"Lifestyle", depth:"Screening",  w:0.9, q:"Describe your ideal Saturday.",
   a:"\"Park with the kids in the morning, then a proper grown-up dinner out in the evening.\""},
  {dim:"Ambition",  depth:"Screening",  w:0.5, q:"Where do you want to be in five years?",
   a:"\"Head teacher — and a calmer, fuller home life. Less hustle, more meaning.\""},
];
const TOTAL = QUESTIONS.length;
```

with (same content, `w` removed, scoring constants added):

```js
const QUESTIONS = [
  {dim:"Family",    depth:"Screening",  q:"Do you want children of your own someday?",
   a:"\"Yes — two, ideally. I'm a deputy head; honestly, kids are my whole world already.\""},
  {dim:"Family",    depth:"Exploratory", q:"What role should extended family play in raising kids?",
   a:"\"Close, but boundaried. Sunday dinners — not daily oversight of how we parent.\""},
  {dim:"Values",    depth:"Screening",  q:"What matters most to you in a partner?",
   a:"\"Honesty. And someone who's kind when absolutely no one is watching.\""},
  {dim:"Family",    depth:"Exploratory", q:"How would you blend two families together?",
   a:"\"Slowly. The kids set the pace, not the adults. You can't rush trust.\""},
  {dim:"Family",    depth:"Deep dive",  l3:true, q:"Your partner's teenage child rejects you outright. What happens?",
   a:"\"I stay consistent and patient. I was a stepkid — rejection is just fear. I'd earn it over years, not force it in weeks.\""},
  {dim:"Lifestyle", depth:"Screening",  q:"Describe your ideal Saturday.",
   a:"\"Park with the kids in the morning, then a proper grown-up dinner out in the evening.\""},
  {dim:"Ambition",  depth:"Screening",  q:"Where do you want to be in five years?",
   a:"\"Head teacher — and a calmer, fuller home life. Less hustle, more meaning.\""},
];
const TOTAL = QUESTIONS.length;
const POINTS_PER_Q = 5;
const TOTAL_POINTS = TOTAL * POINTS_PER_Q;
```

- [ ] **Step 4: Replace the score state variables**

Find:

```js
let idx=0, askedW=0, earnedW=0, stayByDim={}, locked=false;
```

Replace with (points-based `earned`; keep `stayByDim` for the escalation toast flavor):

```js
let idx=0, earned=0, stayByDim={}, locked=false;
```

- [ ] **Step 5: Add the points-readout element and dynamic counter total**

In the HTML ladder block, find:

```html
  <div class="ladder" id="ladder">
    <div class="pct" id="pct">0%<small>YOUR SCORE</small></div>
    <div class="tube"><div class="passline"></div><div class="fill" id="fill"></div></div>
  </div>
```

Replace with (adds the `#pts` readout):

```html
  <div class="ladder" id="ladder">
    <div class="pct" id="pct">0%<small>YOUR SCORE</small></div>
    <div class="tube"><div class="passline"></div><div class="fill" id="fill"></div></div>
    <div class="pts" id="pts">0 / 35 PTS</div>
  </div>
```

In the HUD, find the hardcoded `/ 10` counter:

```html
      <div class="counter">Q<b id="qnum">1</b> / 10 · First Half</div>
```

Replace with a dynamic total:

```html
      <div class="counter">Q<b id="qnum">1</b> / <span id="qtotal">10</span> · First Half</div>
```

Add the `.pts` style. In the `<style>` block, right after the `.ladder .pct small{...}` rule, add:

```css
  .pts{font-family:var(--display); font-weight:700; font-size:13px; letter-spacing:.1em;
    color:var(--ink-dim); text-transform:uppercase}
```

- [ ] **Step 6: Rewrite `react()` and `updateScore()` for point scoring**

Replace the whole `updateScore()` function:

```js
function updateScore(){
  const pctVal = askedW>0 ? Math.round((earnedW/askedW)*100) : 0;
  $('fill').style.height = pctVal+'%';
  if(window.matchMedia('(max-width:760px)').matches){$('fill').style.width=pctVal+'%';$('fill').style.height='100%';}
  $('pct').innerHTML = pctVal+'%<small>YOUR SCORE</small>';
  const passed = pctVal>=70;
  $('ladder').classList.toggle('passed',passed);
  $('fill').classList.toggle('passed',passed);
}
```

with (percentage of total bank points):

```js
function updateScore(){
  const pctVal = Math.round((earned/TOTAL_POINTS)*100);
  $('fill').style.height = pctVal+'%';
  if(window.matchMedia('(max-width:760px)').matches){$('fill').style.width=pctVal+'%';$('fill').style.height='100%';}
  $('pct').innerHTML = pctVal+'%<small>YOUR SCORE</small>';
  $('pts').textContent = earned+' / '+TOTAL_POINTS+' PTS';
  const passed = pctVal>=70;
  $('ladder').classList.toggle('passed',passed);
  $('fill').classList.toggle('passed',passed);
}
```

In `react()`, replace the scoring lines. Find:

```js
  const item=QUESTIONS[idx];
  askedW += item.w;
  const pip=pips.children[idx]; pip.classList.add('done');
  if(kind==='move'){
    earnedW += item.w;
    $('panel').classList.add('flip-move');
    $('panel').querySelector('.ans-label').textContent='✓ CORRECT';
```

Replace with (flat 5 points on Move; nothing accrued on Stay):

```js
  const item=QUESTIONS[idx];
  const pip=pips.children[idx]; pip.classList.add('done');
  if(kind==='move'){
    earned += POINTS_PER_Q;
    $('panel').classList.add('flip-move');
    $('panel').querySelector('.ans-label').textContent='✓ CORRECT';
```

- [ ] **Step 7: Update the "Play again" reset and the counter total init**

Find the restart handler:

```js
$('restart').addEventListener('click', ()=>{
  idx=0; askedW=0; earnedW=0; stayByDim={};
  $('overlay').className='overlay';
  [...pips.children].forEach(p=>p.className='pip');
  updateScore(); render();
});
```

Replace with (reset `earned`):

```js
$('restart').addEventListener('click', ()=>{
  idx=0; earned=0; stayByDim={};
  $('overlay').className='overlay';
  [...pips.children].forEach(p=>p.className='pip');
  updateScore(); render();
});
```

Find the bottom init line:

```js
render(); updateScore();
```

Replace with (set the dynamic counter total before first paint):

```js
$('qtotal').textContent = TOTAL;
render(); updateScore();
```

- [ ] **Step 8: Run the assertion to confirm it passes**

Repeat Step 2's Playwright flow (navigate, resize, click `#btnMove`, wait ~1s, click `#btnStay`, evaluate).
Expected: `hasPtsEl: true`, `pts: "5 / 35 PTS"`, `pctText: "14% YOUR SCORE"`, `fillHeight: "14%"`. Also confirm the HUD counter reads `/ 7`.

- [ ] **Step 9: Commit**

```bash
git add index.html
git commit -m "feat: score 5 points per Move against 70% of total points"
```

---

### Task 4: End-screen narrative in points/percentage terms

**Files:**
- Modify: `/mnt/e/TOOLMAKER/PYTHON/MakeaMoveUI_Mockup/index.html` (JS: `end()`)

**Interfaces:**
- Consumes: `earned`, `TOTAL_POINTS`, `POINTS_PER_Q` from Task 3.
- Produces: final `end()` deciding win/lose by `pctVal = round(earned/TOTAL_POINTS*100) >= 70`, with copy expressed in points and percentage (no weighting-model references).

- [ ] **Step 1: Define the browser assertion (the "failing test")**

Playing all 7 questions as **Stay** (0 Moves → 0%) must show a lose overlay whose message references the points/percentage result and contains no weighting language ("weigh", "Family ones stay"). Assertion snippet (run after the overlay appears):

```js
() => {
  const ov = document.getElementById('overlay');
  const msg = document.getElementById('endmsg').textContent;
  return {
    shown: ov.classList.contains('show'),
    lose: ov.classList.contains('lose'),
    bigpct: document.getElementById('bigpct').textContent,
    mentionsPoints: /points|PTS|%/.test(msg),
    hasWeightingLanguage: /weigh|Family ones stay/i.test(msg)
  };
}
```

- [ ] **Step 2: Run it to confirm it currently fails**

Via Playwright: navigate + resize, then click `#btnStay` 7 times (waiting ~1s between clicks for auto-advance), then `browser_evaluate`.
Expected (current build): `hasWeightingLanguage: true` — the lose copy still says "the thing you weigh most".

- [ ] **Step 3: Rewrite `end()`**

Replace the whole `end()` function:

```js
function end(){
  const pctVal = Math.round((earnedW/askedW)*100);
  const passed = pctVal>=70;
  const ov=$('overlay'); ov.classList.add('show'); ov.classList.add(passed?'win':'lose');
  $('verdict').textContent = passed ? 'You passed this half' : 'So close';
  $('bigpct').textContent = pctVal+'%';
  if(passed){
    $('endmsg').innerHTML = `Your half cleared the 70% line. <span class="partner">David scored 81% on his half too</span> — it's a match. Next stop: a 30-minute video call, then a real date.`;
    $('lifeWrap').innerHTML='';
    if(!reduced) hearts(window.innerWidth/2, window.innerHeight*0.4);
  }else{
    $('endmsg').innerHTML = `Two Stays landed on <span class="partner">Family</span> — the thing you weigh most — so eight other Moves couldn't buy it back. That's the model being honest, not unkind.`;
    $('lifeWrap').innerHTML = `<div class="lifeline">✦ You have one free retry — the questions David already won are swapped out; the Family ones stay.</div>`;
  }
}
```

with (points/percentage framing; `PASS_POINTS`/`PASS_MOVES` derived from the 70% line):

```js
function end(){
  const pctVal = Math.round((earned/TOTAL_POINTS)*100);
  const passed = pctVal>=70;
  const moves = earned/POINTS_PER_Q;
  const PASS_POINTS = Math.ceil(TOTAL_POINTS*0.70);
  const PASS_MOVES = Math.ceil(PASS_POINTS/POINTS_PER_Q);
  const ov=$('overlay'); ov.classList.add('show'); ov.classList.add(passed?'win':'lose');
  $('verdict').textContent = passed ? 'You passed this half' : 'So close';
  $('bigpct').textContent = pctVal+'%';
  if(passed){
    $('endmsg').innerHTML = `You scored <span class="partner">${earned} of ${TOTAL_POINTS} points (${pctVal}%)</span> and cleared the 70% line with ${moves} Moves. David scored 81% on his half too — it's a match. Next stop: a 30-minute video call, then a real date.`;
    $('lifeWrap').innerHTML='';
    if(!reduced) hearts(window.innerWidth/2, window.innerHeight*0.4);
  }else{
    $('endmsg').innerHTML = `You scored <span class="partner">${earned} of ${TOTAL_POINTS} points (${pctVal}%)</span>. The pass line is 70% — that's ${PASS_POINTS} points, or ${PASS_MOVES} Moves out of ${TOTAL}. You gave ${moves}. Close, but the bar held.`;
    $('lifeWrap').innerHTML = `<div class="lifeline">✦ You have one free retry — every question comes back, so a few more Moves clears it.</div>`;
  }
}
```

- [ ] **Step 4: Run the assertion to confirm it passes**

Repeat Step 2's Playwright flow (navigate, resize, 7× Stay with ~1s gaps, evaluate).
Expected: `shown: true`, `lose: true`, `bigpct: "0%"`, `mentionsPoints: true`, `hasWeightingLanguage: false`.

- [ ] **Step 5: Verify the win path**

Via Playwright: navigate + resize, then click `#btnMove` 5 times and `#btnStay` 2 times (any order, ~1s gaps) → `earned = 25`, `pct = round(25/35*100) = 71% ≥ 70`. `browser_evaluate`:

```js
() => {
  const ov = document.getElementById('overlay');
  return { win: ov.classList.contains('win'), bigpct: document.getElementById('bigpct').textContent };
}
```

Expected: `win: true`, `bigpct: "71%"`. `browser_take_screenshot` to confirm the win overlay.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: rewrite end-screen copy in points and percentage terms"
```

---

## Self-Review

**Spec coverage** (against `prompt.md` + design doc):
- Bluish background fills entire screen — preserved (backdrop stays outside `.frame`); confirmed visually in Task 1 Step 5.
- Middle 80% width, 10% padding each side — Task 1.
- Two buttons Move/Stay at the bottom — unchanged (already at bottom); colors fixed by Global Constraints (Stay gray `--stay`, Move pink `--move`).
- Move lights the answer as correct with a pink background, replacing white-on-blue — Task 2.
- Questions from a bank; each worth 5 points; Move awards 5; win at 70% of total — Task 3.
- "Stay" gray / "Move" pink — already in CSS (`--stay`/`--move`), unchanged.
- End-screen narrative matches the new scoring — Task 4.

**Placeholder scan:** No TBD/TODO; every code step shows full before/after content. One inline caution flagged (the `translateY` capitalization in Task 2 Step 3) with the exact correct value given.

**Type/name consistency:** `earned`, `POINTS_PER_Q`, `TOTAL_POINTS`, `TOTAL` introduced in Task 3 and reused identically in Tasks 3–4. `#pts`, `#qtotal`, `.frame`, `.panel.flip-move`, `.pts` are defined and referenced consistently. `askedW`/`earnedW`/`item.w` are fully removed across `QUESTIONS`, `react`, `updateScore`, `end`, and the restart handler — no stragglers. `react('move'|'stay', ev)` signature unchanged across tasks.
