# CLAUDE.md — BOL Parsing Tool

## What this is
A single-page browser tool (`index.html`, no build system, deployed via GitHub Pages at zobott-dot.github.io/bol-extractor — the repository/URL keep the old `bol-extractor` name; only the displayed product name changed to "BOL Parsing Tool" in v5.0). It parses a daily SG360 BOL report PDF and extracts the deduplicated piece count for searched job numbers.

## What the tool does, in order
1. **Filter** — a daily report contains the Bills of Lading and Manifests for ALL jobs that shipped that day. Given a searched job number, discard every row/section not associated with that job. Everything else is noise.
2. **Quantify** — within the isolated job, total the pieces shipped, distinguishing additive drops from true duplicates, and surface a mismatch warning only for a genuine BOL-vs-Manifest discrepancy.

## CRITICAL domain concept: drops and the form/run token
A single job frequently ships as **multiple legitimate drops that must be SUMMED**. The discriminator is the **form/run token** in the Version/Form column — `F7R3`, `F8R3`, `F4R3`, etc. The leading `F#R#` token identifies the drop.

- **Different form/run tokens, same job → additive drops. SUM them.** (Example: job 161683 = F7R3 138,950 + F8R3 108,727 = 247,677.)
- **Same form/run token in both a BOL and its manifest → true duplicate.** Dedupe, count once.
- **BOL and Manifest disagree for the SAME (job, form/run) drop → true mismatch.** This is the only thing that should fire the amber warning.
- Some jobs use non-F#R# versions (`1-004`, `3-020`, etc.). For those, fall back to keying on trip number / section index.

## Architecture invariant — DO NOT REGRESS
Records are keyed by **(job number, form/run)**, NOT by bare job number. Dedup grouping uses this drop key (with trip number / section index as fallback). The mismatch detector compares ONLY same-drop BOL↔Manifest pairs.

**Do not "simplify" the grouping back to job-number-only.** Doing so reintroduces a fixed bug: multiple drops of one job get read as conflicting duplicates and fire a false mismatch warning, even though the headline total is correct. This was fixed on 2026-06-30 (commit "Fix multi-drop false mismatch by keying dedup on form/run"). The three cases — additive drops vs. true duplicate vs. true mismatch — must remain distinct and correctly handled.

## Resolution UX (for genuine mismatches)
When a real mismatch fires the amber warning and the user picks a source:
- The amber box collapses to a calm green single line: `Using {source} total: N — Change`.
- No stale "Currently using" line in the resolved state (that line belongs only on the pre-resolution warning).
- The "Change" affordance re-expands the full options panel with the prior choice highlighted; the user can switch repeatedly. This is the revert path (there is no separate undo button).
- The resolved line is hidden / grayed in print output.

## Testing expectations after any change to parsing or dedup
1. Job 161683 → 247,677, NO mismatch warning (multi-drop sum).
2. A non-F#R# job (e.g. 160473D1, 161685DS) totals correctly via the fallback key.
3. A single-drop job appearing in both a BOL and its manifest (e.g. 161682 = 138,952) is counted ONCE, not doubled.
4. A genuine same-drop BOL/Manifest disagreement still fires the warning.

## Phase roadmap
- **Phases 3 and 4 — COMPLETE.** Parsing, deduplication, confidence/clarification, filter-PDF, filename editing, and export are live. The Export button sits in the results action row next to Filter PDF and opens a menu with three actions, all wired end-to-end: **Copy to Clipboard** (`exportClipboard` — text summary via `navigator.clipboard.writeText`), **Download CSV** (`exportCsv` — Job Number / Total Pieces / Source / Confidence / Notes columns with a descriptive filename built from the source file and searched jobs), and **Print / Save as PDF** (`exportPrint` — `window.print()` against the print stylesheet, with dedicated print header/footer). The CSV path was user-verified on 2026-07-21 producing accurate deduplicated totals with dedup provenance in the Notes column. Export was built against the single-source model, which is the model that actually shipped, so the older "dual-source resequencing / reconcile the stale export commit" concern is resolved and moot. The 4a/4b/4c restructuring plan is retired.
- **Phase 5 (v5.0) — current.** Cosmetic/UX release: product renamed to "BOL Parsing Tool" (display only; repo/URL unchanged), added Quick Start and User Guide modals in the header (Quick Start pulses until first opened; both close via X / backdrop / Esc), introduced the v5.x footer version scheme (bumped per deploy for cache verification), and added the two attention-directing animations — count-up on the headline total (~500 ms ease-out) and staggered fade-and-rise on results sections (~275 ms per block, 75 ms stagger). Both animations and the pulse are suppressed under `prefers-reduced-motion: reduce`. No parsing/dedup/clarification/filter/PDF-generation logic was touched.
