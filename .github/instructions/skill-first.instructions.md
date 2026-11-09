---
trigger: always_on
---
# Skill-First Rule

## Before Acting, Check for Skills

Before executing any task, check if a matching skill exists in the `<available_skills>` list. If a skill matches the task, **invoke it** instead of following generic instructions.

Examples of skill matching:
- User asks to create a PR/MR/pull request/merge request → invoke `github-pr`
- User asks to build/push Docker image → invoke `aws-docker-ecr`
- User asks for an investigation or research → invoke `investigation`

Project-level skills always take priority over generic system prompt workflows.

## Git Safety

When working with git:

1. **Never run `git stash`** without explicit user confirmation
2. **Never run `git pull`** unless the user asks for it
3. **Never create commits** (including `--allow-empty`) unless the user asks for it
4. **Never modify the working directory state** (uncommitted changes, untracked files) without asking
5. **Only do what was requested** — if asked to create a branch and PR, do only that
