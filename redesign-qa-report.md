# Dashboard Redesign QA Report

**Date:** 2026-03-20
**Branch:** main (merged from feature/dashboard-redesign)
**Report tested:** `session-report-20260320-233311.html` (generated after full `/session-report 3 --refresh` pipeline run)
**Transcript reviewed:** `~/2026-03-20-233533-command-messagesession-reportcommand-message.txt`
**Root cause investigation:** 2026-03-21 — all 11 items systematically verified against source code and live data

---

## Summary

| Category | Count | Verified |
|----------|-------|----------|
| Bugs | 4 | 4/4 confirmed |
| Design/UX Issues | 4 | 3 confirmed, 1 downgraded |
| Pipeline/Slash Command Issues | 3 | 3/3 confirmed |
| **Total** | **11** | **All investigated** |

---

## Bugs

### B1. Encoding corruption — em dashes render as `â€"`

- **Severity:** High — visible to user in multiple places
- **Status:** ROOT CAUSE CONFIRMED AND REPRODUCED
- **Location:** Detail panel summary text, timeline expansion text
- **Reproduction:** Open the report, click on "career" project. The detail panel shows: *"Iterated on email wording â€" removed 'don't hesitate' filler"*. Also visible in timeline expansions for claude-session-report sessions: *"backlog items #6, #7, #11...â€" visual and information design"*, *"#1, #4, #5...â€" accessibility fixes"*, *"Reviewed CLAUDE.md â€" no updates needed"*.
- **Affected sessions:** 25 occurrences of mojibake in the report HTML. Corruption persists in the cache.
- **Root cause (VERIFIED):**
  1. The ad-hoc merge script generated during the `/session-report` run uses `open()` without `encoding="utf-8"`
  2. On this Windows system, `locale.getpreferredencoding()` returns `cp1252`
  3. Python's `open()` without explicit encoding defaults to cp1252
  4. The 5 batch JSON files were written by the Write tool as UTF-8
  5. The merge script reads them as cp1252, corrupting multi-byte chars:
     - Em dash `—` (U+2014) is UTF-8 bytes `E2 80 94`
     - cp1252 reads these as three chars: `â` (0xE2), `€` (0x80), `"` (0x94)
  6. The corrupted text is written to `_summaries.json` (also without encoding)
  7. `update_cache_from_file()` reads it correctly as UTF-8, but the data is already corrupted
  - **Verified by:** Reproduced the exact corruption in a test: writing `"hello — world"` as UTF-8 JSON, reading with cp1252 → produces `"hello â€" world"` with chars U+00E2, U+20AC, U+201D — exact match to the cache data.
  - **Note:** The script's own functions (`load_summary_cache`, `save_summary_cache`, `update_cache_from_file`) all correctly specify `encoding="utf-8"`. The bug is solely in the ad-hoc merge script Claude generates at runtime.
- **Fix:** Update `session-report.md` to either: (a) instruct the merge step to use `encoding="utf-8"` on all `open()` calls, or (b) eliminate the merge step by having the script accept multiple input files, or (c) add `PYTHONUTF8=1` to the environment.

### B2. Home directory project displays as bare `~`

- **Severity:** Medium — confusing but not broken
- **Status:** ROOT CAUSE CONFIRMED
- **Location:** Sidebar project name, detail panel project name and path
- **Reproduction:** Open the report. The first project in the sidebar shows `~` as the name, with `~` as the path. The detail panel header shows just `~` for both name and path. The sidebar initials also show `~`.
- **Root cause (VERIFIED):**
  - `short_path("C:\\Users\\Ben")` returns `"~"` (line 237-238: replaces home dir prefix with `~`, result is exactly `"~"`)
  - `short_name` derivation (line 850): `"~".rstrip("/\\").split("/")[-1]` → `"~"`
  - Initials (line 856): `"~"[:2].upper()` → `"~"` (only 1 char available)
  - No special-casing exists for the home directory
- **Fix:** Special-case in `aggregate_projects()`: if `short_path(folder) == "~"`, set `name = "Home"`, `initials = "HM"`.

### B3. Empty "Opening ask:" in unsummarized session

- **Severity:** Low — cosmetic
- **Status:** ROOT CAUSE CONFIRMED
- **Location:** claude-session-report project → timeline → session at "Today 04:00 PM · 2 msgs · 0m"
- **Reproduction:** Click on claude-session-report in sidebar. Scroll to the unsummarized session at 04:00 PM. The expansion shows `Opening ask:` with no text after it.
- **Root cause (VERIFIED):**
  1. The session has 2 messages where user messages start with `<command-` tags (slash command invocation)
  2. `clean_user_text()` (line 246-248) returns `None` for messages starting with `<command-` → `topics` is empty
  3. Fallback: `clean_transcript_text()` strips all `<command-*>` tags → empty string
  4. `truncate("", 200)` → `""`
  5. `first_ask` = `""`
  6. Line 948: `f'<p><strong>Opening ask:</strong> {html_escape(truncate(s["first_ask"], 300))}</p>'` renders `Opening ask:` followed by nothing
- **Fix:** Check `if first_ask` before rendering; skip the line or show `"(slash command)"` placeholder.

### B4. Initials generation fails for single-word project names without hyphens/underscores

- **Severity:** Low — cosmetic
- **Status:** ROOT CAUSE CONFIRMED (same as B2)
- **Location:** Collapsed sidebar initials
- **Root cause (VERIFIED):** `short_name[:2].upper()` on `"~"` (1 char) produces `"~"`. No other real-world projects have single-char names (checked all project directories). This is a sub-issue of B2 — the only affected project is the home directory.
- **Fix:** Fixing B2 with a `~` → `"Home"` special case resolves this.

---

## Design/UX Issues

### D1. Almost everything shows as "Active" — inactive threshold too strict

- **Severity:** Medium — defeats the purpose of the active/inactive split
- **Status:** ROOT CAUSE CONFIRMED
- **Location:** Sidebar project list, filter chips
- **Observation:** 10 of 13 projects show as "Active". Only 3 are inactive (zeno-test-engineer, claude-tool-analyzer, figo-complaint-march-2026).
- **Root cause (VERIFIED):** Two compounding issues:
  1. **`handed_off` not treated as "done":** Line 842 checks `s["_status"] in ("complete", "unknown")`. The `handed_off` status is NOT in this set, so projects with handed_off sessions (like "career" with 12 handed_off sessions) are never classified as all-complete. `handed_off` is functionally "done" at the project level — the work continued in another session.
  2. **24h recency check is redundant:** Line 844 requires `latest_time < inactive_threshold` (24h ago). Projects like "ha" (7.6h ago, all complete) and "claude-session-report" (0.6h ago, all complete) stay active purely due to recency, even though all work is done.
  3. **Stale `in_progress` sessions:** Projects like "homenetwork" (56.7h ago) and "oh-hi-markdown" (77.7h ago) have old `in_progress` or `handed_off` sessions that were never corrected, keeping them artificially active.
- **Fix:** Include `handed_off` in the "done" set. Consider removing the time threshold — if all sessions are done, the project is inactive regardless of when the last activity was.

### D2. No "Next Steps" card visible on most projects

- **Severity:** Low — **DOWNGRADED: less severe than originally reported**
- **Status:** PARTIALLY INCORRECT — re-investigated
- **Location:** Detail panel
- **Original claim:** "The card is rarely seen."
- **Actual finding:** 9 of 13 projects DO have `## Next Steps` in their most recent session's summary. The original observation was based on the report file from the slash command run that:
  1. Used a mix of old-format summaries (from before the heading format update) and new-format summaries
  2. Had 13 sessions unsummarized due to the --max-collect cap (P1)
  After a full refresh with the updated format, most sessions include `## Next Steps` when appropriate. The 4 projects without it (Ben, audio-transcriber, career, ha) have genuinely complete work where no next steps exist — this is correct behavior.
- **Revised assessment:** Not a bug. The slash command's summary format correctly omits `## Next Steps` for fully complete sessions. After a clean refresh run, the card appears for the majority of projects.

### D3. Most recent session in `~` project is the `/session-report` run itself

- **Severity:** Low — inherent structural limitation
- **Status:** ROOT CAUSE CONFIRMED
- **Location:** `~` project detail panel
- **Root cause (VERIFIED):** The `/session-report` slash command runs from `~` (home directory), creating sessions under `C--Users-Ben` project folder. 7 of 15 home-dir sessions are report-generation runs. The most recent session is always a report-gen session, making the `~` project detail panel show meta-information about report generation.
- **Impact:** The `~` project is inherently noisy — mix of session-report runs, plugin installs, config changes, and other utility sessions.
- **Fix options:**
  1. Filter sessions whose title matches "session report generation" from display
  2. Don't auto-select `~` as first project — select the most recently *worked on* project instead
  3. Deprioritize `~` in the sort order (place after other active projects)

### D4. Timeline session with no title shows raw prompt text

- **Severity:** Low — cosmetic
- **Status:** ROOT CAUSE CONFIRMED
- **Location:** `~` project → first timeline session
- **Root cause (VERIFIED):**
  1. Unsummarized sessions use `truncate(first_ask, 80)` as fallback title (line 830)
  2. For slash command sessions, `first_ask` contains the raw prompt including markdown headings
  3. `clean_user_text()` (line 246-248) only blocks `"# Screenshot"` prefix, not general markdown headings
  4. `truncate()` replaces newlines with spaces but preserves markdown syntax
  5. Result: `"# Session Report  Generate a summary report of all recent Claude Code session..."`
- **Fix:** Strip leading markdown heading syntax (`#+ `) from fallback titles before truncation.

---

## Pipeline/Slash Command Issues

### P1. `--max-collect 50` left 13 sessions unsummarized

- **Severity:** Medium — silent data loss
- **Status:** ROOT CAUSE CONFIRMED
- **Location:** `claude-session-report.py` line 2087, line 568
- **Root cause (VERIFIED):**
  1. `--max-collect` defaults to `50` (line 2087 argparse definition)
  2. With `--refresh`, cache is emptied (`cache = {}` at line 537), so all sessions need re-summarization
  3. 67 total sessions - 4 too short = 63 needing summary, but capped at 50 (line 568: `break`)
  4. Line 2140 prints `"(capped at --max-collect 50; run again for more)"` to stdout
  5. The slash command prompt (`session-report.md`) contains no instruction to check for this cap message or loop
  6. The 13 skipped sessions retain their pre-refresh summaries (old heading format)
- **Fix options:**
  1. **Increase default** to 0 (unlimited) — sessions are already limited by `--limit` and the day range
  2. **Add loop instruction** to the slash command: re-run `--collect` after `--update-cache` if the cap was hit
  3. **Pass `--max-collect 0`** in the slash command's `--collect` invocation

### P2. Subagent batches ran sequentially instead of in parallel

- **Severity:** Medium — performance
- **Status:** ROOT CAUSE CONFIRMED
- **Location:** `session-report.md`
- **Root cause (VERIFIED):** The slash command prompt contains zero instructions about parallelism, batching, or subagent dispatch strategy. It says "For each session in the `sessions` array, read its `transcript`..." — leaving the execution strategy to Claude's judgment. Claude chose batch subagents (a good strategy) but dispatched them sequentially (3 of 5 batches ran one-at-a-time before the user intervened).
- **Impact:** ~11m 40s total when parallel dispatch would have been ~3-4 minutes.
- **Fix:** Add explicit instruction in `session-report.md` Step 3: "When summarizing, split sessions into batches of ~10 and dispatch ALL batch subagents in a single message so they run in parallel."

### P3. `--refresh` flag passed to final render step unnecessarily

- **Severity:** Negligible — no functional impact
- **Status:** ROOT CAUSE CONFIRMED
- **Location:** `session-report.md` Step 4, `claude-session-report.py` line 2081
- **Root cause (VERIFIED):**
  1. Step 4 runs: `python "INSTALL_DIR/claude-session-report.py" $ARGUMENTS`
  2. `$ARGUMENTS` includes `--refresh` from the user's `/session-report --refresh` invocation
  3. argparse accepts `--refresh` (line 2081: `store_true`)
  4. `args.refresh` is only referenced at line 2131 inside `if args.collect:` block
  5. The render path (line 2154+) never reads `args.refresh`
  6. No functional impact — the flag is silently accepted and ignored
- **Fix:** Not worth fixing. The flag is harmless. Could strip `--refresh` from `$ARGUMENTS` in Step 4 for cleanliness.

---

## Verification Screenshots

| File | Description |
|------|-------------|
| `redesign-final-overview.png` | Light theme, `~` project selected, sidebar expanded |
| `redesign-final-csr-project.png` | Light theme, claude-session-report project, structured sections visible |
| `redesign-final-career.png` | Light theme, career project, 26 sessions, encoding bug visible |
| `redesign-dark-overview.png` | Dark theme, initial view |
| `redesign-dark-expanded.png` | Dark theme, session expanded with color-coded blocks |
| `redesign-dark-collapsed.png` | Dark theme, sidebar collapsed to initials |
| `redesign-light-overview.png` | Light theme, overview |
| `redesign-with-summaries.png` | Light theme, with cached AI summaries |

---

## Recommendations (Revised Priority Order)

1. **Fix B1 (encoding)** — High impact, 25 corrupted entries in cache. Root cause is the ad-hoc merge script's missing `encoding="utf-8"`. Fix the slash command prompt, then re-run with `--refresh` to regenerate all summaries.

2. **Fix P1 (max-collect cap)** — Medium impact, caused 13 sessions to be skipped silently. Change default to 0 (unlimited) or add looping to slash command.

3. **Fix D1 (inactive threshold)** — Medium impact, 10/13 projects wrongly active. Include `handed_off` in the "done" set and consider removing the 24h recency requirement.

4. **Fix B2 + B4 (`~` project display)** — Quick fix, one special case in `aggregate_projects()`.

5. **Fix P2 (parallel batches)** — Medium performance impact. Add one sentence to `session-report.md`.

6. **Fix B3 (empty opening ask)** — Low, cosmetic. Check if `first_ask` is empty before rendering.

7. **Fix D4 (raw markdown titles)** — Low, cosmetic. Strip `#` headings from fallback titles.

8. **Address D3 (meta-sessions in ~)** — Low, structural. Consider deprioritizing `~` project or filtering report-gen sessions.

9. **D2 (Next Steps card)** — Downgraded. Working as designed after full refresh; no fix needed.

10. **P3 (--refresh on render)** — Negligible. No fix needed.
