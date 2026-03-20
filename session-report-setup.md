# Claude Code Session Report — Setup

Give this file to Claude Code and say "set this up". It will install everything.

## What This Does

Scans all your Claude Code sessions across every project/folder on your machine, generates AI-powered summaries via the `/session-report` slash command, and opens an HTML dashboard in your browser showing what was worked on, what was accomplished, and what's likely next.

## Instructions for Claude

1. **Ask the user where to install the script.** Suggest `~/.claude/scripts/` as the default, but let them pick any path (e.g. `C:\scripts\`, `/opt/tools/`, their project folder, etc.). Use their answer as `INSTALL_DIR` for the remaining steps. If they just say "default" or "sure", use `~/.claude/scripts/`.

2. **Ask the user where the repo is cloned** (or where they downloaded `claude-session-report.py`). Use their answer as `REPO_DIR`.

3. Create `INSTALL_DIR` if it doesn't exist, then copy `REPO_DIR/claude-session-report.py` to `INSTALL_DIR/claude-session-report.py`.

4. Create the slash command file at `~/.claude/commands/session-report.md`. **IMPORTANT**: use the actual absolute path to where you installed the script in step 3 — do NOT hardcode `~/.claude/scripts/`, use whatever `INSTALL_DIR` the user chose:

~~~markdown
# Session Report

Generate a summary report of all recent Claude Code sessions across all projects.

## Arguments
`$ARGUMENTS` - optional: number of days to look back (default: 3), and/or flags

## Instructions

Run the session report script at `INSTALL_DIR/claude-session-report.py` (substitute the real path).

Pass `$ARGUMENTS` directly — the script accepts bare words without `--` prefixes.

### Run the command:
```bash
python INSTALL_DIR/claude-session-report.py $ARGUMENTS
```

### Examples:
- `/session-report` — Last 3 days, HTML report opens in browser
- `/session-report 7` — Last 7 days
- `/session-report 1 text` — Last 24 hours, terminal output
- `/session-report 7 refresh` — Last 7 days, force re-summarize everything
- `/session-report 3 no-ai` — Skip AI summaries
- `/session-report 14 --limit 20` — Last 14 days, cap at 20 sessions

After running, tell the user how many sessions were found and whether it opened in browser or terminal.
~~~

5. Run a quick test: `python INSTALL_DIR/claude-session-report.py 1 --no-open`

## The Script

The script is `claude-session-report.py` in this repo. Do NOT use an embedded copy — always reference the actual file from the repository to ensure you have the latest version.

## Requirements

- Python 3.9+
- Claude Code installed (`~/.claude/` directory must exist)
- No external Python dependencies — uses only the standard library

## Usage After Setup

In any Claude Code session:
```
/session-report                  # last 3 days, HTML dashboard
/session-report 7                # last 7 days
/session-report 1 text           # terminal output
/session-report 3 refresh        # re-summarize everything
/session-report 14 --limit 20   # last 14 days, cap at 20 sessions
```

Or directly:
```bash
python INSTALL_DIR/claude-session-report.py 3
python INSTALL_DIR/claude-session-report.py 7 text
```
