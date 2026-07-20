# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

MakeaMove — a game-show-style dating app. This repo contains **clickable HTML prototypes**, not a production app. There is no build step, no package manager, no test suite, and no framework: each prototype is a single self-contained HTML file (inline CSS + vanilla JS; only external dependency is Google Fonts).

- `index.html` — the canonical end-to-end clickable prototype. Responsive: renders as a framed phone showcase on desktop and full-bleed on mobile. **All new work goes here.**
- `index_mobile_mockup.html` — earlier mobile-only snapshot of the same prototype (kept for reference).
- `index_mockup.html` — standalone mockup of just the game-show screen.
- `updates.md` — running list of requested changes (the informal backlog).

`docs/`, `.claude/`, `.playwright-mcp/`, `prompt.md`, and `questions.md` are gitignored working material — product notes, plans, and browser-test artifacts.

## Running / previewing

Static files only. The user typically previews via VS Code Live Preview at `http://127.0.0.1:3000/index.html` (deep-link to a screen with `#S-XX`, e.g. `#S-A9`). Any static server works:

```bash
python3 -m http.server 3000
```

Verify changes visually with the Playwright MCP browser tools; its logs land in `.playwright-mcp/` (gitignored).

## Architecture of index.html

One file, three layers:

**1. Screens.** Every app screen is a `<section class="screen dark|paper" id="S-XX" data-group="..." data-title="...">` inside `#screenwrap`. `dark` = stage/show surfaces, `paper` = light utility surfaces (the CSS themes both variants of every shared component). Screen IDs follow flow groups A–I:

- `S-A*` / `S-Q1..Q9` / `S-P1..P5` — Onboarding (landing, quiz questions, profile completion)
- `S-B*` Matching · `S-C*` Invite & schedule · `S-D*` The Show · `S-E*` Post-show
- `S-F*` Async game · `S-G*` Payment · `S-H*` Operator dashboard · `S-I*` Trust & safety

**2. Router.** Hash-based: `location.hash = screen id` toggles `.on` on the matching section. The main `<script>` at the bottom of the file builds the ☰ screen index overlay, Prev/Next stepping (inventory order), the desktop flow nav (one link per group), and the breadcrumb. Navigation between screens is plain `<a href="#S-XX">` links.

**3. Desktop reflow.** `classify()` tags each screen with a layout archetype (`lay-feed`, `lay-immersive`, `lay-hero`, `lay-form`) that the ≥768px CSS uses to reflow screens without touching their markup. Archetypes are driven by hardcoded ID sets (`FEED`, `IMMERSIVE`) plus a heuristic — **when adding a list/gallery or full-bleed screen, add its ID to the appropriate set**, otherwise it defaults to a centred form sheet.

Interactive logic notes:

- The Show (`S-D2`) has its own inline `<script>` inside that section: turn data, dimension weights, and Move/Stay scoring. Results are written to `sessionStorage` (`mm_py`, `mm_pt`) and read by the outcome screens `S-D5`/`S-D6` via `updateResult()`.
- The DOB calendar (Q2, `#dobCal`) enforces 18+ (`YMAX = currentYear − 18`); under-18 routes to the rejection screen `S-QREJ`.
- Onboarding branches by nationality: state-of-origin (`S-Q3A`), genotype (`S-Q3B`), and birth-town questions apply to Nigerians only; other Africans get region-of-origin (`S-Q3R`).
- Option selection on question screens uses the shared `pick(el)` helper (`.opt` + `.sel` classes).

## Conventions

- Keep everything in the single file — no external CSS/JS files, no libraries.
- Reuse the CSS custom properties in `:root` (stage/paper palettes, `--move` pink, `--stay` gray, `--display`/`--body` fonts) and shared components (`.btn`, `.card`, `.row`, `.chip`, `.opt`) rather than inventing new styles.
- Mirror substantive changes in `index.html` only; the mobile mockup file is not kept in sync.
