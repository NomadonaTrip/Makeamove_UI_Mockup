# prompt_2.md — Decision Log & Build Workflow

**Date:** 2026-08-24
**Source:** `prompt_2.md` (20 feature requests)
**Status:** Phases 1, 2 and 3 merged (PR #3, #4, #5). 33 decisions resolved —
the original 29 from Phase 0, plus §2.7 and §6.6 added 2026-08-24 from a design
conversation about why the pursued party ghosts, and two the cluster map had
never counted (see below).

**Rollback, 2026-08-26.** §2.7's async eligibility gate was removed from the
product. Async is open to every pursuer and every pursued regardless of profile
depth. The gate shipped in Phase 2 and came out of `index.html` in
`fix(async): remove the profile-depth eligibility gate`. §2.7 now records the
profile model that survived it; §2.1 and §6.6 were rewritten to match.

## How this document works

`prompt_2.md` contains 20 requests spanning seven independent subsystems. This
document is the gate: **no cluster gets a spec until every open decision in it
is answered and recorded here.** Each cluster then gets its own spec → plan →
build cycle, following the pattern already established in this repo.

All implementation lands in `index.html` only. `index_mobile_mockup.html` and
`index_mockup.html` are not kept in sync (see `CLAUDE.md`).

## Cluster map and build order

| Phase | Cluster | prompt_2 items | Screens touched | Decisions |
|-------|---------|----------------|-----------------|-----------|
| 1 | Question engine & authoring | 1, 2, 3, 7, 8, 9 | new deck-authoring screens, `S-C1`, `S-D2`, `S-F2`, `S-F3`, `S-E6`, `S-H3` | 7 ✅ |
| 2 | Async game restructure | 5, 6, 12, 20 | `S-F1`–`S-F4`, `S-G3`, new consent screen | 7 ✅ |
| 3 | Moves stacking (ladder) | 4 | `S-D2` | 1 ✅ |
| 4 | Scheduling core | 13, 15, ⭐ | `S-C1`, `S-C4`, `S-E3`, `S-E4`, new post-async call screen | 4 ✅ |
| 5 | Reviews & personality profile | 14, 17, 18 | `S-E2`, `S-E5`, `S-E6` | 4 ✅ |
| 6 | Karma points (badges deferred) | 10, 11 | `S-B2`, `S-C3`, cross-cutting + new karma screen | 6 ✅ |
| 7 | Safety & two-sided fairness | 16, 19 | `S-I1`, `S-H2`, onboarding, new SOS screen | 4 ✅ |

Build order is dependency-driven: the question engine is foundational, since the
async restructure, the ladder, and the confidence profile all read from it.

## Coverage check

Every one of the 20 requests in `prompt_2.md` maps to a resolved decision. This
table is the evidence for the gate.

| prompt_2 item | Resolved in |
|---|---|
| 1 · Questions should have options | 1.1 |
| 2 · Forced selection, MECE generation, LLM review, 3 replacements | 1.1, 1.3 |
| 3 · Custom questions, 5 each, sync both / async initiator | 1.6 |
| 4 · Other party sees moves stacking up | 3.1 |
| 5 · Async charges both before commencement | 2.3, 2.4 |
| 6 · Answers collected upfront, presented one at a time | 2.1, 2.2 |
| 7 · 10 system questions + optional 5 (testable) | 1.2 |
| 8 · Question variations, confidence, saved to profile | 1.4 |
| 9 · Mirrored questions for both parties | 1.5 |
| 10 · Points / badges / leaderboard | 6.1, 6.2, 6.5 |
| 11 · Karma penalties and rewards | 6.2, 6.3 |
| 12 · 72-hour async window | 2.5 |
| 13 · Scheduled or instant video call after async | 4.1, 4.2 |
| 14 · Written reason for declining a date | 5.1, 5.2 |
| 15 · Mutual time and venue close to both parties | 4.1, 4.3 |
| 16 · Alarm button broadcasting location | 7.1, 7.2 |
| 17 · More detail in the post-show personality profile | 5.3 |
| 18 · Compulsory "did you exchange numbers?" | 5.4 |
| 19 · Reporting fair to both sides | 7.3, 7.4 |
| 20 · Reasons collected when an async game is rejected | 2.6 |

One item is only partly resolved: item 10's badge system is deferred (6.5).
Everything else is fully decided.

Two decisions do not trace to a numbered request in `prompt_2.md` — §2.7 and
§6.6 came out of a design conversation on 2026-08-24 about why the pursued
party ghosts. Both were added after the cluster map was written and neither was
counted in it; the counts above are corrected, so the table now sums to 33.

Their content has since changed. §2.7 was an eligibility gate and is now the
profile model that outlived it (rollback of 2026-08-26); §6.6 surfaced that
gate and is now a plain invitation. They are still recorded here because they
still change what Phases 2 and 6 build.

## Conflicts with the current prototype

Recorded here so the specs resolve them deliberately rather than by accident.

**Item 5 contradicts existing async copy.** `S-F1`, `S-F2` and `S-F4` currently
state that the pursued party's 50% buy-in is soft-held at consent and captured
*only on a pass* — a fail releases it untouched. Item 5 asks that both parties
be charged before the game commences. Resolution belongs to Cluster 2.

**Items 1–2 vs. the quoted answer cards.** The dramatic core of `S-D2` and
`S-F3` is the free-text quoted answer the other party reacts to. Forcing every
question into MECE options removes it. Resolved in Cluster 1 below.

---

## Cluster 1 — Question engine & authoring ✅

Covers prompt_2 items 1, 2, 3, 7, 8, 9.

### 1.1 Answer format — options with an optional "why"

Every question presents MECE options and the responder must select one; the
selection is the sole scoring signal. Beneath it, an optional free-text line
(≤120 chars) in the responder's own voice.

The answer card in `S-D2` / `S-F3` renders the selected option as the primary
element, with the quoted line below it when present. Cards must therefore
render correctly in both states — with a voice line and bare.

Custom questions submitted by users get MECE options generated by the system,
per item 2.

### 1.2 Question count — build both models behind a toggle

Item 7 is flagged as a testable feature, so both models ship:

- **Model A** — exactly 10 questions; the user may replace up to 5 of them with
  their own. Fixed length for every player.
- **Model B** — 10 system questions plus up to 5 user-added, max 15. Length
  varies per player, and the two players may carry different totals.

A dev toggle switches between them, living on the shared dev-settings surface
described in Cluster 6.4 rather than on the authoring screen itself. The board,
ladder and 70% gate arithmetic must all handle a variable slot count for
Model B.

### 1.3 LLM appropriateness review — 3 replacements total, then system backfill

The LLM reviews submitted custom questions for appropriateness. The user gets
**3 replacement attempts across the whole submission**, not per question. When
the attempts are exhausted, the rejected slot is silently backfilled from the
system question bank so the game always starts with a complete set. No dead
end and no blocked user.

### 1.4 Confidence & the personal profile — extend S-E6

`S-E6` ("Who we heard you are") already shows weighted dimensions. It gains a
**"Confirmed about you"** block: each locked-in attribute or preference with a
confidence bar, the questions that triangulated it, and a "that's wrong"
correction control.

Item 8's variations mechanism — asking the same underlying thing several ways
to establish ground truth — is what feeds these confidence scores. A value is
written to the personal profile once confidence passes the high-confidence
threshold.

### 1.5 Mirrored async questions — revealed at the result

Item 9's mirroring is invisible during play. At `S-F4` a side-by-side **"where
you two actually landed"** block shows each mirrored pair: their answer, your
verdict, your answer, their verdict — with divergences flagged.

Keeping it out of play prevents a player reverse-engineering the pairing and
gaming their own answers mid-board, and turns the reciprocity into the payoff.

### 1.6 Authoring — a reusable question deck, seeded per game

New screens under the profile let a user build and edit a deck of up to 5
custom questions once, with LLM review happening there. When they invite
(`S-C1`) or pursue (`S-F2`), the deck is pre-attached with a chance to swap
questions for that specific match. This satisfies "submit those questions
ahead of the game" without forcing a rewrite per match.

Per item 3: in the **sync** game both participants may attach custom
questions; in the **async** game only the initiator may.

### 1.7 Attribution — custom questions are labelled

A custom question carries a visible badge in the show ("Samuel asked this")
distinguishing it from bank questions, so the responder knows this one matters
to the other party personally.

---

## Cluster 2 — Async game restructure ✅

Covers prompt_2 items 5, 6, 12, 20.

### 2.1 Async becomes symmetric

Items 6 and 9 together change the shape of the async game. Today `S-F3` is
asymmetric: only the pursuer reacts, scored against the pursued's weights.
Under item 6, **all preference and attribute data is collected from both
parties before the game begins**, and each party is then shown the other's
answers one at a time to Move or Stay on.

`S-F1`'s explanatory copy ("One-on-one, asymmetric", "Score = pursued's
weights · your Moves") is now wrong and must be rewritten.

**Why symmetry holds.** Revised 2026-08-26, superseding the 2026-08-24 version.

Symmetric play was originally justified on information grounds — both parties
need to be characterised for the match to mean anything. The 2026-08-24 revision
declared that justification obsolete, on the grounds that §2.7's gate already
guaranteed characterisation before a game began, and instructed that it not be
cited again. **The gate is gone, so that instruction is void and the
information argument is live again** — it is now the leading reason for
symmetry, not a retired one. Nothing guarantees the pursued is known before
play except the play itself.

Two further reasons stand on their own, and mattered even while the gate
existed:

- **Endorsement is an act, not a prediction.** A computed score predicts that
  the pursued would have said yes; it is not them saying yes. Consent at
  `S-F2A` is real but it is consent to *engage*, not endorsement of the
  outcome — and "I'll give this a go" is far easier to walk away from than ten
  discrete judgments you made about who this specific person is. Inferring
  agreement is a subtler version of the problem symmetry exists to solve.
- **It is the freshest preference data the system ever gets.** Reacting to a
  real person's real answers is higher-signal than abstract preference
  questions. Remove the pursued's board and an async game produces no new
  preference data from them at all — the profile stops sharpening on that
  path, which contradicts the progressive reconciliation in 2.7 that the whole
  profile model depends on.

There is also a product argument: if the pursued reacts to nothing, they have
paid to have a system compute whether they would like someone, which is a
worse product than playing a game with a person.

**Length is deliberately unchanged for now.** The 2026-08-24 case for shortening
the board rested on the gate — if a player arrived already characterised, fewer
reactions would still reach a confident verdict. Without the gate that headroom
is no longer assumed, and a thin-profile player arriving cold is an argument for
keeping the board's full length rather than trimming it. Either way the decision
waits for evidence: the Model A/B toggle from 1.2 can test length without a code
change, so it waits for real drop-off data.

### 2.2 Presentation — two boards, yours live, theirs filling in

You play your board slot by slot. A second, smaller board shows their verdicts
on *your* answers arriving asynchronously. This carries item 4's "moves
stacking up" into async as well, and makes the reciprocity visible during play
rather than only at the end.

### 2.3 Charging — both charged up front, no refund on a fail

Both parties are charged before the game commences; the pursued is charged at
the moment they consent. **A failed gate refunds nothing** — the money buys the
game, not the outcome. Since both parties do play, capture is consistent with
the prototype's existing "captured only when you play" narrative and with
`S-C5`'s commitment framing.

This resolves the conflict recorded above: `S-F1`, `S-F2` and `S-F4` copy about
soft-holds released on a fail must be rewritten. `S-G3`'s hold lifecycle needs
its async path updated to reflect capture-at-consent.

### 2.4 Consent gate — full disclosure before payment

Before consenting and paying, the pursued sees the **full profile of the
pursuer and the custom questions they'd be answering**.

This deliberately diverges from the sync rule in `S-C1` ("He'll only see your
photo and a few details until he does"). The divergence is justified: real
money now changes hands at consent, and asking someone to pay on a photo alone
is indefensible. A new screen is needed for the pursued's consent view — no
such screen exists today.

### 2.5 The 72-hour window

An async game must complete within 72 hours of starting. On expiry, **the side
that has not finished their board forfeits their charge, and the side that
completed receives a session credit back.** This mirrors the existing no-show
forfeit terms in `S-C5` and `S-D7`, keeping the rules consistent across sync
and async.

A visible countdown belongs on the async board and on any waiting state.

**This does not contradict 2.3.** They govern different events: a game that was
*played* and failed the gate refunds nothing (2.3), while a game that was
*never finished* forfeits the staller and makes the other side whole (2.5). The
distinction is whether the product was delivered, and the copy must make that
distinction plainly.

**Slot count.** `S-F3`'s board is hardcoded to 5 slots. Under Cluster 1.2's
Model B the question count varies 10–15 and the two players may carry different
totals, so the async board — like the ladder in 3.1 — must render a variable
number of slots.

### 2.6 Rejection reasons — MECE options plus optional free text

When an async game is rejected, the rejector picks from MECE options and may
add an optional free-text line. Structured options are what the pre-matching
system in item 20 can actually consume; the free line catches what the options
miss. Consistent with the answer pattern in Cluster 1.1.

Collected reasons are written to the rejector's profile. Note the deliberate
contrast with item 14 (Cluster 5), where a date decline is free-text *only* —
there the friction is the point, because no algorithm consumes it.

### 2.7 Profile depth — progressive, never a gate

Added 2026-08-24 as an eligibility gate. **The gate was rolled back on
2026-08-26**; this section now records the profile model that outlived it.

**What the gate was, and why it is recorded rather than deleted.** A user could
neither initiate an async pursuit nor be pursued unless the system knew enough
about them: 3 attributes at `Confirmed` (≥70% confidence) AND 12 questions
answered, checked at invite and again at consent, then frozen for the game's
duration. It shipped in Phase 2 and was enforced at three points — `S-F1` on
entry, `S-F2` on send, and `S-F2A` on the pursued's consent. All three are open
now, and `MMQ.eligibility()`, the `ELIG_*` thresholds and `MMASYNC`'s freeze
helpers are gone from `index.html`.

**The rule now.** Async is open to every pursuer and every pursued, whatever the
system knows about them. Nothing about profile depth blocks entry to any mode.

The concern the gate existed to answer — that a 70% match resting on a
thinly-known person is exactly the case that produces ghosting — is now carried
by karma instead: §6.2 scores responsiveness, §6.3 makes low karma surface you
less often. A consequence rather than a barrier, which is the shape this
product prefers everywhere else.

Four provisions of the original section survive unchanged, because each is true
independently of the gate and each is built in the merged prototype.

**Answers accumulate across modes.** Every question answered during a sync show
counts toward confirming an attribute, and confirmation draws on sync-game
responses *and* the user profile (`S-P1`–`S-P5`) together. `S-D2` writes
`mm_answers` through `recordAnswer` when a turn is locked.

**The profile is progressively reconciled.** Confirmation is not a one-time
capture. As users play more and discover more about themselves, answers
accumulate, confidence moves, and the correction control on `S-E6` (1.4) lets
them overrule an inference that no longer fits. A user's picture is expected
to sharpen and occasionally change direction over time, not to be fixed at
onboarding.

**The question bank carries 20 questions, not 15.** Confidence is damped by how
few variations of a probe exist, so an attribute needs three differently-worded
questions before consistent answers can clear 70%. Verified against the live
bank on 2026-08-24: at 15 questions only `wants-children` qualified, and nine
probes capped at 67 or 33. The expansion to 20 gives `core-value`, `weekend`
and `money-model` a third variation each and `closeness` two more — five
confirmable attributes. It was a prerequisite of the gate; it is now simply
what makes `S-E6`'s confidence meters mean anything.

**Consequence for the async result screen.** A user can legitimately reach a
game with no recorded answer to some questions — more of them now than under
the gate, which is the point. The mirrored-pairs reveal (1.5) must therefore
render an explicit **"not answered yet"** state for those rows rather than
falling back to a default option and implying an answer the user never gave.
Those blanks are honest, and they double as a visible pull toward answering
more.

**What lapsed with the gate.** The Phase-2 note that the prototype could not
demonstrate "playing the flagship mode is how you unlock async" is moot — there
is nothing to unlock. The observation underneath it still holds and is still
worth fixing: a game draws a fixed ten questions from the front of the bank, so
replaying the sync show re-asks the same set and `recordAnswer`'s dedupe
saturates the answered count. A question-selection strategy that asks what we
do not yet know would resolve it, and still pulls against the mirrored-pairs
reveal (1.5), which needs a mix of answered and unanswered rows to be worth
showing at all. That trade-off remains deferred — it is now a question about
profile quality rather than about access.

The dev-settings profile-depth toggle survives the rollback. It no longer demos
a gate; it drives `S-E6`'s confidence meters and how much of `S-F4`'s reveal
reads "not answered yet".

---

## Cluster 3 — Moves stacking (ladder) ✅

Covers prompt_2 item 4.

### 3.1 A vertical Millionaire ladder replaces the token strip

`S-D2`'s current `.base .tokens` horizontal strip is replaced by a vertical
ladder rail. Rungs light up as the other party's verdicts land, each showing
the dimension and the points won (or an ✕ and a dash for a Stay).

- The **70% gate is drawn as a marked rung**, so the player can see how far
  they are from clearing it.
- On a phone, a **~5-rung window** scrolls with play, anchored on the current
  slot.
- At ≥768px the full ladder sits as a right rail; add `S-D2` handling to the
  desktop reflow accordingly (it is already in the `IMMERSIVE` set).
- Rung count is driven by the question count, which varies 10–15 under
  Cluster 1.2's Model B. The ladder must not assume a fixed length.

The two score heads at the top of `S-D2` stay as they are — they carry both
players' percentages, so the ladder only needs to render the other party's
verdicts on you.

## Cluster 4 — Scheduling core ✅

Covers prompt_2 items 13, 15 and the ⭐ note.

### 4.1 One reusable scheduler — top-3 proposed, full grid behind "see all"

A single scheduling component is built once and reused at every scheduling
moment in the journey: the sync invite (`S-C1`/`S-C4`), the post-async video
call (new), and the physical date (`S-E3`/`S-E4`).

Default view offers **three system-computed best mutual times**, one tap to
confirm. A "see all times" link opens the full paintable week grid for users
who want control. Easy by default, robust on demand — which is what the ⭐
asks for.

This supersedes the bespoke propose-5-slots UI in `S-C1` and the mutual
availability view in `S-C4`; both are refactored onto the shared component.

### 4.2 Instant call — ping, 60-second ring, fall back to scheduling

Item 13's instant-call option is a "call now" action that rings the other
party for 60 seconds. Answered, both go straight into the call; unanswered, the
caller lands in the scheduler with no awkward dead end.

Chosen over presence-gating because it needs no online/offline indicator —
which on a dating platform would leak when each person is using the app.

### 4.3 Venue proximity — system-computed midpoint, no locations shared

For item 15, the system computes a midpoint district from both parties' stored
locations and suggests venues there. **Neither party learns where the other
lives or works.** No new user input is required, and nothing personal is
disclosed before a first in-person meeting.

`S-E3` currently hardcodes "near Canary Wharf"; it becomes a midpoint result.

### 4.4 Timezones — always show both

Every proposed slot renders in both parties' zones, e.g. "Sat 8:00 pm GMT ·
9:00 pm WAT". Onboarding branches on Nigerian and other African nationalities
while candidates live in the UK, so cross-timezone pairs are the norm in this
product, not an edge case. Always showing both removes a whole class of missed
sessions and stops anyone proposing a 3am slot unaware.

---

## Cluster 5 — Reviews & personality profile ✅

Covers prompt_2 items 14, 17, 18.

### 5.1 Date decline — written reason, shared verbatim

Item 14's forced written reason is **shared verbatim with the declined
party**. No options are offered; the friction is the point.

The decision was taken with the trade-off understood and explicitly accepted:
because the text reaches a real person, some decliners will write the polite
lie rather than the true reason, which weakens it as a calibration signal. The
honesty of the exchange was judged worth more than the cleanliness of the data.

Placement: `S-E2` currently offers "Not quite — end politely (credit kept)" as
a bare link to `S-B2`. That link becomes a blocking gate — no return to
matches until a reason is written.

### 5.2 Friction gate — minimum length plus LLM substance check

A reason must clear roughly 40 characters **and** an LLM check that it says
something real, rather than "nothing really" or keyboard mash. Reuses the
review mechanism built in Cluster 1.3.

**Consequence of 5.1:** since the text now reaches a real person, this check
does double duty and must screen for abuse as well as for mush. A decliner who
writes something cruel is asked to rewrite, by the same mechanism. Recorded as
a design assumption, not an answered question — flag it if you want it to work
differently.

### 5.3 Personality profile — stated vs. revealed, with an evidence deep dive

Item 17's added detail is **the gap between what you claimed and what you
played**: your self-reported answers from `S-P1`–`S-P5` set against what your
Moves actually revealed, with divergences called out. `S-E6` already gestures
at this ("We noticed you Moved on partners who were open about money
struggles").

Beneath it, a **"see the full evidence trail"** deep dive expands to every
question, your answer, their verdict, and what the system inferred from it.
Summary by default, receipts on demand.

This composes with Cluster 1.4's confidence meters on the same screen, and
gives the "that's wrong" correction control something concrete to correct.

### 5.4 Number exchange — compulsory yes/no, then options plus optional text

`S-E5`'s post-date review gains a compulsory "did you exchange numbers?"
question that blocks submission. A "no" reveals MECE options with an optional
free-text line, matching the answer pattern from Cluster 1.1.

Note the deliberate asymmetry with 5.1: the date decline is free-text only
because the friction is its purpose, while this one is structured because
calibration consumes it.

## Cluster 6 — Karma points ✅ (badges deferred)

Covers prompt_2 items 10, 11.

### 6.1 No leaderboard

Item 10 mentions a leaderboard; the decision is **not to build one**, in any
form — not public, not anonymised, not cohort-relative.

What carries the incentive instead is visible good and bad karma. The intent
is to reward treating people well *even when you don't want to match with
them*: don't leave an async partner hanging indefinitely, don't abandon a game
mid-board.

### 6.2 Rewarded behaviours

All four earn karma points:

- **Responsiveness** — answering async questions and invites promptly. This is
  item 11's own example of a penalised behaviour, so speed is its mirror.
- **Showing up** — attending scheduled sessions, calls and dates. Reinforces
  the forfeit terms in `S-C5`, `S-D7` and Cluster 2.5.
- **Profile & question completeness** — `S-P1`–`S-P5`, authoring custom
  questions, correcting wrong inferences on `S-E6`.
- **Honest feedback & clean conduct** — substantive post-call and post-date
  reviews, and never being upheld in a report. The conduct half must only be
  scored **after** an operator ruling, so a reported user does not lose points
  before anyone has judged the report.

Referrals are deliberately excluded — they reward marketing the app, not being
a good match.

### 6.3 Karma consequence — match visibility

Low karma means surfacing less often in other users' candidate galleries; high
karma surfaces you more. A real consequence that steers the platform toward
reliable people without ever announcing to anyone that they have been
penalised. `S-B2`'s ordering reads karma.

### 6.4 Karma display — A/B both directions

Undecided between behavioural traits and a numeric score, so the mockup
**builds both behind a toggle** and the optimal one is chosen from testing:

- **Direction A** — behavioural traits: candidate cards show "Responds within a
  day", "Always shows up". Karma translated into what the other person actually
  wants to know. Fits the existing tag styling on `S-B2` and `S-C3`.
- **Direction B** — numeric score: an explicit karma number on profiles.

Same toggle pattern as Cluster 1.2's question-count models; the two toggles
should share one dev-settings surface rather than each inventing their own.

### 6.6 "People wanted to reach you" — the enrichment invitation

Added 2026-08-24 to complete §2.7's gate. **Rewritten 2026-08-26**, when the
gate was rolled back and took the original framing with it.

The original message was **"N people wanted to pursue you this month. Finish
your profile to see who."** Its second half no longer holds: nobody is excluded
from async, so there is nothing gated behind finishing a profile and the
sentence would be a lie.

The first half survives, and it was always the valuable half. **"3 people
pursued you this month"** is true, specific and flattering, and it is still
likely the single most motivating message in the product. What changes is the
ask that follows it. It is no longer about access — it is about match quality:
the more the system knows, the better it can tell you which of those people is
actually a fit, and the better your own matches get. That is an honest offer
now that it cannot be an ultimatum.

Both hard constraints carry over unchanged:

- **The number must be real.** An inflated or vague count turns this into a
  dark pattern. If nobody tried to pursue them, the message does not appear.
- **It must lead somewhere actionable** — straight into the specific questions
  that would confirm their next attribute, not a generic "complete your
  profile" nag.

A third is added by the rollback:

- **It must not imply a gate.** No "unlock", no "eligible", no progress bar to
  a threshold — those all describe a rule that no longer exists. The invitation
  offers a better product, never restored access.

Still Phase 6 rather than Phase 2: it is a reward mechanic and should be
consistent with the karma surfacing decided in 6.4.

### 6.5 Deferred — badges

A badge system is wanted but its specifics are not yet settled, so **badges are
out of scope for Phase 6** and no badge spec is written. Karma points ship
first. Badges get their own decision round and their own phase once the
specifics exist.

---

## Cluster 7 — Safety & two-sided fairness ✅

Covers prompt_2 items 16, 19.

### 7.1 SOS recipients — operator plus nominated emergency contacts

Item 16's alarm broadcasts exact location to the **operator console** (`S-H1`/
`S-H2`, already audit-logged) **and to emergency contacts** the user nominated
during onboarding, with a prominent "call 999" action on the same screen.

Someone who knows the user and someone accountable both receive it. This adds
an emergency-contact step to onboarding, which does not exist today.

### 7.2 SOS trigger — visible button plus discreet gesture

Two paths to the same alarm:

- A **clear alarm control** in the active date-window UI, for when it can be
  used openly — discoverable, and it teaches the feature exists.
- A **discreet silent trigger** — a rapid tap pattern anywhere on screen —
  that fires with **no visual change at all**, for when the other person is
  watching.

Both pass through a **3-second cancel window** to kill false alarms. The
discreet path must never render a countdown, a toast, or any other visible
confirmation.

### 7.3 Evidence — structured game record plus both parties' accounts

For item 19, the operator receives both sides' answers, verdicts, response
timings and karma history, **plus a written statement from each party**.

Genuinely two-sided, and it requires recording nothing private — call
recording was considered and rejected as disproportionate. `S-H2` already
reviews reports; this gives it real material to review.

### 7.4 Right of reply before a ruling

The reported party is **notified that a report exists and may submit their
account before the operator rules**. This is what "fair to both sides" means
in practice.

The known cost is handled explicitly: telling someone they have been reported
can provoke retaliation in a harassment case, so the operator console needs a
**notification-suppression option** for safety cases. `S-H2` must carry that
control.

---

## Deferred

| Item | Why deferred | Unblocks when |
|------|--------------|---------------|
| Badge system (part of item 10) | Specifics not yet settled | A decision round on badge taxonomy and award rules |

## Phase execution protocol

Every phase runs the same loop, matching the pattern already established in
this repo:

1. Write the phase spec to `docs/superpowers/specs/YYYY-MM-DD-<cluster>-design.md`,
   drawing its decisions from this log.
2. Invoke the writing-plans skill to produce
   `docs/superpowers/plans/YYYY-MM-DD-<cluster>.md`.
3. Build in `index.html` only, on a feature branch.
4. Verify visually with the Playwright MCP browser tools.
5. Commit; open a PR.

Constraints that hold across every phase, from `CLAUDE.md`:

- Single self-contained file — no external CSS/JS, no libraries.
- Reuse the `:root` custom properties and the shared `.btn` / `.card` / `.row`
  / `.chip` / `.opt` components rather than inventing new styles.
- New list, gallery or full-bleed screens must be added to the `FEED` or
  `IMMERSIVE` sets in `classify()`, or they default to a centred form sheet on
  desktop.
- `index_mobile_mockup.html` and `index_mockup.html` are not kept in sync.
