# Plan: Merge Three Review Documents into Single Backlog

## Context
Three separate review documents exist for the session report tool: a code review, a UX audit, and a design critique. The user wants these merged into a single prioritized backlog document with detailed entries, category tags, and source attribution. Overlapping findings across documents should be consolidated into single items.

## Output
A new file: `backlog.md` in the project root.

## Structure
- Single ranked list ordered by priority (P0 Critical → P3 Low)
- Each item includes: title, priority, category tag, source(s), location, issue description, user impact, fix recommendation
- Overlapping findings merged into single items citing all sources
- Reviewer gap findings (from my analysis) included as a separate section at the end

## Merged Findings Plan (27 unique items from ~32 raw findings)

### Merges identified:
1. **Footer file path exposure** — Code #11 + UX Sev 1 + Design minor → single item
2. **"Unknown" vs "Unsummarized" + unsummarized visual treatment** — UX #4 + Design Priority #5 → single item
3. **Type scale compression / small fonts** — UX Sev 2 + Design Priority #3 → single item
4. **Folder pills/metadata usefulness** — UX Sev 2 + Design Priority #4 → single item
5. **Session chevron size** — UX Sev 1 + Design minor → single item
6. **Detail-meta density + session ID fragments** — UX Sev 2 #9 + UX Sev 1 #12 → single item

### Priority mapping:
- **P0 (Critical):** Keyboard inaccessibility, non-atomic cache writes, stale setup doc
- **P1 (High):** aria-expanded, heading hierarchy, completed sessions dominate, controls/content visual seam, --max-collect silent cap, save_summary_cache return value, input validation
- **P2 (Medium):** Unknown/Unsummarized terminology + visual treatment, folder pills/metadata, type scale, search focus indicator, bulk buttons, detail-meta density, responsive viewport, no .gitignore
- **P3 (Low):** Footer path, status tooltips, keyboard shortcuts, search input width, chevron sizes, pluralization bug, decode_project_path edge case, mkdir repetition, timestamp function duplication, --limit performance, transcript sampling, print stylesheet

## Verification
- All findings from all three source docs are accounted for
- No duplicates in the final list
- Each item traceable to its source document(s)
