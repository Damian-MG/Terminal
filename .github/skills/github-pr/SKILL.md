---
name: github-pr
description: Create a GitHub pull request following project conventions (branch naming, title format, gh CLI)
argument-hint: "[description]"
allowed-tools:
  - read
  - grep
  - glob
  - exec
permissions:
  allow:
    - Exec(git status)
    - Exec(git diff*)
    - Exec(git log*)
    - Exec(git branch*)
    - Exec(git checkout -b *)
    - Exec(git add *)
    - Exec(git commit *)
    - Exec(gh pr create*)
    - Exec(gh pr view*)
    - Exec(gh issue list*)
    - Exec(gh issue view*)
    - Exec(command -v gh)
  ask:
    - Exec(git push*)
    - Exec(gh issue create*)
---

# Create GitHub Pull Request

Create a pull request for the current changes following all project conventions.

## Step 1: Resolve gh CLI path

```bash
if command -v gh &> /dev/null; then
    GH_CMD="gh"
else
    GH_CMD="$HOME/apps/gh"
fi
```

Use `$GH_CMD` for all `gh` commands in the remaining steps.

## Step 2: Identify or create the tracking issue

Every PR **MUST** be linked to a tracking item — either a Jira ticket
(format `[A-Z][A-Z0-9]+-[0-9]+`, e.g. `DEVOPS-1234`) or a real GitHub
issue (`#NNN`). The shared organization CI workflow `Pre-Merge-Checks`
runs a regex on the PR title and rejects the PR if neither is present:

```
##[error]PR title does not contain a Jira ticket (e.g. DEVOPS-1234)
or a GitHub issue reference (e.g. #123).
```

It also rejects self-references: a `#NNN` that points to a Pull
Request (not an Issue) fails with `Reference '#NNN' points to a Pull
Request, not an issue.` So `(#<this-pr>)` does **not** work.

Resolution flow:

1. **Search for an existing tracking item**:
   - GitHub issues: `$GH_CMD issue list --state open --search "<keyword>"`
   - Jira: ask the user if there is a ticket associated with this work,
     or check the work description / branch name / commits for a
     `[A-Z]+-[0-9]+` reference.

2. **If you find candidates but are unsure** which one applies, **ask
   the user**. Do not guess — the wrong link clutters issue history.

3. **If nothing exists**, ask the user whether to create a GitHub
   issue. Default to creating one with a short title and a body that
   summarises the goal, scope, non-goals, and references. Suggested
   command:

   ```bash
   $GH_CMD issue create \
     --title "<short title aligned with the PR title>" \
     --body "<goal / scope / non-goals / references>"
   ```

4. **Record the reference** (e.g. `#7` or `AI-42`). It will be used in
   the branch name (Step 4) and in the PR title (Step 6).

The desired flow is **issue → branch → PR**, all three linked: the
issue describes the work, the branch implements it, the PR ships it.

## Step 3: Gather context

Run these commands in parallel to understand the current state:

- `git status` - check for uncommitted changes
- `git diff --staged` - review what will be committed
- `git diff` - review unstaged changes
- `git log --oneline -10` - recent commit history and style
- `git branch --show-current` - current branch name

## Step 4: Validate branch name

The current branch **MUST** match one of these prefixes:

| Prefix | Use case |
|--------|----------|
| `feature/` | New features or enhancements |
| `hotfix/` | Urgent bug fixes or patches |

Branch names use **kebab-case** after the prefix and ideally embed the
issue reference from Step 2 so the issue → branch → PR linkage is
visible at a glance.

**Valid:** `feature/add-retry-logic`, `hotfix/fix-memory-leak`,
`feature/issue-7-hexagonal-scaffolding`, `feature/AI-42-add-retry`
**Invalid:** `add-retry-logic`, `feat/add-retry`, `feature_add_retry`, `bugfix/something`

If the current branch does not follow this convention:
1. Ask the user which prefix applies (`feature/` or `hotfix/`)
2. Suggest a kebab-case branch name derived from the work description
   and the issue reference (e.g. `feature/issue-<NNN>-<slug>` or
   `feature/<JIRA-XXX>-<slug>`)
3. Create the branch: `git checkout -b <prefix>/<slug>`

## Step 5: Stage and commit (if needed)

If there are uncommitted changes:
1. Stage the relevant files
2. Write a concise commit message focused on "why" not "what"
3. Commit using the standard format:

```bash
git commit -m "$(cat <<'EOF'
Commit message here.

Generated with [Devin](https://cli.devin.ai/docs)

Co-Authored-By: Devin <158243242+devin-ai-integration[bot]@users.noreply.github.com>
EOF
)"
```

If pre-commit hooks modify files, stage and retry.

## Step 6: Determine PR title prefix and issue reference

**Ask the user** which prefix to use if not obvious from context:

| Prefix | When to use |
|--------|-------------|
| `Major:` | Breaking changes or major new features |
| `Minor:` | New features, backwards compatible |
| `Patch:` | Bug fixes, small improvements |

Format: `<Major|Minor|Patch>: <descriptive title> (<issue-ref>)`

The `<issue-ref>` comes from Step 2 and **must** be either `#NNN`
pointing to a real GitHub Issue (not a PR) or a Jira ticket id like
`AI-42` / `DEVOPS-1234`.

**Valid examples:**
```
Major: Redesign authentication system (#42)
Minor: Add retry logic and circuit breaker (AI-128)
Patch: Fix typo in error message (#15)
```

**Invalid examples:**
```
Feature/retry logic                       (wrong prefix)
fix: some bug                             (wrong prefix)
Add new feature                           (no prefix, no ref)
Major: Add retries                        (no issue ref → CI fails)
Major: Add retries (#5)                   (only OK if #5 is an issue,
                                           NOT if #5 is a PR)
```

## Step 7: Create the PR

Review ALL commits on the branch (not just the latest):

```bash
git log main...HEAD --oneline
git diff main...HEAD --stat
```

Then push and create the PR:

```bash
git push origin $(git branch --show-current)

$GH_CMD pr create --title "<Prefix>: <Title> (<issue-ref>)" --body "$(cat <<'EOF'
## Summary
<bullet points summarizing all changes>

## Changes
<list of files/components modified>

Refs <issue-ref>

#### Test plan
- [ ] <verification steps>

Generated with [Devin](https://cli.devin.ai/docs)
EOF
)"
```

The `Refs <issue-ref>` line in the body keeps the cross-link visible
in the issue timeline even if the PR title is later edited.

## Step 8: Confirm

After creating the PR, show the user:
- PR URL
- PR title
- Target branch
- Files changed count

## Constraints

- **Always link to a tracking issue** (Jira ticket or real GitHub
  Issue, never a PR self-reference). Search first, ask the user when
  in doubt, create a new GitHub Issue only when nothing applies.
- **Never push without asking** the user first
- **Always ask** which title prefix to use if the impact level is unclear
- **Always validate** branch naming before creating the PR
- **Never force-push** unless explicitly asked