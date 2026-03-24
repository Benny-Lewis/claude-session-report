# UX Design Audit Report — Session Report

**Date:** 2026-03-20
**Scope:** Session report HTML page — header, status strip, filter controls, folder/session tree, session detail panels, theme toggle, search, sort, and all JavaScript interactions.
**Source:** `C:\Users\Ben\.claude\reports\session-report-20260320-150533.html` (1859 lines — single file with inline CSS + JS)
**Interface type:** Dashboard / activity log viewer

---

## How to Read This Report

Findings are rated on a 0-4 severity scale (4 = users can't complete tasks, 1 = cosmetic only). Each finding references an established usability principle. Start from the top — the most impactful issues are listed first.

---

## Summary

| Severity | Count |
|----------|-------|
| 4 - Catastrophe | 1 |
| 3 - Major | 2 |
| 2 - Minor | 7 |
| 1 - Cosmetic | 5 |
| **Total findings** | **15** |

---

## Quick Wins

1. **Non-semantic interactive elements** (Severity 4) — Change folder headers and session rows from `<div onclick>` to `<button>` or add `role="button"` + `tabindex="0"` + keydown handler
2. **Inconsistent "Unknown" vs "Unsummarized" labels** (Severity 2) — Pick one term and use it everywhere
3. **Missing `aria-expanded`** (Severity 3) — Add to all collapsible triggers (folder headers, session rows)

---

## Findings

### [Severity 4] Folder headers and session rows are keyboard-inaccessible

- **Principle:** Accessibility (H13)
- **Location:** `.folder-header` (div with onclick), `.session-row` (div with onclick)
- **Issue:** All 56 session rows and 13 folder headers are `<div>` elements with inline `onclick`. They have no `role="button"`, no `tabindex="0"`, and no `onkeydown` handler. Keyboard-only users cannot Tab to or activate any of these — the entire content tree is unreachable without a mouse.
- **User impact:** A keyboard-only user (motor disability, broken mouse, power user navigating with Tab) cannot expand any folder or read any session detail. The page is essentially a static header bar for them.
- **Fix:** Either change these to `<button>` elements, or add `role="button" tabindex="0"` and a keydown handler that triggers on Enter/Space. The `<button>` approach is simpler and inherits keyboard behavior for free.

---

### [Severity 3] No `aria-expanded` on collapsible elements

- **Principle:** Accessibility (H13)
- **Location:** Folder headers, session rows
- **Issue:** Folders and sessions toggle between collapsed/expanded states via CSS class (`.open`, `.expanded`), but there's no `aria-expanded` attribute to communicate state to screen readers. The grid-template-rows animation is purely visual.
- **User impact:** A screen reader user can't tell whether a folder or session is open or closed — they hear the text but get no state information.
- **Fix:** Add `aria-expanded="false"` to each trigger element, and toggle it to `"true"` in `toggleFolder()` and `toggleSession()`.

---

### [Severity 3] Heading hierarchy skips levels

- **Principle:** Accessibility (H13), Structure (H12)
- **Location:** `.detail-body h3`, `.detail-body h4`
- **Issue:** The page has one `<h1>` ("Session Report") and then jumps directly to `<h3>` and `<h4>` inside session details. There are no `<h2>` elements. Folder names (`.folder-path`) are styled as bold text but are `<span>` elements, not headings.
- **User impact:** Screen reader users navigating by headings will find a broken hierarchy — jumping from h1 to h3 is disorienting and makes it harder to build a mental model of the page structure.
- **Fix:** Either make folder names `<h2>` and change detail headings to `<h3>`/`<h4>`, or change the summary headings from `<h3>`/`<h4>` to `<h2>`/`<h3>` since they're the next structural level below the page title.

---

### [Severity 2] "Unknown" badge vs "Unsummarized" chip — inconsistent terminology

- **Principle:** Consistency and Standards (H4)
- **Location:** Filter chip says "Unsummarized", badge says "Unknown"
- **Issue:** The same concept — sessions that haven't been AI-summarized yet — is called "Unsummarized" in the filter chip bar and "Unknown" on the session badge. Users encountering both will wonder if these are different categories.
- **User impact:** Mild confusion — "Does 'Unknown' mean something different from 'Unsummarized'?" Forces the user to mentally map two labels to one concept.
- **Fix:** Use "Unsummarized" consistently in both places. It's more descriptive than "Unknown" and explains *why* the status isn't known.

---

### [Severity 2] Folder status pills show numbers without context

- **Principle:** Perceptibility (H14)
- **Location:** `.fpill` elements in folder meta
- **Issue:** Folder meta shows colored number pills (e.g., a purple `9` next to `~\dev\career`) but the number has no label. The color corresponds to a status (handed_off = purple, in_progress = blue) but the user has to cross-reference the color with the status strip to decode meaning. Not all statuses get pills — only `handed_off` and `in_progress` appear.
- **User impact:** "What does the purple 9 mean next to this folder?" Users must visually color-match against the status strip legend. First-time viewers will not understand these pills.
- **Fix:** Add a `title` attribute (e.g., `title="9 handed off"`) for tooltip context, or display the count with a short label like `9 handed off`.

---

### [Severity 2] Very small font sizes strain readability

- **Principle:** Aesthetic and Minimalist Design (H8)
- **Location:** `.badge` at 0.65rem, `.session-msgs` at 0.7rem, `.session-time` at 0.73rem, `.folder-info` at 0.73rem, `.bulk-btn` at 0.7rem
- **Issue:** Several UI elements use font sizes at or below 0.7rem (~11px). The badge text at 0.65rem (~10.4px) with uppercase and letter-spacing is particularly small. These are below the recommended 0.75rem minimum for captions.
- **User impact:** Users at typical viewing distances, especially on high-DPI screens at non-native scaling, may struggle to read status badges, timestamps, and message counts.
- **Fix:** Bump minimum sizes — badges to 0.7rem, metadata to 0.75rem. The information is useful, so it should be comfortably readable.

---

### [Severity 2] Search input uses subtle border change instead of proper focus indicator

- **Principle:** Accessibility (H13)
- **Location:** `.search-input:focus { border-color: var(--text-3); }`
- **Issue:** The search input's focus style is a border-color change to `--text-3` (a muted gray), with no outline. Other interactive elements (chips, theme toggle, sort select) correctly use `outline: 2px solid var(--progress); outline-offset: 2px` for `:focus-visible`. The search input is inconsistent.
- **User impact:** A keyboard user tabbing into the search field may not notice the subtle border change, making it unclear which element has focus.
- **Fix:** Add `:focus-visible` style matching the other controls: `outline: 2px solid var(--progress); outline-offset: 2px;`

---

### [Severity 2] Expand/collapse bulk buttons are too subtle

- **Principle:** Affordances and Signifiers (H11)
- **Location:** `.bulk-btn` styles
- **Issue:** The expand/collapse buttons use 0.7rem text, `--text-3` color (the lightest text tier), and a subtle border. They visually read as the least important elements in the controls bar, despite being genuinely useful navigation controls.
- **User impact:** Users may not notice these buttons exist, especially first-time users scanning the controls area. They'll manually toggle each folder instead.
- **Fix:** Bump to same styling as the sort select — slightly larger text (0.75rem), `--text-2` color, matching border radius and padding.

---

### [Severity 2] Detail-meta line packs too much info without structure

- **Principle:** Aesthetic and Minimalist Design (H8), Perceptibility (H14)
- **Location:** `.detail-meta` divs
- **Issue:** The detail-meta line crams 4 pieces of info into one dense string: `"Today 01:13 PM -> Today 02:38 PM . 44 user / 70 assistant . 56fcb4d3-13d"`. The session ID fragment at the end is developer-facing information that most users won't need.
- **User impact:** Users expanding a session to read the summary must visually parse a dense metadata string. The session ID is noise for most use cases.
- **Fix:** Either split into separate styled spans (time range | message breakdown) or move the session ID behind a "copy ID" button. Consider whether the truncated session ID serves any purpose in the UI.

---

### [Severity 2] No responsive/mobile viewport handling

- **Principle:** Accessibility (H13), Structure (H12)
- **Location:** `<head>` section — missing `<meta name="viewport">`
- **Issue:** There's no `<meta name="viewport" content="width=device-width, initial-scale=1">` in the `<head>`. The session row layout (`display: flex` with `white-space: nowrap; overflow: hidden; text-overflow: ellipsis`) works well on desktop, but on narrow viewports the controls bar and session rows will not adapt.
- **User impact:** If a user opens this report on a tablet or phone (e.g., reviewing sessions on the go), the page won't scale properly and may require zooming/panning.
- **Fix:** Add viewport meta tag and add a media query at ~768px to stack the controls and adjust session row layout.

---

### [Severity 1] Footer exposes raw file path

- **Principle:** Match Between System and Real World (H2)
- **Location:** Footer element
- **Issue:** Footer shows `Cache: C:\Users\Ben\.claude\session-report-cache.json` — a system implementation detail.
- **User impact:** Cosmetic — the raw path is noise for normal viewing, though useful for debugging.
- **Fix:** Either remove or wrap in a `<details>` toggle for "technical details."

---

### [Severity 1] Session ID fragments in detail-meta serve no purpose

- **Principle:** Match Between System and Real World (H2)
- **Location:** Detail-meta divs (e.g., `4d2843bf-c79`, `b8c96f32-eaf`)
- **Issue:** Truncated UUIDs are shown but not copyable or actionable. They can't be used to resume sessions without the full ID.
- **User impact:** Visual clutter in an otherwise clean detail panel.
- **Fix:** Either make them copyable (click-to-copy full ID) or remove them.

---

### [Severity 1] No tooltips explaining status meanings

- **Principle:** Help and Documentation (H10)
- **Location:** Status badges (`.badge`) and status strip (`.status-group`)
- **Issue:** Users unfamiliar with the workflow won't know what "Handed Off" means (session ended with continuation prompt for new session).
- **User impact:** Minor confusion for first-time viewers or anyone sharing this report.
- **Fix:** Add `title` attributes to status badges and status-strip dots with brief definitions.

---

### [Severity 1] No keyboard shortcuts for common actions

- **Principle:** Flexibility and Efficiency (H7)
- **Location:** JavaScript section
- **Issue:** No keyboard shortcuts for search focus (Ctrl+F or /), Escape to clear search/filters, or arrow-key navigation through sessions.
- **User impact:** Power users must mouse to every control. For a developer-facing tool, keyboard shortcuts would feel natural.
- **Fix:** Add at minimum: `/` or `Ctrl+K` to focus search, `Escape` to clear search, consider arrow keys for session navigation.

---

### [Severity 1] Session chevron is very small

- **Principle:** Affordances and Signifiers (H11)
- **Location:** `.session-chevron` at 8px font-size
- **Issue:** The expand chevron on session rows is 8px — barely visible, especially at distance.
- **User impact:** Users may not notice the chevron indicating expandability. The cursor-pointer on hover helps, but the static signifier is weak.
- **Fix:** Bump to 10px and consider increasing the clickable area around it.

---

## Strengths

1. **Excellent theme system** — The OKLCH color system with full light/dark support is well-implemented. `prefers-reduced-motion` is handled. Theme preference persists via localStorage. The semantic color tokens (progress, blocked, handed, complete) maintain meaning across both themes. This is better than most production dashboards.

2. **Smart auto-expand logic** — Folders with active sessions (in_progress, blocked, handed_off) auto-expand on load while completed-only folders stay collapsed. This respects the user's attention — you see what matters without manual hunting.

3. **Clean information architecture** — The folder > session > detail hierarchy mirrors how the data is actually organized (by project directory). Status counts in the strip, filter chips with counts, folder-level metadata — each level of the UI provides progressively more detail. The empty state is handled.

4. **Forgiving search** — Case-insensitive substring matching across both session content and folder names. No exact-match frustration.

5. **Purposeful color usage** — Four status colors are used consistently and meaningfully throughout (badges, pills, status strip dots, filter chips). The accent colors are reserved for status — the UI chrome stays neutral. This discipline prevents the "everything is colorful so nothing stands out" problem.
