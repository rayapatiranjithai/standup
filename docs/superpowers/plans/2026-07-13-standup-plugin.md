# standup Plugin Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `standup`, a stateless Claude Code plugin whose `/standup` command assembles a copy-ready, redacted Yesterday/Today/Blockers update from live git (and optional GitHub PRs).

**Architecture:** A single user-invocable skill (`skills/standup/SKILL.md`) that runs a READ→assemble→redact→render pipeline over `git`/`gh` output. No hooks, no persisted data. A `reference.md` holds command recipes, window/author resolution rules, and a copied PII/secret redaction regex table. A `.standup/config.json` holds a two-value `privacy_mode`. The plugin lives in its own repo and is added to the existing `rayapatiranjithai-plugins` marketplace via a git-source entry.

**Tech Stack:** Claude Code plugin (Markdown skills + JSON manifests), `git`, optional `gh` CLI, POSIX `date`. No language runtime, no dependencies.

## Global Constraints

- Plugin name: `standup`; version `0.1.0`; author `Ranjith Rayapati <https://github.com/rayapatiranjithai>`; license MIT.
- Git identity for ALL commits in this repo: `Ranjith Rayapati <248677512+rayapatiranjithai@users.noreply.github.com>` (already set as repo-local config).
- **Stateless:** no hooks, no journal, no session data file. Every run computes from current git state.
- **Never fabricate** Today/Blockers content; label them "inferred — confirm/edit". Yesterday is ground truth.
- `privacy_mode`: only `redact_pii` (default) and `raw`. Redaction runs **before** any render.
- Degrade gracefully: missing `gh`, no commits in window, not-a-git-repo, detached HEAD, unparseable `--since` each have a defined message and never crash.
- Defaults: window = since last working day; author = current git user; `--since` and `--all` override.
- No network calls; the plugin only reads `git`/`gh` and prints text.
- Repo root for this plugin: `/Users/ranjithrayapati/plugins/standup`.
- Verification surface: `claude plugin validate .` (structure) + JSON validity + scenario checks against a throwaway git fixture (behavior). There is no unit-test runner.

---

### Task 1: Plugin manifest + repo standards

**Files:**
- Create: `.claude-plugin/plugin.json`
- Create: `README.md`
- Create: `LICENSE`
- Create: `CHANGELOG.md`
- (exists) `.gitignore`

**Interfaces:**
- Produces: a validatable plugin at repo root with name `standup`, version `0.1.0`. Later tasks add `skills/standup/*` and `.standup/config.json` under this root.

- [ ] **Step 1: Write the manifest**

Create `.claude-plugin/plugin.json`:

```json
{
  "name": "standup",
  "description": "Assembles a copy-ready Yesterday/Today/Blockers standup from live git activity (and optional GitHub PRs). Fully local, stateless, no telemetry.",
  "version": "0.1.0",
  "author": {
    "name": "Ranjith Rayapati",
    "url": "https://github.com/rayapatiranjithai"
  },
  "homepage": "https://github.com/rayapatiranjithai/standup",
  "repository": "https://github.com/rayapatiranjithai/standup",
  "license": "MIT"
}
```

- [ ] **Step 2: Write the LICENSE**

Create `LICENSE` — MIT license, copyright holder `Ranjith Rayapati`, year `2026`. Copy the standard MIT text (identical to `prompt-analyzer/LICENSE`, changing nothing but keeping the same holder).

- [ ] **Step 3: Write the CHANGELOG**

Create `CHANGELOG.md`:

```markdown
# Changelog

## 0.1.0 — 2026-07-14

- Initial release of `standup`.
- `/standup` skill: assembles a redacted Yesterday/Today/Blockers update from live git,
  with optional GitHub PR enrichment via `gh`. Stateless — no hooks, no stored data.
- Defaults: window = since last working day (override `--since`); author = current git user
  (override `--all`). Privacy: `redact_pii` (default) or `raw`.
```

- [ ] **Step 4: Write a minimal README**

Create `README.md` with: one-paragraph description, an "Install" section (see below), a "Usage" section documenting `/standup`, `--since=<spec>`, `--all`, the three sections and their trust labels, the `gh`-optional note, and a "Configuration" section documenting `.standup/config.json` `privacy_mode`. Install section:

````markdown
## Install

```
/plugin marketplace add rayapatiranjithai/prompt-analyzer
/plugin install standup@rayapatiranjithai-plugins
```
````

State explicitly: "Runs entirely locally, makes no network calls, and persists nothing."

- [ ] **Step 5: Validate structure**

Run: `cd /Users/ranjithrayapati/plugins/standup && claude plugin validate .`
Expected: `✔ Validation passed` (or a clear message that only a plugin — not a marketplace — is present; if validate requires a marketplace manifest, note it and proceed — marketplace wiring is Task 6).

- [ ] **Step 6: Commit**

```bash
cd /Users/ranjithrayapati/plugins/standup
git add .claude-plugin/plugin.json README.md LICENSE CHANGELOG.md
git commit -m "Add standup plugin manifest and repo standards"
```

---

### Task 2: Privacy config contract

**Files:**
- Create: `.standup/config.json`

**Interfaces:**
- Produces: `privacy_mode` key read by the skill in Task 4. Values: `redact_pii` (default), `raw`.

- [ ] **Step 1: Write the config**

Create `.standup/config.json`:

```json
{
  "privacy_mode": "redact_pii",
  "_help": {
    "privacy_mode": "redact_pii (default) = mask secrets/PII in the rendered standup before showing it | raw = render verbatim. Only these two modes apply — standup is stateless and persists nothing, so prompt-analyzer's hashed/metadata_only modes do not exist here."
  }
}
```

- [ ] **Step 2: Verify JSON validity**

Run: `python3 -c "import json; d=json.load(open('/Users/ranjithrayapati/plugins/standup/.standup/config.json')); assert d['privacy_mode']=='redact_pii'; print('ok')"`
Expected: `ok`

- [ ] **Step 3: Commit**

```bash
cd /Users/ranjithrayapati/plugins/standup
git add .standup/config.json
git commit -m "Add privacy_mode config contract"
```

---

### Task 3: Reference — command recipes, resolution rules, redaction table

**Files:**
- Create: `skills/standup/reference.md`

**Interfaces:**
- Produces: the exact git/gh recipes, window/author resolution rules, and PII regex table that `SKILL.md` (Task 4) references by section number.

- [ ] **Step 1: Write the reference document**

Create `skills/standup/reference.md` with these sections verbatim:

````markdown
# standup — Reference

Detail kept out of SKILL.md. Consult for exact commands, window/author resolution, and redaction.

## 1. Window resolution ("since last working day")

Determine the lookback in days `D` from today's ISO weekday `u` (`date +%u`; 1=Mon … 7=Sun):

| Today (`u`) | Previous working day | `D` (days back) |
|---|---|---|
| 1 (Mon) | Friday | 3 |
| 2–5 (Tue–Fri) | previous day | 1 |
| 6 (Sat) | Friday | 1 |
| 7 (Sun) | Friday | 2 |

Compute the since-timestamp at local midnight `D` days ago. macOS (BSD date) first, GNU fallback:

```bash
u=$(date +%u)
case "$u" in 1) D=3;; 6) D=1;; 7) D=2;; *) D=1;; esac
SINCE=$(date -v-${D}d '+%Y-%m-%d 00:00:00' 2>/dev/null || date -d "-${D} days" '+%Y-%m-%d 00:00:00')
```

`--since=<spec>` overrides. Accept:
- relative duration: `3d`, `12h`, `1w` → pass straight to `git log --since="3 days ago"` style
  (translate `d`→`days`, `h`→`hours`, `w`→`weeks`).
- ISO date `YYYY-MM-DD` → use as `--since="YYYY-MM-DD 00:00:00"`.
- anything else → print accepted formats and stop (see SKILL error table).

## 2. Author resolution

Default: current user. `--all`: no author filter.

```bash
ME=$(git config user.email); [ -z "$ME" ] && ME=$(git config user.name)
# default:  git log --author="$ME" ...
# --all:    omit --author
```

## 3. Git recipes (Yesterday / Today / Blockers inputs)

```bash
# Yesterday: my in-window commits, subject + branch grouping
git log --since="$SINCE" ${AUTHOR:+--author="$ME"} --pretty='%h%x09%s%x09%cI' --all
# current branch (Today)
git rev-parse --abbrev-ref HEAD           # prints HEAD if detached
# uncommitted / staged work (Today)
git status --porcelain
# in-window diff for Blocker markers
git log --since="$SINCE" ${AUTHOR:+--author="$ME"} -p --no-color
```

Group Yesterday commits by branch when derivable (`git branch --contains <hash>` is expensive; for
v0.1 group by day from `%cI` and list branch only for the current HEAD).

## 4. GitHub recipes (optional — skip silently if `gh` absent/unauthed)

Detect: `command -v gh >/dev/null && gh auth status >/dev/null 2>&1`.

```bash
# my merged PRs since the window (Yesterday)
gh pr list --state merged --author "@me" --search "merged:>=$(date -v-${D}d '+%Y-%m-%d' 2>/dev/null || date -d "-${D} days" '+%Y-%m-%d')" --json number,title,mergedAt
# my open PRs awaiting review (Blockers)
gh pr list --state open --author "@me" --json number,title,reviewDecision
```

If either command errors, treat PR data as unavailable and add the one-line note.

## 5. Blocker marker scan

Scan the in-window diff (section 3) additions for these markers (case-insensitive, word-boundary):
`TODO`, `FIXME`, `BLOCKED`, `HACK`, `WIP`. Report as `marker in <file>:<line> — <text>`. Also include
open PRs awaiting review from section 4.

## 6. PII / secret redaction table

Apply (case-insensitive) to ALL assembled text before render when `privacy_mode` is `redact_pii`.
Replace the match with the bracket token. Run specific patterns (email, key, JWT) before generic
ones (TOKEN, PHONE); longer match wins on overlap.

| Token | Regex (ECMAScript) | Notes |
|---|---|---|
| `[EMAIL]` | `[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}` | |
| `[KEY]` | `\b(?:sk-|pk-|ghp_|xox[baprs]-|AKIA|AIza)[A-Za-z0-9_\-]{10,}\b` | API-key prefixes |
| `[JWT]` | `\beyJ[A-Za-z0-9_\-]+\.[A-Za-z0-9_\-]+\.[A-Za-z0-9_\-]+\b` | JWTs |
| `[TOKEN]` | `\b[A-Fa-f0-9]{32,}\b` | long hex (hashes/tokens) |
| `[CARD]` | `\b(?:\d[ -]?){13,16}\b` | card-length digit runs |
| `[SSN]` | `\b\d{3}-\d{2}-\d{4}\b` | US SSN shape |
| `[IP]` | `\b(?:\d{1,3}\.){3}\d{1,3}\b` | IPv4 |
| `[CREDENTIAL]` | `(?i)(password\|passwd\|secret\|api[_-]?key)\s*[:=]\s*\S+` | key/value secrets |

Note: commit hashes are short (7–12 hex) and will NOT match `[TOKEN]` (32+); leave them intact.
````

- [ ] **Step 2: Verify the file exists and lists all five Blocker markers**

Run: `grep -o -E 'TODO|FIXME|BLOCKED|HACK|WIP' /Users/ranjithrayapati/plugins/standup/skills/standup/reference.md | sort -u | tr '\n' ' '`
Expected: `BLOCKED FIXME HACK TODO WIP`

- [ ] **Step 3: Commit**

```bash
cd /Users/ranjithrayapati/plugins/standup
git add skills/standup/reference.md
git commit -m "Add reference: recipes, resolution rules, redaction table"
```

---

### Task 4: The `/standup` skill

**Files:**
- Create: `skills/standup/SKILL.md`

**Interfaces:**
- Consumes: `.standup/config.json` `privacy_mode` (Task 2); `skills/standup/reference.md` sections 1–6 (Task 3).
- Produces: the user-invocable `/standup` command.

- [ ] **Step 1: Write the skill**

Create `skills/standup/SKILL.md`:

````markdown
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
````

- [ ] **Step 2: Validate the plugin now includes the skill**

Run: `cd /Users/ranjithrayapati/plugins/standup && claude plugin validate . 2>&1; grep -c 'name: standup' skills/standup/SKILL.md`
Expected: validation passes (or the Task 1 note) and grep prints `1`.

- [ ] **Step 3: Commit**

```bash
cd /Users/ranjithrayapati/plugins/standup
git add skills/standup/SKILL.md
git commit -m "Add /standup skill"
```

---

### Task 5: Behavior scenario checks against a git fixture

**Files:**
- Create (throwaway, not committed): a fixture repo under the scratchpad.

**Interfaces:**
- Consumes: the full plugin from Tasks 1–4. This task changes no plugin files unless a scenario reveals a defect (then fix the relevant file and re-run).

- [ ] **Step 1: Build the fixture repo**

```bash
FX=/private/tmp/claude-502/-Users-ranjithrayapati-plugins/1717f209-561d-4a2e-9b9b-be2973ce23c1/scratchpad/standup-fixture
rm -rf "$FX"; mkdir -p "$FX"; cd "$FX"; git init -q
git config user.email me@example.com; git config user.name Me
printf 'v1\n' > a.txt; git add a.txt; git commit -q -m "Add feature A"
printf '// FIXME: handle nulls\nkey=sk-abcdef0123456789\n' > b.txt; git add b.txt; git commit -q -m "Add parser (WIP)"
git checkout -q -b feature/login
printf 'wip\n' > login.txt; git add login.txt; git commit -q -m "Start login flow"
echo "fixture at $FX"; git log --oneline
```

- [ ] **Step 2: Run the scenario checklist**

Manually invoke the `/standup` logic (per SKILL.md) against `$FX` and confirm each row. Because the
skill is executed by Claude, "run" means: follow the SKILL.md steps using the fixture as CWD and
inspect the rendered block.

| # | Scenario | Setup | Expected |
|---|---|---|---|
| 1 | Happy path | default `/standup` in `$FX` | Yesterday lists "Add feature A", "Add parser (WIP)", "Start login flow"; Today references branch `feature/login`; Blockers surfaces the `FIXME` in `b.txt`. |
| 2 | Redaction | scenario 1, `privacy_mode=redact_pii` | `sk-abcdef0123456789` is masked to `[KEY]` in output. |
| 3 | Empty window | `/standup --since=2000-01-01`... then a future date `--since=2999-01-01` | "no activity" message + `--since` hint; no crash. |
| 4 | `--all` | add a commit as a different author, run `/standup --all` | the other author's commit appears; default run excludes it. |
| 5 | `--since` override | `/standup --since=3d` and `/standup --since=2026-07-01` | both resolve; commits filtered accordingly. |
| 6 | No `gh` | temporarily ensure `gh` unavailable (or unauthed) | PR lines omitted; footer notes PR data unavailable; git output intact. |
| 7 | Not a git repo | run in `/tmp` (non-repo) | clean "Not a git repository" message, no traceback. |

- [ ] **Step 3: Fix any defects and re-run**

If any scenario fails, edit the responsible file (`SKILL.md` or `reference.md`), commit with a
`fix:`-style message, and re-run that scenario. Repeat until all seven pass.

- [ ] **Step 4: Record the result**

Append a short "Verified 2026-07-14: scenarios 1–7 pass" line to `CHANGELOG.md` under 0.1.0, then:

```bash
cd /Users/ranjithrayapati/plugins/standup
git add CHANGELOG.md
git commit -m "Verify standup scenarios 1-7 against git fixture"
```

---

### Task 6: Marketplace integration + publish

**Files:**
- Modify: `/Users/ranjithrayapati/plugins/prompt-analyzer/.claude-plugin/marketplace.json`
- (GitHub) create repo `rayapatiranjithai/standup`, push, tag.

**Interfaces:**
- Consumes: the finished standup repo (Tasks 1–5) and the existing marketplace file.
- Produces: `standup` installable via `standup@rayapatiranjithai-plugins`.

- [ ] **Step 1: Create the GitHub repo and push**

```bash
cd /Users/ranjithrayapati/plugins/standup
gh repo create rayapatiranjithai/standup --public --source=. --remote=origin --push
git tag -a v0.1.0 -m "standup 0.1.0"; git push origin v0.1.0
```

- [ ] **Step 2: Add the git-source plugin entry to the existing marketplace**

Edit `/Users/ranjithrayapati/plugins/prompt-analyzer/.claude-plugin/marketplace.json`; add a second
object to the `plugins` array (keep `prompt-analyzer` unchanged). Use the marketplace schema's
git-source form; the intended entry is:

```json
{
  "name": "standup",
  "source": { "source": "github", "repo": "rayapatiranjithai/standup" },
  "description": "Assembles a copy-ready Yesterday/Today/Blockers standup from live git activity (and optional GitHub PRs). Fully local and stateless.",
  "version": "0.1.0"
}
```

If `claude plugin validate` rejects that `source` shape, consult the marketplace schema and use the
accepted git-source syntax (e.g. a `"source": "rayapatiranjithai/standup"` string), then re-validate.

- [ ] **Step 3: Validate the updated marketplace**

Run: `cd /Users/ranjithrayapati/plugins/prompt-analyzer && claude plugin validate .`
Expected: `✔ Validation passed`, now listing two plugins.

- [ ] **Step 4: Commit the marketplace change (in the prompt-analyzer repo)**

```bash
cd /Users/ranjithrayapati/plugins/prompt-analyzer
git add .claude-plugin/marketplace.json
git commit -m "Add standup to rayapatiranjithai-plugins marketplace"
git push origin main
```

- [ ] **Step 5: Create the GitHub release for standup**

```bash
cd /Users/ranjithrayapati/plugins/standup
gh release create v0.1.0 --title "v0.1.0 — standup" --notes "Initial release: /standup builds a redacted Yesterday/Today/Blockers update from live git (+ optional gh). Stateless, local, no telemetry."
```

---

## Self-Review

**Spec coverage:**
- On-demand/stateless, `/standup` command, `--since`/`--all` → Tasks 4, 3 (§1–2).
- Window "last working day" + author "only mine" → reference.md §1–2 (Task 3), applied in Task 4.
- Three sections with trust labels + never-fabricate → Task 4 (Rules + Output).
- gh-optional enrichment + graceful degrade → reference.md §4 (Task 3), Task 4 rules, scenario 6.
- Redaction/privacy_mode (2 values) → Task 2 (config), reference.md §6 (Task 3), Task 4 step 6, scenario 2.
- Error handling table → Task 4 GATE + Rules, scenarios 3/6/7.
- Testing strategy (7 scenarios) → Task 5.
- Packaging (own repo + git-source marketplace entry) → Task 6.
- Components (plugin.json, SKILL.md, reference.md, config.json, README/LICENSE/CHANGELOG) → Tasks 1–4.

**Placeholder scan:** No "TBD/TODO-as-work" left. The one deferred detail — exact marketplace git-source syntax — is bounded with a concrete primary form AND a named fallback in Task 6 step 2, because it depends on the installed marketplace schema; this is a genuine environment lookup, not a hidden decision.

**Type/name consistency:** `privacy_mode` values (`redact_pii`/`raw`), marker set (`TODO/FIXME/BLOCKED/HACK/WIP`), `--since`/`--all` flags, and reference.md section numbers (§1–6) are used identically across Tasks 2, 3, 4, and 5.
