# Changelog

## [Unreleased]

### Fixed

- Embed SHA idempotence marker into the pipeline summary note instead of publishing it as a separate empty-looking comment.

## [0.1.0] - 2026-03-19

### Added

- Headless mode (`-H`/`--headless`): skip interactive validation and submit all comments directly.
- Automated GitLab Merge Request review via Claude Code CLI.
- Interactive comment validation with accept/edit/skip workflow.
- Draft notes API for atomic batch review submission on GitLab.
- SQLite persistence for review tracking and analytics (default: `~/.mr-review/mr-review.db`, graceful degradation).
- Resume support for interrupted reviews via database or local draft state files.
- 4-layer configuration: defaults, `~/.mr-review/config.yml`, environment variables, CLI flags.
- Branch checkout management with automatic restore on exit.
- Verdict recalculation based on accepted comment severities.
- Homebrew distribution via `modulotech/tap`.
