---
name: git-commit
description: Create a git commit with an issue-number prefix and a plain-language title, using conventional-commits type/scope in the body. Flags breaking or attention-needed changes in the title. Presents 3 message options for the user to pick from before committing. Use when the user says "gc" or "commit", "commit this", "make a commit", "commit my changes", or otherwise asks to commit the current changes.
user-invocable: false
allowed-tools: Bash(git branch *) Bash(git log *) Bash(git diff *) Bash(git status *) Bash(git add *) Bash(git commit *)
---

# Git Commit

## Context
- Current branch: !`git branch --show-current`
- Staged changes (stat): !`git diff --cached --stat`
- Staged changes (full): !`git diff --cached`
- Recent commits on this branch: !`git log --oneline -20`

## Step 1 — Determine the issue number

Try these in order, and stop as soon as one gives a confident answer:

1. **Parse the branch name.** Look for a pattern like `PROJ-123`, `#123`, or a bare number set off by `/` or `-` (e.g. `feature/456-login-fix`, `bugfix/789`, `123-refactor-auth`). Extract the identifier exactly as it appears (keep the prefix if there is one, e.g. `PROJ-123`, don't strip it to just `123`).
2. **If the branch name is ambiguous or has no identifier**, scan the recent commits above for an existing `[<issue>]` prefix used earlier on this same branch. If one or more commits share the same identifier, use that — consistency with the branch's own history wins over a fresh guess.
3. **If step 1 and step 2 disagree**, or if neither produces a confident match, ask the user with the `AskUserQuestion` tool (not free-text — keep it selectable to save keystrokes). Offer these options:
   - Any confident-but-conflicting candidates from steps 1 and 2 (e.g. `PROJ-123` from the branch, `PROJ-456` from recent commits) as one option each, so the user picks with a single click.
   - **Skip** — commit with no issue-number prefix.

   The user can always type their own key via the tool's built-in "Other" free-text option, so don't add a separate "enter a key" choice. Do not guess or invent an issue number — an incorrect one is worse than asking.

   If the user picks **Skip** (or provides no key), omit the `[<issue-number>]` prefix entirely in all steps below — titles become just `<title>` / `BREAKING: <title>` / `ATTENTION: <title>`.

## Step 2 — Draft 3 title candidates

Generate **3 distinct title candidates**, all using the same issue number from Step 1. Vary them meaningfully — not just synonym swaps of the same sentence. Reasonable axes of variation:
- Different level of detail (terse vs. more descriptive)
- Different emphasis (what changed vs. why/impact)
- Different but equally valid phrasing of the action

Each candidate follows the format below (plain title by default, `BREAKING`/`ATTENTION` marker only if warranted — see Step 3):
```
[<issue-number>]: <title>
[<issue-number>] BREAKING: <title>
[<issue-number>] ATTENTION: <title>
```

## Step 3 — Decide on breaking/attention markers

Use **`BREAKING`** when the change:
- Removes or renames a public API, endpoint, function signature, or config key
- Changes a default behavior that other code or consumers currently rely on
- Requires a database migration, schema change, or is otherwise not backward-compatible

Use **`ATTENTION`** when the change isn't breaking but still needs a human to notice before or after merge — e.g. a manual step required post-deploy, a temporary workaround, a silent behavior change, or a dependency bump with known caveats.

Apply the same marker decision consistently across all 3 candidates — don't mark one candidate as `BREAKING` and leave another plain for the same underlying change. Most commits get no marker at all; only use one when the diff genuinely warrants it, and never stack both.

## Step 4 — Present the options and wait for the user

Show the 3 candidates as a numbered list, titles only (body text comes after selection). Add a 4th option for free text. For example:

```
1. [PROJ-123]: add refresh-token rotation to auth client
2. [PROJ-123]: rotate refresh tokens before expiry in auth client
3. [PROJ-123]: prevent session drops from expired refresh tokens
4. Something else — reply with your own title, or reference the options above (e.g. "use 1 but reword it as ...", "combine 1 and 2")
```

**Stop and wait for the user's reply before running any commit.** Do not pick one automatically, even if one candidate seems clearly best.

When the reply comes back, resolve it like this:
- A bare number ("2", "option 2") → use that candidate as-is.
- A number plus edit instructions ("use 1 but reword it like ...", "option 3, shorter") → start from that candidate and apply the requested change.
- A reference to multiple options ("combination of 1 and 2", "mix 2 and 3") → merge the specified candidates into one coherent title, don't just concatenate them.
- **Free text with no reference to the numbered options** → use it verbatim as the title text. If Step 1 produced an issue number (i.e. the user didn't skip), **always prepend the `[<issue-number>]` prefix**, even if the free text already looks like it has a prefix, contains its own brackets, or seems complete on its own. Default to prefixing — don't try to detect whether one is "already there" and skip it on that basis. If Step 1 was skipped, use the free text as-is with no prefix.
- **Only skip the prefix if the user explicitly says not to** (e.g. "no issue number", "don't prefix this", "use this exact text as-is"). Absent an explicit opt-out like that, always assume positive intent and add it.
- If the reply is ambiguous (e.g. doesn't clearly map to an option or a title), ask a quick follow-up rather than guessing.

## Step 5 — Write the body using conventional-commits type

Below the finalized title, add a blank line, then a `Type:` line using the [Conventional Commits](https://www.conventionalcommits.org) taxonomy:

```
Type: <type>(<optional-scope>)
```

- Choose `type` from: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`, `perf`, `build`, `ci` — based on what the staged diff actually does.
- `scope` is optional; include it only if the change is clearly localized to one module/area.
- If the finalized title carries a `BREAKING` marker, append `!` to the type (e.g. `feat(auth)!`).

After the `Type:` line, add a short paragraph or bullet list explaining *why* the change was made — but only if it's not obvious from the diff alone. Skip the explanation for trivial changes; the `Type:` line should still always be present.

**If the title carries a `BREAKING` marker**, the body must also explain what breaks and what callers/consumers need to do about it — don't leave that only in the title.

## Step 6 — Compose and commit

Full example:
```
[PROJ-456] BREAKING: remove deprecated /v1/login endpoint

Type: feat(auth)!

/v1/login has been replaced by /v2/login since PROJ-200. Callers
still on v1 will get a 410 after this deploy; update client
integrations to use /v2/login before merging.
```

If there are no staged changes (`git diff --cached --stat` is empty), **don't commit and don't stage silently.** First check whether there are any unstaged/untracked changes (`git status --short`):
- **If there's nothing to commit at all**, tell the user and stop.
- **If there are unstaged or untracked changes**, ask for confirmation with the `AskUserQuestion` tool (selectable, so the user can reject in one click) before staging. List what would be staged — added, modified, and deleted files (`git add -A` semantics) — and offer:
  - **Stage all and continue** — run `git add -A`, then proceed with the commit.
  - **Cancel** — stop without staging or committing.

  Only stage after the user picks "Stage all and continue." If they cancel, stop.

Run the commit with:
```
git commit -m "[<issue-number>]: <title>" -m "Type: <type>(<scope>)" -m "<optional body text>"
```
Use separate `-m` flags (as above) rather than embedded `\n`, so the blank-line separation between title, type, and body is preserved correctly.

**Never push to the remote after committing.** This skill only creates the commit locally — do not run `git push` (or otherwise publish the commit), even if it seems like the next step. Leave pushing to the user.
