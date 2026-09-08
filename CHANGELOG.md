# Changelog

All notable changes to this repo are recorded here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
Versions match `.claude-plugin/plugin.json`. Dates are `YYYY-MM-DD`.

## [Unreleased]

## [0.3.0] - 2026-09-08

### Added

- `rust-review` optional HTML report (`--report`): writes `.rust-review/index.html` +
  `.rust-review/findings.json` into the reviewed repo, unlighthouse-style. Self-contained
  static shell (`skills/rust-review/report/template.html`), no build step: neutral
  light UI with a persisted dark toggle, Rust syntax highlight via highlight.js, stat
  tiles, severity + rubric filters, before/after diff cards on mechanical fixes. Data
  injected inline so double-click works; `findings.json` sits next to it for diffing
  and re-render.

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
