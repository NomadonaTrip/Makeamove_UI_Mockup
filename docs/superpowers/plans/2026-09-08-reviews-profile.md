# Reviews & Personality Profile Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the date decline cost something and reach a real person, make the post-date review answer the one question calibration consumes, and put the receipts behind the personality profile's claims on screen.

**Architecture:** Three pure additions to the existing `MMQ` module — a `DEMO.stated` fixture, a `divergences()` derivation over it, and a `reviewReason()` gate that is a sibling to the existing `reviewQuestion()`. Six screens consume them: `S-E2`'s bare decline link becomes a blocking gate at the new `S-E8`, whose text is echoed verbatim to the declined party at the new `S-E9`; `S-E6` grows a stated-vs-played card rendered by the existing `MMCONF.paint()`; the new `S-E6A` lists the full evidence trail; and `S-E5` gains a compulsory number-exchange question modelled on `S-F6`.

**Tech Stack:** HTML + CSS + vanilla JavaScript, single file, no build step, no libraries. Verification is browser-driven via the Playwright MCP tools — this repo has no test runner, so every "test" below is an assertion evaluated in the page.

**Spec:** `docs/superpowers/specs/2026-09-08-reviews-profile-design.md`. Read it alongside this plan — it carries the rejected alternatives behind each locked decision, and the reasoning for the divergence fixture's exact values.

## Global Constraints

- **Single file.** All changes in `/mnt/e/TOOLMAKER/PYTHON/MakeaMoveUI_Mockup/index.html`. No new runtime files, no build step, no libraries, no external CSS/JS. Do not touch `index_mobile_mockup.html` or `index_mockup.html`.
- **Reuse existing components:** `.btn`, `.card`, `.row`, `.chip`, `.opt`, `.dot`, `.note` (+ `.info` / `.gold` / `.danger`), `.field`, `.textarea`, `.confrow` + `.cf-*`, `.mirrorrow` + `.mr-*`, `.eyebrow`, `.muted`, `.pbar`, `.body`, `.foot`, `.btnrow`, `.link`, `.tag`, `.av`. Invent no new colours — every colour comes from a `:root` custom property.
- **`S-E6A` must be added to the `FEED` set** in `classify()`. `S-E8` and `S-E9` must **not** be added to `FEED` or `IMMERSIVE`.
- **The decline reason is set with `textContent`, never interpolated into an `innerHTML` string.** This is the only user-typed string in the prototype that is displayed back to a different user. Every neighbouring render in this file builds HTML by concatenation, so the local idiom is the trap here.
- **Nothing derived may be hardcoded.** The divergence rows, the evidence rows and the trail's counts are all computed from `MMQ.answers()` and must move when the dev depth toggle changes.
- **Do not break:** `window.pick`, `window.tog`, `window.togCap`, `window.DSHOW`, `window.FSHOW`, or `MMQ` / `MMASYNC` / `MMLADDER` / `MMSCHED` / `MMDECK` / `MMDEV` / `MMATTACH` / `MMDECLINE`. `MMCONF` is **extended**, not replaced — its `reject()` contract and existing confidence rows must still work exactly as they do today.
- **Commit after every task**, on the branch `feat/reviews-profile` (already created and holding the spec commit).

## Line numbers shift as you go

Every `index.html:NNNN` reference below was accurate when this plan was written. **Task 1 adds ~70 lines inside `MMQ`** and **Tasks 2, 3 and 5 insert whole `<section>` blocks**, so every citation after those points drifts. Each step quotes the exact text it is replacing — **locate by that text, not by the number.** The numbers are a hint about where to look, nothing more.

## Verification setup

`file://` is blocked by the Playwright MCP browser. A static server is likely **already running** on port 3300 from earlier phases — check before starting one, because `python3 -m http.server 3300` fails with `EADDRINUSE` if it is:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:3300/index.html   # 200 = already serving
cd /mnt/e/TOOLMAKER/PYTHON/MakeaMoveUI_Mockup && python3 -m http.server 3300  # only if the above failed
```

Assertions run at **1280×800** unless a step says otherwise, against `http://127.0.0.1:3300/index.html#<screen>`.

**Tooling gotchas carried from Phases 1–4 — each cost real time there:**

- Never call `location.reload()` inside an evaluated function; it destroys the execution context and the evaluate fails instead of returning.
- Re-navigating to an identical URL serves a stale cached copy. Append a fresh cache-buster (`?v=2`, `?v=3`…) **before** the `#hash` every time you need new code.
- Resize **before** navigating — desktop archetype classes are applied on load.
- **Never assert against a screen's `textContent`.** The inline `<script>` fixtures sit inside the `<section>`, so `textContent` contains every fixture string and any substring assertion false-positives. Query rendered elements.
- **Measuring overflow on these screens:** `.body` and the stage columns are centred flex columns, so content spills in *both* directions and `scrollHeight` counts only part of it. Compute overflow from child rects against the container's box, and let entry animations settle first. A Phase 3 review under-reported a spill by ~115px this way.

## The depth toggle is the demo

`MMDEV.setDepth()` (`index.html:4444`) already calls `MMCONF.paint()`. That means extending `MMCONF.paint()` in Task 4 — rather than adding a second inline script on `S-E6` — is what makes the Dev-settings **Thin / Complete** switch repaint the divergence card for free. Do not add a separate paint path.

Note that `MMQ.recordAnswer()` also writes `mm_answers`, so playing the sync show flips the Dev panel's depth indicator to Complete. That is pre-existing behaviour; do not try to fix it here.

---

### Task 1: The `MMQ` data layer — `stated`, `divergences`, `reviewReason`

**Files:**
- Modify: `index.html` — inside the `MMQ` IIFE (`<script id="mmq">`, around `index.html:1003–1115`).

**Interfaces:**
- Consumes: `MMQ`'s existing `qById`, `answers`, `DEMO`, `o(t,v)`.
- Produces on `window.MMQ`:
  - `DEMO.stated` — `{ [probe]: { v, t, src } }`
  - `majority(probe, ans?)` → `{ v, t, n, tie }`
  - `divergences(ans?)` → `[{ probe, state, said, played }]` where `state` ∈ `'agrees' | 'diverges' | 'unsettled' | 'unplayed'`
  - `reviewReason(text)` → `{ ok, code, reason }` where `code` ∈ `'ok' | 'short' | 'mush' | 'abuse'`

Tasks 2–6 all read these names. They do not change.

- [ ] **Step 1: Write the browser assertion**

Run this first, before writing any code, so you see it fail. Navigate to `#S-E6` and evaluate:

```js
() => {
  const Q = window.MMQ;
  if (!Q || !Q.divergences) return { ok:false, reason:'MMQ.divergences not defined' };

  const byProbe = rows => { const m = {}; rows.forEach(r => m[r.probe] = r.state); return m; };

  // Thin = the sparse DEMO.answers fixture.
  const thin = byProbe(Q.divergences(Q.DEMO.answers));
  // Complete = the twelve-answer fixture the dev toggle installs.
  const full = byProbe(Q.divergences(Q.DEMO.answersFull));

  const R = Q.reviewReason;
  const good  = R("You were lovely but I didn't feel any spark, and I'd rather say that than go quiet on you.");
  const short = R("I didn't feel a spark, sorry.");                        // 29 chars
  const mush  = R("nothing really nothing really nothing really nothing"); // 51 chars, 2 distinct words
  const thin2 = R("There was no chemistry for me at all here.");           // 41 chars, 3 content words
  const mash  = R("asdfgh asdfgh asdfgh asdfgh asdfgh asdfgh asdfgh");     // no vowels to speak of
  const cruel = R("Honestly you were far uglier than your photos and I felt lied to the whole evening.");

  return {
    ok: thin.wantsChildren === undefined &&                 // keys are probe ids, not camelCase
        thin['wants-children'] === 'agrees'   &&
        thin['core-value']     === 'diverges' &&
        thin['money-model']    === 'diverges' &&
        thin['weekend']        === 'unsettled' &&
        thin['closeness']      === 'unplayed' &&
        thin['five-year']      === 'unplayed' &&
        full['wants-children'] === 'agrees'   &&
        full['core-value']     === 'diverges' &&
        full['money-model']    === 'diverges' &&
        full['closeness']      === 'diverges' &&
        full['weekend']        === 'unplayed' &&
        full['five-year']      === 'unplayed' &&
        Q.divergences(Q.DEMO.answers).every(r => r.said && r.said.t && r.said.src) &&
        good.ok === true   && good.code  === 'ok'    &&
        short.ok === false && short.code === 'short' &&
        mush.ok === false  && mush.code  === 'mush'  &&
        thin2.ok === false && thin2.code === 'mush'  &&
        mash.ok === false  && mash.code  === 'mush'  &&
        cruel.ok === false && cruel.code === 'abuse',
    thin, full,
    review: { good:good.code, short:short.code, mush:mush.code,
              thin2:thin2.code, mash:mash.code, cruel:cruel.code }
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `http://127.0.0.1:3300/index.html#S-E6`, evaluate the above.
Expected: `{ ok:false, reason:'MMQ.divergences not defined' }`.

- [ ] **Step 3: Add `DEMO.stated` to the fixtures**

Find this exact text inside `MMQ`'s `DEMO` object (it is the closing of `verdict`, `index.html:1056–1062`):

```js
    /* David's verdict on each of YOUR answers, keyed by question id. */
    verdict: {
      'q-fam-01':'move', 'q-fam-02':'move', 'q-fam-03':'move', 'q-fam-04':'move',
      'q-fam-05':'move', 'q-fam-06':'move', 'q-val-01':'move', 'q-val-02':'move',
      'q-val-03':'stay', 'q-lif-01':'move', 'q-lif-02':'stay', 'q-fin-01':'move',
      'q-fin-02':'move', 'q-amb-01':'move', 'q-int-01':'move'
    }
  };
```

Replace it with the same block plus `stated`:

```js
    /* David's verdict on each of YOUR answers, keyed by question id. */
    verdict: {
      'q-fam-01':'move', 'q-fam-02':'move', 'q-fam-03':'move', 'q-fam-04':'move',
      'q-fam-05':'move', 'q-fam-06':'move', 'q-val-01':'move', 'q-val-02':'move',
      'q-val-03':'stay', 'q-lif-01':'move', 'q-lif-02':'stay', 'q-fin-01':'move',
      'q-fin-02':'move', 'q-amb-01':'move', 'q-int-01':'move'
    },

    /* ---- what you SAID about yourself (spec 5.3) ----
       The claimed side of stated-vs-revealed. Five of the six trace to a
       field a user actually filled in on S-P1–S-P5, and `src` records which,
       so a reader can check the fixture rather than trust it.

       money-model is the exception and is marked INFERRED: no profile screen
       asks how you would pool money. It is kept because it is one of only two
       probes with three answered variations at both depths, which makes it the
       steadiest divergence in the set. If a later phase adds a money-model
       field to the profile, replace the inference with the real claim. */
    stated: {
      'wants-children': { v:'yes',       t:'You want children — two of them',
                          src:'S-P5 · “How many kids do you wish to have?” → 2' },
      'five-year':      { v:'family',    t:'Settled with a family, and soon',
                          src:'S-P5 · “So eager — ready now”' },
      'core-value':     { v:'ambition',  t:'Ambition — a partner who plans',
                          src:'S-P3 · “Wit, ambition, a man who plans”' },
      'weekend':        { v:'adventure', t:'Out and moving — classes, hikes',
                          src:'S-P3 · “Salsa classes, weekend hikes”' },
      'closeness':      { v:'talk',      t:'Talking it through, at length',
                          src:'S-P4 · “Acts of service, long voice notes”' },
      'money-model':    { v:'joint',     t:'Fully joint — one pot',
                          src:'INFERRED from S-P1 income + S-P2 “Rich / Average”' }
    }
  };
```

- [ ] **Step 4: Add `majority` and `divergences`**

Find this exact text (`index.html:965–968`, just below `confLabel`):

```js
  const confLabel = n => n == null ? 'No data'
                       : n >= CONF_GATE ? 'Confirmed' : 'Still forming';

  const probes = () => [...new Set(BANK.map(q => q.probe))];
```

Replace with:

```js
  const confLabel = n => n == null ? 'No data'
                       : n >= CONF_GATE ? 'Confirmed' : 'Still forming';

  const probes = () => [...new Set(BANK.map(q => q.probe))];

  /* ---- stated vs. revealed (spec 5.3 / item 17) ----
     What you claimed on S-P1–S-P5, set against what you actually played.

     `majority` is deliberately separate from `confidence()` above: confidence
     asks how much your answers AGREE WITH EACH OTHER, this asks WHICH ANSWER
     you gave most. A probe can be perfectly consistent and still contradict
     the claim, which is the whole point of the block. */
  function majority(probe, ans){
    const hits = (ans || answers()).filter(a => {
      const q = qById(a.qid); return q && q.probe === probe;
    });
    if(!hits.length) return { v:null, t:null, n:0, tie:false };
    const tally = {}, label = {};
    hits.forEach(a => {
      const opt = qById(a.qid).opts[a.opt];
      tally[opt.v] = (tally[opt.v] || 0) + 1;
      if(!label[opt.v]) label[opt.v] = opt.t;
    });
    const ranked = Object.keys(tally).sort((x,y) => tally[y] - tally[x]);
    const top = ranked[0];
    return { v:top, t:label[top], n:hits.length,
             tie: ranked.length > 1 && tally[ranked[1]] === tally[top] };
  }

  /* Four states, and collapsing any two of them would lose the honesty the
     block exists for: a tie is not a divergence, and an unplayed probe is
     certainly not agreement. */
  function divergences(ans){
    ans = ans || answers();
    return Object.keys(DEMO.stated).map(p => {
      const said = DEMO.stated[p], played = majority(p, ans);
      const state = played.n === 0 ? 'unplayed'
                  : played.tie     ? 'unsettled'
                  : played.v === said.v ? 'agrees' : 'diverges';
      return { probe:p, state, said, played };
    });
  }
```

- [ ] **Step 5: Add `reviewReason`**

Find this exact text (`index.html:978–988`):

```js
  /* ---- the fake LLM (spec 1.3) — deterministic, so it is testable ---- */
  const BLOCKED = ['salary','virgin','body count','how much do you earn','net worth'];
```

Replace with:

```js
  /* ---- the fake LLM (spec 1.3) — deterministic, so it is testable ---- */
  const MIN_REASON = 40;   // characters, spec 5.2
  const CRUEL = ['ugly','uglier','fat','disgusting','pathetic','loser','stupid',
                 'idiot','worthless','gross','hideous','repulsive','desperate'];

  /* Sibling to reviewQuestion, not a reuse of it: that one enforces question
     shape (a trailing '?', >=12 chars), which is wrong for a reason, and this
     one screens for cruelty, which questions never needed (spec 5.2).

     The substance rule is DISTINCT CONTENT WORDS, not a phrase blocklist. A
     blocklist of "nothing really" / "idk" would be dead code, because the
     40-character floor rejects every one of those phrases before the check is
     reached — padding is the only way mush arrives long enough to test, and
     padding is exactly what a distinct-word count catches.

     Known and accepted: this rejects thin-but-honest lines like "There was no
     chemistry for me at all here." Per spec 5.1 the friction is the point, so
     pushing a decliner past their first thin sentence is correct behaviour
     here, not a false positive. */
  function reviewReason(text){
    const t = String(text || '').trim();
    if (t.length < MIN_REASON)
      return { ok:false, code:'short',
               reason:'Say a little more — at least ' + MIN_REASON + ' characters.' };

    const low = t.toLowerCase().replace(/[^a-z0-9\s]/g, ' ').replace(/\s+/g, ' ').trim();
    const words = low ? low.split(' ') : [];
    const letters = low.replace(/\s/g, '');
    const vowels = (letters.match(/[aeiou]/g) || []).length;
    const distinct = new Set(words.filter(w => w.length >= 4)).size;

    const mush = { ok:false, code:'mush',
                   reason:'That doesn’t say anything he can learn from. Try again.' };
    if (!letters.length) return mush;
    if (vowels / letters.length < 0.15) return mush;   // keyboard mash
    if (/(.)\1{4,}/.test(low))          return mush;   // aaaaaaaa
    if (distinct < 5)                   return mush;   // padded repetition

    if (CRUEL.some(w => new RegExp('\\b' + w + '\\b').test(low)))
      return { ok:false, code:'abuse',
               reason:'He is going to read this. Say the true thing without the sting.' };

    return { ok:true, code:'ok', reason:'Cleared' };
  }

  const BLOCKED = ['salary','virgin','body count','how much do you earn','net worth'];
```

- [ ] **Step 6: Export the new names**

Find this exact text (`index.html:1111–1115`):

```js
  window.MMQ = { DIMS, BANK, DEMO, CONF_GATE, MAX_DECK, ATTEMPTS,
                 qById, deck, saveDeck, model, setModel, answers,
                 buildSet, confidence, confLabel, probes, mirrorPairs,
                 reviewQuestion, genOptions, answerCard, o,
                 setProfileDepth, recordAnswer };
```

Replace with:

```js
  window.MMQ = { DIMS, BANK, DEMO, CONF_GATE, MAX_DECK, ATTEMPTS, MIN_REASON,
                 qById, deck, saveDeck, model, setModel, answers,
                 buildSet, confidence, confLabel, probes, mirrorPairs,
                 majority, divergences, reviewReason,
                 reviewQuestion, genOptions, answerCard, o,
                 setProfileDepth, recordAnswer };
```

- [ ] **Step 7: Run the assertion to verify it passes**

Navigate to `http://127.0.0.1:3300/index.html?v=1#S-E6` (fresh cache-buster), evaluate the Step 1 function.
Expected: `ok: true`, with `thin` and `full` matching the two tables in the spec.

- [ ] **Step 8: Verify nothing regressed**

Navigate to `#S-E6` and evaluate:

```js
() => {
  const rows = document.querySelectorAll('#confBlock .confrow');
  return { ok: rows.length > 0 && !!window.MMCONF && !!window.MMQ.reviewQuestion(
             'Do you want children of your own someday?').ok,
           confRows: rows.length };
}
```
Expected: `ok: true`. The existing confidence block still renders and `reviewQuestion` still works.

- [ ] **Step 9: Commit**

```bash
git add index.html
git commit -m "feat(profile): add stated-vs-played and the decline-reason reviewer to MMQ

DEMO.stated is the claimed side of spec 5.3, each value traced to the
S-P1-P5 field it came from (money-model marked INFERRED — no screen asks
it). divergences() reports four states because collapsing any two loses
the honesty the block is for.

reviewReason is a sibling to reviewQuestion rather than a reuse: question
shape is wrong for a reason, and 5.2 adds a cruelty screen. Substance is
measured in distinct content words, since the 40-char floor makes a
phrase blocklist dead code."
```

---

### Task 2: `S-E8` — the decline gate, and rewiring `S-E2`

**Files:**
- Modify: `index.html` — `S-E2`'s foot link; new `<section id="S-E8">` inserted after `S-E7`'s closing `</section>`.

**Interfaces:**
- Consumes: `MMQ.reviewReason`, `MMQ.MIN_REASON` (Task 1).
- Produces on `window`: `MMDECLINEDATE` with `check()` (called on every input) and `submit()` (returns `false` to block navigation). Writes `sessionStorage.mm_decline_reason` as `{ text, at }`. Task 3's `S-E9` reads that key.

Named `MMDECLINEDATE` because `MMDECLINE` already exists on `S-F6` for the *async* decline (`index.html:3991`) and must not be shadowed.

- [ ] **Step 1: Write the browser assertion**

Navigate to `#S-E8` and evaluate:

```js
() => {
  const el = document.getElementById('S-E8');
  if (!el) return { ok:false, reason:'S-E8 does not exist' };
  const ta   = document.getElementById('declineText');
  const btn  = document.getElementById('declineGo');
  const warn = document.getElementById('declineWarn');
  const M    = window.MMDECLINEDATE;
  if (!ta || !btn || !M) return { ok:false, reason:'S-E8 controls missing' };

  const type = v => { ta.value = v; M.check(); };
  const disabled = () => btn.classList.contains('disabled');

  sessionStorage.removeItem('mm_decline_reason');

  type('');                                   const empty = disabled();
  type('I did not feel a spark, sorry.');      const short = disabled();
  type('nothing really nothing really nothing really nothing');
  const mushEnabled = !disabled();            // 51 chars: the counter lets it through
  const mushBlocked = M.submit() === false;   // the reviewer stops it on submit
  const mushWarned  = /learn from/.test(warn.textContent) && warn.className.indexOf('danger') !== -1;

  type('Honestly you were far uglier than your photos and I felt lied to the whole evening.');
  const cruelBlocked = M.submit() === false;
  const cruelWarned  = /without the sting/.test(warn.textContent);

  const real = "You were lovely but I didn't feel any spark, and I'd rather say that than go quiet on you.";
  type(real);
  const okEnabled = !disabled();
  const okSubmit  = M.submit() === true;
  const stored    = JSON.parse(sessionStorage.getItem('mm_decline_reason') || 'null');

  return {
    ok: empty && short && mushEnabled && mushBlocked && mushWarned &&
        cruelBlocked && cruelWarned && okEnabled && okSubmit &&
        stored && stored.text === real,
    empty, short, mushEnabled, mushBlocked, mushWarned,
    cruelBlocked, cruelWarned, okEnabled, okSubmit, stored
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `http://127.0.0.1:3300/index.html?v=2#S-E8`, evaluate.
Expected: `{ ok:false, reason:'S-E8 does not exist' }`.

- [ ] **Step 3: Rewire `S-E2`'s decline link**

Find this exact text (`index.html:3381`):

```html
          <a class="link center" href="#S-B2">Not quite — end politely (credit kept)</a>
```

Replace with:

```html
          <a class="link center" href="#S-E8">Not quite — end politely (credit kept)</a>
```

The copy is unchanged: the credit is still kept, and this is still the polite exit. What changes is that it now costs a sentence.

- [ ] **Step 4: Insert `S-E8`**

Find this exact text — the end of `S-E7` and the start of Group F (`index.html:3538–3542`):

```html
          <div class="btnrow"><a class="btn ghost block" href="#S-B2">Done</a><a class="btn move block" href="#S-E7">Share invite</a></div>
        </div>
      </section>

      <!-- ============ GROUP F · ASYNC GAME (threshold-activated) ============ -->
```

Replace with the same, plus the new section before the Group F comment:

```html
          <div class="btnrow"><a class="btn ghost block" href="#S-B2">Done</a><a class="btn move block" href="#S-E7">Share invite</a></div>
        </div>
      </section>

      <section class="screen paper" id="S-E8" data-group="E · Post-show" data-title="⚠ Date declined · reason">
        <div class="pbar"><a class="back" href="#S-E2">‹</a><h1>Before you go</h1></div>
        <div class="body">
          <div class="note">David will read this, in your words — we don't summarise it and we don't soften it. Your credit is kept either way. A real sentence is kinder than silence, and it's the only thing we ask for.</div>
          <div id="declineWarn" class="note danger" style="display:none"></div>
          <div class="field">
            <label>Why isn't this one for you?
              <span id="declineCount" style="float:right;font-weight:800;color:var(--t-faint)"></span>
            </label>
            <textarea class="textarea" id="declineText" rows="5"
              placeholder="The honest version. He'd rather hear it than wonder."
              oninput="MMDECLINEDATE.check()"></textarea>
          </div>
          <div class="note info">No options to pick from here — this one is meant to take a moment. The post-date review is where we ask the tick-box questions.</div>
        </div>
        <div class="foot">
          <a class="btn move block disabled" id="declineGo" href="#S-B2"
             onclick="return MMDECLINEDATE.submit()">Send &amp; go back to matches</a>
          <a class="link center" href="#S-E9">Demo: see what David receives ›</a>
        </div>
        <script>
        (function(){
          /* §5.1 — a written reason, shared verbatim, with no options offered:
             the friction is the point. §5.2 — 40 characters AND a substance
             check, which does double duty as an abuse screen now that the text
             reaches a real person.

             Two stages on purpose. The counter gates length live, because that
             is a rule the user can see themselves meeting. The substance and
             cruelty checks run on SUBMIT, because flagging cruelty
             keystroke-by-keystroke would be jumpy and would teach the writer to
             game the check rather than to say the true thing.

             There is no escape hatch. Unlimited rewrites, no attempt counter,
             no ghost exit — spec 5.1/5.2, decided with the cost understood. */
          const ta   = () => document.getElementById('declineText');
          const btn  = () => document.getElementById('declineGo');
          const warn = () => document.getElementById('declineWarn');

          function check(){
            const t = (ta().value || '').trim();
            const n = t.length, min = MMQ.MIN_REASON;
            document.getElementById('declineCount').textContent =
              n >= min ? n + ' characters' : (min - n) + ' more to go';
            btn().classList.toggle('disabled', n < min);
            if(n < min) warn().style.display = 'none';   // clear a stale verdict
            return n >= min;
          }

          function submit(){
            const t = (ta().value || '').trim();
            const verdict = MMQ.reviewReason(t);
            if(!verdict.ok){
              const w = warn();
              w.className = 'note danger';
              w.textContent = verdict.reason;
              w.style.display = '';
              ta().focus();
              return false;                              // blocks the navigation
            }
            warn().style.display = 'none';
            sessionStorage.setItem('mm_decline_reason',
              JSON.stringify({ text:t, at:Date.now() }));
            return true;
          }

          window.MMDECLINEDATE = { check, submit };
          check();
        })();
        </script>
      </section>

      <!-- ============ GROUP F · ASYNC GAME (threshold-activated) ============ -->
```

- [ ] **Step 5: Run the assertion to verify it passes**

Navigate to `http://127.0.0.1:3300/index.html?v=3#S-E8`, evaluate the Step 1 function.
Expected: `ok: true`.

- [ ] **Step 6: Verify the gate is reachable and the screen index picked it up**

Navigate to `#S-E2` and evaluate:

```js
() => {
  const link = [...document.querySelectorAll('#S-E2 .foot a')]
    .find(a => /Not quite/.test(a.textContent));
  const indexed = [...document.querySelectorAll('#idxlist a, .idx a, #screenindex a')]
    .some(a => a.getAttribute('href') === '#S-E8');
  return { ok: link && link.getAttribute('href') === '#S-E8',
           href: link && link.getAttribute('href'), indexed };
}
```
Expected: `ok: true`. (`indexed` is informational — the ☰ index is built from the DOM, so a new `<section>` joins it automatically; if it reads `false`, check the selector rather than assuming a bug.)

- [ ] **Step 7: Screenshot both viewports**

Screenshot `#S-E8` at **1280×800** and at **390×844** (resize before navigating). Confirm: the button reads disabled (dimmed) on arrival, the counter shows "40 more to go", and nothing overflows.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "feat(decline): make the post-call decline a written gate (S-E8)

S-E2's bare link to #S-B2 becomes a blocking screen: 40 characters live,
then a substance and cruelty check on submit. No attempt counter and no
alternative exit — spec 5.1/5.2, decided with the cost understood.

Named MMDECLINEDATE so it does not shadow S-F6's MMDECLINE, which is the
async decline and a different thing."
```

---

### Task 3: `S-E9` — what David receives

**Files:**
- Modify: `index.html` — new `<section id="S-E9">` inserted after `S-E8`'s closing `</section>`.

**Interfaces:**
- Consumes: `sessionStorage.mm_decline_reason` written by Task 2.
- Produces: nothing other tasks consume. `window.MMDECLINERECV` with `paint()` exists so the router-driven repaint is testable.

- [ ] **Step 1: Write the browser assertion**

This is the task where a mistake is a real defect rather than a cosmetic one, so the assertion checks escaping explicitly. Navigate to `#S-E9` and evaluate:

```js
() => {
  const host = document.getElementById('e9reason');
  if (!host || !window.MMDECLINERECV) return { ok:false, reason:'S-E9 not built' };

  // 1. A plain reason renders verbatim.
  const plain = "You were lovely but I didn't feel any spark, and I'd rather say that than go quiet on you.";
  sessionStorage.setItem('mm_decline_reason', JSON.stringify({ text:plain, at:Date.now() }));
  MMDECLINERECV.paint();
  const verbatim = host.textContent === plain;

  // 2. Markup in the reason is TEXT, not markup. This is the whole rule.
  const nasty = 'I felt <b>nothing</b> and <img src=x onerror="window.__pwned=1"> we both knew it, honestly.';
  sessionStorage.setItem('mm_decline_reason', JSON.stringify({ text:nasty, at:Date.now() }));
  MMDECLINERECV.paint();
  const escaped = host.textContent === nasty &&
                  host.querySelector('b') === null &&
                  host.querySelector('img') === null &&
                  window.__pwned === undefined;

  // 3. Empty storage falls back to a fixture rather than rendering blank.
  sessionStorage.removeItem('mm_decline_reason');
  MMDECLINERECV.paint();
  const fallback = host.textContent.trim().length > 40;

  return { ok: verbatim && escaped && fallback,
           verbatim, escaped, fallback, fallbackText: host.textContent.trim() };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `http://127.0.0.1:3300/index.html?v=4#S-E9`, evaluate.
Expected: `{ ok:false, reason:'S-E9 not built' }`.

- [ ] **Step 3: Insert `S-E9`**

Find the closing of `S-E8` and the Group F comment you created in Task 2:

```html
        })();
        </script>
      </section>

      <!-- ============ GROUP F · ASYNC GAME (threshold-activated) ============ -->
```

Replace with:

```html
        })();
        </script>
      </section>

      <section class="screen dark" id="S-E9" data-group="E · Post-show" data-title="⚠ You were passed on">
        <div class="body" style="justify-content:center;text-align:center;gap:18px;padding:40px 26px">
          <div class="av g5 s72" style="margin:0 auto"><svg class="icon" style="width:30px;height:30px"><use href="#i-mailheart"/></svg></div>
          <h2 class="scrn" style="font-size:28px;max-width:16ch;margin:0 auto">Tolu isn't taking this one further.</h2>
          <p class="muted" style="max-width:32ch;margin:0 auto;line-height:1.55">She wrote you a reason. We haven't edited it or softened it — these are her words.</p>
          <div class="card" style="text-align:left">
            <div class="eyebrow" style="color:var(--bulb)">In her words</div>
            <div id="e9reason" class="e9quote"></div>
          </div>
          <div class="note">No penalty, nothing charged, and your karma is untouched. Being passed on after a good call is ordinary — it costs you nothing here.</div>
          <div class="btnrow"><a class="btn ghost block" href="#S-G1">See your credits</a><a class="btn move block" href="#S-B2">Back to your matches</a></div>
        </div>
        <script>
        (function(){
          /* §5.1 — the reason reaches a real person, verbatim. That makes this
             the ONE place in the prototype where a user-typed string is shown
             to a different user, so it is set with textContent and never
             concatenated into innerHTML. Every neighbouring render in this file
             builds HTML by string concatenation; copying that idiom here would
             be the bug.

             Falls back to a fixture when storage is empty, the same way S-F4's
             paint() does and for the same reason: this screen is reachable by
             deep link, by Prev/Next and from the index, not only by having just
             written a reason. */
          const FALLBACK = "You were genuinely lovely and I nearly talked myself into it. "
                         + "But I didn't feel the thing I'm holding out for, and you deserve "
                         + "someone who does. I'd rather say that than go quiet on you.";
          function paint(){
            const host = document.getElementById('e9reason');
            if(!host) return;
            let stored = null;
            try { stored = JSON.parse(sessionStorage.getItem('mm_decline_reason')); }
            catch(e){ stored = null; }
            const text = (stored && stored.text) ? stored.text : FALLBACK;
            host.textContent = text;          // never innerHTML — see above
          }
          window.MMDECLINERECV = { paint };
          window.addEventListener('hashchange', () => {
            if(location.hash === '#S-E9') paint();
          });
          paint();
        })();
        </script>
      </section>

      <!-- ============ GROUP F · ASYNC GAME (threshold-activated) ============ -->
```

- [ ] **Step 4: Add the quote style**

Find this exact text in the stylesheet (`index.html:651–652`, the end of the mirrorrow block):

```css
  .mirrorrow .mr-flag.soft{color:var(--ink-faint)}
  .paper .mirrorrow .mr-flag.soft{color:var(--t-faint)}
```

Replace with:

```css
  .mirrorrow .mr-flag.soft{color:var(--ink-faint)}
  .paper .mirrorrow .mr-flag.soft{color:var(--t-faint)}

  /* ===== the decline reason, as the declined party reads it (spec 5.1) =====
     Set with textContent, so white-space:pre-wrap is what preserves the line
     breaks the writer actually typed. */
  .e9quote{margin-top:8px; font-size:14px; line-height:1.6; white-space:pre-wrap;
    border-left:2px solid var(--move); padding-left:12px; color:var(--ink)}
  .paper .e9quote{color:var(--t)}
```

`--ink` is the stage-palette body colour and `--t` is the paper one (`index.html:19` and `:24`). `S-E9` is a `dark` screen so only the first rule is exercised today; the `.paper` override is there because every other shared row component in this file carries one, and a later phase that re-skins the screen would otherwise get invisible text.

- [ ] **Step 5: Run the assertion to verify it passes**

Navigate to `http://127.0.0.1:3300/index.html?v=5#S-E9`, evaluate the Step 1 function.
Expected: `ok: true`, `escaped: true` in particular.

- [ ] **Step 6: Verify the round trip end to end**

Navigate to `#S-E8`, then evaluate:

```js
() => {
  const real = "I keep coming back to the fact that you want to stay in London and I don't, honestly.";
  document.getElementById('declineText').value = real;
  MMDECLINEDATE.check();
  const sent = MMDECLINEDATE.submit();
  location.hash = '#S-E9';
  return new Promise(r => setTimeout(() => {
    const shown = document.getElementById('e9reason').textContent;
    r({ ok: sent === true && shown === real, sent, shown });
  }, 250));
}
```
Expected: `ok: true` — what was typed on `S-E8` is what `S-E9` displays.

- [ ] **Step 7: Screenshot both viewports**

Screenshot `#S-E9` at **1280×800** and **390×844**. Confirm the quote rule renders, the dark palette is correct, and a long reason does not overflow the card.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "feat(decline): render the reason to the declined party (S-E9)

Verbatim, with no reply control. Set with textContent, not innerHTML:
this is the only user-typed string in the prototype displayed back to a
different user, and every neighbouring render here concatenates HTML, so
the local idiom is the trap. Asserted against markup in the reason.

Falls back to a fixture on empty storage, matching S-F4's paint()."
```

---

### Task 4: `S-E6` — the stated-vs-played card

**Files:**
- Modify: `index.html` — `S-E6`'s markup (a new `.card` before the Finances nudge) and its inline `MMCONF` script.

**Interfaces:**
- Consumes: `MMQ.divergences` (Task 1).
- Produces: `MMCONF.paint()` now fills `#divBlock` as well as `#confBlock`. Task 5's `S-E6A` is linked from this card.

- [ ] **Step 1: Write the browser assertion**

Navigate to `#S-E6` and evaluate:

```js
() => {
  const host = document.getElementById('divBlock');
  if (!host) return { ok:false, reason:'divBlock not present' };

  const states = () => [...host.querySelectorAll('.confrow')]
    .map(r => r.getAttribute('data-state'));

  MMQ.setProfileDepth('thin');  MMCONF.paint();  const thin = states();
  MMQ.setProfileDepth('full');  MMCONF.paint();  const full = states();
  MMQ.setProfileDepth('thin');  MMCONF.paint();

  const count = s => a => a.filter(x => x === s).length;
  const link = [...document.querySelectorAll('#S-E6 a')]
    .find(a => a.getAttribute('href') === '#S-E6A');

  return {
    ok: thin.length === 6 && full.length === 6 &&
        count('diverges')(thin) === 2 && count('diverges')(full) === 3 &&
        count('unsettled')(thin) === 1 && count('unsettled')(full) === 0 &&
        count('agrees')(thin) === 1 &&
        thin.join() !== full.join() &&          // the toggle visibly moves it
        !!link &&
        document.querySelectorAll('#confBlock .confrow').length > 0,  // still there
    thin, full, hasLink: !!link
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `http://127.0.0.1:3300/index.html?v=6#S-E6`, evaluate.
Expected: `{ ok:false, reason:'divBlock not present' }`.

- [ ] **Step 3: Insert the card**

Find this exact text in `S-E6` (`index.html:3473–3476`, the Finances nudge card):

```html
          <div class="card">
            <div style="font-size:13px">We noticed you <b>Moved</b> on partners who were open about money struggles. Should we lower <b>Finances</b> further?</div>
```

Replace with the new card *above* it — the nudge is a conclusion drawn from this evidence, so it reads better after it:

```html
          <div class="card">
            <div class="eyebrow" style="color:var(--bulb)">What you said vs. what you played</div>
            <div class="muted" style="font-size:11.5px;margin-top:4px">Your profile answers, set against the choices you actually made in the games. Where they part company is usually the interesting bit.</div>
            <div id="divBlock" style="margin-top:10px"></div>
            <a class="link" href="#S-E6A" style="display:inline-block;margin-top:10px;font-size:12px">See the full evidence trail ›</a>
          </div>
          <div class="card">
            <div style="font-size:13px">We noticed you <b>Moved</b> on partners who were open about money struggles. Should we lower <b>Finances</b> further?</div>
```

- [ ] **Step 4: Teach `MMCONF.paint()` to fill it**

Find this exact text in `S-E6`'s inline script (the end of `paint()` and the start of `reject()`, `index.html:3518–3520`):

```js
            }).join('');
          }
          function reject(p){
```

Replace with:

```js
            }).join('');
            paintDiv();
          }

          /* §5.3 — stated vs. revealed. Rendered from the SAME paint() as the
             confidence rows rather than from a second script, so the two
             derived blocks on this screen cannot drift apart when the dev depth
             toggle changes. MMDEV.setDepth already calls MMCONF.paint(), so
             this gets the toggle for free. */
          const STATE = {
            agrees:    { label:'Matches',      line:'You said it, and you played it.' },
            diverges:  { label:'Doesn’t match', line:null },
            unsettled: { label:'Unsettled',    line:'Your answers here don’t agree with each other yet.' },
            unplayed:  { label:'Not played',   line:'No game has asked you this one yet.' }
          };
          function paintDiv(){
            const host = document.getElementById('divBlock'); if(!host) return;
            host.innerHTML = MMQ.divergences().map(r => {
              const meta = STATE[r.state];
              const body = r.state === 'diverges'
                ? '<div class="cf-val">You played: <b>' + r.played.t + '</b></div>'
                  + '<div class="cf-cite">You said: ' + r.said.t + '</div>'
                : '<div class="cf-val">' + (r.state === 'agrees' ? r.played.t : r.said.t) + '</div>'
                  + '<div class="cf-cite">' + meta.line + '</div>';
              return '<div class="confrow" data-probe="' + r.probe + '"'
                + ' data-state="' + r.state + '">'
                + '<div class="cf-head"><span class="cf-name">' + (NAMES[r.probe] || r.probe) + '</span>'
                + '<span class="cf-label">' + meta.label + '</span></div>'
                + body
                + '<div class="cf-cite">From ' + r.said.src + '</div>'
                + '</div>';
            }).join('');
          }

          function reject(p){
```

- [ ] **Step 5: Export `paintDiv` for the assertion**

Find this exact text (`index.html:3526`):

```js
          window.MMCONF = { paint, reject };
```

Replace with:

```js
          window.MMCONF = { paint, paintDiv, reject };
```

- [ ] **Step 6: Add the divergence state styling**

Find this exact text in the stylesheet (`index.html:615`):

```css
  .confrow[data-rejected="true"]{opacity:.45}
```

Replace with:

```css
  .confrow[data-rejected="true"]{opacity:.45}
  /* stated-vs-played (spec 5.3): a divergence is the interesting row, an
     unplayed probe is a quiet one. Both use existing tokens. */
  .confrow[data-state="unplayed"]{opacity:.6}
  .confrow[data-state="diverges"] .cf-label{color:var(--move)}
```

- [ ] **Step 7: Run the assertion to verify it passes**

Navigate to `http://127.0.0.1:3300/index.html?v=7#S-E6`, evaluate the Step 1 function.
Expected: `ok: true`, `thin` showing 2 `diverges` + 1 `unsettled`, `full` showing 3 `diverges` + 0 `unsettled`.

- [ ] **Step 8: Verify the dev toggle repaints it**

Open the Dev panel (the gear in the toolbar), click **Complete**, and screenshot `#S-E6`. Then click **Thin** and screenshot again. The card's rows must visibly differ — `closeness` reads "Doesn't match" only in Complete, `weekend` reads "Unsettled" only in Thin.

- [ ] **Step 9: Screenshot both viewports and measure overflow**

At **1280×800** and **390×844**, screenshot `#S-E6` and evaluate:

```js
() => {
  const body = document.querySelector('#S-E6 .body');
  const box = body.getBoundingClientRect();
  let top = Infinity, bottom = -Infinity;
  [...body.children].forEach(c => { const r = c.getBoundingClientRect();
    top = Math.min(top, r.top); bottom = Math.max(bottom, r.bottom); });
  return { spillTop: Math.max(0, box.top - top),
           spillBottom: Math.max(0, bottom - box.bottom),
           scrollable: body.scrollHeight > body.clientHeight };
}
```
`S-E6` now carries five cards. If it spills, it must scroll — confirm `scrollable: true` rather than content being clipped. Do not use `scrollHeight` alone to judge this; the rect measurement above is the check.

- [ ] **Step 10: Commit**

```bash
git add index.html
git commit -m "feat(profile): show stated vs. played on S-E6

A fifth card above the Finances nudge, which reads better as a
conclusion drawn from this evidence than as a claim preceding it.

Rendered from MMCONF.paint() rather than a second inline script, so both
derived blocks repaint together — and since MMDEV.setDepth already calls
MMCONF.paint(), the Thin/Complete toggle drives it for free."
```

---

### Task 5: `S-E6A` — the evidence trail

**Files:**
- Modify: `index.html` — new `<section id="S-E6A">` inserted after `S-E6`'s closing `</section>`; `FEED` set in `classify()`.

**Interfaces:**
- Consumes: `MMQ.buildSet`, `MMQ.answers`, `MMQ.qById`, `MMQ.DEMO.verdict`, `MMQ.majority` (Task 1).
- Produces: `window.MMTRAIL` with `paint()`.

- [ ] **Step 1: Write the browser assertion**

Navigate to `#S-E6A` and evaluate:

```js
() => {
  const host = document.getElementById('trailRows');
  if (!host || !window.MMTRAIL) return { ok:false, reason:'S-E6A not built' };

  MMQ.setProfileDepth('thin'); MMTRAIL.paint();
  const rows = [...host.querySelectorAll('.mirrorrow')];
  const set  = MMQ.buildSet();
  const answered = rows.filter(r => r.getAttribute('data-answered') === 'true');
  const blanks   = rows.filter(r => r.getAttribute('data-answered') === 'false');
  const head = document.getElementById('trailHead').textContent;

  // The unanswered rows must SAY they are unanswered, not show a default option.
  const honest = blanks.every(r => /not answered yet/i.test(r.textContent));
  // Every answered row must name the probe it feeds.
  const inferred = answered.every(r => (r.querySelector('.mr-flag') || {}).textContent);

  MMQ.setProfileDepth('full'); MMTRAIL.paint();
  const fullAnswered = [...host.querySelectorAll('.mirrorrow[data-answered="true"]')].length;
  MMQ.setProfileDepth('thin'); MMTRAIL.paint();

  const feed = document.getElementById('S-E6A').classList.contains('lay-feed');

  return {
    ok: rows.length === set.length && answered.length > 0 && honest && inferred &&
        fullAnswered !== answered.length &&        // depth toggle moves it
        new RegExp('^' + set.length + ' questions').test(head.trim()) &&
        feed,
    rows: rows.length, setLen: set.length, answered: answered.length,
    fullAnswered, head: head.trim(), honest, inferred, feed
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `http://127.0.0.1:3300/index.html?v=8#S-E6A`, evaluate.
Expected: `{ ok:false, reason:'S-E6A not built' }`.

- [ ] **Step 3: Insert `S-E6A`**

Find this exact text — the end of `S-E6`'s inline script and the start of `S-E7` (`index.html:3528–3532`):

```html
        })();
        </script>
      </section>

      <section class="screen dark" id="S-E7" data-group="E · Post-show" data-title="Referral">
```

Replace with:

```html
        })();
        </script>
      </section>

      <section class="screen dark" id="S-E6A" data-group="E · Post-show" data-title="Evidence trail">
        <div class="pbar"><a class="back" href="#S-E6">‹</a><h1 style="font-size:18px">The evidence trail</h1></div>
        <div class="body">
          <div class="note gold">Every question you've answered, what you picked, how the other person reacted, and what we took from it. Summary on the last screen; receipts here.</div>
          <div class="card">
            <div class="eyebrow" style="color:var(--bulb)" id="trailHead"></div>
            <div id="trailRows" style="margin-top:8px"></div>
          </div>
        </div>
        <div class="foot"><a class="btn ghost block" href="#S-E6">‹ Back to your profile</a></div>
        <script>
        (function(){
          /* §5.3 — "summary by default, receipts on demand". Rows reuse
             .mirrorrow from the async reveal: already themed for dark and
             paper, and already shaped as question-plus-labelled-sides.

             Two states render honestly rather than falling back to a default,
             exactly as S-F4 does: an answered question with no verdict, and a
             question in the set that was never answered. Filling either with a
             default would imply an answer nobody gave (§2.7). */
          const NAMES = {
            'wants-children':'Wanting children', 'children-timing':'Timing for a first child',
            'blending':'Blending two families',  'step-resilience':'Facing a stepchild’s rejection',
            'core-value':'What you value most',  'non-negotiable':'Your non-negotiable',
            'weekend':'How you spend a weekend', 'money-model':'How you handle money',
            'five-year':'Where you are heading', 'closeness':'What closeness means'
          };
          function paint(){
            const host = document.getElementById('trailRows'); if(!host) return;
            const set = MMQ.buildSet(), ans = MMQ.answers();
            const mine = {};
            ans.forEach(a => { mine[a.qid] = a; });

            let answered = 0, verdicts = 0;
            const html = set.map(q => {
              const a = mine[q.id];
              const v = MMQ.DEMO.verdict[q.id] || null;
              if(a) answered++;
              if(a && v) verdicts++;

              if(!a){
                return '<div class="mirrorrow" data-qid="' + q.id + '" data-answered="false">'
                  + '<div class="mr-q">' + q.text + '</div>'
                  + '<div class="mr-side mr-none"><span><span class="mr-who">You:</span> '
                  + 'not answered yet</span><span class="mr-v">—</span></div>'
                  + '</div>';
              }

              /* What we took from it: which probe it feeds, and whether it sits
                 with your other answers on that probe or against them. */
              const m = MMQ.majority(q.probe, ans);
              const same = (ans.filter(x => {
                const qq = MMQ.qById(x.qid); return qq && qq.probe === q.probe;
              }).length) - 1;
              const agrees = q.opts[a.opt].v === m.v;
              const inference = (NAMES[q.probe] || q.probe)
                + (same === 0 ? ' · the only answer we have on this'
                   : agrees ? ' · agrees with your other ' + same +
                              (same === 1 ? ' answer' : ' answers')
                            : ' · pulls against your other ' + same +
                              (same === 1 ? ' answer' : ' answers'));

              return '<div class="mirrorrow" data-qid="' + q.id + '" data-answered="true">'
                + '<div class="mr-q">' + q.text + '</div>'
                + '<div class="mr-side"><span><span class="mr-who">You:</span> '
                + q.opts[a.opt].t + '</span>'
                + (v ? '<span class="mr-v ' + v + '">he ' + v + 's</span>'
                     : '<span class="mr-v">no verdict yet</span>') + '</div>'
                + (a.why ? '<div class="mr-side"><span><span class="mr-who">In your words:</span> '
                           + a.why + '</span></div>' : '')
                + '<span class="mr-flag' + (agrees ? ' soft' : '') + '">' + inference + '</span>'
                + '</div>';
            }).join('');

            host.innerHTML = html;
            document.getElementById('trailHead').textContent =
              set.length + ' questions · ' + answered + ' answered · ' + verdicts + ' with a verdict';
          }
          window.MMTRAIL = { paint };
          window.addEventListener('hashchange', () => {
            if(location.hash === '#S-E6A') paint();
          });
          paint();
        })();
        </script>
      </section>

      <section class="screen dark" id="S-E7" data-group="E · Post-show" data-title="Referral">
```

- [ ] **Step 4: Register the desktop archetype**

Find this exact text (`index.html:4205`):

```js
    const FEED = new Set(['S-B2','S-E3','S-E4','S-A8','S-H1','S-G3','S-H5','S-QD1']);
```

Replace with:

```js
    const FEED = new Set(['S-B2','S-E3','S-E4','S-E6A','S-A8','S-H1','S-G3','S-H5','S-QD1']);
```

`S-E6A` is a long row list; without this it defaults to a centred form sheet on desktop, which is wrong. Do **not** add `S-E8` or `S-E9` — a centred sheet is correct for both of those.

- [ ] **Step 5: Make the depth toggle repaint the trail**

Find this exact text in `MMDEV` (`index.html:4443–4446`):

```js
    setDepth(d){
      MMQ.setProfileDepth(d); MMDEV.paint();
      if(window.MMCONF)  MMCONF.paint();
    }
```

Replace with:

```js
    setDepth(d){
      MMQ.setProfileDepth(d); MMDEV.paint();
      if(window.MMCONF)  MMCONF.paint();
      if(window.MMTRAIL) MMTRAIL.paint();
    }
```

- [ ] **Step 6: Run the assertion to verify it passes**

Navigate to `http://127.0.0.1:3300/index.html?v=9#S-E6A`, evaluate the Step 1 function.
Expected: `ok: true`, `feed: true`, and `fullAnswered` greater than `answered`.

- [ ] **Step 7: Screenshot both viewports and measure overflow**

Screenshot `#S-E6A` at **1280×800** (confirm the feed archetype renders as a list, not a narrow centred sheet) and **390×844**. Run the overflow measurement from Task 4 Step 9, substituting `#S-E6A`. Confirm the list scrolls rather than clipping.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "feat(profile): add the evidence trail (S-E6A)

One row per question in the current set: your answer, his verdict, and
what the system took from it. Rows reuse .mirrorrow from the async
reveal — already themed for both surfaces and already the right shape.

Unanswered questions and answered-but-unjudged ones say so rather than
falling back to a default, matching S-F4 and §2.7. Registered in FEED,
since a centred form sheet would be wrong for a row list."
```

---

### Task 6: `S-E5` — the compulsory number-exchange question

**Files:**
- Modify: `index.html` — `S-E5`'s body and foot, plus a new inline script.

**Interfaces:**
- Consumes: nothing from earlier tasks.
- Produces on `window`: `MMNUM` with `pick(el)`, `why(el)`, `submit()`. Writes `sessionStorage.mm_numbers` as `{ exchanged:boolean, reason:string|null, note:string }`.

- [ ] **Step 1: Write the browser assertion**

Navigate to `#S-E5` and evaluate:

```js
() => {
  const btn = document.getElementById('numGo');
  const why = document.getElementById('numWhy');
  if (!btn || !why || !window.MMNUM) return { ok:false, reason:'S-E5 controls missing' };
  const disabled = () => btn.classList.contains('disabled');
  const hidden = () => why.style.display === 'none';
  const click = sel => { const el = document.querySelector(sel); el.click(); return el; };

  sessionStorage.removeItem('mm_numbers');
  const onArrival = disabled() && hidden();

  click('#S-E5 .opt[data-num="yes"]');
  const afterYes = !disabled() && hidden();
  const yesSubmit = MMNUM.submit() === true;
  const yesStored = JSON.parse(sessionStorage.getItem('mm_numbers'));

  click('#S-E5 .opt[data-num="no"]');
  const afterNo = disabled() && !hidden();      // re-blocked, options revealed
  const noBlocked = MMNUM.submit() === false;

  const opts = document.querySelectorAll('#numWhy .opt[data-why]');
  click('#numWhy .opt[data-why="not-interested"]');
  const afterWhy = !disabled();
  document.getElementById('numNote').value = 'Lovely evening, just not for me.';
  const noSubmit = MMNUM.submit() === true;
  const noStored = JSON.parse(sessionStorage.getItem('mm_numbers'));

  return {
    ok: onArrival && afterYes && yesSubmit && yesStored.exchanged === true &&
        afterNo && noBlocked && opts.length === 6 && afterWhy && noSubmit &&
        noStored.exchanged === false && noStored.reason === 'not-interested' &&
        /Lovely evening/.test(noStored.note),
    onArrival, afterYes, afterNo, noBlocked, afterWhy, optCount: opts.length,
    yesStored, noStored
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `http://127.0.0.1:3300/index.html?v=10#S-E5`, evaluate.
Expected: `{ ok:false, reason:'S-E5 controls missing' }`.

- [ ] **Step 3: Rebuild `S-E5`**

Find this exact text — the whole of `S-E5` (`index.html:3446–3454`):

```html
      <section class="screen paper" id="S-E5" data-group="E · Post-show" data-title="Post-date feedback">
        <div class="pbar"><h1>How did it go?</h1></div>
        <div class="body">
          <div class="card center"><div style="font-size:30px;letter-spacing:6px">★★★★★</div><div class="muted" style="font-size:12px;margin-top:6px">Rate your date with David</div></div>
          <div class="field"><label>A line for us (private)</label><textarea class="textarea">Honestly the best first date in years. We're seeing each other again Friday.</textarea></div>
          <div class="row"><span><svg class="icon" style="width:20px;height:20px"><use href="#i-flag"/></svg></span><div class="grow"><div class="t1">Report a concern</div><div class="t2">Safety issue, misconduct, or a bad actor</div></div><a class="link" href="#S-I1">Report</a></div>
        </div>
        <div class="foot"><a class="btn move block" href="#S-E6">Submit feedback</a></div>
      </section>
```

Replace with:

```html
      <section class="screen paper" id="S-E5" data-group="E · Post-show" data-title="Post-date feedback">
        <div class="pbar"><h1>How did it go?</h1></div>
        <div class="body">
          <div class="card center"><div style="font-size:30px;letter-spacing:6px">★★★★★</div><div class="muted" style="font-size:12px;margin-top:6px">Rate your date with David</div></div>
          <div class="field">
            <label>Did you exchange numbers? <span class="muted" style="font-weight:400">(required)</span></label>
            <div style="display:flex;flex-direction:column;gap:8px">
              <div class="opt" data-num="yes" onclick="MMNUM.pick(this)"><span class="dot"></span>Yes — we swapped numbers</div>
              <div class="opt" data-num="no" onclick="MMNUM.pick(this)"><span class="dot"></span>No, we didn't</div>
            </div>
          </div>
          <div id="numWhy" style="display:none">
            <div class="field">
              <label>What stopped it?</label>
              <div style="display:flex;flex-direction:column;gap:8px">
                <div class="opt" data-why="didnt-get-to-it" onclick="MMNUM.why(this)"><span class="dot"></span>We just didn't get to it</div>
                <div class="opt" data-why="not-interested" onclick="MMNUM.why(this)"><span class="dot"></span>I wasn't interested</div>
                <div class="opt" data-why="they-werent" onclick="MMNUM.why(this)"><span class="dot"></span>They didn't seem interested</div>
                <div class="opt" data-why="keep-in-app" onclick="MMNUM.why(this)"><span class="dot"></span>I'd rather keep it in-app for now</div>
                <div class="opt" data-why="uncomfortable" onclick="MMNUM.why(this)"><span class="dot"></span>I didn't feel comfortable</div>
                <div class="opt" data-why="other" onclick="MMNUM.why(this)"><span class="dot"></span>Something else</div>
              </div>
            </div>
            <div class="field"><label>Anything to add? (optional)</label><textarea class="textarea" id="numNote" placeholder="Only we see this."></textarea></div>
          </div>
          <div class="field"><label>A line for us (private)</label><textarea class="textarea">Honestly the best first date in years. We're seeing each other again Friday.</textarea></div>
          <div class="row"><span><svg class="icon" style="width:20px;height:20px"><use href="#i-flag"/></svg></span><div class="grow"><div class="t1">Report a concern</div><div class="t2">Safety issue, misconduct, or a bad actor</div></div><a class="link" href="#S-I1">Report</a></div>
        </div>
        <div class="foot"><a class="btn move block disabled" id="numGo" href="#S-E6" onclick="return MMNUM.submit()">Submit feedback</a></div>
        <script>
        (function(){
          /* §5.4 — compulsory, and it blocks submission. A "no" reveals MECE
             options plus an optional line, the same shape as S-F6's async
             decline (MMDECLINE).

             Note the deliberate asymmetry with S-E8: the date decline is free
             text ONLY because the friction is its purpose, while this one is
             structured because calibration actually consumes it. Do not make
             them consistent. */
          let exchanged = null, reason = null;

          function pick(el){
            document.querySelectorAll('#S-E5 .opt[data-num]')
              .forEach(o => o.classList.remove('sel'));
            el.classList.add('sel');
            exchanged = el.getAttribute('data-num') === 'yes';
            document.getElementById('numWhy').style.display = exchanged ? 'none' : '';
            if(exchanged){ reason = null;
              document.querySelectorAll('#numWhy .opt[data-why]')
                .forEach(o => o.classList.remove('sel'));
            }
            sync();
          }
          function why(el){
            document.querySelectorAll('#numWhy .opt[data-why]')
              .forEach(o => o.classList.remove('sel'));
            el.classList.add('sel');
            reason = el.getAttribute('data-why');
            sync();
          }
          /* Yes is complete on its own; No needs a reason before it is. */
          function ready(){ return exchanged === true || (exchanged === false && !!reason); }
          function sync(){
            document.getElementById('numGo').classList.toggle('disabled', !ready());
          }
          function submit(){
            if(!ready()) return false;
            sessionStorage.setItem('mm_numbers', JSON.stringify({
              exchanged, reason,
              note: (document.getElementById('numNote') || {}).value || ''
            }));
            return true;
          }
          window.MMNUM = { pick, why, submit };
          sync();
        })();
        </script>
      </section>
```

- [ ] **Step 4: Run the assertion to verify it passes**

Navigate to `http://127.0.0.1:3300/index.html?v=11#S-E5`, evaluate the Step 1 function.
Expected: `ok: true`.

- [ ] **Step 5: Screenshot both viewports**

Screenshot `#S-E5` at **1280×800** and **390×844**, once on arrival (submit dimmed, options hidden) and once after clicking **No** (options revealed, submit dimmed again). Run the Task 4 Step 9 overflow measurement against `#S-E5` in the revealed state — the screen is materially taller than it was.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(review): make the number-exchange question compulsory on S-E5

Blocks submission until answered. A no reveals six MECE options and
re-blocks until one is picked, plus an optional line — S-F6's pattern.

Structured, unlike S-E8's free-text decline, because calibration consumes
this one while the decline's whole purpose is the friction (§5.4)."
```

---

### Task 7: Full-flow verification sweep

**Files:**
- Modify: `index.html` only if the sweep finds a defect.

**Interfaces:**
- Consumes: everything built in Tasks 1–6.
- Produces: nothing. This task is a gate.

Diff review has passed clean on this repo while screens were visibly broken. The sweep is the check that catches that, and it is not optional.

- [ ] **Step 1: Screenshot every touched screen at both viewports**

At **1280×800** and then **390×844** (resize before navigating, fresh cache-buster each time), screenshot: `#S-E2`, `#S-E5`, `#S-E6`, `#S-E6A`, `#S-E8`, `#S-E9`.

Twelve screenshots. Look at each one. Check specifically:

- `S-E6` at 1280 — five cards; the desktop archetype is still a centred sheet and has not become unreadably long.
- `S-E6A` at 1280 — renders as a **feed**, wide, not a narrow centred column.
- `S-E8` at 390 — the counter sits inside the label without wrapping oddly; the disabled button is legibly disabled.
- `S-E9` at 390 — the quote's left rule and padding survive; a long reason wraps rather than overflowing the card.
- `S-E5` at 390 with the "no" branch open — the screen scrolls; nothing is clipped.

- [ ] **Step 2: Walk the whole decline flow as a user**

At 390×844, with a cleared `sessionStorage`:

1. `#S-E2` → click "Not quite — end politely" → lands on `S-E8`.
2. Type 20 characters → submit stays dimmed.
3. Type a 50-character mush line → submit enables → click → **stays on `S-E8`** with a red note. This is the step most likely to be broken by a stray `href` navigation; confirm the hash did not change.
4. Replace with a real reason → click → lands on `#S-B2`.
5. Navigate to `#S-E9` → the reason you typed is there, verbatim.

- [ ] **Step 3: Verify Prev/Next and the ☰ index**

Evaluate:

```js
() => {
  const ids = [...document.querySelectorAll('#screenwrap .screen')].map(s => s.id);
  const e = ids.filter(i => /^S-E/.test(i));
  return { ok: e.indexOf('S-E6A') === e.indexOf('S-E6') + 1 &&
               e.indexOf('S-E8') > e.indexOf('S-E7') &&
               e.indexOf('S-E9') === e.indexOf('S-E8') + 1,
           order: e };
}
```
Expected: `ok: true`, order `[... S-E6, S-E6A, S-E7, S-E8, S-E9]` — the trail sits beside the profile it belongs to, and the decline branch sits at the end of the group like `S-F5`/`S-F6` do.

Then step Prev/Next through the E group in the UI and confirm no screen renders blank.

- [ ] **Step 4: Verify nothing in Phases 1–4 regressed**

Evaluate on any screen:

```js
() => {
  const need = ['MMQ','MMASYNC','MMLADDER','MMSCHED','MMDEV','MMATTACH','MMCONF',
                'MMDECLINE','MMDECLINEDATE','MMDECLINERECV','MMNUM','MMTRAIL',
                'pick','tog','togCap'];
  const missing = need.filter(n => typeof window[n] === 'undefined');
  const errors = [];
  try { MMQ.buildSet(); MMQ.confidence('wants-children'); MMSCHED.suggestions(3);
        MMQ.reviewQuestion('Do you want children of your own someday?'); }
  catch(e){ errors.push(String(e)); }
  return { ok: missing.length === 0 && errors.length === 0, missing, errors };
}
```
Expected: `ok: true`.

Then check the browser console for errors across `#S-D2`, `#S-F3`, `#S-F4`, `#S-C1` — the screens that consume `MMQ` most heavily and would surface a bad edit to it from Task 1.

- [ ] **Step 5: Fix anything the sweep found, then commit**

If the sweep is clean, there is nothing to commit and this step is a no-op — say so rather than inventing a commit. If it found defects, fix them and:

```bash
git add index.html
git commit -m "fix(reviews): <what the sweep actually found>"
```

- [ ] **Step 6: Open the PR**

```bash
git push -u origin feat/reviews-profile
gh pr create --title "Phase 5 — reviews & the personality profile" --body "$(cat <<'EOF'
Cluster 5 of the prompt_2 decision log (§5.1–5.4, items 14, 17, 18).

## What changed

- **The post-call decline is now a gate.** `S-E2`'s bare link to `#S-B2`
  goes to the new `S-E8`, which requires 40 characters and a substance
  and cruelty check before it lets you leave. No attempt counter, no
  escape hatch.
- **The reason reaches a person.** The new `S-E9` renders it verbatim to
  the declined party, set with `textContent` — the only user-typed string
  in the prototype shown to a different user.
- **`S-E6` shows its working.** A stated-vs-played card, driven by a new
  `MMQ.divergences()` over a `DEMO.stated` fixture whose values each
  trace to an `S-P1`–`S-P5` field. Four states, because a tie is not a
  divergence and an unplayed probe is not agreement.
- **`S-E6A`** lists the full evidence trail — every question, your
  answer, his verdict, what was inferred.
- **`S-E5` asks the compulsory number question**, blocking submission,
  with six MECE options behind a "no".

## Verification

Every touched screen screenshotted at 1280×800 and 390×844. The decline
flow walked end to end on mobile. Escaping asserted against markup in the
reason. The dev depth toggle verified to repaint both `S-E6`'s card and
`S-E6A`'s rows.

Spec: `docs/superpowers/specs/2026-09-08-reviews-profile-design.md`
EOF
)"
```

---

## Notes for the executor

**The one thing not to get wrong.** Task 3's `textContent`. Every other render in this file builds an HTML string and assigns `innerHTML`, and copying that pattern for the decline reason is the only change in this phase that would be a genuine defect rather than a cosmetic one. The assertion in Task 3 Step 1 checks it explicitly — do not weaken it.

**Two globals that look alike.** `MMDECLINE` is `S-F6`'s async decline and already exists. `MMDECLINEDATE` is `S-E8`'s date decline and is new. They are different flows with different rules (structured vs. free text) and neither may shadow the other.

**If `reviewReason` rejects something that feels honest**, that is likely correct — the 5-distinct-content-word threshold deliberately pushes a decliner past their first thin sentence, per §5.1's "the friction is the point". Do not loosen the threshold to make a test case pass; the accepted trade-off is recorded in the source comment and in the spec.
