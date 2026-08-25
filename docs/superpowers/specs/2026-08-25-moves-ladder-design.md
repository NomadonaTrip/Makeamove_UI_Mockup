# Moves Stacking — the Millionaire Ladder

**Date:** 2026-08-25
**Status:** Approved
**Phase:** 3 of the `prompt_2.md` cluster map
**Source decision:** `2026-08-24-prompt2-decisions.md` §3.1 (Cluster 3, item 4)
**Target file:** `index.html` (single self-contained page)
**Screens touched:** `S-D2` (the arena), `S-F3` (async board — compact variant only)

## Summary

`S-D2` currently reports the other party's verdicts as a horizontal strip of
22px tokens in the `.base` slot: a pink heart for a Move, a grey ✕ for a Stay,
no dimension, no points, no relationship to the 70% gate. Item 4 asks that the
other party's Moves visibly *stack up*. This design replaces the strip with a
vertical points ladder — rungs light as verdicts land, each carrying its
dimension and what it was worth, with the 70% gate drawn as a line across the
rail.

The ladder is built once as a shared component and used twice: as a full
vertical rail in the arena, and as a compact horizontal variant replacing
`S-F3`'s `.mini` token strip.

## Decisions (locked)

Resolved in the brainstorming session of 2026-08-25. Each is recorded with its
rejected alternatives, so a later reader can tell what was considered.

- **Scope:** `S-D2` gets the rail; `S-F3`'s `.mini` strip becomes a compact
  variant of the same component; `S-F3`'s flip board is **not** rebuilt.
  Rejected: a generic rail parameterised by whose score it tracks, replacing
  the flip board too.
- **Rung unit:** one rung per *verdict on you*. Rejected: one rung per
  question with the other party's turns muted, and a dual-coded rail carrying
  both scores.
- **Gate:** uniform rung heights, gate line at the cumulative-points crossing,
  plus a foot readout. Rejected: weight-proportional rung heights, and
  Millionaire-style cumulative prize tiers.
- **Phone placement:** in the `.base` slot, in flow, stage squeezed. Rejected:
  a narrow right rail at phone widths, and a peek strip that expands on tap.
- **`S-F3` variant:** horizontal compact rail. Rejected: a 3-rung vertical
  mini window, and a single-line condensed readout.
- **Unreachable gate:** desaturate the gate line, say nothing. Rejected: an
  explicit "you can no longer pass" message.

---

## Why one rung per verdict, not per question

`S-D2` alternates turns — `index.html:2464`, even index you answer, odd index
David answers and you react. David therefore renders a verdict on you on only
half the questions, and your score is accumulated from those turns alone
(`yA`/`yE` at `index.html:2524`).

Making a rung *a verdict on you* means the rail's universe is exactly the set
of events that build your score. "How far am I from the gate" is then
arithmetically true rather than approximately true. A rail with one rung per
question would have half its rungs contributing nothing to the gate it draws.

The consequence is that arena rung count is `ceil(questions / 2)` — 5 under
Model A, 6–8 under Model B — and async rung count is `questions`, because on
`S-F3` the other party has recorded a verdict on every one of your answers.
Both are driven by the question count and neither is hardcoded, which is what
§3.1 requires.

## The component — `MMLADDER`

Declared beside `MMQ` and `MMASYNC`, **before `#screenwrap`**: both consumers'
inline scripts run at parse time and would otherwise consume an undefined
global. Same constraint recorded at `index.html:640`.

```js
MMLADDER.mount(el, {
  rungs:   [{ dim:'Family', w:3.0 }, …],   // ordered, variable length
  subject: "David's reactions to you",      // rendered as the rail's header
  layout:  'rail' | 'compact'
})
```

Returns a handle:

| Member | Behaviour |
|---|---|
| `set(i, 'move'\|'stay')` | Resolves rung `i`, animates it, repaints the foot |
| `focus(i)` | Scrolls the window so rung `i` is in view |
| `reset(rungs)` | Rebuilds from scratch — used by the screens' `reset()` hooks |
| `gateIndex` | Index of the gate rung (see below) |
| `gatePts` | Points needed to clear |

The component holds no game state beyond its own rung verdicts. The caller
owns the mapping from its turn index to a rung index: `S-D2` keeps a
`rungOf[turnIndex]` lookup because only even turns produce rungs, and `S-F3`
maps 1:1.

Points are `Math.round(w * 10)`, reusing the convention already live on the
async board at `index.html:3134`, so the arena and the board quote the same
numbers for the same dimension.

## Gate arithmetic

Let `A_k` be cumulative available points through rung `k`, and
`G = 0.70 × A_total`.

**`gateIndex` is the smallest `k` where `A_k ≥ G`.** The gate divider renders
immediately *above* that rung, so the statement it makes is: win everything
below this line and you have cleared.

Worked against the live bank, Model A with no deck attached. `buildSet()`
returns the first ten bank questions — six Family, then four Values
(`index.html:660`) — and your turns are indices 0, 2, 4, 6, 8:

| Rung | Question | Dim | Pts | Cumulative |
|---|---|---|---|---|
| 1 | `q-fam-01` | Family | 30 | 30 |
| 2 | `q-fam-03` | Family | 30 | 60 |
| 3 | `q-fam-05` | Family | 30 | 90 |
| 4 | `q-val-01` | Values | 15 | 105 |
| 5 | `q-val-03` | Values | 15 | 120 |

`A_total` = 120, `G` = 84, and the smallest `A_k ≥ 84` is rung 3. The gate line
sits above rung 3.

The index genuinely moves with the deck. Attach five custom questions — which
carry `dim:'Values'`, `index.html:1566` — and under Model A they displace bank
questions from the front of the set, so your rungs become 15, 15, 15, 30, 30:
`A_total` = 105, `G` = 73.5, cumulative 15/30/45/75/105, and the gate lands
above rung 4. Nothing about the gate may be precomputed or hardcoded.

## Rendering

**Order.** Rungs climb bottom-to-top, rendered in **reversed DOM order** rather
than with `flex-direction: column-reverse`. Reversed DOM keeps visual order and
reading order in sync; `column-reverse` would have a screen reader announce the
rungs backwards.

**Pending rungs show their dimension and their points.** This reveals which
dimensions are still coming, and that is deliberate: Millionaire shows the
whole money ladder, and seeing a 30-point Family rung ahead is what makes the
climb worth playing. It is also what lets the foot readout's "still available"
figure be checked by eye.

**Resolved rungs** show `♥` and the points won, or `✕` and an em dash for a
Stay — the wording §3.1 specifies.

**Foot readout**, recomputed on every `set()`:

```
60 won · 24 to go · 30 left
```

- *won* — points from Move rungs
- *to go* — `max(0, gatePts − won)`
- *left* — points still on unresolved rungs

**Unreachable gate.** When `won + left < gatePts`, clearing has become
mathematically impossible. The gate line **desaturates to grey** and nothing
else changes: no message, no alarm. The numbers in the foot readout already
state the position honestly, and the player still has turns to sit through.
Failure is announced on `S-D6`, which is the screen whose job that is.

## `S-D2` — the arena

**Markup.** `.base` stops holding `.blbl` + `.tokens` and holds a single mount
element. The header text ("David's reactions to you ↓") moves into the
component as `subject`.

**Wiring.** `addTok()` is deleted. `lock()` calls `ladder.set(rungOf[i], v)` on
the same 600ms beat that currently pops a token (`index.html:2529`), so the
existing reveal choreography is unchanged. `render()` calls `focus()` on the
upcoming rung when it is your turn. `DSHOW.reset()` calls `ladder.reset()`
instead of clearing `#tokens`.

**Phone window.** `max-height` sized to roughly five rungs plus the divider,
`overflow-y:auto`, scrollbar hidden. Anchoring uses
`scrollIntoView({block:'nearest'})` — the technique already proven on the async
board at `index.html:3126`.

The rail costs the `.stage` roughly 150px. That is affordable at 667px tall and
tight below; the question ticket keeps full width, which matters at 23px
display type.

**Desktop (≥768px).** `S-D2` is already in the `IMMERSIVE` set
(`index.html:3536`). §3.1 asks for the full ladder as a right rail, so
`#S-D2.lay-immersive` becomes a grid — no markup change, honouring the reflow
contract in `CLAUDE.md`:

```
grid-template-areas: "heads ladder" "flag ladder" "stage ladder" "act ladder";
grid-template-columns: minmax(0, 620px) 240px;
```

The rail drops its `max-height` here and shows every rung.

Three points of care:

- The grid rule must be scoped to `#S-D2`, not `.lay-immersive`, or it will
  also reflow `S-E1`, the other member of the set.
- `.screen.on` sets `display:flex`; `#S-D2.lay-immersive.on` outranks it on
  specificity, so the grid wins without `!important`.
- `.esc` is positioned `left:50%` (`index.html:2441`) and would centre on the
  viewport rather than on the stage column once the grid offsets it. It needs
  a desktop override. `.reveal` stays `inset:0` — it is a full-screen moment
  and should cover the rail too.

## `S-F3` — the compact variant

`#fmini` becomes the mount point with `layout:'compact'`. Tiles run
horizontally and wrap, each carrying its verdict glyph over its points, with a
gate divider inserted at `gateIndex` and the foot readout inline beneath.

`freact()`'s `tok.classList.add('on', hv)` (`index.html:3143`) becomes
`ladder.set(fi, hv)`; `FSHOW.reset()` calls `ladder.reset()`.

The flip board, its `32vh` cap and its scroll behaviour are untouched. The
comment at `index.html:3016` — *"Phase 3 replaces this with a windowed
ladder"* — is now wrong and must be rewritten to say what Phase 3 actually
did, so it stops promising a rewrite that is not happening.

## Out of scope

Named here so a later reader can tell these were considered and declined
rather than missed:

- Rebuilding `S-F3`'s flip board as a ladder.
- Weight-proportional rung heights.
- Any change to `S-D2`'s two score heads — §3.1 keeps them explicitly, since
  they already carry both players' percentages and their own 70% gate tick
  (`index.html:2373`).
- `index_mobile_mockup.html` and `index_mockup.html`, per `CLAUDE.md`.

## Verification

Playwright MCP at **390×844** and **1440×900**, under **both count models** via
the dev-settings toggle — Model B is the real test, since 15 questions give 8
arena rungs and 15 async tiles.

| Check | Where |
|---|---|
| Rung count tracks the question count, both models | `S-D2`, `S-F3` |
| Gate line lands above the correct rung, deck attached and not | `S-D2` |
| Window anchors on the current rung as play advances | `S-D2` phone |
| Full rail renders as a right column, no clipping | `S-D2` desktop |
| Foot readout arithmetic matches the score heads at completion | `S-D2` |
| Gate line desaturates when the gate becomes unreachable | `S-D2` |
| Compact rail does not push `S-F3`'s stage into the button row | `S-F3` phone |
| `.esc` escalation banner still centres on the stage | `S-D2` desktop |

A visual pass at both viewports is mandatory, not optional: a diff review of
this change can read clean while the screen is visibly broken.
