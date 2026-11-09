---
name: uv-package-manager
description: UV package manager for Python projects including project initialization, dependency management, virtual environments, and pyproject.toml configuration. Use when setting up Python projects or managing dependencies.
metadata:
  author: me
  version: "1.0"
---

# UV Package Manager

UV is a fast Python package manager written in Rust (10-100x faster than pip).

## Project Initialization

### Create New Project
```bash
# Create new project
uv init my-project
cd my-project

# Or initialize in current directory
mkdir my-project && cd my-project
uv init
```

### Generated Structure
```
my-project/
├── .gitignore
├── .python-version      # Python version (e.g., "3.12")
├── README.md
├── main.py
└── pyproject.toml
```

### Specify Python Version
```bash
# Create with specific Python version
uv init --python 3.12

# Change Python version
echo "3.12" > .python-version
uv sync
```

## Dependency Management

### Add Dependencies
```bash
# Add production dependency
uv add requests
uv add "fastapi>=0.100"
uv add httpx aiofiles

# Add development dependency
uv add --dev pytest pytest-asyncio ruff mypy

# Add optional dependency group
uv add --group docs mkdocs mkdocs-material

# Add from git
uv add git+https://github.com/user/repo
uv add git+https://github.com/user/repo@v1.0.0

# Add from local path
uv add ./packages/my-lib
```

### Remove Dependencies
```bash
uv remove requests
uv remove --dev pytest
```

### Update Dependencies
```bash
# Update specific package
uv lock --upgrade-package requests

# Update all packages
uv lock --upgrade

# Sync environment with lockfile
uv sync
```

### View Dependencies
```bash
# Show installed packages
uv pip list

# Show dependency tree
uv pip tree

# Show project version
uv version
```

## pyproject.toml Configuration

### Basic Configuration
```toml
[project]
name = "my-project"
version = "0.1.0"
description = "My awesome project"
readme = "README.md"
requires-python = ">=3.12"
license = { text = "MIT" }
authors = [
    { name = "Your Name", email = "you@example.com" }
]
keywords = ["python", "example"]
classifiers = [
    "Development Status :: 3 - Alpha",
    "Programming Language :: Python :: 3.12",
]
dependencies = [
    "fastapi>=0.100",
    "httpx>=0.25",
    "pydantic>=2.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
    "ruff>=0.3",
    "mypy>=1.8",
]
docs = [
    "mkdocs>=1.5",
    "mkdocs-material>=9.0",
]

[project.scripts]
my-cli = "my_project.cli:main"

[project.urls]
Homepage = "https://github.com/user/my-project"
Documentation = "https://my-project.readthedocs.io"
Repository = "https://github.com/user/my-project"
```

### UV-Specific Configuration
```toml
[tool.uv]
# Use specific Python version
python = "3.12"

# Dev dependencies (alternative to optional-dependencies)
dev-dependencies = [
    "pytest>=8.0",
    "ruff>=0.3",
]

# Index configuration
[[tool.uv.index]]
name = "pytorch"
url = "https://download.pytorch.org/whl/cpu"
explicit = true

# Package sources
[tool.uv.sources]
my-package = { git = "https://github.com/user/my-package", tag = "v1.0.0" }
local-lib = { path = "./libs/local-lib", editable = true }
```

### Build System
```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/my_project"]
```

## Running Commands

### Run Scripts
```bash
# Run Python file
uv run main.py

# Run module
uv run -m pytest

# Run with specific Python
uv run --python 3.12 main.py

# Run CLI entry point
uv run my-cli --help
```

### Run in Environment
```bash
# Activate virtual environment (optional, uv run handles this)
source .venv/bin/activate

# Or use uv run for any command
uv run python -c "import sys; print(sys.version)"
```

## Virtual Environment

### Location
```
.venv/
├── bin/           # Scripts (Linux/Mac)
├── lib/           # Installed packages
└── pyvenv.cfg     # Environment config
```

### Manual Management
```bash
# Create venv manually (usually automatic)
uv venv

# Create with specific Python
uv venv --python 3.12

# Sync dependencies to venv
uv sync

# Sync including dev dependencies
uv sync --dev

# Sync specific group
uv sync --group docs
```

## Lockfile (uv.lock)

- Cross-platform lockfile
- Human-readable TOML format
- Should be committed to version control
- Ensures reproducible builds

```bash
# Generate/update lockfile
uv lock

# Install from lockfile exactly
uv sync --frozen
```

## Migration from pip/poetry

### From requirements.txt
```bash
# Add all dependencies from requirements.txt
uv add -r requirements.txt

# With constraints
uv add -r requirements.txt -c constraints.txt
```

### From Poetry
```bash
# UV can read pyproject.toml from Poetry
# Just run:
uv sync
```

## Building & Publishing

### Build Distribution
```bash
# Build wheel and sdist
uv build

# Output in dist/
# dist/my_project-0.1.0-py3-none-any.whl
# dist/my_project-0.1.0.tar.gz
```

### Publish to PyPI
```bash
# Set PyPI token
export UV_PUBLISH_TOKEN=pypi-xxxxx

# Publish
uv publish
```

## Common Workflows

### New Project Setup
```bash
uv init my-api
cd my-api
uv add fastapi uvicorn
uv add --dev pytest ruff mypy
uv run uvicorn main:app --reload
```

### CI/CD
```yaml
# GitHub Actions
- uses: astral-sh/setup-uv@v1
- run: uv sync --frozen
- run: uv run pytest
- run: uv run ruff check .
```

### Docker
```dockerfile
FROM python:3.12-slim
COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/uv

WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

COPY . .
CMD ["uv", "run", "uvicorn", "main:app", "--host", "0.0.0.0"]
```

See [references/pyproject-template.toml](references/pyproject-template.toml) for complete template.
