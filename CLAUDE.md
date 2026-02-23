# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

CCNotify is a macOS-only Python tool that sends desktop notifications for Claude Code sessions via `terminal-notifier`. It hooks into Claude Code's lifecycle events (UserPromptSubmit, Stop, Notification) to track prompts, calculate task durations, and alert the user when Claude finishes or needs input.

## Commands

```bash
# Run the script (health check — should print "ok")
~/.claude/ccnotify/ccnotify.py

# Run unit tests
python -m unittest tests/test_prompt_tracker.py -v

# Run interactive test runner (all scenarios with optional delay)
python tests/run_tests.py all [delay_seconds]

# Verify database state after tests
python tests/run_tests.py verify

# Generate test data files
python tests/test_data.py
```

## Architecture

**Single-file application** — all core logic lives in `ccnotify.py`.

### Core Class: `ClaudePromptTracker`

Handles the full lifecycle: input validation → SQLite persistence → notification dispatch.

### Event Flow

```
Claude Code hook event (JSON via stdin)
  → ccnotify.py <EventType>
    → validate_input_data() — checks required fields per event type
    → ClaudePromptTracker method:
        handle_user_prompt_submit() — inserts prompt record, captures iTerm session ID
        handle_stop() — updates stoped_at, calculates duration, sends notification
        handle_notification() — detects "waiting" messages, sends notification
    → SQLite DB (~/.claude/ccnotify/ccnotify.db)
    → terminal-notifier subprocess → macOS notification
    → On click: AppleScript activates iTerm pane (or falls back to VS Code)
```

### Database

Single `prompt` table with auto-incrementing `seq` per session (via SQL trigger). Key columns: `session_id`, `prompt`, `cwd`, `seq`, `stoped_at`, `lastWaitUserAt`, `iterm_session_id`.

### Runtime Paths

- Database: `~/.claude/ccnotify/ccnotify.db`
- Logs: `~/.claude/ccnotify/ccnotify.log` (daily rotation, 1 day backup)

## Key Patterns

- **Zero external Python dependencies** — uses only the standard library (`sqlite3`, `subprocess`, `logging`, `json`, `re`, `datetime`).
- External dependency is `terminal-notifier` (Homebrew).
- iTerm session ID is captured via `osascript` AppleScript and validated with regex.
- Duration formatting is human-readable (e.g., "2m30s", "1h30m").
- The script exits with code 1 on validation/JSON errors, logs the reason. This is intentional — hook failures should not be silent.
