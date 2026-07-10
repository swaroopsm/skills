# skills

A set of personal [Claude Code](https://claude.com/claude-code) skills.

## Skills

| Skill | What it does | Triggers |
| ----- | ------------ | -------- |
| [`git-commit`](git-commit/README.md) | Commit with an issue-number prefix and conventional-commits body; offers 3 title candidates. | `gc`, `commit` |
| [`github-pr`](github-pr/README.md) | Open/update a draft PR with an issue-prefixed title and reviewer-focused body; preserves bot/human edits. | `pr`, `open a pr`, `update the pr` |
| [`jira-start`](jira-start/README.md) | Fetch a JIRA ticket, branch off the latest base, self-assign and move to in-progress. | `jira:<PROJ-123>`, `start jira task: <PROJ-123>` |

Each skill's README has a mermaid flow diagram of how it works.

## Install

```bash
npx skills add swaroopsm/skills           # all, project-level
npx skills add swaroopsm/skills -s <name> # one skill
npx skills add swaroopsm/skills -g        # global
npx skills add swaroopsm/skills -l        # list, don't install
```

## Environment variables

Some skills read environment variables. Set them in the `env` block of your Claude Code
`settings.json` (`~/.claude/settings.json` for global) so they're exported to the shell:

```json
{
  "env": {
    "JIRA_BASE_URL": "https://your-org.atlassian.net"
  }
}
```

| Variable         | Used by      | Purpose                                                                 |
| ---------------- | ------------ | ----------------------------------------------------------------------- |
| `JIRA_BASE_URL`  | `jira-start` | Base URL of your JIRA instance — used for the ticket link and any MCP call that needs an explicit base URL. |

## License

MIT
