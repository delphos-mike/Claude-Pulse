# Slack Pulse — Slack Activity Summary

You are generating a "Slack Pulse" — a structured summary of recent Slack activity for a team.

## Arguments: $ARGUMENTS

## Configuration

Read the pulse config file. Check these locations in order:
1. `${CLAUDE_PLUGIN_ROOT}/config/pulse-config.json`
2. `~/.config/pulse/pulse-config.json`

If no config exists, tell the user to run `/pulse-setup` first and stop.

Extract from config:
- `surfaces.slack.channels[]` — channels to scan (each has `name`, `id`, `purpose`)
- `team[]` — team member roster
- `output_dir` — where to write results
- `org_name` — for report headers

If `surfaces.slack.enabled` is false, tell the user Slack is not configured and stop.

## Behavior

### Default (no arguments): Full slack pulse
Scan ALL configured channels for yesterday (or since last Friday if today is Monday). Produce a structured summary grouped by theme, not by channel. Highlight:
- Hot discussions and debates (who said what, key positions taken)
- New initiatives, RFCs, or proposals
- Infrastructure and deployment changes
- Industry news and competitive intel
- Company announcements
- Active sprint movement (summarize, don't enumerate every status change)
- Kudos and team recognition
- Product updates and decisions

### With arguments: Focused pulse
- `#channel-name` — Focus only on that specific channel
- `infrastructure` or `infra` — Focus on infrastructure topics
- `PRs` or `reviews` — Focus on PR and code review discussions
- `tooling` — Focus on developer tooling discussions
- `news` — Focus on industry news and competitive intel
- `person:Name` — Focus on a specific person's activity (mapped via team roster)
- `topic:X` — Search for a specific topic across all channels
- `since:YYYY-MM-DD` — Override time window start
- `until:YYYY-MM-DD` — Override time window end
- Multiple can be combined: `infra topic:terraform since:2026-02-01`

## Execution Strategy

1. **Compute time window**: Default = yesterday midnight local to now (if today is Monday, use last Friday instead). Override with `since:` / `until:` args.
2. **Convert dates to Unix timestamps** using `python3 -c "from datetime import datetime; print(int(datetime(YYYY,M,D).timestamp()))"`.
3. **Read channels in parallel** using `slack_read_channel` with `oldest` timestamp, `response_format=concise`. Use the channel IDs from config.
4. **Follow up on threads**: For messages with significant thread indicators, use `slack_read_thread` to get full context.
5. **Synthesize by theme**, not by channel. Name the people involved. Quote key statements when they add color.
6. **Write output** to `{output_dir}/slack-pulse-{start_date}-to-{end_date}.md`.

## Output Format

```markdown
# Slack Pulse: {start_date} - {end_date}

## {Theme 1} (Hot|New|Update|FYI)

{2-5 sentences summarizing the discussion. Name people. Quote key statements.}

## {Theme 2} ...

## Team Recognition
- {Kudos and shout-outs}

## Industry & News
- {Notable items}
```

When using a focused argument, skip sections that don't match the focus and go deeper on those that do. For person-focused searches, also use `from:username` in slack search.
