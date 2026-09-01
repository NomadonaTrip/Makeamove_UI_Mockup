Modify the flow so that when the user completes the Onfido verification, they're taken immediately to the Matching page. On clicking the "Matches Ready" button, they are shown the people they're prematched with.

There should be a button or some other way by which they may readily return to their profiles to enrich it. It is important that they don't feel that they have to complete their profiles to get matched with potential spouses.

---

# PR body — Phase 3 · Moves stacking, the Millionaire ladder

*(Not a request. **Spent** — merged as PR #5. Safe to delete.)*

Implements **Cluster 3 / decision 3.1** (prompt_2 item 4 — "other party sees moves stacking up").

Spec: `docs/superpowers/specs/2026-08-25-moves-ladder-design.md`
Plan: `docs/superpowers/plans/2026-08-25-moves-ladder.md`

## What changed

`S-D2` reported the other party's verdicts as a strip of 22px tokens — no dimension, no points, no relationship to the 70% gate. That strip is now a vertical points ladder: rungs light as verdicts land, each carrying its dimension and what it was worth, with the gate drawn as a line across the rail.

- **`MMLADDER`** — a new component with a pure `model()` (all the arithmetic, assertable without a DOM) and a `mount()` rendering either a vertical `rail` or a horizontal `compact` strip.
- **`S-D2`** — `.base` holds the rail. One rung per *verdict on you*, so `ceil(questions/2)` rungs, since the arena alternates turns and only your answering turns build your score.
- **`S-D2` at ≥768px** — reflowed to a CSS grid, stage left, full ladder as a right rail. No markup change, per the project's reflow contract.
- **`S-F3`** — the `.mini` token strip became the compact variant. The flip board is deliberately untouched: it tracks Samuel's score, while the ladder tracks yours.

## Decisions worth reviewing

**The gate threshold is `ceil(0.695 × total)`, not `0.70 × total`.** The screens pass on `Math.round(won/total*100) >= 70`, which is true from 0.695 upward. At a 105-point total the naive formula gives 74, so a player on 73 would read "1 to go" while the score head beside it already showed 70%. Verified equivalent across every reachable configuration.

**The gate rung is not a fixed index.** Dimensions carry unequal weight, so the line sits where cumulative available points first cross the threshold — rung 3 for the default set (120 pts, gate 84), rung 4 once a custom deck displaces bank questions (105 pts, gate 73).

**The compact strip fills forward while the rail fills bottom-to-top.** Reversal exists to keep a vertical *climb* readable in the order it is seen; a horizontal strip has no climb, and reversing it made it read against the flip board's 1..N numbering directly below. Divider boundary proven identical in both directions.

**An unreachable gate desaturates and says nothing.** No "you can no longer pass" message — the foot readout already states the position, and `S-D6` is the screen whose job that is.

## Verification

Browser-driven throughout (no test suite in this repo by design). Every task carried live assertions; a whole-branch review measured the rendered screens at 390×667, 390×740, 390×844, 800×720, 900×700, 1024×700, 1440×900, under both question-count models.

Two defects were caught late and fixed:

- the compact strip inherited the rail's reversed order and filled right-to-left with no rung numbers to say so;
- **`S-D2` overflowed its stage on phones shorter than ~745px** — a regression against `main` on the flagship screen. The rail's 196px window left the stage 137px for 176px of content, printing the answer card over the rail header and the chip over the score heads. Fixed with two short-viewport tiers (3 rungs below 800px tall, 2 below 740px); spill now 0 at 390×667 with every rung still reachable.

## Known, deliberately not fixed

- `S-F3` still overlaps at 390×667 (23px Model A / 38px Model B). Pre-existing on `main`; improved at Model A, roughly unchanged at Model B. Clearing it needs that screen's whole column re-budgeted — a design change, not a regression fix.
- Below ~658px viewport height (iPhone-5 era) the arena spills ~6px.
- `#S-F3 .stagef{min-height:190px}` is dead, overridden by `min-height:0`. Pre-existing on `main`; it is the root cause of the item above.

---

# PR body — Remove the async profile-depth gate

*(Not a request. Parked here for copying into the pull request for `fix/ungate-async`:
https://github.com/NomadonaTrip/Makeamove_UI_Mockup/pull/new/fix/ungate-async)*

Rolls back **§2.7**, the async eligibility gate. Async is now open to every pursuer and every pursued, regardless of how much the system knows about them. Nothing about profile depth blocks entry to any mode.

This restores a standing product principle that predates the gate — from the top of this very file: *"It is important that they don't feel that they have to complete their profiles to get matched with potential spouses."* §2.7 had made async the exception to that rule; it no longer is.

## The gate was enforced at three points, not one

Worth calling out, because it is easy to assume this was a single check:

| Screen | Check |
|---|---|
| `S-F1` | entry to async |
| `S-F2` | the send action (reachable directly by deep link, so it gated for itself) |
| `S-F2A` | **the pursued's consent-and-pay screen** |

The third is the one that matters most. A thin-profile *pursued* party — someone who never opted into anything, who was simply invited — hit "Before you can accept, we need to know you a little better" and could not accept. All three now render their CTA unconditionally.

## Retired with it

`MMQ.eligibility()`, `MMQ.ELIG_CONFIRMED`, `MMQ.ELIG_ANSWERED`, `MMASYNC.lockEligibility()`, `MMASYNC.lockedEligibility()`, `MMASYNC.clearEligibility()` and their two call sites, and the `MMELIG.paint()` repaint hook. 140 lines out, 29 in.

## Kept, deliberately

- **The dev profile-depth toggle.** Not dead — it drives `S-E6`'s confidence meters and how much of `S-F4`'s mirrored-pairs reveal reads "not answered yet". Relabelled so it no longer claims to demo a gate.
- **`S-F2A`'s "you've already answered 5 of these, only 5 are new."** More useful now, not less: a cold profile legitimately reaches that screen and will honestly see "all 10 are new to you."
- **"Unlocked at 200 users"** and the operator console's async-unlock threshold. Those are platform-scale switches, unrelated to profile richness. Only `S-F1`'s `data-title` ("Async unlock entry" → "Async entry") referred to the removed gate.

## Decision log

**§2.7 is repurposed, not deleted.** Four of its provisions are true independently of the gate and are built in the merged prototype — answers accumulating across modes via `recordAnswer`, progressive reconciliation through `S-E6`'s correction control, the 20-question bank that makes confidence meaningful at all, and `S-F4`'s "not answered yet" state. Striking the section would have orphaned all four. It is now *"Profile depth — progressive, never a gate"*, carrying those plus a record of what the gate was and why it went. The ghosting concern it existed to answer is now carried by karma (§6.2 scores responsiveness, §6.3 makes low karma surface you less often) — a consequence rather than a barrier.

**§2.1's symmetry argument inverted.** It had declared the information-grounds justification for symmetric async *"obsolete… should not be cited again"* — but only *because* the gate guaranteed characterisation before play. With no gate, nothing guarantees the pursued is known except the play itself, so that argument is live again and is now the leading reason for symmetry. The "length is deliberately unchanged" note rested on the same premise and is rewritten: its case for a shorter board assumed players arrived pre-characterised.

**§6.6 keeps its count, loses its access claim.** "N people pursued you this month" is still true and still the most motivating line in the product; the ask is now match quality rather than restored access. A third hard constraint is added — it must not imply a gate that no longer exists (no "unlock", no "eligible", no progress bar to a threshold).

**Cluster-map counts corrected.** Pre-existing error: the counts were never bumped when §2.7 and §6.6 were added. Phase 2 showed 6 against 7 decisions, Phase 6 showed 5 against 6 — the table summed to 31 while the document held 33.

The Phase 2 plan gets a rollback banner at its head. Task 9 is left intact as the accurate record of what was built at the time.

## Verification

Browser-driven at 390×844 with the thinnest profile (8 answers — below the old 12-answer floor, so previously blocked at all three points):

- all three CTAs enabled, zero disabled buttons, no blocker wording;
- every eligibility symbol reports `undefined`, no stale `mm_async_elig` key;
- 69 sections intact after tag surgery on `S-F1`, whose gate div and closing `</div>` were adjacent;
- async board plays through to `S-F4` (`{you: 94, them: 100}`);
- the decline path and the dev depth toggle both survive their removed calls;
- zero console errors.
