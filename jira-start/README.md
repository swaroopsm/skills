# jira-start

Start work on a JIRA ticket: fetch it, branch off the latest base, and self-assign + move to
in-progress — all behind one confirmation. Triggered by "jira:<PROJ-123>" / "start jira
task: <PROJ-123>".

## Flow

```mermaid
flowchart TD
    start([jira:PROJ-123]) --> id[Parse ticket id]
    id --> fetch[Fetch ticket via JIRA MCP]
    fetch --> mcp{JIRA MCP<br/>available?}
    mcp -- no --> stopmcp([Stop: needs JIRA MCP])
    mcp -- yes --> dirty{Working tree<br/>dirty?}
    dirty -- yes --> stopdirty([Stop & warn: commit/stash first])
    dirty -- no --> base[git fetch origin + resolve default branch]

    base --> name[Compute branch name: ticket-feature-slug]
    name --> plan[Work out plan — read-only]
    plan --> planb[Branch: new vs existing]
    plan --> plana[Assignment: me / mine / someone else]
    plan --> plant[Transition: match 'in progress']

    planb --> confirm
    plana --> confirm
    plant --> confirm{{Confirm plan}}
    confirm -- Cancel --> stop([Stop — no changes])
    confirm -- Proceed / Switch / Recreate --> exec[Execute confirmed plan]

    exec --> b[Create / switch / recreate branch]
    exec --> a[Assign to me if unassigned]
    exec --> t[Apply in-progress transition]
    b --> report([Report: branch, assignment, transition, URL])
    a --> report
    t --> report
```
