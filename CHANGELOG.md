# Changelog

All notable changes to this repo are recorded here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
Versions match `.claude-plugin/plugin.json`. Dates are `YYYY-MM-DD`.

## [Unreleased]

## [0.2.0] - 2026-09-08

### Added

- `rust-review` workflow skill: pins a diff or file, applies each relevant rubric skill,
  reports findings as `path:line: <severity>: <problem>. <fix>. [<skill>]`.
- `skill-maintainer` workflow skill: the fold procedure that promotes `LEARNINGS.md`
  entries into rubric bodies and checklists, behind a review gate.
- `LEARNINGS.md`: append-only inbox for field corrections, and the two-stage
  capture then fold loop it feeds.
- Claude Code plugin packaging: `.claude-plugin/plugin.json` and
  `.claude-plugin/marketplace.json`. Installable with `/plugin`.
- `CONTEXT.md`: shared vocabulary (rubric vs workflow skill, the clippy line, the loop).

## [0.1.0] - 2026-09-08

### Added

- Rubric skills: `rust-error-design`, `rust-idioms`, `rust-async`, `rust-api-design`.
  Each carries before/after examples, counter-cases, and a review checklist.
- Repo scaffold: `README.md`, `CONTRIBUTING.md`, dual MIT / Apache-2.0 license.
