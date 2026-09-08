Modify the flow so that when the user completes the Onfido verification, they're taken immediately to the Matching page. On clicking the "Matches Ready" button, they are shown the people they're prematched with.

There should be a button or some other way by which they may readily return to their profiles to enrich it. It is important that they don't feel that they have to complete their profiles to get matched with potential spouses.

---

# PR body — Phase 5 · Reviews & the personality profile

*(Not a request. Parked here for copying into the pull request for `feat/reviews-profile`:
https://github.com/NomadonaTrip/Makeamove_UI_Mockup/pull/new/feat/reviews-profile)*

Implements **Cluster 5** (prompt_2 items 14, 17, 18).

Spec: `docs/superpowers/specs/2026-09-08-reviews-profile-design.md`
Plan: `docs/superpowers/plans/2026-09-08-reviews-profile.md`

## What changed

Three requests that turn out to share one idea: the prototype collected feedback nobody ever read, and showed the user a profile it never showed its working for. `S-E2` ended a match with one tap on a bare link — no reason asked, nothing recorded, nobody told. `S-E5` invited a rating and a note, neither required and neither going anywhere. `S-E6` asserted "we noticed you Moved on partners who were open about money struggles" without ever showing the Moves it noticed.

- **`S-E8`** *(new)* — declining after the call now costs a sentence. 40 characters live, then a substance-and-cruelty check on submit. No attempt counter and no escape hatch: §5.1 decided the friction is the point.
- **`S-E9`** *(new)* — the declined party reads that reason **verbatim**. Set with `textContent`, never `innerHTML` — the only user-typed string in this prototype shown to a different person, and every neighbouring render here concatenates HTML, so the local idiom was the trap.
- **`S-E6`** — a "what you said vs. what you played" card, driven by a new `MMQ.divergences()` over a `DEMO.stated` fixture whose values each trace to the `S-P1`–`S-P5` field they came from.
- **`S-E6A`** *(new)* — the full evidence trail: every question, your answer, his verdict, what was inferred.
- **`S-E5`** — a compulsory "did you exchange numbers?" that blocks submission, with six MECE options behind a "no".

## Decisions worth reviewing

**Four divergence states, not two.** `agrees` / `diverges` / `unsettled` / `unplayed`. Collapsing any two loses the honesty the block exists for: a tie is not a divergence, and a probe you have never been asked about is certainly not agreement. The fixture is tuned so the dev depth toggle visibly changes the picture — Thin renders all four states, Complete renders three.

**`money-model`'s stated value is inferred, and says so.** Five of the six probes trace to a field a user actually filled in. No screen asks how you would pool money, so that one is marked INFERRED in the source rather than implying a claim the user never made.

**The substance check counts distinct content words, not banned phrases.** A blocklist of "idk" / "nothing really" would have been dead code — the 40-character floor rejects every such phrase before any check runs, so padding is the only way mush arrives long enough to test. Known and accepted: this also rejects thin-but-honest lines like *"There was no chemistry for me at all here."* Per §5.1 that is correct behaviour, not a false positive.

## Two defects only a whole-branch review could see

Both came from `S-E6` and `S-E6A` reading **different universes of questions**; each half was correct alone, so six per-task reviews passed them.

- **The receipts screen was silent about the claims it backs.** `S-E6A` built from the 10-question set (Family and Values only) while the divergence card computed over *all* answers. `money-model` and `closeness` are never in the default set, so clicking "See the full evidence trail" from a highlighted divergence landed on a screen with zero rows about it. The trail now appends answered questions from outside the current set.
- **`S-E6` never repainted on navigation** while `S-E6A` always did. Play the sync show, walk to your profile: stale summary, with the dev panel beside it reading "Complete" — and fresh data one click away on the trail.

## Known, and deliberately not fixed

The gate is bypassable via the ☰ screen index, Prev/Next, the browser back button and direct hash edits. The router sets `location.hash` unconditionally by design and the index exists so any screen reaches any other; `S-F6`'s async decline already behaves the same way. §5.1 was written about the screen's own controls, which are gated. Router-level enforcement would be a separate, cross-cutting decision.

## Verification

Every touched screen screenshotted at 1280×800 and 390×844. The decline flow walked end to end on mobile, confirming a mush submit **stays** on `S-E8` with the hash unchanged. Escaping asserted by feeding `<img onerror>` through the reason. The depth toggle verified to repaint all three derived surfaces. Regression pass over `S-D2`, `S-F3`, `S-F4`, `S-C1` — the screens leaning hardest on `MMQ` — with zero console errors.
