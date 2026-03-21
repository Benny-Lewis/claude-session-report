# Dashboard Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the flat session log HTML output with a project-first master-detail layout (sidebar + detail panel).

**Architecture:** The existing single-file Python script (`claude-session-report.py`) generates self-contained HTML. This plan replaces the `generate_html()` function and adds supporting helpers, keeping all other pipeline logic (collection, caching, summarization, text output, CLI) unchanged. No new files are created — all changes are in `claude-session-report.py`.

**Tech Stack:** Python 3 (stdlib only), inline HTML/CSS/JS, OKLCH colors, Google Fonts (Outfit + Source Serif 4)

**Spec:** `docs/superpowers/specs/2026-03-20-dashboard-redesign-design.md`
**Mockup:** `.superpowers/brainstorm/144780-1774063179/full-design-v3.html`

---

## File Structure

All changes are in one file:
- **Modify:** `claude-session-report.py`
  - Add: `aggregate_projects()` (~line 620, before HTML output section)
  - Add: `parse_summary_sections()` (~line 620, before HTML output section)
  - Add: `read_corrections_from_report()` (~line 620, before HTML output section)
  - Replace: `generate_html()` (currently lines ~662-1304) — complete rewrite
  - Modify: `main()` (~line 1412) — wire in corrections reading

No new files. The text output (`print_text_report`) and all pipeline subcommands (`--collect`, `--update-cache`, `--review-statuses`, `--update-statuses`) remain unchanged.

---

## Task 1: Add project-level aggregation helper

**Files:**
- Modify: `claude-session-report.py` — add function before the HTML output section (~line 620)

This function groups sessions by project and computes per-project metadata needed by the new template: project name, status, session count, last active time, most recent session title, active/inactive classification.

- [ ] **Step 1: Add `aggregate_projects()` function**

Insert before the `# ─── HTML output` section comment. The function takes the same inputs as `generate_html()` (all_sessions, summaries) and returns a structured list of project dicts.

```python
def aggregate_projects(all_sessions, summaries, days):
    """Group sessions by project with computed metadata for the sidebar."""
    by_project = defaultdict(list)
    now = datetime.now()
    inactive_threshold = now - timedelta(hours=24)

    for s in all_sessions:
        sid = s["session_id"]
        s_data = summaries.get(sid, {})
        if isinstance(s_data, dict):
            s["_summary"] = s_data.get("summary", "")
            s["_status"] = s_data.get("status", "unknown")
            s["_title"] = s_data.get("title") or extract_title_from_summary(s_data.get("summary", ""))
        else:
            s["_summary"] = ""
            s["_status"] = "unknown"
            s["_title"] = None
        if not s["_title"]:
            s["_title"] = truncate(s["first_ask"], 80)
        by_project[s["cwd"] or s["project_path"]].append(s)

    projects = []
    for folder, sessions in by_project.items():
        sessions.sort(key=lambda x: x["latest"], reverse=True)
        latest_session = sessions[0]
        status_counts = defaultdict(int)
        for s in sessions:
            status_counts[s["_status"]] += 1

        # Project-level status: heuristic for v1 — priority: blocked > in_progress > handed_off > latest session
        # NOTE: Spec calls for AI-synthesized project status via --review-statuses pipeline.
        # This mechanical heuristic is a v1 simplification. A follow-up should update
        # session-report.md prompt to produce a project_status field during the review pass.
        project_status = latest_session["_status"]
        for s in sessions:
            if s["_status"] == "blocked":
                project_status = "blocked"
                break
            if s["_status"] == "in_progress":
                project_status = "in_progress"
                break
            if s["_status"] == "handed_off" and project_status not in ("in_progress", "blocked"):
                project_status = "handed_off"

        # Active/inactive: inactive if all sessions complete and latest > 24h ago
        all_complete = all(s["_status"] in ("complete", "unknown") for s in sessions)
        latest_time = max(s["latest"] for s in sessions)
        is_inactive = all_complete and latest_time < inactive_threshold

        # Short name: last component of the path
        short_name = short_path(folder).rstrip("/\\").split("/")[-1].split("\\")[-1]
        # Initials: first two chars of first two words, or first two chars
        parts = re.split(r'[-_]', short_name)
        if len(parts) >= 2:
            initials = (parts[0][0] + parts[1][0]).upper()
        else:
            initials = short_name[:2].upper()

        projects.append({
            "folder": folder,
            "short_path": short_path(folder),
            "name": short_name,
            "initials": initials,
            "sessions": sessions,
            "session_count": len(sessions),
            "latest": latest_time,
            "status": project_status,
            "status_counts": dict(status_counts),
            "is_inactive": is_inactive,
            "last_title": latest_session["_title"],
        })

    # Sort: active first (by recency), then inactive (by recency)
    active = sorted([p for p in projects if not p["is_inactive"]], key=lambda p: p["latest"], reverse=True)
    inactive = sorted([p for p in projects if p["is_inactive"]], key=lambda p: p["latest"], reverse=True)

    return active, inactive
```

- [ ] **Step 2: Add `format_duration()` helper**

Insert near the other format functions (`format_timestamp`, `format_timestamp_short`):

```python
def format_duration(start, end):
    """Format duration between two datetimes as human-readable string."""
    if not start or not end:
        return ""
    delta = end - start
    total_minutes = int(delta.total_seconds() / 60)
    if total_minutes < 60:
        return f"{total_minutes}m"
    hours = total_minutes // 60
    minutes = total_minutes % 60
    if minutes == 0:
        return f"{hours}h"
    return f"{hours}h {minutes}m"
```

- [ ] **Step 3: Verify the function runs without errors**

Run: `python claude-session-report.py 3 no-ai --no-open`

This won't call the new functions yet (they're not wired in), but ensures the script still parses correctly with the new code added.

- [ ] **Step 4: Commit**

```bash
git add claude-session-report.py
git commit -m "Add aggregate_projects() and format_duration() helpers"
```

---

## Task 2: Add summary section parser

**Files:**
- Modify: `claude-session-report.py` — add function near the markdown conversion section (~line 620)

Parses AI summary markdown into structured sections (What Was Done, Key Decisions, Issues Encountered) for color-coded block rendering. Falls back to rendering the full summary as a single block if sections aren't found.

- [ ] **Step 1: Add `parse_summary_sections()` function**

Insert after `md_to_html()`:

```python
# Section accent colors (muted tinted neutrals — distinct from status colors)
SECTION_STYLES = {
    "what was done": ("oklch(0.55 0.03 55)", "done"),
    "what was accomplished": ("oklch(0.55 0.03 55)", "done"),
    "summary": ("oklch(0.55 0.03 55)", "done"),
    "key decisions": ("oklch(0.50 0.025 200)", "decisions"),
    "decisions made": ("oklch(0.50 0.025 200)", "decisions"),
    "decisions": ("oklch(0.50 0.025 200)", "decisions"),
    "issues encountered": ("oklch(0.52 0.035 25)", "issues"),
    "issues": ("oklch(0.52 0.035 25)", "issues"),
    "blockers": ("oklch(0.52 0.035 25)", "issues"),
    "next steps": ("oklch(0.50 0.03 130)", "next"),
    "what's next": ("oklch(0.50 0.03 130)", "next"),
}

# Sections that should be rendered as the "Next Steps" card instead of a color-coded block
NEXT_STEPS_SECTIONS = {"next steps", "what's next"}

def parse_summary_sections(summary_text):
    """Parse summary markdown into structured sections for color-coded rendering.

    Returns list of {label, css_class, color, html_content} dicts.
    Falls back to a single 'Summary' section if no recognized headings found.
    """
    if not summary_text or summary_text.startswith("("):
        return []

    lines = summary_text.split("\n")
    sections = []
    current_label = None
    current_lines = []

    for line in lines:
        stripped = line.strip()
        heading_match = re.match(r'^#{1,3}\s+(.+)$', stripped)
        if heading_match:
            # Save previous section
            if current_label is not None and current_lines:
                sections.append((current_label, "\n".join(current_lines)))
            current_label = heading_match.group(1).strip()
            current_lines = []
        else:
            current_lines.append(line)

    # Save final section
    if current_label is not None and current_lines:
        sections.append((current_label, "\n".join(current_lines)))

    # If no headings found, treat entire text as one section
    if not sections:
        return [{
            "label": "Summary",
            "css_class": "done",
            "color": "oklch(0.55 0.03 55)",
            "html": md_to_html(summary_text),
        }]

    result = []
    next_steps = None
    for label, content in sections:
        label_lower = label.lower().strip()
        color, css_class = SECTION_STYLES.get(label_lower, ("oklch(0.55 0.03 55)", "done"))
        html = md_to_html(content)
        if not html.strip():
            continue
        entry = {"label": label, "css_class": css_class, "color": color, "html": html}
        # Separate "Next Steps" for special card rendering
        if label_lower in NEXT_STEPS_SECTIONS:
            next_steps = entry
        else:
            result.append(entry)

    if not result:
        result = [{"label": "Summary", "css_class": "done",
                    "color": "oklch(0.55 0.03 55)", "html": md_to_html(summary_text)}]

    return result, next_steps  # next_steps may be None
```

- [ ] **Step 2: Verify script still runs**

Run: `python claude-session-report.py 3 no-ai --no-open`

- [ ] **Step 3: Commit**

```bash
git add claude-session-report.py
git commit -m "Add parse_summary_sections() for structured summary rendering"
```

---

## Task 3: Add status corrections reader

**Files:**
- Modify: `claude-session-report.py` — add function near the cache section (~line 95)

Reads the most recent report HTML from `~/.claude/reports/`, extracts the embedded `#status-corrections` JSON block, and applies corrections to the cache.

- [ ] **Step 1: Add `read_corrections_from_previous_report()` function**

Insert after the cache functions:

```python
def read_corrections_from_previous_report():
    """Read status corrections embedded in the most recent report HTML.

    Returns dict of {session_id: new_status} or empty dict.
    """
    if not REPORT_DIR.exists():
        return {}
    reports = sorted(REPORT_DIR.glob("session-report-*.html"), key=lambda f: f.stat().st_mtime, reverse=True)
    if not reports:
        return {}

    try:
        content = reports[0].read_text(encoding="utf-8")
    except OSError:
        return {}

    # Extract JSON from <script type="application/json" id="status-corrections">
    match = re.search(
        r'<script\s+type="application/json"\s+id="status-corrections">\s*(\{.*?\})\s*</script>',
        content, re.DOTALL
    )
    if not match:
        return {}

    try:
        corrections = json.loads(match.group(1))
        if isinstance(corrections, dict):
            # Validate entries
            valid = {}
            for sid, status in corrections.items():
                if isinstance(status, str) and status in STATUS_LABELS:
                    valid[sid] = status
            if valid:
                print(f"  Found {len(valid)} status correction(s) from previous report")
            return valid
    except json.JSONDecodeError:
        pass

    return {}


def apply_corrections_to_cache(corrections):
    """Apply user corrections to the summary cache."""
    if not corrections:
        return
    cache = load_summary_cache()
    updated = 0
    for sid, new_status in corrections.items():
        if sid in cache and cache[sid].get("status") != new_status:
            cache[sid]["status"] = new_status
            updated += 1
    if updated > 0:
        save_summary_cache(cache)
        print(f"  Applied {updated} status correction(s) to cache")
```

- [ ] **Step 2: Verify script still runs**

Run: `python claude-session-report.py 3 no-ai --no-open`

- [ ] **Step 3: Commit**

```bash
git add claude-session-report.py
git commit -m "Add status corrections reader from previous report HTML"
```

---

## Task 4: Replace `generate_html()` — new template

**Files:**
- Modify: `claude-session-report.py` — replace the entire `generate_html()` function (lines ~662-1304)

This is the largest task. The new function uses `aggregate_projects()` and `parse_summary_sections()` to build the master-detail layout. The mockup at `.superpowers/brainstorm/144780-1774063179/full-design-v3.html` is the visual reference.

**Important implementation notes:**
- The HTML template is an f-string — all CSS `{` `}` must be doubled as `{{` `}}`
- All user-derived strings must go through `html_escape()`
- The first active project's detail is rendered inline; switching projects is handled by JS using data attributes
- Session data for all projects is embedded as a JSON blob in a `<script>` tag, and JS renders the detail panel client-side when projects are selected

- [ ] **Step 1: Write the new `generate_html()` function signature and project data preparation**

Replace the existing function. Start with the data preparation (using `aggregate_projects`), project data serialization as JSON for client-side rendering, and the HTML skeleton.

```python
def generate_html(all_sessions, summaries, days):
    active_projects, inactive_projects = aggregate_projects(all_sessions, summaries, days)
    all_projects = active_projects + inactive_projects

    if not all_projects:
        # Minimal empty-state page
        return "<!DOCTYPE html><html><body><h1>No sessions found</h1></body></html>"

    now_str = datetime.now().strftime("%b %d, %Y %I:%M %p")
    total_sessions = sum(p["session_count"] for p in all_projects)
    total_projects = len(all_projects)

    # Prepare per-project session data as JSON for client-side rendering
    projects_json = []
    for p in all_projects:
        sessions_data = []
        for s in p["sessions"]:
            sections, next_steps = parse_summary_sections(s["_summary"])
            sections_html = ""
            for sec in sections:
                sections_html += f'<div class="summary-block"><div class="summary-bar" style="background:{sec["color"]}"></div><div><div class="summary-block-label" style="color:{sec["color"]}">{html_escape(sec["label"])}</div><div class="summary-block-content">{sec["html"]}</div></div></div>'

            next_steps_html = next_steps["html"] if next_steps else ""

            # Unsummarized fallback
            if not sections and not s["_summary"]:
                raw_parts = [f'<p><strong>Opening ask:</strong> {html_escape(truncate(s["first_ask"], 300))}</p>']
                if s["last_assistant"]:
                    clean = clean_transcript_text(s["last_assistant"])
                    if clean and len(clean) > 10:
                        raw_parts.append(f'<p><strong>Last response:</strong> {html_escape(truncate(clean, 300))}</p>')
                sections_html = "\n".join(raw_parts)

            sessions_data.append({
                "id": s["session_id"],
                "title": s["_title"],
                "status": s["_status"],
                "summary_html": sections_html,
                "next_steps_html": next_steps_html,
                "time": format_timestamp_short(s["latest"]),
                "time_full": format_timestamp(s["latest"]),
                "msgs": s["total_messages"],
                "user_msgs": s["user_msg_count"],
                "assistant_msgs": s["assistant_msg_count"],
                "duration": format_duration(s["earliest"], s["latest"]) if s["earliest"] and s["latest"] else "",
            })

        projects_json.append({
            "folder": p["folder"],
            "short_path": p["short_path"],
            "name": p["name"],
            "initials": p["initials"],
            "status": p["status"],
            "session_count": p["session_count"],
            "latest": p["latest"].isoformat(),
            "last_title": p["last_title"],
            "is_inactive": p["is_inactive"],
            "status_counts": p["status_counts"],
            "sessions": sessions_data,
        })
    # ... (continues in step 2)
```

Note: You'll need to add a `format_duration()` helper near the other format functions:

```python
def format_duration(start, end):
    """Format duration between two datetimes as human-readable string."""
    if not start or not end:
        return ""
    delta = end - start
    total_minutes = int(delta.total_seconds() / 60)
    if total_minutes < 60:
        return f"{total_minutes}m"
    hours = total_minutes // 60
    minutes = total_minutes % 60
    if minutes == 0:
        return f"{hours}h"
    return f"{hours}h {minutes}m"
```

- [ ] **Step 2: Write the CSS portion of the template**

Continue the `generate_html()` function. Copy the CSS from the v3 mockup (`.superpowers/brainstorm/144780-1774063179/full-design-v3.html`), doubling all `{` `}` for f-string compatibility. Add the light theme variables.

The CSS should include:
- Reset, theme variables (dark + light), base styles
- Sidebar styles (expanded, collapsed, project items, avatars, filters, search)
- Detail panel styles (header, summary sections, next steps, timeline)
- Animation keyframes
- Responsive breakpoint at 900px (auto-collapse sidebar)
- `prefers-reduced-motion`: set `animation: none !important; transition-duration: 0.01ms !important` on `*, *::before, *::after` — disables sidebar collapse animation, detail panel fadeUp, timeline expansion, and all hover transitions

Reference the mockup file for exact values. Add a light theme by inverting lightness values in OKLCH:
```css
:root, [data-theme="light"] {{
  --bg:           oklch(0.97 0.005 55);
  --surface:      oklch(0.94 0.007 55);
  --surface-2:    oklch(0.91 0.008 55);
  --surface-hover: oklch(0.88 0.01 55);
  --border:       oklch(0.85 0.008 55);
  --border-2:     oklch(0.78 0.01 55);
  --text:         oklch(0.15 0.02 55);
  --text-2:       oklch(0.35 0.015 55);
  --text-3:       oklch(0.55 0.01 55);
  --text-4:       oklch(0.68 0.008 55);
  /* Status colors — slightly adjusted for light bg */
  --progress:     oklch(0.50 0.18 250);
  --progress-bg:  oklch(0.94 0.04 250);
  --blocked:      oklch(0.55 0.16 70);
  --blocked-bg:   oklch(0.94 0.04 70);
  --handed:       oklch(0.50 0.17 300);
  --handed-bg:    oklch(0.94 0.04 300);
  --complete:     oklch(0.48 0.12 160);
  --complete-bg:  oklch(0.94 0.04 160);
  /* Section accents */
  --section-done:      oklch(0.45 0.03 55);
  --section-decisions: oklch(0.42 0.025 200);
  --section-issues:    oklch(0.44 0.035 25);
  color-scheme: light;
}}
```

- [ ] **Step 3: Write the HTML structure**

Continue the template. Build the sidebar and detail panel HTML using Python loops over the project data. The detail panel's content is populated client-side by JS (using the embedded projects JSON), but the sidebar is rendered server-side.

**Sidebar HTML** — loop over `active_projects` and `inactive_projects` to build project items. Each item includes `data-project-index` for JS selection.

**Detail panel HTML** — render the first active project's detail server-side as the default view. Include:
- Project header (name, path, status badge with `▾`)
- Color-coded summary sections (from `parse_summary_sections`)
- Session metadata line
- Next steps card
- Recent sessions timeline

**Embedded data** — include `<script type="application/json" id="projects-data">{json_data}</script>` for client-side project switching.

**Corrections block** — include empty `<script type="application/json" id="status-corrections">{{}}</script>` for status correction persistence.

- [ ] **Step 4: Write the JavaScript**

The JS must implement these features. Use the mockup at `.superpowers/brainstorm/144780-1774063179/full-design-v3.html` as a starting reference for the simpler interactions (sidebar toggle, project select, timeline expand).

**Core functions to implement:**

1. **`toggleTheme()`** — toggle `data-theme` attribute between `dark`/`light`, persist to localStorage key `sr-theme`, update icon text. Same pattern as current script.

2. **`toggleSidebar()`** — toggle `.collapsed` class on `#sidebar`, persist to localStorage key `sr-sidebar-collapsed`, flip chevron direction.

3. **`selectProject(index)`** — set `.selected` on clicked project item, call `renderProjectDetail(index)` to update the detail panel. Persist selected index to localStorage.

4. **`renderProjectDetail(index)`** — the core function. Reads `projects-data` JSON, finds the project at the given index. Rebuilds `.detail-inner` HTML with:
   - Project header (name in Source Serif 4, path in monospace, status badge with `▾`)
   - Color-coded summary sections from `sessions[0].summary_html` (the most recent session)
   - Session metadata line (time · msgs · duration)
   - Next Steps card from `sessions[0].next_steps_html` (render only if non-empty)
   - Recent Sessions timeline (all sessions, first has glowing dot, each has expandable summary)
   - Detail panel session filter chips: "All N", "In Progress N", "Complete N" — these filter which sessions appear in the timeline for THIS project (independent of sidebar filters)

5. **`toggleTimeline(el)`** — toggle `.expanded` class on a timeline item.

6. **`showStatusDropdown(badgeEl, sessionId)`** — create and position a dropdown with status options. On selection: update badge text/color in DOM, write `{sessionId: newStatus}` to both localStorage (`sr-corrections`) and the `#status-corrections` script block. Use `event.stopPropagation()`.

7. **`filterProjects(status)`** — toggle `.active` on clicked filter chip, show/hide project items based on their `data-status` attribute. Chip options: "All", "Active" (in_progress), "Handed Off", "Blocked" — no "Complete" chip (completed projects are in the inactive section).

8. **`searchProjects(query)`** — filter project items by matching query against project name and last session title. Show/hide items accordingly.

9. **`toggleInactive()`** — toggle `.open` class on inactive section, rotate arrow.

10. **Keyboard shortcuts** — on `keydown`:
    - `/` or `Ctrl+K` (when not in input): `e.preventDefault()`, focus search input
    - `Escape`: clear search, or clear filter, or blur input

11. **Init on load:**
    - Read theme from localStorage, apply
    - Read sidebar state from localStorage, apply
    - Auto-collapse sidebar if `window.innerWidth < 900`
    - Read status corrections from localStorage, merge with `#status-corrections` block, apply to DOM
    - Select most recent project (index 0) or last-selected from localStorage

- [ ] **Step 5: Close the function — return the assembled HTML string**

The function should end with:
```python
    return html
```

- [ ] **Step 6: Verify the new template renders**

Run: `python claude-session-report.py 3 no-ai --no-open`
Then open the generated HTML file manually to verify it renders.

Run: `python claude-session-report.py 3 --no-open`
(with cached summaries) to verify summarized content renders with color-coded sections.

- [ ] **Step 7: Commit**

```bash
git add claude-session-report.py
git commit -m "Replace generate_html() with master-detail layout"
```

---

## Task 5: Wire corrections into the pipeline

**Files:**
- Modify: `claude-session-report.py:main()` (~line 1412)

Before loading summaries and rendering, read and apply corrections from the previous report.

- [ ] **Step 1: Add corrections reading to main()**

In `main()`, after the `REPORT_DIR.mkdir()` call and before `summaries = load_cached_summaries(...)`, add:

```python
    # Apply any status corrections from previous report
    corrections = read_corrections_from_previous_report()
    apply_corrections_to_cache(corrections)
```

- [ ] **Step 2: Verify the full pipeline**

Run: `python claude-session-report.py 3 --no-open`
Check that:
1. The report generates without errors
2. If corrections exist in a previous report, they're applied
3. The output HTML includes the empty `#status-corrections` block

- [ ] **Step 3: Commit**

```bash
git add claude-session-report.py
git commit -m "Wire status corrections from previous report into pipeline"
```

---

## Task 6: Visual verification and polish

**Files:**
- Modify: `claude-session-report.py` — fixes found during verification

- [ ] **Step 1: Generate a full report with AI summaries**

Run: `python claude-session-report.py 3 --no-open`

Open the generated HTML in a browser. Verify:
- Sidebar shows projects grouped into active/inactive
- Detail panel shows the most recent project's context
- Clicking projects in the sidebar updates the detail panel
- Sidebar collapses to initials + status dots
- Timeline sessions are expandable
- Status badges show dropdown arrows
- Light/dark theme toggle works
- Search filters projects
- Filter chips work
- Keyboard shortcuts (`/` for search, `Escape` to clear) work

- [ ] **Step 2: Test with no-ai flag**

Run: `python claude-session-report.py 3 no-ai --no-open`

Verify that unsummarized sessions render properly (opening ask / last response fallback).

- [ ] **Step 3: Test text output is unchanged**

Run: `python claude-session-report.py 3 text`

Verify terminal text output still works correctly (it should be completely unaffected).

- [ ] **Step 4: Test responsive behavior**

Open the report and resize the browser window below 900px. Verify:
- Sidebar auto-collapses to icon view
- Detail panel fills the width

- [ ] **Step 5: Fix any issues found, commit**

```bash
git add claude-session-report.py
git commit -m "Polish dashboard redesign after visual verification"
```

---

## Task 7: Update summarization prompt for structured sections

**Files:**
- Modify: `session-report.md` (the slash command prompt)

The summary section parser relies on AI summaries having `## What Was Done`, `## Key Decisions`, `## Issues Encountered`, and `## Next Steps` headings. The current slash command prompt encourages structured output but doesn't enforce specific headings. This task makes the heading format explicit.

- [ ] **Step 1: Update the summarization format instructions in `session-report.md`**

Find the section where summary format is described and update it to specify the exact heading structure:

```
Each summary MUST use these markdown headings:
## What Was Done
## Key Decisions  (include only if non-trivial decisions were made)
## Issues Encountered  (include only if there were problems; omit if none)
## Next Steps  (include only if there are clear follow-ups)
```

- [ ] **Step 2: Commit**

```bash
git add session-report.md
git commit -m "Enforce structured summary headings for dashboard redesign"
```

---

## Task 8: Update CLAUDE.md

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Update the HTML template documentation in CLAUDE.md**

Update the "Working with the HTML Template" section and line number references to reflect the new template structure. Update the "Reading the Script" section with new line ranges.

- [ ] **Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "Update CLAUDE.md for dashboard redesign"
```

---

## Implementation Notes

### What stays the same
- All CLI flags and pipeline subcommands (`--collect`, `--update-cache`, `--review-statuses`, `--update-statuses`)
- Cache format and invalidation logic
- Session scanning and transcript sampling
- Text output (`print_text_report`)
- The `md_to_html()` function (still used inside summary sections)

### What changes
- `generate_html()` — complete rewrite
- `main()` — add corrections reading before render

### What's new
- `aggregate_projects()` — project-level grouping and metadata
- `parse_summary_sections()` — structured summary parsing
- `format_duration()` — time duration formatting helper
- `read_corrections_from_previous_report()` — extract corrections from HTML
- `apply_corrections_to_cache()` — apply corrections to cache
- `SECTION_STYLES` — color mapping for summary sections
- `NEXT_STEPS_SECTIONS` — set of section labels rendered as "Next Steps" card

### Testing approach
This project has no test infrastructure. Verification is done by:
1. Running `python claude-session-report.py 3 no-ai --no-open` (quick, no AI dependency)
2. Opening the generated HTML in a browser for visual inspection
3. Running `python claude-session-report.py 3 text` to ensure text output is unaffected
4. Testing interactive features (sidebar collapse, project selection, status correction) in the browser
