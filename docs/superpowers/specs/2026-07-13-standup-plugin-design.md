# standup plugin — design spec

**Date:** 2026-07-13
**Status:** Approved (design), pending spec review
**Author:** Ranjith Rayapati

## Summary

`standup` is a Claude Code plugin with a single slash command, `/standup`, that assembles a
copy-ready **Yesterday / Today / Blockers** update from live git activity (optionally enriched with
GitHub PR data via `gh`). It is **stateless**: no hooks, no journal, no persisted per-session data.
Every run computes from the current git state on demand.

It ships as a **second plugin** in the existing `rayapatiranjithai-plugins` marketplace, reusing the
architectural patterns of `prompt-analyzer` (a skill that runs a READ→assemble→redact→render
pipeline, a `privacy_mode` config contract, and the PII/secret redaction regex table).

## Goals

- Produce an accurate standup from **ground truth** (commits, merged PRs) rather than memory.
- Zero setup and zero background overhead — no hook, no state file.
- Safe to paste into a public channel — redact secrets/PII before rendering.
- Degrade gracefully when optional inputs (`gh`, PRs) are unavailable.

## Non-goals (explicitly out of scope for v0.1)

- Multi-repo aggregation (fast-follow).
- Weekly-report / changelog / on-call-handoff variants (fast-follow).
- Direct posting to Slack or any external service (the plugin only renders text to copy).
- Any hook or passive background capture of test/build runs.

## User-facing behavior

### Command

```
/standup [--since=<spec>] [--all]
```

- `--since=<spec>` — override the time window. Accepts a relative duration (`3d`, `12h`, `1w`) or an
  ISO date (`2026-07-10`). Default: **since the last working day**.
- `--all` — include every author's commits (team summary). Default: **only the current git user**.

### Time window resolution (default)

"Since the last working day" means:
- On **Tuesday–Friday**: since 00:00 of the previous calendar day.
- On **Monday**: since 00:00 of the previous **Friday** (spans the weekend).
- On **Sunday/Saturday**: since 00:00 of the previous Friday.

This is a heuristic, not a holiday-aware calendar. `--since` overrides it whenever the default is
wrong for the user's schedule. No last-run timestamp is stored.

### Author resolution (default)

Filter commits to the current repo's `git config user.email` (fall back to `user.name` if email is
unset). `--all` removes the author filter.

## Data flow

```
/standup [--since=X] [--all]
  1. preflight   → confirm inside a git work tree; else clear error + stop
  2. window      → resolve --since or default "last working day" → <since-timestamp>
  3. author      → resolve --all or default current git user → <author-filter>
  4. Yesterday   → git log --since=<ts> [--author=<a>] : subjects grouped by branch/day
                 → (gh present) my merged PRs since <ts>
  5. Today       → infer from: current branch name, uncommitted changes (git status),
                   and the most recent WIP/unfinished commit subjects
  6. Blockers    → scan recent diff + tracked files for TODO/FIXME/BLOCKED/HACK/WIP markers
                   introduced in-window; (gh present) my open PRs awaiting review
  7. redact      → apply privacy_mode (default redact_pii) to ALL assembled text
  8. render      → markdown Y/T/B block, Today/Blockers labeled "inferred — confirm"
```

## Sections and trust labeling

- **Yesterday** — ground truth. Commit subjects (grouped by branch) + merged PRs. Rendered plainly.
- **Today** — *inferred, confirm*. Derived from current branch name, uncommitted/staged work, and
  unfinished WIP commits. Rendered under a "(inferred — confirm/edit)" note.
- **Blockers** — *scanned, confirm*. From in-window `TODO`/`FIXME`/`BLOCKED`/`HACK`/`WIP` markers
  and open PRs awaiting review. Rendered under a "(scanned — confirm/edit)" note.

The skill MUST NOT fabricate Today/Blockers content. If nothing is inferable, it says so and prompts
the user to fill the section in.

## Components

| Path | Purpose |
|---|---|
| `.claude-plugin/plugin.json` | Plugin manifest — name `standup`, version `0.1.0`, author, repo, license. |
| `skills/standup/SKILL.md` | The `/standup` skill: preflight, window/author resolution, the READ→assemble→redact→render pipeline, the output template, and the "never fabricate" rule for Today/Blockers. |
| `skills/standup/reference.md` | Exact git & `gh` command recipes, window/author resolution rules with worked examples, and a **copy** of the PII/secret redaction regex table (Claude Code plugins cannot cross-reference another plugin's files, so the table is duplicated here). |
| `.standup/config.json` | `privacy_mode` + `_help`. Tracked in git. Because the plugin is **stateless** (nothing is persisted), only two modes are meaningful and offered: `redact_pii` (default — mask secrets/PII in the rendered output) and `raw` (render verbatim). The `hashed`/`metadata_only` modes from prompt-analyzer are intentionally omitted — they govern how stored prompt text is persisted, and standup stores nothing. |
| `README.md`, `LICENSE`, `CHANGELOG.md`, `.gitignore` | Project standards, matching prompt-analyzer's conventions. |

## Packaging / marketplace integration

**Decision (needs confirmation in spec review):** `standup` lives in its **own git repository**
(`rayapatiranjithai/standup`) and is added to the **existing** `rayapatiranjithai-plugins`
marketplace via a git-source plugin entry, rather than restructuring the `prompt-analyzer` repo.

Rationale: the current marketplace lists `prompt-analyzer` with `"source": "."` (plugin at repo
root). Adding a second root-source plugin to the same repo would require moving `prompt-analyzer`
into a subdirectory (a breaking layout change for existing installers). Referencing `standup` from
its own repo keeps both plugins in one marketplace with no disruption to `prompt-analyzer`.

The marketplace entry (added to `prompt-analyzer/.claude-plugin/marketplace.json`) will point at the
standup repo as its source. Exact source syntax to be finalized against the marketplace schema
during implementation.

Install (once published):
```
/plugin marketplace add rayapatiranjithai/prompt-analyzer
/plugin install standup@rayapatiranjithai-plugins
```

## Redaction & privacy

- Reuse the redaction approach from `prompt-analyzer/skills/analyze-prompt/reference.md`: mask
  emails, phones, cards, SSNs, API keys, long hex/JWT tokens, IPs, credentials, and contextual names
  before rendering. The regex table is **copied** into `skills/standup/reference.md`.
- Controlled by `.standup/config.json` `privacy_mode`: `redact_pii` (default) or `raw`.
- Redaction runs in step 7, **before** any output is rendered — nothing unredacted is ever shown.
- The plugin makes **no network calls** and persists nothing; it only reads git/`gh` and prints text.

## Error handling

| Condition | Behavior |
|---|---|
| Not inside a git work tree | Print a clear one-line error and stop. |
| No commits in the resolved window | Say so; suggest widening with `--since`. Still render Today/Blockers if inferable. |
| `gh` not installed or not authenticated | Skip PR enrichment silently; add a one-line note that PR data was unavailable. |
| `--since` value unparseable | Print the accepted formats and stop. |
| Detached HEAD / no branch | Omit branch-based Today inference; note it. |

## Testing strategy

Because the skill is instructions executed by Claude (not compiled code), "tests" are
scenario checks run against a real git repo:

1. **Happy path** — repo with several in-window commits by the current user → correct Yesterday
   grouping; Today reflects current branch; Blockers surface a planted `FIXME`.
2. **Empty window** — no in-window commits → graceful "no activity" message + `--since` hint.
3. **`--all`** — commits by multiple authors → all included.
4. **`--since` override** — relative (`3d`) and ISO date both resolve correctly.
5. **No `gh`** — PR enrichment skipped with a note; git-only output still correct.
6. **Not a git repo** — clean error, no traceback.
7. **Redaction** — a commit subject containing a fake API key is masked in the output.

## Open questions for spec review

1. **Packaging** — confirm the "own repo + git-source marketplace entry" decision above, vs.
   restructuring the prompt-analyzer repo into a monorepo marketplace.
2. Any additional marker keywords for Blockers beyond `TODO/FIXME/BLOCKED/HACK/WIP`?
