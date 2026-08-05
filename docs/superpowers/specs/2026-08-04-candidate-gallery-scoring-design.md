# Candidate Gallery — Portraits, Fit Scoring & Paid Restore

**Date:** 2026-08-04
**Status:** Approved
**Target file:** `index.html` (single self-contained page)
**Screens touched:** `S-B2` (candidate gallery), `S-B3` (candidate profile)

## Summary

The candidate gallery currently shows three hardcoded rows with gradient
initial-letter avatars, a static "Family fit / AA" tag pair, and no way to act
on a candidate other than opening them. This design adds four things:

1. **Placeholder portraits** for every candidate and for the user.
2. **A pre-match fit score** per candidate, decomposed into weighted bands,
   each band justified by bulleted observations traceable to supplied data.
3. **Remove → backfill → paid restore.** Removing a candidate pulls the
   next-best person off the bench; restoring a removed candidate costs £3.
4. **A gap-driven enrichment prompt** that names the specific profile sections
   whose absence is suppressing scores, and deep-links to them.

This is a prototype change confined to one file. No build step, no framework,
no new runtime files.

## Decisions (locked)

- **Portraits:** remote URLs, hand-picked from Unsplash rather than a random
  avatar service. Rationale in [Portraits](#portraits).
- **Score format:** score + descriptive label + meter, per band and overall.
  Never a bare number.
- **Rationale:** bulleted observations, never prose summary. Every bullet cites
  the data that produced it and the source that supplied it.
- **Restore flow:** free undo window, then a "Removed" drawer on `S-B2` with a
  £3 confirm sheet. No separate paywall screen.
- **Enrichment prompt:** invitational, never gating, placed *after* the
  candidate list.

---

## Scoring model

### The gate

Dealbreakers are a pass/fail gate, not a scored band. A candidate who fails the
gate never appears in the pool at all, so the gate contributes no points.

| Gate check | Source |
|---|---|
| Genotype safety (AA-only dealbreaker) | `S-Q3B` |
| Marital eligibility (married ⇒ ineligible) | `S-Q4` |
| Extended dealbreakers (blended family, non-smoker, location) | `S-A9` |

The gate is displayed on `S-B3` as cleared, so the user can see it was applied.

### The bands

Five weighted bands sum to 100.

| Band | Weight | Data sources |
|---|---|---|
| Family & intent | 30 | `S-Q4` relationship status, `S-P5` Marriage & family |
| Geography | 20 | `S-Q6` country, `S-Q7` city of residence |
| Life stage | 15 | `S-Q2` age, `S-Q5` vocational status, `S-P1` education/income |
| Personality & lifestyle | 20 | `S-P3` |
| Love & intimacy | 15 | `S-P4` |

`S-P2` Partner preferences is not a band. It supplies the user's *target*
values, which every band compares the candidate against.

**Geography is a rule of thumb, not a preference match.** Absent a stated
travel radius, proximity scores higher on the assumption that partners
generally prefer to be geographically closer. Same city > same metro >
same country > different country.

### Sparse data and cold start

Overall fit is the weighted mean **across bands that have data on both sides**,
renormalised to that subset. A band with no data is not scored as zero — it is
excluded and reported as blank. Scoring a missing band as zero would punish
candidates for silence rather than for mismatch.

- **Confidence** = sum of the weights of scored bands ÷ 100.
  Displayed as High (≥70%), Medium (40–69%), Low (<40%).
- When a candidate's confidence is below 40%, their placement is **partly
  random** rather than ranked, and the UI says so explicitly. This is the
  honest description of cold-start behaviour: with only onboarding data
  available, there is not enough signal to rank.
- Demo scores sit deliberately in the **48–74** range. `S-B2` carries a line
  explaining the pool is young and that scores rise as more people join,
  enrich their profiles, and play shows.

### Display thresholds

| Overall / band score | Label |
|---|---|
| ≥ 70 | Strong |
| 50–69 | Moderate |
| < 50 | Weak |
| no data | — · Not enough data |

---

## Rationale format

Each band renders its score, then **1–4 bullets**. A bullet is an observed fact
pair — the user's datum, the candidate's datum, and the source — never an
inferred summary. No bullet may appear without a backing field.

Three bullet states:

- **✓ observed match** — both sides supplied data and they align.
- **✗ observed mismatch** — both sides supplied data and they diverge. Shown,
  not hidden. A mismatch on a non-dealbreaker is legitimate signal, and a
  candidate scoring 61% needs visible reasons why it is not higher, or the
  number reads as arbitrary.
- **○ no data** — one or both sides left the field blank.

The `!` random-surfacing advisory is **candidate-level, not band-level**. It
appears once, at the top of the breakdown, only when the candidate's overall
confidence is below 40%. A single blank band does not trigger it — a candidate
can have one band blank and still be confidently ranked on the rest.

Worked example — David, whose Personality & lifestyle and Love & intimacy bands
are both blank on the *user's* side. Weight 35 of 100 is unscored, leaving
confidence at 65%, so no advisory applies (the threshold is 40%):

```
FAMILY & INTENT                    74% · Strong
  ✓ You set "must welcome a blended family" as a
    dealbreaker · he answered "open to children
    from a previous marriage"        [onboarding]
  ✓ You have 2 children · he has 2 children
                                     [onboarding]
  ✓ You want marriage within 2 years · he answered
    "within 2 years"                    [profile]
  ✗ You are open to one more child · he answered
    "no more children"                  [profile]

GEOGRAPHY                          81% · Strong
  ✓ You live in London · he lives in London — same
    city                             [onboarding]
  ○ Neither of you has given a travel radius
                                    [no data]

PERSONALITY & LIFESTYLE       — · Not enough data
  ○ He has not completed Personality & lifestyle
                                    [no data]
  ○ He has not played a show, so no gameplay
    signal exists                   [no data]
```

Contrast — a bench candidate at 28% confidence, where only onboarding-derived
bands scored. The advisory sits above the bands:

```
! Surfaced partly at random. Only 2 of 5 bands
  could be scored, so he is not meaningfully
  ranked against your other candidates.
```

Source tags are `[onboarding]`, `[profile]`, `[gameplay]`, `[no data]`.

---

## Screen: `S-B2` candidate gallery

### User portrait

The top bar gains the user's own placeholder portrait beside the existing
`Credits ›` link, at `.av s28`.

### Candidate card

Each of the three live rows keeps its existing `.row` shell and 96×120 avatar
block, with the initial letter replaced by a portrait. The text column gains:

- **`FIT 76% · Strong`** with a thin meter.
- **The two most decisive bullets** — the highest-weight observed match and,
  where one exists, the sharpest observed mismatch. Full bullets for all bands
  would make a three-candidate list a wall of text.
- **`See all 13 reasons ›`** — the count is the total number of bullets across
  all bands, and it opens `S-B3`.
- A **✕ remove** control.

```
┌──────┬────────────────────────────────┐
│      │ David, 41                      │
│ [img]│ Deputy head · 2 kids · London  │
│      │ FIT 76% · Strong  ▓▓▓▓▓▓▓▓░░░  │
│      │ ✓ Welcomes a blended family    │
│      │ ✗ Wants no more children;      │
│      │   you're open to one           │
│      │ See all 13 reasons ›           │
└──────┴────────────────────────────────┘
```

Low-confidence candidates additionally carry a `Low confidence` tag.

### Removed drawer

A collapsible **Removed (n)** section sits below the live list, above the
enrichment prompt. Each entry shows the portrait, name, retained fit score, and
a **Restore · £3** action. The drawer is hidden entirely when empty.

---

## Remove → backfill → restore

1. **Remove.** The row animates out. A toast appears: *"Marcus removed. UNDO"*,
   live for **6 seconds**. Undo within the window is free and restores the row
   in place — the user should never be charged for a misclick.
2. **Backfill.** On removal, the **highest-scoring bench candidate** is inserted
   into the live list with a brief highlight. Bench candidates always score
   lower than those already shown, which makes "next-best" legible rather than
   magical.
3. **Drawer.** Once the undo window expires, the removed candidate moves into
   the Removed drawer.
4. **Restore.** `Restore · £3` opens a confirm sheet naming the candidate and
   the price. Confirming re-inserts them into the live list and returns the
   **weakest live candidate** to the bench, holding the list at three.

The price lives in a single constant `RESTORE_PRICE = 3` (GBP) so it can be
changed in one line after the competitive cross-check.

The restore charge is **independent of the session-credit model** in group G.
Credits buy show sessions at £19; restore is a £3 micro-transaction. The confirm
sheet must not imply a credit is being spent.

---

## Profile-gap prompt (`S-B2`)

Replaces the existing generic "Enrich your profile" row. Placed **after** the
candidate list — the matches come first.

It distinguishes the two causes of a blank band, and only one is actionable:

- **Your gap** — your unanswered `S-P3` suppresses that band for *every*
  candidate simultaneously. Actionable, and worth surfacing.
- **Their gap** — the candidate's own sparse data. Not fixable by the user, and
  the prompt must not imply otherwise. Promising a better score that only the
  candidate's completion can deliver is a lie the UI gets caught in.

```
┌────────────────────────────────────────┐
│ ✦ 2 bands are blank on your side       │
│                                        │
│ Personality & lifestyle and Love &     │
│ intimacy are unscored for all 3 of     │
│ your candidates — because you haven't  │
│ answered them yet.                     │
│                                        │
│ ~8 questions · about 3 minutes         │
│                                        │
│ [ Answer Personality & lifestyle › ]   │
│   Love & intimacy ›                    │
│                                        │
│ Optional, always. You're already       │
│ matched — this only sharpens who we    │
│ put in front of you next.              │
└────────────────────────────────────────┘
```

Deep-links go straight to the specific screen (`S-P3`, `S-P4`), not to `S-A9`
for the user to hunt through. When no gaps remain the prompt collapses to a
quiet confirmation — *"All 5 bands scored on your side."* — rather than
vanishing, so completion is visibly rewarded.

**Second placement.** On `S-B3`, a band blank because of *the user's* missing
data carries an inline link: *"Blank because you haven't answered Love &
intimacy. Answer it ›"*. This is the highest-intent moment — they are looking
directly at the gap. Bands blank on the candidate's side get the honest
statement and no link.

This must not contradict the existing product principle stated on `S-A9` and in
`updates.md`: users must never feel they have to complete their profile to be
matched. Copy stays invitational throughout.

---

## Screen: `S-B3` candidate profile

The existing "Why you matched" card is replaced by the full breakdown:

- **Data-completeness meter** at the top — *"Scored on 3 of 5 bands · he's
  completed 40% of his profile"* — so a low score is attributable to missing
  input rather than to a broken engine.
- **The random-surfacing advisory**, directly beneath the meter, when overall
  confidence is below 40%.
- **The dealbreaker gate**, shown as cleared.
- **All five bands**, each with score, label, meter, and its full bullet list.
- Sparse bands rendered greyed with their `○` bullets.

The hero image becomes the candidate's portrait. Existing invite CTA and
back-link are unchanged.

---

## Data model

A `CANDIDATES` array of 8 people — 3 live, 5 bench — inline in `S-B2`,
mirroring how The Show keeps its turn data inline in `S-D2`.

Each candidate carries: `id`, `name`, `age`, `blurb`, `city`, `photo`, and a
`bands` map. Each band holds `score` (or `null` when unscored) and a `bullets`
array of `{state, text, source}`.

Demo pool, scores descending. Geography degrades down the bench, which
demonstrates the proximity rule of thumb:

| # | Name | Detail | Fit | Confidence |
|---|---|---|---|---|
| 1 | David, 41 | Deputy head · 2 kids · London | 76 | Medium |
| 2 | Samuel, 44 | Architect · wants kids · London | 68 | Medium |
| 3 | Marcus, 39 | GP · blended-family open · Reading | 61 | Medium |
| 4 | Tunde, 43 | Structural engineer · 1 kid · Croydon | 58 | Low |
| 5 | Emeka, 40 | Pharmacist · no kids · Luton | 55 | Low |
| 6 | Kwame, 45 | Secondary teacher · 3 kids · Birmingham | 52 | Low |
| 7 | Ifeanyi, 38 | Data analyst · no kids · Manchester | 49 | Low |
| 8 | Bode, 42 | Logistics manager · 2 kids · Dublin | 48 | Low |

Rows 1–3 are live; 4–8 are the bench. Candidates 4–8 have Low confidence, so
they carry the random-surfacing advisory — which is the intended cold-start
demonstration.

**State** persists in `sessionStorage` under `mm_cand`, following the existing
`mm_py` / `mm_pt` convention, so removals survive navigation to `S-B3` and back.

### Portraits

Hand-picked Unsplash direct URLs (`images.unsplash.com/photo-…`), sized with
`?w=400&q=70&fm=jpg&crop=faces&fit=crop`.

Not a random avatar service. randomuser.me and pravatar return faces from an
opaque, predominantly white pool. For a product whose onboarding is built around
Nigerian state-of-origin, AA genotype, and a London-area diaspora, a mismatched
pool would undercut the prototype's credibility with stakeholders. Hand-picked
URLs let each face match the actual target user base: Black men aged 38–45 for
candidates, a Black woman around 40 for the user, who is established elsewhere
in the prototype as a London project manager, mother of two, twelve months
divorced.

Exact photo IDs are selected and verified to resolve during implementation, via
the Playwright MCP browser tools.

`.av` already declares `background-size:cover` and `background-position:center`,
so portraits need no new CSS — only a `background-image` on the existing
element.

---

## Constraints

- **Single file.** All changes in `index.html`. No new runtime files, no build
  step, no libraries. Portraits are remote images, which is a deliberate and
  narrow exception to the prototype's self-containment — noted below as a risk.
- **Reuse existing components.** `.row`, `.card`, `.tag`, `.av`, `.note`,
  `.prog`, `.btn`, and the `:root` custom properties. The meter reuses `.prog`.
  New CSS only for the toast, the confirm sheet, and the bullet list.
- **`S-B2` is already in the `FEED` set** in `classify()`, so no desktop reflow
  change is required.
- **Both themes.** `S-B2` and `S-B3` are `paper`; any new component must still
  be styled for `dark`, per the existing convention that every shared component
  themes both ways.
- **Do not break** the existing tabbar, the `S-B3` invite CTA into `S-C1`, or
  the back-links from `S-C*` and `S-E*` into `S-B2`.

## Risks

- **Remote portraits require network at view time.** If Unsplash is unreachable
  the gallery degrades to the current gradient blocks. Mitigation: keep the
  gradient class on the element so it shows through as a fallback rather than
  rendering an empty box.
- **£3 is unvalidated.** Held in one constant pending the competitive check.

## Non-goals

- No real payment integration. The £3 confirm sheet is a mockup step, like the
  existing group G screens.
- No real matching engine. Scores and bullets are authored fixtures chosen to
  demonstrate the model, not computed.
- `index_mobile_mockup.html` and `index_mockup*.html` are not updated, per the
  standing convention that only `index.html` is kept current.

## Open items

- **£3 restore price** to be cross-checked against Badoo and comparable dating
  apps. Isolated in `RESTORE_PRICE` so the change is one line.
- **Band weights** (30/20/15/20/15) are a first proposal derived from the
  `S-P1`–`S-P5` enrichment structure. They are authored fixtures in a
  prototype, so they can be revised without rework.
