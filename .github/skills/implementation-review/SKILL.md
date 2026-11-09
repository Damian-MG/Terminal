---
name: implementation-review
description: Evaluate an existing implementation against its reference documentation (investigation, issue, spec, PRD, design doc). Use when the user says something like "I implemented X that is in <doc reference> and I need an evaluation".
argument-hint: "<path or url to the reference document>"
allowed-tools:
  - read
  - grep
  - glob
  - exec
  - webfetch
  - write
  - edit
permissions:
  allow:
    - Read(**)
    - Write(docs/investigations/**)
    - Edit(docs/investigations/**)
---

# Implementation Review

Evaluate an implementation against the document that specifies what it should do. Produce a written evaluation only — do not change code. The only permitted write is the dedicated code-review side-car described below.

## What the Invocation Prompt Needs to Carry

The user's invocation only needs to provide **session-specific information**:

1. **What was implemented** — a one-line summary of the change.
2. **Where it lives** — the reference document (section if applicable), plus any of: branch name, commit range, PR number, or touched paths.
3. **What to review** — explicit concerns or axes the user wants emphasized (e.g., "focus on whether X was violated", "check if sibling module Y was accidentally touched", "the implementation felt too big for what the doc described").
4. **Hard constraints for this specific feature**, if any — e.g., "must only touch module A, not B or C", "must not change observable behavior of function F".

Everything else — the general flow, default evaluation axes, output shape, guardrails — is in this skill. The user should not need to restate them.

## Required Flow

1. **Open the reference.** Read the document the user pointed to in full (or the named section). Identify the stated goal, the design choices, the explicit non-goals, and any acceptance criteria.
2. **Locate the implementation.** Map each claim in the reference to the actual code, tests, and wiring. Use git history (`git log main..HEAD`, `git diff --stat main..HEAD`) when the user does not name files.
3. **Read before judging.** Do not evaluate a module from its name. Open each file the reference mentions or that shows up in the diff.
4. **Cross-check claims.** For every "we reuse X" / "this is stateless" / "this handles Y" assertion in the reference, verify it in code. Flag any that is aspirational rather than real.
5. **Trace at least one end-to-end path.** Follow a representative input through the new code into the surrounding system. Latent bugs usually live at the seams, not inside the new module.
6. **Check the scope boundary.** Compare the diff against the stated scope. Flag any touched module that was declared out of scope (sibling processors, unrelated services, shared parents that rippled into unintended places).
7. **Check the tests.** Note what the test suite does and does not cover. A case that the tests never exercise is a likely gap. Disproportionate test volume vs. production code is a smell to surface.
8. **Load the code-review loop state.** If a follow-up artifact already exists, read its dedicated side-car `<follow-up-path>.state.md`; never use `task/PLAN_<NAME>.state.md` for this flow. If it does not exist, treat this as the first round.
9. **Write the evaluation.** Output it directly in the chat; do not create files unless the user asks.
10. **Record the review round.** When a follow-up artifact is known, update only its dedicated side-car with the iteration, verdict trail, findings raised, and resurfaced finding IDs. Do not modify source code, tests, plans, or the original investigation.

## Code-review loop state

The implementation-review ↔️ followup-from-review loop has its own state, separate from plan review. The side-car lives beside the follow-up artifact:

`docs/investigations/<slug>/FOLLOWUP_<n>_<name>.md.state.md`

Its format is:

```markdown
# Code review state — FOLLOWUP_<n>_<name>

- Iteration: 1
- Last review: 2026-09-18 by implementation-review
- Verdict trail: needs-rework

## Decisions locked (round → decision)

_(none yet)_

## Discarded paths

_(none yet)_

## Round summaries

- r1: raised [F1, F2] → verdict needs-rework
```

`implementation-review` owns iteration, verdict trail, and findings raised or resurfaced. `followup-from-review` owns decisions, discarded paths, and dispositions. The plan-review side-car `task/PLAN_<NAME>.state.md` is never read or written for this flow, and `execute-plan` does not consume either review side-car.

## Default Evaluation Axes

Evaluate against whatever axes the user named. Always apply the following defaults in addition, because they catch the recurring failure modes on this codebase:

- **Correctness** — does the code do what the reference says it does, including at seams and on edge cases the reference enumerates?
- **Scope adherence** — did the change touch only the modules it was supposed to? Sibling/parent modules pulled in by signature changes or inheritance are a red flag. Required reading for this axis: the explicit scope section of the reference and any "non-goals" it lists.
- **Behavioral preservation** — for refactors and additions to shared code paths, was observable behavior of existing callers preserved? Signature changes that force ripple edits elsewhere usually mean no.
- **Complexity vs. mental model** — when the user describes the intended solution as simple and the implementation is large, locate *where* the size comes from: indirection layers, synthetic entries shuttled between phases, dead defaults, speculative generality. Compare the delivered shape against the simplest shape that would satisfy the reference.
- **Cognitive Complexity** — this axis is **mandatory and quantitative**, not opt-in. For every function that is **new or has its body modified** in the diff under review, estimate its SonarQube Cognitive Complexity score (rule `python:S3776`, threshold 15) and report it as a number in the findings, regardless of whether SonarQube has been run yet. A qualitative flag ("this looks complex") is not acceptable output for this axis.
    - **How to estimate** (SonarQube rule, summarised): start at 0. Add **+1** for each control-flow structure (`if`, `elif`, `else`, `for`, `while`, `try`/`except`/`finally`, `with`, ternary `x if c else y`, comprehensions with `if`, `match`/`case`). Add **+nesting_level** on top of that for any of those structures that is nested inside another control-flow structure inside the same function (so an `if` inside a `for` is +1+1, an `if` inside a `for` inside a `while` is +1+2). Add **+1 per extra binary boolean operator** in a chain (`a and b and c` → +2; mixing `and`/`or` adds +1 at each switch). Add **+1** for each `break`/`continue` with a label (rare in Python) and for each recursion call. Lambda bodies and inline list/set/dict/generator comprehensions count as control-flow inside the parent function.
    - **What to report**: a per-function table or bullet list with `path:function → score (status)`, where status is `OK` for ≤ 15, `WARN` for 16-19, `BLOCKER` for ≥ 20. List the dominant cost contributors (which lines or constructs drive the bulk of the score) for any function over 15. If you cannot estimate confidently (e.g., very long function, many helper boundaries), say so and give a conservative lower bound.
    - **Coverage rule**: at minimum, score every public method, every async method, and every function with > 30 logical lines in the diff. Unchanged functions are out of scope unless the reference document explicitly required modifying them.
    - **Typical smells** that push the score above 15: nesting depth > 3, long `if/elif` chains on the same variable, nested loops with inner conditionals and `break`/`continue`, boolean expressions combining more than 3 operators, flag parameters that branch the function body into distinct flows, a single function doing validation + transformation + I/O + reporting.
    - **Recommendations** for any function ≥ 16: name the specific extraction (guard clauses, extracted helpers, dispatch tables, polymorphism, predicate functions) that would land it under 15, and indicate the largest single block whose extraction would yield the biggest reduction. Do not write the patch unless asked.
- **Single Responsibility** (definition: `software-design-principles`) — does each new class/module have one reason to change? Logic split across the orchestrator and the new module (e.g., part of the construction of a data object in one place, part in another) fails this.
- **Dependency Injection** (definition: `software-design-principles`) — are collaborators passed via the constructor or pulled from global/module state? Are defaults in the constructor actually reachable in production, or dead surface?
- **Reuse over reimplementation** — are the reuse points promised by the reference actually reused? Is there a third copy of something that already existed twice?
- **DRY** — duplicated helpers, parallel parsing logic, repeated string-munging across modules.
- **Overengineering** — dead defaults, defensive `getattr`, unreachable branches, premature abstractions, parameters no caller varies.
- **Settings / configuration** — are new knobs justified? Are existing settings being bypassed or duplicated? A new setting that mirrors an existing one is almost always wrong.
- **Project conventions** — agreement with rules files (`AGENTS.md`, `.windsurf/rules/*`, style guides, exception-handling rules) that apply to the changed code.
- **Tests** — do tests cover the scenarios the reference enumerates, including multi-variant / edge cases? Is test volume proportional to the behavior, or is a fragile wiring test fleet masking a thin behavioral core?

## Output Expectation

Report, in this order:

1. **Overall shape** — one short paragraph: does the implementation match the reference at a high level? Name the single thing most worth fixing first.
2. **Findings**, grouped by severity (🔴 correctness or scope violation / 🟡 design / 🟢 minor). For each finding:
   - what the code does,
   - what the reference says or what the convention requires,
   - why it matters (concrete downstream consequence, not abstract criticism),
   - pointer to file and function / line range.
3. **Cognitive Complexity table** — always render a `path:function → score (status)` table covering every function in scope per the Coverage rule above, even when all scores are `OK`. If no qualifying function exists in the diff, state that explicitly. Functions scoring ≥ 16 must also appear as a 🔴 finding in §2.
4. **Summary against the axes the user named** — a short table or bullet list with a verdict per axis. Useful when the user explicitly listed concerns in the invocation.
5. **Recommendations** — ordered by priority, each actionable in one or two sentences. Do not write patches unless asked.
6. **Next Step** — always close the output with the block defined in the "Next Step" section below, resolved to the concrete reference path. Never leave placeholders.

## Next Step

Always print this block as the last section of the review. Resolve `<reference-path>` to the actual reference passed to this review before printing — never leave the placeholder.

**Available commands:**

- `followup-from-review @<reference-path>` — turn the findings into a follow-up doc that the user can triage and convert into a plan.
- `investigation-cleanup` — only when the investigation folder still holds temporary artifacts that should be retired.
- `implementation-review @<reference-path>` — re-evaluate the same scope after a fix.
- `plan-review @task/PLAN_<NAME>.md @<reference-path>` — only when the implementation surfaced that the plan itself was wrong; re-anchor the plan first.

**Recommendation:** depends on the findings above:

- 🔴 scope or correctness violations → `followup-from-review @<reference-path>` to scope the remediation before any code change.
- Any 🟡 or 🟢 finding to remediate → `followup-from-review @<reference-path>`.
- No findings, no 🔴/🟡 items → `investigation-cleanup` if the investigation folder still holds temporary artifacts; otherwise terminal step, no follow-up needed.

## Guardrails

1. Do not modify source code, tests, or configuration as part of the review.
2. Do not rely on the reference alone — always verify against the code.
3. Do not invent findings. If a concern is speculative, mark it as such and say what would confirm or refute it.
4. Keep the output focused on the axes the user named, plus the defaults above. Do not expand scope unbidden.
5. If the reference is missing, ambiguous, or cannot be located, say so and ask before proceeding.
6. Respond in the language the user wrote the invocation in.