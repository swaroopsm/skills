# skills

A set of personal [Claude Code](https://claude.com/claude-code) skills.

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
