# Changelog

## [Unreleased]

## [0.3.1] - 2026-04-13

### Fixed

- `issue-md` invocation failed with a Bundler error when called from mr-review (itself running under `bundler/inline`). The child process inherited the parent's `BUNDLE_*` environment, which broke issue-md's own `bundler/inline` setup. The shellout is now wrapped in `Bundler.with_unbundled_env`.

## [0.3.0] - 2026-04-13

### Added

- Accept a GitLab issue or work_item URL as input. mr-review looks up MRs related to the issue (`/issues/:iid/related_merge_requests`) and reviews the matching MR. With a single related MR it proceeds automatically; with multiple, the user is prompted to pick one (in `--headless` mode, this is a fatal error and the user is asked to pass the MR URL as an additional argument: `mr-review <ISSUE_URL> <MR_URL>`).
- When an issue URL is provided, the issue is exported to markdown via the `issue-md` CLI and injected into the Claude review and consolidation prompts as `## Issue context`, so the reviewer can judge whether the changes match the business intent. New `--issue-md PATH` flag, `ISSUE_MD_BIN` env var, and `issue_md_bin` config key (default: `issue-md`).

## [0.2.0] - 2026-04-10

### Added

- Multi-pass review with consolidation: run Claude review N times (default: 3, configurable via `--review-passes`, `REVIEW_PASSES`, or config file), accumulate all comments, then pass them through a consolidation Claude call that eliminates praise-only remarks, removes false positives, deduplicates overlapping comments, and verifies relevance. The consolidated result feeds into the existing validation/submission flow.

### Changed

- mr-review no longer auto-approves MRs on GitLab. The verdict (APPROVE / REQUEST CHANGES) is now displayed in the summary note instead.

## [0.1.3] - 2026-04-02

### Added

- `-v` / `--version` CLI flag to display the current version.
- `-l` / `--language LANG` CLI flag to set the review language (default: French). Also configurable via `language` in config file or `MR_REVIEW_LANGUAGE` env var.

## [0.1.2] - 2026-03-24

### Fixed

- Fix pipeline summary comments showing literal `\n` instead of actual newlines in GitLab notes.

## [0.1.1] - 2026-03-23

### Fixed

- Embed SHA idempotence marker into the pipeline summary note instead of publishing it as a separate empty-looking comment.
- Handle `diff_refs` being nil on freshly created MRs: retry up to 3 times (5s interval) before raising a clear error instead of crashing with `NoMethodError`.

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
