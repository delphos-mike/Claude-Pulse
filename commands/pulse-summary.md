# Pulse Summary — Combined Activity Summary

You are generating a "Pulse Summary" — a unified, cross-referenced summary combining activity from all configured surfaces.

## Arguments: $ARGUMENTS

## Configuration

Read the pulse config file. Check these locations in order:
1. `${CLAUDE_PLUGIN_ROOT}/config/pulse-config.json`
2. `~/.config/pulse/pulse-config.json`

If no config exists, tell the user to run `/pulse-setup` first and stop.

Extract from config:
- `surfaces` — which surfaces are enabled (github, slack, linear, fellow, notion)
- `surfaces.slack.post_summary_channel` — where to post the summary (optional)
- `surfaces.notion.report_database` — Notion database for full reports (optional)
- `team[]` — team member roster
- `output_dir` — where to write results
- `org_name` — for report headers

## Behavior

### Default (no arguments): Full combined pulse
Run all **enabled** surfaces for yesterday (or since last Friday if today is Monday) and synthesize into a single narrative. Cross-reference across sources:
- A Notion RFC mentioned in Slack gets linked together
- A PR discussed in Slack gets enriched with its GitHub status
- Linear ticket IDs in PR titles get correlated with Linear issue status
- Meeting action items get correlated with Slack follow-ups
- Fellow decisions get connected to resulting PRs or docs

After writing the summary file:
- If `surfaces.notion.report_database` is configured, **create a Notion page** with the full report
- If `surfaces.slack.post_summary_channel` is configured, **post a condensed version to Slack** (with user confirmation)

### With arguments: Targeted pulse
- `brief` — Shorter summary, highlights only (top 5 items across all sources)
- `no-post` — Skip posting to Slack and Notion (just write the local file)
- `person:Name` — Cross-source view of one person's activity
- `topic:X` — Cross-source view of a topic
- `since:YYYY-MM-DD` / `until:YYYY-MM-DD` — Override time window (applied to all sub-pulses)
- Source-specific filters with prefix: `slack:#channel git:PRs linear:done fellow:action-items notion:RFCs`
- Multiple can be combined: `person:Jane since:2026-02-01`

## Execution Strategy

### Phase 1: Compute time window
Default = yesterday to now. If today is Monday, use last Friday instead. Override with `since:` / `until:`. This window is propagated to ALL sub-pulses.

Compute the dates:
```python
python3 -c "
from datetime import datetime, timedelta
today = datetime.now().replace(hour=0, minute=0, second=0, microsecond=0)
if today.weekday() == 0:  # Monday
    start = today - timedelta(days=3)  # Friday
else:
    start = today - timedelta(days=1)  # Yesterday
print(f'start={start.strftime(\"%Y-%m-%d\")}')
print(f'end={today.strftime(\"%Y-%m-%d\")}')
print(f'start_ts={int(start.timestamp())}')
"
```

### Phase 2: Parallel data collection
Launch one subagent per **enabled** surface using the Task tool (subagent_type="general-purpose"). Pass the computed `since:` and `until:` dates to each. Each runs its respective pulse command logic.

**IMPORTANT — Do NOT use `run_in_background: true`**. MCP tools (Slack, Notion, Fellow, Linear) are not available in background subagents. Instead, launch all Task calls in a single message — they will run concurrently in the foreground while retaining MCP tool access.

Only launch subagents for surfaces where `surfaces.{name}.enabled` is true in config.

**Each subagent prompt should include:**
1. The full config contents (so it knows repos, channels, teams, etc.)
2. The time window
3. Instructions from the respective pulse command

### Phase 3: Cross-reference and synthesize
Once all subagents return, synthesize their outputs:

1. **Identify shared entities**: PR numbers, Linear ticket IDs, Notion URLs, person names, project names that appear in multiple sources.
2. **Merge related items**: If a Linear ticket has a PR and was discussed in Slack and a meeting, combine into one section.
3. **Build narrative by theme**: Group by project/initiative, not by source.
4. **Surface action items by person**: Build a per-person action item table from all sources.
5. **Identify blockers**: Team-wide blockers, stale PRs, unassigned issues needing triage.

### Phase 4: Write local output
Write to `{output_dir}/pulse-summary-{start_date}-to-{end_date}.md`

### Phase 5: Write to Notion (if configured)
If `surfaces.notion.report_database` is configured and `no-post` was not passed:

Create a Notion page in the configured report database with the full summary content.

Use `mcp__notion__notion-create-pages` (or equivalent Notion MCP tool) with:
```
parent: { data_source_id: "{report_database.id}" }
pages: [{
  properties: {
    "Pulse Doc": "Pulse Summary: {start_date} — {end_date}",
    "Headline": "{one-sentence summary of the day's most important theme}"
  },
  content: "{full markdown summary body}"
}]
```

### Phase 6: Post to Slack (if configured)
If `surfaces.slack.post_summary_channel` is configured and `no-post` was not passed:

Post a condensed summary to the configured channel.

**Important**: Before sending, show the user a draft of the Slack message and ask for confirmation. Only post after approval.

Use the Slack MCP send message tool with:
```
channel_id: "{post_summary_channel.id}"
message: <the formatted summary>
```

**Slack message format:**

```
*{org_name} Pulse Summary — {date range}*

*Action Items & Blockers*
• *Team-wide*: {blockers or decisions affecting everyone}
• *{Person}*: {action items from Linear, PRs, Slack, Fellow}

*Key Themes*
*{Theme 1}*: {2-sentence summary}
*{Theme 2}*: {2-sentence summary}

*Code & Linear Velocity*
• {N} PRs merged, {N} open ({M} awaiting review)
• {N} Linear issues completed, {N} in progress, {N} new & unassigned
• Top contributors: {person} ({n} items)

*Meetings*
• {meeting}: {key outcome}

*New Documents*
• <{notion_url}|{title}> — {author}

_Generated by Pulse_
```

## Output Format (full document)

```markdown
# {org_name} Pulse Summary: {start_date} - {end_date}

> **Sources**: {list of enabled sources with notes on gaps}

## Action Items & Blockers

### Team-Wide
- [ ] {blockers or decisions that affect everyone}

### By Person
| Person | Action Items |
|--------|-------------|
| **{Name}** | {Linear assignments, PR reviews, Slack commitments, Fellow action items} |

## {Initiative/Theme 1} (Hot|New|Update)

**Slack**: {what was discussed, by whom}
**GitHub**: {related PRs — status, key review comments}
**Linear**: {related issues — status transitions, assignees}
**Notion**: {related docs — RFCs, design docs}
**Fellow**: {related meeting decisions or discussions}

{2-5 sentence synthesis tying it all together}

## {Initiative/Theme 2} ...

## Code & Linear Velocity
- {N} PRs merged
- {N} PRs open ({M} awaiting review)
- {N} Linear issues completed across {M} teams
- {N} Linear issues created, {N} unassigned (needing triage)
- Top contributors: {person} ({n} PRs + issues)
- Key merges: {1-liner per notable merge}

## Meeting Highlights
- {Meeting series}: {key decision or outcome}

## Industry & External
- {Notable news items from Slack}

## New Documents
- {Notion docs created this window}

## Recently Edited Documents
- {Notion docs edited this window — with editor and summary}
```

## When using `brief` argument
Limit output to:
1. Top 3 themes (2 sentences each)
2. Action items by person (max 10 items total)
3. Code + Linear velocity stats (numbers only)

## When using `person:Name`
Look up person in team roster from config. Show everything that person did across all enabled sources.

## When using `topic:X`
Show everything related to that topic across all enabled sources.
