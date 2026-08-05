# Candidate Gallery Scoring Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give `S-B2`/`S-B3` real portraits, a banded fit score with bulleted evidence, remove → backfill → £3 restore, and a gap-driven enrichment prompt.

**Architecture:** A `CANDIDATES` fixture array plus a small render engine live in an inline `<script>` inside `S-B2`, mirroring how The Show keeps its turn data inline in `S-D2`. Fit and confidence are *derived* from band scores at render time, never hardcoded, so editing a band automatically moves the headline number. State persists in `sessionStorage` under `mm_cand`. `S-B3` reads the same array to render the full breakdown.

**Tech Stack:** HTML + CSS + vanilla JavaScript, single file. Remote portraits from Unsplash. Verification is browser-driven via the Playwright MCP tools.

## Global Constraints

- **Single file.** All changes in `/mnt/e/TOOLMAKER/PYTHON/MakeaMoveUI_Mockup/index.html`. No new runtime files, no build step, no libraries.
- **Reuse existing components:** `.row`, `.card`, `.tag`, `.av`, `.note`, `.prog`, `.btn`, `.eyebrow`, `.muted`, and the `:root` custom properties. New CSS only for the toast, the confirm sheet, the drawer and the bullet list.
- **Theme both variants.** `S-B2`/`S-B3` are `paper`, but every new component must also be styled under `.dark`, per the standing convention.
- **`RESTORE_PRICE = 3`** (GBP), declared once. `UNDO_MS = 6000`.
- **Band weights are exactly** family 30, geo 20, stage 15, person 20, love 15 — summing to 100.
- **Score labels:** `≥70` Strong · `50–69` Moderate · `<50` Weak · `null` "— · Not enough data".
- **Confidence labels:** scored-weight `≥70` High · `40–69` Medium · `<40` Low.
- **The random-surfacing advisory is candidate-level**, shown once above the bands, only when confidence `< 40`. Never per-band.
- **Bullet source tags** are exactly `onboarding`, `profile`, `gameplay`, `no data`.
- **The enrichment prompt must never gate.** Copy stays invitational — users must never feel they have to complete a profile to be matched.
- **Do not break:** the `S-B2` tabbar, the `S-B3` → `S-C1` invite CTA, or the back-links into `S-B2` from `S-C*`/`S-E*`.
- **`S-B2` is already in the `FEED` set** in `classify()` (line ~1891). No desktop-reflow change is needed.

## Verification setup

`file://` is blocked by the Playwright MCP browser. Serve the project instead, once, before Task 1:

```bash
cd /mnt/e/TOOLMAKER/PYTHON/MakeaMoveUI_Mockup && python3 -m http.server 3300
```

All assertions run at **1280×800** against `http://127.0.0.1:3300/index.html#S-B2` (or `#S-B3`). Every task's "run the check" step means: `browser_navigate` to that URL, `browser_resize` to 1280×800, then `browser_evaluate` the given function. Because the router is hash-based, navigating between `#S-B2` and `#S-B3` does not reload the page — call `location.reload()` in the evaluate step when a task needs fresh `sessionStorage`.

## File Structure

One file, three insertion points:

- **CSS block** (`<style>`, lines ~11–460): append a new "candidate gallery" section after the existing `.tabbar` rules (~line 307). Adds `.fit`, `.why`, `.mm-toast`, `.mm-sheet`, `.drawer`.
- **`S-B2`** (lines 958–982): the three hardcoded `<a class="row">` blocks and the trailing "Enrich your profile" row are replaced by empty mount points that the engine fills.
- **`S-B3`** (lines 984–1005): the hero `.av` and the "Why you matched" card are replaced by mount points.
- **Inline `<script>`** at the end of `S-B2`: the fixture array, derivation helpers, and render/remove/restore logic. `S-B3` is rendered by the same script, because both screens live in the same document and the router only toggles visibility.

---

### Task 1: Data fixture and derivation helpers

**Files:**
- Modify: `index.html` — add an inline `<script>` immediately before the closing `</section>` of `S-B2` (after the `.tabbar` div, line ~981).

**Interfaces:**
- Produces, on `window`: `CANDIDATES` (array of 8), `BANDS` (array of 5 `{key,name,w}`), `RESTORE_PRICE`, `UNDO_MS`, and the helpers `fit(c)`, `conf(c)`, `scoreLabel(n)`, `confLabel(c)`, `bulletCount(c)`. Later tasks call these by exactly these names.

- [ ] **Step 1: Write the browser assertion**

This is the check the task must satisfy. Run it first, before writing any code, so you see it fail:

```js
() => {
  if (!window.CANDIDATES) return { ok:false, reason:'CANDIDATES not defined' };
  const byId = Object.fromEntries(CANDIDATES.map(c => [c.id, fit(c)]));
  const expected = { david:74, samuel:68, marcus:61, tunde:58,
                     emeka:55, kwame:52, ifeanyi:49, bode:48 };
  const confExpected = { david:'High', samuel:'Medium', marcus:'Medium',
                         tunde:'Low', emeka:'Low', kwame:'Low',
                         ifeanyi:'Low', bode:'Low' };
  const bad = Object.entries(expected).filter(([k,v]) => byId[k] !== v);
  const badConf = Object.entries(confExpected)
    .filter(([k,v]) => confLabel(CANDIDATES.find(c=>c.id===k)) !== v);
  return {
    ok: CANDIDATES.length === 8 && bad.length === 0 && badConf.length === 0 &&
        BANDS.reduce((s,b)=>s+b.w,0) === 100,
    weightSum: BANDS.reduce((s,b)=>s+b.w,0),
    computed: byId, badFit: bad, badConf
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Navigate to `http://127.0.0.1:3300/index.html#S-B2`, resize 1280×800, evaluate the above.
Expected: `{ ok:false, reason:'CANDIDATES not defined' }`.

- [ ] **Step 3: Add the fixture and helpers**

Insert this `<script>` immediately before `</section>` at the end of `S-B2`:

```html
<script>
/* ===== CANDIDATE GALLERY ENGINE (S-B2 / S-B3) =====
   Fixtures, not a real matching engine. Fit and confidence are DERIVED from
   band scores at render time so editing a band moves the headline number. */
(function(){
  const RESTORE_PRICE = 3;      // £ — cross-check vs Badoo before launch
  const UNDO_MS = 6000;
  /* Ask Unsplash to face-crop to the ASPECT OF THE SLOT. A square-ish source
     dropped into a tall 96x264 column gets ~45% of its width cropped away and
     slices the face off, so each usage requests its own dimensions. */
  const PHOTO = (pid, w, h) =>
    `https://images.unsplash.com/photo-${pid}?w=${w}&h=${h}&fit=crop&crop=faces&q=70&fm=jpg`;
  const cardPhoto = c => PHOTO(c.pid, 200, 540);   // tall gallery column
  const facePhoto = c => PHOTO(c.pid,  80,  80);   // small round-square avatar
  const heroPhoto = c => PHOTO(c.pid, 900, 420);   // wide S-B3 hero

  const BANDS = [
    { key:'family', name:'Family & intent',        w:30 },
    { key:'geo',    name:'Geography',              w:20 },
    { key:'stage',  name:'Life stage',             w:15 },
    { key:'person', name:'Personality & lifestyle',w:20 },
    { key:'love',   name:'Love & intimacy',        w:15 }
  ];

  /* bullet: s = 'y' match | 'n' mismatch | 'o' no data; src = source tag */
  const B = (s,t,src) => ({ s, t, src });

  /* Bench candidates all share a shape: only the two onboarding-derived bands
     score, which is exactly the cold-start case the prototype demonstrates. */
  function benchBands(geoScore, geoState, geoText, stageScore, stageText){
    return {
      family:{ score:null, blankOwner:'them', bullets:[
        B('o','He has not completed Marriage & family','no data'),
        B('o','Neither side has supplied enough to compare on children','no data')
      ]},
      geo:{ score:geoScore, bullets:[
        B(geoState, geoText,'onboarding'),
        B('o','Neither of you has given a travel radius','no data')
      ]},
      stage:{ score:stageScore, bullets:[ B('y', stageText,'onboarding') ]},
      person:{ score:null, blankOwner:'them', bullets:[
        B('o','He has not completed Personality & lifestyle','no data'),
        B('o','He has not played a show, so no gameplay signal exists','no data')
      ]},
      love:{ score:null, blankOwner:'them', bullets:[
        B('o','He has not completed Love & intimacy','no data')
      ]}
    };
  }

  const CANDIDATES = [
    { id:'david', name:'David', age:41, city:'London',
      blurb:'Deputy head · 2 kids · London',
      pid:'1531901599143-df5010ab9438',
      bands:{
        family:{ score:74, bullets:[
          B('y','You set "must welcome a blended family" as a dealbreaker · he answered "open to children from a previous marriage"','onboarding'),
          B('y','You have 2 children · he has 2 children','onboarding'),
          B('y','You want marriage within 2 years · he answered "within 2 years"','profile'),
          B('n','You are open to one more child · he answered "no more children"','profile')
        ]},
        geo:{ score:84, bullets:[
          B('y','You live in London · he lives in London — same city','onboarding'),
          B('o','Neither of you has given a travel radius','no data')
        ]},
        stage:{ score:70, bullets:[
          B('y','You are 40 · he is 41 — inside your stated 38–48 range','onboarding'),
          B('y','You are employed full time · he is employed full time','onboarding'),
          B('y','You both hold a postgraduate qualification','profile')
        ]},
        /* blankOwner 'you' — HE answered; the gap is on the user's side, so this
           is the one band that earns an actionable link on S-B3. */
        person:{ score:null, blankOwner:'you', bullets:[
          B('o','You have not completed Personality & lifestyle','no data'),
          B('o','He answered all 8 questions — the missing side is yours','profile')
        ]},
        love:{ score:65, bullets:[
          B('y','You both rate quality time as your first love language','profile'),
          B('n','You want daily contact · he answered "a few times a week"','profile')
        ]}
      }},

    { id:'samuel', name:'Samuel', age:44, city:'London',
      blurb:'Architect · wants kids · London',
      pid:'1596580817363-a4a8f67d4bc8',
      bands:{
        family:{ score:59, bullets:[
          B('y','He answered "welcomes a blended family" — your dealbreaker','onboarding'),
          B('n','You have 2 children and want no more · he has none and wants two','profile'),
          B('y','You want marriage within 2 years · he answered "within 3 years"','profile')
        ]},
        geo:{ score:84, bullets:[
          B('y','You live in London · he lives in London — same city','onboarding'),
          B('o','Neither of you has given a travel radius','no data')
        ]},
        stage:{ score:66, bullets:[
          B('y','You are 40 · he is 44 — inside your stated 38–48 range','onboarding'),
          B('y','You are employed full time · he is self-employed','onboarding')
        ]},
        person:{ score:null, blankOwner:'them', bullets:[
          B('o','He has not completed Personality & lifestyle','no data'),
          B('o','He has not played a show, so no gameplay signal exists','no data')
        ]},
        love:{ score:null, blankOwner:'them', bullets:[
          B('o','He has not completed Love & intimacy','no data')
        ]}
      }},

    { id:'marcus', name:'Marcus', age:39, city:'Reading',
      blurb:'GP · blended-family open · Reading',
      pid:'1605980776566-0486c3ac7617',
      bands:{
        family:{ score:66, bullets:[
          B('y','He answered "blended-family open" — your dealbreaker','onboarding'),
          B('y','You have 2 children · he has 1 child','onboarding'),
          B('o','He has not given a marriage timeline','no data')
        ]},
        geo:{ score:55, bullets:[
          B('n','You live in London · he lives in Reading — about 40 miles apart','onboarding'),
          B('o','Neither of you has given a travel radius','no data')
        ]},
        stage:{ score:60, bullets:[
          B('y','You are 40 · he is 39 — inside your stated 38–48 range','onboarding'),
          B('y','You are employed full time · he is employed full time','onboarding')
        ]},
        person:{ score:null, blankOwner:'them', bullets:[
          B('o','He has not completed Personality & lifestyle','no data'),
          B('o','He has not played a show, so no gameplay signal exists','no data')
        ]},
        love:{ score:null, blankOwner:'them', bullets:[
          B('o','He has not completed Love & intimacy','no data')
        ]}
      }},

    { id:'tunde', name:'Tunde', age:43, city:'Croydon',
      blurb:'Structural engineer · 1 kid · Croydon',
      pid:'1614023342667-6f060e9d1e04',
      bands:benchBands(62,'y','You live in London · he lives in Croydon — same metro area',
                       52,'You are 40 · he is 43 — inside your stated 38–48 range') },

    { id:'emeka', name:'Emeka', age:40, city:'Luton',
      blurb:'Pharmacist · no kids · Luton',
      pid:'1565884280295-98eb83e41c65',
      bands:benchBands(56,'n','You live in London · he lives in Luton — about 30 miles apart',
                       54,'You are 40 · he is 40 — inside your stated 38–48 range') },

    { id:'kwame', name:'Kwame', age:45, city:'Birmingham',
      blurb:'Secondary teacher · 3 kids · Birmingham',
      pid:'1584119164246-461d43e9bab3',
      bands:benchBands(44,'n','You live in London · he lives in Birmingham — about 120 miles apart',
                       62,'You are 40 · he is 45 — inside your stated 38–48 range') },

    { id:'ifeanyi', name:'Ifeanyi', age:38, city:'Manchester',
      blurb:'Data analyst · no kids · Manchester',
      pid:'1522529599102-193c0d76b5b6',
      bands:benchBands(40,'n','You live in London · he lives in Manchester — about 200 miles apart',
                       60,'You are 40 · he is 38 — inside your stated 38–48 range') },

    { id:'bode', name:'Bode', age:42, city:'Dublin',
      blurb:'Logistics manager · 2 kids · Dublin',
      pid:'1617244147299-5ef406921c35',
      bands:benchBands(38,'n','You live in the UK · he lives in Ireland — different country',
                       61,'You are 40 · he is 42 — inside your stated 38–48 range') }
  ];

  /* ---- derivations ---- */
  function conf(c){                       // scored weight, out of 100
    return BANDS.reduce((s,b) =>
      s + (c.bands[b.key] && c.bands[b.key].score != null ? b.w : 0), 0);
  }
  function fit(c){                        // weighted mean over scored bands
    let sw = 0, tot = 0;
    BANDS.forEach(b => {
      const bd = c.bands[b.key];
      if (bd && bd.score != null) { sw += b.w; tot += b.w * bd.score; }
    });
    return sw ? Math.round(tot / sw) : 0;
  }
  const scoreLabel = n => n >= 70 ? 'Strong' : n >= 50 ? 'Moderate' : 'Weak';
  const confLabel  = c => { const k = conf(c);
    return k >= 70 ? 'High' : k >= 40 ? 'Medium' : 'Low'; };
  const bulletCount = c => BANDS.reduce((n,b) =>
    n + ((c.bands[b.key] && c.bands[b.key].bullets) || []).length, 0);

  Object.assign(window, { CANDIDATES, BANDS, RESTORE_PRICE, UNDO_MS,
    fit, conf, scoreLabel, confLabel, bulletCount,
    PHOTO, cardPhoto, facePhoto, heroPhoto });
})();
</script>
```

- [ ] **Step 4: Run the assertion again**

Expected: `ok:true`, `weightSum:100`, and `computed` matching `{david:74, samuel:68, marcus:61, tunde:58, emeka:55, kwame:52, ifeanyi:49, bode:48}`.

If a number is off by one, adjust that candidate's band scores — do **not** hardcode the fit value. The derivation is the point.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(gallery): add candidate fixtures and fit derivation helpers"
```

---

### Task 2: CSS components

**Files:**
- Modify: `index.html` — CSS block, append after the `.tabbar` rules (~line 307).

**Interfaces:**
- Produces the classes `.fit`, `.fit .n`, `.why`, `.why li`, `.why li.y/.n/.o`, `.why .src`, `.mm-toast`, `.mm-toast.on`, `.mm-sheet`, `.mm-sheet.on`, `.mm-sheet .box`, `.drawer`, `.drawer.open`, `.gone`. Tasks 3–6 use exactly these names.

- [ ] **Step 1: Write the browser assertion**

`index.html` contains **three** `<style>` blocks — the main one at line 10 plus two screen-scoped blocks inside `S-D2` and `S-F3`. Scan all of them; indexing `document.styleSheets[length-1]` would read the `S-F3` block and fail no matter where the CSS is correctly placed.

```js
() => {
  const need = ['.fit','.why','.mm-toast','.mm-sheet','.drawer'];
  const css = [...document.styleSheets]
    .flatMap(s => { try { return [...s.cssRules]; } catch(e){ return []; } })
    .map(r => r.selectorText || '').join(' ');
  const missing = need.filter(s => !css.includes(s));
  // each must also carry a .dark variant
  const noDark = need.filter(s => !new RegExp('\\.dark\\s+\\' + s).test(css));
  const okNoDark = noDark.every(s => s === '.drawer' || s === '.mm-toast');
  return { ok: missing.length === 0 && okNoDark, missing, noDark };
}
```

Two of the five legitimately need no `.dark` rule, and the check names them explicitly rather than counting:

- **`.drawer`** is a layout-only wrapper — it toggles `display` and has nothing colour-bearing to theme.
- **`.mm-toast`** is already dark on both surfaces by design: it is built from `--stage-800` and `--ink`, so it reads as a dark pill over `paper` and over `dark` alike. That is the conventional treatment for a transient overlay.

`.fit`, `.why` and `.mm-sheet` each carry text or panel colour, so each must have its `.dark` rule.

- [ ] **Step 2: Run it to confirm it fails**

Expected: `{ ok:false, missing:['.fit','.why','.mm-toast','.mm-sheet','.drawer'], noDark:[…same five…] }`.

- [ ] **Step 3: Add the CSS**

```css
  /* ===== CANDIDATE GALLERY ===== */
  .fit{display:flex; align-items:center; gap:8px; margin-top:8px; font-size:12px; font-weight:700}
  .fit .n{font-family:var(--display); font-weight:900; font-size:15px; color:var(--t)}
  .dark .fit .n{color:var(--ink)}
  .fit .prog{flex:1; min-width:48px}

  .why{list-style:none; margin:8px 0 0; padding:0; display:flex; flex-direction:column; gap:6px}
  .why li{display:flex; gap:7px; font-size:12px; line-height:1.45; color:var(--t-dim)}
  .dark .why li{color:var(--ink-dim)}
  .why li::before{flex:0 0 auto; font-weight:800; width:11px}
  .why li.y::before{content:"\2713"; color:var(--pass)}
  .why li.n::before{content:"\2717"; color:var(--move)}
  .why li.o::before{content:"\25CB"; color:var(--t-faint)}
  .why .src{opacity:.6; font-size:10.5px; white-space:nowrap}

  .drawer{display:none} .drawer.open{display:flex; flex-direction:column; gap:10px}
  .gone{display:none !important}

  .mm-toast{position:absolute; left:18px; right:18px; bottom:74px; z-index:40;
    display:flex; align-items:center; gap:12px; padding:12px 14px; border-radius:12px;
    background:var(--stage-800); color:var(--ink); font-size:13px; font-weight:600;
    box-shadow:0 10px 30px rgba(0,0,0,.3); opacity:0; transform:translateY(8px);
    pointer-events:none; transition:opacity .25s, transform .25s}
  .mm-toast.on{opacity:1; transform:none; pointer-events:auto}
  .mm-toast button{margin-left:auto; background:none; border:none; cursor:pointer;
    font-family:var(--display); font-weight:800; letter-spacing:.1em; text-transform:uppercase;
    font-size:11px; color:var(--bulb)}

  .mm-sheet{position:absolute; inset:0; z-index:50; display:none; align-items:flex-end;
    background:rgba(8,12,26,.55)}
  .mm-sheet.on{display:flex}
  .mm-sheet .box{width:100%; background:var(--paper); border-radius:18px 18px 0 0;
    padding:20px 18px calc(20px + env(safe-area-inset-bottom));
    display:flex; flex-direction:column; gap:12px}
  .dark .mm-sheet .box{background:var(--stage-850)}
```

- [ ] **Step 4: Run the assertion again**

Expected: `{ ok:true, missing:[] }`.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(gallery): add fit meter, evidence list, toast, sheet and drawer styles"
```

---

### Task 3: Render the gallery — portraits, fit, evidence bullets

**Files:**
- Modify: `index.html` — `S-B2` markup (lines 958–982) and the inline script from Task 1.

**Interfaces:**
- Consumes: `CANDIDATES`, `BANDS`, `fit`, `conf`, `scoreLabel`, `confLabel`, `bulletCount` from Task 1.
- Produces: `state` (`{live:[ids], bench:[ids], removed:[ids]}`), `save()`, `load()`, `byId(id)`, `render()`, and the mount points `#b2list`, `#b2count`. Tasks 4–6 call `render()` after mutating `state`.

- [ ] **Step 1: Write the browser assertion**

```js
() => {
  location.hash = '#S-B2';
  const rows = [...document.querySelectorAll('#b2list .cand')];
  if (rows.length !== 3) return { ok:false, reason:'expected 3 cards', got:rows.length };
  const first = rows[0];
  const av = first.querySelector('.av');
  const bg = getComputedStyle(av).backgroundImage;
  const fitTxt = first.querySelector('.fit .n').textContent;
  const bullets = first.querySelectorAll('.why li').length;
  const link = first.querySelector('.seeall').textContent;
  const userAv = document.querySelector('#S-B2 .pbar .av');
  return {
    ok: bg.includes('images.unsplash.com') && fitTxt === '74%' &&
        bullets === 2 && /See all \d+ reasons/.test(link) && !!userAv &&
        getComputedStyle(userAv).backgroundImage.includes('unsplash'),
    bg: bg.slice(0,60), fitTxt, bullets, link, userAvatar: !!userAv
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Expected: `{ ok:false, reason:'expected 3 cards', got:0 }`.

- [ ] **Step 3: Replace the `S-B2` body markup**

Replace lines 964–979 (the three `<a class="row">` candidate blocks and the trailing "Enrich your profile" row) with:

```html
          <div id="b2list" style="display:flex;flex-direction:column;gap:12px"></div>
          <div id="b2drawerwrap"></div>
          <div id="b2gap"></div>
```

Change the `.note.pass` on line 961 so the count is dynamic — replace `<b>You have 3 pre-matches.</b>` with `<b id="b2count">You have 3 pre-matches.</b>`.

Add the user's portrait to the top bar. Replace line 959 with:

```html
        <div class="pbar"><span class="brand" style="font-size:18px">Make<span class="a">a</span>Move</span><a class="act" href="#S-G1">Credits ›</a><a class="av g2 s28" href="#S-A9" style="background-image:url('https://images.unsplash.com/photo-1531123897727-8f129e1688ce?w=80&amp;h=80&amp;fit=crop&amp;crop=faces&amp;q=70&amp;fm=jpg')" aria-label="Your profile"></a></div>
```

- [ ] **Step 4: Add the render logic**

Append inside the Task 1 IIFE, before the `Object.assign(window, …)` line:

```js
  /* ---- state ---- */
  const KEY = 'mm_cand';
  const DEFAULT = { live:['david','samuel','marcus'],
                    bench:['tunde','emeka','kwame','ifeanyi','bode'],
                    removed:[] };
  let state;
  const byId = id => CANDIDATES.find(c => c.id === id);
  function load(){
    try { state = JSON.parse(sessionStorage.getItem(KEY)) || {...DEFAULT}; }
    catch(e){ state = {...DEFAULT}; }
    if (!state.live || !state.bench || !state.removed) state = {...DEFAULT};
  }
  const save = () => sessionStorage.setItem(KEY, JSON.stringify(state));

  /* ---- rendering ---- */
  function bulletHTML(b){
    return `<li class="${b.s}"><span>${b.t} <span class="src">[${b.src}]</span></span></li>`;
  }
  /* The card carries only the two most decisive bullets: the highest-weight
     observed match, plus the sharpest observed mismatch if one exists. */
  function topBullets(c){
    const ordered = BANDS.slice().sort((a,b) => b.w - a.w);
    let match = null, miss = null;
    ordered.forEach(b => {
      const bd = c.bands[b.key];
      if (!bd) return;
      bd.bullets.forEach(x => {
        if (x.s === 'y' && !match) match = x;
        if (x.s === 'n' && !miss)  miss  = x;
      });
    });
    return [match, miss].filter(Boolean).slice(0,2);
  }
  /* Keep a gradient class on every portrait: if Unsplash is unreachable the
     gradient shows through, so the gallery degrades to its previous look
     rather than rendering empty boxes. */
  const GRAD = c => 'g' + (1 + (CANDIDATES.indexOf(c) % 4));

  function cardHTML(c){
    const f = fit(c), lab = scoreLabel(f), cl = confLabel(c);
    const lowTag = cl === 'Low'
      ? '<span class="tag warn" style="margin-left:6px">Low confidence</span>' : '';
    return `<div class="row cand" data-id="${c.id}" style="padding:0;overflow:hidden;align-items:stretch">
      <div class="av ${GRAD(c)}" style="width:96px;height:auto;min-height:120px;border-radius:0;background-image:url('${cardPhoto(c)}')"></div>
      <div class="grow" style="padding:12px">
        <div class="t1">${c.name}, ${c.age}${lowTag}</div>
        <div class="t2">${c.blurb}</div>
        <div class="fit"><span class="n">${f}%</span><span class="muted">${lab}</span>
          <span class="prog"><i style="width:${f}%"></i></span></div>
        <ul class="why">${topBullets(c).map(bulletHTML).join('')}</ul>
        <a class="link seeall" href="#S-B3" data-id="${c.id}"
           style="font-size:12px;margin-top:8px;display:inline-block">See all ${bulletCount(c)} reasons ›</a>
      </div>
      <button class="cand-x" data-id="${c.id}" aria-label="Remove ${c.name}"
        style="flex:0 0 auto;background:none;border:none;cursor:pointer;padding:10px 12px;align-self:flex-start;color:var(--t-faint);font-size:16px">✕</button>
    </div>`;
  }
  function render(){
    const list = document.getElementById('b2list');
    if (!list) return;
    list.innerHTML = state.live.map(id => cardHTML(byId(id))).join('');
    const n = state.live.length;
    const count = document.getElementById('b2count');
    if (count) count.textContent = `You have ${n} pre-match${n === 1 ? '' : 'es'}.`;
    list.querySelectorAll('.cand-x').forEach(btn =>
      btn.addEventListener('click', () => remove(btn.dataset.id)));
    list.querySelectorAll('.seeall').forEach(a =>
      a.addEventListener('click', () => sessionStorage.setItem('mm_cand_open', a.dataset.id)));
    renderDrawer();   // defined in Task 4
    renderGap();      // defined in Task 5
  }
  function remove(id){}          // replaced in Task 3 step 5 / Task 4
  function renderDrawer(){}      // replaced in Task 4
  function renderGap(){}         // replaced in Task 5

  load();
  document.addEventListener('DOMContentLoaded', render);
  if (document.readyState !== 'loading') render();
```

Add `render`, `state`, `byId`, `save`, `load` to the `Object.assign(window, …)` list.

- [ ] **Step 5: Run the assertion again**

Expected: `ok:true`, `fitTxt:'74%'`, `bullets:2`, `link:'See all 13 reasons ›'` (David has 4+2+3+2+2 bullets across his five bands), `userAvatar:true`.

Then take a screenshot at 1280×800 and confirm visually: three portraits load (not empty boxes), the meters read left-to-right, and the ✕ sits top-right of each card.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(gallery): render candidates with portraits, fit meters and evidence"
```

---

### Task 4: Remove, undo toast, backfill, drawer and £3 restore

**Files:**
- Modify: `index.html` — `S-B2` markup (add toast + sheet mount points) and the inline script.

**Interfaces:**
- Consumes: `state`, `save`, `render`, `byId`, `RESTORE_PRICE`, `UNDO_MS` from Tasks 1 and 3.
- Produces: working `remove(id)` and `renderDrawer()`, plus `restore(id)`, `confirmRestore(id)`, and the elements `#mmToast`, `#mmSheet`.

- [ ] **Step 1: Write the browser assertion**

```js
async () => {
  location.hash = '#S-B2'; location.reload();
  await new Promise(r => setTimeout(r, 600));
  const before = state.live.slice();
  document.querySelector('.cand-x[data-id="marcus"]').click();
  await new Promise(r => setTimeout(r, 100));
  const toastOn = document.getElementById('mmToast').classList.contains('on');
  const backfilled = state.live.includes('tunde') && !state.live.includes('marcus');
  const heldAtThree = state.live.length === 3;
  // wait out the undo window so Marcus lands in the drawer
  await new Promise(r => setTimeout(r, UNDO_MS + 400));
  const inDrawer = state.removed.includes('marcus');
  const drawerOpen = document.querySelector('.drawer').classList.contains('open');
  const priceTxt = document.querySelector('.drawer .restore').textContent;
  return {
    ok: toastOn && backfilled && heldAtThree && inDrawer && drawerOpen &&
        priceTxt.includes('£3'),
    before, after: state.live.slice(), toastOn, inDrawer, drawerOpen, priceTxt
  };
}
```

- [ ] **Step 2: Run it to confirm it fails**

Expected: a TypeError on `document.getElementById('mmToast')` being null, or `ok:false` with `backfilled:false` — `remove()` is still the empty stub from Task 3.

- [ ] **Step 3: Add the toast and sheet markup**

Insert immediately before the `.tabbar` div in `S-B2`:

```html
        <div class="mm-toast" id="mmToast"><span id="mmToastMsg"></span><button id="mmUndo">Undo</button></div>
        <div class="mm-sheet" id="mmSheet"><div class="box">
          <h3 class="scrn" style="font-size:19px;margin:0" id="mmSheetTitle"></h3>
          <p class="muted" style="font-size:13px;margin:0" id="mmSheetBody"></p>
          <button class="btn move block" id="mmSheetGo"></button>
          <button class="btn ghost block" id="mmSheetNo">Not now</button>
        </div></div>
```

- [ ] **Step 4: Replace the `remove` and `renderDrawer` stubs**

```js
  let undoTimer = null, pendingUndo = null;

  function backfill(){
    while (state.live.length < 3 && state.bench.length){
      state.live.push(state.bench.shift());     // bench is fit-descending
    }
  }
  function toast(msg, onUndo){
    const t  = document.getElementById('mmToast');
    const btn = document.getElementById('mmUndo');
    document.getElementById('mmToastMsg').textContent = msg;
    t.classList.add('on');
    btn.onclick = () => { clearTimeout(undoTimer); t.classList.remove('on'); onUndo(); };
    clearTimeout(undoTimer);
    undoTimer = setTimeout(() => { t.classList.remove('on'); commitRemoval(); }, UNDO_MS);
  }
  function remove(id){
    const i = state.live.indexOf(id);
    if (i === -1) return;
    state.live.splice(i, 1);
    pendingUndo = { id, index:i, backfilled:null };
    const benchTop = state.bench[0];
    backfill();
    if (state.live.includes(benchTop)) pendingUndo.backfilled = benchTop;
    save(); render();
    toast(`${byId(id).name} removed.`, () => {
      // free undo: put him back, send the backfill to the front of the bench
      if (pendingUndo.backfilled){
        state.live = state.live.filter(x => x !== pendingUndo.backfilled);
        state.bench.unshift(pendingUndo.backfilled);
      }
      state.live.splice(pendingUndo.index, 0, pendingUndo.id);
      pendingUndo = null; save(); render();
    });
  }
  function commitRemoval(){
    if (!pendingUndo) return;
    state.removed.push(pendingUndo.id);
    pendingUndo = null; save(); render();
  }
  function renderDrawer(){
    const wrap = document.getElementById('b2drawerwrap');
    if (!wrap) return;
    if (!state.removed.length){ wrap.innerHTML = ''; return; }
    wrap.innerHTML = `
      <div class="eyebrow" style="margin-top:6px">Removed (${state.removed.length})</div>
      <div class="drawer open">${state.removed.map(id => {
        const c = byId(id);
        return `<div class="row" data-id="${id}">
          <div class="av s40 ${GRAD(c)}" style="background-image:url('${cardPhoto(c)}')"></div>
          <div class="grow"><div class="t1">${c.name}, ${c.age}</div>
            <div class="t2">Fit ${fit(c)}% · removed by you</div></div>
          <button class="btn sm ghost restore" data-id="${id}">Restore · £${RESTORE_PRICE}</button>
        </div>`;
      }).join('')}</div>`;
    wrap.querySelectorAll('.restore').forEach(b =>
      b.addEventListener('click', () => confirmRestore(b.dataset.id)));
  }
  function confirmRestore(id){
    const c = byId(id), sheet = document.getElementById('mmSheet');
    document.getElementById('mmSheetTitle').textContent = `Restore ${c.name}?`;
    document.getElementById('mmSheetBody').textContent =
      `£${RESTORE_PRICE} one-off. This is not a session credit — your credits stay untouched. ` +
      `${c.name} returns to your list and your weakest current candidate goes back to the pool.`;
    const go = document.getElementById('mmSheetGo');
    go.textContent = `Pay £${RESTORE_PRICE} · restore ${c.name}`;
    go.onclick = () => { restore(id); sheet.classList.remove('on'); };
    document.getElementById('mmSheetNo').onclick = () => sheet.classList.remove('on');
    sheet.classList.add('on');
  }
  function restore(id){
    state.removed = state.removed.filter(x => x !== id);
    // hold the list at three: weakest live candidate returns to the bench front
    if (state.live.length >= 3){
      const weakest = state.live.slice().sort((a,b) => fit(byId(a)) - fit(byId(b)))[0];
      state.live = state.live.filter(x => x !== weakest);
      state.bench.unshift(weakest);
      state.bench.sort((a,b) => fit(byId(b)) - fit(byId(a)));
    }
    state.live.push(id);
    state.live.sort((a,b) => fit(byId(b)) - fit(byId(a)));
    save(); render();
  }
```

Delete the two stub lines `function remove(id){}` and `function renderDrawer(){}` — the real definitions above replace them. Add `remove`, `restore`, `confirmRestore` to the `Object.assign(window, …)` list.

- [ ] **Step 5: Run the assertion again**

Expected: `ok:true` — toast shown, Tunde backfilled, list held at 3, Marcus in the drawer after the undo window, restore button reading `Restore · £3`.

- [ ] **Step 6: Verify undo separately**

```js
async () => {
  location.hash = '#S-B2'; location.reload();
  await new Promise(r => setTimeout(r, 600));
  document.querySelector('.cand-x[data-id="samuel"]').click();
  await new Promise(r => setTimeout(r, 150));
  document.getElementById('mmUndo').click();
  await new Promise(r => setTimeout(r, 150));
  return { ok: state.live.join() === 'david,samuel,marcus' &&
               state.removed.length === 0, live: state.live, removed: state.removed };
}
```

Expected: `ok:true` — undo is free and restores original order, nothing charged, nothing in the drawer.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat(gallery): add remove with free undo, bench backfill and paid restore"
```

---

### Task 5: Profile-gap enrichment prompt

**Files:**
- Modify: `index.html` — the inline script (`renderGap` stub).

**Interfaces:**
- Consumes: `state`, `byId`, `BANDS` from Tasks 1 and 3.
- Produces: working `renderGap()`, the helper `blocking(gap)`, and the constant `USER_GAPS` (array of `{band, name, screen, qs}`).

The user's own blank sections suppress a band for *every* candidate at once. Those are the only actionable gaps. A band blank on the candidate's side gets no prompt, because nothing the user does can fill it.

- [ ] **Step 1: Write the browser assertion**

```js
() => {
  location.hash = '#S-B2';
  const box = document.querySelector('#b2gap .card');
  if (!box) return { ok:false, reason:'no gap prompt' };
  const txt = box.textContent.replace(/\s+/g,' ');
  const links = [...box.querySelectorAll('a')].map(a => a.getAttribute('href'));
  return {
    ok: /2 bands are blank on your side/.test(txt) &&
        links.includes('#S-P3') && links.includes('#S-P4') &&
        /Optional, always/.test(txt) &&
        !/must|required|complete your profile to/i.test(txt) &&
        /Personality & lifestyle — blocking 1 of your 3 candidates/.test(txt) &&
        /Love & intimacy — not blocking anyone yet/.test(txt),
    txt: txt.slice(0,320), links
  };
}
```

Two checks here are load-bearing:

- **The negative regex** enforces the non-gating constraint — the prompt must never imply completion is required.
- **The two per-gap counts** enforce honesty. With the current fixtures only David has a band blank on the *user's* side, so Personality blocks exactly 1 of 3 while Love & intimacy blocks nobody. Claiming both were "unscored for all 3 of your candidates" would be false, and the user would find that out by opening any profile.

- [ ] **Step 2: Run it to confirm it fails**

Expected: `{ ok:false, reason:'no gap prompt' }`.

- [ ] **Step 3: Replace the `renderGap` stub**

```js
  /* Sections the *user* has left blank. In the prototype these are fixtures;
     in production they would be read from the user's profile record. */
  const USER_GAPS = [
    { band:'person', name:'Personality & lifestyle', screen:'#S-P3', qs:8 },
    { band:'love',   name:'Love & intimacy',         screen:'#S-P4', qs:6 }
  ];

  /* How many live candidates is THIS gap actually blocking? A candidate only
     counts if he answered and the user did not — otherwise completing your
     side changes nothing for him, and claiming otherwise would be a lie. */
  function blocking(gap){
    return state.live.filter(id => {
      const bd = byId(id).bands[gap.band];
      return bd && bd.score == null && bd.blankOwner === 'you';
    }).length;
  }

  function renderGap(){
    const wrap = document.getElementById('b2gap');
    if (!wrap) return;
    const n = USER_GAPS.length, live = state.live.length;
    if (!n){
      wrap.innerHTML = `<div class="note pass">All 5 bands scored on your side.
        Nothing else is needed from you.</div>`;
      return;
    }
    const totalQs = USER_GAPS.reduce((s,g) => s + g.qs, 0);
    const mins = Math.max(1, Math.round(totalQs * 0.25));
    const [first, ...rest] = USER_GAPS;
    const lines = USER_GAPS.map(g => {
      const k = blocking(g);
      return `<li class="${k ? 'n' : 'o'}"><span><b>${g.name}</b> — ${
        k ? `blocking ${k} of your ${live} candidate${live===1?'':'s'} from a full score`
          : `not blocking anyone yet; none of your candidates has answered theirs either`
      } <span class="src">[no data]</span></span></li>`;
    }).join('');
    wrap.innerHTML = `<div class="card" style="margin-top:4px">
      <div class="eyebrow">✦ ${n} band${n===1?'':'s'} blank on your side</div>
      <ul class="why" style="margin-top:10px">${lines}</ul>
      <p class="muted" style="font-size:12.5px;line-height:1.5;margin:10px 0 0">
        Answering both means every future candidate is scored on all 5 bands
        instead of 3.</p>
      <div class="muted" style="font-size:11.5px;margin-top:6px">
        ~${totalQs} questions · about ${mins} minutes</div>
      <a class="btn primary block" href="${first.screen}" style="margin-top:12px">Answer ${first.name} ›</a>
      ${rest.map(g => `<a class="link center" href="${g.screen}" style="margin-top:8px;display:block">${g.name} ›</a>`).join('')}
      <div class="note" style="margin-top:12px">Optional, always. You're already
        matched — this only sharpens who we put in front of you next.</div>
    </div>`;
  }
```

Delete the `function renderGap(){}` stub. Add `USER_GAPS` to the `Object.assign(window, …)` list.

- [ ] **Step 4: Run the assertion again**

Expected: `ok:true`, links containing `#S-P3` and `#S-P4`.

- [ ] **Step 5: Verify the satisfied state**

```js
() => { USER_GAPS.length = 0; render();
        const t = document.querySelector('#b2gap').textContent;
        USER_GAPS.push({band:'person',name:'Personality & lifestyle',screen:'#S-P3',qs:8},
                       {band:'love',name:'Love & intimacy',screen:'#S-P4',qs:6}); render();
        return { ok:/All 5 bands scored on your side/.test(t), t:t.trim() }; }
```

Expected: `ok:true` — completion is acknowledged, not silently dropped.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(gallery): add gap-driven profile enrichment prompt"
```

---

### Task 6: `S-B3` full band breakdown

**Files:**
- Modify: `index.html` — `S-B3` markup (lines 984–1005) and the inline script.

**Interfaces:**
- Consumes: everything from Tasks 1, 3 and 5.
- Produces: `renderProfile()`, wired to `hashchange`, and the mount points `#b3hero`, `#b3name`, `#b3body`.

- [ ] **Step 1: Write the browser assertion**

```js
async () => {
  location.hash = '#S-B2'; location.reload();
  await new Promise(r => setTimeout(r, 600));
  const open = async id => {
    sessionStorage.setItem('mm_cand_open', id);
    location.hash = '#S-B2'; location.hash = '#S-B3';
    await new Promise(r => setTimeout(r, 250));
    const bands = [...document.querySelectorAll('#b3body .band')];
    return {
      count: bands.length,
      names: bands.map(b => b.querySelector('.bname').textContent),
      blank: bands.filter(b => /Not enough data/.test(b.textContent)).length,
      advisory: !!document.querySelector('#b3body .advisory'),
      meter: document.querySelector('#b3body .completeness').textContent.replace(/\s+/g,' '),
      gapLink: [...document.querySelectorAll('#b3body a')]
        .some(a => a.getAttribute('href') === '#S-P3'),
      hero: document.querySelector('#b3hero').style.backgroundImage.includes('unsplash')
    };
  };
  const d = await open('david'), m = await open('marcus');
  return {
    ok: d.count === 5 && d.names[0] === 'Family & intent' && d.hero &&
        d.blank === 1 && !d.advisory && /Scored on 4 of 5 bands/.test(d.meter) &&
        d.gapLink === true &&
        m.blank === 2 && !m.advisory && /Scored on 3 of 5 bands/.test(m.meter) &&
        m.gapLink === false,
    david: d, marcus: m
  };
}
```

Two expectations here carry the design's sharpest distinctions, so read them before implementing:

- **`advisory:false` for both.** David is High confidence (80) and Marcus Medium (65). The random-surfacing line is candidate-level and fires only below 40, so neither shows it despite both having blank bands.
- **`david.gapLink === true` but `marcus.gapLink === false`.** David's Personality band is blank because *the user* hasn't answered — he has. Marcus's is blank because *he* hasn't. Only the first is actionable, so only the first gets a link. Linking on Marcus would promise the user an improvement they cannot cause.

- [ ] **Step 2: Run it to confirm it fails**

Expected: a TypeError on reading `.completeness` of null — `#b3body` does not exist yet.

- [ ] **Step 3: Replace the `S-B3` body**

Replace lines 985–999 (`.pbar` through the closing `</div>` of the `.note`) with:

```html
        <div class="pbar"><a class="back" href="#S-B2">‹</a><h1 id="b3name">Candidate</h1></div>
        <div class="body">
          <div class="av g1" id="b3hero" style="width:100%;height:200px;border-radius:16px"></div>
          <div id="b3body" style="display:flex;flex-direction:column;gap:14px"></div>
          <div class="note">Full bio, extra photos and his answers stay hidden until he accepts. You're choosing who to invite — not judging a wall of text.</div>
        </div>
```

Leave the `.foot` block (lines 1001–1004) untouched — the invite CTA into `S-C1` must keep working.

- [ ] **Step 4: Add `renderProfile`**

```js
  function renderProfile(){
    const id = sessionStorage.getItem('mm_cand_open') || state.live[0];
    const c = byId(id);
    if (!c || !document.getElementById('b3body')) return;
    document.getElementById('b3name').textContent = `${c.name}, ${c.age}`;
    document.getElementById('b3hero').style.backgroundImage = `url('${heroPhoto(c)}')`;

    const scored = BANDS.filter(b => c.bands[b.key] && c.bands[b.key].score != null).length;
    const k = conf(c), f = fit(c);
    const advisory = k < 40 ? `<div class="note danger advisory">! Surfaced partly at
      random. Only ${scored} of 5 bands could be scored, so he is not meaningfully
      ranked against your other candidates.</div>` : '';

    /* A link only helps when HE has already answered — completing your side
       then actually scores the band. If he is the missing side, the link would
       promise an improvement the user cannot cause. */
    const gapFor = (key, bd) =>
      bd.blankOwner === 'you' ? USER_GAPS.find(g => g.band === key) : null;

    document.getElementById('b3body').innerHTML = `
      <div class="card">
        <div class="eyebrow">Overall fit</div>
        <div class="fit" style="margin-top:6px"><span class="n" style="font-size:26px">${f}%</span>
          <span class="muted">${scoreLabel(f)} · ${confLabel(c)} confidence</span></div>
        <div class="prog" style="margin-top:8px"><i style="width:${f}%"></i></div>
        <div class="muted completeness" style="font-size:12px;margin-top:8px">Scored on
          ${scored} of 5 bands · he's completed ${Math.round(k)}% of his profile</div>
      </div>
      ${advisory}
      <div class="note pass">✓ Dealbreakers cleared — genotype AA, marital status
        eligible, and welcomes a blended family.</div>
      ${BANDS.map(b => {
        const bd = c.bands[b.key] || { score:null, bullets:[] };
        const blank = bd.score == null;
        const gap = blank && gapFor(b.key, bd);
        return `<div class="card band" ${blank ? 'style="opacity:.72"' : ''}>
          <div style="display:flex;align-items:baseline;gap:8px">
            <b class="bname" style="font-size:13.5px">${b.name}</b>
            <span class="muted" style="font-size:11px;margin-left:auto">${
              blank ? '— · Not enough data'
                    : `${bd.score}% · ${scoreLabel(bd.score)}`}</span>
          </div>
          ${blank ? '' : `<div class="prog" style="margin-top:8px"><i style="width:${bd.score}%"></i></div>`}
          <ul class="why">${bd.bullets.map(bulletHTML).join('')}</ul>
          ${gap ? `<a class="link" href="${gap.screen}" style="font-size:12px;margin-top:8px;display:inline-block">Blank because you haven't answered ${gap.name}. Answer it ›</a>` : ''}
        </div>`;
      }).join('')}`;
  }
  window.addEventListener('hashchange', () => {
    if (location.hash === '#S-B3') renderProfile();
  });
```

Add `renderProfile` to the `Object.assign(window, …)` list, and call `renderProfile()` at the end of `render()`.

- [ ] **Step 5: Run the assertion again**

Expected: `ok:true` — five bands, Family & intent first, two blank, no advisory, `Scored on 3 of 5 bands`, and a `#S-P3` gap link present.

- [ ] **Step 6: Verify the advisory fires for a Low-confidence candidate**

```js
async () => {
  sessionStorage.setItem('mm_cand_open','bode');
  location.hash = '#S-B2'; location.hash = '#S-B3';
  await new Promise(r => setTimeout(r, 300));
  const adv = document.querySelector('#b3body .advisory');
  const pos = adv && adv.compareDocumentPosition(document.querySelector('#b3body .band'));
  return { ok: !!adv && (pos & Node.DOCUMENT_POSITION_FOLLOWING) !== 0,
           text: adv && adv.textContent.replace(/\s+/g,' ').trim() };
}
```

Expected: `ok:true`, and the advisory text mentions "Only 2 of 5 bands". `DOCUMENT_POSITION_FOLLOWING` confirms it sits *above* the bands, once, as specified.

- [ ] **Step 7: Screenshot both screens**

At 1280×800, capture `#S-B2` and `#S-B3`. Confirm: portraits load, no empty gradient boxes, bullet glyphs (✓ ✗ ○) render in the right colours, and the desktop feed reflow still applies to `S-B2`.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "feat(gallery): add full band breakdown with evidence on candidate profile"
```

---

## Portrait reference

Verified 200-OK on 2026-08-04. Each usage requests its own dimensions via `PHOTO(pid, w, h)` so Unsplash face-crops to that slot's aspect: `cardPhoto` 200x540, `facePhoto` 80x80, `heroPhoto` 900x420.

| Candidate | Unsplash photo ID |
|---|---|
| David, 41 | `photo-1531901599143-df5010ab9438` |
| Samuel, 44 | `photo-1596580817363-a4a8f67d4bc8` |
| Marcus, 39 | `photo-1605980776566-0486c3ac7617` |
| Tunde, 43 | `photo-1614023342667-6f060e9d1e04` |
| Emeka, 40 | `photo-1565884280295-98eb83e41c65` |
| Kwame, 45 | `photo-1584119164246-461d43e9bab3` |
| Ifeanyi, 38 | `photo-1522529599102-193c0d76b5b6` |
| Bode, 42 | `photo-1617244147299-5ef406921c35` |
| User (self) | `photo-1531123897727-8f129e1688ce` |

Each was chosen by eye from an Unsplash contact sheet to match the prototype's
target demographic. The user portrait reads slightly younger than the persona
(a 40-year-old London project manager, mother of two); swap the ID if that
matters for a stakeholder demo.

If Unsplash is unreachable the `.av` gradient shows through, because the
gradient class is retained on every portrait element — the gallery degrades to
the current appearance rather than rendering empty boxes.
