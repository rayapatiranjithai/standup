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
- relative duration: `Nd`, `Nh`, `Nw` → normalize to an absolute timestamp (same shape as the
  default window above), not passed as a phrase to git. Parse the number `N` and the unit, then
  compute with BSD `date` first, GNU fallback: `d`→days (`-v-Nd` / GNU `-N days`), `h`→hours
  (`-v-NH` / GNU `-N hours`), `w`→weeks (`-v-Nw` / GNU `-N weeks`). Example for `3d`:
  ```bash
  N=3
  SINCE=$(date -v-${N}d '+%Y-%m-%d %H:%M:%S' 2>/dev/null || date -d "-${N} days" '+%Y-%m-%d %H:%M:%S')
  ```
  (for `12h`, use `-v-12H` / `-d "-12 hours"`; for `1w`, use `-v-1w` / `-d "-1 weeks"`).
- ISO date `YYYY-MM-DD` → set `SINCE="YYYY-MM-DD 00:00:00"`.
- anything else → print accepted formats and stop (see SKILL error table).

All three forms leave `$SINCE` as an absolute `YYYY-MM-DD HH:MM:SS` timestamp, so every downstream
use — the git-log `--since` calls in §3, the future-date guard below, and the `${SINCE%% *}` date
extraction in §4 — sees a consistent, date-prefixed value.

**Future-dated window guard:** after resolving `SINCE`, compare it to the current date/time
(`date '+%Y-%m-%d %H:%M:%S'`). If `SINCE` is later than now, the window is empty by definition —
skip **both** `git log` calls in §3 that take `--since="$SINCE"`: the Yesterday commit list
(`git log --since="$SINCE" ... --pretty=...`) *and* the Blockers diff-scan
(`git log --since="$SINCE" ... -p --no-color`). Go straight to the "no in-window commits" path
(SKILL.md Rules) for Yesterday, and treat §5's marker scan as having no diff to scan (report no
markers) for Blockers. Do not rely on `git log --since` alone to report zero results here: some git
builds fail to filter dates far enough in the future (observed: years beyond ~2100) and return the
full history instead of nothing for *both* calls.

## 2. Author resolution

Default: current user. `--all`: no author filter.

```bash
ME=$(git config user.email); [ -z "$ME" ] && ME=$(git config user.name)
AUTHOR_FILTER=(--author="$ME")   # default: current user only
# --all: AUTHOR_FILTER=()        # no author filter — every author included
```

## 3. Git recipes (Yesterday / Today / Blockers inputs)

```bash
# Yesterday: my in-window commits, subject + branch grouping
git log --since="$SINCE" "${AUTHOR_FILTER[@]}" --pretty='%h%x09%s%x09%cI' --all
# current branch (Today)
git rev-parse --abbrev-ref HEAD           # prints HEAD if detached
# uncommitted / staged work (Today)
git status --porcelain
# in-window diff for Blocker markers (same scope as Yesterday: author filter + all refs)
git log --since="$SINCE" "${AUTHOR_FILTER[@]}" -p --no-color --all
```

Group Yesterday commits by branch when derivable (`git branch --contains <hash>` is expensive; for
v0.1 group by day from `%cI` and list branch only for the current HEAD).

## 4. GitHub recipes (optional — skip silently if `gh` absent/unauthed)

Detect: `command -v gh >/dev/null && gh auth status >/dev/null 2>&1`.

```bash
# my merged PRs since the window (Yesterday)
gh pr list --state merged --author "@me" --search "merged:>=${SINCE%% *}" --json number,title,mergedAt
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
ones (TOKEN); longer match wins on overlap.

| Token | Regex (ECMAScript) | Notes |
|---|---|---|
| `[EMAIL]` | `[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}` | |
| `[KEY]` | `\b(?:sk-|pk-|ghp_|xox[baprs]-|AKIA|AIza)[A-Za-z0-9_\-]{10,}\b` | API-key prefixes |
| `[JWT]` | `\beyJ[A-Za-z0-9_\-]+\.[A-Za-z0-9_\-]+\.[A-Za-z0-9_\-]+\b` | JWTs |
| `[TOKEN]` | `\b[A-Fa-f0-9]{32,}\b` | long hex (hashes/tokens) |
| `[CARD]` | `\b(?:\d[ -]?){13,16}\b` | card-length digit runs |
| `[SSN]` | `\b\d{3}-\d{2}-\d{4}\b` | US SSN shape |
| `[IP]` | `\b(?:\d{1,3}\.){3}\d{1,3}\b` | IPv4 |
| `[CREDENTIAL]` | `(password\|passwd\|secret\|api[_-]?key)\s*[:=]\s*\S+` | key/value secrets |

Note: commit hashes are short (7–12 hex) and will NOT match `[TOKEN]` (32+); leave them intact.
