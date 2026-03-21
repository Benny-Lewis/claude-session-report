# Session Report Dashboard Redesign — Design Spec

**Date:** 2026-03-20
**Status:** Approved
**Mockup:** `.superpowers/brainstorm/144780-1774063179/full-design-v3.html`

---

## Problem

The current dashboard is a flat session log viewer — it lists all sessions grouped by project folder, treating every session with roughly equal visual weight. The user's actual job when opening this report is **re-orientation**: "What was I working on? Where did I leave off? What's next?" The current design doesn't prioritize this.

## Design Vision

**Primary job:** Re-orient to where you left off, per project.
**Secondary job:** See the state of each project at a glance (portfolio view).
**Tone:** Warm/crafted with editorial refinement. Not flashy, not generic — feels like a thoughtfully made developer tool.

## Target Audience

A developer (the tool's creator) reviewing their Claude Code sessions across 3-10 active projects. Power user who values density and keyboard shortcuts but also appreciates good typography and visual design.

---

## Layout

### Master-Detail (Sidebar + Detail Panel)

The page is a side-by-side layout:

- **Sidebar (300px, left):** Project list with search, filters, and navigation. Collapsible to ~56px showing initials + status dots.
- **Detail panel (remaining width):** Full context for the selected project — last session summary, next steps, and recent session timeline.

The most recently active project is auto-selected on load so the user lands with context already visible.

### Responsive Behavior

- Below ~900px, the sidebar auto-collapses to the icon/initials view.
- The detail panel gets full width minus the collapsed sidebar.
- Light and dark themes both supported, persisted via localStorage. Dark is default.

---

## Sidebar

### Header
- **Title:** "Session Report"
- **Subtitle:** "Last N days · X projects"
- **Collapse toggle:** Double chevron `«` / `»` button. Collapses sidebar to 56px.

### Search
- Text input with placeholder "Search projects... (/)"
- `/` keyboard shortcut focuses the search
- Filters project list by name and session titles

### Status Filter Chips
- Horizontal row of filter chips: All, Active, Handed Off, Blocked
- Chips show counts
- Active chip is filled (inverted colors), others are outline
- Filters which projects appear in the list

### Project List (Expanded Sidebar)
Each project item shows:
- **Project name** (e.g., "session-report") — bold, primary text
- **Status badge** — colored pill with dropdown arrow `▾`, clickable to change status
- **Last session title** — single line, truncated with ellipsis
- **Recency + session count** — e.g., "2h ago · 6 sessions"

Selected project has a left border accent (status color) and slightly elevated background.

### Project List (Collapsed Sidebar)
Each project becomes a 36px square:
- **Two-letter initials** in neutral color (e.g., "SR", "CA")
- **Status dot** in bottom-right corner (9px circle, Slack-style online indicator)
- Selected project gets status-colored background
- Tooltip on hover shows project name

### Inactive Projects
- Collapsed section at bottom: "▶ 3 inactive projects"
- Click to expand/collapse
- Inactive items shown at 40% opacity
- "Inactive" = projects not touched within the report's time window

### Footer
- Theme toggle button (sun/moon icon)
- Label: "Dark theme" / "Light theme"

---

## Detail Panel

Content has a max-width of 780px for readability, left-aligned within the detail panel (not centered). Sections animate in with staggered fadeUp on project selection.

### Project Header
- **Project name** — Source Serif 4 (serif), large (~1.25-1.5rem), editorial feel
- **Project path** — monospace, muted (e.g., `~/dev/claude-session-report`)
- **Status badge** — same as sidebar but larger. Clickable with dropdown arrow to correct AI-assigned status.

### Summary Sections (Color-Coded Blocks)
The last session's AI summary is broken into structured sections, each with:
- A **thin left bar** (3px) in a muted accent color
- A **section label** in the same accent color
- **Content** — bullet lists or prose

Section accents are **tinted neutrals**, deliberately distinct from the status palette:
- **What Was Done** — warm muted tone (`oklch(0.55 0.03 55)`)
- **Key Decisions** — cool muted blue-gray (`oklch(0.50 0.025 200)`)
- **Issues Encountered** — soft warm tone (`oklch(0.52 0.035 25)`)

If color confusion with status badges becomes an issue, fall back toward editorial dividers (gradient `hr` between sections, no color coding on labels).

Not all sessions will have all three sections — render only what the AI summary provides.

### Session Metadata
Below the summary sections, a single line:
- Time, message count, duration separated by small dots
- e.g., "Today 2:38 PM · 44 messages · 1h 25m"

### Next Steps Card
- Elevated surface (`var(--surface)`) with border
- Uppercase label: "NEXT STEPS"
- Bullet list of next actions (read-only, not checkable)

### Recent Sessions Timeline
- **Header row:** "RECENT SESSIONS" label + filter chips (All, In Progress, Complete)
- **Timeline:** Vertical line on the left with dots at each session
  - First (most recent) session: glowing dot in status color with box-shadow halo
  - Other sessions: neutral dot
  - Timeline line uses gradient (strong at top, fading to transparent at bottom)
- **Each session item shows:**
  - Title (bold for most recent, regular for others)
  - Status badge with dropdown arrow `▾` (clickable to correct)
  - Metadata line: time · msg count · duration
- **Click to expand:** Shows inline expansion with:
  - Session summary text
  - (Future: "Changed:" line listing files — see backlog #32)
- **Scroll:** All sessions in the time window, no artificial cap. Natural scroll.
- **Session status filters** within the detail panel filter which sessions show in the timeline

---

## Status System

### Statuses
- **In Progress** — blue (`oklch hue 250`)
- **Blocked** — amber (`oklch hue 70`)
- **Handed Off** — purple (`oklch hue 300`)
- **Complete** — teal (`oklch hue 160`)
- **Abandoned** — (future, from backlog #31)

### Project-Level Status
AI-synthesized from the project's recent sessions. Not a mechanical rule (not just "most recent session's status" or "worst status wins") — the AI has context from summaries to make a smart judgment. For example: "blocked session from yesterday but new in-progress session today → project is in progress."

### Status Correction
All status badges (project-level and session-level) are clickable:
- Badge shows a subtle dropdown arrow `▾`
- On hover, badge subtly brightens
- Click opens a dropdown with all available statuses
- Current status has a checkmark
- `event.stopPropagation()` on badge clicks to prevent parent toggle

**Persistence mechanism — embedded corrections:**
1. When the user selects a new status, the correction is stored in `localStorage` (for immediate visual persistence across page reloads) AND written into a `<script type="application/json" id="status-corrections">` block in the report HTML's DOM.
2. The JS updates this block on every correction, maintaining a map of `{ sessionId: newStatus }`.
3. On page load, the JS reads both `localStorage` and the embedded block, merging them (localStorage wins on conflict since it's newer).
4. On the next `/session-report` run, the Python script reads the most recent existing report HTML from `~/.claude/reports/`, extracts the `#status-corrections` JSON block, and applies those corrections to the cache before generating the new report.
5. This requires no manual steps — the report itself is the persistence layer.

---

## Typography

- **UI chrome:** Outfit (geometric sans-serif) — weights 300-700
- **Project name (detail panel):** Source Serif 4 (serif) — editorial accent, weight 600
- **Code/paths:** SFMono-Regular / Cascadia Code / Consolas (monospace)
- **Type scale:** Fluid with `clamp()`:
  - `--step-0`: 0.8-0.875rem (captions, metadata)
  - `--step-1`: 0.875-1rem (body text, summaries)
  - `--step-2`: 1-1.15rem (sidebar title)
  - `--step-3`: 1.25-1.5rem (detail panel project name)

**Font loading:** Outfit and Source Serif 4 are loaded from Google Fonts. The report requires network access on first load; fonts are browser-cached after that. The fallback stacks (`-apple-system, BlinkMacSystemFont, 'Segoe UI', system-ui, sans-serif` and `Georgia, serif`) ensure the page is readable without network.

---

## Color System

OKLCH throughout. Warm neutral foundation (hue 55, amber-charcoal).

### Light Theme
Mirror the dark theme's structure with inverted lightness values. Maintain the same hues and relative chroma relationships. The warm neutral foundation should feel like warm paper/cream, not stark white.

### Dark Theme
Current mockup values — warm charcoal backgrounds, slightly warm text tones.

---

## Interactions

- **Sidebar collapse:** 0.3s ease with `cubic-bezier(0.16, 1, 0.3, 1)`. Content fades out, width animates.
- **Session expand:** `max-height` transition on the expansion container.
- **Detail panel entrance:** Staggered `fadeUp` animation (0.4s, 30-50ms between sections).
- **Hover states:** Subtle background change on project items, timeline items. Badge brightness increase on hover.
- **`prefers-reduced-motion`:** All animations (sidebar collapse, session expand, fadeUp entrance, hover transitions) should be disabled or reduced to near-instant when this media query matches.
- **Keyboard shortcuts:**
  - `/` or `Ctrl+K` — focus search
  - `Escape` — clear search, or clear filter, or blur
  - Arrow keys for project navigation (future consideration)

---

## Data Flow Changes

The existing Python script handles session scanning, caching, and HTML rendering. The redesign requires:

1. **Project-level aggregation:** Group sessions by project and compute project-level metadata (status, session count, last active time, most recent session title).
2. **AI project status:** During the `--review-statuses` pass (which already does cross-session review), add project-level status synthesis. Store as a new `project_status` field in the cache, keyed by project path. The slash command prompt should instruct Claude to assess overall project health from the session summaries.
3. **Structured summaries:** The AI summarization prompt should produce structured sections (What Was Done, Key Decisions, Issues Encountered) rather than free-form prose. The current prompt already encourages this — enforce it more explicitly with a format template.
4. **Status correction persistence:** The report HTML embeds corrections in a `<script type="application/json" id="status-corrections">` block. On the next run, the Python script reads the most recent report HTML from `~/.claude/reports/`, extracts corrections, and applies them to the cache before rendering.
5. **Active/inactive split:** A project is "inactive" if all its sessions in the report window are `complete` and its most recent session is older than 24 hours.

---

## What This Design Does NOT Include

- Task management (checkable next steps, assigning work)
- Cross-project search results view
- Print stylesheet
- Session diffing or code-level detail
- Real-time updates or polling

These could be future additions but are out of scope for this redesign.
