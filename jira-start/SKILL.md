---
name: jira-start
description: Start work on a JIRA ticket — fetch it via the JIRA MCP, create/switch to a branch named <ticket>-<feature> off the freshly-fetched default branch, and self-assign + move to in-progress when unassigned. Use when the user says "jira:<PROJ-123>", "start jira task: <PROJ-123>", "start <PROJ-123>", "pick up <PROJ-123>", "work on <PROJ-123>", or otherwise gives a JIRA ticket to begin.
user-invocable: false
allowed-tools: Bash(git fetch *) Bash(git checkout *) Bash(git switch *) Bash(git branch *) Bash(git status *) Bash(git rev-parse *) Bash(git remote *) Bash(git symbolic-ref *)
---

# Start a JIRA Ticket

You are a dev about to pick up a JIRA ticket. Fetch it, branch off the latest base, and get
the ticket into the right state — but never touch the user's uncommitted work or someone
else's assignment.

The JIRA MCP tool names vary by server, so **discover them at runtime** (search available
tools for the JIRA "get issue", "current user", "assign", "list transitions", and
"transition" operations) rather than assuming fixed names. If no JIRA MCP tools are
available at all, stop and tell the user this skill needs a JIRA MCP server configured.

## Step 1 — Get the ticket id

Parse the ticket id from the invocation — `jira:<PROJ-123>`, `start jira task: <PROJ-123>`,
or a bare `<PROJ-123>` anywhere in the message. It looks like `<PROJECT-KEY>-<number>`
(e.g. `PROJ-1234`). Uppercase the project key. If no ticket id is present, ask the user for
one — it's required input, don't guess.

## Step 2 — Fetch the ticket

Fetch the ticket via the JIRA MCP "get issue" tool. Read its **summary/title**,
**description**, **assignee**, and **status** — you'll need all four below. If a tool
requires an explicit base URL, use `$JIRA_BASE_URL` (exported from `~/.claude/settings.json`;
read it with `echo $JIRA_BASE_URL`). If the fetch fails (bad id, no access), stop and report
it — don't create a branch for a ticket you couldn't read.

## Step 3 — Guard: dirty working tree

Run `git status --short`. **If there are any uncommitted or untracked changes, stop and
warn** — tell the user to commit or stash first. Do not stash, switch, or discard anything on
their behalf. Only continue when the tree is clean.

## Step 4 — Fetch and resolve the base branch

Always `git fetch origin` first so the base is current. Resolve the default branch:

```
git symbolic-ref refs/remotes/origin/HEAD --short   # e.g. origin/main
```

Strip the `origin/` prefix to get `<base>`. If that command fails, fall back to `main`, then
`master` (whichever exists on `origin`).

## Step 5 — Compute the branch name

`<ticket-id>-<feature>`, where `<feature>` is a short, understandable kebab-case slug
(~3–6 words) derived from the ticket title/description. Keep the ticket id exactly as-is —
don't lowercase or strip it.

Example: ticket `PROJ-1234` titled "Rotate refresh tokens before expiry" →
`PROJ-1234-refresh-token-rotation`.

## Step 6 — Work out the plan (read-only, no changes yet)

Gather everything needed to decide what *would* happen, without changing anything:

- **Branch** — does a local branch for this ticket already exist (`git branch --list
  '<ticket-id>*'`)? Note whether the plan is *create new* or *switch to existing*.
- **Assignment** — get the current user via the JIRA MCP ("myself" / current-user tool) and
  compare to the ticket's assignee. Decide: *assign to me* (unassigned), *no-op* (already
  mine), or *leave with `<other person>`* (someone else's — never reassign).
- **Transition** — list the ticket's available transitions and find the one whose name
  matches `in progress` (case-insensitive — handles "In Progress", "Start Progress", etc.).
  Decide: *move to in-progress*, *skip* (already in-progress), or *none available*.

## Step 7 — Confirm before acting

**Nothing above this point changed any state; nothing below runs until the user confirms.**
Present the plan and ask with the `AskUserQuestion` tool (selectable, one click). Spell out
exactly what will happen, e.g.:

> About to: create branch `PROJ-1234-refresh-token-rotation` off `origin/main`; assign
> PROJ-1234 to you; move it to In Progress.

Options:
- **If no branch exists yet:** *Proceed* / *Cancel*.
- **If a branch already exists:** *Switch to it* / *Recreate from base* (delete + branch
  fresh, discards its commits) / *Cancel*.

On **Cancel**, stop — make no branch, assignment, or transition changes. Only act on an
explicit confirmation.

## Step 8 — Execute the confirmed plan

Carry out exactly what was confirmed:

- **Branch:** create off base (`git switch -c <branch-name> origin/<base>`), switch to the
  existing one (`git switch <existing-branch>`), or recreate (`git switch -C <branch-name>
  origin/<base>`) per the choice.
- **Assignment:** assign to the current user only if the plan said so; otherwise leave it.
- **Transition:** apply the matched in-progress transition if there is one; if none is
  available, warn and continue — don't fail the whole flow over the transition.

## Step 9 — Report

Summarize what happened, concisely:
- Branch: created or switched, and the base it came from.
- Assignment: assigned to you / already yours / left with `<other person>`.
- Transition: moved to in-progress / already in-progress / no matching transition.
- Ticket URL: `$JIRA_BASE_URL/browse/<ticket-id>`.
