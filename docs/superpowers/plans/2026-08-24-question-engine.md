# Question Engine & Authoring Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace free-text answers with MECE options plus an optional voice line, add a reusable custom-question deck with LLM review, and derive attribute confidence from question variations so confirmed truths land on `S-E6`.

**Architecture:** A single document-scope `window.MMQ` namespace holds the question bank, the user's deck, the two count models, the shared answer-card renderer and the confidence derivation. It is declared **before** `#screenwrap` because the inline scripts inside `S-D2` and `S-F3` run at parse time and consume it. Confidence is *derived* from how many variations of the same `probe` were answered consistently, never hardcoded, so editing a fixture answer moves the meter. State persists in `sessionStorage`.

**Tech Stack:** HTML + CSS + vanilla JavaScript, single file. Verification is browser-driven via the Playwright MCP tools.

**Spec:** `docs/superpowers/specs/2026-08-24-prompt2-decisions.md` — Cluster 1 (§1.1–1.7). Read it alongside this plan.

## Global Constraints

- **Single file.** All changes in `/mnt/e/TOOLMAKER/PYTHON/MakeaMoveUI_Mockup/index.html`. No new runtime files, no build step, no libraries.
- **Reuse existing components:** `.card`, `.row`, `.tag`, `.chip`, `.opt`, `.btn`, `.note`, `.field`, `.eyebrow`, `.muted`, `.prog` and the `:root` custom properties. New CSS only for the answer card, the author badge and the confidence block.
- **Theme both variants.** `S-D2`/`S-F3` are `dark`, the deck screens and `S-E6` involve `paper` surfaces. Every new component must be styled under both, per the standing convention.
- **`MMQ` must be defined before any screen's inline script runs.** Its `<script>` goes immediately before `<div class="screenwrap" id="screenwrap">` (~line 533).
- **The selected option is the sole scoring signal.** The optional `why` line is never scored, never required, and never blocks.
- **`why` is capped at 120 characters** and cards must render correctly with and without it.
- **Dimension weights are exactly** `Family 3.0, Values 1.5, Lifestyle 0.9, Finances 1.0, Ambition 0.6, Intimacy 1.4` — unchanged from the existing `W` in `S-D2`.
- **`MMQ.CONF_GATE = 70`.** A probe at or above it is "confirmed" and written to the profile; below it is "still forming".
- **`MMQ.MAX_DECK = 5`** and **`MMQ.ATTEMPTS = 3`** (replacement attempts *total per submission*, not per question).
- **Model A** = exactly 10 questions, up to 5 of them replaced by deck questions. **Model B** = 10 bank + up to 5 deck, max 15. Nothing may assume a fixed count.
- **Do not break:** the `S-D2` → `S-D5`/`S-D6` outcome routing via `sessionStorage` `mm_py`/`mm_pt`, the `S-F3` → `S-F4` transition, or the `DSHOW`/`FSHOW` reset entry points used by the router.
- **New screens `S-QD1`–`S-QD3`** must be registered: `S-QD1` is a list screen and belongs in the `FEED` set in `classify()`.

## Deferred out of this phase

**§1.5 (mirrored async pairs revealed at `S-F4`) is not implemented here.** Phase 2 rebuilds `S-F4` wholesale for the async restructure; building the mirror block now would be thrown away. This plan builds the *data* that §1.5 needs — the `mirror` field on every question and `MMQ.mirrorPairs()` — and Phase 2 renders it. Task 1 covers the data; no rendering task exists for it.

## Verification setup

`file://` is blocked by the Playwright MCP browser. Serve the project once, before Task 1:

```bash
cd /mnt/e/TOOLMAKER/PYTHON/MakeaMoveUI_Mockup && python3 -m http.server 3300
```

All assertions run at **1280×800** against `http://127.0.0.1:3300/index.html#<screen>`. Every "run the check" step means: `browser_navigate` to that URL, `browser_resize` to 1280×800, then `browser_evaluate` the given function.

**Never call `location.reload()` inside an evaluated function** — it destroys the execution context mid-call and the evaluate fails with "Execution context was destroyed" rather than returning a result. When an assertion needs fresh state, reset it from *outside*: `browser_navigate` to a cache-busted URL, then a separate `browser_evaluate` running `sessionStorage.clear()`, then run the assertion body.

Because the router is hash-based, switching screens by assigning `location.hash` does not reload the page.

## File Structure

One file, five insertion points:

- **CSS block** (`<style>`, lines ~11–460): append a "question engine" section after the existing `.opt` rules. Adds `.anscard2`, `.qauthor`, `.confrow`, `.devpanel`.
- **New `<script id="mmq">`** immediately before `<div class="screenwrap">` (~line 533): the whole `MMQ` namespace. Everything else consumes it.
- **`S-D2`** (~lines 1673–1848): `TURNS` is replaced by a set built from `MMQ`; the `.ans` markup is replaced by `MMQ.answerCard()`.
- **`S-F3`** (~lines 2068–2199): `QF` replaced likewise; slot count becomes dynamic.
- **New screens `S-QD1`–`S-QD3`** inserted after `S-P5` (~line 981), plus attach cards in `S-C1` and `S-F2`, plus the confidence block in `S-E6`.

---

### Task 1: The MMQ namespace — bank, deck, models, confidence

**Files:**
- Modify: `index.html` — insert a new `<script id="mmq">` immediately before `<div class="screenwrap" id="screenwrap">` (~line 533).

**Interfaces:**
- Produces on `window`: `MMQ` with `DIMS`, `BANK` (15 questions), `CONF_GATE`, `MAX_DECK`, `ATTEMPTS`, `DEMO`, and the functions `qById(id)`, `deck()`, `saveDeck(arr)`, `model()`, `setModel(m)`, `buildSet(opts)`, `confidence(probe, answers)`, `confLabel(n)`, `probes()`, `mirrorPairs(set)`, `reviewQuestion(text)`, `genOptions(text)`, `answers()`. Every later task calls these by exactly these names.

- [ ] **Step 1: Write the browser assertion**

Run this first, before writing any code, so you see it fail:

```js
() => {
  if (!window.MMQ) return { ok:false, reason:'MMQ not defined' };
  const M = window.MMQ;
  sessionStorage.removeItem('mm_deck');
  sessionStorage.removeItem('mm_qmodel');

  // every bank question is well formed
  const malformed = M.BANK.filter(q =>
    !q.id || !q.dim || !q.probe || !q.text ||
    !Array.isArray(q.opts) || q.opts.length !== 4 ||
    q.opts.some(o => !o.t || !o.v) ||
    !(q.dim in M.DIMS));

  // Model A is always exactly 10; deck questions displace bank ones
  const a0 = M.buildSet({ model:'A', deck:[] });
  const a5 = M.buildSet({ model:'A', deck:M.BANK.slice(0,5).map(q=>({...q, id:'c'+q.id, src:'custom', author:'You'})) });
  // Model B is 10 + deck, capped at 15
  const b0 = M.buildSet({ model:'B', deck:[] });
  const b5 = M.buildSet({ model:'B', deck:M.BANK.slice(0,5).map(q=>({...q, id:'c'+q.id, src:'custom', author:'You'})) });

  // confidence rises with corroborating variations
  const conf = {
    children: M.confidence('wants-children', M.DEMO.answers),
    value:    M.confidence('core-value',     M.DEMO.answers),
    weekend:  M.confidence('weekend',        M.DEMO.answers),
    money:    M.confidence('money-model',    M.DEMO.answers),
    absent:   M.confidence('no-such-probe',  M.DEMO.answers)
  };

  return {
    ok: malformed.length === 0 &&
        M.BANK.length === 15 &&
        a0.length === 10 && a5.length === 10 &&
        a5.filter(q=>q.src==='custom').length === 5 &&
        b0.length === 10 && b5.length === 15 &&
        conf.children === 100 && conf.value === 67 &&
        conf.weekend === 33 && conf.money === 33 && conf.absent === null &&
        M.confLabel(100) === 'Confirmed' && M.confLabel(67) === 'Still forming' &&
        M.CONF_GATE === 70,
    malformed: malformed.map(q=>q.id),
    lens: { a0:a0.length, a5:a5.length, b0:b0.length, b5:b5.length },
    customInA5: a5.filter(q=>q.src==='custom').length,
    conf
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `http://127.0.0.1:3300/index.html#S-D2`, resize 1280×800, evaluate the above.
Expected: `{ ok:false, reason:'MMQ not defined' }`.

- [ ] **Step 3: Insert the MMQ script**

Insert immediately before `<div class="screenwrap" id="screenwrap">`:

```html
<script id="mmq">
/* ===== MMQ · THE QUESTION ENGINE =====
   Shared by S-D2 (the arena), S-F3 (the async board), S-C1/S-F2 (deck attach),
   S-QD1-3 (deck authoring) and S-E6 (confidence). Declared BEFORE #screenwrap
   because those screens' inline scripts run at parse time and consume it.

   Spec: docs/superpowers/specs/2026-08-24-prompt2-decisions.md, Cluster 1.
   Fixtures, not a real engine. Confidence is DERIVED from how many variations
   of the same probe were answered consistently, so editing a demo answer
   moves the meter on S-E6. */
(function(){
  const DIMS = { Family:3.0, Values:1.5, Lifestyle:0.9, Finances:1.0, Ambition:0.6, Intimacy:1.4 };

  const CONF_GATE = 70;   // at or above: confirmed, written to the profile
  const MAX_DECK  = 5;    // custom questions per user (spec 1.6)
  const ATTEMPTS  = 3;    // replacement attempts TOTAL per submission (spec 1.3)

  /* o(text, value) — value is the normalised answer, so two differently
     worded questions on the same probe can be compared for consistency. */
  const o = (t,v) => ({ t, v });

  /* Every question carries a `probe`: the underlying attribute it measures.
     Several questions share a probe — those are the "variations" of spec 1.8
     that let confidence be triangulated rather than taken on one answer. */
  const BANK = [
    { id:'q-fam-01', dim:'Family', depth:'Screening', probe:'wants-children',
      text:'Do you want children of your own someday?',
      opts:[o('Yes — more than one','yes'), o('Yes — one','yes'),
            o('Open, not set on it','open'), o('No','no')] },

    { id:'q-fam-02', dim:'Family', depth:'Screening', probe:'children-timing',
      text:'When would you want your first child?',
      opts:[o('Within 2 years of marriage','soon'), o('3–5 years in','later'),
            o('Open — no fixed timeline','open'), o("I don't want children",'no')] },

    { id:'q-fam-03', dim:'Family', depth:'Exploratory', probe:'wants-children',
      text:'If a partner already had children, would you still want your own?',
      opts:[o('Yes, definitely','yes'), o('Yes, if they were open to it','yes'),
            o("I'd be content either way",'open'), o('No — theirs would be enough','no')] },

    { id:'q-fam-04', dim:'Family', depth:'Exploratory', probe:'blending',
      text:'How would you blend two families together?',
      opts:[o('Slowly — the kids set the pace','slow'), o('Together from day one','fast'),
            o('Separate homes at first','separate'), o("I'd follow my partner's lead",'defer')] },

    { id:'q-fam-05', dim:'Family', depth:'Deep dive', probe:'step-resilience',
      text:"A partner's teenager rejects you outright. What happens?",
      opts:[o("I stay patient and earn it over years",'persist'),
            o('I give them space and let my partner lead','defer'),
            o("I'd want family counselling",'support'), o("I'd struggle to stay",'exit')] },

    { id:'q-fam-06', dim:'Family', depth:'Deep dive', probe:'wants-children',
      text:"If you learned you couldn't have biological children, what then?",
      opts:[o("We'd adopt — I want to raise children",'yes'), o("We'd try every route",'yes'),
            o("I'd make peace with it",'open'), o("I'd be relieved",'no')] },

    { id:'q-val-01', dim:'Values', depth:'Screening', probe:'core-value',
      text:'What matters most to you in a partner?',
      opts:[o('Honesty','honesty'), o('Kindness','kindness'),
            o('Ambition','ambition'), o('Faith','faith')] },

    { id:'q-val-02', dim:'Values', depth:'Screening', probe:'non-negotiable',
      text:'What would you never compromise on?',
      opts:[o('Time with my family','family'), o('My faith','faith'),
            o('My career','career'), o('My independence','independence')] },

    { id:'q-val-03', dim:'Values', depth:'Exploratory', probe:'core-value',
      text:'Which would you forgive more easily?',
      opts:[o('A painful truth','honesty'), o('A kind lie','kindness'),
            o('Neither, honestly','neither'), o('It would depend entirely','depends')] },

    { id:'q-lif-01', dim:'Lifestyle', depth:'Screening', probe:'weekend',
      text:'Describe your ideal Saturday.',
      opts:[o('Out with family','family'), o('Quiet at home','home'),
            o('Somewhere new','adventure'), o('Working on something','work')] },

    { id:'q-lif-02', dim:'Lifestyle', depth:'Screening', probe:'weekend',
      text:'Describe your perfect weekend.',
      opts:[o('A long lunch with people I love','family'), o('Nothing planned at all','home'),
            o('A trip somewhere unfamiliar','adventure'), o('Deep in a project','work')] },

    { id:'q-fin-01', dim:'Finances', depth:'Screening', probe:'money-model',
      text:'What does financial partnership look like to you?',
      opts:[o('Fully joint — one pot','joint'), o('Joint plan, separate accounts','hybrid'),
            o('Mostly separate','separate'), o('Whoever earns, manages','earner')] },

    { id:'q-fin-02', dim:'Finances', depth:'Exploratory', probe:'money-model',
      text:'How do you handle money in a relationship?',
      opts:[o('Open books, one budget','joint'), o('Shared goals, own accounts','hybrid'),
            o('We each cover our own','separate'), o("I'd rather not manage it",'earner')] },

    { id:'q-amb-01', dim:'Ambition', depth:'Screening', probe:'five-year',
      text:'Where do you want to be in five years?',
      opts:[o('Further along in my career','career'), o('Settled with a family','family'),
            o('Somewhere new entirely','change'), o('Much as I am now','steady')] },

    { id:'q-int-01', dim:'Intimacy', depth:'Screening', probe:'closeness',
      text:'What does closeness look like for you?',
      opts:[o('Small daily rituals','rituals'), o('Long honest conversations','talk'),
            o('Physical affection','touch'), o('Being in the same room','presence')] }
  ];

  /* Questions sharing a probe are each other's mirror in async play, so a
     Move/Stay from one party reads against the other's answer on the same
     attribute (spec 1.5). Phase 2 renders this; Phase 1 only supplies it. */
  BANK.forEach(q => { q.src = 'bank'; q.author = null; q.mirror = q.probe; });

  const qById = id => BANK.concat(deck()).find(q => q.id === id) || null;

  /* ---- persisted state ---- */
  const read = (k, d) => { try { return JSON.parse(sessionStorage.getItem(k)) ?? d; }
                           catch(e){ return d; } };
  const deck      = () => read('mm_deck', []);
  const saveDeck  = a  => sessionStorage.setItem('mm_deck', JSON.stringify(a.slice(0,MAX_DECK)));
  const model     = () => sessionStorage.getItem('mm_qmodel') || 'A';
  const setModel  = m  => sessionStorage.setItem('mm_qmodel', m === 'B' ? 'B' : 'A');
  const answers   = () => read('mm_answers', DEMO.answers);

  /* ---- the two count models (spec 1.2) ----
     A: exactly 10, deck questions DISPLACE bank ones.
     B: 10 bank PLUS the deck, capped at 15.                                */
  function buildSet(opts){
    opts = opts || {};
    const m = opts.model || model();
    const d = (opts.deck || deck()).slice(0, MAX_DECK);
    if (m === 'B') return BANK.slice(0,10).concat(d);
    return d.concat(BANK.slice(0, 10 - d.length));
  }

  /* ---- confidence (spec 1.4 / item 8) ----
     Consistency across variations, damped by how few variations exist.
     3 consistent answers on a probe -> 100. 2 -> 67. 1 -> 33.               */
  function confidence(probe, ans){
    ans = ans || answers();
    const hits = ans.filter(a => { const q = qById(a.qid); return q && q.probe === probe; });
    if (!hits.length) return null;
    const vals = hits.map(a => { const q = qById(a.qid); return q.opts[a.opt].v; });
    const tally = {};
    vals.forEach(v => { tally[v] = (tally[v]||0) + 1; });
    const agree = Math.max.apply(null, Object.values(tally));
    return Math.round((agree / hits.length) * Math.min(1, hits.length / 3) * 100);
  }
  const confLabel = n => n == null ? 'No data'
                       : n >= CONF_GATE ? 'Confirmed' : 'Still forming';

  const probes = () => [...new Set(BANK.map(q => q.probe))];

  /* Pairs of questions in a set that probe the same attribute (spec 1.5). */
  function mirrorPairs(set){
    const by = {};
    (set || buildSet()).forEach(q => { (by[q.probe] = by[q.probe] || []).push(q.id); });
    return Object.entries(by).filter(([,ids]) => ids.length > 1)
                             .map(([probe, ids]) => ({ probe, ids }));
  }

  /* ---- the fake LLM (spec 1.3) — deterministic, so it is testable ---- */
  const BLOCKED = ['salary','virgin','body count','how much do you earn','net worth'];
  function reviewQuestion(text){
    const t = String(text || '').trim();
    const low = t.toLowerCase();
    if (t.length < 12)                  return { ok:false, reason:'Too vague to answer meaningfully.' };
    if (t.indexOf('?') === -1)          return { ok:false, reason:'Write it as a question.' };
    const hit = BLOCKED.find(b => low.includes(b));
    if (hit)                            return { ok:false, reason:'Not appropriate before you have met.' };
    return { ok:true, reason:'Cleared' };
  }

  /* Options the system generates for a user's custom question (spec 1.2/2). */
  function genOptions(text){
    const low = String(text || '').toLowerCase().trim();
    if (/^(how many|when|how soon|how often)/.test(low))
      return [o('Very soon','soon'), o('Within a few years','later'),
              o('No fixed timeline','open'), o('Never / not at all','no')];
    if (/^(would|do|did|can|could|have|are|is|will)\b/.test(low))
      return [o('Yes, without hesitation','yes'), o('Yes, with conditions','yes-if'),
              o('I am open to it','open'), o('No','no')];
    return [o('Very important to me','high'), o('Somewhat important','mid'),
            o('Neutral','neutral'), o('Not important to me','low')];
  }

  /* ---- demo fixtures ----
     Answers are what S-E6 reads when nothing has been played yet.
     wants-children: 3 consistent -> 100 (confirmed)
     core-value:     2 consistent -> 67
     weekend:        2 conflicting -> 33
     money-model:    1 answer     -> 33                                     */
  const DEMO = {
    answers: [
      { qid:'q-fam-01', opt:0, why:'I always pictured a bigger, noisier house.' },
      { qid:'q-fam-03', opt:1, why:'' },
      { qid:'q-fam-06', opt:0, why:'' },
      { qid:'q-val-01', opt:0, why:'Honesty. And kindness when no one is watching.' },
      { qid:'q-val-03', opt:0, why:'' },
      { qid:'q-lif-01', opt:0, why:'Park with the kids, then a proper dinner out.' },
      { qid:'q-lif-02', opt:1, why:'' },
      { qid:'q-fin-01', opt:1, why:'Shared goals, honest numbers, some fun money.' }
    ],
    /* David's recorded answers, keyed by question id, for the arena. */
    them: {
      'q-fam-01': { opt:0, why:"Two, ideally. I'm a deputy head; kids are my world." },
      'q-fam-02': { opt:0, why:'' },
      'q-fam-03': { opt:0, why:'' },
      'q-fam-04': { opt:0, why:'The kids set the pace, not the adults.' },
      'q-fam-05': { opt:0, why:'I was a stepkid. Rejection is fear.' },
      'q-fam-06': { opt:1, why:'' },
      'q-val-01': { opt:0, why:'' },
      'q-val-02': { opt:0, why:'Sunday dinner with my mum. Non-negotiable.' },
      'q-val-03': { opt:0, why:'' },
      'q-lif-01': { opt:0, why:'Park with the kids, then a grown-up dinner.' },
      'q-lif-02': { opt:0, why:'' },
      'q-fin-01': { opt:1, why:'Shared goals, honest numbers, a little fun money.' },
      'q-fin-02': { opt:1, why:'I budget like an architect — everything load-bearing.' },
      'q-amb-01': { opt:0, why:'Head teacher — and a calmer, fuller home life.' },
      'q-int-01': { opt:0, why:'Tea made without asking. A hand on the shoulder.' }
    },
    /* David's verdict on each of YOUR answers, keyed by question id. */
    verdict: {
      'q-fam-01':'move', 'q-fam-02':'move', 'q-fam-03':'move', 'q-fam-04':'move',
      'q-fam-05':'move', 'q-fam-06':'move', 'q-val-01':'move', 'q-val-02':'move',
      'q-val-03':'stay', 'q-lif-01':'move', 'q-lif-02':'stay', 'q-fin-01':'move',
      'q-fin-02':'move', 'q-amb-01':'move', 'q-int-01':'move'
    }
  };

  window.MMQ = { DIMS, BANK, DEMO, CONF_GATE, MAX_DECK, ATTEMPTS,
                 qById, deck, saveDeck, model, setModel, answers,
                 buildSet, confidence, confLabel, probes, mirrorPairs,
                 reviewQuestion, genOptions, o };
})();
</script>
```

- [ ] **Step 4: Run the assertion to verify it passes**

Re-evaluate the Step 1 function.
Expected: `ok: true`, `lens: {a0:10, a5:10, b0:10, b5:15}`, `customInA5: 5`, `conf: {children:100, value:67, weekend:33, money:33, absent:null}`.

If `conf.children` is not 100, check that `q-fam-01`, `q-fam-03` and `q-fam-06` all resolve to `v:'yes'` at the demo answer indices 0, 1 and 0 respectively.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(questions): add MMQ engine — bank, deck store, count models, confidence"
```

---

### Task 2: Shared answer card and author badge

**Files:**
- Modify: `index.html` — append CSS to the main `<style>` block after the `.opt` rules; add `answerCard` to the `MMQ` namespace in the `<script id="mmq">` from Task 1.

**Interfaces:**
- Consumes: `MMQ.qById`, `MMQ.o` from Task 1.
- Produces: `MMQ.answerCard(q, ans, label)` returning an HTML string. Tasks 3 and 4 both call it — this is the DRY point for spec 1.1 and 1.7.

- [ ] **Step 1: Write the browser assertion**

```js
() => {
  const M = window.MMQ;
  if (!M || !M.answerCard) return { ok:false, reason:'MMQ.answerCard not defined' };
  const q = M.qById('q-fam-01');
  const custom = { ...q, id:'c1', src:'custom', author:'Samuel' };

  const withWhy = M.answerCard(q, { qid:'q-fam-01', opt:0, why:'A noisier house.' }, 'David said');
  const bare    = M.answerCard(q, { qid:'q-fam-01', opt:2, why:'' }, 'David said');
  const authored= M.answerCard(custom, { qid:'c1', opt:0, why:'' }, 'David said');

  const host = document.createElement('div');
  host.innerHTML = withWhy;
  const optText = host.querySelector('.ac-opt').textContent.trim();

  return {
    ok: optText === 'Yes — more than one' &&
        withWhy.includes('A noisier house.') &&
        withWhy.includes('ac-why') &&
        !bare.includes('ac-why') &&
        bare.includes('Open, not set on it') &&
        authored.includes('qauthor') &&
        authored.includes('Samuel asked this') &&
        !withWhy.includes('qauthor'),
    optText,
    bareHasWhy: bare.includes('ac-why'),
    authoredHasBadge: authored.includes('qauthor')
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `#S-D2`, evaluate.
Expected: `{ ok:false, reason:'MMQ.answerCard not defined' }`.

- [ ] **Step 3: Add the CSS**

Append to the main `<style>` block, after the existing `.opt` rules:

```css
/* ===== question engine · answer card (spec 1.1) =====
   Rendered on dark stage surfaces (S-D2, S-F3) and paper ones (S-E6 deep
   dive), so every colour resolves through a variable that both themes set. */
.anscard2{position:relative; border-radius:12px; padding:12px 14px;
  background:rgba(255,255,255,.05); border:1px solid rgba(255,182,39,.35);
  text-align:left; max-width:34ch}
.paper .anscard2{background:var(--paper-2); border-color:var(--line)}
.anscard2 .lb{font-size:9.5px; letter-spacing:.16em; text-transform:uppercase;
  color:var(--ink-faint); font-weight:700; margin-bottom:6px}
.paper .anscard2 .lb{color:var(--t-faint)}
/* the selected option is the primary element — it is the scoring signal */
.anscard2 .ac-opt{font-weight:800; font-size:15px; line-height:1.3}
/* the optional voice line, secondary and italic, absent when not written */
.anscard2 .ac-why{margin-top:7px; font-size:12.5px; line-height:1.45;
  font-style:italic; color:var(--ink-dim)}
.paper .anscard2 .ac-why{color:var(--t-dim)}
.anscard2 .ac-why::before{content:'\201C'} .anscard2 .ac-why::after{content:'\201D'}
/* spec 1.7 — a custom question is visibly the other party's */
.qauthor{display:inline-flex; align-items:center; gap:5px; font-family:var(--display);
  font-weight:700; font-size:9.5px; letter-spacing:.14em; text-transform:uppercase;
  color:var(--move); background:rgba(255,45,111,.14); border:1px solid rgba(255,45,111,.4);
  border-radius:999px; padding:3px 9px; margin-bottom:7px}
```

- [ ] **Step 4: Add the renderer**

Inside the `<script id="mmq">` IIFE, immediately before the `window.MMQ = {...}` line, add:

```js
  /* The one place an answer becomes markup. S-D2 and S-F3 both call this so
     the option-primary / voice-secondary treatment cannot drift apart. */
  function answerCard(q, ans, label){
    const opt   = q.opts[ans.opt];
    const badge = q.src === 'custom' && q.author
      ? '<span class="qauthor"><svg class="icon" style="width:11px;height:11px">'
        + '<use href="#i-heart"/></svg>' + q.author + ' asked this</span>'
      : '';
    const why   = ans.why
      ? '<div class="ac-why">' + ans.why + '</div>'
      : '';
    return '<div class="anscard2">' + badge
         + '<div class="lb">' + label + '</div>'
         + '<div class="ac-opt">' + opt.t + '</div>'
         + why + '</div>';
  }
```

Then add `answerCard` to the exported object:

```js
  window.MMQ = { DIMS, BANK, DEMO, CONF_GATE, MAX_DECK, ATTEMPTS,
                 qById, deck, saveDeck, model, setModel, answers,
                 buildSet, confidence, confLabel, probes, mirrorPairs,
                 reviewQuestion, genOptions, answerCard, o };
```

- [ ] **Step 5: Run the assertion to verify it passes**

Re-evaluate Step 1. Expected: `ok: true`, `optText: 'Yes — more than one'`, `bareHasWhy: false`, `authoredHasBadge: true`.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(questions): add shared answer card with author badge"
```

---

### Task 3: The arena (S-D2) reads from MMQ

**Files:**
- Modify: `index.html` — the inline `<script>` inside `S-D2` (~lines 1770–1847), and the `.ans` usage inside `render()`.

**Interfaces:**
- Consumes: `MMQ.buildSet`, `MMQ.answerCard`, `MMQ.DEMO`, `MMQ.DIMS`.
- Produces: no new globals. `window.DSHOW` keeps its existing `{lock, react, reset}` shape — the router depends on it.

- [ ] **Step 1: Write the browser assertion**

```js
() => {
  const M = window.MMQ;
  sessionStorage.removeItem('mm_deck');
  M.setModel('A');
  if (!window.DSHOW) return { ok:false, reason:'DSHOW missing' };
  DSHOW.reset();

  const flag = document.getElementById('turnflag').textContent;
  const stage = document.getElementById('stage');
  const card  = stage.querySelector('.anscard2');
  const opt   = card && card.querySelector('.ac-opt').textContent.trim();
  // turn 1 is yours; the option shown must be a real option of a real question
  const known = M.BANK.some(q => q.opts.some(o2 => o2.t === opt));

  return {
    ok: !!card && known &&
        /\/\s*10\b/.test(flag) &&      // count comes from the set, not "20"
        !/\/\s*20\b/.test(flag) &&
        typeof DSHOW.lock === 'function' &&
        typeof DSHOW.react === 'function' &&
        typeof DSHOW.reset === 'function',
    flag: flag.trim(), opt, known, hasCard: !!card
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `#S-D2`, evaluate.
Expected: `ok:false` — `hasCard: false`, because `S-D2` still renders the old `.ans` div and the flag still says `/ 20`.

- [ ] **Step 3: Replace the turn data**

In the `S-D2` inline script, replace the `const W={...}` and `const TURNS=[...]` declarations (through the closing `];`) with:

```js
          const W = MMQ.DIMS;
          /* Turns are built from the live question set, so the deck and the
             count model (spec 1.2) drive the arena. Even index = you answer,
             odd = David answers and you react. */
          const SET = MMQ.buildSet();
          const TURNS = SET.map((q, n) => ({
            q, mode: n % 2 === 0 ? 'you' : 'them',
            l3: q.depth === 'Deep dive',
            ans: n % 2 === 0
              ? (MMQ.answers().find(a => a.qid === q.id) || { qid:q.id, opt:0, why:'' })
              : Object.assign({ qid:q.id }, MMQ.DEMO.them[q.id] || { opt:0, why:'' }),
            verdict: MMQ.DEMO.verdict[q.id] || 'move'
          }));
```

- [ ] **Step 4: Replace the render body**

Inside `render()`, replace the whole `if(t.mode==='you'){...}else{...}` block with:

```js
            const chip = '<span class="chip '+(t.l3?'l3':'')+' qchip">'
                       + t.q.dim + ' · ' + t.q.depth + '</span>';
            const total = TURNS.length;
            if(t.mode==='you'){
              $('turnflag').innerHTML='Question <b>'+(i+1)+'</b> / '+total
                +' · <b style="color:var(--move)">your turn to answer</b>';
              $('stage').innerHTML = chip + '<div class="qtext">'+t.q.text+'</div>'
                + MMQ.answerCard(t.q, t.ans, 'Your answer');
              $('act').innerHTML='<button class="btn move" onclick="DSHOW.lock()">Lock my answer</button>';
            }else{
              $('turnflag').innerHTML='Question <b>'+(i+1)+'</b> / '+total
                +' · <b style="color:var(--spotlight-soft)">David is answering — you react</b>';
              $('stage').innerHTML = chip + '<div class="qtext">'+t.q.text+'</div>'
                + MMQ.answerCard(t.q, t.ans, "David's answer");
              $('act').innerHTML='<button class="btn ghost" onclick="DSHOW.react(\'stay\')">Stay<span class="sub">not feeling it</span></button><button class="btn move" onclick="DSHOW.react(\'move\')"><svg class="icon"><use href="#i-heart"/></svg> Move<span class="sub">I want you</span></button>';
            }
```

Then, in `render()`, replace the two references to `t.dim` in the escalation line with `t.q.dim`:

```js
            $('esc').classList.toggle('on', !!t.l3 && sd>=1);
```

(unchanged — it reads `t.l3`, which the new shape still provides).

- [ ] **Step 5: Fix the scorers**

Both scorer lines already declare `const t` (and `const v`) on the same line as
the arithmetic, so you must replace the **whole line** — replacing only the
`yA+=…` fragment re-declares `t` and throws
`SyntaxError: Identifier 't' has already been declared`, killing the arena.

In `lock()`, replace this entire line (currently `index.html:1820`):

```js
            const t=TURNS[i]; const v=t.verdict||'move'; yA+=W[t.dim]; if(v==='move') yE+=W[t.dim];
```

with:

```js
            const t=TURNS[i]; const v=t.verdict||'move'; yA+=W[t.q.dim]; if(v==='move') yE+=W[t.q.dim];
```

In `react()`, replace this entire line (currently `index.html:1834`):

```js
            const t=TURNS[i]; tA+=W[t.dim]; if(kind==='move') tE+=W[t.dim]; else sd++;
```

with:

```js
            const t=TURNS[i]; tA+=W[t.q.dim]; if(kind==='move') tE+=W[t.q.dim]; else sd++;
```

- [ ] **Step 6: Run the assertion to verify it passes**

Re-evaluate Step 1. Expected: `ok: true`, `flag` matching `Question 1 / 10 · your turn to answer`, `known: true`.

- [ ] **Step 7: Play the board through to check the outcome routing still works**

```js
() => {
  DSHOW.reset();
  let guard = 0;
  while (location.hash.replace('#','') === 'S-D2' && guard++ < 40) {
    const act = document.getElementById('act');
    const lock = act.querySelector('button[onclick*="lock"]');
    if (lock) { DSHOW.lock(); document.getElementById('rnext').click(); }
    else { DSHOW.react('move'); }
  }
  return { py: sessionStorage.getItem('mm_py'),
           pt: sessionStorage.getItem('mm_pt'),
           landed: location.hash };
}
```

Expected: `py` and `pt` are numeric strings and `landed` is `#S-D5` or `#S-D6`. If the loop hits the guard, the turn advance is broken — check that `rnext.onclick` still increments `i`.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "feat(arena): drive S-D2 turns from MMQ with option-based answers"
```

---

### Task 4: The async board (S-F3) reads from MMQ with a variable slot count

**Files:**
- Modify: `index.html` — the `.slots` markup in `S-F3` (~lines 2131–2139) and its inline `<script>` (~lines 2142–2198).

**Interfaces:**
- Consumes: `MMQ.buildSet`, `MMQ.answerCard`, `MMQ.DEMO`, `MMQ.DIMS`.
- Produces: no new globals. `window.FSHOW` keeps `{react, reset}`.

- [ ] **Step 1: Write the browser assertion**

```js
() => {
  const M = window.MMQ;
  sessionStorage.removeItem('mm_deck');
  M.setModel('A');
  if (!window.FSHOW) return { ok:false, reason:'FSHOW missing' };
  FSHOW.reset();
  const slotsA = document.querySelectorAll('#S-F3 .slot').length;

  // Model B with a 5-question deck must produce 15 slots, not a hardcoded 5
  M.setModel('B');
  M.saveDeck(M.BANK.slice(0,5).map((q,n) => ({ ...q, id:'c'+n, src:'custom', author:'You' })));
  FSHOW.reset();
  const slotsB = document.querySelectorAll('#S-F3 .slot').length;
  const card = document.querySelector('#S-F3 .anscard2');

  M.setModel('A'); sessionStorage.removeItem('mm_deck'); FSHOW.reset();

  return { ok: slotsA === 10 && slotsB === 15 && !!card, slotsA, slotsB, hasCard: !!card };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `#S-F3`, evaluate.
Expected: `slotsA: 5, slotsB: 5` — the board is hardcoded to five `<div class="slot">` elements.

- [ ] **Step 3: Empty the hardcoded board**

Replace the whole `<div class="board" id="fboard">…</div>` block with:

```html
        <div class="board" id="fboard"><div class="slots" id="fslots"></div></div>
```

- [ ] **Step 4: Replace the question data and add slot building**

In the `S-F3` inline script, replace `const QF=[…];` and `const TOTW=…` with:

```js
          /* The board is built from the live set, so its length follows the
             count model (spec 1.2) rather than a hardcoded five. */
          const QF = MMQ.buildSet().map(q => ({
            q, w: MMQ.DIMS[q.dim],
            ans: Object.assign({ qid:q.id }, MMQ.DEMO.them[q.id] || { opt:0, why:'' })
          }));
          const TOTW = QF.reduce((s,t) => s + t.w, 0);
          function fslots(){
            document.getElementById('fslots').innerHTML = QF.map((t,n) =>
              '<div class="slot"><div class="in">'
              + '<div class="face front"><b>'+(n+1)+'</b></div>'
              + '<div class="face back"></div></div></div>').join('');
          }
```

- [ ] **Step 5: Rewrite the stage renderer**

Replace the body of `frender()` with:

```js
          function frender(){
            if(fi>=QF.length) return ffinish();
            const t=QF[fi];
            $('fstage').innerHTML='<span class="chip" style="animation:rise .4s var(--ease-show) both">'
              + t.q.dim + ' · slot ' + (fi+1) + ' of ' + QF.length + '</span>'
              + '<div class="qbar">' + t.q.text + '</div>'
              + MMQ.answerCard(t.q, t.ans, 'Samuel said (recorded)')
              + '<div class="wnote">Weight ' + t.w.toFixed(1) + " on Samuel's board.</div>";
            $('fact').innerHTML='<button class="btn ghost" onclick="FSHOW.react(\'stay\')">Stay<span class="sub">not feeling it</span></button>'
              +'<button class="btn move" onclick="FSHOW.react(\'move\')"><svg class="icon"><use href="#i-heart"/></svg> Move<span class="sub">I want this</span></button>';
          }
```

- [ ] **Step 6: Point the flip logic at the rebuilt slots**

In `freact()`, replace `back.innerHTML = kind==='move' ? '…'+t.dim+'…'` so it reads `t.q.dim`:

```js
            back.innerHTML = kind==='move'
              ? '<span><span class="hh">♥</span>'+t.q.dim+'</span><span class="pts">'+Math.round(t.w*10)+'</span>'
              : '<span style="letter-spacing:.12em">'+t.q.dim+' · stay</span><span class="pts">—</span>';
```

- [ ] **Step 7: Rebuild slots on reset**

Replace the `reset()` in the `window.FSHOW` assignment with:

```js
          window.FSHOW={react:freact, reset(){ fi=0; fgot=0; $('fpct')._v=null; $('fpct').textContent='0%';
            $('fboard').classList.remove('done'); fslots(); frender(); }};
          fslots();
          frender();
```

There is exactly **one** trailing bare `frender();` line after the old
assignment (currently `index.html:2196`) — delete that one. Do **not** touch the
`frender()` call inside the `setTimeout` in `freact()` (currently
`index.html:2182`); that one drives turn advance and must stay.

- [ ] **Step 8: Run the assertion to verify it passes**

Re-evaluate Step 1. Expected: `ok: true`, `slotsA: 10`, `slotsB: 15`, `hasCard: true`.

- [ ] **Step 9: Commit**

```bash
git add index.html
git commit -m "feat(async): drive S-F3 board from MMQ with a variable slot count"
```

---

### Task 5: The dev-settings panel

**Files:**
- Modify: `index.html` — append CSS to the main `<style>`; add a `⚙` button to `.toolbar` (line ~528, after the `⟲ Restart` button); add the panel markup before the closing `</body>`; add its script to the main router `<script>`.

**Interfaces:**
- Consumes: `MMQ.model`, `MMQ.setModel`.
- Produces on `window`: `MMDEV.open()`, `MMDEV.close()`, `MMDEV.setModel(m)`. Phase 6 adds the karma display A/B toggle to this same panel (spec 6.4) — it must be built to hold more than one row.

- [ ] **Step 1: Write the browser assertion**

```js
() => {
  if (!window.MMDEV) return { ok:false, reason:'MMDEV not defined' };
  MMDEV.setModel('B');
  const bAfter = MMQ.model();
  const bLen = MMQ.buildSet().length;
  MMDEV.setModel('A');
  const aAfter = MMQ.model();
  MMDEV.open();
  const visible = document.getElementById('devpanel').classList.contains('on');
  const rows = document.querySelectorAll('#devpanel .devrow').length;
  MMDEV.close();
  const hidden = !document.getElementById('devpanel').classList.contains('on');
  return { ok: bAfter==='B' && aAfter==='A' && bLen===10 && visible && hidden && rows >= 1,
           bAfter, aAfter, bLen, visible, hidden, rows };
}
```

(`bLen` is 10 because the deck is empty — Model B is 10 bank + 0 deck.)

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `#S-D2`, evaluate. Expected: `{ ok:false, reason:'MMDEV not defined' }`.

- [ ] **Step 3: Add the CSS**

Append to the main `<style>` block:

```css
/* ===== dev settings · A/B toggles for testable features =====
   Holds the question-count model (spec 1.2) and, from Phase 6, the karma
   display direction (spec 6.4). Built as rows so more can be added. */
.devpanel{position:fixed; right:14px; bottom:14px; z-index:120; width:260px;
  background:var(--paper); color:var(--t); border:1px solid var(--line);
  border-radius:14px; box-shadow:0 18px 50px rgba(0,0,0,.35); padding:14px;
  display:none}
.devpanel.on{display:block}
.devpanel h3{font-family:var(--display); font-size:13px; margin:0 0 4px;
  letter-spacing:.06em; text-transform:uppercase}
.devpanel .hint2{font-size:11px; color:var(--t-faint); margin-bottom:10px}
.devrow{border-top:1px solid var(--line); padding-top:10px; margin-top:10px}
.devrow:first-of-type{border-top:none; padding-top:0; margin-top:0}
.devrow .lbl2{font-size:12px; font-weight:700; margin-bottom:6px}
.devrow .seg{display:flex; gap:6px}
.devrow .seg button{flex:1; font-size:11px; padding:7px 4px; border-radius:8px;
  border:1px solid var(--line); background:var(--paper-2); color:var(--t-dim);
  cursor:pointer; font-family:var(--body)}
.devrow .seg button.on{background:var(--move); border-color:var(--move); color:#fff; font-weight:700}
.devrow .note2{font-size:10.5px; color:var(--t-faint); margin-top:6px; line-height:1.4}
/* .tbtn is styled for the dark toolbar; re-theme it for this paper panel or
   the Close button renders near-invisible. */
.devpanel .tbtn{color:var(--t-dim); border-color:var(--line)}
.devpanel .tbtn:hover{color:var(--t); border-color:var(--move)}
```

- [ ] **Step 4: Add the trigger and the panel markup**

In `.toolbar`, immediately after the `⟲ Restart` button (line ~528), add:

```html
    <button class="tbtn" onclick="MMDEV.open()" title="Dev settings"><svg class="icon"><use href="#i-gear"/></svg></button>
```

The `#i-gear` symbol already exists in the sprite at the top of `<body>` — do
not add a new one.

Immediately before the closing `</body>`, add:

```html
<div class="devpanel" id="devpanel">
  <h3>Dev settings</h3>
  <div class="hint2">A/B switches for features still under test. Prototype only.</div>
  <div class="devrow">
    <div class="lbl2">Question count model</div>
    <div class="seg">
      <button id="devqa" onclick="MMDEV.setModel('A')">A · 10 total</button>
      <button id="devqb" onclick="MMDEV.setModel('B')">B · 10 + 5</button>
    </div>
    <div class="note2">A replaces up to 5 bank questions with yours. B adds yours on top, max 15.</div>
  </div>
  <div style="text-align:right;margin-top:12px">
    <button class="tbtn" onclick="MMDEV.close()">Close</button>
  </div>
</div>
```

- [ ] **Step 5: Add the script**

Append inside the main router `<script>` at the bottom of the file — after its
final `})();` and before `</script>`. That script mixes top-level code with
IIFEs; this goes at top level, at the very end:

```js
  /* Dev settings — spec 1.2 and 6.4 both call for an A/B switch, and they
     share one surface rather than each inventing their own. */
  window.MMDEV = {
    open(){ document.getElementById('devpanel').classList.add('on'); MMDEV.paint(); },
    close(){ document.getElementById('devpanel').classList.remove('on'); },
    paint(){
      /* The panel markup sits after this script in document order, so at
         parse time these are null. Guard, or the TypeError aborts the rest
         of the router script and takes MMATTACH down with it. */
      const a = document.getElementById('devqa'), b = document.getElementById('devqb');
      if(!a || !b) return;
      const m = MMQ.model();
      a.classList.toggle('on', m === 'A');
      b.classList.toggle('on', m === 'B');
    },
    setModel(m){
      MMQ.setModel(m); MMDEV.paint();
      /* rebuild any board that is already on screen so the change is visible */
      if(window.DSHOW) DSHOW.reset();
      if(window.FSHOW) FSHOW.reset();
    }
  };
  MMDEV.paint();
```

- [ ] **Step 6: Make the arena's question set rebuildable**

`MMDEV.setModel()` calls `DSHOW.reset()` expecting the board to pick up the new
model — but `S-D2` currently computes its set **once at parse time**:

```js
          const SET = MMQ.buildSet();
          const TURNS = SET.map((q, n) => ({ …
```

`reset()` re-renders from that frozen array, so switching the model changes
nothing visible in the arena. (Task 4 already solved the identical problem in
`S-F3` with a rebuildable `fbuild()`; this mirrors it.)

Change the two `const` declarations to a rebuild function. Replace:

```js
          const SET = MMQ.buildSet();
          const TURNS = SET.map((q, n) => ({
```

with:

```js
          let SET, TURNS;
          function dbuild(){
            SET = MMQ.buildSet();
            TURNS = SET.map((q, n) => ({
```

Close the new function immediately after the existing `}));` that ends the
`.map(...)` call, so it becomes `})); }` — then call `dbuild();` once on the
line after it, preserving parse-time behaviour.

Then make `reset()` rebuild first:

```js
          window.DSHOW={lock,react,reset(){dbuild();i=0;yA=yE=tA=tE=sd=0;$('tokens').innerHTML='';$('reveal').className='reveal';render();}};
```

- [ ] **Step 7: Extend the assertion to prove the arena actually rebuilds**

The Step 1 assertion only proves `MMQ.model()` changed. Add this second
assertion, which fails if `TURNS` is still frozen:

```js
() => {
  sessionStorage.removeItem('mm_deck');
  location.hash = 'S-D2';
  MMDEV.setModel('A');
  const a = document.getElementById('turnflag').textContent;
  MMQ.saveDeck(MMQ.BANK.slice(0,3).map((q,n)=>({...q,id:'c'+n,src:'custom',author:'You'})));
  MMDEV.setModel('B');
  const b = document.getElementById('turnflag').textContent;
  MMDEV.setModel('A'); sessionStorage.removeItem('mm_deck'); DSHOW.reset();
  return { ok: /\/\s*10\b/.test(a) && /\/\s*13\b/.test(b),
           modelA: a.replace(/\s+/g,' ').trim(), modelB: b.replace(/\s+/g,' ').trim() };
}
```

Expected: `ok: true`, `modelA` showing `/ 10`, `modelB` showing `/ 13`
(10 bank + a 3-question deck).

- [ ] **Step 8: Run both assertions to verify they pass**

Re-evaluate Step 1 (`ok: true`, `bAfter:'B'`, `aAfter:'A'`, `bLen:10`, `rows: 1`)
and then Step 7's.

- [ ] **Step 9: Commit**

```bash
git add index.html
git commit -m "feat(dev): add shared dev-settings panel with question model A/B"
```

---

### Task 6: Deck authoring screens (S-QD1–S-QD3)

**Files:**
- Modify: `index.html` — insert three new `<section class="screen paper">` blocks after `S-P5` (~line 981); add `'S-QD1'` to the `FEED` set in `classify()` (~line 2419).

**Interfaces:**
- Consumes: `MMQ.deck`, `MMQ.saveDeck`, `MMQ.reviewQuestion`, `MMQ.genOptions`, `MMQ.MAX_DECK`, `MMQ.ATTEMPTS`, `MMQ.BANK`.
- Produces on `window`: `MMDECK.render()`, `MMDECK.add(text)`, `MMDECK.remove(i)`, `MMDECK.submit()`, `MMDECK.replace(i, text)`, `MMDECK.attemptsLeft()`.

- [ ] **Step 1: Write the browser assertion**

```js
() => {
  if (!window.MMDECK) return { ok:false, reason:'MMDECK not defined' };
  sessionStorage.removeItem('mm_deck');
  sessionStorage.removeItem('mm_deck_used');

  // cap at 5
  for (let n=0;n<7;n++) MMDECK.add('Would you move abroad for a partner? #'+n);
  const capped = MMQ.deck().length;
  const gen = MMQ.deck()[0].opts.length;

  // a blocked question is rejected, and rejection burns no attempt by itself
  const bad = MMQ.reviewQuestion('What is your salary?');
  const vague = MMQ.reviewQuestion('why?');
  const good = MMQ.reviewQuestion('Would you move abroad for a partner?');

  // three replacements, then backfill from the bank
  sessionStorage.removeItem('mm_deck_used');
  const a0 = MMDECK.attemptsLeft();
  MMDECK.replace(0,'What is your salary?');          // fails review, burns one
  MMDECK.replace(0,'What is your salary?');
  MMDECK.replace(0,'What is your salary?');
  const a3 = MMDECK.attemptsLeft();
  MMDECK.replace(0,'What is your salary?');          // exhausted -> backfill
  const filled = MMQ.deck()[0];

  sessionStorage.removeItem('mm_deck');
  sessionStorage.removeItem('mm_deck_used');

  return {
    ok: capped === 5 && gen === 4 &&
        bad.ok === false && vague.ok === false && good.ok === true &&
        a0 === 3 && a3 === 0 &&
        filled.src === 'bank',
    capped, gen, a0, a3, backfilledFrom: filled.src,
    reasons: { bad: bad.reason, vague: vague.reason }
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `#S-P5`, evaluate. Expected: `{ ok:false, reason:'MMDECK not defined' }`.

- [ ] **Step 3: Insert the three screens**

Insert after the closing `</section>` of `S-P5`:

```html
      <section class="screen paper" id="S-QD1" data-group="A · Onboarding" data-title="Question deck · your 5 questions">
        <div class="pbar"><a class="back" href="#S-P5">‹</a><h1>Your questions</h1></div>
        <div class="body">
          <div class="note gold">Write up to <b>5 questions</b> of your own. We turn each into a set of options so nobody can dodge with a vague answer — and we check them for you before anyone sees them.</div>
          <div id="deckList"></div>
          <div class="field"><label>Add a question</label>
            <input class="input" id="deckNew" placeholder="e.g. Would you move abroad for a partner?">
          </div>
          <button class="btn ghost block" onclick="MMDECK.addFromInput()">Add to deck</button>
          <div class="note info">Your deck is attached automatically when you invite someone or start an async game. You can swap questions per match.</div>
        </div>
        <div class="foot"><a class="btn move block" href="#S-QD3" onclick="MMDECK.submit()">Submit for review</a></div>
      </section>

      <section class="screen paper" id="S-QD2" data-group="A · Onboarding" data-title="Question deck · generated options">
        <div class="pbar"><a class="back" href="#S-QD1">‹</a><h1>Your options</h1></div>
        <div class="body">
          <div class="note">These are the options we generated. Everyone answering picks exactly one — that is what gets scored. They may add a line of their own beneath it, but it is optional.</div>
          <div id="deckOpts"></div>
        </div>
        <div class="foot"><a class="btn move block" href="#S-QD1">Back to deck</a></div>
      </section>

      <section class="screen paper" id="S-QD3" data-group="A · Onboarding" data-title="Question deck · review results">
        <div class="pbar"><a class="back" href="#S-QD1">‹</a><h1>Review results</h1></div>
        <div class="body">
          <div id="deckReview"></div>
        </div>
        <div class="foot"><a class="btn move block" href="#S-B2">Done</a></div>
      </section>
```

- [ ] **Step 4: Add the deck script**

Insert a `<script>` immediately before the closing `</section>` of `S-QD3`:

```html
        <script>
        /* ===== MMDECK · custom question authoring (spec 1.3, 1.6) =====
           3 replacement attempts TOTAL per submission, then the rejected slot
           is backfilled from the bank so a game always has a full set. */
        (function(){
          const used = () => +(sessionStorage.getItem('mm_deck_used') || 0);
          const burn = () => sessionStorage.setItem('mm_deck_used', used() + 1);
          const attemptsLeft = () => Math.max(0, MMQ.ATTEMPTS - used());

          function mk(text){
            return { id:'c-' + Date.now() + '-' + Math.random().toString(36).slice(2,7),
                     dim:'Values', depth:'Screening', probe:'custom',
                     text:String(text).trim(), opts:MMQ.genOptions(text),
                     src:'custom', author:'You', mirror:'custom',
                     review:MMQ.reviewQuestion(text) };
          }
          function add(text){
            const d = MMQ.deck();
            if(d.length >= MMQ.MAX_DECK) return false;
            d.push(mk(text)); MMQ.saveDeck(d); render(); return true;
          }
          function addFromInput(){
            const el = document.getElementById('deckNew');
            if(el.value.trim()){ add(el.value); el.value = ''; }
          }
          function remove(i){ const d = MMQ.deck(); d.splice(i,1); MMQ.saveDeck(d); render(); }

          /* A rejected question can be rewritten 3 times. On the 4th, we stop
             asking and drop a bank question into the slot instead. */
          function replace(i, text){
            const d = MMQ.deck(); if(!d[i]) return;
            if(attemptsLeft() === 0){
              const taken = d.map(q => q.id);
              const fill = MMQ.BANK.find(q => taken.indexOf(q.id) === -1);
              d[i] = Object.assign({}, fill);
              MMQ.saveDeck(d); render(); return;
            }
            const next = mk(text);
            if(!next.review.ok) burn();
            d[i] = next; MMQ.saveDeck(d); render();
          }
          function submit(){ render(); }

          function render(){
            const d = MMQ.deck();
            const list = document.getElementById('deckList');
            if(list){
              list.innerHTML = d.length ? d.map((q,i) =>
                '<div class="card" style="margin-bottom:8px">'
                + '<div style="display:flex;justify-content:space-between;gap:8px">'
                + '<div style="font-weight:700;font-size:13.5px">'+q.text+'</div>'
                + '<span class="tag '+(q.src==='bank'?'':(q.review && q.review.ok?'pass':'danger'))+'">'
                + (q.src==='bank' ? 'From bank' : (q.review && q.review.ok ? 'Cleared' : 'Flagged'))
                + '</span></div>'
                + (q.review && !q.review.ok ? '<div class="muted" style="font-size:11.5px;margin-top:6px">'+q.review.reason+'</div>' : '')
                + '<div class="btnrow" style="margin-top:10px">'
                + '<button class="btn ghost sm" onclick="MMDECK.remove('+i+')">Remove</button>'
                + '<a class="btn ghost sm" href="#S-QD2" onclick="MMDECK.showOpts('+i+')">See options</a>'
                + '</div></div>').join('')
                : '<div class="note">No questions yet. Add up to '+MMQ.MAX_DECK+'.</div>';
            }
            const rev = document.getElementById('deckReview');
            if(rev){
              const flagged = d.filter(q => q.src !== 'bank' && q.review && !q.review.ok).length;
              rev.innerHTML =
                '<div class="note '+(flagged?'':'pass')+'">'
                + (flagged ? flagged+' of your questions need a rewrite.' : 'All your questions cleared review.')
                + '</div>'
                + '<div class="row"><div class="grow"><div class="t1">Replacement attempts left</div>'
                + '<div class="t2">3 per submission, then we fill the slot from the bank</div></div>'
                + '<span class="tag '+(attemptsLeft()?'':'danger')+'">'+attemptsLeft()+'</span></div>'
                + d.map((q,i) => '<div class="row"><div class="grow"><div class="t1">'+q.text+'</div>'
                    + '<div class="t2">'+(q.src==='bank' ? 'Filled from the question bank'
                        : (q.review.ok ? 'Cleared' : q.review.reason))+'</div></div>'
                    + (q.src!=='bank' && !q.review.ok
                        ? '<button class="btn ghost sm" onclick="MMDECK.replace('+i+',prompt(\'Rewrite your question\',\''+q.text.replace(/'/g,"")+'\')||\'\')">Rewrite</button>'
                        : '')
                    + '</div>').join('');
            }
          }
          function showOpts(i){
            const q = MMQ.deck()[i]; if(!q) return;
            document.getElementById('deckOpts').innerHTML =
              '<div class="card"><div class="eyebrow">'+q.text+'</div>'
              + '<div style="display:flex;flex-direction:column;gap:8px;margin-top:10px">'
              + q.opts.map(op => '<div class="opt"><span class="dot"></span>'+op.t+'</div>').join('')
              + '</div></div>';
          }
          window.MMDECK = { add, addFromInput, remove, replace, submit, render, showOpts, attemptsLeft };
          render();
        })();
        </script>
```

- [ ] **Step 5: Register S-QD1 as a feed screen**

In `classify()`, change the `FEED` set to include `'S-QD1'`:

```js
    const FEED = new Set(['S-B2','S-E3','S-E4','S-A8','S-H1','S-G3','S-H5','S-QD1']);
```

- [ ] **Step 6: Run the assertion to verify it passes**

Re-evaluate Step 1. Expected: `ok: true`, `capped: 5`, `gen: 4`, `a0: 3`, `a3: 0`, `backfilledFrom: 'bank'`.

- [ ] **Step 7: Check the screens render**

Navigate to `#S-QD1`, screenshot. The deck list, the add field and the submit button must all be visible, and the screen must reflow as a grid at 1280×800 (it is in `FEED`). Then navigate to `#S-QD3` and screenshot — the attempts-left row must show a number.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "feat(deck): add question deck authoring with LLM review and backfill"
```

---

### Task 7: Attach the deck at invite and pursue

**Files:**
- Modify: `index.html` — `S-C1` (~lines 1536–1554) and `S-F2` (~lines 2054–2066).

**Interfaces:**
- Consumes: `MMQ.deck`, `MMQ.MAX_DECK`, `MMDECK.render`.
- Produces on `window`: `MMATTACH.paint()`.

- [ ] **Step 1: Write the browser assertion**

```js
() => {
  if (!window.MMATTACH) return { ok:false, reason:'MMATTACH not defined' };
  sessionStorage.removeItem('mm_deck');
  MMDECK.add('Would you move abroad for a partner?');
  MMDECK.add('Would you live with a parent long term?');
  MMATTACH.paint();
  const c1 = document.getElementById('attachC1').textContent;
  const f2 = document.getElementById('attachF2').textContent;
  sessionStorage.removeItem('mm_deck'); MMATTACH.paint();
  const c1empty = document.getElementById('attachC1').textContent;
  return {
    ok: /2 of 5/.test(c1) && /2 of 5/.test(f2) &&
        /only the initiator/i.test(f2) && /0 of 5/.test(c1empty),
    c1: c1.replace(/\s+/g,' ').trim(), f2: f2.replace(/\s+/g,' ').trim()
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `#S-C1`, evaluate. Expected: `{ ok:false, reason:'MMATTACH not defined' }`.

- [ ] **Step 3: Add the attach card to S-C1**

In `S-C1`, insert immediately after the `<div class="note gold">…</div>` line:

```html
          <div class="card" id="attachC1"></div>
```

- [ ] **Step 4: Add the attach card to S-F2**

In `S-F2`, insert immediately after the `<div class="note info">…</div>` line:

```html
          <div class="card" id="attachF2"></div>
```

- [ ] **Step 5: Add the script**

Append inside the main router `<script>` at the bottom of the file, after the `MMDEV` block:

```js
  /* Deck attach (spec 1.6). Sync: both participants may attach questions.
     Async: only the initiator may, so S-F2 says so explicitly. */
  window.MMATTACH = {
    paint(){
      const d = MMQ.deck();
      const lines = d.length
        ? d.map(q => '<div class="t2">· ' + q.text + '</div>').join('')
        : '<div class="t2">No custom questions yet.</div>';
      const head = n => '<div style="display:flex;justify-content:space-between;align-items:center">'
        + '<div style="font-weight:700;font-size:13px">Your questions</div>'
        + '<span class="tag">' + n + ' of ' + MMQ.MAX_DECK + '</span></div>';
      const edit = '<div class="btnrow" style="margin-top:10px">'
        + '<a class="btn ghost sm" href="#S-QD1">Edit deck</a></div>';

      const c1 = document.getElementById('attachC1');
      if(c1) c1.innerHTML = head(d.length)
        + '<div style="margin-top:8px">' + lines + '</div>'
        + '<div class="muted" style="font-size:11px;margin-top:8px">Both of you can attach questions to a live show.</div>'
        + edit;

      const f2 = document.getElementById('attachF2');
      if(f2) f2.innerHTML = head(d.length)
        + '<div style="margin-top:8px">' + lines + '</div>'
        + '<div class="muted" style="font-size:11px;margin-top:8px">In an async game only the initiator sets custom questions — that is you.</div>'
        + edit;
    }
  };
  MMATTACH.paint();
  window.addEventListener('hashchange', () => MMATTACH.paint());
```

- [ ] **Step 6: Run the assertion to verify it passes**

Re-evaluate Step 1. Expected: `ok: true`, both strings containing `2 of 5`, and `f2` containing the initiator-only sentence.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat(deck): attach the question deck at S-C1 and S-F2"
```

---

### Task 8: Confirmed-about-you block on S-E6

**Files:**
- Modify: `index.html` — `S-E6` (~lines 2006–2024); append CSS to the main `<style>`.

**Interfaces:**
- Consumes: `MMQ.confidence`, `MMQ.confLabel`, `MMQ.CONF_GATE`, `MMQ.probes`, `MMQ.answers`, `MMQ.qById`, `MMQ.BANK`.
- Produces on `window`: `MMCONF.paint()`, `MMCONF.reject(probe)`.

- [ ] **Step 1: Write the browser assertion**

```js
() => {
  if (!window.MMCONF) return { ok:false, reason:'MMCONF not defined' };
  sessionStorage.removeItem('mm_conf_rejected');
  sessionStorage.removeItem('mm_deck');   // a leftover deck adds a 'custom' probe row
  MMCONF.paint();
  const host = document.getElementById('confBlock');
  const rows = [...host.querySelectorAll('.confrow')];
  const byProbe = Object.fromEntries(rows.map(r => [
    r.dataset.probe,
    { pct: +r.dataset.pct, label: r.querySelector('.cf-label').textContent.trim(),
      cites: r.querySelectorAll('.cf-cite').length }
  ]));
  const kids = byProbe['wants-children'];

  // rejecting a confirmed truth must remove it from the confirmed list
  MMCONF.reject('wants-children');
  const after = document.querySelector('.confrow[data-probe="wants-children"]');
  sessionStorage.removeItem('mm_conf_rejected'); MMCONF.paint();

  return {
    ok: !!kids && kids.pct === 100 && kids.label === 'Confirmed' && kids.cites === 3 &&
        byProbe['core-value'].pct === 67 &&
        byProbe['core-value'].label === 'Still forming' &&
        !byProbe['closeness'] &&                       // unanswered probes are omitted
        after && after.dataset.rejected === 'true',
    byProbe, rowCount: rows.length
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `#S-E6`, evaluate. Expected: `{ ok:false, reason:'MMCONF not defined' }`.

- [ ] **Step 3: Add the CSS**

Append to the main `<style>` block:

```css
/* ===== confirmed-about-you (spec 1.4) =====
   A truth is written to the profile once corroborating variations of the same
   probe agree. The bar shows that confidence, not a preference weight. */
.confrow{border-top:1px solid rgba(255,255,255,.08); padding:11px 0}
.paper .confrow{border-top-color:var(--line)}
.confrow:first-child{border-top:none}
.confrow[data-rejected="true"]{opacity:.45}
.confrow .cf-head{display:flex; justify-content:space-between; align-items:baseline; gap:10px}
.confrow .cf-name{font-weight:700; font-size:13.5px}
.confrow .cf-label{font-size:10px; letter-spacing:.12em; text-transform:uppercase;
  font-family:var(--display); font-weight:700; color:var(--ink-faint)}
.paper .confrow .cf-label{color:var(--t-faint)}
.confrow[data-confirmed="true"] .cf-label{color:var(--pass)}
.confrow .cf-val{font-size:12.5px; margin:3px 0 7px; color:var(--ink-dim)}
.paper .confrow .cf-val{color:var(--t-dim)}
.confrow .cf-cites{margin-top:8px; display:flex; flex-direction:column; gap:3px}
.confrow .cf-cite{font-size:11px; color:var(--ink-faint); line-height:1.4}
.paper .confrow .cf-cite{color:var(--t-faint)}
.confrow .cf-wrong{margin-top:8px; font-size:11.5px; background:none; border:none;
  padding:0; cursor:pointer; color:var(--move); font-family:var(--body); text-decoration:underline}
```

- [ ] **Step 4: Add the block to S-E6**

In `S-E6`, insert immediately after the `<div class="note gold">…</div>` line:

```html
          <div class="card">
            <div class="eyebrow" style="color:var(--bulb)">Confirmed about you</div>
            <div class="muted" style="font-size:11.5px;margin-top:4px">We ask the same thing several ways. When your answers agree, we write it down.</div>
            <div id="confBlock" style="margin-top:10px"></div>
          </div>
```

- [ ] **Step 5: Add the script**

Insert a `<script>` immediately before the closing `</section>` of `S-E6`:

```html
        <script>
        /* ===== MMCONF · confirmed attributes (spec 1.4 / item 8) =====
           Confidence is derived from MMQ, never hardcoded — change a demo
           answer and these meters move. */
        (function(){
          const NAMES = {
            'wants-children':'Wanting children', 'children-timing':'Timing for a first child',
            'blending':'Blending two families',  'step-resilience':'Facing a stepchild’s rejection',
            'core-value':'What you value most',  'non-negotiable':'Your non-negotiable',
            'weekend':'How you spend a weekend', 'money-model':'How you handle money',
            'five-year':'Where you are heading', 'closeness':'What closeness means'
          };
          const rejected = () => { try { return JSON.parse(sessionStorage.getItem('mm_conf_rejected')) || []; }
                                   catch(e){ return []; } };

          function paint(){
            const host = document.getElementById('confBlock'); if(!host) return;
            const ans = MMQ.answers(), out = rejected();
            const rows = MMQ.probes().map(p => ({ p, pct: MMQ.confidence(p, ans) }))
                                     .filter(r => r.pct !== null)
                                     .sort((a,b) => b.pct - a.pct);
            host.innerHTML = rows.map(r => {
              const hits = ans.filter(a => { const q = MMQ.qById(a.qid); return q && q.probe === r.p; });
              const val = MMQ.qById(hits[0].qid).opts[hits[0].opt].t;
              const conf = r.pct >= MMQ.CONF_GATE;
              const isOut = out.indexOf(r.p) !== -1;
              return '<div class="confrow" data-probe="'+r.p+'" data-pct="'+r.pct+'"'
                + ' data-confirmed="'+conf+'" data-rejected="'+isOut+'">'
                + '<div class="cf-head"><span class="cf-name">'+(NAMES[r.p]||r.p)+'</span>'
                + '<span class="cf-label">'+MMQ.confLabel(r.pct)+'</span></div>'
                + '<div class="cf-val">'+val+'</div>'
                + '<div class="prog"><i style="width:'+r.pct+'%;background:'
                + (conf?'var(--pass)':'var(--bulb)')+'"></i></div>'
                + '<div class="cf-cites">' + hits.map(a =>
                    '<div class="cf-cite">· '+MMQ.qById(a.qid).text+'</div>').join('') + '</div>'
                + (isOut ? '<div class="cf-cite" style="color:var(--move)">You told us this is wrong — we have stopped using it.</div>'
                         : '<button class="cf-wrong" onclick="MMCONF.reject(\''+r.p+'\')">That’s wrong about me</button>')
                + '</div>';
            }).join('');
          }
          function reject(p){
            const out = rejected();
            if(out.indexOf(p) === -1) out.push(p);
            sessionStorage.setItem('mm_conf_rejected', JSON.stringify(out));
            paint();
          }
          window.MMCONF = { paint, reject };
          paint();
        })();
        </script>
```

- [ ] **Step 6: Run the assertion to verify it passes**

Re-evaluate Step 1. Expected: `ok: true`, with `wants-children` at `pct: 100`, label `Confirmed`, `cites: 3`, and `core-value` at `pct: 67`, label `Still forming`.

- [ ] **Step 7: Screenshot the screen**

Navigate to `#S-E6` at 1280×800 and screenshot. The confirmed block must sit above the existing "What you weigh most" card, the bars must be visibly different lengths, and the pass-green must only appear on rows at or above 70.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "feat(profile): add confirmed-about-you block driven by answer confidence"
```

---

### Task 9: Full-flow regression

**Files:**
- None — verification only.

- [ ] **Step 1: Walk the whole prototype**

With a clean session, navigate through every screen the index lists and confirm nothing throws. Run:

```js
() => {
  const ids = [...document.querySelectorAll('.screen')].map(s => s.id);
  const errs = [];
  window.onerror = (m) => errs.push(m);
  ids.forEach(id => { location.hash = id; });
  return { screens: ids.length, hasQD: ids.includes('S-QD1'), errs };
}
```

Expected: `screens` is the old count plus 3, `hasQD: true`, `errs: []`.

- [ ] **Step 2: Check the console is clean**

Run `browser_console_messages`. Expected: no `error`-level entries. Unsplash 404s from the gallery are pre-existing and acceptable; anything referencing `MMQ`, `MMDECK`, `MMCONF`, `MMATTACH` or `MMDEV` is not.

- [ ] **Step 3: Confirm both count models survive a full arena run**

**This loop must be `async`.** `lock()` reveals David's verdict behind a 600ms
`setTimeout` and `react()` advances the turn behind a 750ms one, so a synchronous
`while` spins without ever letting a turn advance and always hits the guard. Wait
between actions:

```js
async () => {
  const sleep = ms => new Promise(r => setTimeout(r, ms));
  const out = {};
  for (const m of ['A','B']) {
    sessionStorage.removeItem('mm_deck');
    MMQ.setModel(m);
    if (m === 'B') MMQ.saveDeck(MMQ.BANK.slice(0,3).map((q,n)=>({...q,id:'c'+n,src:'custom',author:'You'})));
    location.hash = 'S-D2'; DSHOW.reset();
    let guard = 0;
    while (location.hash.replace('#','') === 'S-D2' && guard++ < 40) {
      const lock = document.getElementById('act').querySelector('button[onclick*="lock"]');
      if (lock) {
        DSHOW.lock();
        await sleep(700);                       // verdict reveal
        document.getElementById('rnext').click();
      } else {
        DSHOW.react('move');
        await sleep(850);                       // turn advance
      }
      await sleep(30);
    }
    out[m] = { landed: location.hash, py: sessionStorage.getItem('mm_py'), guard };
  }
  MMQ.setModel('A'); sessionStorage.removeItem('mm_deck');
  return out;
}
```

Expected: both `A` and `B` land on `#S-D5` (all-Move runs pass the gate) with a numeric `py`, and neither hits the guard of 40.

- [ ] **Step 4: Exercise the paper-surface answer card**

No Phase 1 screen renders an answer card on a `paper` surface — `S-D2` and
`S-F3` are both `dark` — so the `.paper .anscard2` override ships unexercised.
Confirm it resolves rather than leaving it to Phase 5:

```js
() => {
  const q = MMQ.qById('q-fam-01');
  const host = document.createElement('div');
  host.className = 'paper';
  host.innerHTML = MMQ.answerCard(q, { qid:'q-fam-01', opt:0, why:'A noisier house.' }, 'Test');
  document.body.appendChild(host);
  const cs = getComputedStyle(host.querySelector('.anscard2'));
  const why = getComputedStyle(host.querySelector('.ac-why'));
  const out = { bg: cs.backgroundColor, border: cs.borderTopColor, whyColor: why.color };
  host.remove();
  // must resolve to real colours, not transparent or the dark-surface values
  out.ok = out.bg !== 'rgba(0, 0, 0, 0)' &&
           !out.bg.startsWith('rgba(255, 255, 255, 0.05') &&
           out.whyColor !== 'rgba(0, 0, 0, 0)';
  return out;
}
```

Expected: `ok: true`, with `bg` resolving to the paper-2 colour rather than the
dark surface's translucent white.

- [ ] **Step 5: Clean up the stale counter literal**

`S-D2`'s static pre-JS markup still reads `Question <b id="qn">1</b> / 20`.
`render()` overwrites it synchronously before paint, so it is never visibly
wrong, but it is a stale literal that misleads anyone reading the raw HTML and
would surface if `MMQ` ever failed to load. Change the `20` to `—`:

```html
        <div class="turnflag" id="turnflag">Question <b id="qn">1</b> / —</div>
```

- [ ] **Step 6: Commit any fixes**

```bash
git add index.html
git commit -m "fix(questions): resolve full-flow regression findings"
```

---

## Self-review

**Spec coverage.** §1.1 → Tasks 2, 3, 4. §1.2 → Tasks 1, 5 (and exercised in 9). §1.3 → Task 6. §1.4 → Tasks 1, 8. §1.5 → Task 1 supplies `mirror`/`mirrorPairs`; rendering is explicitly deferred to Phase 2 and recorded at the top of this plan. §1.6 → Tasks 6, 7. §1.7 → Task 2. No gaps.

**Type consistency.** `MMQ.answerCard(q, ans, label)` is defined in Task 2 and called with that exact signature in Tasks 3 and 4. `MMQ.confidence(probe, answers)` is defined in Task 1 and called with both arguments in Task 8. Questions carry `{id, dim, depth, probe, text, opts:[{t,v}], src, author, mirror}` throughout; Task 3 reads `t.q.dim` after the turn shape changes, and Steps 5–6 of that task fix every former `t.dim` reference.

**Known sharp edge.** Task 3 changes the shape of `TURNS` from flat (`t.dim`, `t.q` as a string) to nested (`t.q.dim`, `t.q.text`). Step 4 rewrites the render body and Step 5 rewrites both scorers, but any other `t.dim` or `t.q` string usage left in `S-D2` will break silently. Grep for `t\.dim` and `t\.a\b` inside `S-D2` before running the Task 3 assertion.
