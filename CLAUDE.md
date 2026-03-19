# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A single-file Ruby CLI tool (`bin/mr-review`) distributed via Homebrew (`modulotech/tap`) that automates GitLab Merge Request code reviews using Claude Code. It fetches an MR diff, invokes `claude -p` to produce a structured JSON review, presents each comment interactively for human validation (accept/edit/skip), then submits the approved comments as GitLab draft notes and approves or requests changes. All review data is persisted to SQLite for tracking and analytics.

## Running

```bash
# Review by MR URL
./bin/mr-review https://source.modulotech.fr/modulosource/ff/fast/core/-/merge_requests/687

# Review by project path + MR IID
./bin/mr-review modulosource/ff/fast/core 687

# With explicit database path
./bin/mr-review -d sqlite://path/to/reviews.db <MR_URL>

# With explicit token
./bin/mr-review -t glpat-xxxx <MR_URL>
```

Dependencies are installed automatically via `bundler/inline` (no separate `bundle install` needed). Requires Ruby and the `claude` CLI on PATH.

## Configuration

Settings are resolved in 4 layers (highest priority wins):

1. **Defaults** — `claude_bin: "claude"`, `claude_timeout: 360`
2. **Config file** — `~/.mr-review/config.yml`
3. **Environment variables** — `GITLAB_API_TOKEN`, `DATABASE_URL`, `CLAUDE_BIN`, `CLAUDE_TIMEOUT`
4. **CLI flags** — `-d`/`--database-url`, `-t`/`--token`, `--claude-bin`, `--claude-timeout`

### Config file example (`~/.mr-review/config.yml`)

```yaml
database_url: sqlite://~/.mr-review/mr-review.db
gitlab_api_token: glpat-xxxxxxxxxxxxxxxxxxxx
claude_bin: claude
claude_timeout: 360
```

### CLI flags

- `-d` / `--database-url URL` — SQLite connection URL
- `-t` / `--token TOKEN` — GitLab API token
- `--claude-bin PATH` — Path to claude binary
- `--claude-timeout SECONDS` — Review timeout in seconds
- `-h` / `--help` — Show help

## Architecture

The script follows a linear pipeline in the `main` method:

1. **Argument parsing** (`parse_args` + `Config.load`) — OptionParser for flags, positional args for MR URL or `<project_path> <mr_iid>`
2. **Database connection** (`Database.connect`) — optional, graceful degradation if unavailable
3. **Fetch MR metadata** — GitLab API call
4. **INSERT review** — DB row with `status: pending`
5. **Idempotence check** — DB `last_reviewed_sha` first, GitLab notes fallback
6. **Resume check** — DB `find_validated_review` first, `DraftState` file fallback
7. **Branch checkout** (`BranchManager`) — fetches the MR source branch into a local `review/mr-<iid>` branch; restores the original branch via `at_exit`
8. **Claude Code review** (`run_claude_review`) — `status: reviewing`, measures duration
9. **Interactive validation** (`CommentValidator`) — returns ALL comments with `_review_status` (accepted/edited/skipped); all inserted into DB
10. **UPDATE validated** — verdict, summary, comments_kept
11. **Final review screen** — edit summary / cancel / submit
12. **GitLab submission** (`ReviewSubmitter`) — draft notes + bulk publish + optional approve
13. **UPDATE submitted** — `finished_at`, cleanup DraftState file

### Error handling

All `abort` calls have been replaced by a custom exception hierarchy:

```
MrReviewError < StandardError
  ClaudeError < MrReviewError
    ClaudeTimeoutError < ClaudeError
    ClaudeParseError < ClaudeError
```

The `main` method wraps everything in `begin...rescue`:
- `MrReviewError`, `ReviewSubmitter::ApiError` → log to DB + stderr + exit 1
- `Interrupt` → `status: cancelled` + exit 130
- `StandardError` (catch-all) → `status: error` + backtrace to DB + exit 1

A `current_phase` variable tracks the pipeline stage for error reporting.

## SQLite Schema

Three normalized tables (`CREATE TABLE IF NOT EXISTS` auto-migration at each launch):

- **`reviews`** — one row per script execution (project, MR, SHA, status lifecycle, verdict, timing)
- **`review_comments`** — every Claude comment with its validation outcome (accepted/edited/skipped, original_body if edited)
- **`review_errors`** — errors logged with phase and backtrace

The DB defaults to `~/.mr-review/mr-review.db` (SQLite). If connection fails, the script falls back to `DraftState` (local JSON file) and GitLab notes for idempotence.

## Key Design Decisions

- **Review language**: The Claude prompt mandates that all `body` and `summary` text be written in **French**; JSON keys/enums stay in English.
- **Verdict logic**: After interactive validation, the verdict is recalculated — if any accepted comment has severity `error` or `warning`, the verdict becomes `request_changes` regardless of Claude's original verdict.
- **Draft notes over regular notes**: Uses GitLab's draft notes API (`/draft_notes`) + bulk publish so the review appears as a single atomic batch, not individual comments.
- **No gem for draft notes**: `ReviewSubmitter` uses `Net::HTTP` directly because the `gitlab` gem doesn't expose the draft notes endpoint.
- **Graceful degradation**: All DB operations are guarded by `Database.connected?` — the script works identically without a database.
- **All comments tracked**: `CommentValidator` returns every comment (not just accepted), enriched with `_review_status` and `_original_body` for full audit trail.
- **Config layering**: 4-layer priority (defaults < config.yml < env < CLI) via `Config` module avoids hardcoded env var lookups.
