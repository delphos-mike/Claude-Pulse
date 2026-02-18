# Linear Pulse — Issue Tracker Activity Summary

You are generating a "Linear Pulse" — a structured summary of recent Linear issue tracker activity across your team.

## Arguments: $ARGUMENTS

## Configuration

Read the pulse config file. Check these locations in order:
1. `${CLAUDE_PLUGIN_ROOT}/config/pulse-config.json`
2. `~/.config/pulse/pulse-config.json`

If no config exists, tell the user to run `/pulse-setup` first and stop.

Extract from config:
- `surfaces.linear.teams[]` — teams to scan (each has `name`, `id`, `prefix`)
- `team[]` — team member roster
- `output_dir` — where to write results
- `org_name` — for report headers

If `surfaces.linear.enabled` is false, tell the user Linear is not configured and stop.

## Behavior

### Default (no arguments): Full linear pulse
Scan all configured teams for issues updated yesterday. Produce a structured summary organized into three categories:
- **New & Unassigned**: Issues created in the window that have no assignee yet (triage queue)
- **Currently Assigned**: Issues that are actively in progress or in review, grouped by assignee
- **Completed**: Issues moved to Done/completed in the window

Also include:
- High/urgent priority issues
- Active cycle progress per team
- Blocked or stale issues

### With arguments: Focused pulse
- `team:Name` — Focus on one team (use team name from config)
- `created` — Only newly created issues
- `done` or `completed` — Only issues moved to Done
- `in-progress` — Only issues currently In Progress
- `in-review` — Only issues in review
- `blocked` — Issues that are blocked
- `unassigned` — Only unassigned issues
- `priority:urgent` or `priority:high` — Filter by priority
- `project:Name` — Filter by project name
- `person:Name` — Issues assigned to or created by a specific person
- `topic:X` — Search issue titles/descriptions for a keyword
- `since:YYYY-MM-DD` — Override time window start
- `until:YYYY-MM-DD` — Override time window end
- Multiple can be combined: `team:Engineering done since:2026-02-13`

## Execution Strategy

1. **Compute time window**: Default = yesterday. Override with `since:` / `until:`.

2. **Query issues updated in window** (run in parallel for each configured team):
```
mcp__linear__list_issues(team="{team_name}", updatedAt="-P1D", limit=100, orderBy="updatedAt")
```

3. **Query newly created issues** (to identify unassigned, parallel per team):
```
mcp__linear__list_issues(team="{team_name}", createdAt="-P1D", limit=50, orderBy="createdAt")
```

4. **Query high/urgent priority issues**:
```
mcp__linear__list_issues(priority=1, state="started", limit=20)
mcp__linear__list_issues(priority=2, state="started", limit=20)
```

5. **Check active cycles** for each configured team:
```
mcp__linear__list_cycles(teamId="{team_id}", type="current")
```

6. **Categorize issues**: Separate into new & unassigned, currently assigned (in progress / in review), and completed.

7. **Write output** to `{output_dir}/linear-pulse-{start_date}-to-{end_date}.md`.

## Output Format

```markdown
# Linear Pulse: {start_date} - {end_date}

## Summary Stats
- {N} issues updated across {M} teams
- {N} completed, {N} in progress, {N} new, {N} unassigned

## Completed
### {Team Name}
- **{PREFIX}-{number}** {title} — {assignee} (priority: {p})

## Currently Assigned
### In Progress
| Issue | Title | Assignee | Team | Priority |
|-------|-------|----------|------|----------|
| {PREFIX}-{N} | {title} | {person} | {team} | {p} |

### In Review
| Issue | Title | Assignee | Team |
|-------|-------|----------|------|
| {PREFIX}-{N} | {title} | {person} | {team} |

## New & Unassigned (Triage Queue)
| Issue | Title | Team | Priority | Created |
|-------|-------|------|----------|---------|
| {PREFIX}-{N} | {title} | {team} | {p} | {date} |

## High Priority / Urgent
- **{PREFIX}-{N}** {title} — {assignee}, status: {state}

## Blocked / Stale
- **{PREFIX}-{N}** {title} — blocked reason: {reason}

## Cycle Progress
- **{Team}**: {current cycle name} — {N}% complete, {M} issues remaining
```

When using `person:Name`, look up their display name from the team roster in config and use as the `assignee` filter. When using `topic:X`, use it as the `query` parameter.
