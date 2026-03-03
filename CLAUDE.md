# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A single-file Ruby CLI tool (`mr_review`) that automates GitLab Merge Request code reviews using Claude Code. It fetches an MR diff, invokes `claude -p` to produce a structured JSON review, presents each comment interactively for human validation (accept/edit/skip), then submits the approved comments as GitLab draft notes and approves or requests changes.

## Running

```bash
# Review by MR URL
./mr_review https://source.modulotech.fr/modulosource/ff/fast/core/-/merge_requests/687

# Review by project path + MR IID
./mr_review modulosource/ff/fast/core 687
```

Dependencies are installed automatically via `bundler/inline` (no separate `bundle install` needed). Requires Ruby and the `claude` CLI on PATH.

## Required Environment Variables

- `GITLAB_API_TOKEN` — Personal Access Token with `api` scope (required)
- `CLAUDE_BIN` — path to claude binary (default: `"claude"`)
- `CLAUDE_TIMEOUT` — review timeout in seconds (default: `360`)

## Architecture

The script follows a linear pipeline in `main`:

1. **Argument parsing** (`parse_args`) — accepts either a full GitLab MR URL or `<project_path> <mr_iid>`
2. **Idempotence check** (`last_reviewed_sha`) — scans MR notes for the marker tag `<!-- mr-reviewer-sha:... -->` to skip already-reviewed commits
3. **Branch checkout** (`BranchManager`) — fetches the MR source branch into a local `review/mr-<iid>` branch; restores the original branch via `at_exit`
4. **Claude Code review** (`run_claude_review`) — sends a prompt to `claude -p` requesting a JSON object with `summary`, `verdict`, and `comments[]`; extracts JSON from possibly fenced or prefixed output
5. **Interactive validation** (`CommentValidator`) — TTY-based accept/edit/skip loop per comment, with file context display
6. **Draft persistence** (`DraftState`) — saves validated review to `.mr_review_draft_<iid>.json` before any API call, enabling retry after network failures
7. **GitLab submission** (`ReviewSubmitter`) — uses raw `Net::HTTP` for draft notes (the `gitlab` gem lacks this endpoint), then bulk-publishes and optionally approves

## Key Design Decisions

- **Review language**: The Claude prompt mandates that all `body` and `summary` text be written in **French**; JSON keys/enums stay in English.
- **Verdict logic**: After interactive validation, the verdict is recalculated — if any accepted comment has severity `error` or `warning`, the verdict becomes `request_changes` regardless of Claude's original verdict.
- **Draft notes over regular notes**: Uses GitLab's draft notes API (`/draft_notes`) + bulk publish so the review appears as a single atomic batch, not individual comments.
- **No gem for draft notes**: `ReviewSubmitter` uses `Net::HTTP` directly because the `gitlab` gem doesn't expose the draft notes endpoint.
