---
name: investigation
description: Document-first investigation workflow for non-coding tasks. Use when the user asks for an investigation, research, discovery, task analysis, or process guidance.
argument-hint: "[topic to investigate]"
allowed-tools:
  - read
  - grep
  - glob
  - exec
  - write
  - edit
  - webfetch
permissions:
  allow:
    - Read(docs/investigations/**)
    - Write(docs/investigations/**)
    - Exec(ls docs/investigations*)
    - Exec(mkdir -p docs/investigations/*)
    - Exec(find docs/investigations*)
---

# Investigation Workflow

Use this skill for investigation work, not for code implementation.

Treat investigations as living documentation under `docs/investigations/`.

## Purpose and Scope

Investigation is about **understanding**, not planning execution.

The goal is to go deep into the topic and surface:

1. Problems and root causes
2. Risks, pitfalls, and things to be careful about
3. Open doubts and unknowns that need clarification
4. Possible solutions, alternatives and trade-offs (without committing to one)
5. Constraints, dependencies, and affected systems
6. Relevant evidence: code refs, docs, links, data

An investigation is **not** an implementation plan. Do not produce step-by-step
task lists, sprints, milestones, "Phase 1/2/3" breakdowns, branch/PR strategy,
or estimates. Building the plan is a separate, later task that consumes the
investigation as input.

If the user asks for both, complete the investigation first and explicitly stop
before planning, then ask whether to proceed with planning as a separate step.

## Required Flow

1. Ensure `docs/investigations/` exists. Create it only if missing.
2. Extract the investigation topic and convert it to a short slug.
3. Search `docs/investigations/` for related investigations before writing anything.
4. If related documentation exists, update existing files first.
5. Do not create a new document when related documentation already exists.
6. Create a new folder only when no related investigation exists.
7. When creating a new investigation, add `README.md` as the entry point and index.

## Relatedness Rule

Treat an investigation as related when topic, scope, system, or objective overlap meaningfully.

When uncertain, prefer updating an existing investigation instead of creating a new one.

## New Investigation Structure

Create:

`docs/investigations/<topic-slug>/README.md`

Initialize `README.md` with:

1. Title
2. Context and goal of the investigation
3. Main findings (summary of what was learned)
4. Problems and risks detected
5. Possible solutions / alternatives with trade-offs (no commitment to one)
6. Open questions, doubts, and follow-ups that need clarification
7. Evidence and references (code paths, docs, links)
8. Index of related documents in the same folder
9. Last updated date

Do not include implementation steps, phases, task breakdowns, or estimates.
Those belong to a separate planning task.

## Update Rules

1. Keep previous conclusions unless they are superseded by better evidence.
2. Add new findings clearly with date/context.
3. Keep README as the canonical index to other documents.
4. Prefer appending or refining existing docs over creating new files.

## Guardrails

1. Do not start code-generation workflows unless the user explicitly asks for code.
2. Do not produce implementation plans, phase/sprint breakdowns, or estimates.
   If the user asks for a plan, finish the investigation first and propose
   planning as a separate task.
3. Prefer depth over breadth: dig into the topic, validate assumptions, cite
   evidence. Surface doubts explicitly instead of papering over them.
4. Do not duplicate content across multiple investigation files.
5. Do not create parallel investigations for the same topic.
6. Keep documents concise, practical, and focused on understanding.
7. When the investigation's scope involves creating or modifying classes/modules, invoke `software-design-principles` before writing "Main Findings", and check the candidate design against its axes (DI, SRP, DRY, composition, Law of Demeter).

## Output Expectation

When using this skill, report:

1. Which existing investigation was reused (if any)
2. Which files were updated
3. Whether a new investigation folder was created (only if no related topic existed)
4. The Next Step block below, with `<slug>` resolved to the actual folder name.

## Next Step

Always print this block as the last section of the report. Resolve `<slug>` to the actual investigation folder name before printing — never leave the placeholder.

**Available commands:**

- `resolve-open-questions @docs/investigations/<slug>/README.md` — close remaining open questions interactively before planning.
- `generate-plan @docs/investigations/<slug>/README.md` — feed the investigation into planning.

**Recommendation:** depends on the state of the investigation:

- Open Questions still block planning → `resolve-open-questions @docs/investigations/<slug>/README.md` first.
- No Open Questions left → `generate-plan @docs/investigations/<slug>/README.md`.
- User explicitly handing off to humans / GitHub Projects without immediate intent to plan → terminal step, no automatic continuation.