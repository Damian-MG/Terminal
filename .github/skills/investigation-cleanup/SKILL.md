---
name: investigation-cleanup
description: "Clean up temporary investigation documentation after review. Use when a repo has investigation, discovery, research, or incident notes that should remain temporary: keep only unresolved investigations, move durable conclusions into normal documentation when needed, and delete resolved temporary artifacts when no permanent documentation is warranted."
allowed-tools:
  - read
  - grep
  - glob
  - exec
  - write
  - edit
permissions:
  allow:
    - Read(docs/**)
    - Write(docs/**)
    - Edit(docs/**)
    - Exec(ls docs/investigations*)
    - Exec(find docs/investigations*)
---

> Note: deletion of resolved investigation files is part of this skill, but `rm` is intentionally NOT pre-approved.
> Each delete is requested individually so the user can approve each destructive operation.

# Investigation Cleanup

Use this skill when the task is to review an investigation area and decide what should stay, move, or be removed.

This skill is about documentation hygiene, not implementation.

In these repos the investigation area is `docs/investigations/` (created and maintained by the `investigation` skill).

## Goal

Treat investigation material as temporary working documentation.

After cleanup:

- unresolved investigations remain in the investigation area
- resolved investigations do not remain there
- durable knowledge is moved into regular project documentation only when it is still useful

## Working model

Classify each investigation into one of these states:

1. Still open
2. Resolved, but contains durable knowledge worth preserving
3. Resolved, with no durable value beyond the temporary investigation itself

## Required flow

1. Identify the investigation area used by the repository (default `docs/investigations/`).
2. Inventory the investigation documents before editing anything.
3. Read enough code and documentation to determine whether each investigation is still active or already resolved.
4. For each resolved investigation, decide whether any conclusion should survive outside the investigation area.
5. Move or rewrite only the durable conclusions into normal project documentation when needed.
6. Delete resolved investigation artifacts once their durable content, if any, has been preserved elsewhere.
7. Leave still-open investigations in place.

## How to decide

Treat an investigation as still open when:

- the underlying defect, decision, or missing behavior is not yet implemented
- the repository still reflects the gap described by the investigation
- the notes are still actively guiding unfinished work

Treat an investigation as resolved when:

- the underlying problem is already fixed or the decision has already been implemented
- the investigation mainly records past analysis, intermediate options, or implementation follow-ups that are no longer active

Treat content as durable when it documents:

- user-facing behavior that remains relevant
- stable operational guidance
- a lasting architectural or API contract
- a permanent workflow rule that future contributors still need

Treat content as non-durable when it is mainly:

- debugging history
- temporary plans
- intermediate evaluation notes
- issue triage context that no longer affects current work

## Migration rules

- Prefer updating existing permanent documentation instead of creating a new permanent document.
- Rewrite temporary notes into concise permanent documentation instead of copying investigation history verbatim.
- Keep permanent docs focused on the final rule, contract, or behavior.
- Do not preserve investigation chronology unless it is still operationally important.

## Guardrails

1. Do not keep resolved investigations in the investigation area as archive material.
2. Do not create permanent docs unless the information is still useful after the issue is resolved.
3. Do not move speculative, outdated, or superseded conclusions into permanent docs.
4. Do not mix source-code changes into an investigation-cleanup task unless the user explicitly asks for implementation work too.
5. If local changes already exist in investigation files, preserve the user's edits while applying the cleanup outcome.

## Expected output

When using this skill, report:

1. Which investigations were kept because they are still open
2. Which investigations were removed because they were resolved
3. Which durable conclusions were migrated into permanent documentation
4. Which resolved investigations were deleted without replacement because no lasting documentation was needed
