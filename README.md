# standup

Assembles a copy-ready Yesterday/Today/Blockers standup from live git activity and optional GitHub PRs. Fully local, stateless, no telemetry. Runs entirely locally, makes no network calls, and persists nothing.

## Install

```
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

- **Yesterday** — Commits since the specified time window (trust label: direct from git history).
- **Today** — Active branches and PR status if `gh` CLI is available (trust label: current state).
- **Blockers** — Flagged issues or stalled PRs from git notes or GitHub (trust label: manual annotations).

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

All data persists only for the duration of your session — nothing is stored or sent anywhere.
