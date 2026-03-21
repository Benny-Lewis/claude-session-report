# Session Report

Generate a summary report of all recent Claude Code sessions across all projects.

## Arguments

`$ARGUMENTS` - optional: number of days to look back (default: 3), and/or flags like `text`, `refresh`, `no-ai`

## Instructions

You are orchestrating a session report. Follow these steps exactly.

### Step 1: Collect sessions

Run the collection step to find sessions needing summarization:

```bash
python "INSTALL_DIR/claude-session-report.py" $ARGUMENTS --collect
```

This scans all Claude Code sessions and outputs a JSON file at `~/.claude/reports/_collect.json`.

### Step 2: Check if summarization is needed

Read the file at `~/.claude/reports/_collect.json`. Check the `needs_summary` field.

- If `needs_summary` is 0: tell the user all sessions are cached, then skip to Step 4.
- Otherwise: continue to Step 3.

### Step 3: Summarize sessions

For each session in the `sessions` array, read its `transcript` and `cwd` fields and generate a summary using EXACTLY this markdown format.

Each summary MUST use these markdown headings:

```
## What Was Done
- bullet points of concrete accomplishments

## Key Decisions
- bullet points of non-trivial decisions and their rationale

## Issues Encountered
- bullet points of problems, errors, or blockers hit during the session

## Next Steps
- where the work left off and what someone picking this up would likely do next
```

Rules for section inclusion:
- `## What Was Done` is REQUIRED in every summary.
- `## Key Decisions` — include ONLY if non-trivial decisions were made; omit entirely if none.
- `## Issues Encountered` — include ONLY if there were problems; omit entirely if none.
- `## Next Steps` — include ONLY if there are clear follow-ups; omit entirely if the work is fully complete.

Also determine the session's **status** and a short **title**:

**Status** — must be exactly one of these values:
- `complete` — All work finished, no meaningful pending items
- `in_progress` — Has unfinished work or clear actionable next steps
- `blocked` — Waiting on something external (hardware, deploy, another person, user action)
- `handed_off` — Session ended with an explicit continuation prompt for a new session

**Title** — a 5-10 word description of what the session was about (e.g., "Zeno panel interview prep - slide outline", "Weekly auto-approve audit review")

Rules:
- Work ONLY with the transcript provided. Never say you need more info.
- Be concise. Focus on what matters to a developer resuming this work.
- If the session is very short or unclear, summarize what you can see.
- Do NOT include a "Status" field in the markdown. Status is a separate JSON field.
- Very short sessions (< 6 messages) with no real work are usually `complete`.
- Sessions that end with "paste this to continue" or a handoff prompt are `handed_off`.

Build a JSON object where each key is the session's `session_id` and each value is an object with these EXACT fields:
- `"summary"`: the markdown summary you wrote
- `"status"`: one of `complete`, `in_progress`, `blocked`, `handed_off`
- `"title"`: the short title you wrote
- `"msg_count"`: copied VERBATIM from the collect data (do not modify)
- `"latest_ts"`: copied VERBATIM from the collect data (do not modify)
- `"folder"`: copied VERBATIM from the collect data (do not modify)

When there are more than 10 sessions to summarize, split them into batches of ~10 and dispatch ALL batch subagents in a single message so they run in parallel. Each subagent writes its batch to a separate file (e.g., `_summaries_batch1.json`, `_summaries_batch2.json`).

After all batches complete, merge them into a single `_summaries.json`. **CRITICAL (Windows encoding):** All `open()` calls in any merge script MUST use `encoding="utf-8"`. Example:

```python
import json, glob
merged = {}
for f in sorted(glob.glob(str(Path.home() / ".claude/reports/_summaries_batch*.json"))):
    with open(f, encoding="utf-8") as fh:
        merged.update(json.load(fh))
with open(str(Path.home() / ".claude/reports/_summaries.json"), "w", encoding="utf-8") as fh:
    json.dump(merged, fh, indent=2, ensure_ascii=False)
```

If there are 10 or fewer sessions, write directly to `~/.claude/reports/_summaries.json` (still using `encoding="utf-8"`).

Then update the cache:

```bash
python "INSTALL_DIR/claude-session-report.py" --update-cache ~/.claude/reports/_summaries.json
```

### Step 3b: Cross-session status review

After all summaries are cached, review statuses across related sessions in the same folder:

```bash
python "INSTALL_DIR/claude-session-report.py" $ARGUMENTS --review-statuses
```

Read `~/.claude/reports/_review.json`. Check `needs_review`.

- If `needs_review` is 0: skip to Step 4.
- Otherwise: For each folder in the `folders` array, read ALL sessions' summaries together (they are sorted chronologically). Determine if any statuses should be adjusted based on cross-session context:
  - If a session ends abruptly but the next session in that folder clearly continues the same work → earlier session should be `handed_off`
  - If work from an `in_progress` session is clearly completed in a later session → earlier session should be `complete`
  - If multiple sessions are part of the same ongoing work, only the most recent should be `in_progress` — earlier ones should be `complete` or `handed_off`
  - If a session was marked `in_progress` but a later session starts completely different work → the earlier one is likely `complete` (work was abandoned or finished off-screen)

Build a JSON object where each key is a session_id that needs a status **change**, and each value is an object with:
- `"status"`: the corrected status

Only include sessions whose status actually needs to change. If no corrections are needed, skip the update step.

Write to `~/.claude/reports/_status_corrections.json`.

Then update the cache:

```bash
python "INSTALL_DIR/claude-session-report.py" --update-statuses ~/.claude/reports/_status_corrections.json
```

### Step 4: Generate the report

```bash
python "INSTALL_DIR/claude-session-report.py" $ARGUMENTS
```

### Step 5: Clean up

```bash
rm -f ~/.claude/reports/_collect.json ~/.claude/reports/_summaries.json ~/.claude/reports/_summaries_batch*.json ~/.claude/reports/_review.json ~/.claude/reports/_status_corrections.json
```

Tell the user:
- How many sessions were found total
- How many had cached summaries
- How many were newly summarized in this run
- How many statuses were adjusted by cross-session review (if any)
- Whether the report opened in the browser or was printed to terminal

### Examples

- `/session-report` -- Last 3 days, HTML report opens in browser
- `/session-report 7` -- Last 7 days
- `/session-report 1 text` -- Last 24 hours, terminal output
- `/session-report 7 refresh` -- Last 7 days, re-summarize everything
- `/session-report 3 no-ai` -- Skip all summaries, show raw session data
- `/session-report 14 --limit 20` -- Last 14 days, cap at 20 sessions
