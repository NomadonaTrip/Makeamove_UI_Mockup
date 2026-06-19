# MakeaMove Game-Show UI — Refinement Design

**Date:** 2026-06-18
**Status:** Approved
**Source spec:** `prompt.md`
**Target file:** `index.html` (single self-contained page)

## Summary

Refine the existing `MakeaMove` game-show mockup so it matches the four
requirements in `prompt.md` that the current build does not yet meet, while
preserving everything that already works (stage backdrop, animations, host
voice, keyboard controls, responsive layout, reduced-motion support).

This is a **refinement of one existing file**, not a rewrite. No new runtime
files, no build step. The page remains a self-contained `index.html`.

## Decisions (locked)

- **Starting point:** refine existing `index.html` (keep the polished build).
- **Theme/content:** keep the current "MakeaMove" dating-show questions and
  host voice.
- **Scoring:** flat 5 points per question; win threshold = **70% of total
  available points** (`earned ÷ totalBankPoints`).

## Requirements (from `prompt.md`)

1. Game-show front end inspired by a game-show set + "Who Wants to Be a
   Millionaire" layout.
2. Q&A format with two reaction buttons: **Move** and **Stay**.
3. Bluish-themed background fills the **entire** screen.
4. Foreground visual elements laid out within the **middle 80% width** (10%
   padding on each side).
5. Buttons pinned at the **bottom** of the screen.
6. **Stay = gray**, **Move = pink**.
7. On **Move**, the answer "lights up as the correct answer" and takes a
   **pink background**, changing from its traditional white-text-on-blue.
8. Questions come from a question bank; goal is a **70% score**.
9. Each question carries **5 points**; every **Move** awards the responder 5
   points; win at 70% of total available points.

## Gap analysis (current build vs. spec)

| Spec item | Current state | Action |
|-----------|---------------|--------|
| 80% width rail (item 4) | Foreground uses full-width padding; score ladder absolute at `left:22px`, outside the 80% zone | Wrap foreground in centered 80% rail; move ladder inside it |
| Answer lights pink (item 7) | Move only adds glow/border (`.flip-move`); panel stays blue with white text | On Move, fill the answer panel with the `--move` pink gradient as the chosen/correct answer |
| 5 pts/question, 70% of total (items 8–9) | Weighted dimensions (`w`), percentage of *asked* weight | Flat 5 pts each; headline % and meter = `earned ÷ totalBankPoints`; pass line at 70% |
| End narrative | "Lose" copy explains the weighting model | Rewrite to talk in points/percentage |
| Bluish background fills screen (item 3) | Dark navy stage already fills 100% | Keep (already compliant); verify it reads as bluish |
| Buttons at bottom, gray/pink (items 5–6) | Controls at bottom; `--stay` gray, `--move` pink | Keep (already compliant) |

## Design

### Architecture

Three visual layers inside `.stage` (full-screen):

1. **Backdrop (100% width):** bulb-wall, drifting beams, amber rail, grain,
   vignette. Decorative only, `pointer-events:none` where applicable.
2. **Content rail (80% width, centered):** a new wrapper providing exactly 10%
   gutters on each side. Contains HUD, center stage (turn line, category chip,
   question, answer panel), score meter, and controls.
3. **Overlay:** end-of-round win/lose screen, full-screen.

A single vanilla-JS engine drives state from a `QUESTIONS` bank. No frameworks.

### Layout: the 80% rule

- Add `.rail` wrapper: `width:80%; max-width:1100px; margin:0 auto;
  display:flex; flex-direction:column; height:100%`.
- Move the score ladder from absolute `left:22px` into the rail as a slim
  meter (right-aligned within the center row, Millionaire-style), so it sits
  inside the 80% zone.
- The backdrop layers remain direct children of `.stage` so they keep filling
  100% of the screen.
- Responsive (`max-width:760px`): rail may widen toward full width and the
  meter becomes a horizontal bar, as the existing media query already handles.

### Answer-lights-pink on Move

- `.panel.flip-move` changes the panel **background** to the `--move` pink
  gradient (solid fill, not just a glow), keeping text legible (white text on
  pink), and marks it as the chosen/correct answer.
- `.panel.flip-stay` keeps the existing dimmed/desaturated treatment.
- Transition is animated; respects `prefers-reduced-motion` (color/opacity
  only, no transform) as the existing rules already do.

### Scoring engine

- Each question is worth a flat **5 points** (`POINTS_PER_Q = 5`). Remove the
  per-question `w` weighting from the data and logic.
- `totalBankPoints = QUESTIONS.length * 5`.
- On **Move:** `earned += 5`. On **Stay:** no points.
- Headline percentage = `round(earned / totalBankPoints * 100)`.
- Fill bar height = that percentage; **pass line stays at 70%**.
- Points readout shown with the meter, e.g. `25 / 35 PTS`.
- **Win** when final percentage ≥ 70% (i.e. earned ≥ 70% of total). With 7
  questions (35 pts) that is ≥ 25 pts → 5 Moves.

### Content / narrative

- Keep the dating-show questions and answers; strip the `w` field (all flat 5).
- Keep the category chip, "deep dive" (`l3`) styling, and host voice as flavor;
  `l3` no longer affects scoring.
- Rewrite the end-screen **lose** copy to explain the result in points/%
  terms instead of the weighting model. Keep the **win** copy's match framing.

### Error / edge handling

- Buttons disabled during the answer-reveal animation (`locked` flag), as today.
- Keyboard shortcuts retained: `←`/`S` = Stay, `→`/`M` = Move.
- `prefers-reduced-motion`: disable bursts/transforms, keep color/opacity.
- Responsive layout at `≤760px` preserved.
- "Play again" resets score, pips, and panel state.

## Verification

Drive the page in a real browser (Playwright):

1. **80% layout:** confirm foreground content sits within the middle 80% with
   ~10% gutters each side; backdrop fills 100%.
2. **Move → pink answer:** clicking Move turns the answer panel pink and marks
   it correct; Stay dims it.
3. **Scoring:** each Move adds 5 points; meter and headline % update; pass line
   at 70%.
4. **Win/lose:** overlay triggers at end; win iff final ≥ 70% (≥5 Moves of 7).
5. **Controls:** Stay is gray, Move is pink, both pinned at the bottom.

## Out of scope

- Backend, persistence, multiplayer, or real question-bank loading.
- New themes or non-dating content.
- Changing the overall visual identity beyond the four spec gaps above.
