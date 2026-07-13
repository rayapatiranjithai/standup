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
