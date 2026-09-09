Modify the flow so that when the user completes the Onfido verification, they're taken immediately to the Matching page. On clicking the "Matches Ready" button, they are shown the people they're prematched with.

There should be a button or some other way by which they may readily return to their profiles to enrich it. It is important that they don't feel that they have to complete their profiles to get matched with potential spouses.

---

# PR body — Phase 6 · Karma points

*(Not a request. Parked here for copying into the pull request for `feat/karma-points`:
https://github.com/NomadonaTrip/Makeamove_UI_Mockup/pull/new/feat/karma-points)*

Implements **Cluster 6** (prompt_2 items 10, 11). Badges remain deferred per §6.5.

Spec: `docs/superpowers/specs/2026-09-08-karma-points-design.md`
Plan: `docs/superpowers/plans/2026-09-08-karma-points.md`

## What changed

Karma did not exist in this prototype. Two mentions in 336KB: a CSS comment reserving a dev-panel row for it, and `S-E9`'s promise that "your karma is untouched" — a reassurance about a quantity the product had never once shown. Meanwhile `S-B2` ordered its candidates by a stored array, so §6.3's one consequence was not in the product either.

- **`MMKARMA`** *(new)* — standing is a derivation over the four behaviours §6.2 rewards (responsiveness 30, showing up 30, profile completeness 20, honest feedback 20), never a stored number. Screens declare a host (`data-karma-of` / `-head` / `-ledger` / `-raise`) and `paint()` fills it on navigation.
- **`S-B2`** — the gallery now orders by `fit × (0.7 + 0.3 × karma/100)`. Samuel renders **above** David at 68% fit against David's 76%, because David's conduct score is 33. That inversion is the feature.
- **`S-C3` / `S-F2A`** — your standing as *someone else* sees it. This is the mirror §6.3 is actually about.
- **`S-B4`** *(new)* — your standing, the ledger of everything that moved it (including what you lost), and a derived "what would raise it".
- **Dev settings** — a third row toggling §6.4's two undecided display directions.
- **Seven existing actions now earn karma**, and a **"3 people pursued you this month"** card (§6.6) invites the specific questions those people were asking about.

## Decisions worth reviewing

**One data shape, two renderings.** Each person carries four sub-scores. Direction A renders the ones clearing a gate as trait phrases; Direction B renders their weighted mean as a number. Had Direction A's traits been their own fixture copy, the A/B test would compare a real number against invented sentences and could settle nothing.

**`traits()` returns positive traits only.** §6.3's "never announcing to anyone that they have been penalised" is enforced in the derivation, not trusted to the copy — a low-standing candidate carries fewer chips, or none. You see your own penalties; nobody else ever does.

**Direction B necessarily leaks low standing where Direction A structurally cannot.** David's card simply reads `33`. That is not a bug in either direction — it is a real difference the A/B test should weigh against §6.3's intent, and it is recorded in the spec rather than papered over.

**David is deliberately the strongest fit and the weakest karma.** The sort has to visibly disagree with the fit column or §6.3 is unprovable. Retuning either his record or the rank formula means re-deriving that; the source comment says so.

**Blank profile sections derive from the ledger.** The final review caught `S-B4` contradicting itself once this phase's own events fired — a ledger row saying "Completed Personality & lifestyle" sitting directly above a card insisting it was still blank. Fixed by deriving from the ledger rather than mutating the gaps fixture, because a mutated array is lost on reload while the ledger row persists.

**The seeded ledger names nobody.** A named past partner would be a fourth person the prototype never shows, and the fixture world has no answer when a reader asks who they were.

**Deliberately absent:** no leaderboard in any form (§6.1); no conduct events at all, since §6.2 requires those be scored only after an operator ruling, which is Phase 7's `S-H2`; and no live penalty path, because the prototype offers no way to ghost — you cannot click *not* doing something. Every penalty in the ledger is seeded.

**`S-F2A` is included beyond the cluster map's listed screens** — same card, same moment, and showing standing on one but not the other would be a visible inconsistency for a one-line saving.

## Verification

Playwright at 1280×800 and 390×844, per screen. Overflow measured from child rects against the container, never `scrollHeight`. Assertions query rendered elements, never a screen's `textContent`.

Two verification gaps were found by review and closed by observation rather than argument: four of the seven emit points had only ever been exercised by code reading, and nothing proved the §6.6 count was derived rather than hardcoded. Both now driven through the interface — the count probe stubs the list to 2 and to 1 and requires the card to follow, including the singular "1 person".

## Known and deferred

- **Mobile `S-B2` shows no candidate above the fold.** A consequence of putting §6.6 full-card on `S-B2` rather than on `S-B4`. Worth a deliberate look.
- **Karma traits reuse `.tag pass`** — the same green as "Family fit", so a conduct claim and a compatibility claim look alike on `S-C3` / `S-F2A`.
- **Pre-existing, not from this branch:** the gallery's undo toast throws `TypeError: pendingUndo.backfilled` if its button is activated after the 6-second window. `pointer-events:none` blocks the mouse but not the keyboard, so it is latent and keyboard-reachable. The enclosing code is byte-identical to `main`. One line fixes it (`btn.onclick = null` in the timer and in `commitRemoval`) whenever that flow gets an owner.
