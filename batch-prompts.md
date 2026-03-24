# Batch Prompts

Copy-paste each prompt into a new Claude Code session.

---

## Batch 1: Accessibility

```
Read backlog.md and implement items #1, #4, #5, #14, and #17. These are the accessibility fixes.

The changes are all in the HTML template embedded in claude-session-report.py (it's a single-file Python script — the HTML is an f-string starting around line 745). Key details:

- #1 (P0): Folder headers and session rows are <div onclick> — need to become <button> elements or get role="button" tabindex="0" + keydown handlers for Enter/Space. This is the most important item.
- #4 (P1): Add aria-expanded="false" to folder headers and session rows, toggle to "true" in toggleFolder() and toggleSession() JS functions.
- #5 (P1): Heading hierarchy skips from h1 to h3. Either make folder names h2 and shift detail headings down, or promote detail headings up. Pick whichever approach requires fewer changes.
- #14 (P2): .search-input:focus uses a subtle border change — add :focus-visible with outline: 2px solid var(--progress); outline-offset: 2px to match other interactive elements.
- #17 (P2): Add <meta name="viewport" content="width=device-width, initial-scale=1"> and a basic media query at ~768px.

Remember: all CSS { } in the template must be doubled ({{ }}) because it's an f-string.

After making changes, run: python claude-session-report.py 3 no-ai --no-open
Then open the output HTML in a browser to verify the changes render correctly.
```

---

## Batch 2: Atomic Cache + Code Quality

```
Read backlog.md and implement items #2 and #9. These are fixes to save_summary_cache in claude-session-report.py.

- #2 (P0): save_summary_cache (around line 79) does a direct open(CACHE_FILE, "w") which truncates immediately. Replace with write-to-temp-then-rename pattern using tempfile.mkstemp. The backlog has the exact replacement code.
- #9 (P1): save_summary_cache returns nothing. Have it return True on success, False on failure. Update callers — especially update_statuses_from_file (around line 560-585) which currently returns True regardless of save outcome.

After making changes, run: python claude-session-report.py 3 no-ai --no-open
to verify the script still runs without errors.
```

---

## Batch 3: Visual Hierarchy + Information Design

```
Use superpowers for this request. Read backlog.md and implement items #6, #7, #11, #12, #13, #15, and #16. These are visual/information design improvements to the HTML template in claude-session-report.py. Use impeccable frontend-design and relevant subskills when updating and reviewing.

All changes are in the HTML template (f-string starting around line 745). CSS { } must be doubled as {{ }}.

- #7 (P1): Add more visual separation between the controls area and content. Increase margin-bottom on .controls from 20px to ~36px.
- #6 (P1): De-emphasize completed sessions. Collapse them by default within folders — show active/handed-off/blocked sessions, with a "Show N completed" toggle at the bottom of each folder. This is the most complex item in this batch.
- #13 (P2): Spread the type scale — folder names to 1rem, detail h3 to 0.88rem+, bump minimum font sizes to 0.7rem for badges and 0.75rem for metadata.
- #11 (P2): Change "Unknown" badge text to "Unsummarized" everywhere. Give unsummarized session details some structure — bold the "Opening ask:" label, add a subtle left-border accent.
- #12 (P2): Add title attributes to folder status pills (e.g. title="9 handed off"). Consider replacing total message count in folder info with a status breakdown.
- #15 (P2): Bump bulk buttons to 0.75rem, --text-2 color, matching border/padding with the sort select.
- #16 (P2): Split the detail-meta line into visually distinct segments. Remove or hide the truncated session ID (or make it click-to-copy with the full ID).

After making changes, run the full tool, then open the output HTML to verify. Use playwright for visual assessment. Pay special attention to the completed-session collapsing (#6) — make sure the toggle works and the count is correct.
```

---

## Batch 4: Small Standalone Fixes

```
Use superpowers for this request. Read backlog.md and implement items #3, #8, #10, #18, and #24. These are independent small fixes.

- #3 (P0): The embedded script in session-report-setup.md is a completely different, older version of claude-session-report.py. Remove the embedded script from the setup doc and replace it with instructions to reference the actual file from the repo. Don't try to update the embedded script — just remove it and point to the real file.
- #8 (P1): --max-collect defaults to 15 silently. Either increase the default to something reasonable (50?) or add a note in session-report.md so the slash command passes an explicit --max-collect value.
- #10 (P1): update_cache_from_file and update_statuses_from_file (around lines 496-585) accept arbitrary file paths. Add validation that the file is within ~/.claude/reports/.
- #18 (P2): Add a .gitignore with standard Python ignores (__pycache__, *.pyc, .env, etc.) plus .playwright-mcp/ and plans/.
- #24 (P3): Fix "1 sessions" pluralization bug in the HTML template f-string.

After making changes, run the full tool, then open the output HTML to verify. Use playwright for visual assessment.
```

---

## Batch 5: Polish (P3 — Optional)

```
Use superpowers for this request. Read backlog.md and implement whichever of items #19-23 and #26-27 feel worthwhile. These are all low-priority polish. Skip any that feel like over-engineering.

Candidates roughly in order of value:
- #21: Add / or Ctrl+K to focus search, Escape to clear. Quick JS addition.
- #20: Add title attributes to status badges and status-strip dots with brief definitions (e.g. "Handed Off — session ended expecting continuation in a new session").
- #22: Change .search-input from 170px fixed to min-width: 170px with flex growth.
- #23: Normalize session chevron and folder arrow to same size (10px).
- #19: Wrap footer cache path in a <details> toggle.
- #26: Move REPORT_DIR.mkdir() to a single call at top of main().
- #27: Consolidate format_timestamp and format_timestamp_short into a shared helper.

Items #25 (decode_project_path edge case), #28 (--limit perf), #29 (transcript sampling), and #30 (print stylesheet) are probably not worth the effort right now — skip unless you're motivated.

After making changes, run the full tool, then open the output HTML to verify. Use playwright for visual assessment.
```
