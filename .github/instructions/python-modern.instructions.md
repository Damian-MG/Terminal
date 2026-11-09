---
trigger: always_on
description: Python 3.12+ best practices, type hints, asyncio, testing, and security guidelines
---

# Python 3.12+ Development Standards

## Technology Stack

•⁠  ⁠*Runtime*: Python 3.12+
•⁠  ⁠*Package Manager*: UV (replaces pip/poetry)
•⁠  ⁠*Type Checking*: mypy (strict mode)
•⁠  ⁠*Linting/Formatting*: Ruff
•⁠  ⁠*Testing*: pytest with pytest-asyncio

## Commands

### File-Scoped (Preferred)
⁠ bash
uv run pytest tests/test_specific.py -v
uv run ruff check src/module.py
uv run mypy src/module.py
 ⁠

### Full Suite
⁠ bash
uv run pytest --cov=src
uv run ruff check .
uv run mypy .
 ⁠

## Code Style

•⁠  ⁠*Type hints*: Required on all functions (PEP 695 syntax)
•⁠  ⁠*Formatting*: Ruff (line length 100)
•⁠  ⁠*Imports*: Group stdlib, third-party, local
•⁠  ⁠*Docstrings*: Google style
•⁠  ⁠*Async*: Use ⁠ async ⁠/⁠ await ⁠ for all I/O

### Good Examples
⁠ python
type Vector = list[float]

async def fetch_data(url: str) -> dict[str, Any]:
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        return response.json()

async with asyncio.TaskGroup() as tg:
    tasks = [tg.create_task(fetch(url)) for url in urls]
 ⁠

### Avoid
⁠ python
def process(data):  # No type hints
    return data

try:
    risky()
except:  # Bare except
    pass

try:
    risky()
except Exception as e:
    raise
 ⁠

## Cognitive Complexity

Keep functions simple. SonarQube flags "Cognitive Complexity of functions should not be too high" as a code smell — *target ≤ 15 per function* (SonarQube default). Write code that reads top-to-bottom; do not nest logic that can be flattened or extracted.

### Avoid
•⁠  ⁠Nesting depth > 3 levels (each ⁠ if ⁠/⁠ for ⁠/⁠ while ⁠/⁠ try ⁠ inside another increases cognitive load)
•⁠  ⁠Long ⁠ if/elif/elif/... ⁠ chains branching on the same variable
•⁠  ⁠Nested loops with inner conditionals and ⁠ break ⁠/⁠ continue ⁠
•⁠  ⁠Boolean expressions combining more than 3 ⁠ and ⁠/⁠ or ⁠/⁠ not ⁠ operators — extract named sub-conditions
•⁠  ⁠Flag parameters (⁠ do_x: bool ⁠) that toggle distinct code paths inside one function — split into separate functions
•⁠  ⁠A single function doing validation + transformation + I/O + reporting

### Prefer
•⁠  ⁠*Guard clauses / early returns* to flatten nesting
•⁠  ⁠*Extract helpers* as soon as nesting reaches 3 levels or the function has more than one clear responsibility
•⁠  ⁠*Dict dispatch or polymorphism* instead of long ⁠ if/elif ⁠ chains
•⁠  ⁠*Named sub-expressions* (⁠ is_eligible = x and y and not z ⁠) instead of inline boolean soup

### Good
⁠ python
def process(order: Order) -> Result:
    if not order.is_valid:
        return Result.invalid()
    if order.is_draft:
        return Result.skipped()

    items = _prepare_items(order)
    total = _compute_total(items)
    return Result.ok(total)
 ⁠

### Avoid
⁠ python
def process(order):
    if order.is_valid:
        if not order.is_draft:
            items = []
            for line in order.lines:
                if line.qty > 0 and line.product.active:
                    items.append(line)
            total = 0
            for item in items:
                if item.discounted:
                    total += item.price * (1 - item.discount)
                else:
                    total += item.price
            return Result.ok(total)
        else:
            return Result.skipped()
    else:
        return Result.invalid()
 ⁠

## Testing

•⁠  ⁠Framework: pytest with pytest-asyncio
•⁠  ⁠Async: ⁠ @pytest.mark.asyncio ⁠ or ⁠ asyncio_mode = "auto" ⁠
•⁠  ⁠Coverage: 80%+ for new code

## Security

•⁠  ⁠Never hardcode secrets
•⁠  ⁠Use environment variables or AWS Secrets Manager
•⁠  ⁠Validate input with Pydantic
•⁠  ⁠Use parameterized queries only

## Permissions

### Allowed Without Prompting
•⁠  ⁠Read files, list directories
•⁠  ⁠Lint/format single files
•⁠  ⁠Run specific tests
•⁠  ⁠Type checking

### Require Approval
•⁠  ⁠⁠ uv add ⁠ (package installations)
•⁠  ⁠⁠ git push ⁠, ⁠ git commit ⁠
•⁠  ⁠File deletion
•⁠  ⁠Full test suites