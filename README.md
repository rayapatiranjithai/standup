# standup

Assembles a copy-ready Yesterday/Today/Blockers standup from live git activity and optional GitHub PRs. Fully local, stateless, no telemetry. Runs entirely locally, makes no network calls, and persists nothing.

## Install

```
# The marketplace catalog is hosted in the prompt-analyzer repo; this installs the standup plugin from it.
/plugin marketplace add rayapatiranjithai/prompt-analyzer
/plugin install standup@rayapatiranjithai-plugins
```

## Usage

### `/standup` skill

Generates a standup update with Yesterday, Today, and Blockers sections from your git activity.

**Options:**

- `--since=<spec>` — Specify the time window for git activity (e.g., `--since="2 days ago"`). Default: since last working day.
- `--all` — Include commits from all authors. Default: current git user only.
- `--privacy=<mode>` — Privacy mode: `raw` (preserve all text) or `redact_pii` (default, redacts emails/keys/IPs/names).

**Output sections:**

- **Yesterday** — Ground truth: real commits since the window, plus merged PRs if `gh` is available. Never fabricated.
- **Today** _(inferred — confirm/edit)_ — Guessed from the current branch name and uncommitted/staged/WIP work. Not a source of truth; you're expected to review and edit it.
- **Blockers** _(scanned — confirm/edit)_ — Scanned from `TODO`/`FIXME`/`BLOCKED`/`HACK`/`WIP` markers in the in-window diff, plus your open PRs awaiting review if `gh` is available. Also expected to be reviewed and edited.

Note: GitHub PR enrichment requires the `gh` CLI to be installed and authenticated. Without it, standup uses git history only.

## Configuration

Configure privacy and behavior via `.standup/config.json`:

```json
{
  "privacy_mode": "redact_pii",
  "since_default": "last working day"
}
```

- `privacy_mode` — Controls how prompt text is handled:
  - `raw` — Preserve all text.
  - `redact_pii` — Mask emails, keys, IPs, and names (default).

The plugin persists nothing: it reads git (and optionally `gh`) fresh on every run, with no cache or session state.
