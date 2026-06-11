# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Normal run (collect + analyze + notify per schedule)
python -m trendradar

# Debug mode (verbose logging)
python -m trendradar --debug

# Environment health check
python -m trendradar --doctor

# Show current schedule status
python -m trendradar --show-schedule

# Test notification channels
python -m trendradar --test-notification

# Install dependencies
uv sync                          # recommended
pip install -r requirements.txt
```

No test suite exists. No linter/formatter is configured.

## Architecture

TrendRadar is a financial news aggregation engine. Each invocation runs 4 phases:

```
collect → filter → analyze → report+notify
```

**Entry point**: `trendradar/__main__.py::main()` → creates `NewsAnalyzer` → calls `run()`.

### Key abstractions

- **`AppContext`** (`trendradar/context.py`) — DI container and central service locator. Provides lazy-singleton access to storage, scheduler, AI filter, notification dispatcher. All components receive dependencies through this rather than module-level globals. Also contains `run_ai_filter()` (264-line orchestration of the full AI classification pipeline) and `convert_ai_filter_to_report_data()`.

- **`Scheduler`** (`trendradar/core/scheduler.py`) — Timeline-based scheduling engine. `resolve()` examines the current time, checks `week_map → day_plan → periods`, finds which period (if any) contains the current time, merges period settings with defaults, and returns a `ResolvedSchedule` dict with `{collect, analyze, push, report_mode, ai_mode, once_analyze, once_push}`.

- **`NewsAnalyzer`** (`trendradar/__main__.py`) — Per-invocation orchestrator. Owns the `AppContext`. Its `run()` method calls `_crawl_data()` → `_crawl_rss_data()` → `_execute_mode_strategy()` which resolves the schedule, loads data per mode, runs the analysis pipeline, and sends notifications.

- **`StorageManager`** (`trendradar/storage/manager.py`) — Unified storage singleton. Delegates to either `LocalStorageBackend` or `RemoteStorageBackend` (S3-compatible). Both share SQLite operations via `SQLiteStorageMixin`.

### Three report modes

| Mode | Scope | Use case |
|------|-------|----------|
| `current` | News currently on hotlists right now | Quick check-ins |
| `daily` | All news from today | Full-day summaries |
| `incremental` | Only new items since last push | Avoiding duplicates |

The mode is determined by timeline period, falling back to `report.mode` in config.yaml.

### Configuration layers (priority: low → high)

1. `config/config.yaml` — base settings (486 lines, 13 sections)
2. Environment variables — 40+ keys override YAML values (`AI_API_KEY`, `FEISHU_WEBHOOK_URL`, `SCHEDULE_PRESET`, etc.)
3. `config/timeline.yaml` — time-based behavior overrides (periods define what happens at what time)

Config loading is in `trendradar/core/loader.py` — each `_load_X_config()` function reads a YAML section then applies env var overrides via `_get_env_str()`, `_get_env_bool()`, etc.

### Two-layer scheduling

**Layer 1 — Cron triggers** (`.github/workflows/crawler.yml`): GitHub Actions schedule decides when the program wakes up. Currently 3 times/day: 05:29, 15:24, 17:29 UTC (arriving ~09:00, ~18:55, ~21:00 Beijing after ~3.5h typical GH Actions delay).

**Layer 2 — Timeline periods** (`config/timeline.yaml`): When awakened, the scheduler checks which period contains the current time and applies that period's behavior. Periods can override: `collect`, `analyze`, `push`, `report_mode`, `ai_mode`, `filter_method`, `frequency_file`, `interests_file`, and `once` dedup controls.

Key concept: Cron says *when to wake up*. Timeline says *what to do* after waking.

### Filtering strategies

- **`keyword`** — Parses `config/frequency_words.txt` (supports `+required`, `!exclude`, `@max_count`, `//regex`, `[aliases]`, `=> display_name`). Fast, no API cost.
- **`ai`** — `AIFilter` does 3-stage classification: extract tags from interest description → update tags when interests change → classify each news item against tags with relevance scores (0.0–1.0). Results cached in SQLite (`ai_filter_tags`, `ai_filter_results` tables). Falls back gracefully to keyword matching on failure.

Set via `filter.method` in config.yaml, overridable per timeline period.

### Notification system

`NotificationDispatcher.dispatch_all()` sends to all configured channels: Feishu, DingTalk, WeChat Work, Telegram, Email, ntfy, Bark, Slack, generic webhook. Multi-account support via `;`-separated URLs. Content is translated (if `ai_translation.enabled`), rendered per-channel, split to channel-specific byte limits, and sent in batches.

### AI subsystem (`trendradar/ai/`)

- **`AIClient`** — LiteLLM wrapper, supports 100+ providers via `provider/model` format
- **`AIFilter`** — Tag extraction + batch news classification
- **`AIAnalyzer`** — Generates structured reports (trends, sentiment, signals, RSS insights, event impact, strategy)
- **`AITranslator`** — Batch title translation

API key goes in `.env` as `AI_API_KEY`, never in config.yaml.

### Data output structure

```
output/
├── {date}/data.sqlite           # Per-day database
├── html/{date}/{time}.html      # Timestamped report
├── html/latest/{mode}.html      # Latest per mode (overwritten)
├── html/latest/index.html       # Latest overall
└── meta/doctor_report.json      # Health check output
```

Root `index.html` is also written for GitHub Pages deployment.

## Important patterns

- **No global state**: Everything flows through `AppContext`. Components receive config dicts and function references as constructor args.
- **Graceful degradation**: AI filter fails → keyword fallback. AI analysis fails → skip analysis, continue. Remote storage fails → local fallback.
- **API keys**: Never in `config/config.yaml`. Always via `.env` file (`.env.example` is committed as template) or environment variables.
- **Cross-day periods**: Timeline periods with `start > end` (e.g., `23:30–00:10`) are treated as spanning midnight.
- **Dedup**: `once.push` / `once.analyze` in timeline periods prevent duplicate work within a window. State tracked in SQLite `period_executions` table.
- **UV is preferred** over pip for dependency management. `uv sync` for installation.
- **Update README.md** when pushing config/schedule changes to GitHub.
