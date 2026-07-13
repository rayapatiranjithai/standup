# Changelog

## 0.1.0 — 2026-07-14

- Initial release of `standup`.
- `/standup` skill: assembles a redacted Yesterday/Today/Blockers update from live git,
  with optional GitHub PR enrichment via `gh`. Stateless — no hooks, no stored data.
- Defaults: window = since last working day (override `--since`); author = current git user
  (override `--all`). Privacy: `redact_pii` (default) or `raw`.
