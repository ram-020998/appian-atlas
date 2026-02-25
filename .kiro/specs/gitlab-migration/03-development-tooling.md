# Appian Atlas GitLab Migration — Development Tooling

**Parent document:** [00-overview.md](00-overview.md)
**Reference repo:** `gitlab-mcp-server` (`gitlab.appian-stratus.com/appian/prod/gitlab-mcp-server`)

This document covers all development infrastructure files for the `atlas-mcp-server` repository: build tooling, local development setup, ignore files, and test strategy.

---

## 1. `Makefile`

**Reference:** `gitlab-mcp-server/Makefile`

Single entry point for all development tasks. Targets:

```makefile
IMG ?= atlas-mcp-server:latest
REGISTRY ?= registry.gitlab.appian-stratus.com/appian/atlas/atlas-mcp-server

.PHONY: help
help: ## Display this help.
	@awk 'BEGIN {FS = ":.*##"; printf "\nUsage:\n  make \033[36m<target>\033[0m\n"} /^[a-zA-Z_0-9-]+:.*?##/ { printf "  \033[36m%-15s\033[0m %s\n", $$1, $$2 } /^##@/ { printf "\n\033[1m%s\033[0m\n", substr($$0, 5) } ' $(MAKEFILE_LIST)

##@ Development

.PHONY: test
test: ## Run tests
	python run_tests.py

.PHONY: lint
lint: ## Run linting
	flake8 --max-line-length=88 --extend-ignore=E203,W503 atlas_mcp/ main.py
	black --check atlas_mcp/ main.py
	isort --check-only atlas_mcp/ main.py

.PHONY: format
format: ## Format code
	black atlas_mcp/ main.py
	isort atlas_mcp/ main.py

.PHONY: install
install: ## Install all dependencies (prod + dev)
	pip install -r requirements-dev.txt

.PHONY: run
run: ## Run the server locally
	python main.py

##@ Build

.PHONY: docker-build
docker-build: ## Build docker image (production — requires internal ECR access)
	docker build -t ${IMG} .
	@if [ "${IMG}" != "atlas-mcp-server:latest" ]; then \
		echo "Tagging ${IMG} as atlas-mcp-server:latest"; \
		docker tag ${IMG} atlas-mcp-server:latest; \
	fi

.PHONY: docker-build-local
docker-build-local: ## Build docker image for local development (public base images)
	docker build -f Dockerfile.local -t ${IMG} .
	@if [ "${IMG}" != "atlas-mcp-server:latest" ]; then \
		echo "Tagging ${IMG} as atlas-mcp-server:latest"; \
		docker tag ${IMG} atlas-mcp-server:latest; \
	fi

.PHONY: docker-run-local
docker-run-local: ## Run docker container locally
	@echo "Make sure to set GITLAB_TOKEN environment variable"
	docker run --rm -i \
		--env GITLAB_TOKEN \
		${IMG}

.PHONY: docker-push
docker-push: ## Push docker image to registry
	docker push ${IMG}
```

---

## 2. `tox.ini`

**Reference:** `gitlab-mcp-server/tox.ini`

Orchestrates test environments. CI pipeline calls tox, not raw pytest.

> **Note:** Unlike the reference repo which inlines deps, we use `requirements-dev.txt` (which includes `requirements.txt` via `-r`) to stay consistent with the split dependency approach.

```ini
[tox]
envlist = py311
skipsdist = true

[testenv]
deps = -r requirements-dev.txt
setenv =
    PYTHONPATH = {toxinidir}
commands =
    pytest tests/ -v

[testenv:lint]
deps = -r requirements-dev.txt
setenv =
    PYTHONPATH = {toxinidir}
commands =
    flake8 atlas_mcp --max-line-length=120 --ignore=E203,W503,W293,W291,F401,F841,E128,E722,E501,W504

[testenv:typecheck]
deps = -r requirements-dev.txt
setenv =
    PYTHONPATH = {toxinidir}
commands =
    mypy atlas_mcp --ignore-missing-imports
```

---

## 3. `Dockerfile.local`

**Reference:** `gitlab-mcp-server/Dockerfile.local`

Simpler Dockerfile using public base images for local development. The production Dockerfile uses internal Chainguard images from ECR that may not be accessible outside CI.

```dockerfile
FROM python:3.11-slim

WORKDIR /app

RUN apt-get update && apt-get install -y \
    gcc \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

RUN mkdir -p /app/logs && \
    mkdir -p /root/.aws/amazonq

ENV PYTHONPATH=/app
ENV PYTHONUNBUFFERED=1

ENTRYPOINT ["python3", "main.py"]
```

---

## 4. `docker-compose.yml`

**Reference:** `gitlab-mcp-server/docker-compose.yml`

For local container testing. `stdin_open` and `tty` are required because MCP uses stdio transport.

```yaml
version: '3.8'

services:
  atlas-mcp-server:
    build: .
    image: atlas-mcp-server:local
    container_name: atlas-mcp-server-test
    environment:
      - GITLAB_TOKEN=${GITLAB_TOKEN}
    stdin_open: true
    tty: true
    restart: unless-stopped
```

---

## 5. `.dockerignore`

**Reference:** `gitlab-mcp-server/.dockerignore`

Excludes tests, docs, CI config, and IDE files from the Docker build context.

```
# Git
.git
.gitignore

# Python
__pycache__
*.pyc
*.pyo
*.pyd
.Python
env
pip-log.txt
pip-delete-this-directory.txt
.tox
.coverage
.coverage.*
.cache
nosetests.xml
coverage.xml
*.cover
*.log
.git
.mypy_cache
.pytest_cache
.hypothesis

# Virtual environments
venv/
env/
ENV/

# IDE
.vscode/
.idea/
*.swp
*.swo
*~

# OS
.DS_Store
.DS_Store?
._*
.Spotlight-V100
.Trashes
ehthumbs.db
Thumbs.db

# Documentation
*.md
docs/

# CI/CD
.gitlab-ci.yml
.github/

# Test files
tests/
test_*.py
*_test.py

# Development files
.env
.env.example
requirements-dev.txt
tox.ini
```

Note: No `data/` exclusion needed — `atlas-mcp-server` repo does not contain a data directory. Data lives in the separate `solutions-knowledge-base` repo and is fetched at runtime via GitLab API.

---

## 6. `.gitignore`

**Reference:** `gitlab-mcp-server/.gitignore`

```
# Python
__pycache__/
*.py[cod]
*.egg-info/

# Testing
.coverage
htmlcov/
.tox/
.pytest_cache/

# Virtual environments
venv/
.venv/

# IDE
.vscode/
.idea/

# OS
.DS_Store
Thumbs.db

# Logs
*.log

# Docker
.docker/
docker-compose.override.yml

# Environment files
.env
.env.local

# Kiro Steering docs
.kiro/
```

---

## 7. `.env.example`

Reference file documenting all environment variables. Not used by the application directly — serves as developer documentation.

> **Note:** The reference repo (`gitlab-mcp-server`) does not ship a `.env.example`. This is an improvement over the reference pattern for developer onboarding clarity.

```bash
# =============================================================================
# Appian Atlas MCP Server — Environment Variables
# =============================================================================
# Copy this file to .env and fill in the values for local development.
# Only GITLAB_TOKEN is required. All other variables have sensible defaults.
# =============================================================================

# --- Required ---

# GitLab personal access token (read-only scopes: read_api, read_repository)
# Get one at: GitLab → User Settings → Access Tokens
GITLAB_TOKEN=

# --- Optional (development/testing overrides) ---

# Override the hardcoded solutions-knowledge-base project ID.
# Only needed if testing against a different data repo.
# Default: hardcoded SOLUTIONS_KB_PROJECT_ID in config.py
# ATLAS_DATA_PROJECT_ID=

# Branch to read data from in solutions-knowledge-base.
# Default: main
# ATLAS_DATA_BRANCH=main

# Path prefix for data files in the data repo.
# Default: data
# ATLAS_DATA_PREFIX=data
```

---

## 8. `pytest.ini`

**Reference:** `gitlab-mcp-server/pytest.ini`

```ini
[tool:pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts =
    -v
    --tb=short
    --strict-markers
    --disable-warnings
    --cov=atlas_mcp
    --cov-report=term-missing
    --cov-report=html:htmlcov
    --cov-fail-under=80
asyncio_mode = auto
markers =
    asyncio: mark test as async
```

Key: `--cov-fail-under=80` enforces 80% minimum coverage.

---

## 9. `run_tests.py`

**Reference:** `gitlab-mcp-server/run_tests.py`

Standalone test runner script:

```python
#!/usr/bin/env python3
"""Test runner script for Appian Atlas MCP Server."""

import sys
import subprocess
from pathlib import Path


def run_tests(args=None):
    if args is None:
        args = []
    cmd = ["python", "-m", "pytest"]
    cmd.extend(args)
    try:
        result = subprocess.run(cmd, cwd=Path(__file__).parent)
        return result.returncode
    except KeyboardInterrupt:
        print("\nTests interrupted by user")
        return 1
    except Exception as e:
        print(f"Error running tests: {e}")
        return 1


def main():
    args = sys.argv[1:] if len(sys.argv) > 1 else []
    if "-h" in args or "--help" in args:
        print("Atlas MCP Server Test Runner")
        print()
        print("Usage: python run_tests.py [pytest-options]")
        print()
        print("Examples:")
        print("  python run_tests.py                    # Run all tests")
        print("  python run_tests.py -v                 # Verbose output")
        print("  python run_tests.py -k test_config     # Run tests matching pattern")
        print("  python run_tests.py tests/test_config.py  # Run specific test file")
        return 0
    return run_tests(args)


if __name__ == "__main__":
    sys.exit(main())
```

---

## 10. Test Strategy

### Test Structure

```
tests/
├── __init__.py
├── test_basic.py              # Import tests, basic sanity checks
├── test_config.py             # Config initialization, token resolution, error cases
├── test_client.py             # HTTP client, error handling, pagination
├── test_utils.py              # Utility functions
├── test_server.py             # MCP server setup, tool routing, error handling
├── test_token_validation.py   # Scope checking, forbidden scopes, fallback inference
├── test_datasource.py         # GitLab data source, caching, URL construction
├── test_application.py        # Application tool tests
├── test_bundle.py             # Bundle tool tests
├── test_object.py             # Object tool tests
└── test_orphan.py             # Orphan tool tests
```

### What Each Test File Covers

| File | Tests |
|------|-------|
| `test_basic.py` | Package imports, version metadata, module availability |
| `test_config.py` | Token from env var, from CLI arg, missing token error, initialization state, data project config |
| `test_client.py` | GET requests, error responses (401, 403, 404, 500), pagination, timeouts |
| `test_utils.py` | Text content creation, JSON formatting, param validation, path encoding |
| `test_server.py` | Tool registration, tool routing, unknown tool handling, error responses |
| `test_token_validation.py` | Allowed scopes pass, forbidden scopes rejected, scope inference, write permission detection |
| `test_datasource.py` | URL construction (pointing to solutions-knowledge-base), file fetching, directory listing, LRU cache behavior, pinned files |
| `test_application.py` | list_applications, get_app_overview with mock data |
| `test_bundle.py` | search_bundles, get_bundle with mock data |
| `test_object.py` | search_objects, get_dependencies, get_object_detail with mock data |
| `test_orphan.py` | list_orphans, get_orphan with mock data |

### Testing Approach
- Use `unittest.mock` / `pytest-mock` to mock HTTP calls — no real GitLab API calls in tests
- Use `pytest-asyncio` for async test support
- Coverage threshold: **80% minimum** (enforced by pytest.ini)
- CI reports coverage in **Cobertura format** for GitLab merge request integration
