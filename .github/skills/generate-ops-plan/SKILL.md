---
name: generate-ops-plan
description: Generate a phased operations plan for non-code work (canon/config changes, AGENTS.md rewrites, skill-layout migrations, repo-layout changes, sync-check propagation) from any context source (investigation, report, or conversation). Produces `task/PLAN_<NAME>.md` with the same Objectives + Locked Decisions + 8-section phase contract as `generate-plan`, so it stays compatible with `plan-review-ops` and `resolve-plan-review-decisions`, but with phase gates expressed as verification checks instead of tests/build. Use when the work is operational/config/documentation rather than software implementation.
argument-hint: "[@task/PRD.md | @<source-doc>]"
allowed-tools:
  - read
  - grep
  - glob
  - exec
  - write
  - edit
  - webfetch
---

# Generate Operations Plan

Generate a phased operations plan from various context sources such as investigations, reports, conversations, or referenced documents.

## Role

You are a canon/ops operator specializing in incremental, verifiable, sequential delivery of configuration, documentation, and repository-layout changes. Produce a phased operations plan from the provided context.

## When to use this instead of `generate-plan`

- The work does not produce or modify application source code, and has no tests/build/lint to run.
- Typical triggers: enabling or disabling a canon skill set for a target in `repos.toml`, rewriting an `AGENTS.md`, migrating a skills layout (e.g. `agents/` → `.agents/`), onboarding a new target repo, propagating canon changes with `sync-check apply`, or any multi-step change to config/docs/repo layout across one or more checkouts.
- If the work mixes both (it also touches application source with real tests), prefer `generate-plan` instead.

---

## Instructions

### Step 1: Identify and Read the Context Source

Same as `generate-plan`:

- **Referenced document**: if the user mentions a specific file (e.g., `@docs/investigations/<slug>/README.md`), read that file.
- **Conversation context**: if the user provided the decisions in the conversation itself, use that as the source.
- **Default PRD**: if no specific context is provided, look for `task/PRD.md`.

After reading the context, extract:
- A short descriptive name for the plan (derived from the source filename, investigation slug, or conversation topic).
- The full scope of the operation to be planned.

### Step 2: Extract the Implementation Objectives

Before planning phases, distill the context into a compact **Objectives** section that will sit at the very top of the plan. This section is the north star for any independent agent that later executes the plan: it must be able to read only the Objectives and know what success looks like, without re-reading the source document.

Extract and write:

- **Primary Goal** — one sentence: the real-world state that must exist after the operation (e.g. "the canon has Rust enabled for target X and its `AGENTS.md` follows the dual-role template"). Not a feature name, not a technical mechanism.
- **In Scope** — bullet list of the concrete outcomes that belong to this operation.
- **Out of Scope / Non-Goals** — bullet list of things that might look adjacent but must NOT be touched (sibling repos, unrelated open plans in a target repo, unrelated skills or targets).
- **Success Criteria** — bullet list of observable, verifiable conditions that prove the operation is done. Each criterion must be checkable with a command or a direct content comparison, without reading the plan phases.
- **Hard Constraints** — non-negotiable rules the executing agent must respect across all phases (e.g. "never write to the mount if it is down", "never mix this work with `task/PLAN_X.md`", "must not modify any file outside `repos.toml` and `<target>/AGENTS.md`").

These five fields are mandatory. If the source context does not provide enough information to fill any of them, ask the user before generating the plan.

### Step 3: Apply These Constraints

- Each phase must be independently deliverable and leave every touched repo/target in a stable, verifiable state. There is no "buildable" state to preserve — "stable" means: `git status` is clean or matches the expected pending change, and `sync-check diff` (when applicable) shows no unexpected drift.
- Phases must not rely on future phases to function.
- Verification steps must be simple, deterministic, and re-runnable (`sync-check diff`, `git status --short`, `git diff`, an exact file-content comparison, or a single command with a known expected output) — never a subjective check ("looks right").
- Unlike `generate-plan`, concrete operational detail (exact file paths, exact commands, exact target/skill names) SHOULD be specified whenever the source context already decided them. Ops work is usually the deterministic execution of an already-closed decision, not a design space left to the implementer. Do not invent a command or path the source context does not support.
- Before any phase that writes to a remote-mounted target, that phase's verification must include the remote mount guardrail below.
- Cross-cutting concerns (a canon change needed by more than one phase or target) must be addressed in the earliest phase that needs them.
- Maximum automated repair attempts per phase before aborting: 3 — and only for idempotent, read-only checks (e.g. re-running a verification command to rule out a transient issue). Destructive or remote-write steps are never retried automatically; see Failure Policy in Step 4.

#### Remote mount guardrail

If a phase touches a target with `[targets.mount]` in `repos.toml` (or otherwise lives behind a remote mount such as sshfs), that phase's Required Tests must include, before any write: `sync-check mounts status` reports the target `ok`, **and** a real-content `ls` of the mount confirms live data (mount status alone is not sufficient evidence — see `docs/PROCEDIMIENTOS-CANON.md`). If the mount is down, the phase must STOP before writing — an empty local mount-point silently absorbs writes and hides them once the mount reconnects.

### Step 4: Structure Each Phase with These 8 Sections

Every phase must contain exactly these 8 sections, in this order (identical names to `generate-plan`, so the plan stays structurally familiar — but review it with `plan-review-ops`, not `plan-review`: the content below is ops-flavored, and `plan-review`'s checklist wording assumes software gates and will misjudge it):

#### 1. Phase Goal
A single sentence describing what this phase achieves.

#### 2. Functional Deliverable
The concrete state that exists after this phase: a file's new content, a config entry added/removed, a directory renamed, a target added to `repos.toml`. Not a "feature" — a verifiable state.

#### 3. Boundaries
What is explicitly out of scope for this phase. Name any repo, file, or unrelated open plan (in this repo or a target) that must NOT be touched.

#### 4. Required Tests
- **Unit**: a single deterministic check confirming one file/config change (e.g. `sync-check diff --target X --paths <p>` reports `OK`).
- **Integration**: a check confirming the target's overall state is consistent after the phase (e.g. `sync-check diff --target X` reports no unexpected `DRIFT`/`MISSING_IN_TARGET`).
- **Edge cases**: things that must explicitly NOT have changed — unrelated files, unrelated plans, other targets, other skills.

#### 5. Quality Gates
A checklist (`- [ ]`) of concrete, re-runnable verification conditions, restated in ops terms — never literally "tests pass" / "build succeeds". Examples: `sync-check diff --target X` clean for the paths this phase touched; mount status `ok` verified immediately before and after (if remote); the rewritten file's content matches the decided template/structure; no file outside this phase's Functional Deliverable changed.

#### 6. Validation Strategy
The exact commands to run and the expected output, so any agent can re-verify the phase mechanically.

#### 7. Commit Rule
Commit only after all Quality Gates pass. Respect this repo's Git Safety rules: never push unless the user asked, never force, ask before any destructive git operation, use the applicable PR/MR skill (`gitlab-mr`, `github-pr`) if the phase's deliverable lands on a target with a remote.

#### 8. Failure Policy
If any Quality Gate fails, STOP. Automatically retry only idempotent, read-only checks (max 3 attempts) to rule out transient issues (e.g. a mount that reconnects). Never automatically retry a destructive or remote-write step (`git mv`, `sync-check apply`, a commit). Report the failure and ask before retrying it.

---

## Step 5: Generate the Plan

Create `task/PLAN_<NAME>.md` where `<NAME>` is the descriptive name extracted in Step 1 (e.g. `PLAN_home_servers_canon_onboarding.md`).

Use this structure:

```markdown
# Operations Plan

> Generated from: [source file path or "conversation context"]
> Kind: ops

## Objectives

> **These objectives are the contract for the operation. Any agent executing this plan must re-read this section before starting each phase and verify every change against it. Do not deviate without updating this section first.**

### Primary Goal
[One sentence — the real-world state that must exist afterward]

### In Scope
- [Concrete outcome 1]
- [Concrete outcome 2]

### Out of Scope / Non-Goals
- [Thing that must NOT be touched 1]
- [Thing that must NOT be touched 2]

### Success Criteria
- [ ] [Observable, verifiable condition 1]
- [ ] [Observable, verifiable condition 2]

### Hard Constraints
- [Non-negotiable rule 1]
- [Non-negotiable rule 2]

## Locked Decisions

> **Decisions taken during review rounds that change what the executor does. Each entry is a settled contract item the executing agent must honor exactly, like a Hard Constraint. Populated by `resolve-plan-review-decisions`; empty at plan creation. Never contains discarded alternatives or rationale — those live only in the loop-state side-car.**

_(none yet)_

---

## Phase 1 — [Short descriptive title]
### 1. Phase Goal
### 2. Functional Deliverable
### 3. Boundaries
### 4. Required Tests
### 5. Quality Gates
### 6. Validation Strategy
### 7. Commit Rule
### 8. Failure Policy

## Phase 2 — [Short descriptive title]
(same 8 sections)

...repeat for all phases...
```

**Important Guidelines:**

- No prose outside the phase sections.
- Concrete paths, target names, and commands ARE expected here (unlike `generate-plan`) whenever the source context already decided them — do not abstract away a decision that was already closed.
- Only objectives, deliverables, boundaries, and verification.
- Order phases by dependency (earlier phases must not depend on later ones).
- Each phase must leave every touched repo/target in a stable, verifiable state.
- No phase may introduce an unverified change.
- The plan must cover the entire scope of the source context — nothing left unplanned.
- Quality gates must be checkable via a concrete command or content comparison.
- Document the source context in the plan header (file path or "conversation context"), plus the `> Kind: ops` marker.
- Include the `## Locked Decisions` section (empty, `_(none yet)_`) right after Objectives. It is filled later by `resolve-plan-review-decisions`; do not populate it here.

---

## Step 6: Initialize the loop-state side-car

Identical format to `generate-plan`: the `plan-review-ops` ↔️ `resolve-plan-review-decisions` loop keeps its state in `task/PLAN_<NAME>.state.md`, next to the plan. This is what keeps ops plans compatible with `resolve-plan-review-decisions` (shared, unchanged) while `plan-review-ops` (not `plan-review`) supplies the correct structural checklist.

Create `task/PLAN_<NAME>.state.md` with the initial state:

```markdown
# Plan state — PLAN_<NAME>

- Iteration: 0
- Last review: (none yet)
- Verdict trail: (none yet)

## Decisions locked (round → decision)

_(none yet)_

## Discarded paths

_(none yet)_

## Round summaries

_(none yet)_
```

---

## Validation Criteria

Before finalizing the plan, verify:
- [ ] The **Objectives** section is present at the top and contains all five fields (Primary Goal, In Scope, Out of Scope, Success Criteria, Hard Constraints).
- [ ] A `## Locked Decisions` section exists right after Objectives, empty (`_(none yet)_`).
- [ ] The loop-state side-car `task/PLAN_<NAME>.state.md` was created with `Iteration: 0`.
- [ ] Every Success Criterion maps to at least one phase's Functional Deliverable or Required Tests.
- [ ] No phase introduces work that falls outside "In Scope" or violates "Hard Constraints".
- [ ] Every phase contains all 8 sections — no section may be omitted.
- [ ] Phases are ordered by dependency.
- [ ] Every phase touching a remote-mounted target includes the mount guardrail in its Required Tests.
- [ ] No phase introduces an unverified change.
- [ ] The plan covers the entire scope of the source context.
- [ ] Quality gates are checkable via a concrete command or content comparison.
- [ ] Output filename follows `PLAN_<NAME>.md` format.
- [ ] Source context and `> Kind: ops` are documented in the plan header.

---

## Next Step (REQUIRED final output)

**You MUST end your response with the block below.** Resolve `<NAME>` and `<reference-path>` to the concrete paths produced/consumed in this run before printing — never leave placeholders.

Print exactly:

```
**Available follow-ups:**

- `plan-review-ops @task/PLAN_<NAME>.md @<reference-path>` — evaluate the plan against its source before any execution.
- `resolve-plan-review-decisions @task/PLAN_<NAME>.md @<reference-path>` — process the findings after `plan-review-ops`; it is not a shortcut that skips the review pass.

**Recommendation:** `plan-review-ops @task/PLAN_<NAME>.md @<reference-path>` — run the operations review first. Only then may `resolve-plan-review-decisions` process that review's findings; there is no valid shortcut from `generate-ops-plan` to `execute-ops-plan`. Do not use plain `plan-review` — its structural checklist assumes software gates (tests/lint/build) and will misjudge this plan.
```