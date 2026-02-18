# Pulse — Multi-Surface Activity Summary for Claude Code

Pulse is a Claude Code plugin that orchestrates agents across your team's work surfaces to produce cross-referenced activity summaries. It scans GitHub, Slack, Linear, Fellow, and Notion — then synthesizes everything into a unified daily pulse.

## What it does

Each day, your team's work is scattered across PRs, Slack threads, issue trackers, meeting notes, and docs. Pulse deploys parallel agents to collect context from each surface, then cross-references it all:

- A PR linked to a Linear ticket gets correlated with its Slack discussion and the meeting where it was decided
- Action items surface from Fellow transcripts, Linear assignments, and Slack commitments
- Blockers and stale work get flagged across all surfaces

The output is a narrative summary grouped by theme (not by tool), with per-person action items and velocity stats.

## Supported Surfaces

| Surface | What Pulse scans | Required |
|---------|-----------------|----------|
| **GitHub** | PRs (open, merged, reviews), commits | `gh` CLI |
| **Slack** | Channel messages, threads, discussions | Slack integration in Claude |
| **Linear** | Issues by status, priority, team, cycle | Linear integration in Claude |
| **Fellow** | Meeting summaries, transcripts, action items | Fellow integration in Claude |
| **Notion** | Documents, RFCs, meeting notes | Notion integration in Claude |

Mix and match — enable only the surfaces your team uses.

## Prerequisites

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) installed
- [GitHub CLI](https://cli.github.com/) (`gh`) authenticated — for GitHub scanning
- Claude integrations enabled for any of: Slack, Linear, Fellow, Notion

## Installation

```bash
claude plugin install https://github.com/Delphos-Labs/pulse-plugin
```

## Setup

After installing, run the setup wizard:

```
/pulse-setup
```

The wizard will:
1. Detect which integrations are available
2. Discover your GitHub repos, Slack channels, Linear teams, Fellow meetings, and Notion workspace
3. Let you pick what to monitor
4. Build a team roster for cross-surface correlation
5. Validate all connections

Setup is re-runnable — add surfaces or change config anytime.

## Usage

### Individual surface commands

```
/git-pulse                    # GitHub activity since yesterday
/slack-pulse                  # Slack activity since yesterday
/linear-pulse                 # Linear issues since yesterday
/fellow-pulse                 # Meeting activity since yesterday
/notion-pulse                 # Notion docs since yesterday
```

### Combined summary

```
/pulse-summary                # Full cross-referenced summary
/pulse-summary brief          # Top 5 highlights only
/pulse-summary no-post        # Local file only, skip Slack/Notion
```

### Filtering

All commands support:

```
person:Jane                   # One person's activity across all surfaces
topic:authentication          # Everything related to a topic
since:2026-02-01              # Custom start date
until:2026-02-15              # Custom end date
```

Commands can be combined:

```
/pulse-summary person:Jane since:2026-02-01
/git-pulse merged topic:api
/linear-pulse team:Engineering done
```

## Architecture

Pulse uses an orchestrator pattern:

1. **`/pulse-summary`** computes the time window and launches parallel subagents
2. Each subagent runs one surface's collection logic using the appropriate MCP tools or CLI
3. Results flow back to the orchestrator, which cross-references entities (people, tickets, PRs) across surfaces
4. The synthesized summary is written locally, optionally to Notion and Slack

```
/pulse-summary
    ├── Task: git-pulse      (gh CLI)
    ├── Task: slack-pulse    (Slack MCP)
    ├── Task: linear-pulse   (Linear MCP)
    ├── Task: fellow-pulse   (Fellow MCP)
    └── Task: notion-pulse   (Notion MCP)
         │
         └──> Cross-reference & synthesize
              ├── Local markdown file
              ├── Notion database entry (optional)
              └── Slack summary post (optional, with confirmation)
```

## Configuration

Config lives at `config/pulse-config.json` inside the plugin directory. See `config/pulse-config.example.json` for the schema. The setup wizard creates this for you.

Key config sections:
- **`surfaces`**: Which tools to scan and their specific settings
- **`team`**: Roster mapping names across surfaces (GitHub username, Slack display, email)
- **`output_dir`**: Where local reports are written
- **`org_name`**: Your org name for report headers

## License

MIT
