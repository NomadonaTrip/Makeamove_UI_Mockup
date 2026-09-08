# Karma points — a consequence you can see the working of

**Date:** 2026-09-08
**Status:** Approved
**Phase:** 6 of the `prompt_2.md` cluster map
**Source decisions:** `2026-08-24-prompt2-decisions.md` §6.1–6.4, §6.6 (Cluster 6, items 10, 11). Badges deferred per §6.5.
**Target file:** `index.html` (single self-contained page)
**Screens touched:** `S-B2`, `S-C3`, `S-F2A`, `S-A9`, plus one new screen `S-B4`

## Summary

Karma does not exist in this prototype. Two mentions in 336KB: a CSS comment
reserving a dev-panel row for it (`index.html:588`), and `S-E9`'s promise that
"your karma is untouched" (`:3972`) — a reassurance about a quantity the
product has never once shown.

That absence is the whole problem §6.2 describes. The prototype asks Tolu to
answer promptly, show up, finish her profile and write honest reviews, and
gives her nothing back that says any of it registered. Meanwhile `S-B2` orders
its candidates by a stored array, so the one consequence §6.3 specifies —
reliable people surface more — is not in the product either.

This phase makes standing a real quantity derived from §6.2's four behaviours,
lets it order the gallery, shows Tolu the receipts behind her own, and puts the
§6.4 A/B display question behind the toggle that decides it.

## Decisions (locked)

Resolved in the brainstorming session of 2026-09-08. Each records its rejected
alternatives so a later reader can tell what was considered.

- **One data shape, two renderings.** Each person carries four sub-scores, one
  per §6.2 behaviour. Direction A renders the ones clearing a gate as trait
  phrases; Direction B renders their weighted mean as a number. Rejected:
  Direction A's traits as their own fixture copy — the A/B test would then be
  comparing a real number against invented sentences and could not settle
  anything.
- **`MMKARMA` holds both sides — candidates' standing and Tolu's.** Rejected:
  candidate karma inside the `CANDIDATES` fixtures, which `S-C3` and `S-F2A`
  cannot reach from outside the gallery closure, and which would leave the
  toggle repainting two sources; and no global at all, with `S-B4` running its
  own section script — the exact drift Phase 5 avoided by giving
  `MMCONF.paint()` both derived blocks on `S-E6`.
- **The consequence is demonstrated by sorting her gallery, and stated once in
  words on her own screen.** Rejected: sorting only, which never tells Tolu the
  same rule applies to her and so drops the half that carries §6.2's incentive;
  and a "where you show" block giving her rank in a candidate's gallery, which
  invents a surface the log never asked for and is the closest this phase could
  come to §6.3's "never announcing that they have been penalised".
- **A seeded month of history plus live events.** Rejected: a fixture-only
  ledger, which makes the screen a picture of karma rather than karma; and live
  events only, which lands a deep link or a fresh session on an empty screen —
  ruled against twice already in this file, by `S-F4`'s `paint()` and `S-E9`'s
  fallback.
- **§6.6 gets a full card on `S-B2`, and the gap card yields to it.** Rejected:
  the invitation living on `S-B4` with a pointer from `S-B2`, which puts the
  most motivating message in the product one tap off the busiest screen; and
  merging it into the gap card, which rewrites a Phase-2 component's logic and
  leaves §6.6 absent from the karma surface it is meant to be consistent with.

## Scope note — this is a frontend mockup

No backend. Sub-scores are fixtures, the seeded ledger is written, and the
"weighted mean" is four multiplications. Nothing here is a scoring model; the
point is to make the mechanic visible so §6.4's A/B question can actually be
answered by looking at it.

Two boundaries are held deliberately rather than by omission:

- **No conduct events at all this phase.** §6.2 requires the clean-conduct half
  of behaviour four be scored *only after an operator ruling*, so that a
  reported user does not lose points before anyone has judged the report.
  Rulings are Phase 7's `S-H2`. The behaviour exists in the data shape and
  earns nothing until Phase 7 emits into it.
- **No live penalties.** The prototype offers no way to ghost a partner or
  abandon a board — you cannot click *not* doing something. Every penalty in
  the ledger is seeded. A live penalty path would require inventing an
  interaction the product does not have.

---

## Screen inventory

| ID | Class | `data-title` | Purpose |
|---|---|---|---|
| `S-B4` | paper | Your karma | Her standing, the receipts behind it, and what would raise it |

### Why `S-B4`

Group B is where §6.3's consequence lives, and `S-B2` is the screen the
invitation is read from. `S-B4` is a leaf off the gallery rather than a branch
out of a flow, so Prev/Next stepping reaches it after `S-B3` without wedging a
dead end into the middle of an arc — the same reasoning that put `S-E6A` beside
`S-E6` while the decline branch went to `S-E8`/`S-E9`.

### Desktop archetype

`S-B4` is added to the **`FEED`** set in `classify()` (`index.html:4675`). It is
a long row list; the default centred form sheet would be wrong.

---

## Data — `MMKARMA`

A new global, defined immediately after `MMSCHED` (`index.html:1720`) and
before `<div class="screenwrap">` at `:1724`. It must be in that first script
block, not the router block at the foot: the gallery's section script runs at
`:2348`, and a global defined at `:4889` would not exist yet.

```js
window.MMKARMA = {
  BEHAVIOURS, TRAIT_GATE, MAX_TAGS,
  of, score, band, dots, traits,          // derivations over a person record
  dir, setDir,                            // §6.4 — 'A' | 'B'
  tagHTML,                                // the card slot, direction-aware
  ledger, emit,                           // §6.2's receipts
  pursued,                                // §6.6's three people
  paint                                   // repaints the hosts it owns
};
```

### `BEHAVIOURS` — §6.2's four, weighted

Weights sum to 100, mirroring the existing `BANDS` table in the gallery
(`:2364`) rather than inventing a second weighting idiom.

| Key | Name | `w` | Direction A trait |
|---|---|---|---|
| `responds` | Responsiveness | 30 | "Responds within a day" |
| `shows` | Showing up | 30 | "Always shows up" |
| `complete` | Profile & questions | 20 | "Profile fully answered" |
| `feedback` | Honest feedback | 20 | "Leaves honest reviews" |

`score(id)` is the weighted mean of the four sub-scores, 0–100. `band(id)` is
`Reliable` ≥ 75, `Steady` ≥ 50, `Patchy` below. `dots(id)` is
`Math.round(score/20)`, out of five.

`traits(id)` returns the traits whose sub-score clears `TRAIT_GATE` (75),
highest first, capped at `MAX_TAGS` (2). **It never returns a negative trait.**
A low-standing candidate simply carries fewer tags, or none. This is §6.3's
"without ever announcing to anyone that they have been penalised", enforced in
the derivation rather than trusted to the copy.

### The people

All nine gallery candidates carry a record, not only the three currently live —
the drawer can restore a bench candidate into `live` at any time, and an
unranked candidate would sort to the bottom for the wrong reason. `of()` falls
back to a neutral 50-across record for an id it does not know, so a candidate
added by a later phase cannot break the sort.

| Person | `responds` | `shows` | `complete` | `feedback` | score | band |
|---|---|---|---|---|---|---|
| `you` (Tolu) | 82 | 90 | 55 | 78 | **78** | Reliable |
| `samuel` | 88 | 84 | 62 | 80 | **80** | Reliable |
| `marcus` | 70 | 76 | 58 | 68 | **69** | Steady |
| `david` | 25 | 35 | 50 | 25 | **33** | Patchy |

Bench candidates take mid-range records; their exact values only matter for
ordering after a restore.

Four properties of this table are requirements, not coincidence:

- **David scores highest on fit and lowest on karma.** His fit of 76 is the
  highest in the live set — it is a derivation over his band scores
  (`:2399`), not a number this phase may choose — so his karma has to be low
  enough for the sort to visibly disagree with the fit column. Below ~47 is
  where that happens; see the sort table. A fixture set where karma and fit
  agreed would make the reorder unobservable and the feature unprovable.
- **The three live candidates land in three different bands** — Samuel
  Reliable, Marcus Steady, David Patchy — so `band()` is exercised across its
  whole range on one screen.
- **Marcus clears the trait gate on exactly one behaviour** (`shows` 76, the
  rest below 75), so Direction A renders one tag for him, two for Samuel and
  none for David. Each branch of `traits()` is visible in a single screenshot.
- **Tolu's `complete` is her weakest at 55**, which is what makes §6.6's ask and
  the gap card honest — she is being invited to fix the thing that is actually
  low, not nagged at random.

`score()` rounds, so Tolu's 78.2 displays as 78; the other three are exact.

**On David being Patchy.** He is the man Tolu invites, plays a show with and
takes to a date, and this makes him a strong fit who treats people poorly. That
is not an awkward fixture — it is the tension karma exists to express, and it is
why §6.3 chose reduced visibility over exclusion: he still reaches her gallery,
just not at the top of it.

### The sort — §6.3

```
rank = fit × (0.7 + 0.3 × karma/100)
```

Karma moves a candidate by up to ~30% and never swamps fit. `fit` is the
existing derivation in the gallery (`fit()`, `:2536`) — a weighted mean over
the scored bands — and is not a number this phase writes. On the fixtures:

| Candidate | fit | karma | rank | position |
|---|---|---|---|---|
| Samuel | 68 | 80 | 63.9 | 1st |
| David | 76 | 33 | 60.7 | 2nd |
| Marcus | 61 | 69 | 55.3 | 3rd |

Samuel above David is the demonstration. Marcus stays third in both models, so
the change is a single visible swap — cheap to assert and unambiguous when it
regresses.

**The coefficient range is what sets David's ceiling.** `0.7 + 0.3 × karma/100`
spans 0.7 to 1.0, a maximum swing of 1.43×, and David leads Samuel on fit by 8
points. Overtaking therefore needs David below roughly karma 47 — which is why
his record sits at 33 rather than somewhere merely mediocre. **Anyone retuning
either the formula or David's sub-scores must re-derive this**, or the sort
silently stops demonstrating anything while still looking correct.

**The order does not change with the §6.4 toggle.** Ordering reads karma; the
toggle only decides how karma is drawn. A build where flipping the toggle
reordered the list would be wrong.

The apparent inversion needs no explanatory copy beyond one `.muted` line under
the heading — *"Ordered by fit and by how each of them treats people"* — because
the card slot is itself the explanation: under Direction A Samuel carries two
trait tags and David carries none; under Direction B their numbers differ.

### The ledger

Seeded rows plus live events, newest first, read through `ledger()`. Live
events are appended to `mm_karma` in `sessionStorage`; each carries an `id` so
a revisited screen cannot append it twice, and each raises its behaviour's
sub-score so the standing genuinely moves rather than the rows piling up under
a frozen number.

**The sub-score table above is the state *after* the seeded history, not
before it.** Seeded rows are displayed but never applied — they are already
baked into the fixtures. Only live events add. Applying both would double-count
the seeded month and drift Tolu's 78 away from the number the table specifies,
which is the sort of error that renders correctly and is invisible until
someone checks the arithmetic.

**Seeded rows name nobody.** "Answered an invite in 3 hours", not "answered
Kunle's invite" — a named past partner would be a fourth person the prototype
never shows, and the first reader to ask who that is finds the fixture world
has no answer. The cast is Tolu and David plus the gallery; the ledger stays
inside it.

| Seeded row | Behaviour | Δ |
|---|---|---|
| Left a substantive review after your last call | `feedback` | +5 |
| **Left an async board unanswered for 72 hours** | `responds` | **−8** |
| Showed up · Friday's show | `shows` | +6 |
| Completed Marriage & family | `complete` | +6 |
| Answered an invite in 3 hours | `responds` | +4 |

One row per behaviour plus the penalty. **Tolu sees her own penalty**; what
§6.3 forbids is showing it to anyone else, and `traits()` already enforces
that.

### Live emit points

Seven, every one of them unambiguously an action *Tolu* takes on her own phone:

| Where | Anchor | Behaviour | Δ |
|---|---|---|---|
| `S-F3` finishing the async board | `ffinish()`, `:4294` | `responds` | +4 |
| `S-P3` Save · back to profile | `:2128` | `complete` | +6 |
| `S-P4` Save · back to profile | `:2145` | `complete` | +6 |
| `S-QD1` Submit for review | `MMDECK.submit()`, `:2183` | `complete` | +5 |
| `S-E6` correcting a wrong inference | `MMCONF.reject()`, `:3761` | `complete` | +3 |
| `S-E5` submitting the review | `MMNUM.submit()`, `:3657` | `feedback` | +5 |
| `S-E8` decline clearing its gate | `MMDECLINEDATE.submit()`, `:3957` | `feedback` | +5 |

`MMCONF.reject()` is named by §6.2 outright ("correcting wrong inferences on
`S-E6`") and is already wired, so it costs one line.

The deck emit fires **only when `MMQ.deck().length > 0`**. `MMDECK.submit()`
also runs on an empty deck, where it just clears the attempt budget, and
crediting completeness for submitting nothing would be a lie the ledger then
displays.

**`S-C3` and `S-C4` are deliberately not emit points.** `S-C3` carries a
"David's view" tag in its `.pbar` (`:2919`) — the promptness earned by
responding there is David's, not Tolu's, and `S-C4` is reached from it. This is
the trap in wiring responsiveness: the screens that most look like a prompt
reply are the ones on the other party's phone.

### `pursued()` — §6.6's three

A list of three entries, each naming what that person asked about and which
profile section would answer it. The count is derived from the list's length,
never written as a literal, because §6.6's first hard constraint is that the
number be real — a hardcoded `3` beside a list of two is exactly the dark
pattern the decision forbids.

---

## Behaviour

### §6.4 — the toggle

A third `.devrow` in the dev panel (`:4991`), beside the question-count model
and profile depth. `MMDEV` already carries the comment reserving it — both
`:588` and `:4887` name spec 6.4 — so this is filling a slot the file was built
with, not adding a surface.

`MMDEV.setKarmaDir(d)` persists to `mm_karma_dir` and repaints: `MMGAL.render()`
for the cards and the §6.6 card, `MMKARMA.paint()` for the `S-B4` header and the
`S-C3` / `S-F2A` tag slots. **Direction A is the default** — it fits the
existing `.tag` styling, and the log lists it first.

The card slot is one place: a `<span>` next to the name where `Low confidence`
already sits (`lowTag`, `:2592`), filled by `tagHTML(id)`. Both directions
occupy that slot, so the toggle swaps content at exactly one point in the
markup rather than showing and hiding two parallel treatments.

- **Direction A** — up to two `.tag` chips carrying trait phrases.
- **Direction B** — one `.tag` carrying the number.

**A recorded observation for whoever reads the test.** Direction B necessarily
leaks low standing — David's card reads `33` and there is no way to render a
number that does not. Direction A cannot leak it, because `traits()` returns
only what clears the gate. That asymmetry is not a bug in either direction; it
is a real difference between them and the test should weigh it, since §6.3
wants the consequence felt without the penalty being announced.

### §6.4 on the other party's screens

`S-C3` (`:2924`) and `S-F2A` (`:4055`) both render Tolu's card to someone else,
with identical markup and the same tag row. Both get an empty
`<span id="c3karma">` / `<span id="f2akarma">` in that row, filled by
`MMKARMA.paint()`.

`S-F2A` is beyond the cluster map's listed screens, and is included
deliberately: it is the same card in the same situation — someone deciding
whether to commit to Tolu, on `S-F2A` while being asked to pay 50% — and
showing her standing on one but not the other would be a visible inconsistency
for a one-line saving. This is the same consistency argument Phase 5 used
between `S-E8` and `S-F6`.

This is also where the mirror lands: Tolu's karma appearing on someone else's
phone is what §6.3 is actually about, and it is the reason the log lists `S-C3`
at all.

### `S-B4` — her standing and the receipts

`paper`, `data-group="B · Matching"`, `data-title="Your karma"`. Four blocks:

1. **The header**, direction-aware. Under A: the five dots, the band label, and
   her trait phrases. Under B: the number and the band label. The dots are the
   qualitative rendering, so the header still says something under A rather
   than going blank.
2. **The §6.3 mirror**, one `.note` line: the same ordering works on her side,
   and her standing puts her in front of more people. Stated once, in words, and
   not elaborated into a rank.
3. **The ledger.** Rows reuse `.confrow` and its `.cf-*` children (`:616`),
   already themed for both surfaces: `.cf-name` the behaviour, `.cf-label` the
   delta, `.cf-val` the date and context. Two rules are added in that same CSS
   block — `.confrow[data-state="gain"] .cf-label` and `[data-state="loss"]` —
   using `var(--pass)` and `var(--move)`, the tokens the neighbouring
   `[data-confirmed]` and `[data-state="diverges"]` rules already use. **No new
   colours.**
4. **What would raise it**, linking into the specific gap screens — `#S-P3` and
   `#S-P4`, the two sections `USER_GAPS` (`:2626`) records as blank.

Reached from a `.row` on `S-A9` (`:2013`), which is the profile tab's
destination and already a list of exactly these rows, and from a secondary link
in the §6.6 card's foot.

### §6.6 — "3 people pursued you this month"

A card in a new `#b2pursued` host directly above `#b2list` (`:2336`), rendered
by `renderPursued()` inside the gallery closure so it can read `USER_GAPS` for
the specific questions. Called from `render()` (`:2609`) beside
`renderDrawer` / `renderGap` / `renderProfile`.

The card opens with the derived count, says what those people asked about, and
its CTA goes into the first blank section. Its foot carries the secondary
`See your standing ›` to `#S-B4`.

**The render is guarded on `pursued()` being non-empty** and emits nothing at
all when it is. §6.6's "if nobody tried to pursue them, the message does not
appear" is a hard constraint, and a card that degrades to "0 people pursued
you" would be worse than the gate the rollback removed.

**No "unlock", no "eligible", no progress bar.** §6.6's third constraint, added
by the 2026-08-26 rollback: the invitation offers a better product, never
restored access. Nothing in this card may imply a threshold exists.

**The gap card yields.** `renderGap()` (`:2643`) keeps its per-band blocking
evidence and its "~9 questions · about 2 minutes" line — that is the honest
detail behind the invitation — but its `.btn primary block` drops to a `.link`,
so the screen carries one primary ask instead of two buttons pointing at the
same screen.

---

## Verification

Playwright MCP, per screen, at **both 1280×800 and 390×844**. Prior phases in
this repo passed diff review with screens visibly broken; the viewport sweep is
not optional and not a spot check.

Two measurement traps carried forward from earlier phases:

- **Overflow on `S-B4`** is measured from child rects against the container
  box, not `scrollHeight`. A Phase 3 review under-reported a spill by ~115px
  that way. Let entry animations settle first.
- **Assertions query rendered elements, never a screen's `textContent`.** The
  inline `<script>` fixtures sit inside the `<section>`, so `textContent`
  carries every fixture string and any substring assertion false-positives.

Specific to this phase, assert that:

- `S-B2` renders Samuel above David, and that David's fit reads higher than
  Samuel's on the same screen — the inversion is the feature.
- Flipping the §6.4 toggle changes the card slot and the `S-B4` header, and
  **leaves the list order identical**.
- Under Direction A no candidate renders a tag for a sub-score below 75 —
  specifically, David's card carries no karma tag at all.
- Tolu's karma tag appears on `S-C3` and `S-F2A`, and both follow the toggle.
- The §6.6 card does not render when `pursued()` is empty, and its count
  matches that list's length rather than the string "3".
- One emit appends exactly one ledger row, and revisiting the emitting screen
  does not append a second.
- `MMDECK.submit()` on an empty deck emits nothing.
- `S-B2` still removes, undoes and restores candidates, and the restored
  candidate sorts into a sensible position rather than the bottom.

## Constraints carried from `CLAUDE.md`

- Single self-contained file. No external CSS/JS, no libraries, no build step.
- Reuse the `:root` custom properties and the shared `.btn` / `.card` / `.row`
  / `.chip` / `.opt` / `.note` / `.tag` / `.confrow` components. Invent no new
  colours — the two new `.confrow` state rules use existing tokens.
- `S-B4` must be added to the `FEED` set in `classify()` (`:4675`).
- `index_mobile_mockup.html` and `index_mockup.html` are not kept in sync.
- Do not break `window.pick`, `window.tog`, `window.togCap`, `window.DSHOW`,
  `window.FSHOW`, or `MMQ` / `MMASYNC` / `MMLADDER` / `MMSCHED` / `MMDECK` /
  `MMDEV` / `MMCONF` / `MMTRAIL` / `MMNUM` / `MMDECLINEDATE`. `MMDEV` is
  extended with a third row, not replaced: `setModel` and `setDepth` must still
  behave exactly as they do today.
