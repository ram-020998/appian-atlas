# Appian Atlas GitLab Migration — Technical Specification

**Parent document:** [00-overview.md](00-overview.md)
**Reference repo:** `gitlab-mcp-server` (`gitlab.appian-stratus.com/appian/prod/gitlab-mcp-server`)

This document specifies every file in the `atlas-mcp-server` repository, following the patterns established by the reference repo.

---

## 1. `main.py` — Entry Point

**Reference:** `gitlab-mcp-server/main.py`

Async entry point that orchestrates startup in this exact order:

1. Setup logging (before anything else)
2. Initialize configuration (reads `GITLAB_TOKEN`)
3. Validate token permissions (async — reject write-scoped tokens)
4. Create MCP server instance
5. Run stdio server

Must include:
- `asyncio.run(main())` entry point
- Detailed error handling with different messages for:
  - Token validation errors → print instructions for creating a read-only token
  - Configuration errors → print how to provide the token (env var, CLI arg)
  - General errors → print log file location
- All error messages go to `stderr`
- `finally` block for shutdown logging

```python
#!/usr/bin/env python3
"""
Appian Atlas MCP Server — Entry point

A Model Context Protocol server for exploring parsed Appian application
data stored in GitLab (solutions-knowledge-base repo).

Configuration:
    Token can be provided via:
    - Command argument: python3 main.py <token>
    - Environment variable: GITLAB_TOKEN

MCP Integration:
    Add to your mcp.json configuration file to integrate with Kiro / Amazon Q
"""

import asyncio
import sys
import logging
import mcp.server.stdio

from atlas_mcp.config import config
from atlas_mcp.server import AtlasMCPServer
from atlas_mcp.logging_config import setup_logging, log_startup_info, log_shutdown_info


async def main():
    """Main entry point for the Appian Atlas MCP server."""
    log_file_path = None

    try:
        log_file_path = setup_logging()
        log_startup_info()

        logger = logging.getLogger(__name__)
        logger.info("Starting Appian Atlas MCP Server...")

        config.initialize()
        logger.info(f"Configuration initialized. API URL: {config.get_api_url()}")

        logger.info("Validating GitLab token permissions...")
        await config.ensure_token_validated()
        logger.info("Token validation completed — read-only access confirmed")

        atlas_server = AtlasMCPServer()
        server = atlas_server.get_server()
        logger.info(f"Atlas MCP Server created with {len(atlas_server.tool_handlers)} tools")

        logger.info("Starting MCP stdio server...")
        async with mcp.server.stdio.stdio_server() as (read_stream, write_stream):
            from atlas_mcp.logging_config import log_mcp_server_ready, flush_logs
            log_mcp_server_ready()

            await server.run(
                read_stream,
                write_stream,
                server.create_initialization_options()
            )

    except ValueError as e:
        logger = logging.getLogger(__name__)
        logger.error(f"Configuration error: {e}")
        print(f"Configuration error: {e}", file=sys.stderr)

        if "Token validation failed" in str(e):
            print("", file=sys.stderr)
            print("Token validation failed. Please ensure your GitLab token has only read permissions.", file=sys.stderr)
            print("Acceptable token scopes: read_api, read_repository, read_user, read_registry", file=sys.stderr)
            print("", file=sys.stderr)
            print("To create a read-only token:", file=sys.stderr)
            print("1. Go to GitLab → User Settings → Access Tokens", file=sys.stderr)
            print("2. Create a new token with only 'read_api' and 'read_repository' scopes", file=sys.stderr)
            print("3. Set GITLAB_TOKEN environment variable with this token", file=sys.stderr)
        else:
            print("GitLab token must be provided via:", file=sys.stderr)
            print("  - Command argument: python3 main.py <token>", file=sys.stderr)
            print("  - Environment variable: GITLAB_TOKEN", file=sys.stderr)

        if log_file_path:
            print(f"\nDetailed logs available at: {log_file_path}", file=sys.stderr)

        log_shutdown_info()
        sys.exit(1)

    except Exception as e:
        logger = logging.getLogger(__name__)
        logger.error(f"MCP server error: {e}", exc_info=True)
        print(f"MCP server error: {e}", file=sys.stderr)
        if log_file_path:
            print(f"Log file location: {log_file_path}", file=sys.stderr)

        log_shutdown_info()
        sys.exit(1)

    finally:
        if log_file_path:
            log_shutdown_info()


if __name__ == "__main__":
    asyncio.run(main())
```

---

## 2. `atlas_mcp/__init__.py` — Package Metadata

```python
"""Appian Atlas MCP Server — A Model Context Protocol server for Appian application exploration."""

__version__ = "1.0.0"
__author__ = "Appian Atlas Team"
__description__ = ("A Model Context Protocol server for exploring parsed Appian "
                   "application data including bundles, objects, and dependencies.")
```

---

## 3. `atlas_mcp/config.py` — Configuration Management

**Reference:** `gitlab-mcp-server/gitlab_mcp/config.py`

Singleton configuration class that:
- Reads token from CLI args, then env var `GITLAB_TOKEN`
- Sets `PRIVATE-TOKEN` header
- Tracks initialization and validation state
- Delegates async token validation to `token_validator.py`
- Exposes `get_headers()`, `get_api_url()`, `is_initialized()`, `is_token_validated()`
- Provides `ensure_token_validated()` async method called from `main.py`
- Global instance: `config = AtlasConfig()`

Must also include the **data project configuration** since the Atlas server fetches data from the separate `solutions-knowledge-base` repo:

```python
"""Configuration management for Appian Atlas MCP server."""

import asyncio
import logging
import os
import sys
from typing import Dict, Optional

INTERNAL_GITLAB_API = "https://gitlab.appian-stratus.com/api/v4"

# Hardcoded project ID for the solutions-knowledge-base repository.
# This is the GitLab project ID assigned when the repo is created at
# gitlab.appian-stratus.com/appian/atlas/solutions-knowledge-base.
# It is hardcoded (not passed via env var or Docker args) because:
#   1. It is a fixed, known internal repo — it will not change.
#   2. It avoids requiring users to know or pass an internal project ID.
#   3. It keeps the mcp.json config simple (only GITLAB_TOKEN needed).
#   4. It matches how the GitLab MCP server hardcodes its own API base URL.
# To find this value: GitLab repo → Settings → General → Project ID.
# UPDATE THIS after creating the solutions-knowledge-base repo in GitLab.
SOLUTIONS_KB_PROJECT_ID = ""  # TODO: Set after repo creation (e.g., "12345")

logger = logging.getLogger(__name__)


class AtlasConfig:
    """Atlas MCP server configuration management."""

    def __init__(self):
        self.api_url: str = INTERNAL_GITLAB_API
        self.token: str = ""
        self.headers: Dict[str, str] = {}
        self.data_project_id: str = SOLUTIONS_KB_PROJECT_ID
        self.data_branch: str = "main"
        self.data_prefix: str = "data"
        self._initialized = False
        self._token_validated = False

    def initialize(self, token: Optional[str] = None) -> None:
        """Initialize configuration with GitLab token.

        Token resolution order: provided arg → CLI arg → GITLAB_TOKEN env var.

        Data project configuration:
        - data_project_id: Hardcoded to SOLUTIONS_KB_PROJECT_ID constant.
          Can be overridden via ATLAS_DATA_PROJECT_ID env var for testing.
        - ATLAS_DATA_BRANCH: Branch to read data from (default: main)
        - ATLAS_DATA_PREFIX: Path prefix for data files (default: data)
        """
        if token:
            self.token = token
        elif len(sys.argv) > 1:
            self.token = sys.argv[1]
        else:
            self.token = os.getenv("GITLAB_TOKEN", "")

        if not self.token:
            raise ValueError(
                "GitLab token must be provided via argument, command line, "
                "or GITLAB_TOKEN environment variable"
            )

        self.headers = {"PRIVATE-TOKEN": self.token}

        # Hardcoded by default; env var override for testing/development only
        self.data_project_id = os.getenv("ATLAS_DATA_PROJECT_ID", SOLUTIONS_KB_PROJECT_ID)
        if not self.data_project_id:
            raise ValueError(
                "Data project ID not configured. Set SOLUTIONS_KB_PROJECT_ID "
                "constant in config.py or ATLAS_DATA_PROJECT_ID env var."
            )

        self.data_branch = os.getenv("ATLAS_DATA_BRANCH", "main")
        self.data_prefix = os.getenv("ATLAS_DATA_PREFIX", "data")
        self._initialized = True

    async def _validate_token_async(self) -> None:
        if self._token_validated:
            return
        from .client import AtlasClient
        from .token_validator import validate_token, TokenValidationError
        try:
            client = AtlasClient()
            await validate_token(client)
            self._token_validated = True
        except TokenValidationError as e:
            raise ValueError(f"Token validation failed: {e}")

    async def ensure_token_validated(self) -> None:
        if not self._token_validated:
            await self._validate_token_async()

    def is_initialized(self) -> bool:
        return self._initialized

    def is_token_validated(self) -> bool:
        return self._token_validated

    def get_headers(self) -> Dict[str, str]:
        if not self._initialized:
            raise RuntimeError("Configuration not initialized. Call initialize() first.")
        return self.headers.copy()

    def get_api_url(self) -> str:
        if not self._initialized:
            raise RuntimeError("Configuration not initialized. Call initialize() first.")
        return self.api_url


config = AtlasConfig()
```

---

## 4. `atlas_mcp/client.py` — GitLab HTTP Client

**Reference:** `gitlab-mcp-server/gitlab_mcp/client.py`

Dedicated HTTP client class with:
- Persistent `requests.Session` for connection reuse
- `get(endpoint, params)` → JSON response
- `get_text(endpoint, params)` → raw text response
- `get_paginated(endpoint, params, per_page)` → follows `Link` header pagination
- Centralized `_make_request(method, endpoint, params, data)` with:
  - 30-second timeout
  - Error parsing from response JSON
  - `GitLabAPIError` exception with status code and response data
- `parse_link_header()` for pagination
- Global singleton: `client = AtlasClient()`

Follow the reference implementation exactly — the Atlas datasource will use `get()` for JSON and `get_text()` for raw file content from the `solutions-knowledge-base` repo.

---

## 5. `atlas_mcp/token_validator.py` — Read-Only Token Enforcement

**Reference:** `gitlab-mcp-server/gitlab_mcp/token_validator.py`

Validates at startup that the provided token has only read permissions:

- **Allowed scopes:** `read_api`, `read_repository`, `read_user`, `read_registry`
- **Forbidden scopes:** `api`, `write_repository`, `write_registry`, `sudo`, `admin_mode`, `create_runner`, `manage_runner`, `k8s_proxy`, `read_service_ping`
- Checks via `/personal_access_tokens/self` endpoint first
- Falls back to scope inference by testing API endpoints
- Falls back to write permission testing if no scopes detected
- Raises `TokenValidationError` with clear user-facing messages
- Convenience function: `validate_token(client)` called from `config.py`

Follow the reference implementation exactly.

---

## 6. `atlas_mcp/logging_config.py` — Structured Logging

**Reference:** `gitlab-mcp-server/gitlab_mcp/logging_config.py`

Provides:
- `setup_logging(log_level)` → configures file + stderr handlers, returns log file path
- Log file location: same directory as user's `mcp.json` (searches `~/.aws/amazonq`, `~/.config/mcp`, cwd)
- `find_mcp_config_directory()` → locates mcp.json directory
- `log_startup_info()` → Python version, working directory, PID, env vars (sanitized), token presence
- `log_shutdown_info()` → separator lines + flush
- `log_mcp_server_ready()` → "server is READY and accepting requests"
- `log_tool_call(tool_name, arguments)` → logs tool name + argument keys (not values)
- `flush_logs()` → force flush all handlers
- `get_log_file_path()` → returns current log file path

Log file name: `atlas-mcp-server.log`

Follow the reference implementation, substituting "GitLab MCP Server" with "Atlas MCP Server" in log messages.

---

## 7. `atlas_mcp/server.py` — MCP Server Setup

**Reference:** `gitlab-mcp-server/gitlab_mcp/server.py`

Class-based MCP server with tool routing:

```python
"""MCP server setup and tool routing for Appian Atlas MCP server."""

from typing import Any, Dict, List
import mcp.types as types
from mcp.server import Server

from .models import ToolSchemas
from .tools import ApplicationTools, BundleTools, ObjectTools, OrphanTools


class AtlasMCPServer:
    """Appian Atlas MCP Server with modular tool routing."""

    def __init__(self):
        self.server = Server("atlas-mcp-server")
        self._setup_handlers()

        self.tool_handlers = {
            # Application tools
            "list_applications": ApplicationTools.list_applications,
            "get_app_overview": ApplicationTools.get_app_overview,

            # Bundle tools
            "search_bundles": BundleTools.search_bundles,
            "get_bundle": BundleTools.get_bundle,

            # Object tools
            "search_objects": ObjectTools.search_objects,
            "get_dependencies": ObjectTools.get_dependencies,
            "get_object_detail": ObjectTools.get_object_detail,

            # Orphan tools
            "list_orphans": OrphanTools.list_orphans,
            "get_orphan": OrphanTools.get_orphan,
        }

    def _setup_handlers(self):
        @self.server.list_tools()
        async def handle_list_tools() -> List[types.Tool]:
            return ToolSchemas.get_all_tools()

        @self.server.call_tool()
        async def handle_call_tool(name: str, arguments: Dict[str, Any]) -> List[types.TextContent]:
            from .logging_config import log_tool_call
            log_tool_call(name, arguments)

            if name not in self.tool_handlers:
                error_result = {
                    "error": f"Unknown tool: {name}",
                    "available_tools": list(self.tool_handlers.keys())
                }
                return [types.TextContent(type="text", text=str(error_result))]

            try:
                handler = self.tool_handlers[name]
                result = await handler(arguments)

                import logging
                logger = logging.getLogger(__name__)
                logger.info(f"Tool '{name}' executed successfully")

                return result
            except Exception as e:
                import logging
                logger = logging.getLogger(__name__)
                logger.error(f"Tool '{name}' execution failed: {e}", exc_info=True)

                error_result = {
                    "error": f"Tool execution failed: {name}",
                    "details": str(e),
                    "arguments": arguments
                }
                return [types.TextContent(type="text", text=str(error_result))]

    def get_server(self) -> Server:
        return self.server
```

---

## 8. `atlas_mcp/models.py` — Data Models & Tool Schemas

**Reference:** `gitlab-mcp-server/gitlab_mcp/models.py`

Two responsibilities:

### Data Models
Dataclass models for Atlas entities:
- `Application` — app name, overview path, bundle count, object count
- `Bundle` — bundle name, type, objects list
- `AppObject` — object UUID, name, type, dependencies
- `Orphan` — orphan UUID, name, type, code

### Tool Schemas
`ToolSchemas` class with a `get_all_tools()` class method that returns `List[types.Tool]`. Each tool defined with:
- `name` — tool identifier
- `description` — what the tool does
- `inputSchema` — JSON Schema for parameters

All 9 tools must be defined here:
- `list_applications`, `get_app_overview`
- `search_bundles`, `get_bundle`
- `search_objects`, `get_dependencies`, `get_object_detail`
- `list_orphans`, `get_orphan`

Extract these from the current `server.py` inline definitions.

---

## 9. `atlas_mcp/datasource.py` — GitLab Data Source

Refactored from the current `GitHubDataSource` to `GitLabDataSource`. This is the core data-fetching layer. It reads data from the **`solutions-knowledge-base`** repo via GitLab API.

**What changes:**
- URL templates: GitHub raw/contents → GitLab files/raw and repository/tree
- Auth header: `Authorization: token` → `PRIVATE-TOKEN`
- Uses `AtlasClient` from `client.py` instead of inline HTTP calls
- Project ID comes from `config.data_project_id` (pointing to `solutions-knowledge-base`)

**What stays the same:**
- LRU cache with configurable `maxsize`
- Pinned anchor files (`app_overview.json`, `search_index.json`, `orphans/_index.json`)
- Application list caching
- All public methods and their signatures
- JSON parsing and error handling

```python
class GitLabDataSource:
    """Reads data from solutions-knowledge-base GitLab repository via API.

    Uses the same GitLab API endpoints that the existing GitLab MCP server
    relies on internally. All caching (LRU + pinned anchor files) is preserved
    from the GitHub implementation — only the HTTP URLs and auth header change.

    The data project (solutions-knowledge-base) is separate from this server's
    repo (atlas-mcp-server). The project ID is hardcoded in config.py as
    SOLUTIONS_KB_PROJECT_ID (set after repo creation). An env var override
    (ATLAS_DATA_PROJECT_ID) exists for development/testing only.
    """

    _PINNED_FILES = {'app_overview.json', 'search_index.json', 'orphans/_index.json'}

    def __init__(self, project_id: str, branch: str = "main",
                 data_prefix: str = "data", maxsize: int = 500):
        self._project_id = project_id
        self._branch = branch
        self._prefix = data_prefix
        self._maxsize = maxsize
        self._pinned: dict[str, dict | list] = {}
        self._cache: OrderedDict[str, dict | list] = OrderedDict()
        self._app_list: list[str] | None = None

    # Uses client.get() for JSON, client.get_text() for raw content
    # URL encoding via utils.encode_path() for file paths
```

---

## 10. `atlas_mcp/utils.py` — Utility Functions

**Reference:** `gitlab-mcp-server/gitlab_mcp/utils.py`

Utility functions needed by the Atlas server:

- `create_text_content(content)` → wraps string in `List[types.TextContent]`
- `format_json_response(data)` → JSON-serializes and wraps in TextContent
- `validate_required_params(params, required)` → raises ValueError for missing params
- `build_query_params(**kwargs)` → filters out None values
- `encode_path(path)` → URL-encode file paths for GitLab API
- `page_text(text, page, max_length)` → text pagination/truncation for large responses

---

## 11. `atlas_mcp/tools/__init__.py` — Tool Exports

```python
"""Tool implementations for Appian Atlas MCP server."""

from .application import ApplicationTools
from .bundle import BundleTools
from .object import ObjectTools
from .orphan import OrphanTools

__all__ = [
    'ApplicationTools',
    'BundleTools',
    'ObjectTools',
    'OrphanTools'
]
```

---

## 12. `atlas_mcp/tools/` — Tool Implementations

Each tool module contains a class with static async methods. Tools consume data from `GitLabDataSource` (which reads from `solutions-knowledge-base`) and return `List[types.TextContent]`.

### `application.py`
- `ApplicationTools.list_applications(arguments)` — lists available applications
- `ApplicationTools.get_app_overview(arguments)` — returns comprehensive app map

### `bundle.py`
- `BundleTools.search_bundles(arguments)` — finds bundles by keyword
- `BundleTools.get_bundle(arguments)` — loads bundle at requested detail level

### `object.py`
- `ObjectTools.search_objects(arguments)` — searches parsed objects by name
- `ObjectTools.get_dependencies(arguments)` — returns dependency subgraph
- `ObjectTools.get_object_detail(arguments)` — returns object info by UUID

### `orphan.py`
- `OrphanTools.list_orphans(arguments)` — lists unreachable objects
- `OrphanTools.get_orphan(arguments)` — returns orphan detail with code

These are refactored from the current monolithic `server.py`. The tool logic itself is unchanged — only the module organization changes.

---

## 13. `Dockerfile` — Production Container

**Reference:** `gitlab-mcp-server/Dockerfile`

Multi-stage build with Chainguard base image:

```dockerfile
# Stage 1: Build
ARG CI_DEPENDENCY_PROXY_GROUP_IMAGE_PREFIX
FROM ${CI_DEPENDENCY_PROXY_GROUP_IMAGE_PREFIX}python:3.11-slim as builder

WORKDIR /app

RUN apt-get update && apt-get install -y \
    gcc \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

COPY . .

# Stage 2: Runtime (Chainguard)
FROM 364133425616.dkr.ecr.us-east-1.amazonaws.com/chainguard/chainguard-base:20230214-202511120048

WORKDIR /app

RUN apk add --no-cache python3 py3-pip

RUN mkdir -p /app/logs && \
    mkdir -p /home/appuser/.local && \
    mkdir -p /home/appuser/.aws/amazonq

COPY --from=builder /app .

COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

RUN chown -R 65532:65532 /home/appuser && \
    chown -R 65532:65532 /app && \
    chown -R 65532:65532 /root/.local

RUN cp -r /root/.local /home/appuser/ && \
    chown -R 65532:65532 /home/appuser/.local

USER 65532:65532

ENV PATH=/home/appuser/.local/bin:$PATH
ENV PYTHONPATH=/app
ENV HOME=/home/appuser
ENV PYTHONUSERBASE=/home/appuser/.local

RUN python3 -c "import sys; print('Python path:', sys.path)" && \
    python3 -c "import mcp; print('MCP module found successfully')"

ENTRYPOINT ["python3", "main.py"]
```

Key points:
- `CI_DEPENDENCY_PROXY_GROUP_IMAGE_PREFIX` for GitLab dependency proxy
- Chainguard base image from internal ECR
- Non-root user (UID 65532)
- Module verification step
- No data directory — data fetched at runtime from `solutions-knowledge-base` via GitLab API

---

## 14. `.gitlab-ci.yml` — CI/CD Pipeline

**Reference:** `gitlab-mcp-server/.gitlab-ci.yml`

```yaml
include:
  - project: appian/prod/k8s-gitlab-runners
    file: /templates/.gitlab-ci.v1.yaml
  - component: $CI_SERVER_FQDN/appian/prod/stratus-pipeline-tools/stratus-service@stable
    inputs:
      image-sync-enabled: false
      image-signing-enabled: false
      enableCommercialStandard: false
      multiArchitecture: true

stages:
  - Lint
  - Test
  - Build
  - Tag
  - Security Services

workflow:
  rules:
    - if: '$CI_PIPELINE_SOURCE != "merge_request_event"'

default:
  id_tokens:
    STRATUS_JWT:
      aud: $CI_SERVER_URL

lint:
  extends:
    - .executor-small
  stage: Lint
  image: $CI_DEPENDENCY_PROXY_GROUP_IMAGE_PREFIX/python:3.11-slim
  tags:
    - k8s-executor
  before_script:
    - pip install tox
  script:
    - tox -e lint
  rules:
    - if: '$CI_COMMIT_TAG'
      when: never
    - when: on_success
  retry: 2

test:
  extends:
    - .executor-small
  stage: Test
  image: $CI_DEPENDENCY_PROXY_GROUP_IMAGE_PREFIX/python:3.11-slim
  tags:
    - k8s-executor
  before_script:
    - pip install tox
  script:
    - tox -e py311
  coverage: '/TOTAL.*\s+(\d+%)$/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
  rules:
    - if: '$CI_COMMIT_TAG'
      when: never
    - when: on_success
  retry: 2

tag-latest:
  extends:
    - .executor-small
  stage: Tag
  image: $CI_DEPENDENCY_PROXY_GROUP_IMAGE_PREFIX/alpine:latest
  tags:
    - k8s-executor
  before_script:
    - apk add --no-cache curl
    - curl -L "https://github.com/google/go-containerregistry/releases/latest/download/go-containerregistry_Linux_x86_64.tar.gz" | tar xz
    - mv crane /usr/local/bin/
    - chmod +x /usr/local/bin/crane
  script:
    - crane auth login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
    - crane copy "${REGISTRY_PATH}/${IMAGE_NAME}:${CI_COMMIT_SHA}" "${REGISTRY_PATH}/${IMAGE_NAME}:latest"
  rules:
    - if: '$CI_COMMIT_TAG'
      when: never
    - if: '$CI_PROJECT_PATH == "appian/atlas/atlas-mcp-server" && $CI_COMMIT_BRANCH == "main"'
      when: on_success
    - when: never
  retry: 2

variables:
  IMAGE_NAME: "atlas-mcp-server"
  IMAGE_TAG: "${CI_COMMIT_SHORT_SHA}"
  REGISTRY_PATH: "${CI_REGISTRY_IMAGE}"
  IMAGE_FULL_NAME: "${REGISTRY_PATH}/${IMAGE_NAME}:${IMAGE_TAG}"
  IMAGE_NAME_AND_TAG: "${IMAGE_NAME}:${IMAGE_TAG}"
  DOCKER_CONTEXT: "."
  DOCKER_FILE: "Dockerfile"
  CI_DEPENDENCY_PROXY_GROUP_IMAGE_PREFIX: "${CI_DEPENDENCY_PROXY_GROUP_IMAGE_PREFIX}"
```

---

## 15. `mcp.json` — Reference Configuration

Shipped at repo root for easy user setup.

The `solutions-knowledge-base` project ID is **hardcoded** in `config.py` (as `SOLUTIONS_KB_PROJECT_ID`), so it does NOT need to be passed via Docker args or environment variables. Users only need to provide `GITLAB_TOKEN`. This keeps the configuration identical in simplicity to the GitLab MCP server — one env var, one Docker image.

```json
{
  "mcpServers": {
    "appian-atlas": {
      "command": "docker",
      "args": [
        "run",
        "--rm",
        "-i",
        "--env",
        "GITLAB_TOKEN",
        "registry.gitlab.appian-stratus.com/appian/atlas/atlas-mcp-server/atlas-mcp-server:latest"
      ],
      "env": {
        "GITLAB_TOKEN": "YOUR_GITLAB_TOKEN_HERE"
      }
    }
  }
}
```

> **Why no `ATLAS_DATA_PROJECT_ID` in the config?** The project ID for `solutions-knowledge-base` is a fixed internal value that never changes. Hardcoding it in the server code (like the GitLab MCP server hardcodes its API base URL) avoids exposing an internal implementation detail to users and keeps the setup to a single env var. An env var override (`ATLAS_DATA_PROJECT_ID`) exists for development/testing only.

---

## 16. `requirements.txt` (Production)

Production dependencies only. This is what the Dockerfile installs.

```
requests>=2.31.0
mcp
```

## 16b. `requirements-dev.txt` (Development & Testing)

Includes production deps plus test/lint tooling. Used by developers locally and by `tox.ini`.

```
-r requirements.txt
pytest>=7.0.0
pytest-asyncio>=0.21.0
pytest-cov>=4.0.0
flake8>=6.0.0
black>=23.0.0
isort>=5.12.0
mypy>=1.0.0
types-requests
```

> **Note:** This deviates from the reference repo (`gitlab-mcp-server`) which keeps all deps in a single `requirements.txt`. We split for a cleaner production image — test dependencies are not installed in the Docker container.

---

## 17. `docs/` Directory

### `docs/SECURITY.md`
- Token validation approach and allowed scopes
- Read-only enforcement rationale
- How to create a properly scoped GitLab token
- Container security (Chainguard, non-root user)

### `docs/USAGE_EXAMPLES.md`
- Example queries for each tool
- Common workflows (exploring an app, finding dependencies, identifying orphans)
- Troubleshooting common issues
