# Claude Session Report

A reporting tool for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) that scans all your sessions across every project on your machine, generates AI-powered summaries, and renders an HTML dashboard showing what was worked on, status, and next steps.

![Dashboard screenshot](screenshots/redesign-final-overview.png)

## Features

- Scans all Claude Code session transcripts across every project
- AI-powered session summaries with status tracking (complete, in progress, blocked, handed off)
- Self-contained HTML dashboard with sidebar navigation, dark mode, and inline status corrections
- Summary caching with smart invalidation — only re-summarizes when sessions change
- Works as a Claude Code slash command (`/session-report`) or standalone script

## Requirements

- Python 3.9+
- Claude Code installed (`~/.claude/` directory must exist)
- No external Python dependencies — standard library only

## Installation

Give `session-report-setup.md` to Claude Code and say "set this up". It will:

1. Copy the script to your preferred install location (default: `~/.claude/scripts/`)
2. Create the `/session-report` slash command
3. Run a quick test

Or install manually:

```bash
# Copy the script
mkdir -p ~/.claude/scripts
cp claude-session-report.py ~/.claude/scripts/claude-session-report.py

# Copy the slash command
cp session-report.md ~/.claude/commands/session-report.md
```

## Usage

### As a slash command (recommended)

In any Claude Code session:

```
/session-report                  # last 3 days, HTML dashboard
/session-report 7                # last 7 days
/session-report 1 text           # terminal output
/session-report 3 refresh        # re-summarize everything
/session-report 14 --limit 20   # last 14 days, max 20 sessions
```

### Standalone

```bash
python claude-session-report.py           # last 3 days, opens in browser
python claude-session-report.py 7         # last 7 days
python claude-session-report.py 3 no-ai   # skip AI summaries
python claude-session-report.py 1 text    # terminal output
```

## How It Works

The script reads JSONL transcript files from `~/.claude/projects/` and cross-references session metadata. For long sessions, it samples messages (first 6, middle 8, last 6) to stay within token limits.

When used as a slash command, Claude Code itself generates the summaries — the script collects session data and renders the HTML report. Summaries are cached at `~/.claude/session-report-cache.json` and only regenerated when a session's message count or timestamp changes.

## License

MIT
