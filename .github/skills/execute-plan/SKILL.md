---
name: execute-plan
description: Execute an implementation plan with autonomous sub-agents, dependency-based parallel scheduling, isolated worktrees, reviewed commits and validated integration into the work branch. Use when the user asks to execute, run, or implement a plan file.
argument-hint: "[@task/PLAN_<NAME>.md]"
allowed-tools:
  - read
  - grep
  - glob
  - exec
  - edit
  - write
---

# Execute Implementation Plan

The plan defines outcomes, dependencies and acceptance. The orchestrator decides how to deliver them, including parallel work across phases. Apply project rules and the user's existing authorization throughout the execution.

## Input and Contract

- Plan path (default: `task/PLAN.md`) and the reference document or conversation from which it was generated.
- Read Objectives, Locked Decisions, deliverables, Dependencies and Acceptance Evidence before organizing work. Give every agent the relevant contract before its task.
- Ignore `task/PLAN_<NAME>.state.md`: it is private review-loop memory, not execution input. Locked Decisions in the plan are authoritative.
- A material scope change requires user agreement; editing the plan does not authorize it. Report contradictory requirements rather than silently dropping them.
- For an older plan, derive scheduling dependencies from its actual requirements. If it contains conflicting permission, commit or scheduling policies, resolve those against explicit user decisions and applicable project rules; ask only if a material conflict remains. Do not invent missing acceptance requirements.
- Invoking this skill or approving a plan does not itself authorize sub-agents, commits, network access or destructive operations. Obtain any explicit authorization required by applicable instructions before the affected operation; delegate only permissions actually granted. If required authorization is absent or denied, report the limitation rather than bypassing it.

## Prepare a Reproducible Base

1. Verify the authorized work branch and inspect HEAD, staged and unstaged changes, and untracked files. Reuse authorization already given; obtain a missing branch/work authorization under `AGENTS.md` before modifying files.
2. Distinguish prerequisites of this execution from unrelated pre-existing work. Preserve the latter; a dirty checkout alone is not a reason to stop.
3. Prepare and record a committed base containing the plan and any authorized prerequisite changes needed by workers. Stage only identified work-related files or hunks. If ownership or intent is ambiguous and isolation cannot resolve it, ask before including or changing those edits.
4. Use an integration worktree if unrelated changes make the current checkout unsuitable. Record the target branch, base commit and initial unrelated state so the final comparison can verify preservation.

Never use stash, pull, discard changes or publish work merely to prepare execution. Follow applicable project permissions for those operations.

## Roles and Isolation

- **Orchestrator:** owns decomposition, scheduling, worktree setup, review coordination, integration and final evidence. Monitors actual progress and resolves or delegates integration issues.
- **Implementer:** owns its scoped implementation, tests, documentation, repairs and local commits; reports commit hashes and validation evidence.
- **Reviewer:** checks the delivery against the contract, diff and evidence; reports concrete findings and their disposition. Use an agent other than the implementer for review.

Each writing sub-agent works in its own worktree and branch from a recorded committed base. Separate file assignments in a shared checkout do not isolate Git's index. Coordinate overlapping changes and interfaces; workers do not independently update the target branch. Read-only reviewers may inspect an immutable committed delivery without a separate writing worktree.

## Schedule by Dependencies

Build a task graph from the deliverables and their real prerequisites. Task scope and acceptance must map back to the plan; decomposition does not expand it.

- Launch ready independent tasks concurrently within available agent capacity, including tasks from different phases. Phase numbering is not a scheduling barrier.
- Make a dependency's required committed result available in the worker's base before starting work that consumes it. Readiness depends on the actual prerequisite outcome, not on completing unrelated tasks in its phase.
- If work is indivisible, delegate it to one implementer and arrange separate review. Do not silently substitute a wholly sequential orchestrator implementation for delegation.
- Track task, role, agent, worktree, branch, base, dependencies, status, commits and review results. Explain serialization caused by real dependencies or resource limits.
- If delegation or isolation is unavailable, report the limitation and its effect instead of claiming that delegated parallel execution occurred.

## Delegate Autonomous Tasks

Provide each agent:

- Role, task scope, committed base, worktree and branch, with boundaries and interface agreements.
- Objectives and Locked Decisions, supplied directly or through explicit readable references; include task-specific dependencies, acceptance and validation requirements.
- The existing authorization to perform its assigned work and necessary local commits without asking again for each operation. Escalate decisions outside its scope to the orchestrator.
- An instruction to read applicable project rules and relevant skills. For code tasks, explicitly invoke `software-design-principles` and applicable stack/framework skills; do not assume these from directory names or assume every environment exposes identical agent capabilities.
- A delivery requirement: commit all task changes, report ordered commit hashes, commands and observed results, outstanding failures and any uncommitted state. Select own changes explicitly rather than using indiscriminate `git add -A`.

Workers may make multiple local commits, including intermediate checkpoints. A checkpoint is not an accepted delivery. Completed deliveries and integrations must pass their applicable checks.

## Review, Integrate and Validate

1. Review each committed delivery with a separate reviewer for acceptance, scope, project conventions, relevant design/architecture rules and adequate evidence. Apply complexity limits only when required by the project or work contract; distinguish estimates from analyzer measurements.
2. Route findings to the implementer for correction and new commits; verify their resolution. Re-review relevant changes before accepting the delivery.
3. Integrate reviewed commits into the authorized target branch under the orchestrator's supervision. Serialize mutations of that branch. Choose an appropriate Git integration method and record which task commits were incorporated, including equivalent commits if cherry-picking changes hashes.
4. Delegate conflict fixes or necessary integration changes to an isolated writing agent, then review and integrate its commits. A clean Git merge alone does not prove functional integration.
5. Run the plan's and project's applicable checks with their required configuration and extras. Workers validate their scope; validate affected interactions after integration and the complete result at closure. Do not replace the plan's commands with a fixed Python/frontend checklist.
6. Release newly ready work as prerequisites become available; there is no need to wait for every agent in a phase to finish.

## Repair Without Unproductive Loops

For each failure, record the failing check, hypothesis, attempted correction and observed result. Continue autonomous repair within scope while there is new evidence or a measurable reduction in failures; no fixed three-attempt limit applies.

If an attempt produces no progress, reassess the cause before another attempt. If the reassessed attempt reproduces the same blocker without new evidence, stop that repair line and report the blocker. Do not reset this history by changing agents or renaming the task.

Continue independent viable tasks. Ask the user when a missing decision, scope expansion or unavailable external prerequisite requires their input; do not turn every technical failure into a permission request. Preserve committed progress and report incomplete status when the blocker cannot be resolved within the contract. Never relax acceptance to manufacture a pass.

## Verify Closure

Declare completion only when:

- All Success Criteria and Acceptance Evidence are satisfied with observed results.
- Review findings affecting acceptance are resolved.
- Every delivery and necessary integration correction is committed and incorporated into the authorized work branch; verify ancestry or recorded equivalent diffs as appropriate.
- Final integrated checks pass on the recorded final commit. Any subsequent code/document change is committed and its relevant checks repeated.
- No execution-owned changes remain uncommitted in the integration or worker worktrees, and no required task commits remain only on worker branches.
- Initial unrelated changes remain preserved. Distinguish them from pending execution work rather than requiring a globally clean original checkout.

Report the base and final commits, branch, per-deliverable agents/roles, reviews, integrated commits, validations, repairs and any limitations. If incomplete, identify accepted progress and unresolved work instead of reporting success. Do not push or open a PR/MR unless authorized. Preserve unintegrated work; clean up only task-owned temporary worktrees/branches when their committed results are safely integrated and project permissions allow it.

## Next Step

After successful closure, recommend `implementation-review` with the actual source investigation / PRD / spec path (or original conversation context). If the plan proved contradictory or incomplete, recommend `plan-review` with the plan and reference. Resume `execute-plan` after a blocker is resolved using recorded task and integration state; do not repeat completed work or reset repair history.