---
name: standup
description: "Use when the user wants a standup, daily update, worklog, or status report built from their real git activity. Keywords: standup, daily update, what did I do, worklog, yesterday today blockers, status update, scrum update."
when_to_use: "Examples: '/standup', 'write my standup', 'what did I do yesterday', 'daily update', 'status update from git'."
user-invocable: true
allowed-tools: "Read Bash(git *) Bash(gh *) Bash(date *) Bash(command *)"
---

# standup — Yesterday / Today / Blockers from live git

You assemble a copy-ready standup from the current repo's real git activity. You do NOT invent work.
You are stateless: read git (and optionally `gh`) fresh every run and persist nothing.

## GATE
1. Confirm you are inside a git work tree: `git rev-parse --is-inside-work-tree`. If it fails, print
   `Not a git repository — run /standup from inside a repo.` and STOP.
2. Read `.standup/config.json` for `privacy_mode` (default `redact_pii` if the file or key is absent).

## Arguments
Parse from the invocation:
- `--since=<spec>` — override the window (relative `3d`/`12h`/`1w` or ISO `YYYY-MM-DD`). See
  reference.md §1. If unparseable, print the accepted formats and STOP.
- `--all` — include every author (omit the author filter). Default: current user only.

## Assemble (follow reference.md exactly)
1. **Window** — resolve `SINCE` per reference.md §1 (or `--since`).
2. **Author** — resolve per reference.md §2 (`--all` removes the filter).
3. **Yesterday (ground truth)** — run the git-log recipe (§3); collect `hash / subject / date`.
   If `gh` is available (§4), add my merged PRs in the window.
4. **Today (inferred — confirm)** — from the current branch name (§3), uncommitted/staged work
   (`git status --porcelain`), and any WIP/unfinished commit subjects. If detached HEAD, skip the
   branch inference and note it.
5. **Blockers (scanned — confirm)** — scan the in-window diff for the markers in reference.md §5;
   add my open PRs awaiting review if `gh` is available.
6. **Redact** — if `privacy_mode` is `redact_pii`, apply the reference.md §6 table to ALL text now,
   before rendering. If `raw`, skip.

## Rules
- **Never fabricate.** Yesterday is only real commits/PRs. If Today or Blockers has nothing to infer,
  render the section with `_(nothing inferred — add yours)_` rather than making something up.
- If `gh` is absent or unauthenticated, skip PR enrichment and add the note in the output footer.
- If there are no in-window commits, say so and suggest `--since=<spec>`; still render Today/Blockers.
- Keep subjects verbatim (post-redaction). Do not editorialize commit messages.

## Output (render from the assembled, redacted data)
```
**Standup — <today's date>**  (window: since <SINCE>, <author scope>)

**Yesterday**
- <subject>  (<short-hash>)
- Merged PR #<n>: <title>            ← only if gh available

**Today**  _(inferred — confirm/edit)_
- <branch/WIP-based inference, or "nothing inferred — add yours">

**Blockers**  _(scanned — confirm/edit)_
- <marker> in <file>:<line> — <text>
- PR #<n> awaiting review            ← only if gh available
- <or "none found — add yours">

_<footer note: e.g. "PR data unavailable (gh not found)" when applicable>_
```

Render nothing before redaction. This block is meant to be copied straight into Slack.
