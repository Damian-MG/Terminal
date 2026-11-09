---
name: generate-plan
description: Generate an implementation plan with identified deliverables, explicit dependencies and acceptance evidence from a PRD, investigation, spec, report or conversation. Use when the user asks to plan implementation or break work into phases.
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

# Generate Implementation Plan

Produce a phased implementation plan from the provided context. Each deliverable is verifiable and integrable once its declared dependencies are satisfied. Phases group related deliverables; their numbering does not impose execution order.

## Context Discovery

Determine the source of planning context:

1. **Referenced document** — If the user mentions a file (e.g., `docs/investigations/some-topic/README.md`, `task/FEATURE_SPEC.md`), read it.
2. **Conversation context** — If the user describes requirements directly, use that.
3. **Default** — If no context is provided, ask the user what to plan.

Extract from the source:
- A short descriptive name for the plan (from filename, feature, or topic)
- The full requirements and scope

## Extract the Implementation Objectives

Before writing phases, distill the context into a compact **Objectives** section that will sit at the very top of the plan. This section is the north star for any independent agent that later implements the plan: it must be able to read only the Objectives and know what success looks like, without re-reading the source document.

Extract and write:

- **Primary Goal** — one sentence: what the implementation must achieve in the real world (user-visible or system-visible outcome). Not the feature name, not the technical change.
- **In Scope** — bullet list of the concrete outcomes that belong to this implementation.
- **Out of Scope / Non-Goals** — bullet list of things that might look adjacent but must NOT be touched or built. Be explicit about sibling modules, shared parents, and tempting refactors.
- **Success Criteria** — bullet list of observable, testable conditions that prove the implementation is done. Each criterion must be checkable without reading the plan phases.
- **Hard Constraints** — non-negotiable rules the implementing agent must respect across all phases (e.g., "must not change observable behavior of function F", "must only touch module A", "must preserve existing API contract").

These five fields are mandatory; use the source, conversation and applicable project rules. Ask only when a missing decision materially affects scope, behavior or acceptance. State when no additional work-specific hard constraints apply rather than inventing them.

## Constraints

- Identify deliverables with stable IDs and declare their actual dependencies, including across phases. Explain the prerequisite outcome each dependency supplies; use "none" for independent deliverables.
- Dependencies must be acyclic. Each delivery must be verifiable and leave the integrated repository buildable once its prerequisites are satisfied.
- Tests must be simple and readable
- Do NOT predefine low-level implementation steps (function names, line changes). Define objectives and deliverables
- Model shared types, config and utilities as prerequisites of the deliverables that consume them. Document necessary interface agreements between deliverables.
- Hard Constraints contain only justified, work-specific restrictions. Reference applicable project rules without duplicating them. Do not introduce general permission, commit or repair policies.
- Leave agent allocation, concurrency, worktrees and integration strategy to `execute-plan`. Identify independent outcomes without forcing artificial task splits or a fixed agent count.
- If a phase creates or modifies classes/modules, invoke `software-design-principles` before drafting that phase's Functional Deliverable, and design it consistently with DI/SRP/composition

## Phase Structure

Every phase must contain exactly these 8 sections:

### 1. Phase Goal
Single sentence describing what this phase achieves.

### 2. Functional Deliverable
What working functionality exists after this phase, with a stable ID for each deliverable and any interface agreement required by its consumers.

### 3. Boundaries
What is explicitly out of scope for this phase.

### 4. Required Tests
- **Unit**: Specific unit test scenarios
- **Integration**: End-to-end scenarios
- **Edge cases**: Boundary conditions and error cases

### 5. Quality Gates
- [ ] All tests pass
- [ ] No lint or type errors
- [ ] No regressions in integrated functionality
- [ ] Build succeeds

### 6. Validation Strategy
Commands to run and expected outputs to verify the phase is complete.

### 7. Dependencies
For each deliverable ID, list prerequisite deliverable IDs and the outcome required from each, or "none". Include external prerequisites and their availability. Numbering is not a dependency.

### 8. Acceptance Evidence
Map deliverable IDs and Success Criteria to the concrete test results, observable behavior, artifacts or document checks that demonstrate acceptance. Reference Required Tests and Validation Strategy rather than duplicating them.

## Output

Create `task/PLAN_<NAME>.md` with this structure:

```markdown
# Implementation Plan

> Generated from: [source path or "conversation context"]

## Objectives

> **These objectives are the contract for the implementation. Any agent executing this plan must read these objectives and Locked Decisions before its task and verify every change against them. Material changes to this contract require user agreement; editing the plan does not authorize a scope change.**

### Primary Goal
[One sentence — the real-world outcome]

### In Scope
- [Concrete outcome 1]
- [Concrete outcome 2]

### Out of Scope / Non-Goals
- [Thing that must NOT be touched 1]
- [Thing that must NOT be touched 2]

### Success Criteria
- [ ] [Observable, testable condition 1]
- [ ] [Observable, testable condition 2]

### Hard Constraints
- [Non-negotiable rule 1]
- [Non-negotiable rule 2]

## Locked Decisions

> **Decisions taken during review rounds that change what the executor builds. Each is a settled contract item, honored exactly like a Hard Constraint. Filled by `resolve-plan-review-decisions`; empty at creation. Never holds discarded alternatives or rationale — those live only in the loop-state side-car.**

_(none yet)_

---

## Phase 1 — [Short title]
### 1. Phase Goal
### 2. Functional Deliverable
### 3. Boundaries
### 4. Required Tests
### 5. Quality Gates
### 6. Validation Strategy
### 7. Dependencies
### 8. Acceptance Evidence

## Phase 2 — [Short title]
(same 8 sections)
```

Guidelines:
- Keep the plan focused on Objectives, Locked Decisions and the phase contract.
- Allow useful paths, existing interfaces and validation commands; avoid prescribing unnecessary internal implementation details.
- State dependencies explicitly. Presentation order does not constrain scheduling, including between phases.
- Each accepted delivery leaves its integrated dependency set buildable with applicable regression checks passing.
- The plan must cover the entire scope — nothing left unplanned
- Include the `## Locked Decisions` section (empty, `_(none yet)_`) right after Objectives; it is filled later by `resolve-plan-review-decisions`

## Initialize the loop-state side-car

The plan-review ↔️ resolve-plan-review-decisions loop keeps its state in `task/PLAN_<NAME>.state.md`, next to the plan. It is the loop's memory (iteration count, decisions taken, discarded paths, per-round finding summaries) and is **never read by `execute-plan`**. Create it with:

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

Ownership is shared: `generate-plan` initializes it; `resolve-plan-review-decisions` records decisions locked, discarded paths, and consistency-check result; `plan-review` records each round's findings and verdict. The plan wins over the side-car: a `Locked Decision` already in the plan is authoritative.

## Validation Checklist

Before finalizing, verify:
- [ ] The **Objectives** section is present at the top and contains all five fields (Primary Goal, In Scope, Out of Scope, Success Criteria, Hard Constraints)
- [ ] A `## Locked Decisions` section exists right after Objectives, empty (`_(none yet)_`)
- [ ] The loop-state side-car `task/PLAN_<NAME>.state.md` was created with `Iteration: 0`
- [ ] Every Success Criterion maps to identified deliverables and concrete Acceptance Evidence
- [ ] No phase introduces work that falls outside "In Scope" or violates "Hard Constraints"
- [ ] Every phase has all 8 sections
- [ ] All deliverables have stable IDs and justified dependencies, with no missing references or cycles
- [ ] Interface agreements and external prerequisites are explicit where needed
- [ ] Each delivery is verifiable and buildable with its declared dependencies
- [ ] No unjustified execution barriers, duplicated project rules or general permission/commit/repair policies were introduced
- [ ] No untested functionality introduced
- [ ] Full scope covered
- [ ] Source context documented in header

## Next Step

After the plan file is generated and the Validation Criteria above all pass, always print this block. Resolve `<NAME>` and `<reference-path>` to the concrete paths produced/consumed in this run before printing — never leave placeholders.

**Available skills:**

- `plan-review` (inputs: `task/PLAN_<NAME>.md`, `<reference-path>`) — evaluate the plan against its source before any execution.
- `resolve-plan-review-decisions` (inputs: `task/PLAN_<NAME>.md`, `<reference-path>`) — process the findings after `plan-review`; it is not a shortcut that skips the review pass.

**Recommendation:** invoke `plan-review` on `task/PLAN_<NAME>.md` and `<reference-path>` first. Only then may `resolve-plan-review-decisions` process that review's findings; there is no valid shortcut from `generate-plan` to `execute-plan`.