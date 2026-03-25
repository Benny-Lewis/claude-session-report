# Claude Session Report

A reporting tool for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) that scans all your sessions across every project, generates AI-powered summaries, and renders an HTML dashboard showing what was worked on, current status, and next steps.

## Features

- Scans all Claude Code session transcripts across every project
- AI-powered session summaries with status tracking (complete, in progress, blocked, handed off)
- Self-contained HTML dashboard with sidebar navigation, dark/light mode, and inline status corrections
- Summary caching with smart invalidation — only re-summarizes when sessions change
- Works as a Claude Code slash command (`/session-report`) or as a standalone script
- Cross-platform: tested on Windows and compatible with macOS/Linux

## Requirements

- Python 3.9+
- Claude Code CLI installed (`~/.claude/` directory must exist)
- No external Python dependencies — uses only the standard library

## Installation

### Automated (recommended)

Give [`session-report-setup.md`](session-report-setup.md) to Claude Code and it will handle the rest — copies the script, installs the slash command, and runs a test.

### Manual

```bash
# Install the script
mkdir -p ~/.claude/scripts
cp claude-session-report.py ~/.claude/scripts/claude-session-report.py

# Install the slash command
cp session-report.md ~/.claude/commands/session-report.md
```

## Usage

### Slash command

In any Claude Code session, type:

```
/session-report                  # last 3 days, HTML dashboard
/session-report 7                # last 7 days
/session-report 1 text           # terminal output instead of HTML
/session-report 3 refresh        # force re-summarize all sessions
/session-report 14 --limit 20   # last 14 days, cap at 20 sessions
```

### Standalone

```bash
python claude-session-report.py           # last 3 days, opens in browser
python claude-session-report.py 7         # last 7 days
python claude-session-report.py 3 no-ai   # skip AI summaries, show raw data
python claude-session-report.py 1 text    # terminal output
python claude-session-report.py 3 --no-open  # generate HTML without opening browser
```

## How It Works

The slash command orchestrates a multi-step pipeline:

1. **Collect** — the script scans JSONL transcripts from `~/.claude/projects/` and identifies sessions needing summaries
2. **Summarize** — Claude Code reads the transcripts and generates structured summaries (title, status, what was done, next steps)
3. **Cache** — summaries are stored at `~/.claude/session-report-cache.json`, keyed by session ID with message count + timestamp for invalidation
4. **Review** — a cross-session pass lets Claude correct statuses (e.g., marking an earlier session as `handed_off` when a later one continues the work)
5. **Render** — the script generates a self-contained HTML report and opens it in the browser

For long sessions (>20 messages), transcripts are sampled (first 6, middle 8, last 6 messages) to stay within token limits.

When run standalone without the slash command, the script skips AI summarization and shows raw session data.

## License

This project is licensed under the [Mozilla Public License 2.0](LICENSE).
