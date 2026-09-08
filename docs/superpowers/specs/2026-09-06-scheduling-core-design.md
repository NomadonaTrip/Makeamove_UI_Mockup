# Scheduling Core — one scheduler, reused everywhere

**Date:** 2026-09-06
**Status:** Approved
**Phase:** 4 of the `prompt_2.md` cluster map
**Source decisions:** `2026-08-24-prompt2-decisions.md` §4.1–4.4 (Cluster 4, items 13, 15, ⭐)
**Target file:** `index.html` (single self-contained page)
**Screens touched:** `S-C1`, `S-C3`, `S-C4`, `S-E3`, `S-E4`, plus two new screens `S-E0` and `S-E2A`

## Summary

Scheduling is currently three unrelated pieces of hardcoded markup and one
number nobody chose. `S-C1` lists five fixed `.opt` rows. `S-C4` hand-builds
overlap cards. `S-E4` announces "Saturday 28 Jun · 7:30 pm" for a date whose
time is never picked on any screen. Two flows — sync pass and async pass —
jump straight into `S-E1`'s video call with nothing scheduling it, and
`S-F4`'s button reads "Schedule the video call" while scheduling nothing.

This phase builds one scheduler component and uses it at every scheduling
moment in the journey, adds the two screens the flow is missing, replaces
`S-E3`'s hardcoded neighbourhood with a computed midpoint, and puts both
parties' timezones on every proposed slot.

## Decisions (locked)

Resolved in the brainstorming session of 2026-09-06. Each records its rejected
alternatives so a later reader can tell what was considered.

- **`S-C1` proposes; everything after chooses.** One component, two modes.
  Rejected: top-3 at `S-C1` too, and top-3 plus an "add your own" row — both
  would have required rethinking `S-C3`, which exists to receive a proposal.
- **One call screen serves both paths.** `S-D5` and `S-F4` both land on `S-E0`.
  Rejected: async only (as §4.1 literally says), and one screen whose default
  flips per path.
- **A Lagos-resident candidate is added, and both timezones always render**,
  even when identical. Rejected: rendering the second line only when the zones
  differ, and leaving the fixtures UK-only so the feature never demos.
- **The physical date gets its own time screen**, `S-E2A`, before `S-E3`.
  Rejected: time and venue together on `S-E3`, and venue first with the time
  moved onto `S-E4`.
- **The 60-second ring is an overlay, not a screen.** Rejected: a separate
  ringing screen with its own id.

## Scope note — this is a frontend mockup

The prototype has no backend and is not pretending to. Every "system-computed"
value in this phase — suggested times, mutual overlaps, the venue midpoint — is
a fixture presented as if computed. The purpose is to make the user experience
visible so it can be iterated cheaply. Nothing here should be read as a data
model or an availability subsystem, and no decision below depends on one.

---

## The component — `MMSCHED`

Declared beside `MMQ`, `MMASYNC` and `MMLADDER`, **before `#screenwrap`**:
consumers' inline scripts run at parse time and would otherwise read an
undefined global. Same constraint recorded at `index.html:720`.

```js
MMSCHED.mount(el, {
  mode:  'propose' | 'choose',
  self:  { name:'Tolu',  tz:'GMT' },
  other: { name:'David', tz:'GMT' },
  max:   5                       // propose mode only; ignored in choose mode
}) → handle
```

Handle:

| Member | Behaviour |
|---|---|
| `selected()` | The chosen slot object, or `null` |
| `reset()` | Rebuild from fixtures, clearing any selection |
| `openGrid()` | Open the full week grid overlay |

A slot is `{ id, day, time, mutual }` — `day` a display string, `time` a
24-hour `HH:MM` in `self`'s zone, `mutual` a boolean the choose mode uses to
mark suggestions. Slots come from a module-level fixture, in the same spirit as
`MMQ.DEMO`.

**`propose` mode** (`S-C1`): pick up to `max` of your own times. Selection is
multi, capped, and the cap is enforced visibly rather than silently.

**`choose` mode** (`S-C4`, `S-E0`, `S-E2A`): three suggested times as tappable
rows, one tap to select, with a **"see all times"** link opening the full
paintable week grid as an overlay. Easy by default, robust on demand — which is
what the ⭐ note asks for.

The chosen slot is written to `sessionStorage` so later screens display what was
actually picked rather than a hardcoded string, the same way `mm_py`/`mm_pt`
and `MMASYNC.saveResult()` already work.

## Timezones

§4.4 asks that every proposed slot render in both parties' zones. A small
offset map drives it:

```js
const TZ = { GMT: 0, WAT: 1 };
```

**Both lines render always, even when the zones are identical.** A reader never
has to notice whether a second line is missing, which is the safety the decision
is for.

**One fixture must change for this to mean anything.** Every persona in the
prototype resides in the UK or Dublin — city of residence defaults to London
(`index.html:1470`) and the candidate blurbs are London, Reading, Croydon,
Luton, Birmingham, Manchester, Dublin (`index.html:1929`–`2043`). Lagos and
Abuja appear only as *state-of-origin* options in onboarding
(`index.html:1354`–`1357`), never as residences. So a cross-timezone pair is
currently unreachable and the feature would demo nothing. **A Lagos-resident
candidate is added to the gallery fixture**, which matches §4.4's own rationale
about a Nigerian-diaspora product.

**Known gap, recorded honestly (added 2026-09-07).** The Lagos candidate makes a
cross-timezone pair *exist*, but the scheduling flow's partner is hardcoded as
David, who is `city:'London'`. Every scheduling screen therefore renders two
identical `GMT` lines — correct per "always show both", and exactly the
same-zone case this section describes, but it means **the `WAT` path and
`fmtLine`'s day-roll branch are never exercised in a walkthrough.**

During the build one task set the partner to `WAT` to make the feature visible.
That was reverted: David lives in London, and a false fixture is worse than an
undemonstrated one. Closing the gap properly means making the scheduling partner
the Lagos persona — a product decision about who the demo narrative follows, not
a rendering fix, and deliberately left to a later phase.

## Screens

### `S-C1` — propose

The five hardcoded `.opt` rows (`index.html:2405`–`2409`) become `MMSCHED` in
propose mode, capped at 5. Everything else on the screen is unchanged.

### `S-C3` — receive a proposal

Structurally unchanged: it still reads "She proposed N times — pick one, or
offer your own". Its slot list is re-pointed at the same fixture so the count
is truthful and each row gains the timezone line. This screen is why `S-C1`
stays a proposer: a top-3 picker at `S-C1` would leave `S-C3` with nothing to
receive.

### `S-C4` — choose

The bespoke overlap cards (`index.html:2457`–`2465`) become `MMSCHED` in choose
mode. The existing legend showing whose slots are whose is kept, since the
week grid needs the same key.

### `S-E0` — the call screen *(new)*

Sits physically before `S-E1` so Prev/Next inventory order follows the flow.
Reached from **both** `S-D5` (sync pass) and `S-F4` (async pass) — both
currently jump straight into the call, and `S-F4`'s button already promises
scheduling it does not do.

Two actions: **Call now** (§4.2) and `MMSCHED` in choose mode.

### `S-E2A` — the date's time *(new)*

Sits between `S-E2` and `S-E3`, following the `S-F2A` precedent for an inserted
screen. `MMSCHED` in choose mode. This is the screen whose absence let `S-E4`
announce a time nothing had chosen.

### `S-E3` — venue

`index.html:2890` currently reads "Google Places · near Canary Wharf". It
becomes a **computed midpoint district**, with copy making explicit that
neither party learns where the other lives or works (§4.3). The venue rows
themselves stay as they are; the search box stays decorative.

### `S-E4` — date confirmed

Displays the slot actually chosen at `S-E2A`, with both timezone lines, instead
of the hardcoded "Saturday 28 Jun · 7:30 pm".

## The instant call

§4.2's "call now" rings the other party for 60 seconds. **The ring is an
overlay on `S-E0`, not a separate screen** — the same pattern as `S-D2`'s
`.reveal` (`index.html:2590`).

- **Answered** → `S-E1`, the existing Whereby call.
- **Unanswered** → the overlay closes. The caller is already standing in the
  scheduler, which is exactly the "no awkward dead end" §4.2 asks for, and it
  costs no extra screen.

`S-E0` carries a demo link to fire the overlay directly, matching the
"demo · expire ›" affordance on `S-F3` (`index.html:3196`).

Presence-gating was considered and rejected in §4.2 itself: an online/offline
indicator on a dating platform leaks when each person is using the app.

## Desktop reflow

`S-E0` and `S-E2A` are centred form sheets and need no entry in the `FEED` or
`IMMERSIVE` sets in `classify()` — the default archetype is correct for both.
`S-E3` is already in `FEED` and stays there. The week-grid overlay is
absolutely positioned within its screen and is unaffected by archetype.

## Out of scope

Named so a later reader can tell these were considered and declined:

- Real timezone arithmetic beyond the offset map — no DST, no IANA zones.
- Venue search. `S-E3`'s search box remains decorative, as today.
- Calendar integration or invitations.
- Any change to `S-C5`'s soft-hold and CRA-waiver copy beyond displaying the
  chosen slot.
- `index_mobile_mockup.html` and `index_mockup.html`, per `CLAUDE.md`.

## Verification

Playwright MCP at **390×844** and **1440×900**.

| Check | Where |
|---|---|
| Propose mode caps at 5 and says so | `S-C1` |
| `S-C3`'s stated count matches the fixture | `S-C3` |
| Choose mode renders 3 suggestions; "see all" opens the grid | `S-C4`, `S-E0`, `S-E2A` |
| Both timezone lines on every slot, including identical zones | all scheduling screens |
| A Lagos pairing renders a genuinely different second line | `S-C4` |
| The chosen slot propagates to `S-C5` and to `S-E4` | `S-C5`, `S-E4` |
| `S-E0` reachable from both `S-D5` and `S-F4` | `S-D5`, `S-F4` |
| Ring overlay: answered → `S-E1`; unanswered → back to the scheduler | `S-E0` |
| Midpoint copy discloses no location | `S-E3` |
| Week-grid overlay does not spill its screen | all, both viewports |

A visual pass at both viewports is mandatory. Two prior phases in this repo
passed clean diff review while a screen was visibly broken, and this phase adds
two screens and an overlay to fixed-height flex columns — the exact condition
that produced those failures. When measuring overflow on a centred flex column,
compute it from child rects: `scrollHeight` understates spill on these screens.
