# 🐙 istota

[istota.xyz](https://istota.xyz)

A self-hosted AI agent that lives in your Nextcloud instance. Powered by Claude Code.

Istota joins your Nextcloud as a regular user with its own account, collaborates on files, manages calendars, handles email, and does pretty much anything Claude Code can do — through Nextcloud Talk or email. Your data stays on your stack, permissions follow Nextcloud's sharing model, and each invocation runs in a bubblewrap sandbox.

## Requirements

- Python 3.11+
- [uv](https://docs.astral.sh/uv/) for package management
- [Claude Code](https://docs.anthropic.com/en/docs/build-with-claude/claude-code) CLI installed and authenticated
- A Nextcloud instance with a dedicated user account for the bot (mounted via Rclone)
- Linux (Debian 13 recommended) for production — bubblewrap sandboxing requires Linux
- Optional: [bubblewrap](https://github.com/containers/bubblewrap) for filesystem sandboxing

## Quick start

```bash
git clone https://github.com/stefankubicki/istota.git
cd istota

# Install dependencies
uv sync

# Copy and edit config
cp config/config.example.toml config/config.toml
# Edit config/config.toml with your Nextcloud credentials

# Initialize the database
uv run istota init

# Test with a dry run (shows the full prompt without calling Claude)
uv run istota task "What's on my calendar today?" -u alice -x --dry-run

# Execute a single task
uv run istota task "What's on my calendar today?" -u alice -x

# Run the scheduler daemon
uv run istota-scheduler -d
```

## Configuration

Minimum config requires Nextcloud credentials and at least one user:

```toml
[nextcloud]
url = "https://nextcloud.example.com"
username = "istota"
app_password = "xxxxx-xxxxx-xxxxx-xxxxx-xxxxx"

[talk]
enabled = true

[users.alice]
display_name = "Alice"
email_addresses = ["alice@example.com"]
```

CalDAV is auto-derived from Nextcloud credentials — no separate calendar config needed.

Config is searched in order: `config/config.toml`, `~/.config/istota/config.toml`, `/etc/istota/config.toml`. Per-user config files go in `config/users/` (e.g. `config/users/alice.toml`). See `config/config.example.toml` for all options.

## How it works

```
Talk message ──>┐
Email ─────────>├──> SQLite queue -> Scheduler -> Claude Code -> Response
TASKS.md ──────>│
CLI ───────────>┘
```

Messages arrive through Talk polling, IMAP, TASKS.md file watching, or the CLI. The scheduler claims tasks from a SQLite queue, builds a prompt with the user's resources, skills, memory, and conversation context, then invokes Claude Code in a sandbox. Responses go back through the same channel.

Per-user worker threads handle concurrency — foreground tasks (chat) and background tasks (scheduled jobs, briefings) run on separate pools so a long-running job never blocks a conversation.

## Features

**Messaging** — Nextcloud Talk (DMs and multi-user rooms with @mention support), email (IMAP/SMTP with threading), TASKS.md file polling, CLI.

**Skills** — Loaded on demand based on prompt keywords, resource types, and source types. Ships with: Nextcloud file management, CalDAV calendar, email, web browsing (Dockerized Playwright), git/GitLab/GitHub development workflows, beancount accounting with invoicing, Karakeep bookmarks, voice transcription (faster-whisper), OCR (Tesseract), RSS/Atom/Tumblr/Are.na feeds, and more. Skills are a curated standard library rather than a plugin marketplace.

**Memory** — Per-user persistent memory (USER.md, auto-loaded into prompts), per-channel memory (CHANNEL.md), and dated memory files from nightly extraction. Hybrid BM25 + vector search (sqlite-vec, MiniLM) across conversations and memory files.

**Scheduling** — Cron jobs via CRON.md (AI prompts or shell commands), natural-language reminders as one-shot cron entries, scheduled briefings with calendar/markets/news/todos components, invoice generation schedules.

**Briefings** — Configurable morning/evening summaries. Components include calendar events, market data (futures, indices via yfinance + FinViz), news from RSS feeds, todos, email summaries, notes, and reminders. Output to Talk, email, or both.

**Heartbeat monitoring** — User-defined health checks: file age, shell commands, URL health, calendar conflicts, task deadlines, and system self-checks. Cooldowns, quiet hours, and per-check intervals.

**Multi-user** — Per-user config files, resource permissions, worker pools, and filesystem sandboxing. Admin/non-admin isolation. Each user gets their own Nextcloud workspace with config files, exports, and memory.

**Security** — Bubblewrap sandbox per invocation (PID namespace, restricted mounts, credential isolation). Non-admin users can't see the database, other users' files, or system config. Deferred DB writes via JSON files for sandboxed operations. Credential stripping from subprocess environments.

**Constitution** — An emissaries layer defines how the agent reasons about data, handles the boundary between private and public action, and what it owes to people beyond its operator. Customizable per-user persona on top.

## User workspace

Each user gets a shared Nextcloud folder:

```
/Users/alice/istota/
├── config/
│   ├── USER.md          # Persistent memory
│   ├── TASKS.md         # File-based task queue
│   ├── PERSONA.md       # Personality customization
│   ├── BRIEFINGS.md     # Briefing schedule
│   ├── CRON.md          # Scheduled jobs
│   └── HEARTBEAT.md     # Health monitoring config
├── exports/             # Bot-generated files
└── examples/            # Reference documentation
```

## Deployment

Istota expects its own Debian/Ubuntu VM, separate from your Nextcloud server. Nextcloud files are accessed via an rclone FUSE mount.

```bash
# Download and run the interactive installer
curl -fsSL https://raw.githubusercontent.com/stefankubicki/istota/main/deploy/install.sh -o install.sh
sudo bash install.sh
```

The installer walks through Nextcloud connection, user setup, and optional features. It validates credentials, sets up the rclone mount, generates all config files, and starts the scheduler service. Re-run with `--update` to pull code and regenerate config without re-prompting.

An Ansible role (`deploy/ansible/`) is also available for infrastructure-as-code deployments. See `deploy/README.md` for details.

Preview the installer output without making any system changes:

```bash
bash deploy/install.sh --dry-run
```

## Development

```bash
uv sync                                    # Install dependencies
uv run pytest tests/ -v                    # Run tests (~1950 unit tests)
uv run pytest -m integration -v            # Integration tests (needs live config)
uv run istota task "hello" -u alice -x     # Test execution
```

Optional dependency groups:

```bash
uv sync --extra memory-search    # sqlite-vec + sentence-transformers for semantic search
uv sync --extra whisper           # faster-whisper for audio transcription
```

## Further reading

- [ARCHITECTURE.md](ARCHITECTURE.md) — detailed system architecture and design decisions
- [DEVLOG.md](DEVLOG.md) — history of changes and design rationale

## License

[AGPL-3.0-or-later](LICENSE)
