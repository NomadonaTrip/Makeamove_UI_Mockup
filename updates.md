Modify the flow so that when the user completes the Onfido verification, they're taken immediately to the Matching page. On clicking the "Matches Ready" button, they are shown the people they're prematched with.

There should be a button or some other way by which they may readily return to their profiles to enrich it. It is important that they don't feel that they have to complete their profiles to get matched with potential spouses.

---

# PR body — Phase 4 · Scheduling core

*(Not a request. Parked here for copying into the pull request for `feat/scheduling-core`:
https://github.com/NomadonaTrip/Makeamove_UI_Mockup/pull/new/feat/scheduling-core)*

Implements **Cluster 4** (prompt_2 items 13, 15 and the ⭐).

Spec: `docs/superpowers/specs/2026-09-06-scheduling-core-design.md`
Plan: `docs/superpowers/plans/2026-09-06-scheduling-core.md`

## What changed

Scheduling was three unrelated pieces of hardcoded markup and one number nobody chose. `S-C1` listed five fixed rows; `S-C4` hand-built overlap cards; `S-E4` announced "Saturday 28 Jun · 7:30 pm" for a date no screen ever picked; and both the sync and async wins jumped straight into the live video call with nothing scheduling it — `S-F4`'s button literally read "Schedule the video call" while scheduling nothing.

- **`MMSCHED`** — one component, two modes. `propose` (pick up to 5 of your own times) at `S-C1`; `choose` (three suggestions, one tap, plus a full-screen "see all times" week grid) at `S-C4`, `S-E0` and `S-E2A`.
- **`S-E0`** *(new)* — the post-pass call screen, reached from **both** `S-D5` and `S-F4`. "Call now" fires a 60-second ring overlay; unanswered, it closes back onto the scheduler with no dead end.
- **`S-E2A`** *(new)* — the physical date's time, which is what `S-E4` had been missing.
- `S-E3`'s hardcoded "near Canary Wharf" becomes a computed midpoint that discloses neither party's address.
- A Lagos-resident candidate, so a cross-timezone pair exists at all.

## Decisions worth reviewing

**Three storage keys, never one.** The session (`S-C4`), the call (`S-E0`) and the date (`S-E2A`) are three different facts. A shared key would make picking one silently rewrite another, so `S-C5` would start announcing the wrong thing. Each mount namespaces under `mm_slot_<key>`, and independence is asserted end-to-end.

**`S-C1` proposes; everything after chooses.** At invite time the invitee hasn't engaged, so there is no overlap to compute — and `S-C3` exists precisely to *receive* a proposal. A top-3 picker at `S-C1` would have left it with nothing to receive.

**The ring is an overlay, not a screen.** Unanswered, it simply closes, and the caller is already standing in the scheduler.

**Overlays anchor to `.screen`, and the router closes them.** Three separate mechanisms tried to trap them: an inline `position:relative`, the site-wide entrance stagger's persistent `transform`, and `.screen` being its own scroll container. The last one shipped a Critical — on a scrolled phone the grid opened 50px high with its close button clipped off, undismissable — invisible at any viewport ≥844px.

## Verification

Browser-driven throughout (no test suite in this repo by design). Eight task reviews, a whole-branch review, and a final verification pass at 390×667, 390×844, 1280×800 and 1440×900.

Defects caught after their own task passed review: an overlay surviving navigation and leaking its scroll freeze (making a section permanently unscrollable); the two timezone lines collapsing onto one line inside the grid; a Cancel button rendering dark-on-dark; `S-C1` badging "both free" before the invitee existed; `S-C4`'s colour legend outliving its key; and `S-E0`'s confirm dropping the user into the live call after agreeing a future time.

## Known gaps, deliberately not fixed

- **The `WAT` path renders nowhere in a walkthrough.** David is `city:'London'`, so every scheduling screen shows two identical GMT lines — correct per "always show both", but the cross-timezone case and the day-roll branch go unexercised. A build task set David to WAT to make it visible; that was reverted, because a false fixture is worse than an undemonstrated one. Closing it properly means making the Lagos persona the scheduling partner — a product decision about who the demo follows.
- `S-E0` needs a scroll at 390×667 (124px over a 626px band); `S-C1` already scrolls 641px there and shipped that way.
- `S-C3` still shows "both free" before David has engaged — same class as the `S-C1` fix, on a screen the final fix wave's scope didn't name.
- `S-F3`'s action-bar overlap at 390×667 — pre-existing on `main`, untouched here.

## One caveat on process

The scoped re-review of the final fix wave was cut off by a weekly rate limit, so **I ran that verification myself rather than an independent reviewer.** All six findings measured as addressed, plus a probe nobody had run (`closeAllOverlays()` is now `route()`'s first statement in a 71-screen app — walked five non-scheduler screens, no error). Browser back/forward and the full 12-combination escape matrix rest on the fixer's own measurements, not an independent check.
