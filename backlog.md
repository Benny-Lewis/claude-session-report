# Session Report — Unified Backlog

Merged from three review documents (all dated 2026-03-20):
- **Code Review** (`code-review-findings.md`) — Full codebase review
- **UX Audit** (`ux-audit-report.md`) — Heuristic usability evaluation
- **Design Critique** (`design-critique-report.md`) — Visual/information design review

Items marked `[DONE]` have been implemented.

---

## P0 — Critical

### 1. [DONE] Folder headers and session rows are keyboard-inaccessible

- **Category:** Accessibility
- **Source:** UX Audit (Severity 4)
- **Location:** `.folder-header` (div with onclick), `.session-row` (div with onclick) in HTML template
- **Issue:** All session rows and folder headers are `<div>` elements with inline `onclick`. They have no `role="button"`, no `tabindex="0"`, and no `onkeydown` handler. Keyboard-only users cannot Tab to or activate any of these — the entire content tree is unreachable without a mouse.
- **User impact:** A keyboard-only user (motor disability, broken mouse, power user navigating with Tab) cannot expand any folder or read any session detail. The page is essentially a static header bar for them.
- **Fix:** Either change these to `<button>` elements, or add `role="button" tabindex="0"` and a keydown handler that triggers on Enter/Space. The `<button>` approach is simpler and inherits keyboard behavior for free.

---

### 2. [DONE] Non-atomic cache writes risk data loss

- **Category:** Code Quality
- **Source:** Code Review (Critical #1)
- **Location:** `claude-session-report.py:79-84` — `save_summary_cache()`
- **Issue:** `save_summary_cache` does a direct `open(CACHE_FILE, "w")` which truncates the file immediately. If the process crashes mid-write, the cache file is destroyed. On Windows, another process reading the file simultaneously could also cause corruption.
- **User impact:** A crash or concurrent access during cache save destroys all cached summaries, forcing a full re-summarization pass.
- **Fix:** Write to a temporary file first, then rename:
  ```python
  import tempfile

  def save_summary_cache(cache: dict):
      try:
          fd, tmp_path = tempfile.mkstemp(dir=CACHE_FILE.parent, suffix=".tmp")
          with os.fdopen(fd, "w", encoding="utf-8") as f:
              json.dump(cache, f, indent=2, ensure_ascii=False)
          Path(tmp_path).replace(CACHE_FILE)
      except OSError as e:
          print(f"  WARNING: Failed to write cache: {e}")
  ```
  Note: `Path.replace()` is atomic on POSIX but not guaranteed atomic on Windows NTFS — however it is still much safer than the current approach since the original file is only replaced after the new content is fully written.

---

### 3. [DONE] Stale embedded script in `session-report-setup.md`

- **Category:** Documentation
- **Source:** Code Review (Critical #2)
- **Location:** `session-report-setup.md` vs `claude-session-report.py`
- **Issue:** The embedded script in the setup doc (~1010 lines inside a fenced code block) is a completely different, older version that:
  - Imports `os`, `threading`, `concurrent.futures`, `anthropic`, `httpx`
  - Contains `summarize_with_haiku()` which calls a LiteLLM proxy directly
  - Uses `concurrent.futures.ThreadPoolExecutor` for parallel Haiku calls
  - Has a different `generate_html()` lacking session statuses, status filtering, sorting, theme toggle, and the status badge system
  - Lacks the `--collect`, `--update-cache`, `--review-statuses`, and `--update-statuses` pipeline subcommands
  - References `SETTINGS_FILE` and reads `ANTHROPIC_BASE_URL` / `ANTHROPIC_AUTH_TOKEN` from settings.json
  - The `md_to_html` function does NOT call `html_escape()` on list item content, creating an actual XSS vulnerability in that version
- **User impact:** Anyone using the setup doc gets a stale, less secure, less functional version of the script.
- **Fix:** Either update the setup doc to embed the current version of the script, or (better) remove the embedded script from the setup doc and have it reference the repo/installation path instead.

---

## P1 — High

### 4. [DONE] No `aria-expanded` on collapsible elements

- **Category:** Accessibility
- **Source:** UX Audit (Severity 3)
- **Location:** Folder headers, session rows in HTML template
- **Issue:** Folders and sessions toggle between collapsed/expanded states via CSS class (`.open`, `.expanded`), but there's no `aria-expanded` attribute to communicate state to screen readers. The grid-template-rows animation is purely visual.
- **User impact:** A screen reader user can't tell whether a folder or session is open or closed — they hear the text but get no state information.
- **Fix:** Add `aria-expanded="false"` to each trigger element, and toggle it to `"true"` in `toggleFolder()` and `toggleSession()`.

---

### 5. [DONE] Heading hierarchy skips levels

- **Category:** Accessibility
- **Source:** UX Audit (Severity 3)
- **Location:** `.detail-body h3`, `.detail-body h4` in HTML template
- **Issue:** The page has one `<h1>` ("Session Report") and then jumps directly to `<h3>` and `<h4>` inside session details. There are no `<h2>` elements. Folder names (`.folder-path`) are styled as bold text but are `<span>` elements, not headings.
- **User impact:** Screen reader users navigating by headings will find a broken hierarchy — jumping from h1 to h3 is disorienting and makes it harder to build a mental model of the page structure.
- **Fix:** Either make folder names `<h2>` and shift detail headings to `<h3>`/`<h4>`, or change the summary headings from `<h3>`/`<h4>` to `<h2>`/`<h3>` since they're the next structural level below the page title.

---

### 6. [DONE] Completed sessions visually dominate over active ones

- **Category:** Information Architecture
- **Source:** Design Critique (Priority #2)
- **Location:** Folder session lists in HTML template
- **Issue:** In folders with many sessions, completed sessions take up the majority of visible space, pushing the interesting (active/handed-off) sessions down the list. The title dimming (`color: var(--text-2)`) is subtle — not enough to create a clear visual tier.
- **User impact:** The reason you look at this dashboard is to find what's active or needs attention, but the page treats all sessions as roughly equal visual weight. The user has to scan linearly through a list of "complete" badges to find the one "in progress" item.
- **Fix:** Options: (a) collapse completed sessions by default within a folder (show only active/handed-off, with a "Show 12 complete" toggle), (b) make completed rows significantly more muted (reduce opacity, smaller padding), or (c) visually group active sessions at the top of each folder with a separator. Option (a) is the most impactful — completed sessions are archival, not actionable.

---

### 7. [DONE] No visual seam between controls and content

- **Category:** Visual Hierarchy
- **Source:** Design Critique (Priority #1)
- **Location:** `.status-strip`, `.controls`, folder list in HTML template
- **Issue:** The status strip (summary counts), filter chips (interactive controls), and folder list (the actual content) are visually stacked with similar spacing and no clear boundary. The border-bottom on `.status-strip` helps, but the chips float in the same visual zone as the folder list below.
- **User impact:** Users scanning the page can't immediately distinguish "these are controls I act on" from "this is the content I'm reading." The controls bar bleeds into the data.
- **Fix:** Options: (a) give the controls area a `var(--surface)` background panel, (b) increase the margin-bottom on `.controls` from 20px to 32-40px, or (c) add a thin border-bottom on `.controls`. The lightest touch would be option (b) — just more air.

---

### 8. [DONE] `--max-collect` silently caps summarization at 15

- **Category:** CLI/UX
- **Source:** Code Review (Important #3)
- **Location:** `claude-session-report.py:1271`
- **Issue:** The `--max-collect` flag defaults to 15, but `session-report.md` never passes it. If a user runs `/session-report 14` and has 50 sessions needing summarization, only 15 will be collected per run. The script prints a message about this cap, but the slash command doesn't loop.
- **User impact:** Users expecting all sessions to be summarized may not realize some were silently skipped.
- **Fix:** Either increase the default, document this behavior in the slash command, or have the slash command pass an explicit `--max-collect` value.

---

### 9. [DONE] `save_summary_cache` returns no success indicator

- **Category:** Code Quality
- **Source:** Code Review (Important #4)
- **Location:** `claude-session-report.py:79-84, 560-585`
- **Issue:** At line 583 in `update_statuses_from_file`, `save_summary_cache` is called when `updated > 0`, but returns no success indicator. The caller returns `True` regardless of whether the save succeeded, silently losing status corrections.
- **User impact:** Status corrections from the cross-session review pass could be silently lost if the cache write fails.
- **Fix:** Have `save_summary_cache` return a boolean, and propagate failures to callers.

---

### 10. [DONE] No input validation on `--update-cache` and `--update-statuses` file paths

- **Category:** Security
- **Source:** Code Review (Important #6)
- **Location:** `claude-session-report.py:496-522, 560-585`
- **Issue:** `update_cache_from_file` and `update_statuses_from_file` read arbitrary files specified on the command line and merge their JSON contents into the cache. No validation that the file path is within expected directories, the JSON content has reasonable size limits, or session IDs are plausible formats.
- **User impact:** Low risk for a local tool, but if the slash command ever passes unsanitized user input, it could overwrite cache entries arbitrarily.
- **Fix:** Validate that file paths are within `~/.claude/reports/` and add basic sanity checks on JSON content.

---

## P2 — Medium

### 11. [DONE] "Unknown" vs "Unsummarized" inconsistent terminology + unsummarized sessions feel second-class

- **Category:** Terminology / Visual Design
- **Source:** UX Audit (Severity 2, #4) + Design Critique (Priority #5)
- **Location:** Filter chip bar ("Unsummarized"), session badges ("Unknown"), unsummarized session detail panels
- **Issue:** The same concept — sessions without AI summaries — is called "Unsummarized" in the filter chip bar and "Unknown" on the session badge. Additionally, unsummarized sessions have a dashed-border badge and less visual structure (no h3/h4 headings inside, just raw ask/response paragraphs), signaling "broken" rather than "needs summarization."
- **User impact:** Label mismatch causes confusion ("Does 'Unknown' mean something different from 'Unsummarized'?"). The weak visual treatment makes potentially important ongoing work look like second-class content.
- **Fix:** (a) Use "Unsummarized" consistently everywhere. (b) Give the raw ask/response format minimal structure — bold the "Opening ask:" label, add a subtle left border accent. (c) Auto-generate a title from the first 60 chars of the opening message, formatted cleanly.

---

### 12. [DONE] Folder status pills and metadata don't tell the most useful story

- **Category:** Information Design
- **Source:** UX Audit (Severity 2, #5) + Design Critique (Priority #4)
- **Location:** `.fpill` elements and `.folder-info` in folder headers
- **Issue:** Folder meta shows colored number pills (e.g., a purple `9`) but the number has no label — users must cross-reference the color with the status strip legend. The folder info line (`"26 sessions . 4,478 msgs . Today 02:38 PM"`) includes total message count, which is a vanity metric that doesn't help decide whether to expand the folder.
- **User impact:** "What does the purple 9 mean?" First-time viewers won't understand the pills. The metadata doesn't answer the relevant question: how many sessions are active?
- **Fix:** (a) Add `title` attributes to pills (e.g., `title="9 handed off"`), or display counts with short labels. (b) Replace total message count with a more useful signal: a compact status breakdown like `"3 active . 9 handed off . 14 complete"` or the title of the most recent session.

---

### 13. [DONE] Type scale too compressed / small font sizes

- **Category:** Typography
- **Source:** UX Audit (Severity 2, #6) + Design Critique (Priority #3)
- **Location:** Multiple elements — `.badge` at 0.65rem, `.session-msgs` at 0.7rem, `.session-time` at 0.73rem, `.folder-info` at 0.73rem, `.bulk-btn` at 0.7rem; folder path (0.88rem), session title (0.84rem), detail body (0.82rem), detail h3 (0.85rem)
- **Issue:** Several elements use font sizes below the 0.75rem minimum for captions. More broadly, the entire type scale below the h1 lives in a narrow band between 0.65rem and 0.88rem — folder names and session titles feel like the same hierarchy level. Detail h3 is actually *smaller* than the session title it's nested under (inverted hierarchy).
- **User impact:** Small text strains readability, especially on high-DPI screens. Compressed scale makes the page feel visually flat — no clear anchors for the eye.
- **Fix:** Bump minimums (badges to 0.7rem, metadata to 0.75rem). Spread the scale: folder names to 1rem–1.05rem, session titles at 0.84rem, detail h3 to at least 0.88rem. Badges can stay small since color and uppercase treatment carry readability.

---

### 14. [DONE] Search input focus indicator inconsistent

- **Category:** Accessibility
- **Source:** UX Audit (Severity 2, #7)
- **Location:** `.search-input:focus` CSS
- **Issue:** The search input's focus style is a border-color change to `--text-3` (a muted gray), with no outline. Other interactive elements (chips, theme toggle, sort select) correctly use `outline: 2px solid var(--progress); outline-offset: 2px` for `:focus-visible`.
- **User impact:** A keyboard user tabbing into the search field may not notice the subtle border change, making it unclear which element has focus.
- **Fix:** Add `:focus-visible` style matching the other controls: `outline: 2px solid var(--progress); outline-offset: 2px;`

---

### 15. [DONE] Expand/collapse bulk buttons are too subtle

- **Category:** Visual Design
- **Source:** UX Audit (Severity 2, #8)
- **Location:** `.bulk-btn` CSS
- **Issue:** The expand/collapse buttons use 0.7rem text, `--text-3` color (the lightest text tier), and a subtle border. They visually read as the least important elements in the controls bar, despite being genuinely useful navigation controls.
- **User impact:** Users may not notice these buttons exist, especially first-time users scanning the controls area. They'll manually toggle each folder instead.
- **Fix:** Bump to same styling as the sort select — slightly larger text (0.75rem), `--text-2` color, matching border radius and padding.

---

### 16. [DONE] Detail-meta line too dense / session ID fragments serve no purpose

- **Category:** Information Design
- **Source:** UX Audit (Severity 2, #9 + Severity 1, #12)
- **Location:** `.detail-meta` divs in HTML template
- **Issue:** The detail-meta line crams 4 pieces of info into one dense string: `"Today 01:13 PM -> Today 02:38 PM . 44 user / 70 assistant . 56fcb4d3-13d"`. The truncated session ID at the end is developer-facing information that most users won't need — and it's not copyable or actionable (can't resume a session without the full ID).
- **User impact:** Users expanding a session must visually parse a dense metadata string. The session ID is noise for most use cases and visual clutter.
- **Fix:** Either split into separate styled spans (time range | message breakdown) or move the session ID behind a "copy ID" button. Consider removing the truncated ID entirely, or making it click-to-copy with the full ID.

---

### 17. No responsive/mobile viewport handling

- **Category:** Accessibility
- **Source:** UX Audit (Severity 2, #10)
- **Location:** `<head>` section of HTML template — missing `<meta name="viewport">`
- **Issue:** No `<meta name="viewport" content="width=device-width, initial-scale=1">` in the `<head>`. The session row layout works well on desktop, but on narrow viewports the controls bar and session rows will not adapt.
- **User impact:** If a user opens this report on a tablet or phone (e.g., reviewing sessions on the go), the page won't scale properly and may require zooming/panning.
- **Fix:** Add viewport meta tag and add a media query at ~768px to stack the controls and adjust session row layout.

---

### 18. [DONE] No `.gitignore`

- **Category:** Project Setup
- **Source:** Code Review (Important #5)
- **Location:** Repository root
- **Issue:** There is no `.gitignore` file. Python `__pycache__/`, `.pyc` files, or editor temp files could accidentally be committed.
- **User impact:** Repository hygiene — unwanted files could end up in commits.
- **Fix:** Add a standard Python `.gitignore`.

---

## P3 — Low

### 19. [DONE] Footer exposes raw filesystem path

- **Category:** Privacy / Polish
- **Source:** Code Review (#11) + UX Audit (Severity 1, #11) + Design Critique (minor)
- **Location:** Footer element in HTML template
- **Issue:** Footer shows the full cache file path — a system implementation detail. Acceptable for a local tool, but would leak information if reports are ever shared.
- **User impact:** Cosmetic — visual noise for normal viewing, though useful for debugging.
- **Fix:** Either remove or wrap in a `<details>` toggle for "technical details."

---

### 20. [DONE] No tooltips explaining status meanings

- **Category:** Help / Documentation
- **Source:** UX Audit (Severity 1, #13)
- **Location:** Status badges (`.badge`) and status strip (`.status-group`)
- **Issue:** Users unfamiliar with the workflow won't know what "Handed Off" means (session ended with continuation prompt for new session).
- **User impact:** Minor confusion for first-time viewers or anyone sharing this report.
- **Fix:** Add `title` attributes to status badges and status-strip dots with brief definitions.

---

### 21. [DONE] No keyboard shortcuts for common actions

- **Category:** Efficiency
- **Source:** UX Audit (Severity 1, #14)
- **Location:** JavaScript section of HTML template
- **Issue:** No keyboard shortcuts for search focus (Ctrl+F or /), Escape to clear search/filters, or arrow-key navigation through sessions.
- **User impact:** Power users must mouse to every control. For a developer-facing tool, keyboard shortcuts would feel natural.
- **Fix:** Add at minimum: `/` or `Ctrl+K` to focus search, `Escape` to clear search.

---

### 22. [DONE] Search input fixed width is too narrow

- **Category:** Visual Design
- **Source:** Design Critique (minor)
- **Location:** `.search-input` CSS (170px fixed width)
- **Issue:** Long search terms get truncated in the input. The fixed width doesn't adapt to available space.
- **User impact:** Users can't see their full search term.
- **Fix:** Change to `min-width: 170px` with some flex growth.

---

### 23. [DONE] Session chevron and folder arrow sizes inconsistent

- **Category:** Visual Design
- **Source:** UX Audit (Severity 1, #15) + Design Critique (minor)
- **Location:** `.session-chevron` at 8px, `.folder-arrow` at 10px
- **Issue:** The expand chevron on session rows is 8px — barely visible, especially at distance. The folder arrow uses a different size (10px) for the same interaction pattern.
- **User impact:** Users may not notice the chevron indicating expandability. Inconsistent sizing for the same affordance.
- **Fix:** Normalize to one size (10px) and consider increasing the clickable area.

---

### 24. [DONE] "1 sessions" pluralization bug

- **Category:** Code Quality
- **Source:** Design Critique (minor)
- **Location:** f-string in HTML template using `{len(sessions)} sessions`
- **Issue:** Doesn't pluralize correctly — shows "1 sessions" instead of "1 session".
- **User impact:** Minor grammar issue.
- **Fix:** Use `f"{len(sessions)} session{'s' if len(sessions) != 1 else ''}"` or similar.

---

### 25. `decode_project_path` edge case with `--` in directory names

- **Category:** Code Quality
- **Source:** Code Review (Minor #7)
- **Location:** `claude-session-report.py:193-205`
- **Issue:** If a Unix directory name contains `--` (e.g., `/home/user/my--project`), the encoded form would produce paths with double slashes. Inherent limitation of Claude Code's encoding scheme.
- **User impact:** Rare edge case — most directory names don't contain `--`.
- **Fix:** Document the limitation. No practical fix since it's Claude Code's encoding.

---

### 26. [DONE] `REPORT_DIR.mkdir()` repeated 4 times

- **Category:** Code Quality
- **Source:** Code Review (Minor #8)
- **Location:** `claude-session-report.py:1299, 1312, 1328, 1341`
- **Issue:** The same `mkdir(parents=True, exist_ok=True)` call is made in 4 separate code paths.
- **User impact:** None — purely a code cleanliness issue.
- **Fix:** Do it once at the top of `main()` when any output-producing mode is detected.

---

### 27. `format_timestamp` and `format_timestamp_short` share duplicated logic

- **Category:** Code Quality
- **Source:** Code Review (Minor #9)
- **Location:** `claude-session-report.py:126-149`
- **Issue:** Both functions have nearly identical today/yesterday/other branching.
- **User impact:** None — code duplication.
- **Fix:** Share a helper with a format parameter.

---

### 28. `--limit` applies after full collection

- **Category:** Performance
- **Source:** Code Review (Minor #10)
- **Location:** `claude-session-report.py:1305-1307`
- **Issue:** The `--limit` flag truncates the session list after all sessions have been collected and sorted. All files are still read and parsed even if only 10 are needed.
- **User impact:** Negligible for typical use. Could matter at scale (hundreds of sessions).
- **Fix:** Early termination during collection. Low priority given the tool's local nature.

---

### 29. Transcript sampling could miss critical context

- **Category:** Design Limitation
- **Source:** Code Review (Minor #12)
- **Location:** `claude-session-report.py:400-427`
- **Issue:** The sampling strategy (first 6, middle 8, last 6) could miss critical turning points in long sessions.
- **User impact:** Summaries of very long sessions might miss important context. Reasonable tradeoff for token limits.
- **Fix:** Consider a smarter sampling strategy (e.g., weighting messages near topic changes), but current approach is pragmatic.

---

### 30. No print stylesheet

- **Category:** Accessibility / Output
- **Source:** Design Critique (minor)
- **Location:** HTML template CSS
- **Issue:** If someone wants to print or PDF this report (e.g., for a manager or timesheet), the dark theme, interactive controls, and collapsed sections won't translate well.
- **User impact:** Edge case — most users won't print this. But PDF export could be useful for sharing.
- **Fix:** Add a `@media print` stylesheet that forces light theme, expands all sections, and hides interactive controls.

---

### 31. Add "abandoned" session status

- **Category:** Feature
- **Source:** User request
- **Location:** `STATUS_LABELS`, `STATUS_ORDER`, `STATUS_TOOLTIPS`, summarization prompt in `session-report.md`, HTML template (CSS, filter chips, status strip)
- **Issue:** There's no status for sessions that were started but intentionally dropped — e.g., a wrong approach, an exploratory spike that went nowhere, or a session interrupted and never resumed. These currently show as "in progress" or "handed off" forever, cluttering the active view.
- **User impact:** Stale sessions pollute the dashboard with false signals of active work. Users can't distinguish "still working on it" from "gave up on this."
- **Fix:** Add `abandoned` as a valid status alongside the existing five. Needs: a color/badge style (muted red or gray-red), a filter chip, a status-strip dot, inclusion in `STATUS_ORDER` (between `handed_off` and `complete`), a tooltip definition, and updated summarization instructions in `session-report.md` so Claude knows when to assign it.

---

### 32. Include changed files in session summaries

- **Category:** Feature
- **Source:** Dashboard redesign brainstorming (2026-03-20)
- **Location:** Summarization prompt in `session-report.md`, HTML template (session expansion)
- **Issue:** The redesigned dashboard's session expansion view could show which files were changed during a session, but the current AI summaries don't reliably include file lists.
- **User impact:** Users expanding a session in the timeline can't see at a glance which files were touched — they only get the summary text.
- **Fix:** Update the summarization prompt in `session-report.md` to instruct Claude to include a "Files changed:" section listing the key files modified during the session. The HTML template would then render these as inline code blocks in the session expansion.

---

## Reviewer Gap Findings

Issues identified during the cross-document merge that none of the three reviews caught:

### G1. HTML template f-string XSS surface not audited

- **Category:** Security
- **Issue:** The code review praises the `md_to_html` escape ordering and flags the XSS bug in the *old* setup doc version, but doesn't audit the f-string template injection surface in the *current* script. Session titles, folder paths, and other user-derived strings are interpolated into the HTML template. While `md_to_html` escapes correctly, not all template interpolations may go through that function.
- **Recommendation:** Audit all f-string interpolations in the HTML template to confirm every user-derived value is escaped.

### G2. Slash command prompt (`session-report.md`) not reviewed

- **Category:** Documentation / Design
- **Issue:** The slash command is ~150 lines of orchestration logic that instructs Claude Code how to run the pipeline. None of the three reviews evaluate whether the prompt's instructions are correct, complete, or could produce bad summaries.
- **Recommendation:** Review `session-report.md` for correctness, especially the summarization format instructions and the status review pass logic.

### G3. No performance profiling at scale

- **Category:** Performance
- **Issue:** None of the reviews measure load time, DOM size, or rendering cost. The single-file HTML with inline CSS/JS and 56 sessions is likely fine, but at 200+ sessions the page could become heavy.
- **Recommendation:** Test with a synthetic dataset of 200+ sessions and measure rendering performance.
