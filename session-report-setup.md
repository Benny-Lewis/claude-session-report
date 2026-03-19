# Claude Code Session Report — Setup

Give this file to Claude Code and say "set this up". It will install everything.

## What This Does

Scans all your Claude Code sessions across every project/folder on your machine, uses Haiku to summarize each one, and opens an HTML dashboard in your browser showing what was worked on, what was accomplished, and what's likely next.

## Instructions for Claude

1. **Ask the user where to install the script.** Suggest `~/.claude/scripts/` as the default, but let them pick any path (e.g. `C:\scripts\`, `/opt/tools/`, their project folder, etc.). Use their answer as `INSTALL_DIR` for the remaining steps. If they just say "default" or "sure", use `~/.claude/scripts/`.

2. Install Python dependencies:
```bash
pip install anthropic httpx
```

3. Create `INSTALL_DIR` if it doesn't exist, then write the full script below to `INSTALL_DIR/claude-session-report.py`.

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
- `/session-report 3 no-ai` — Skip Haiku summaries
- `/session-report 14 --limit 20` — Last 14 days, cap at 20 sessions

After running, tell the user how many sessions were found and whether it opened in browser or terminal.
~~~

5. Verify the API config exists in `~/.claude/settings.json`. AI summaries **require a LiteLLM proxy** — the script will NOT call the Anthropic API directly. The user's settings.json `env` block must have:
   - `ANTHROPIC_BASE_URL` — the LiteLLM proxy URL (REQUIRED for AI summaries)
   - `ANTHROPIC_AUTH_TOKEN` or `ANTHROPIC_API_KEY` — auth token for the proxy
   - If `ANTHROPIC_BASE_URL` is missing, warn the user that AI summaries are disabled. The report still works with `--no-ai` but won't have Haiku-generated summaries.

6. Run a quick test: `python ~/.claude/scripts/claude-session-report.py 1 --no-open`

## The Script

```python
#!/usr/bin/env python
"""
Claude Code Session Reporter
Scans all Claude Code sessions for recent activity and uses Haiku to produce
concise summaries of what was accomplished, where things left off, etc.

Works on Windows, macOS, and Linux — reads from ~/.claude/ which is the same
structure on all platforms. Requires `anthropic` and `httpx` packages for
AI summaries (falls back to raw output without them).

Usage:
    python claude-session-report.py [days]              # default: 3 days, HTML
    python claude-session-report.py 7                   # last 7 days
    python claude-session-report.py 1 --text            # terminal text output
    python claude-session-report.py 3 --no-ai           # skip Haiku summaries
    python claude-session-report.py 3 --refresh         # force re-summarize all
    python claude-session-report.py 3 --limit 20        # only process 20 most recent sessions
    python claude-session-report.py 7 text              # bare words work too (no -- needed)
"""

import json
import os
import sys
import re
import argparse
import threading
import webbrowser
import concurrent.futures
from datetime import datetime, timedelta, timezone
from pathlib import Path
from collections import defaultdict
from html import escape as html_escape

# Fix Windows console encoding
if sys.platform == "win32":
    try:
        sys.stdout.reconfigure(encoding="utf-8", errors="replace")
    except Exception:
        pass

CLAUDE_DIR = Path.home() / ".claude"
PROJECTS_DIR = CLAUDE_DIR / "projects"
SESSIONS_DIR = CLAUDE_DIR / "sessions"
HISTORY_FILE = CLAUDE_DIR / "history.jsonl"
SETTINGS_FILE = CLAUDE_DIR / "settings.json"
CACHE_FILE = CLAUDE_DIR / "session-report-cache.json"
REPORT_DIR = CLAUDE_DIR / "reports"
HOME_STR = str(Path.home())

# Minimum messages to bother calling Haiku — shorter sessions get raw output
MIN_MESSAGES_FOR_AI = 4

# LiteLLM / Anthropic config
LITELLM_BASE_URL = None
LITELLM_API_KEY = None
HAIKU_MODEL = None

_thread_local = threading.local()

# ─── Cache ────────────────────────────────────────────────────────────────────


def load_summary_cache() -> dict:
    if not CACHE_FILE.exists():
        return {}
    try:
        with open(CACHE_FILE, "r", encoding="utf-8") as f:
            return json.load(f)
    except (json.JSONDecodeError, OSError):
        return {}


def save_summary_cache(cache: dict):
    try:
        with open(CACHE_FILE, "w", encoding="utf-8") as f:
            json.dump(cache, f, indent=2, ensure_ascii=False)
    except OSError:
        pass


def get_cached_summary(cache, session_id, msg_count, latest_ts):
    entry = cache.get(session_id)
    if not entry:
        return None
    summary = entry.get("summary", "")
    if not summary or summary.startswith("(Haiku summary failed") or summary.startswith("(error"):
        return None
    if entry.get("msg_count") == msg_count and entry.get("latest_ts") == latest_ts:
        return summary
    return None


# ─── Settings & API ──────────────────────────────────────────────────────────


def load_claude_settings():
    global LITELLM_BASE_URL, LITELLM_API_KEY, HAIKU_MODEL
    if not SETTINGS_FILE.exists():
        return
    try:
        with open(SETTINGS_FILE, "r", encoding="utf-8") as f:
            settings = json.load(f)
        env = settings.get("env", {})
        LITELLM_BASE_URL = env.get("ANTHROPIC_BASE_URL")
        LITELLM_API_KEY = env.get("ANTHROPIC_AUTH_TOKEN") or env.get("ANTHROPIC_API_KEY")
        HAIKU_MODEL = env.get(
            "ANTHROPIC_DEFAULT_HAIKU_MODEL", "claude-haiku-4-5-20251001"
        )
    except (json.JSONDecodeError, OSError):
        pass


def get_anthropic_client():
    client = getattr(_thread_local, "anthropic_client", None)
    if client is not None:
        return client
    import anthropic
    import httpx

    cert_file = os.environ.get("SSL_CERT_FILE") or os.environ.get("REQUESTS_CA_BUNDLE")
    http_client = httpx.Client(verify=cert_file if cert_file else True)
    client = anthropic.Anthropic(
        base_url=LITELLM_BASE_URL, api_key=LITELLM_API_KEY, http_client=http_client
    )
    _thread_local.anthropic_client = client
    return client


def clean_transcript_text(text):
    """Strip XML command tags and other noise from message text for Haiku."""
    if not text:
        return ""
    text = re.sub(r'<command-name>.*?</command-name>', '', text, flags=re.DOTALL)
    text = re.sub(r'<command-message>.*?</command-message>', '', text, flags=re.DOTALL)
    text = re.sub(r'<command-args>.*?</command-args>', '', text, flags=re.DOTALL)
    text = re.sub(r'<local-command-caveat>.*?</local-command-caveat>', '', text, flags=re.DOTALL)
    text = re.sub(r'<system-reminder>.*?</system-reminder>', '', text, flags=re.DOTALL)
    text = re.sub(r'<task-notification>.*?</task-notification>', '', text, flags=re.DOTALL)
    text = re.sub(r'\s+', ' ', text).strip()
    return text


def summarize_with_haiku(session_data: dict) -> str:
    if not LITELLM_BASE_URL or not LITELLM_API_KEY:
        return None
    try:
        import anthropic
    except ImportError:
        return None

    messages = session_data.get("raw_messages", [])
    if len(messages) <= 20:
        sampled = messages
    else:
        first = messages[:6]
        last = messages[-6:]
        mid_start = len(messages) // 3
        mid_end = 2 * len(messages) // 3
        middle = messages[mid_start : mid_start + 4] + messages[mid_end : mid_end + 4]
        sampled = (
            first
            + [{"role": "system", "text": f"... ({len(messages) - 20} messages omitted) ..."}]
            + middle
            + last
        )

    transcript_parts = []
    for m in sampled:
        text = clean_transcript_text(m["text"][:500])
        if not text or len(text) < 5:
            continue
        if text.startswith("[tool result") or text.startswith("[tool:"):
            continue
        transcript_parts.append(f"{m['role'].upper()}: {text}")

    transcript = "\n".join(transcript_parts)[:12000]

    if len(transcript.strip()) < 50:
        return None

    folder = session_data.get("cwd", "unknown")

    prompt = f"""Summarize this Claude Code session. The session was in folder: {folder}

Use EXACTLY this markdown format:

## Summary
2-3 sentences: What was the main goal or task?

## What Was Done
- bullet points of concrete accomplishments

## What's Next
Describe where the work left off and what the natural next steps would be.
Be specific and open-ended — don't classify into categories, just describe
what someone picking this up would likely want to do next.

IMPORTANT RULES:
- Work ONLY with the transcript provided. Never say you need more info.
- If the session is very short or unclear, summarize what you can see.
- Be concise. Focus on what matters to a developer resuming this work.
- Do NOT include a "Status" field. The "What's Next" section covers that.

SESSION TRANSCRIPT:
{transcript}"""

    try:
        client = get_anthropic_client()
        response = client.messages.create(
            model=HAIKU_MODEL, max_tokens=600, messages=[{"role": "user", "content": prompt}]
        )
        return response.content[0].text
    except Exception as e:
        return f"(Haiku summary failed: {e})"


# ─── Session parsing ─────────────────────────────────────────────────────────


def decode_project_path(encoded: str) -> str:
    """Convert Claude's encoded project dir name back to a readable filesystem path."""
    parts = encoded.split("--")
    if sys.platform == "win32":
        if parts and len(parts[0]) == 1 and parts[0].isalpha():
            drive = parts[0] + ":"
            return drive + "\\" + "\\".join(parts[1:])
        return "\\".join(parts)
    else:
        if encoded.startswith("--"):
            return "/" + "/".join(parts[1:])
        if parts and len(parts[0]) == 1 and parts[0].isalpha():
            return parts[0] + ":/" + "/".join(parts[1:])
        return "/".join(parts)


def extract_text_from_content(content):
    if isinstance(content, str):
        return content
    if isinstance(content, list):
        texts = []
        for block in content:
            if isinstance(block, dict):
                if block.get("type") == "text":
                    texts.append(block.get("text", ""))
                elif block.get("type") == "tool_use":
                    texts.append(f"[tool: {block.get('name', '')}]")
            elif isinstance(block, str):
                texts.append(block)
        return " ".join(texts)
    return str(content)


def parse_session_file(filepath):
    messages = []
    session_id = None
    cwd = None
    try:
        with open(filepath, "r", encoding="utf-8", errors="replace") as f:
            for line in f:
                line = line.strip()
                if not line:
                    continue
                try:
                    entry = json.loads(line)
                except json.JSONDecodeError:
                    continue
                entry_type = entry.get("type")
                if entry_type not in ("user", "assistant"):
                    continue
                if not session_id:
                    session_id = entry.get("sessionId")
                if not cwd:
                    cwd = entry.get("cwd")
                timestamp_str = entry.get("timestamp")
                timestamp = None
                if timestamp_str:
                    try:
                        if isinstance(timestamp_str, str):
                            timestamp = datetime.fromisoformat(timestamp_str.replace("Z", "+00:00"))
                        elif isinstance(timestamp_str, (int, float)):
                            timestamp = datetime.fromtimestamp(timestamp_str / 1000, tz=timezone.utc)
                    except (ValueError, OSError):
                        pass
                msg = entry.get("message", {})
                role = msg.get("role", entry_type)
                content = msg.get("content", "")
                text = extract_text_from_content(content)
                if text and text.strip():
                    messages.append({"role": role, "text": text.strip(), "timestamp": timestamp})
    except (OSError, PermissionError):
        return None
    if not messages:
        return None
    return {"session_id": session_id or filepath.stem, "cwd": cwd, "messages": messages}


def get_session_cwd_from_sessions_dir():
    mapping = {}
    if not SESSIONS_DIR.exists():
        return mapping
    for f in SESSIONS_DIR.glob("*.json"):
        try:
            with open(f, "r", encoding="utf-8", errors="replace") as fh:
                data = json.loads(fh.read())
                sid = data.get("sessionId")
                cwd = data.get("cwd")
                if sid and cwd:
                    mapping[sid] = cwd
        except (json.JSONDecodeError, OSError):
            pass
    return mapping


def get_history_sessions():
    sessions = defaultdict(lambda: {"project": None})
    if not HISTORY_FILE.exists():
        return sessions
    try:
        with open(HISTORY_FILE, "r", encoding="utf-8", errors="replace") as f:
            for line in f:
                try:
                    entry = json.loads(line.strip())
                    sid = entry.get("sessionId")
                    if sid and entry.get("project"):
                        sessions[sid]["project"] = entry["project"]
                except json.JSONDecodeError:
                    continue
    except OSError:
        pass
    return sessions


# ─── Helpers ──────────────────────────────────────────────────────────────────


def truncate(text, max_len=120):
    text = text.replace("\n", " ").strip()
    return text if len(text) <= max_len else text[: max_len - 3] + "..."


def format_timestamp(ts):
    if not ts:
        return "unknown"
    local = ts.astimezone()
    now = datetime.now(timezone.utc).astimezone()
    if local.date() == now.date():
        return f"Today {local.strftime('%I:%M %p')}"
    elif local.date() == (now - timedelta(days=1)).date():
        return f"Yesterday {local.strftime('%I:%M %p')}"
    else:
        return local.strftime("%b %d %I:%M %p")


def short_path(folder):
    """Replace home directory prefix with ~ for display."""
    if folder.startswith(HOME_STR):
        return "~" + folder[len(HOME_STR):]
    return folder


def clean_user_text(text):
    text = text.strip()
    if not text or len(text) < 5:
        return None
    for p in ("[tool result", "[Request interrupted", "<task-notification",
              "<command-", "<local-command", "# Screenshot"):
        if text.startswith(p):
            return None
    if text.startswith("This session is being continued"):
        return "(continued from previous session)"
    return text


# ─── Collect sessions ────────────────────────────────────────────────────────


def collect_sessions(days):
    cutoff = datetime.now(timezone.utc) - timedelta(days=days)
    history_sessions = get_history_sessions()
    session_cwd_map = get_session_cwd_from_sessions_dir()
    all_sessions = []

    if not PROJECTS_DIR.exists():
        return all_sessions

    print("Scanning sessions...", end="", flush=True)

    for project_dir in sorted(PROJECTS_DIR.iterdir()):
        if not project_dir.is_dir():
            continue
        project_path = decode_project_path(project_dir.name)

        for session_file in project_dir.glob("*.jsonl"):
            try:
                mtime = datetime.fromtimestamp(session_file.stat().st_mtime, tz=timezone.utc)
                if mtime < cutoff:
                    continue
            except OSError:
                continue

            data = parse_session_file(session_file)
            if not data or not data["messages"]:
                continue

            timestamps = [m["timestamp"] for m in data["messages"] if m["timestamp"]]
            if not timestamps:
                continue
            latest = max(timestamps)
            earliest = min(timestamps)
            if latest < cutoff:
                continue

            cwd = data["cwd"]
            sid = data["session_id"]
            if not cwd and sid in session_cwd_map:
                cwd = session_cwd_map[sid]
            if not cwd and sid in history_sessions:
                cwd = history_sessions[sid]["project"]
            if not cwd:
                cwd = project_path

            user_msgs = [m for m in data["messages"] if m["role"] == "user"]
            assistant_msgs = [m for m in data["messages"] if m["role"] == "assistant"]

            topics = []
            for m in user_msgs:
                clean = clean_user_text(m["text"])
                if clean:
                    topics.append(truncate(clean, 100))

            first_ask = topics[0] if topics else (
                truncate(clean_transcript_text(user_msgs[0]["text"]), 200) if user_msgs else "N/A"
            )

            last_user = last_assistant = None
            for m in reversed(data["messages"]):
                if m["role"] == "user" and not last_user:
                    last_user = m["text"]
                if m["role"] == "assistant" and not last_assistant:
                    last_assistant = m["text"]
                if last_user and last_assistant:
                    break

            all_sessions.append({
                "session_id": sid,
                "cwd": cwd,
                "project_path": project_path,
                "earliest": earliest,
                "latest": latest,
                "user_msg_count": len(user_msgs),
                "assistant_msg_count": len(assistant_msgs),
                "total_messages": len(data["messages"]),
                "first_ask": first_ask,
                "topics": topics,
                "last_user": last_user,
                "last_assistant": last_assistant,
                "raw_messages": data["messages"],
            })
            print(".", end="", flush=True)

    print(f" done! ({len(all_sessions)} sessions found)\n")
    all_sessions.sort(key=lambda s: s["latest"], reverse=True)
    return all_sessions


# ─── Summarize ────────────────────────────────────────────────────────────────


def generate_summaries(all_sessions, use_ai, refresh):
    summaries = {}
    cache = {} if refresh else (load_summary_cache() if use_ai else {})

    if not use_ai:
        return summaries

    need_summary = []
    skipped = 0
    for s in all_sessions:
        sid = s["session_id"]
        if s["total_messages"] < MIN_MESSAGES_FOR_AI:
            skipped += 1
            continue
        latest_ts = s["latest"].isoformat() if s["latest"] else ""
        cached = get_cached_summary(cache, sid, s["total_messages"], latest_ts)
        if cached:
            summaries[sid] = cached
        else:
            need_summary.append(s)

    cached_count = len(all_sessions) - len(need_summary) - skipped
    if cached_count:
        print(f"  {cached_count} session(s) loaded from cache")
    if skipped:
        print(f"  {skipped} session(s) too short for AI summary")

    if need_summary:
        print(f"Generating Haiku summaries for {len(need_summary)} session(s)", end="", flush=True)
        with concurrent.futures.ThreadPoolExecutor(max_workers=5) as executor:
            future_map = {executor.submit(summarize_with_haiku, s): s for s in need_summary}
            for future in concurrent.futures.as_completed(future_map):
                s = future_map[future]
                sid = s["session_id"]
                try:
                    result = future.result()
                    if result:
                        summaries[sid] = result
                        latest_ts = s["latest"].isoformat() if s["latest"] else ""
                        cache[sid] = {
                            "summary": result,
                            "msg_count": s["total_messages"],
                            "latest_ts": latest_ts,
                            "folder": s.get("cwd", ""),
                        }
                except Exception as e:
                    summaries[sid] = f"(error: {e})"
                print(".", end="", flush=True)
        print(" done!")
        save_summary_cache(cache)
    elif not skipped or cached_count:
        print("  All summaries loaded from cache!")
    print()
    return summaries


# ─── Markdown conversion ─────────────────────────────────────────────────────


def md_to_html(text):
    lines = text.split("\n")
    html_lines = []
    in_list = False

    for line in lines:
        stripped = line.strip()
        if in_list and not stripped.startswith("- ") and not stripped.startswith("* "):
            html_lines.append("</ul>")
            in_list = False
        if stripped.startswith("## "):
            html_lines.append(f"<h4>{html_escape(stripped[3:])}</h4>")
        elif stripped.startswith("# "):
            html_lines.append(f"<h3>{html_escape(stripped[2:])}</h3>")
        elif stripped.startswith("- ") or stripped.startswith("* "):
            if not in_list:
                html_lines.append("<ul>")
                in_list = True
            content = stripped[2:]
            content = re.sub(r'\*\*(.+?)\*\*', r'<strong>\1</strong>', content)
            content = re.sub(r'`(.+?)`', r'<code>\1</code>', content)
            html_lines.append(f"<li>{content}</li>")
        elif stripped == "":
            continue
        else:
            content = html_escape(stripped)
            content = re.sub(r'\*\*(.+?)\*\*', r'<strong>\1</strong>', content)
            content = re.sub(r'`(.+?)`', r'<code>\1</code>', content)
            html_lines.append(f"<p>{content}</p>")

    if in_list:
        html_lines.append("</ul>")
    return "\n".join(html_lines)


# ─── HTML output ──────────────────────────────────────────────────────────────


def generate_html(all_sessions, summaries, days):
    by_project = defaultdict(list)
    for s in all_sessions:
        by_project[s["cwd"] or s["project_path"]].append(s)

    total_msgs = sum(s["total_messages"] for s in all_sessions)
    total_user = sum(s["user_msg_count"] for s in all_sessions)
    now_str = datetime.now().strftime("%Y-%m-%d %I:%M %p")

    folder_cards = []
    sorted_folders = sorted(
        by_project.items(), key=lambda x: max(s["latest"] for s in x[1]), reverse=True
    )
    for folder_idx, (folder, sessions) in enumerate(sorted_folders):
        sf = short_path(folder)
        latest_ts = format_timestamp(max(s["latest"] for s in sessions))
        total_folder_msgs = sum(s["total_messages"] for s in sessions)

        session_cards = []
        for s in sessions:
            sid = s["session_id"]
            summary = summaries.get(sid, "")

            if summary and not summary.startswith("("):
                body = md_to_html(summary)
            else:
                parts = [f"<h4>Opening Ask</h4><p>{html_escape(truncate(s['first_ask'], 300))}</p>"]
                if s["last_assistant"]:
                    clean = clean_transcript_text(s["last_assistant"])
                    if clean and len(clean) > 10:
                        parts.append(f"<h4>Last Response</h4><p>{html_escape(truncate(clean, 300))}</p>")
                body = "\n".join(parts)

            session_cards.append(f"""
            <div class="session-card">
                <div class="session-header">
                    <span class="session-id">{sid[:8]}...</span>
                    <span class="session-msgs">{s['total_messages']} msgs</span>
                </div>
                <div class="session-meta">
                    <span>{format_timestamp(s['earliest'])} &rarr; {format_timestamp(s['latest'])}</span>
                    <span>{s['user_msg_count']} user / {s['assistant_msg_count']} assistant</span>
                </div>
                <div class="session-body">{body}</div>
            </div>
            """)

        collapsed_cls = "" if folder_idx == 0 else " collapsed"
        folder_cards.append(f"""
        <div class="folder-section{collapsed_cls}">
            <div class="folder-header" onclick="this.parentElement.classList.toggle('collapsed')">
                <span class="folder-arrow">&#9660;</span>
                <span class="folder-name">{html_escape(sf)}</span>
                <div class="folder-badges">
                    <span class="folder-badge">{len(sessions)} session(s)</span>
                    <span class="folder-badge">{total_folder_msgs} msgs</span>
                    <span class="folder-badge">{latest_ts}</span>
                </div>
            </div>
            <div class="folder-body">
                {''.join(session_cards)}
            </div>
        </div>
        """)

    html = f"""<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<title>Claude Code Session Report - Last {days} day(s)</title>
<style>
    :root {{
        --bg: #0d1117; --surface: #161b22; --surface2: #21262d;
        --border: #30363d; --text: #e6edf3; --text2: #8b949e;
        --accent: #58a6ff; --font: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, monospace;
    }}
    * {{ box-sizing: border-box; margin: 0; padding: 0; }}
    body {{ background: var(--bg); color: var(--text); font-family: var(--font); padding: 24px; max-width: 1100px; margin: 0 auto; }}
    .header {{ text-align: center; margin-bottom: 32px; }}
    .header h1 {{ font-size: 24px; margin-bottom: 8px; }}
    .header .subtitle {{ color: var(--text2); font-size: 14px; }}
    .stats-bar {{
        display: flex; gap: 12px; justify-content: center;
        flex-wrap: wrap; margin: 20px 0 32px;
    }}
    .stat-box {{
        background: var(--surface); border: 1px solid var(--border);
        border-radius: 8px; padding: 12px 20px; text-align: center; min-width: 120px;
    }}
    .stat-box .num {{ font-size: 28px; font-weight: bold; color: var(--accent); }}
    .stat-box .label {{ font-size: 12px; color: var(--text2); margin-top: 4px; }}
    .controls {{ text-align: center; margin-bottom: 20px; }}
    .controls button {{
        background: var(--surface2); border: 1px solid var(--border); color: var(--text2);
        padding: 6px 16px; border-radius: 6px; cursor: pointer; font-family: var(--font);
        font-size: 13px; margin: 0 4px;
    }}
    .controls button:hover {{ background: var(--border); color: var(--text); }}
    .search-box {{
        text-align: center; margin-bottom: 20px;
    }}
    .search-box input {{
        background: var(--surface); border: 1px solid var(--border); color: var(--text);
        padding: 8px 16px; border-radius: 8px; font-family: var(--font); font-size: 14px;
        width: 100%; max-width: 400px; outline: none;
    }}
    .search-box input:focus {{ border-color: var(--accent); }}
    .search-box .search-hint {{ color: var(--text2); font-size: 12px; margin-top: 4px; }}
    .folder-section.search-hidden {{ display: none; }}
    .session-card.search-hidden {{ display: none; }}
    .folder-section {{
        background: var(--surface); border: 1px solid var(--border);
        border-radius: 10px; margin-bottom: 12px; overflow: hidden;
    }}
    .folder-header {{
        padding: 14px 20px; cursor: pointer; display: flex; align-items: center;
        gap: 10px; user-select: none; font-weight: 600; font-size: 15px;
    }}
    .folder-header:hover {{ background: var(--surface2); }}
    .folder-arrow {{ transition: transform 0.2s; font-size: 10px; color: var(--text2); }}
    .folder-section.collapsed .folder-arrow {{ transform: rotate(-90deg); }}
    .folder-section.collapsed .folder-body {{ display: none; }}
    .folder-name {{ flex: 1; }}
    .folder-badges {{ display: flex; gap: 8px; }}
    .folder-badge {{
        background: var(--surface2); padding: 2px 10px;
        border-radius: 10px; font-size: 12px; color: var(--text2); font-weight: 400;
    }}
    .folder-body {{ padding: 12px; border-top: 1px solid var(--border); }}
    .session-card {{
        background: var(--surface2); border: 1px solid var(--border);
        border-radius: 8px; padding: 16px; margin-bottom: 12px;
    }}
    .session-card:last-child {{ margin-bottom: 0; }}
    .session-header {{
        display: flex; justify-content: space-between; align-items: center; margin-bottom: 8px;
    }}
    .session-id {{ font-family: monospace; color: var(--text2); font-size: 13px; }}
    .session-msgs {{
        background: var(--bg); padding: 2px 8px; border-radius: 8px;
        font-size: 11px; color: var(--text2);
    }}
    .session-meta {{
        display: flex; gap: 20px; font-size: 12px; color: var(--text2);
        margin-bottom: 12px; flex-wrap: wrap;
    }}
    .session-body {{ font-size: 14px; line-height: 1.6; }}
    .session-body h3 {{ font-size: 15px; margin: 12px 0 6px; color: var(--accent); }}
    .session-body h4 {{ font-size: 14px; margin: 10px 0 4px; color: var(--accent); }}
    .session-body ul {{ margin: 4px 0 8px 20px; }}
    .session-body li {{ margin-bottom: 4px; }}
    .session-body p {{ margin: 4px 0; }}
    .session-body code {{
        background: var(--bg); padding: 1px 5px; border-radius: 3px;
        font-size: 13px; color: #f0883e;
    }}
    .session-body strong {{ color: #fff; }}
    .footer {{ text-align: center; color: var(--text2); font-size: 12px; margin-top: 32px; padding-top: 16px; border-top: 1px solid var(--border); }}
</style>
</head>
<body>
    <div class="header">
        <h1>Claude Code Session Report</h1>
        <div class="subtitle">Last {days} day(s) &middot; Generated {now_str}</div>
    </div>

    <div class="stats-bar">
        <div class="stat-box"><div class="num">{len(all_sessions)}</div><div class="label">Sessions</div></div>
        <div class="stat-box"><div class="num">{len(by_project)}</div><div class="label">Folders</div></div>
        <div class="stat-box"><div class="num">{total_msgs:,}</div><div class="label">Messages</div></div>
        <div class="stat-box"><div class="num">{total_user:,}</div><div class="label">User Prompts</div></div>
    </div>

    <div class="controls">
        <button onclick="document.querySelectorAll('.folder-section').forEach(e=>e.classList.remove('collapsed'))">Expand All</button>
        <button onclick="document.querySelectorAll('.folder-section').forEach(e=>e.classList.add('collapsed'))">Collapse All</button>
    </div>

    <div class="search-box">
        <input type="text" id="searchInput" placeholder="Filter sessions..." oninput="filterSessions(this.value)">
        <div class="search-hint">Search by folder name, session content, or keywords</div>
    </div>

    {''.join(folder_cards)}

    <div class="footer">
        Summaries by Claude Haiku &middot; Cache: {str(CACHE_FILE)}
    </div>

<script>
function filterSessions(query) {{
    const q = query.toLowerCase().trim();
    document.querySelectorAll('.folder-section').forEach(folder => {{
        if (!q) {{
            folder.classList.remove('search-hidden');
            folder.querySelectorAll('.session-card').forEach(c => c.classList.remove('search-hidden'));
            return;
        }}
        const folderName = folder.querySelector('.folder-name').textContent.toLowerCase();
        let folderHasMatch = folderName.includes(q);
        folder.querySelectorAll('.session-card').forEach(card => {{
            const cardText = card.textContent.toLowerCase();
            if (cardText.includes(q)) {{
                card.classList.remove('search-hidden');
                folderHasMatch = true;
            }} else {{
                card.classList.add('search-hidden');
            }}
        }});
        if (folderHasMatch) {{
            folder.classList.remove('search-hidden');
            folder.classList.remove('collapsed');
        }} else {{
            folder.classList.add('search-hidden');
        }}
    }});
}}
</script>
</body>
</html>"""
    return html


# ─── Text output (terminal) ──────────────────────────────────────────────────


def print_text_report(all_sessions, summaries, days):
    by_project = defaultdict(list)
    for s in all_sessions:
        by_project[s["cwd"] or s["project_path"]].append(s)

    print(f"Found {len(all_sessions)} active session(s) across {len(by_project)} folder(s)\n")

    for folder, sessions in sorted(
        by_project.items(), key=lambda x: max(s["latest"] for s in x[1]), reverse=True
    ):
        sf = short_path(folder)
        print(f"{'─'*80}")
        print(f"  FOLDER: {sf}")
        print(f"  Sessions: {len(sessions)}")
        print(f"{'─'*80}")

        for i, s in enumerate(sessions, 1):
            print(f"\n  Session {i}/{len(sessions)}  [{s['session_id'][:8]}...]")
            print(f"  Started:  {format_timestamp(s['earliest'])}")
            print(f"  Latest:   {format_timestamp(s['latest'])}")
            print(f"  Messages: {s['total_messages']} total ({s['user_msg_count']} user, {s['assistant_msg_count']} assistant)")

            sid = s["session_id"]
            if sid in summaries and summaries[sid] and not summaries[sid].startswith("("):
                print(f"\n{summaries[sid]}")
            else:
                print(f"\n  OPENING ASK:")
                print(f"    {truncate(s['first_ask'], 200)}")
                if len(s["topics"]) > 1:
                    print(f"\n  TOPICS ({len(s['topics'])} user prompts):")
                    show = s["topics"]
                    if len(show) > 8:
                        for t in show[:5]:
                            print(f"    - {t}")
                        print(f"    ... ({len(show) - 7} more) ...")
                        for t in show[-2:]:
                            print(f"    - {t}")
                    else:
                        for t in show:
                            print(f"    - {t}")
                print(f"\n  WHAT'S NEXT:")
                if s["last_user"]:
                    clean = clean_user_text(s["last_user"])
                    if clean:
                        print(f"    Last ask: {truncate(clean, 200)}")
                if s["last_assistant"]:
                    text = clean_transcript_text(s["last_assistant"])
                    if text and len(text) > 10:
                        print(f"    Last response: {truncate(text, 200)}")
            print()

    total_msgs = sum(s["total_messages"] for s in all_sessions)
    total_user = sum(s["user_msg_count"] for s in all_sessions)
    print(f"\n{'='*80}")
    print(f"  SUMMARY")
    print(f"{'='*80}")
    print(f"  Total sessions:  {len(all_sessions)}")
    print(f"  Total messages:  {total_msgs} ({total_user} user prompts)")
    print(f"  Active folders:  {len(by_project)}")
    print(f"\n  Most active folders:")
    folder_activity = sorted(by_project.items(), key=lambda x: sum(s["total_messages"] for s in x[1]), reverse=True)
    for folder, sessions in folder_activity[:10]:
        msg_count = sum(s["total_messages"] for s in sessions)
        sf = short_path(folder)
        print(f"    {sf:<50} {len(sessions):>2} session(s), {msg_count:>5} msgs")
    print()


# ─── Main ─────────────────────────────────────────────────────────────────────


def cleanup_old_reports(keep=10):
    """Delete old HTML reports, keeping the most recent `keep` files."""
    if not REPORT_DIR.exists():
        return
    reports = sorted(REPORT_DIR.glob("session-report-*.html"), key=lambda f: f.stat().st_mtime, reverse=True)
    for old in reports[keep:]:
        try:
            old.unlink()
        except OSError:
            pass
    removed = len(reports) - min(len(reports), keep)
    if removed > 0:
        print(f"  Cleaned up {removed} old report(s)")


def normalize_args(argv):
    """Accept bare words (text, refresh, no-ai) as aliases for --flags.

    This lets the slash command pass $ARGUMENTS straight through without
    Claude having to parse and reconstruct flags every time.
    """
    normalized = []
    bare_words = {"text": "--text", "refresh": "--refresh", "no-ai": "--no-ai", "no-open": "--no-open"}
    for arg in argv:
        normalized.append(bare_words.get(arg.lower(), arg))
    return normalized


def main():
    parser = argparse.ArgumentParser(description="Claude Code Session Reporter")
    parser.add_argument("days", nargs="?", type=int, default=3, help="Days to look back (default: 3)")
    parser.add_argument("--no-ai", action="store_true", help="Skip Haiku summarization")
    parser.add_argument("--refresh", action="store_true", help="Force re-summarize all sessions")
    parser.add_argument("--text", action="store_true", help="Terminal text output instead of HTML")
    parser.add_argument("--no-open", action="store_true", help="Generate HTML but don't open browser")
    parser.add_argument("--limit", type=int, default=0, help="Max sessions to process (0 = unlimited)")
    args = parser.parse_args(normalize_args(sys.argv[1:]))

    days = args.days
    use_ai = not args.no_ai
    load_claude_settings()

    if use_ai and not LITELLM_BASE_URL:
        print("WARNING: ANTHROPIC_BASE_URL not set in settings.json.")
        print("AI summaries require a LiteLLM proxy — direct Anthropic API is not allowed.")
        print("Falling back to raw output. Use --no-ai to suppress this warning.\n")
        use_ai = False
    elif use_ai and not LITELLM_API_KEY:
        print("WARNING: No API key found (ANTHROPIC_AUTH_TOKEN or ANTHROPIC_API_KEY).")
        print("Falling back to raw output.\n")
        use_ai = False

    print(f"\n{'='*80}")
    print(f"  CLAUDE CODE SESSION REPORT - Last {days} day(s)")
    print(f"  Generated: {datetime.now().strftime('%Y-%m-%d %I:%M %p')}")
    if use_ai:
        print(f"  Summaries powered by: {HAIKU_MODEL}")
    print(f"{'='*80}\n")

    all_sessions = collect_sessions(days)
    if not all_sessions:
        print(f"No sessions with activity in the last {days} day(s).\n")
        return

    if args.limit > 0 and len(all_sessions) > args.limit:
        print(f"  Limiting to {args.limit} most recent sessions (of {len(all_sessions)} found)")
        all_sessions = all_sessions[: args.limit]

    summaries = generate_summaries(all_sessions, use_ai, args.refresh)

    if args.text:
        print_text_report(all_sessions, summaries, days)
    else:
        html = generate_html(all_sessions, summaries, days)
        REPORT_DIR.mkdir(parents=True, exist_ok=True)
        report_file = REPORT_DIR / f"session-report-{datetime.now().strftime('%Y%m%d-%H%M%S')}.html"
        with open(report_file, "w", encoding="utf-8") as f:
            f.write(html)
        cleanup_old_reports(keep=10)
        print(f"Report saved: {report_file}")
        if not args.no_open:
            webbrowser.open(str(report_file))
            print("Opened in browser.")


if __name__ == "__main__":
    main()
```

## Requirements

- Python 3.9+
- `pip install anthropic httpx`
- Claude Code installed (`~/.claude/` directory must exist)
- API key in `~/.claude/settings.json` for AI summaries (optional — `--no-ai` works without)

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
python ~/.claude/scripts/claude-session-report.py 3
python ~/.claude/scripts/claude-session-report.py 7 text
```
