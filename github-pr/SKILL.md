---
name: github-pr
description: Open or update a GitHub pull request with an issue-number-prefixed title and a conventional-commits body, mirroring the git-commit style. Confirms before pushing an unpushed branch, defaults to a draft PR, and presents 3 title candidates for the user to pick from. When a PR already exists it refreshes only its managed section, preserving bot-generated and human-edited content. Use when the user says "pr", "open a pr", "create a pull request", "raise a pr", "update the pr", "update pr description", "refresh the pr", or asks to open, create, or update a PR for the current branch.
user-invocable: false
allowed-tools: Bash(gh pr *) Bash(gh repo *) Bash(git branch *) Bash(git status *) Bash(git log *) Bash(git diff *) Bash(git rev-parse *) Bash(git push *)
---

# GitHub PR

## Context
- Current branch: !`git branch --show-current`
- Upstream tracking: !`git rev-parse --abbrev-ref --symbolic-full-name @{u} 2>/dev/null || echo "NONE"`
- Default branch: !`gh repo view --json defaultBranchRef --jq .defaultBranchRef.name 2>/dev/null || echo "main"`
- Unpushed commits: !`git log @{u}.. --oneline 2>/dev/null || git log --oneline -20`
- Recent commits on this branch: !`git log --oneline -20`
- PR template: !`cat .github/PULL_REQUEST_TEMPLATE.md docs/PULL_REQUEST_TEMPLATE.md PULL_REQUEST_TEMPLATE.md 2>/dev/null; ls .github/PULL_REQUEST_TEMPLATE/ 2>/dev/null || echo "NONE"`

Let `<base>` be the default branch above and `<head>` be the current branch.

## Step 1 — Make sure the branch is pushed

Determine whether `<head>` is fully pushed:
- If **Upstream tracking** is `NONE`, the branch has never been pushed.
- Otherwise, if **Unpushed commits** lists any commits, the branch is behind its upstream.

If the branch is not fully pushed, **always confirm before pushing** using the `AskUserQuestion` tool (selectable, so the user can reject with one click):
- **Push and continue** — push `<head>` to `origin` (`git push -u origin <head>` if there's no upstream yet, otherwise `git push`), then continue.
- **Cancel** — stop here without pushing and without opening a PR.

Do not push without an explicit confirmation. If the user picks Cancel, stop.

If the branch is already fully pushed, skip straight to Step 2.

## Step 2 — Determine the issue number

Reuse the git-commit logic. Try these in order, stop at the first confident answer:

1. **Parse the branch name.** Look for `PROJ-123`, `#123`, or a bare number set off by `/` or `-` (e.g. `feature/456-login-fix`, `bugfix/789`, `123-refactor-auth`). Keep the identifier exactly as it appears (don't strip a `PROJ-` prefix to just `123`).
2. **Scan the branch's commits** (see Context) for an existing `[<issue>]` prefix. If commits share one identifier, use it — the branch's own history wins over a fresh guess.
3. **If step 1 and step 2 disagree, or neither is confident**, ask with `AskUserQuestion` (selectable — don't force free text). Offer each conflicting candidate as its own option, plus a **Skip** option (open the PR with no issue-number prefix). The tool's built-in "Other" covers manual entry, so don't add a separate "enter a key" choice. Never guess an issue number.

If the user skips (or no key is found), omit the `[<issue-number>]` prefix everywhere below.

## Step 3 — Draft 3 title candidates

Base the title on the full set of commits going into the PR (the branch's commits vs `<base>`), not just the latest one. Generate **3 distinct candidates** sharing the same issue number, varied meaningfully:
- Different level of detail (terse vs. more descriptive)
- Different emphasis (what changed vs. why/impact)
- Different but equally valid phrasing

Format (plain by default; `BREAKING`/`ATTENTION` only if warranted — see Step 4):
```
[<issue-number>]: <title>
[<issue-number>] BREAKING: <title>
[<issue-number>] ATTENTION: <title>
```

## Step 4 — Decide on breaking/attention markers

Use **`BREAKING`** when the change removes/renames a public API, endpoint, signature, or config key; changes a relied-on default; or requires a migration / is not backward-compatible.

Use **`ATTENTION`** when it isn't breaking but a human must notice — a manual post-deploy step, a temporary workaround, a silent behavior change, or a dependency bump with caveats.

Apply the same marker across all 3 candidates. Most PRs get no marker; never stack both.

## Step 5 — Present the options and wait for the user

Show the 3 candidates as a numbered list (titles only), plus a 4th free-text option:

```
1. [PROJ-123]: add refresh-token rotation to auth client
2. [PROJ-123]: rotate refresh tokens before expiry in auth client
3. [PROJ-123]: prevent session drops from expired refresh tokens
4. Something else — reply with your own title, or reference the options above
```

**Stop and wait for the reply before opening the PR.** Resolve it:
- A bare number → use that candidate as-is.
- A number plus edits ("use 1 but shorter") → start there and apply the change.
- Multiple options ("combine 1 and 2") → merge into one coherent title.
- **Free text, no reference to an option** → use verbatim; if Step 2 produced an issue number, always prepend the `[<issue-number>]` prefix (even if the text looks like it already has one). If Step 2 was skipped, use as-is.
- Only skip the prefix if the user explicitly opts out ("no issue number").
- Ambiguous → ask a quick follow-up rather than guessing.

## Step 6 — Compose the body

The audience is a **code reviewer**, so write for review: point them at what changed and why, and flag anything that needs a closer look. Keep it concise — no filler.

**Wrap the entire generated body in managed-section markers** so it can be safely refreshed later without touching anything else:

```
<!-- github-pr:begin -->
<generated body>
<!-- github-pr:end -->
```

These markers matter for updates (Step 7) — everything outside them (bot-generated preview links, human edits) is off-limits.

**If the repo has a PR template** (see Context — `.github/PULL_REQUEST_TEMPLATE.md`, `docs/`, root, or a `.github/PULL_REQUEST_TEMPLATE/` directory): use it verbatim as the structure and fill in each section from the diff. Don't drop, reorder, or rename its headings; leave checklist items and HTML comments intact (tick boxes only when the diff clearly satisfies them). If there are multiple templates in the directory, pick the best fit and tell the user which.

**If there's no template**, use this default:

```
Type: <type>(<optional-scope>)

## Summary
<1-2 sentences: what this changes and why>

## Changes
- <notable change a reviewer should know>

## Review notes
- <anything needing a closer look: tricky logic, trade-offs, follow-ups>

## Test plan
- <how it was / should be verified>
```

- `type` from `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`, `perf`, `build`, `ci`; add `scope` only if clearly localized; append `!` if the title is `BREAKING`.
- If `BREAKING`, spell out what breaks and what consumers must do.
- Ground everything in the actual diff — don't invent changes or test steps. Drop the "Review notes" section if there's genuinely nothing to flag.
- Emojis: a light touch is welcome (e.g. a section header or to flag a ⚠️ caveat), but don't overuse them — most lines need none.

## Step 7 — Create or update the PR

First check whether a PR already exists for `<head>`: `gh pr view --json url,body,isDraft 2>/dev/null` (non-zero exit = no PR).

### No PR yet → create a draft

```
gh pr create --draft --base <base> --head <head> --title "<finalized title>" --body "<body>"
```

### PR already exists → update it, preserving everything outside our markers

The existing PR body may contain **bot-generated content (preview links, CI summaries, coverage) and human edits — those are important and must never be overwritten.** Only touch this skill's own managed section.

1. Fetch the current body (`gh pr view --json body --jq .body`).
2. Locate the `<!-- github-pr:begin -->` … `<!-- github-pr:end -->` block:
   - **If present:** replace *only* the text between the markers with the freshly generated body (regenerated from the current commit set — drop stale points, add new ones). Keep every character before `begin` and after `end` exactly as-is.
   - **If absent** (older PR, or created by hand): do **not** rewrite the existing body. Append a new marked section at the end, leaving all prior content untouched.
3. Write it back with `gh pr edit --body "<full reconstructed body>"`. Pass the whole body (preserved parts + refreshed section) — `gh` replaces the description wholesale, so anything you drop is lost. When in doubt, preserve.
4. Update the title too if the finalized title changed (`gh pr edit --title`).

Never blindly `--body` the generated text over an existing PR — always reconstruct from the fetched body so bot/human content survives.

### Always

- **Never mark a PR ready for review.** Always create as draft, and never run `gh pr ready` or otherwise flip an existing PR from draft to ready — even if the user asks. Leave that action to the user.
- Print the PR URL when done.
