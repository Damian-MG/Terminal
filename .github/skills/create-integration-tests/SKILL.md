---
name: create-integration-tests
description: Design and create functional tests (integration + E2E with variants) that run the real application against real AWS services. Use when the user needs functional/integration test artifacts under `task/integration_test/` to validate a feature against the DEV account (no mocks, real S3/Bedrock/DynamoDB/etc.).
argument-hint: "@<context-doc> (investigation/plan/PRD)"
allowed-tools:
  - read
  - grep
  - glob
  - exec
  - write
  - edit
---

# Create Integration Tests

Design and generate functional tests that **run the real application against real AWS services** in a controlled environment. No mocks, no simulations — the application executes for real with real resources (S3 buckets, Bedrock jobs, DynamoDB tables, etc.). If a test needs a file in S3, the file gets created in S3. If a test needs to invoke Bedrock, Bedrock gets invoked.

Tests cover two levels:

- **Integration tests**: validate specific components or service interactions in isolation (e.g., "can we invoke Bedrock with the new model?", "does the S3 upload handle metadata correctly?").
- **E2E tests with variants**: run the full application flow end-to-end with different configurations, data sets, and conditions to validate the complete behavior under multiple scenarios.

This skill requires thorough investigation: reading documentation, exploring how the application is configured, and understanding what was developed on the current branch to know exactly how to launch and validate the application.

All artifacts are created inside `task/integration_test/` — the active development directory. When the development is complete, `task/` gets copied to `tasks/{branch-name}/` as historical archive.

## Usage

Create integration tests using the investigation doc as context:

```
create-integration-tests @docs/investigations/{feature}/README.md
```

Create using a plan or any other context document:

```
create-integration-tests @task/PLAN_{feature}.md
```

## Prerequisites

- Active development branch with code changes (`git diff main --stat` shows changes).
- Context document describing the feature (investigation, plan, PRD, etc.).
- Existing test examples in `tasks/` for reference patterns.

**CRITICAL**: Tests run against the **DEV account**. If you are unsure which AWS account or profile to use, **ask the user**. Never guess or invent credentials, accounts, or profiles.

## Steps

### 1. Understand the Development

Read the context document provided by the user to extract:

- **What was built**: new methods, classes, APIs, configuration parameters.
- **What changed**: rewired flows, new error handling, new dependencies.
- **What to validate**: the specific behaviors that need functional testing.

Then inspect the actual code changes on the branch:

```bash
# Files changed vs main
git diff main --stat

# Detailed diff for understanding the implementation
git diff main -- src/
```

Read each changed source file to understand:

- Method signatures, parameters, return types.
- Error handling paths (exceptions thrown/caught).
- Configuration parameters and their defaults/ranges.
- External service interactions (API calls, S3 operations).

**Understand how to launch the application:**

- Read project documentation (`README.md`, `AGENTS.md`, `CONTRIBUTING.md`).
- Read `env/local.yaml` and configuration files to understand how the app is configured.
- Identify the entry point(s): server mode (`run_server.py`), batch mode (`batch_main.py`), CLI, etc.
- Identify which configuration parameters control the behavior being tested.
- Understand the AWS resources involved (S3 buckets, prefixes, Bedrock models, DynamoDB tables).

**CRITICAL**: do not invent assumptions about the feature or how the application works. If the code or documentation is ambiguous, **ask the user** for clarification before proceeding.

### 2. Study Existing Integration Test Patterns

Explore `tasks/` to find previous integration test examples:

```bash
# Find integration_test directories in tasks/
find tasks/ -type d -name "integration_test"

# Find test scripts and procedures
find tasks/ -name "TESTING_PROCEDURE*" -o -name "MANUAL_TEST_PLAN*" -o -name "setup_test*" -o -name "run_*test*"
```

Read the existing examples to extract:

- **Directory structure**: what files they include (procedures, scripts, configs, results templates).
- **Setup pattern**: how they provision test data in S3 (shell scripts, Python async scripts).
- **Test execution pattern**: how they run tests (automated scripts, project entry points, manual steps).
- **Verification pattern**: how they validate results (log grep, S3 checks, output counts).
- **Configuration pattern**: what goes in `local.yaml` for the test.
- **Results documentation**: how they track and report results.

**Reuse existing components whenever possible** — do not duplicate what already works.

### 3. Create the Integration Test Directory

Create `task/integration_test/` with this structure:

```
task/integration_test/
  TESTING_PROCEDURE.md        # Step-by-step manual testing guide
  TESTING_RESULTS.md          # Template for documenting results
  local.yaml.example          # Configuration template for the test
  scripts/
    setup_test_batch.sh       # S3 test data provisioning (shell)
    setup_test_files.py       # S3 test data provisioning (Python async)
    run_integration_test.py   # Automated integration test script
```

Each file has a specific purpose and format described below.

### 4. Create TESTING_PROCEDURE.md

Write a step-by-step manual testing guide covering:

1. **Prerequisites** — credentials, environment, permissions (especially NEW permissions required by the feature).
2. **Setup de archivos** — how to copy test data to S3 (reference the setup scripts).
3. **Configuration** — what to put in `env/local.yaml`, with profiles (conservador/normal/agresivo).
4. **Test scenarios** — one section per scenario, each with:
   - Objective (what it validates).
   - Setup (specific config changes, test data needed).
   - Execution command.
   - Expected results (specific log messages to look for).
   - Validation commands (grep patterns for pass/fail).
5. **Monitoring** — what to watch during execution.
6. **Verification** — post-execution checks.
7. **Cleanup** — how to clean test artifacts.
8. **Troubleshooting** — common issues and solutions.
9. **Quick reference** — copy-paste command summary.

**Scenario design guidelines:**

**Integration tests** (targeted, fast):

- **Permission/access** — verify new API permissions work.
- **Service interaction** — validate individual AWS calls (S3 read/write, Bedrock invoke, DynamoDB query).
- **Configuration** — verify the app reads and applies new parameters correctly.
- **Error handling** — force specific errors and verify graceful degradation.

**E2E tests with variants** (full flow, slower):

- **Happy path — single item**: one input through the complete flow (baseline).
- **Happy path — multiple items**: several inputs to test concurrency and ordering.
- **Different data variants**: different file types, sizes, or content that exercise different code paths.
- **Different config variants**: same input but with different configuration profiles (e.g., conservative vs aggressive timeouts, small vs large batch sizes).
- **Failure variants**: force different failure modes (timeout, service error, invalid input, missing resource) and verify recovery or graceful failure.
- **Behavioral variants**: verify specific patterns (backoff progression, retry behavior, status transitions) under different conditions.
- **Scale variants**: test with varying volumes (1, 10, 50 items) to validate behavior at different loads.

Include a **before vs after comparison table** showing what changed from old behavior to new.

### 5. Create Setup Scripts

#### `setup_test_batch.sh` (Shell)

Following the pattern from existing tests:

- Accept flags (`--full` for more files, `--clean-only` for cleanup).
- Define source and destination S3 prefixes.
- Clean destination before copying.
- Copy test files + their metadata from production-like source.
- Remove stale batch state files.
- Verify and report what was copied.
- Print next-steps instructions.

Use the same S3 bucket, AWS profile, and source paths as existing tests in `tasks/`.

#### `setup_test_files.py` (Python)

Following the pattern from existing test setup scripts in `tasks/`:

- Use the project's async S3 utilities (check existing scripts for import paths).
- Accept `--full` flag via argparse.
- Clean destination prefix.
- Copy each file + metadata with error handling.
- Verify and report results.
- Non-zero exit code on failure.

### 6. Create `run_integration_test.py`

Design the automated test script with:

- **Individual test functions** — each returning a pass/fail result with details.
- **Test selection** — `--test {name}` flag to run specific tests, `--test all` for everything.
- **Variant support** — `--variant {name}` flag to run a specific E2E variant (e.g., `--variant conservative`, `--variant high-volume`).
- **Pre-flight checks** — verify test files exist in S3 before running.
- **Output cleanup** — remove previous test outputs before running.
- **Result tracking** — collect test name, variant, pass/fail, duration, error details.
- **Summary report** — print pass/fail table at the end grouped by test type (integration vs E2E).
- **Non-zero exit** on any failure.

**Test design principles:**

- Each test is self-contained (own setup, own assertions).
- Tests run sequentially (avoid AWS throttling).
- Integration tests first (fast, targeted checks).
- E2E tests after (full flow, slower, with variants).
- Each E2E variant changes **one dimension** at a time (data, config, scale, or failure mode) so failures are easy to diagnose.
- Monkey-patching for behavioral validation (e.g., intercept `asyncio.sleep` to track intervals).
- Graceful handling of inconclusive results (e.g., job finishes faster than expected timeout).

### 7. Create `local.yaml.example`

Create a complete configuration template that:

- Contains ALL settings needed to run the tests.
- Points S3 prefix settings to the test data prefix (not production).
- Has the new configuration parameters introduced by the feature (commented with explanations).
- **Includes named variant sections** — different config profiles for each E2E variant (e.g., `# Variant: conservative` with low timeouts, `# Variant: high-volume` with large batch size).
- Uses the same base config as the project's `env/local.yaml` (copy from it).

### 8. Create TESTING_RESULTS.md

Create a results template with:

- **Header**: branch, date, tester, environment.
- **Executive summary**: total tests, passed, failed, verdict.
- **Pre-flight checks table**: credentials, files, config, new permissions.
- **Integration test results**: one section per integration test with pass/fail.
- **E2E variant results**: one section per variant with checkboxes for each validation point.
- **Variant comparison table**: results across all E2E variants side by side (which passed, which failed, timing differences).
- **Before vs after comparison table**: what changed from the old behavior to the new.
- **Issues found table**: severity, description, variant affected, status.
- **Recommendations section**.
- **Log file references**.

### 9. Validate the Generated Artifacts

After creating all files, verify:

```bash
# Python syntax check
python -c "import py_compile; py_compile.compile('task/integration_test/scripts/run_integration_test.py', doraise=True)"
python -c "import py_compile; py_compile.compile('task/integration_test/scripts/setup_test_files.py', doraise=True)"

# Shell syntax check
bash -n task/integration_test/scripts/setup_test_batch.sh

# Verify directory structure
find task/integration_test -type f | sort
```

Fix any syntax errors before reporting completion.

## Constraints

- **Real environment, no mocks**: tests run the real application against real AWS services. No mocking, no simulations. If a resource is needed, create it for real.
- **Never guess**: if unsure about accounts, profiles, credentials, or feature behavior — **ask the user**.
- **DEV environment only**: tests target the DEV account. Never run against PRE or PRO.
- **Create in `task/integration_test/`** — never in `tasks/` (that's the archive).
- **Follow existing patterns** — match the style and structure from `tasks/*/integration_test/`.
- **Use project utilities** — reuse existing S3 helpers, AWS profiles, and shared code from the project.
- **Same test data sources** — use the same production-like S3 source paths as existing tests.
- **No execution** — this skill creates the test artifacts, it does not execute them.
- **No changes to source code** — only create files inside `task/integration_test/`.
- **No changes to `env/local.yaml`** — only create a `.example` template.
- **Cognitive Complexity <= 15 per function** (SonarQube default). Test setup, execution, and assertion helpers are code too — keep them flat. Avoid nested `if`/`for` with inline assertions, long `if/elif` chains dispatching on test variants, and flag-parameter helpers. Prefer guard clauses, one assertion helper per concern, and parametrized tests over branching inside a single test function.

## Output Guidelines

**DO:**

- Read ALL changed source files to understand what needs testing.
- Read existing integration tests in `tasks/` to match patterns exactly.
- Ask the user when unsure about accounts, profiles, or feature behavior.
- Create scripts that work with `uv run` from the project root.
- Use the same AWS profile, bucket, and region as existing tests.
- Include both shell and Python setup options.
- Design tests that validate the NEW behavior specifically.
- Include a before-vs-after comparison in the procedure and results template.

**DON'T:**

- Invent or guess AWS accounts, profiles, or credentials.
- Create tests without understanding what is being developed.
- Create tests that only repeat what unit tests already cover.
- Hardcode secrets or credentials in scripts (use AWS profiles).
- Create tests that modify production S3 prefixes.
- Skip syntax validation of generated scripts.
- Create overly complex test infrastructure — keep it simple.
- Duplicate existing test scripts or utilities.

## Next Step (REQUIRED final output)

**You MUST end your response with the block below.** This is not documentation — it is the literal closing message of every `create-integration-tests` invocation. Skipping it is a failure of the skill.

Print exactly:

```
**Terminal step.** This skill does not have a canonical successor yet; how it fits into the development cycle (subordinate to `execute-plan`, branch of `implementation-review`, or independent) is unresolved by design.

Options if needed:

- Manually run the generated scripts under `task/integration_test/` to validate the new behavior.
- Resume the cycle that triggered the test creation (typically the `execute-plan` or `implementation-review` flow that requested the tests).
```