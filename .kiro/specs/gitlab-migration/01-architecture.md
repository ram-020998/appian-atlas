# Appian Atlas GitLab Migration — Architecture

**Parent document:** [00-overview.md](00-overview.md)

---

## Current Architecture (GitHub-based)

### Repository Structure
```
GitHub (ram-020998)
├── appian-parser (private)
├── atlas-docs (public — GitHub Pages)
├── gam-knowledge-base
│   ├── data/                          # Parsed application data (JSON)
│   │   ├── ClauseAutomation/
│   │   └── SourceSelection/
│   └── mcp_server/                    # Python MCP server
├── power-appian-atlas
├── power-appian-atlas-developer
├── power-appian-atlas-product-owner
├── power-appian-atlas-ux-designer
└── power-appian-reference
```

### Current Data Flow
```
┌─────────────────────────────────────────────────────────────────┐
│                         User's IDE (Kiro)                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Appian Atlas Power (GitHub URL)                          │  │
│  │  - Provides steering instructions                         │  │
│  │  - Configures MCP server connection                       │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                  │
│                              ▼                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Knowledge Base MCP Server (Python)                       │  │
│  │  - Installed via: pipx install git+https://github.com/... │  │
│  │  - Runs locally on user's machine                         │  │
│  │  - Tools: list_applications, search_bundles, etc.         │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                  │
└──────────────────────────────┼──────────────────────────────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   GitHub API         │
                    │  (Public Internet)   │
                    └──────────────────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │  gam-knowledge-base  │
                    │  Repository (GitHub) │
                    │  - data/ folder      │
                    │  - JSON files        │
                    └──────────────────────┘
```

### Current Limitations
- **External dependency**: Relies on public GitHub infrastructure
- **Monorepo**: MCP server code and application data in the same repo despite different change cadences
- **Fragmented powers**: 5 separate repos for power configurations
- **Inconsistent deployment**: Different pattern from other Appian MCP servers
- **Security concerns**: Data hosted outside Appian's infrastructure
- **Token management**: Requires separate GitHub personal access tokens
- **No containerization**: Python package installation on user machines
- **No automated data refresh**: Data updated manually

---

## Proposed Architecture (GitLab-based)

### Repository Structure
```
GitLab (gitlab.appian-stratus.com/appian/atlas)
├── atlas-mcp-server                   # MCP server (Docker, CI/CD, registry)
│   ├── atlas_mcp/                     # Python MCP server package
│   │   ├── __init__.py
│   │   ├── config.py
│   │   ├── server.py
│   │   ├── client.py
│   │   ├── models.py
│   │   ├── datasource.py
│   │   ├── token_validator.py
│   │   ├── logging_config.py
│   │   ├── utils.py
│   │   └── tools/
│   ├── tests/
│   ├── docs/
│   ├── main.py
│   ├── Dockerfile
│   ├── Dockerfile.local
│   ├── docker-compose.yml
│   ├── Makefile
│   ├── .gitlab-ci.yml
│   └── mcp.json
│
├── solutions-knowledge-base           # Application data (JSON)
│   ├── data/
│   │   ├── ClauseAutomation/
│   │   └── SourceSelection/
│   ├── .gitlab-ci.yml                 # Daily scheduled pipeline
│   └── README.md
│
├── atlas-parser                       # Appian app parser
│   └── ...                            # Parser source code
│
└── atlas-kiro-powers                  # All Kiro powers (consolidated)
    ├── atlas/                         # Landing page power
    ├── atlas-developer/               # Developer persona
    ├── atlas-product-owner/           # Product Owner persona
    ├── atlas-ux-designer/             # UX Designer persona
    ├── appian-reference/              # SAIL language reference
    └── README.md
```

### Proposed Data Flow
```
┌─────────────────────────────────────────────────────────────────┐
│                         User's IDE (Kiro)                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Appian Atlas Power (from atlas-kiro-powers repo)         │  │
│  │  - Provides steering instructions                         │  │
│  │  - Configures MCP server connection                       │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                  │
│                              ▼                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Docker Container (GitLab Registry)                       │  │
│  │  registry.gitlab.appian-stratus.com/appian/atlas/         │  │
│  │  atlas-mcp-server:latest                                  │  │
│  │                                                           │  │
│  │  - Runs in isolated container                             │  │
│  │  - No local Python installation needed                    │  │
│  │  - Tools: list_applications, search_bundles, etc.         │  │
│  │  - Data fetched at runtime via GitLab API (NOT baked in)  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                  │
└──────────────────────────────┼──────────────────────────────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   GitLab API         │
                    │  (Internal Network)  │
                    └──────────────────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ solutions-knowledge- │
                    │ base (GitLab)        │
                    │  - data/ folder      │
                    │  - JSON files        │
                    │  - Updated daily by  │
                    │    scheduled pipeline│
                    └──────────────────────┘
```

### Data Pipeline Flow

> **Deferred:** Detailed data pipeline spec will be a separate implementation plan.

```
┌──────────────────────────────────────────────────────────────┐
│  solutions-knowledge-base — Daily Scheduled CI Pipeline      │
│                                                              │
│  1. Pulls atlas-parser (as pip dependency or Docker image)   │
│  2. Runs parser against Appian dev environments              │
│  3. Commits updated JSON files to own repo                   │
│                                                              │
│  No cross-repo write permissions needed.                     │
│  atlas-parser is consumed as a read-only dependency.         │
└──────────────────────────────────────────────────────────────┘
```

---

## Data Packaging Decision

**Decision:** Runtime fetch via GitLab API. Data is NOT baked into the Docker image.

**Rationale:**
- A daily pipeline in `solutions-knowledge-base` regenerates application data from dev environments
- If data were baked into the image, users would need to `docker pull` daily to get fresh data — bad UX and easy to forget, leading to stale results
- Runtime fetch means the server always gets the latest data from `solutions-knowledge-base`
- The LRU cache + pinned anchor files in the datasource handle performance
- Users only need to update their Docker image when the MCP server code itself changes (infrequent)

The clean separation of `atlas-mcp-server` (code) and `solutions-knowledge-base` (data) into distinct repos reinforces this architecture.

---

## API Mapping (GitHub → GitLab)

The current `GitHubDataSource` uses exactly two API operations. Both have direct 1:1 equivalents in the GitLab API — the same endpoints the existing GitLab MCP server uses internally.

| Operation | Current GitHub API | Proposed GitLab API |
|-----------|-------------------|---------------------|
| **Read file content** | `GET https://raw.githubusercontent.com/{owner}/{repo}/{branch}/{path}` | `GET /api/v4/projects/{id}/repository/files/{file_path}/raw?ref={branch}` |
| **List directory** | `GET https://api.github.com/repos/{owner}/{repo}/contents/{path}?ref={branch}` | `GET /api/v4/projects/{id}/repository/tree?path={path}&ref={branch}` |
| **Auth header** | `Authorization: token {GITHUB_TOKEN}` | `PRIVATE-TOKEN: {GITLAB_TOKEN}` |

The `{id}` in the GitLab API refers to the `solutions-knowledge-base` project ID. This is **hardcoded** in the server's `config.py` as `SOLUTIONS_KB_PROJECT_ID` (set once after the repo is created in GitLab). It is not passed via environment variables or Docker args — users only need `GITLAB_TOKEN`. An env var override (`ATLAS_DATA_PROJECT_ID`) exists for development/testing only.

The MCP tools (`list_applications`, `search_bundles`, etc.) are completely decoupled from the data-fetching layer — they consume JSON from the datasource and don't care whether it came from GitHub or GitLab. All caching logic (LRU cache, pinned anchor files) remains unchanged.

---

## Alignment with Existing Appian MCP Servers

### Side-by-Side Comparison

| Aspect | GitLab MCP Server | Jira MCP Server | Atlas MCP (Proposed) |
|--------|-------------------|-----------------|----------------------|
| **Repo** | `appian/prod/gitlab-mcp-server` | `appian/prod/jira-mcp-proxy` | `appian/atlas/atlas-mcp-server` |
| **Package** | `gitlab_mcp/` | `jira_mcp/` | `atlas_mcp/` |
| **Dockerfile** | Chainguard multi-stage | Chainguard multi-stage | Chainguard multi-stage |
| **CI/CD** | Shared templates + tox | Shared templates + tox | Shared templates + tox |
| **Token** | `GITLAB_TOKEN` | `JIRA_TOKEN` + `JIRA_EMAIL` | `GITLAB_TOKEN` (reused) |
| **Deployment** | Docker from registry | Docker from registry | Docker from registry |
| **Token validation** | Yes — read-only enforced | Yes | Yes — read-only enforced |
| **Logging** | File + stderr | File + stderr | File + stderr |

### User Configuration (All Three Servers)

```json
{
  "mcpServers": {
    "gitlab": {
      "command": "docker",
      "args": ["run", "--rm", "-i", "--env", "GITLAB_TOKEN",
               "registry.gitlab.appian-stratus.com/appian/prod/gitlab-mcp-server/gitlab-mcp-server:latest"],
      "env": {"GITLAB_TOKEN": "${GITLAB_TOKEN}"}
    },
    "jira-mcp-server": {
      "command": "docker",
      "args": ["run", "--rm", "-i", "--env", "JIRA_EMAIL", "--env", "JIRA_TOKEN", "--env", "JIRA_URL",
               "registry.gitlab.appian-stratus.com/appian/prod/jira-mcp-proxy/jira-mcp-proxy:latest"],
      "env": {
        "JIRA_URL": "https://appian-eng.atlassian.net",
        "JIRA_EMAIL": "${JIRA_EMAIL}",
        "JIRA_TOKEN": "${JIRA_TOKEN}"
      }
    },
    "appian-atlas": {
      "command": "docker",
      "args": ["run", "--rm", "-i", "--env", "GITLAB_TOKEN",
               "registry.gitlab.appian-stratus.com/appian/atlas/atlas-mcp-server/atlas-mcp-server:latest"],
      "env": {"GITLAB_TOKEN": "${GITLAB_TOKEN}"}
    }
  }
}
```

All three servers follow the same Docker-based deployment pattern. Users reuse their existing `GITLAB_TOKEN`.

---

## Current vs Proposed Comparison

| Aspect | Current (GitHub) | Proposed (GitLab) |
|--------|------------------|-------------------|
| **Repository Host** | GitHub (public) | GitLab (internal) |
| **Repo Structure** | 8 repos (monorepo for data+server, 5 separate power repos) | 4 repos (server, data, parser, powers) |
| **Installation** | `pipx install git+https://github.com/...` | Docker container from GitLab registry |
| **Deployment** | Python package on user machine | Docker container (isolated) |
| **Authentication** | GitHub personal access token | GitLab personal access token (same as other MCP servers) |
| **Data Access** | GitHub API (public internet) | GitLab API (internal network) |
| **Data Freshness** | Manual updates | Daily automated pipeline in `solutions-knowledge-base` |
| **Architecture** | Custom Python package | Follows GitLab MCP / Jira MCP pattern |
| **CI/CD** | None | GitLab CI/CD with shared templates |
| **Container Registry** | N/A | GitLab container registry |
| **Security** | External dependency | Internal infrastructure, read-only token enforcement |
| **Consistency** | Different from other Appian MCPs | Aligned with GitLab MCP / Jira MCP |

---

## Benefits

### Security
- **Internal infrastructure**: Data and code hosted within Appian's GitLab
- **Network isolation**: No external API calls to public GitHub
- **Consistent authentication**: Same GitLab tokens used across all Appian MCP servers
- **Token validation**: Read-only enforcement at startup (following GitLab MCP pattern)
- **Hardened container**: Chainguard base image, non-root user

### Maintainability
- **Clean repo separation**: Server code, data, parser, and powers each in their own repo with independent CI/CD
- **Standardized architecture**: Follows established patterns from GitLab MCP and Jira MCP
- **Containerization**: Isolated, reproducible environments
- **CI/CD integration**: Automated testing, linting, and deployment
- **Version control**: Easy rollback and version management

### User Experience
- **Simplified installation**: No Python environment setup required
- **Consistent deployment**: Same Docker pattern as other Appian MCP servers
- **Single token**: Reuse existing GitLab tokens
- **Always fresh data**: Runtime API fetch from `solutions-knowledge-base` means no stale data from old Docker images
- **Better reliability**: Container isolation prevents dependency conflicts

### Operational
- **4 focused repos**: Each with a single responsibility and appropriate CI/CD
- **Automated builds**: GitLab CI/CD handles building and publishing
- **Container registry**: Versioned Docker images in GitLab registry
- **Daily data pipeline**: Automated data refresh from dev environments in `solutions-knowledge-base`

---

## GitHub → GitLab Repository Mapping

| GitHub (Current) | GitLab (Proposed) | Notes |
|-------------------|-------------------|-------|
| `gam-knowledge-base` (data + server) | `atlas-mcp-server` (server only) | Server code extracted, renamed |
| `gam-knowledge-base` (data + server) | `solutions-knowledge-base` (data only) | Data extracted, renamed |
| `appian-parser` | `atlas-parser` | Renamed |
| `atlas-docs` | *(dropped)* | Not needed for GitLab |
| `power-appian-atlas` | `atlas-kiro-powers/atlas/` | Consolidated |
| `power-appian-atlas-developer` | `atlas-kiro-powers/atlas-developer/` | Consolidated |
| `power-appian-atlas-product-owner` | `atlas-kiro-powers/atlas-product-owner/` | Consolidated |
| `power-appian-atlas-ux-designer` | `atlas-kiro-powers/atlas-ux-designer/` | Consolidated |
| `power-appian-reference` | `atlas-kiro-powers/appian-reference/` | Consolidated |

---

## Tools (Unchanged)

All existing tools maintain their exact interfaces and functionality:

| Tool | Description |
|------|-------------|
| `list_applications` | Lists available applications |
| `get_app_overview` | Returns comprehensive app map |
| `search_bundles` | Finds bundles by keyword |
| `search_objects` | Searches parsed objects by name |
| `get_bundle` | Loads bundle at requested detail level |
| `get_dependencies` | Returns dependency subgraph |
| `get_object_detail` | Returns object info by UUID |
| `list_orphans` | Lists unreachable objects |
| `get_orphan` | Returns orphan detail with code |

**User Impact:** Zero — all queries and workflows remain identical.
