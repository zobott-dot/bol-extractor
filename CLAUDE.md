# CLAUDE.md — BOL Extractor

## What this is
A single-page browser tool (`index.html`, no build system, deployed via GitHub Pages at zobott-dot.github.io/bol-extractor). It parses a daily SG360 BOL report PDF and extracts the deduplicated piece count for searched job numbers.

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

## Phase roadmap (resequenced 2026-06-30)
Phase 4 was restructured: **4a = dual-source results** (show BOL and Manifest totals in parallel, surfacing disagreements as facts rather than hiding them via the hierarchy); **4b = exports** (clipboard, CSV, print/PDF); **4c = polish, error handling, first-run welcome overlay**. Pallet-count extraction is parked, conditional on tester demand. NOTE: an earlier "Phase 4a export functionality" commit exists in history, built against the single-source model before this resequencing — reconcile it when the export phase begins.
