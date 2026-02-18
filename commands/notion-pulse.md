# Notion Pulse — Workspace Activity Summary

You are generating a "Notion Pulse" — a structured summary of recent Notion workspace activity.

## Arguments: $ARGUMENTS

## Configuration

Read the pulse config file. Check these locations in order:
1. `${CLAUDE_PLUGIN_ROOT}/config/pulse-config.json`
2. `~/.config/pulse/pulse-config.json`

If no config exists, tell the user to run `/pulse-setup` first and stop.

Extract from config:
- `surfaces.notion.report_database` — database for writing reports (has `name`, `id`)
- `team[]` — team member roster
- `output_dir` — where to write results
- `org_name` — for report headers

If `surfaces.notion.enabled` is false, tell the user Notion is not configured and stop.

## Behavior

### Default (no arguments): Full notion pulse
Scan the Notion workspace for yesterday (or since last Friday if today is Monday). Produce a structured summary covering:
- Recently created or significantly updated documents (RFCs, ADRs, design docs, proposals)
- Meeting notes from the window
- New pages in key areas (engineering, product, security)
- Action items from meeting notes

### With arguments: Focused pulse
- `RFCs` or `proposals` — Focus on RFCs, ADRs, and design documents
- `meetings` — Just meeting notes from the window
- `person:Name` — Docs created/edited by a specific person
- `topic:X` — Semantic search for a specific topic across the workspace
- `since:YYYY-MM-DD` — Override time window start
- `until:YYYY-MM-DD` — Override time window end
- Multiple can be combined: `RFCs since:2026-02-01`

## Execution Strategy

1. **Compute time window**: Default = yesterday to now (if today is Monday, use last Friday instead). Override with `since:` / `until:`.

2. **Search for recently created AND edited docs** (run in parallel):

### Semantic searches for NEW documents (created in window)
```
notion-search(query="RFC proposal design document", filters={created_date_range: {start_date: "{start}", end_date: "{end}"}})
```
```
notion-search(query="ADR architecture decision", filters={created_date_range: {start_date: "{start}", end_date: "{end}"}})
```
```
notion-search(query="engineering update technical document", filters={created_date_range: {start_date: "{start}", end_date: "{end}"}})
```

### Semantic searches for EDITED documents (modified in window)
```
notion-search(query="RFC proposal design document", filters={last_edited_date_range: {start_date: "{start}", end_date: "{end}"}})
```
```
notion-search(query="ADR architecture decision", filters={last_edited_date_range: {start_date: "{start}", end_date: "{end}"}})
```

**Deduplication**: Docs that appear in both created and edited results should be listed once under "New Documents" (creation takes precedence). Docs that were only edited go under "Recently Updated".

### Meeting notes from the window
```
notion-query-meeting-notes(filter={
  operator: "and",
  filters: [{
    property: "created_time",
    filter: {
      operator: "date_is_within",
      value: { type: "exact", value: { type: "daterange", start_date: "{start}", end_date: "{end}" } }
    }
  }]
})
```

3. **For interesting results**, fetch full page content with `notion-fetch` to get summaries and key points.

4. **Synthesize by category**: group docs by type (RFC, ADR, meeting note, general), highlight key content and decisions.

## Output Format

Write to `{output_dir}/notion-pulse-{start_date}-to-{end_date}.md`:

```markdown
# Notion Pulse: {start_date} - {end_date}

## New Documents & RFCs

### RFCs / Proposals
- **{Title}** by {author} — {1-2 sentence summary}
  {Notion URL}

### Technical Documents
- ...

## Meeting Notes

### {Meeting Title} ({date})
- Attendees: {list}
- Key decisions: {bullets}
- Action items: {bullets}

### ...

## Recently Edited (not new, but modified in window)
- **{Title}** — last edited by {person}, {summary of what changed if discernible}
  {Notion URL}
```

When using `person:Name`, look up their name from the team roster in config for Notion user search, then filter by `created_by_user_ids`. When using `topic:X`, use it as the semantic search query with date filters.
