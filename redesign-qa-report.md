# Dashboard Redesign QA Report

**Date:** 2026-03-20
**Branch:** main (merged from feature/dashboard-redesign)
**Report tested:** `session-report-20260320-233311.html` (generated after full `/session-report 3 --refresh` pipeline run)
**Transcript reviewed:** `~/2026-03-20-233533-command-messagesession-reportcommand-message.txt`

---

## Summary

| Category | Count |
|----------|-------|
| Bugs | 4 |
| Design/UX Issues | 4 |
| Pipeline/Slash Command Issues | 3 |
| **Total** | **11** |

---

## Bugs

### B1. Encoding corruption — em dashes render as `â€"`

- **Severity:** High — visible to user in multiple places
- **Location:** Detail panel summary text, timeline expansion text
- **Reproduction:** Open the report, click on "career" project. The detail panel shows: *"Iterated on email wording â€" removed 'don't hesitate' filler"*. Also visible in timeline expansions for claude-session-report sessions: *"backlog items #6, #7, #11...â€" visual and information design"*, *"#1, #4, #5...â€" accessibility fixes"*, *"Reviewed CLAUDE.md â€" no updates needed"*.
- **Root cause:** The em dash character (`—`, U+2014) is being corrupted during the summarization pipeline. The most likely cause is the subagent batch merge step — the transcript shows a Python script merging 5 batch JSON files into `_summaries.json`. If any step reads/writes without explicit `encoding="utf-8"`, Windows will use the system default encoding (cp1252 or similar), corrupting multi-byte UTF-8 characters. The em dash is a classic UTF-8 mojibake victim: the 3-byte sequence `E2 80 94` gets interpreted as three cp1252 characters `â€"`.
- **Affected sessions:** Multiple — any summary containing em dashes, curly quotes, or other non-ASCII characters
- **Fix:** Ensure all JSON read/write operations in the merge script use `encoding="utf-8"`. Also audit `update_cache_from_file()` and the cache save/load functions. The script's own functions already use `encoding="utf-8"`, so the issue is likely in the ad-hoc merge script the slash command session generated (line 75-78 of transcript: `python3 -c "import json..."`).
- **Screenshot:** `redesign-final-career.png` — third bullet point in "What Was Done" section

### B2. Home directory project displays as bare `~`

- **Severity:** Medium — confusing but not broken
- **Location:** Sidebar project name, detail panel project name and path
- **Reproduction:** Open the report. The first project in the sidebar shows `~` as the name, with `~` as the path. The detail panel header shows just `~` for both name and path. The sidebar initials also show `~`.
- **Root cause:** The `aggregate_projects()` function derives `short_name` from the last path component of `short_path(folder)`. For the home directory, `short_path()` returns `~`, so the last component is also `~`. The initials logic `re.split(r'[-_]', '~')` produces `['~']` which gives initials `"~"` (length < 2, uses first 2 chars, but string is only 1 char).
- **Expected behavior:** Should display as "Home" or "~ (Home)" or at minimum handle the single-character case for initials.
- **Fix:** Add a special case in `aggregate_projects()`: if `short_name == "~"`, set `name = "Home"` and `initials = "HM"`. Or more generally, if `short_path(folder)` is exactly `~`, use a friendlier display name.

### B3. Empty "Opening ask:" in unsummarized session

- **Severity:** Low — cosmetic
- **Location:** claude-session-report project → timeline → session at "Today 04:00 PM · 2 msgs · 0m"
- **Reproduction:** Click on claude-session-report in sidebar. Scroll to the unsummarized session at 04:00 PM. The expansion shows `Opening ask:` with no text after it.
- **Root cause:** This is a 2-message session where the first user message was apparently empty or contained only tool use blocks (no text content). The `first_ask` field is empty, so `truncate("", 300)` returns an empty string.
- **Expected behavior:** Should either show "(empty)" or skip the "Opening ask:" line entirely when there's no text.
- **Fix:** In the unsummarized fallback rendering (inside `generate_html()`), check if `first_ask` is empty and either skip the line or show a placeholder.

### B4. Initials generation fails for single-word project names without hyphens/underscores

- **Severity:** Low — cosmetic
- **Location:** Collapsed sidebar initials for projects like "career", "dev", "ha"
- **Reproduction:** Collapse the sidebar. Projects with short single-word names show: "CA" for career (ok), "DE" for dev (ok), "HA" for ha (ok — happens to work). But "~" shows as "~" (only 1 char, no fallback).
- **Root cause:** `re.split(r'[-_]', short_name)` only splits on hyphens and underscores. For names shorter than 2 characters, `short_name[:2].upper()` may return fewer than 2 characters.
- **Fix:** Add a fallback: `initials = short_name[:2].upper() if len(short_name) >= 2 else short_name[0].upper()`. Or pad with a space/default character.

---

## Design/UX Issues

### D1. Almost everything shows as "Active" — inactive threshold too strict

- **Severity:** Medium — defeats the purpose of the active/inactive split
- **Location:** Sidebar project list, filter chips
- **Observation:** 8 of 11 projects show as "Active". The filter chips show "Active 8 / Inactive 3". The 3 inactive projects are only the ones with ALL sessions complete AND last activity > 24h ago. Projects like "ha" (last active 7h ago, most recent session `complete`) and "career" (last active 8h ago, most recent session `complete` — all interview prep is done) still show as Active.
- **Impact:** The active/inactive split was designed to surface "what needs attention" vs "what's quiet". With the current threshold, almost nothing is inactive, so the split adds little value.
- **Possible fixes:**
  - Relax the threshold: inactive if all sessions are `complete` (regardless of recency)
  - Use a longer time window: inactive if latest activity > 12h or 24h ago AND all sessions complete
  - Let the AI determine it during the `--review-statuses` pass (as the spec originally intended)

### D2. No "Next Steps" card visible on most projects

- **Severity:** Low — by design, but worth noting
- **Location:** Detail panel
- **Observation:** The "Next Steps" card only appears when the most recent session's summary includes a `## Next Steps` heading. For the claude-session-report project, it shows "Restart Claude Code for updated slash command definition to take effect". But for career (most recent session is `complete`), there are no next steps shown. For most projects where the most recent session is `complete`, the card is absent.
- **Impact:** The "Next Steps" card was a key part of the re-orientation design ("what's next for this project"). It's invisible for completed sessions, which is arguably correct (nothing is next), but it means the card is rarely seen.
- **Possible fix:** Consider aggregating next steps from the most recent *non-complete* session, or showing a "Last known next steps" from any recent session that had them.

### D3. Most recent session in `~` project is the `/session-report` run itself

- **Severity:** Low — inherent limitation
- **Location:** `~` project detail panel
- **Observation:** The `/session-report` slash command runs from `~` (home directory), so it generates a session that becomes the most recent entry for the `~` project. This means the `~` project's detail panel always shows the meta-session about generating the report, not the actual work.
- **Impact:** The `~` project contains a mix of session-report runs, plugin installs, and other utility sessions. It's noisy by nature.
- **Possible fix:** Consider filtering out `/session-report` sessions from the display, or at least not auto-selecting `~` as the first project. The most recently *worked on* project (excluding report generation) would be more useful as the default.

### D4. Timeline session with no title shows raw prompt text

- **Severity:** Low — cosmetic
- **Location:** `~` project → first timeline session
- **Observation:** The most recent session in `~` shows its title as: *"# Session Report Generate a summary report of all recent Claude Code session..."*. This is the raw slash command prompt text, truncated. It's not a useful title.
- **Root cause:** This session hasn't been summarized, so the title falls back to `truncate(first_ask, 80)`. The first ask is the slash command prompt, which starts with markdown.
- **Impact:** Unsummarized sessions with slash command or markdown prompts look messy.
- **Possible fix:** Strip markdown syntax from fallback titles (`#` headings, `*` bold, etc.).

---

## Pipeline/Slash Command Issues

### P1. `--max-collect 50` left 13 sessions unsummarized

- **Severity:** Medium — silent data loss
- **Location:** Transcript line 25: "50 sessions need summarization"
- **Observation:** With `--refresh`, all 67 sessions needed re-summarization. 4 were too short. 50 were processed. 13 were silently skipped due to the `--max-collect 50` default. The script does print a message about this cap, but the slash command doesn't loop to process remaining sessions.
- **Impact:** The user ran `--refresh` expecting all sessions to be re-summarized, but 13 were missed. These 13 sessions retain their old summaries with the old heading format (or no summaries if they were new).
- **Fix options:**
  - Increase `--max-collect` default (e.g., 100 or unlimited)
  - Have the slash command loop: re-run `--collect` after `--update-cache` to see if more sessions need processing
  - Add `--max-collect 0` to mean "unlimited"
  - Pass `--max-collect 100` in the slash command prompt

### P2. Subagent batches ran sequentially instead of in parallel

- **Severity:** Medium — performance
- **Location:** Transcript lines 47-70
- **Observation:** The slash command dispatched 5 subagent batches for summarization. The first 3 ran sequentially (each waited for the previous to complete). The user intervened ("Why aren't they running in parallel") and batches 4-5 then ran in parallel. Total time was ~11m 40s when it could have been ~3-4 minutes.
- **Root cause:** The slash command prompt doesn't explicitly instruct Claude to dispatch all batch agents in parallel using a single message with multiple tool calls. Claude defaulted to sequential dispatch.
- **Fix:** Add explicit instruction in `session-report.md`: "Dispatch ALL batch agents in a single message so they run in parallel."

### P3. `--refresh` flag passed to final render step unnecessarily

- **Severity:** Negligible — no functional impact
- **Location:** Transcript line 147: `python ... --refresh` on the render step
- **Observation:** The `--refresh` flag is meaningful for `--collect` (skip cache, re-summarize everything). Passing it to the final render step has no effect since the render step doesn't use it.
- **Root cause:** The slash command prompt passes `$ARGUMENTS` (which includes `--refresh`) to all script invocations, including the final render. The script's argparse accepts it without error but ignores it.
- **Fix:** Not critical. Could strip `--refresh` from `$ARGUMENTS` for the render step, but it's harmless.

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

## Recommendations (Priority Order)

1. **Fix B1 (encoding)** — High impact, affects visible content. Root cause is likely the merge script's encoding. Fix the slash command prompt to specify `encoding="utf-8"` in the merge step, and re-run with `--refresh`.

2. **Fix B2 (`~` project display)** — Quick fix in `aggregate_projects()`.

3. **Fix P1 (max-collect cap)** — Increase default or add looping to the slash command.

4. **Address D1 (inactive threshold)** — Relax to "all sessions complete" regardless of recency.

5. **Fix P2 (parallel batches)** — Add instruction to slash command prompt.

6. **Fix B3 (empty opening ask)** — Skip the line when empty.

7. **Address D3 (filter report sessions)** — Consider filtering or de-prioritizing `/session-report` sessions.
