# Git Pulse — GitHub Activity Summary

You are generating a "Git Pulse" — a structured summary of recent GitHub activity for a team.

## Arguments: $ARGUMENTS

## Configuration

Read the pulse config file. Check these locations in order:
1. `${CLAUDE_PLUGIN_ROOT}/config/pulse-config.json`
2. `~/.config/pulse/pulse-config.json`

If no config exists, tell the user to run `/pulse-setup` first and stop.

Extract from config:
- `surfaces.github.repos[]` — list of repos to scan (each has `owner`, `name`, `default_branch`)
- `team[]` — team member roster (each has `name`, `github`, `slack`, `email`)
- `output_dir` — where to write results
- `org_name` — for report headers

If `surfaces.github.enabled` is false, tell the user GitHub is not configured and stop.

## Behavior

### Default (no arguments): Full git pulse
Scan all configured repos for yesterday (or since last Friday if today is Monday). Produce a structured summary covering:
- Commits merged to the default branch in the window
- Open PRs: status, age, staleness, reviewers, draft status
- Recently merged PRs: who merged what, key changes
- Notable review discussions or comment threads on open PRs
- Dependabot/automated PRs worth noting

### With arguments: Focused pulse
- `repo:owner/name` — Only a specific repo
- `PRs` or `open` — Just open PRs across all repos
- `merged` — Just recently merged PRs
- `person:Name` — Activity from a specific person (mapped via team roster in config)
- `since:YYYY-MM-DD` — Override time window start
- `until:YYYY-MM-DD` — Override time window end
- `topic:X` — Filter PRs/commits by keyword in title/body
- Multiple can be combined: `merged since:2026-02-01`

## Execution Strategy

1. **Compute time window**: Default = yesterday to now (if today is Monday, use last Friday instead). Override with `since:` / `until:`.
2. **For each configured repo**, run these gh commands in parallel (use Bash tool):

### Open PRs
```bash
gh pr list --repo {owner}/{name} --state open --limit 50 \
  --json number,title,author,createdAt,updatedAt,reviewRequests,isDraft,labels,headRefName
```

### Recently merged PRs (in window)
```bash
gh pr list --repo {owner}/{name} --state merged --limit 30 \
  --json number,title,author,mergedAt,mergedBy
```

### Commits to default branch (in window)
```bash
gh api "repos/{owner}/{name}/commits?sha={default_branch}&since={start_iso}&per_page=100" \
  --jq '.[] | "\(.sha[0:7]) \(.commit.author.name): \(.commit.message | split("\n")[0])"'
```

3. **Filter merged PRs** to only those within the time window.
4. **For PRs with notable review activity**, check comment counts:
```bash
gh pr view {number} --repo {owner}/{name} --json comments,reviews,reviewDecision
```
5. **Synthesize by category**: merged PRs grouped by area (infra, features, tooling, security, deps), open PRs by status, review activity.

## Output Format

Write to `{output_dir}/git-pulse-{start_date}-to-{end_date}.md`:

```markdown
# Git Pulse: {start_date} - {end_date}

## Merged ({count})

### {Category 1}
- **PR #{number}** {title} — {author}, merged {date}

### {Category 2}
- ...

## Open PRs ({count})

### In Review
- **PR #{number}** {title} — {author}, {age} old, reviewers: {list}

### Active
- **PR #{number}** {title} — {author}, {age} old {draft?}

### Stale (no updates > 3 days)
- ...

## {Repo 2} Activity
- {commits and PRs summary}

## Notable Review Discussions
- **PR #{number}**: {summary of review thread if significant}
```

When using `person:Name` argument, look up the GitHub username from the team roster in config and filter with `--search "author:{username}"` for PRs and `--author={username}` for commits.
