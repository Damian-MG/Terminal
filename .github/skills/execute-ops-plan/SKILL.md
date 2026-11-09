---
name: execute-ops-plan
description: Execute a phased operations plan (`task/PLAN_<NAME>.md` produced by `generate-ops-plan`) sequentially in the current session — no parallel sub-agents, no isolated git worktrees. Use for config/canon/doc/migration work, especially when phases touch a single remote mount or a shared config file where parallel worktrees would conflict or corrupt state.
argument-hint: "[@task/PLAN_<NAME>.md]"
allowed-tools:
  - read
  - grep
  - glob
  - exec
  - edit
  - write
---

# Execute Operations Plan (Sequential)

Execute an operations plan phase by phase, directly in the current session. Unlike `execute-plan`, there is no orchestrator/sub-agent split and no isolated git worktrees: ops phases are typically small and sequential, and often touch a single shared config file or a single remote mount that cannot be safely parallelized.

## When to use this instead of `execute-plan`

- The plan was produced by `generate-ops-plan` (Quality Gates are verification commands, not tests/build/lint).
- Any phase touches a remote mount (sshfs or similar) — opening multiple git worktrees against a single mount point is unsafe and can silently corrupt or lose writes.
- Phases are inherently sequential (edit canon config, then rewrite a file in a different repo, then migrate a directory, then propagate) rather than independently parallelizable units of code.

## Prerequisites

- Plan file exists with phases defined (default `task/PLAN.md` if none is specified).
- Every repo the plan touches is in a clean, known state.
- If any phase touches a remote-mounted target: `sync-check mounts status` **and** a real-content `ls` on the mount confirm it is live (mount status alone is not sufficient — see `docs/PROCEDIMIENTOS-CANON.md`). Re-verify immediately before that specific phase writes, not only at the start.
- You are already on the intended branch in every repo the plan touches. This skill never switches or creates branches on your behalf; if the plan requires a dedicated branch, create it before invoking this skill.

## Execution Protocol

### 0. Initial Setup

- Confirm every repo's working tree is clean (`git status --short`).
- If any phase touches a remote mount, verify it now (see Prerequisites).
- Read the plan file. Re-read the `## Objectives` and `## Locked Decisions` sections. If Objectives is missing, STOP and ask the user — do not proceed with execution.
- **Ignore the loop-state side-car** (`task/PLAN_<NAME>.state.md`) if present — it is the review loop's private memory, not execution input. The plan itself, including `## Locked Decisions`, is the complete contract.

For each phase in the plan file, in order:

### 1. Phase Preparation

Re-read the current phase's Functional Deliverable, Boundaries, Required Tests, and Quality Gates.

Verify the phase's deliverable falls within "In Scope", does not violate any "Hard Constraint", and honors every "Locked Decision". If it does not, STOP and report to the user before making any change.

Treat the phase's Boundaries as absolute: before taking any step, confirm it does not cross a named boundary (an unrelated repo, file, or open plan).

### 2. Execute the Phase

Perform the phase's operations directly in the current session — no sub-agent spawn, no worktree:

- Edit files, run commands (`git mv`, `sync-check apply`, config edits) as described by the phase's Functional Deliverable.
- One phase at a time; never start work belonging to a later phase.
- Any destructive or remote-write step (a `git mv` on a remote mount, a force operation, deleting files) follows the general Destructive Operations policy: state exactly what is about to run and why, and pause for explicit confirmation before running it — every destructive action gets its own confirmation, even if a prior phase or step already asked.

### 3. Verify (run the phase's Quality Gates literally)

Run every command in the phase's Quality Gates checklist and record pass/fail. Run the phase's Required Tests (Unit / Integration / Edge cases) as literal checks against the real repo/target state, not as code tests.

### 4. Handle Failures

Any failed gate: STOP. Do not proceed to the next phase.

Apply the phase's Failure Policy: retry only idempotent, read-only checks (max 3 attempts) to rule out a transient issue (e.g. re-checking a mount that may have just reconnected). Never automatically retry a destructive or remote-write step. Report the failure with the exact command and output, and ask the user before any further action.

### 5. Commit

Once all Quality Gates pass, commit per the phase's Commit Rule:

- Respect this repo's (and every target's) Git Safety rules: never push unless the user explicitly asked for that push, never force-push, never commit to `main` on a target that has a Git Safety rule against it, use the applicable skill (`gitlab-mr`, `github-pr`) if the phase's deliverable is meant to land via PR/MR on a target with a remote.
- If the phase's deliverable spans multiple repos (e.g. a canon change in the source repo plus a change in a remote target), commit each repo separately with its own message — do not squash unrelated repos into one commit.

### 6. Report and Advance

Status Report Format per phase:

```
## Phase {N} — {Title} — {STATUS}

### Quality Gates
- [x] ...
- [x] ...

### Repair Attempts: {0-3}
### Commit(s): {repo}: {hash}
```

Move to the next phase only after the current phase's commit succeeds and every gate passes. Return to Step 1 for the next phase.

## Constraints

- Execute ONE phase at a time, in order. Never start phase N+1 before phase N is committed and validated.
- No sub-agents, no git worktrees, no branch switching performed by this skill.
- Never push, force-push, or open a PR/MR without the user's explicit go-ahead for that specific action, even if the plan describes it as the phase deliverable.
- Every repo and remote target the plan touches must remain stable and verifiable after every phase.
- Do not skip or reorder phases.

## Final Summary

After all phases complete:

```
# Execution Summary

| Phase | Title | Status | Commit(s) | Repair Attempts |
|-------|-------|--------|-----------|------------------|
| 1     | ...   | PASS   | ...       | 0                |
```

## Next Step

After the Final Summary, always print this block. Resolve `<reference-path>` to the actual investigation / report that fed `generate-ops-plan` in this cycle before printing — never leave the placeholder.

**Available follow-ups:**

- `implementation-review @<reference-path>` — **known gap, not adapted for ops**: this skill judges an implementation against tests/observable code behavior, not against a config/canon/doc outcome. Useful only for a loose, manual read; do not expect its structural framing to fit. Needs an `implementation-review-ops` sibling analogous to `plan-review-ops`, not built yet — tracked in the source investigation's "Known gaps for future work".
- `plan-review-ops @task/PLAN_<NAME>.md @<reference-path>` — only if the plan itself proved wrong during execution (scope mismatch, missing phases) and needs re-anchoring before further work.
- `execute-ops-plan @task/PLAN_<NAME>.md` — only if one or more phases ended in FAIL after the repair budget AND the underlying blocker has already been resolved.

## Guardrails

1. Never spawn sub-agents or create git worktrees; this skill runs entirely in the current session, on the branch(es) already checked out.
2. Never write to a remote mount-point that has not been verified live (mount status `ok` **and** real-content `ls`) immediately before that write.
3. Never perform a destructive operation (`git mv`, force, `rm`, `stash`, push) without explicit per-action confirmation, regardless of what the plan says.
4. If a phase's Boundaries, Hard Constraints, or Locked Decisions would be violated by the only available way to deliver it, STOP and ask rather than improvising a workaround.
5. Respond in the language the user used in the invocation.