---
trigger: always_on
description: UV package manager commands and best practices for Python dependency management
---

# UV Package Manager

UV is a fast Python package manager (10-100x faster than pip).

## Commands

### Environment Setup
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
uv venv
source .venv/bin/activate
uv sync
uv sync --group dev
```

### Dependency Management
```bash
uv add boto3                    # Add runtime dependency
uv add --group dev pytest       # Add dev dependency
uv add "fastapi>=0.110,<1"      # With version constraint
uv remove package-name          # Remove dependency
uv lock                         # Update lock file
uv sync                         # Install from lock
```

### Running Commands
```bash
uv run pytest tests/            # Run in venv
uv run python script.py         # Run script
uv run ruff check .             # Run tool
```

## pyproject.toml Structure

```toml
[project]
name = "my-project"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = ["pydantic>=2.0", "httpx>=0.25"]

[project.optional-dependencies]
dev = ["pytest>=8.0", "ruff>=0.3", "mypy>=1.8"]

[project.scripts]
my-cli = "my_project.cli:app"
```

## Best Practices

- Always commit `uv.lock` for reproducibility
- Use `uv sync` for consistent environments
- Specify version constraints for critical packages
- Use `--group dev` for development dependencies
- Prefer `uv run` over activating venv manually
