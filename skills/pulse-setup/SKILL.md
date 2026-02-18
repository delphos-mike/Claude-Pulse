---
name: pulse-setup
description: Interactive setup wizard for configuring the Pulse plugin. Discovers your GitHub repos, Slack channels, Linear teams, Fellow meetings, and Notion workspace.
---

# Pulse Setup

You are running the Pulse plugin setup wizard. Your job is to walk the user through configuring Pulse for their organization.

## Config file location

The config will be saved to: `${CLAUDE_PLUGIN_ROOT}/config/pulse-config.json`

If a config already exists, load it and offer to update it (don't start from scratch).

## Setup Flow

### Step 1: Prerequisites check

Verify available tools:
- **gh CLI**: Run `gh auth status` to check GitHub authentication
- **Slack MCP**: Check if Slack MCP tools are available (try a test search)
- **Linear MCP**: Check if Linear MCP tools are available (try listing teams)
- **Fellow MCP**: Check if Fellow MCP tools are available (try searching meetings)
- **Notion MCP**: Check if Notion MCP tools are available (try a search)

Report which surfaces are available. If a surface isn't available, explain what the user needs to set up (e.g., "Slack integration requires connecting Slack in Claude's settings") and mark it as disabled. The user can re-run setup later after configuring the integration.

### Step 2: Organization info

Ask the user:
- What is your organization/team name? (used in report headers)
- Where should pulse reports be saved? (default: `./pulse-output`)

### Step 3: Configure each available surface

For each surface that passed the prerequisites check, run discovery and let the user select what to monitor.

#### GitHub
1. Run `gh repo list --limit 30 --json nameWithOwner,defaultBranchRef` to discover repos
2. Present the list and ask which repos to include
3. If the user wants repos not in the list, let them add manually (owner/name format)

#### Slack
1. Use `slack_search_channels` to search for channels the user might want
2. Ask the user which channels to monitor — suggest starting with 3-5 core channels
3. Ask if they want pulse summaries posted to a Slack channel, and if so, which one

#### Linear
1. Use `mcp__linear__list_teams` to discover available teams
2. Present the list and ask which teams to track
3. Record team IDs and prefixes

#### Fellow
1. Use `mcp__claude_ai_Fellow_ai__search_meetings` with a recent date range to discover meeting series
2. Present discovered meetings and ask which series to track
3. Let the user add series names manually if needed

#### Notion
1. Use `notion-search` to discover the workspace
2. Ask if the user wants full pulse reports written to a Notion database
3. If yes, help them identify or create the target database

### Step 4: Team roster

Build a team member roster by cross-referencing discovered data:
1. Pull member lists from available surfaces (GitHub org members, Linear team members, etc.)
2. Present the merged list and ask the user to confirm/edit
3. For each person, map their identities across surfaces: name, GitHub username, Slack display name, email

The roster is used to correlate activity across surfaces (e.g., "Jane's PR" + "Jane's Linear issue" + "Jane's Slack message").

If the user doesn't want to set up a full roster, that's fine — cross-referencing will be best-effort based on name matching.

### Step 5: Validate connections

For each enabled surface, run a quick test:
- **GitHub**: `gh pr list --repo {first_repo} --limit 1`
- **Slack**: Read 1 message from the first configured channel
- **Linear**: List 1 issue from the first configured team
- **Fellow**: Search for 1 recent meeting
- **Notion**: Run a search query

Report results: "GitHub: connected, Slack: connected, Linear: connected, Fellow: not available, Notion: connected"

### Step 6: Save config

Write the config to `${CLAUDE_PLUGIN_ROOT}/config/pulse-config.json` using the schema from `config/pulse-config.example.json`.

Print a summary:
```
Pulse configured for {org_name}:
  GitHub: {N} repos
  Slack: {N} channels (posting to #{channel})
  Linear: {N} teams
  Fellow: {N} meeting series
  Notion: {enabled/disabled}
  Team: {N} members

Run /pulse-summary to generate your first report!
```

## Re-running setup

If config already exists, present current config and ask what the user wants to change:
- Add/remove surfaces
- Add/remove repos/channels/teams/series
- Update team roster
- Change output settings
