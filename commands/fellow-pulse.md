# Fellow Pulse — Meeting Activity Summary

You are generating a "Fellow Pulse" — a structured summary of recent meeting activity from Fellow.

## Arguments: $ARGUMENTS

## Configuration

Read the pulse config file. Check these locations in order:
1. `${CLAUDE_PLUGIN_ROOT}/config/pulse-config.json`
2. `~/.config/pulse/pulse-config.json`

If no config exists, tell the user to run `/pulse-setup` first and stop.

Extract from config:
- `surfaces.fellow.meeting_series[]` — meeting series to track (each has `name`, `cadence`)
- `team[]` — team member roster
- `output_dir` — where to write results
- `org_name` — for report headers

If `surfaces.fellow.enabled` is false, tell the user Fellow is not configured and stop.

## Behavior

### Default (no arguments): Full fellow pulse
Scan Fellow for yesterday (or since last Friday if today is Monday). Pull meetings from the configured meeting series. Produce a structured summary covering:
- Meeting summaries for each series that met in the window
- Key decisions made in meetings
- Action items assigned across all meetings
- Notable discussion topics from meeting transcripts

### With arguments: Focused pulse
- `action-items` or `todos` — Just outstanding action items
- `summaries` — Just meeting summaries, no transcript analysis
- `series:Name` — Focus on one meeting series
- `person:Name` — Meetings with a specific person
- `topic:X` — Search transcripts and summaries for a specific topic
- `since:YYYY-MM-DD` — Override time window start
- `until:YYYY-MM-DD` — Override time window end
- Multiple can be combined: `action-items since:2026-02-01`

## Execution Strategy

1. **Compute time window**: Default = yesterday to now (if today is Monday, use last Friday instead). Override with `since:` / `until:`.

2. **Search for each configured meeting series** (run in parallel):
```
mcp__claude_ai_Fellow_ai__search_meetings(
  from_date="{start_date}",
  to_date="{end_date}",
  note_summary="{series_name}"
)
```

3. **For each meeting found**, get the summary:
```
mcp__claude_ai_Fellow_ai__get_meeting_summary(meeting_id="{id}")
```

4. **Get action items** across meetings:
```
mcp__fellow__get_all_action_items()
```
Filter to items from the time window.

5. **For topic-focused searches**, search transcripts:
```
mcp__claude_ai_Fellow_ai__search_meetings(
  from_date="{start_date}",
  to_date="{end_date}",
  transcript="{topic}",
  note_summary="{topic}"
)
```

6. **For meetings >= 15 min with interesting summaries**, optionally pull transcript excerpts around key topics using `get_meeting_transcript` with start_time/end_time.

## Output Format

Write to `{output_dir}/fellow-pulse-{start_date}-to-{end_date}.md`:

```markdown
# Fellow Pulse: {start_date} - {end_date}

## Meeting Summary

### {Series Name}: {Meeting Title} ({date})
- Attendees: {list}
- Duration: {length}
- **Key Decisions:**
  - {decision 1}
  - {decision 2}
- **Discussion Highlights:**
  - {notable topic or quote}
- **Action Items:**
  - [ ] {action} — assigned to {person}

### ...

## Outstanding Action Items
- [ ] {action} — from {meeting title} ({date}), assigned to {person}

## Topics Discussed Across Meetings
- **{Topic}**: Came up in {meeting 1}, {meeting 2}. Key takeaway: {summary}
```

When using `person:Name`, look up their email from the team roster in config and use `participant_emails=["{email}"]` filter. When using `topic:X`, use it as the `transcript` and `note_summary` search parameter.
