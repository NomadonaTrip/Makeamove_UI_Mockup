# Reviews & the personality profile — friction that reaches a person

**Date:** 2026-09-08
**Status:** Approved
**Phase:** 5 of the `prompt_2.md` cluster map
**Source decisions:** `2026-08-24-prompt2-decisions.md` §5.1–5.4 (Cluster 5, items 14, 17, 18)
**Target file:** `index.html` (single self-contained page)
**Screens touched:** `S-E2`, `S-E5`, `S-E6`, plus three new screens `S-E6A`, `S-E8`, `S-E9`

## Summary

Three unrelated-looking requests turn out to share one idea: the prototype
collects feedback nobody ever reads, and shows the user a profile it never
shows its working for.

`S-E2` lets you end a match with a single tap on a bare link — no reason asked,
nothing recorded, nobody told. `S-E5` invites a star rating and a private note,
neither of which is required and neither of which goes anywhere. `S-E6` asserts
that "we noticed you Moved on partners who were open about money struggles"
without ever showing the Moves it noticed.

This phase makes the decline cost something and reach someone, makes the
post-date review answer the one question calibration actually consumes, and
puts the receipts behind the profile's claims on screen.

## Decisions (locked)

Resolved in the brainstorming session of 2026-09-08. Each records its rejected
alternatives so a later reader can tell what was considered.

- **The declined party gets a screen.** §5.1's "shared verbatim" is the whole
  point of the decision, and a prototype that never renders David reading it
  leaves that as an assertion. Rejected: the sender's side only; and the
  receiving screen plus a notification row on `S-B2`, which reaches into
  Cluster 6's screen for no gain this phase.
- **The stated side of §5.3 is a fixture in `MMQ`, not live profile state.**
  Rejected: wiring `S-P1`–`S-P5` to persist their selections — five onboarding
  screens outside this cluster's scope, whose fields do not map onto the ten
  probes; and hardcoding the divergence rows, which would freeze a block whose
  whole value is that it moves when the answers do.
- **The evidence trail is its own screen, `S-E6A`.** Rejected: an inline
  expander on `S-E6`, which is already four cards tall on a `dark` surface;
  and an overlay, a heavier pattern than a row list needs.
- **No escape from the decline gate.** Unlimited rewrites, submit blocked until
  the text clears. Rejected: 3 attempts then accept whatever was written, which
  forwards the mush or the cruelty to a real person and defeats the check §5.2
  added precisely because the text now lands on someone; and a ghost-exit path
  after 3 failures, which invents a karma consequence Phase 6 has not specified.

## Scope note — this is a frontend mockup

No backend, and not pretending to one. The LLM substance check is a
deterministic function over a word list. The stated profile is a fixture. The
"inference" shown on each evidence row is computed from the demo answers, not
retrieved from a model. The purpose is to make the experience visible so it can
be iterated cheaply; nothing here is a data model.

---

## Screen inventory

| ID | Class | `data-title` | Purpose |
|---|---|---|---|
| `S-E6A` | dark | Evidence trail | Every question, your answer, their verdict, the inference |
| `S-E8` | paper | ⚠ Date declined · reason | The gate. Write a reason or you do not leave. |
| `S-E9` | dark | ⚠ You were passed on | David's view: the reason, verbatim |

### Why the decline branch is `S-E8`/`S-E9` and not `S-E2B`/`S-E2C`

Prev/Next steps in inventory order. Numbering the branch off `S-E2` wedges a
dead end into the middle of the date-planning arc, so stepping through the
happy path becomes `S-E2 → S-E2A → S-E2B (dead end) → S-E2C (dead end) →
S-E3`. Group F already resolved this: `S-F6`, the async decline, sits at the
end of its group rather than beside `S-F2A`, which is where it branches from.

`S-E6A` is the exception because it is a leaf off `S-E6` rather than a branch
out of the flow, and reads correctly in sequence.

### Desktop archetypes

`S-E6A` is added to the **`FEED`** set in `classify()` (`index.html:4205`) — it
is a long row list and the default centred form sheet would be wrong.

`S-E8` and `S-E9` are **not** added to either set. A centred sheet is correct
for both.

---

## Data — three additions to `MMQ`

Everything §5.3 needs is a derivation over data `MMQ` already holds, so it goes
there rather than into a fourth global beside `MMQ` / `MMASYNC` / `MMLADDER` /
`MMSCHED`.

### `DEMO.stated` — what you claimed

A probe-keyed map of claimed values, each traceable to a visible selection on
`S-P1`–`S-P5` so the fixture is not arbitrary. The comment in the source must
carry the citation, because a reader will otherwise assume the values were
invented to produce a pleasing number of divergences.

| Probe | Stated | Traceable to |
|---|---|---|
| `wants-children` | `yes` | `S-P5` — "How many kids": **2** |
| `five-year` | `family` | `S-P5` — "So eager — ready now" |
| `core-value` | `ambition` | `S-P3` — "Wit, ambition, a man who plans" |
| `weekend` | `adventure` | `S-P3` — "Salsa classes, weekend hikes" |
| `closeness` | `talk` | `S-P4` — "long voice notes" |
| `money-model` | `joint` | *inferred* — see below |

Five of the six trace to a field a user actually filled in. **`money-model`
does not** — no screen in `S-P1`–`S-P5` asks how you'd pool money, so its
stated value is inferred from the adjacent signals (`S-P1`'s income figure,
`S-P2`'s "Rich / Average"). The source comment must say so rather than implying
a claim the user never made. It is kept because it is one of only two probes
with three answered variations at both depths, which makes it the most solid
divergence in the set — but a later phase that adds a money-model field to the
profile should replace the inference with the real claim.

### `divergences(ans)` — what you played, against it

Returns one row per probe in `DEMO.stated`, each in exactly one of four states.
The four states exist because collapsing them loses the honesty the block is
for: a tie is not a divergence, and an unanswered probe is not agreement.

| State | Condition | Renders as |
|---|---|---|
| `agrees` | majority played value === stated | "You said it, and you played it." |
| `diverges` | majority played value !== stated | stated vs. played, called out |
| `unsettled` | no single majority (a tie) | "Your answers here don't agree yet." |
| `unplayed` | no answers on this probe | "You haven't played this one yet." |

**The fixture is tuned so the dev depth toggle changes the picture.** This is a
requirement, not an accident — the confidence meters beside it already have the
property, and a block that renders identically at Thin and Complete would look
hardcoded even though it is not.

| Probe | At Thin (`DEMO.answers`) | At Complete (`DEMO.answersFull`) |
|---|---|---|
| `wants-children` | agrees (`yes` ×3) | agrees |
| `core-value` | **diverges** — played `honesty`, said `ambition` | **diverges** |
| `money-model` | **diverges** — played `hybrid`, said `joint` | **diverges** |
| `weekend` | unsettled — `family` / `home` tie | unplayed |
| `closeness` | unplayed | **diverges** — played `rituals`, said `talk` |
| `five-year` | unplayed | unplayed |

Two divergences at Thin, three at Complete. **Thin renders all four states;
Complete renders three** (nothing is unsettled once the answers agree), which
is itself the demonstration — the block is visibly reading the answers rather
than reciting a fixture. `five-year` is unplayed in both by design: it is the
honest blank §2.7 requires, and it doubles as the pull toward answering more.

### `reviewReason(text)` — the §5.2 gate

Sibling to `reviewQuestion` (`index.html:980`), sharing its
deterministic-so-it-is-testable design. It is a sibling rather than a reuse
because `reviewQuestion` enforces question shape — a trailing `?`, ≥12
characters — which is wrong for a reason, and because §5.2 adds an abuse screen
that questions never needed.

```js
MMQ.reviewReason(text) → { ok, code, reason }
```

`code` is one of:

| Code | Rule | Message to the user |
|---|---|---|
| `short` | trimmed length < 40 | handled by the counter, not the reviewer |
| `mush` | matches the no-content list, or fails the mash heuristic | "That doesn't say anything he can learn from. Try again." |
| `abuse` | matches the cruelty list | "He is going to read this. Say the true thing without the sting." |
| `ok` | — | — |

The mash heuristic: a vowel ratio under 0.15, or any run of 5+ identical
characters. The no-content list covers "nothing really", "no reason", "idk",
"just because", "not feeling it" and similar — matched as the *whole* trimmed
text, so a real reason that happens to contain the phrase is not rejected.

---

## Behaviour

### §5.1 / §5.2 — the decline gate

`S-E2:3381`'s `Not quite — end politely (credit kept)` currently points at
`#S-B2`. It points at `#S-E8` instead. The "credit kept" promise is unchanged
and the copy stays.

**On `S-E8`:**

- A `.note` explains, plainly, that David reads this in your words. Not a
  warning — the honesty is the product, and framing it as a threat would
  produce exactly the polite lie §5.1 already accepted as the cost.
- A textarea with a live character counter. Submit carries `.btn.disabled`
  (`index.html:182`) until 40 characters.
- **The substance and abuse checks run on submit, not on input.** Flagging
  cruelty keystroke-by-keystroke would be jumpy and would teach the user to
  game the check rather than to write the true thing. Two stages is also what
  §5.2 literally describes: a minimum length **and** a check.
- A failed check renders `.note danger` inline above the textarea and stays on
  the screen. No attempt counter, no alternative exit.
- On success the text is written to `mm_decline_reason` and the user lands on
  `#S-B2`.
- The foot carries a secondary `Demo: see what David receives ›` to `#S-E9`,
  matching the existing demo affordance at `S-E4:3425`.

**On `S-E9`:** the reason rendered verbatim, no reply control, no rating. Falls
back to a fixture reason when storage is empty — the same
render-something-representative pattern `S-F4`'s `paint()` uses
(`index.html:3888`), and for the same reason: the screen is reachable by deep
link and by Prev/Next.

**The one hard rule in this phase.** The reason is user-typed text being
displayed back to a different person. It is set with `textContent`, never
interpolated into an `innerHTML` string. Every neighbouring render in this file
— `f4mirror`, `confBlock`, `deckList` — builds HTML by concatenation, so
following the local idiom here is the mistake. There is no other user-typed
string echoed to another user anywhere in the prototype.

### §5.3 — stated vs. revealed, on `S-E6`

A fifth card, **"What you said vs. what you played"**, placed above the
existing Finances nudge card (`index.html:3473`) — the nudge is a conclusion
drawn from this evidence and reads better after it.

Rendered by `MMCONF.paint()` — which grows to fill both `#confBlock` and the
new host — rather than by a second inline script, so the two derived blocks on
this screen repaint together and cannot drift out of sync when the depth toggle
changes. `MMCONF.reject()` and the `mm_conf_rejected` behaviour are unchanged.

Rows reuse `.confrow` and its `.cf-*` children (`index.html:612`), which are
already themed for both surfaces. The card's footer links to `#S-E6A`.

### §5.3 — the evidence trail, `S-E6A`

One row per question in the current set, in set order. Rows reuse
**`.mirrorrow` and its `.mr-*` children** (`index.html:633`) — already themed
for `dark` and `paper`, and already shaped as question-plus-two-labelled-sides
with flags. Do not invent a row class.

Each row carries:

- the question text
- your selected option, and your voice line when present
- David's verdict from `DEMO.verdict`, as the existing `.mr-v.move` /
  `.mr-v.stay` treatment
- the inference: which probe it counts toward, and whether it agrees with your
  other answers on that probe or conflicts with a specific one

Two states must render honestly rather than falling back to a default, exactly
as `S-F4` does:

- answered, but no verdict in `DEMO.verdict` → **"no verdict yet"**
- in the set but unanswered → **"not answered yet"** (`.mr-side.mr-none`)

A header line counts what is on screen — *"10 questions · 6 answered · 5 with a
verdict"*, illustrative only. The counts are computed from the set and the
answers, never written, so they move with the question-count model (10 under
Model A, 10–15 under Model B) and with the depth toggle.

### §5.4 — the number question, on `S-E5`

A required block under the rating card and above the private note.

- "Did you exchange numbers?" — Yes / No, using the `.opt` + `.dot` pattern.
- Submit (`index.html:3453`) starts `.disabled`.
- **Yes** enables submit.
- **No** reveals six MECE options and keeps submit disabled until one is
  picked, plus an optional free-text line.

MECE options for a no:

- We just didn't get to it
- I wasn't interested
- They didn't seem interested
- I'd rather keep it in-app for now
- I didn't feel comfortable
- Something else

A straight port of `S-F6`'s `MMDECLINE` pattern (`index.html:3976`) as
`MMNUM`, writing `mm_numbers`. Note the deliberate asymmetry §5.4 records: this
one is structured because calibration consumes it, while §5.1's decline is free
text because the friction is its purpose.

---

## Verification

Playwright MCP, per screen, at **both 1280×800 and 390×844**. Prior phases in
this repo passed diff review with screens visibly broken; the viewport sweep is
not optional and not a spot check.

Two measurement traps carried forward, both of which cost real time in earlier
phases:

- **Overflow on `S-E6` and `S-E6A`** is measured from child rects against the
  container box, not `scrollHeight`. These are centred flex columns, content
  spills in both directions, and `scrollHeight` counts only part of it. A
  Phase 3 review under-reported a spill by ~115px this way. Let entry
  animations settle first.
- **Assertions query rendered elements, never a screen's `textContent`.** The
  inline `<script>` fixtures sit inside the `<section>`, so `textContent`
  contains every fixture string and any substring assertion false-positives.

Specific to this phase, assert that:

- `S-E8` cannot be submitted at 39 characters, and cannot be submitted with 60
  characters of mush or cruelty.
- The reason shown on `S-E9` is byte-identical to what was typed, and that a
  reason containing `<b>` renders as literal text.
- Toggling dev depth Thin → Complete changes the divergence rows on `S-E6`.
- `S-E5`'s submit is disabled on arrival, enabled by Yes, and re-disabled by No
  until a reason is picked.

## Constraints carried from `CLAUDE.md`

- Single self-contained file. No external CSS/JS, no libraries, no build step.
- Reuse the `:root` custom properties and the shared `.btn` / `.card` / `.row`
  / `.chip` / `.opt` / `.note` / `.confrow` / `.mirrorrow` components. Invent
  no new colours.
- `S-E6A` must be added to the `FEED` set in `classify()`.
- `index_mobile_mockup.html` and `index_mockup.html` are not kept in sync.
- Do not break `window.pick`, `window.tog`, `window.togCap`, `window.DSHOW`,
  `window.FSHOW`, or `MMQ` / `MMASYNC` / `MMLADDER` / `MMSCHED` / `MMDECK` /
  `MMDEV`. `MMCONF` is extended, not replaced: its `reject()` contract and the
  existing confidence rows must still work exactly as they do today.
