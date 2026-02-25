# Appian Atlas GitLab Migration Architecture

> ⚠️ **SUPERSEDED** — This document (v1.1) has been replaced by the multi-document spec in `gitlab-migration/` directory. See [00-overview.md](gitlab-migration/00-overview.md) for the current plan. This file is retained for historical reference only.

**Document Version:** 1.1  
**Date:** February 24, 2026  
**Status:** ~~Proposal - Pending Approval~~ **SUPERSEDED by v2.1**  
**Changelog:** v1.1 — Added API endpoint mapping analysis, datasource refactor risk assessment, CI/CD fixes, Dockerfile pinning, fallback strategy, timeline adjustment.

---

## Executive Summary

This document proposes migrating the Appian Atlas knowledge base infrastructure from GitHub to GitLab, following the established patterns used by Appian's existing GitLab MCP and Jira MCP servers. This migration will:

- Consolidate all Appian Atlas repositories within Appian's internal GitLab instance
- Standardize deployment using Docker containers hosted in GitLab's container registry
- Maintain all existing functionality while improving security and maintainability
- Align with Appian's internal tooling ecosystem

---

## Current Architecture (GitHub-based)

### Repository Structure
```
GitHub (ram-020998)
├── appian-parser (private)
├── atlas-docs (public - GitHub Pages)
├── gam-knowledge-base
│   ├── data/                          # Parsed application data (JSON)
│   │   ├── CaseManagementStudio/
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
- **Inconsistent deployment**: Different pattern from other Appian MCP servers
- **Security concerns**: Data hosted outside Appian's infrastructure
- **Token management**: Requires separate GitHub personal access tokens
- **No containerization**: Python package installation on user machines

---

## Proposed Architecture (GitLab-based)

### Repository Structure
```
GitLab (gitlab.appian-stratus.com/appian/atlas)
├── appian-parser
├── atlas-docs
├── gam-knowledge-base
│   ├── data/                          # Parsed application data (JSON)
│   │   ├── CaseManagementStudio/
│   │   └── SourceSelection/
│   ├── mcp_server/                    # Python MCP server
│   │   ├── main.py                    # Entry point
│   │   ├── config.py                  # Configuration management
│   │   ├── server.py                  # MCP server setup
│   │   ├── datasource.py              # GitLab data source
│   │   └── tools/                     # Tool implementations
│   ├── Dockerfile                     # Container definition
│   └── .gitlab-ci.yml                 # CI/CD pipeline
├── power-appian-atlas
├── power-appian-atlas-developer
├── power-appian-atlas-product-owner
├── power-appian-atlas-ux-designer
└── power-appian-reference
```

### Proposed Data Flow
```
┌─────────────────────────────────────────────────────────────────┐
│                         User's IDE (Kiro)                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Appian Atlas Power (GitLab URL)                          │  │
│  │  - Provides steering instructions                         │  │
│  │  - Configures MCP server connection                       │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                  │
│                              ▼                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Docker Container (GitLab Registry)                       │  │
│  │  registry.gitlab.appian-stratus.com/appian/atlas/         │  │
│  │  gam-knowledge-base:latest                                │  │
│  │                                                           │  │
│  │  - Runs in isolated container                             │  │
│  │  - No local Python installation needed                    │  │
│  │  - Tools: list_applications, search_bundles, etc.         │  │
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
                    │  gam-knowledge-base  │
                    │  Repository (GitLab) │
                    │  - data/ folder      │
                    │  - JSON files        │
                    └──────────────────────┘
```

---

## Alignment with Existing Appian MCP Servers

### GitLab MCP Server (Existing)

**Repository:** `gitlab.appian-stratus.com/appian/prod/gitlab-mcp-server`

**Architecture:**
```
gitlab-mcp-server/
├── main.py                    # Entry point
├── gitlab_mcp/
│   ├── config.py              # Configuration management
│   ├── server.py              # MCP server setup
│   ├── client.py              # GitLab API client
│   ├── token_validator.py     # Security validation
│   └── tools/                 # Tool implementations
│       ├── repository.py
│       ├── merge_request.py
│       ├── pipeline.py
│       └── search.py
├── Dockerfile
└── .gitlab-ci.yml
```

**Deployment:**
```json
{
  "mcpServers": {
    "gitlab": {
      "command": "docker",
      "args": [
        "run", "--rm", "-i",
        "--env", "GITLAB_TOKEN",
        "registry.gitlab.appian-stratus.com/appian/prod/gitlab-mcp-server/gitlab-mcp-server:latest"
      ],
      "env": {
        "GITLAB_TOKEN": "${GITLAB_TOKEN}"
      }
    }
  }
}
```

**Tools Provided:**
- `list_repositories`, `list_branches`, `list_commits`
- `get_merge_request`, `list_merge_requests`
- `get_pipeline`, `list_pipelines`, `get_job_log`
- `search_projects`, `search_code`, `search_files`

---

### Jira MCP Server (Existing)

**Repository:** `gitlab.appian-stratus.com/appian/prod/jira-mcp-proxy`

**Architecture:**
```
jira-mcp-proxy/
├── main.py
├── jira_mcp/
│   ├── config.py
│   ├── server.py
│   ├── client.py
│   └── tools/
│       ├── issue.py
│       ├── project.py
│       └── search.py
├── Dockerfile
└── .gitlab-ci.yml
```

**Deployment:**
```json
{
  "mcpServers": {
    "jira-mcp-server": {
      "command": "docker",
      "args": [
        "run", "--rm", "-i",
        "--env", "JIRA_EMAIL",
        "--env", "JIRA_TOKEN",
        "--env", "JIRA_URL",
        "registry.gitlab.appian-stratus.com/appian/prod/jira-mcp-proxy/jira-mcp-proxy:latest"
      ],
      "env": {
        "JIRA_URL": "https://appian-eng.atlassian.net",
        "JIRA_EMAIL": "${JIRA_EMAIL}",
        "JIRA_TOKEN": "${JIRA_TOKEN}"
      }
    }
  }
}
```

**Tools Provided:**
- `get_jira_issue`, `search_jira_issues`
- `get_jira_projects`, `get_project_details`
- `get_issue_comments`, `get_issue_worklogs`

---

### Appian Atlas Knowledge Base MCP (Proposed)

**Repository:** `gitlab.appian-stratus.com/appian/atlas/gam-knowledge-base`

**Architecture:**
```
gam-knowledge-base/
├── main.py                    # Entry point (NEW - follows GitLab MCP pattern)
├── mcp_server/
│   ├── config.py              # Configuration management (NEW)
│   ├── server.py              # MCP server setup (REFACTORED)
│   ├── datasource.py          # GitLab data source (UPDATED)
│   └── tools/                 # Tool implementations (REORGANIZED)
│       ├── application.py     # list_applications, get_app_overview
│       ├── bundle.py          # search_bundles, get_bundle
│       ├── object.py          # search_objects, get_dependencies
│       └── orphan.py          # list_orphans, get_orphan
├── data/                      # Knowledge base data (UNCHANGED)
│   ├── CaseManagementStudio/
│   └── SourceSelection/
├── Dockerfile                 # Container definition (NEW)
├── .gitlab-ci.yml             # CI/CD pipeline (NEW)
└── requirements.txt
```

**Deployment:**
```json
{
  "mcpServers": {
    "appian-atlas": {
      "command": "docker",
      "args": [
        "run", "--rm", "-i",
        "--env", "GITLAB_TOKEN",
        "registry.gitlab.appian-stratus.com/appian/atlas/gam-knowledge-base:latest"
      ],
      "env": {
        "GITLAB_TOKEN": "${GITLAB_TOKEN}"
      }
    }
  }
}
```

**Tools Provided:** (UNCHANGED)
- `list_applications`, `get_app_overview`
- `search_bundles`, `get_bundle`
- `search_objects`, `get_dependencies`, `get_object_detail`
- `list_orphans`, `get_orphan`

---

## Comparison: Current vs Proposed

| Aspect | Current (GitHub) | Proposed (GitLab) |
|--------|------------------|-------------------|
| **Repository Host** | GitHub (public) | GitLab (internal) |
| **Installation** | `pipx install git+https://github.com/...` | Docker container from GitLab registry |
| **Deployment** | Python package on user machine | Docker container (isolated) |
| **Authentication** | GitHub personal access token | GitLab personal access token (same as other MCP servers) |
| **Data Access** | GitHub API (public internet) | GitLab API (internal network) |
| **Architecture** | Custom Python package | Follows GitLab MCP / Jira MCP pattern |
| **CI/CD** | None | GitLab CI/CD pipeline |
| **Container Registry** | N/A | GitLab container registry |
| **Security** | External dependency | Internal infrastructure |
| **Consistency** | Different from other Appian MCPs | Aligned with GitLab MCP / Jira MCP |

---

## Technical Implementation

### 0. Datasource API Mapping (GitHub → GitLab)

The current `GitHubDataSource` uses exactly two GitHub API operations. Both have direct
1:1 equivalents in the GitLab API — the same endpoints the existing GitLab MCP server
uses internally. This makes the datasource refactor a low-risk, thin-layer swap.

| Operation | Current GitHub API | Proposed GitLab API |
|-----------|-------------------|---------------------|
| **Read file content** | `GET https://raw.githubusercontent.com/{owner}/{repo}/{branch}/{path}` | `GET /api/v4/projects/{id}/repository/files/{file_path}/raw?ref={branch}` |
| **List directory** | `GET https://api.github.com/repos/{owner}/{repo}/contents/{path}?ref={branch}` | `GET /api/v4/projects/{id}/repository/tree?path={path}&ref={branch}` |
| **Auth header** | `Authorization: token {GITHUB_TOKEN}` | `PRIVATE-TOKEN: {GITLAB_TOKEN}` |

The existing GitLab MCP server (`gitlab-mcp-server`) already proves this pattern works
at scale with the same auth mechanism and API surface. The Appian Atlas MCP server's
tools (`list_applications`, `search_bundles`, etc.) are completely decoupled from the
data-fetching layer — they consume JSON from the datasource and don't care whether it
came from GitHub or GitLab. All caching logic (LRU cache, pinned anchor files) remains
unchanged.

### 1. Code Refactoring

**Add Configuration Management** (`config.py`):
```python
class AppianAtlasConfig:
    """Configuration management for Appian Atlas MCP server."""
    
    def __init__(self):
        self.gitlab_url = "https://gitlab.appian-stratus.com/api/v4"
        self.token = ""
        self.headers = {}
        self._initialized = False
    
    def initialize(self, token: Optional[str] = None):
        """Initialize configuration with GitLab token."""
        self.token = token or os.getenv("GITLAB_TOKEN", "")
        if not self.token:
            raise ValueError("GITLAB_TOKEN required")
        self.headers = {"PRIVATE-TOKEN": self.token}
        self._initialized = True
```

**Update Data Source** (`datasource.py`):
```python
class GitLabDataSource(DataSource):
    """Reads data from GitLab repository via API.

    Uses the same two GitLab API endpoints that the existing GitLab MCP server
    relies on internally. All caching (LRU + pinned anchor files) is preserved
    from the GitHub implementation — only the HTTP URLs and auth header change.
    """

    _PINNED_FILES = {'app_overview.json', 'search_index.json', 'orphans/_index.json'}

    def __init__(self, project_id: str, branch: str = "main",
                 token: str | None = None, data_prefix: str = "data",
                 maxsize: int = 500):
        self._project_id = project_id
        self._branch = branch
        self._token = token or os.environ.get("GITLAB_TOKEN", "")
        self._prefix = data_prefix
        self._maxsize = maxsize
        self._pinned: dict[str, dict | list] = {}
        self._cache: OrderedDict[str, dict | list] = OrderedDict()
        self._app_list: list[str] | None = None
        self._base_url = os.environ.get(
            "GITLAB_API_URL", "https://gitlab.appian-stratus.com/api/v4"
        )

    def _raw_url(self, path: str) -> str:
        """GitLab raw file content endpoint."""
        encoded_path = urllib.parse.quote(path, safe="")
        return (f"{self._base_url}/projects/{self._project_id}"
                f"/repository/files/{encoded_path}/raw?ref={self._branch}")

    def _tree_url(self, path: str) -> str:
        """GitLab repository tree endpoint (replaces GitHub contents API)."""
        return (f"{self._base_url}/projects/{self._project_id}"
                f"/repository/tree?path={path}&ref={self._branch}")

    def _headers(self) -> dict[str, str]:
        h: dict[str, str] = {}
        if self._token:
            h["PRIVATE-TOKEN"] = self._token
        return h

    # _fetch_raw, _fetch_json, LRU cache, pinned files — all unchanged from GitHub impl
```

**Reorganize Tools** (`tools/` directory):
- `application.py` - Application-level operations
- `bundle.py` - Bundle search and retrieval
- `object.py` - Object search and dependencies
- `orphan.py` - Orphan detection and retrieval

### 2. Docker Containerization

**Dockerfile:**
```dockerfile
FROM python:3.11.8-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY mcp_server/ ./mcp_server/
COPY main.py .

ENTRYPOINT ["python", "main.py"]
```

> **Note:** The base image is pinned to a specific patch version (`3.11.8-slim`)
> for reproducible CI builds. Update deliberately when upgrading Python.

**Build and Push:**
```bash
docker build -t registry.gitlab.appian-stratus.com/appian/atlas/gam-knowledge-base:latest .
docker push registry.gitlab.appian-stratus.com/appian/atlas/gam-knowledge-base:latest
```

### 3. CI/CD Pipeline

**.gitlab-ci.yml:**
```yaml
stages:
  - test
  - build
  - deploy

test:
  stage: test
  script:
    - pip install -r requirements.txt
    - python -m pytest tests/

build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
  only:
    - main
```

### 4. User Configuration

**Power Configuration Update:**

Users update their Kiro power configuration from:
```json
{
  "mcpServers": {
    "appian-atlas": {
      "command": "appian-atlas",
      "args": ["--github", "ram-020998/gam-knowledge-base"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

To:
```json
{
  "mcpServers": {
    "appian-atlas": {
      "command": "docker",
      "args": [
        "run", "--rm", "-i",
        "--env", "GITLAB_TOKEN",
        "registry.gitlab.appian-stratus.com/appian/atlas/gam-knowledge-base:latest"
      ],
      "env": {
        "GITLAB_TOKEN": "${GITLAB_TOKEN}"
      }
    }
  }
}
```

---

## Benefits

### Security
- **Internal infrastructure**: Data and code hosted within Appian's GitLab
- **Network isolation**: No external API calls to public GitHub
- **Consistent authentication**: Same GitLab tokens used across all Appian MCP servers
- **Token validation**: Built-in security checks (following GitLab MCP pattern)

### Maintainability
- **Standardized architecture**: Follows established patterns from GitLab MCP and Jira MCP
- **Containerization**: Isolated, reproducible environments
- **CI/CD integration**: Automated testing and deployment
- **Version control**: Easy rollback and version management

### User Experience
- **Simplified installation**: No Python environment setup required
- **Consistent deployment**: Same Docker pattern as other Appian MCP servers
- **Single token**: Reuse existing GitLab tokens
- **Better reliability**: Container isolation prevents dependency conflicts

### Operational
- **Centralized hosting**: All Appian Atlas components in one place
- **Automated builds**: GitLab CI/CD handles building and publishing
- **Container registry**: Versioned Docker images in GitLab registry
- **Monitoring**: Leverage GitLab's built-in monitoring and logging

---

## Migration Plan

### Phase 1: Repository Migration
1. Create GitLab group: `gitlab.appian-stratus.com/appian/atlas`
2. Migrate all repositories from GitHub to GitLab
3. Update submodule references in parent repository
4. Verify data integrity

### Phase 2: Code Refactoring
1. Refactor MCP server to follow GitLab MCP pattern
2. Add `config.py` for configuration management
3. Implement `GitLabDataSource` class
4. Reorganize tools into modular structure
5. Add comprehensive error handling and logging

### Phase 3: Containerization
1. Create Dockerfile
2. Set up GitLab CI/CD pipeline
3. Configure container registry
4. Test Docker image locally
5. Push to GitLab registry

### Phase 4: Testing
1. Test MCP server with GitLab data source
2. Verify all tools work correctly
3. Test Docker container deployment
4. Validate token authentication
5. Performance testing

### Phase 5: Documentation & Rollout
1. Update README with new installation instructions
2. Update Power configurations
3. Create migration guide for existing users
4. Announce migration timeline
5. Provide support during transition

### Phase 6: Deprecation
1. Mark GitHub repositories as archived
2. Redirect users to GitLab
3. Remove GitHub-based Powers
4. Complete migration

---

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| **User disruption** | Medium | Provide clear migration guide, maintain GitHub temporarily |
| **Token management** | Low | Users already have GitLab tokens for other MCP servers |
| **Docker dependency** | Low | Docker already required for GitLab MCP and Jira MCP |
| **Data migration issues** | Low | Simple repository copy, no data transformation needed |
| **Tool compatibility** | Low | Tool interfaces remain unchanged |
| **Network access** | Low | Users already access internal GitLab for other work |
| **Datasource refactor** | Low | Only 2 API endpoints change (file read + directory list); same pattern already proven by GitLab MCP server; all caching logic unchanged |
| **GitLab single point of failure** | Medium | If GitLab goes down, all three MCP servers (GitLab, Jira proxy, Atlas) become unavailable simultaneously. Mitigate with local data caching in the Docker container and clear incident communication. Currently GitHub provides infrastructure diversity — this is traded for consistency. |
| **Approval lead time** | Medium | Security review and infrastructure approval may add 2-3 weeks before Phase 1 can start. Engage reviewers early in parallel with planning. |

---

## Success Criteria

1. ✅ All repositories successfully migrated to GitLab
2. ✅ Docker image builds and runs successfully
3. ✅ All MCP tools function identically to current implementation
4. ✅ CI/CD pipeline automates builds and deployments
5. ✅ Users can install and use with same ease as GitLab MCP
6. ✅ Documentation updated and comprehensive
7. ✅ Zero data loss during migration
8. ✅ Performance equal to or better than current implementation

---

## Timeline Estimate

| Phase | Duration | Dependencies |
|-------|----------|--------------|
| Phase 0: Approvals & Reviews | 2-3 weeks | Security review, infrastructure team, stakeholder sign-off (run in parallel with planning) |
| Phase 1: Repository Migration | 1 week | Phase 0 complete (GitLab group creation approval) |
| Phase 2: Code Refactoring | 2 weeks | Phase 1 complete |
| Phase 3: Containerization | 1 week | Phase 2 complete |
| Phase 4: Testing | 1 week | Phase 3 complete |
| Phase 5: Documentation & Rollout | 1 week | Phase 4 complete |
| Phase 6: Deprecation | 2 weeks | Phase 5 complete, user adoption |
| **Total** | **~10-11 weeks** | (8 weeks execution + 2-3 weeks approval lead time) |

---

## Approval Requirements

### Technical Approval
- [ ] Architecture review by platform team
- [ ] Security review for token handling
- [ ] Infrastructure team approval for GitLab resources

### Operational Approval
- [ ] GitLab group creation
- [ ] Container registry access
- [ ] CI/CD pipeline permissions

### Stakeholder Approval
- [ ] Product owner sign-off
- [ ] User communication plan approval
- [ ] Migration timeline approval

---

## Appendix A: Configuration Examples

### Current GitHub Configuration
```json
{
  "mcpServers": {
    "appian-atlas": {
      "command": "appian-atlas",
      "args": ["--github", "ram-020998/gam-knowledge-base"],
      "env": {
        "GITHUB_TOKEN": "ghp_xxxxxxxxxxxxxxxxxxxx"
      }
    }
  }
}
```

### Proposed GitLab Configuration
```json
{
  "mcpServers": {
    "gitlab": {
      "command": "docker",
      "args": ["run", "--rm", "-i", "--env", "GITLAB_TOKEN",
               "registry.gitlab.appian-stratus.com/appian/prod/gitlab-mcp-server/gitlab-mcp-server:latest"],
      "env": {"GITLAB_TOKEN": "glpat-xxxxxxxxxxxxxxxxxxxx"}
    },
    "jira-mcp-server": {
      "command": "docker",
      "args": ["run", "--rm", "-i", "--env", "JIRA_EMAIL", "--env", "JIRA_TOKEN", "--env", "JIRA_URL",
               "registry.gitlab.appian-stratus.com/appian/prod/jira-mcp-proxy/jira-mcp-proxy:latest"],
      "env": {
        "JIRA_URL": "https://appian-eng.atlassian.net",
        "JIRA_EMAIL": "user@appian.com",
        "JIRA_TOKEN": "ATATTxxxxxxxxxxxxxxxxxxxx"
      }
    },
    "appian-atlas": {
      "command": "docker",
      "args": ["run", "--rm", "-i", "--env", "GITLAB_TOKEN",
               "registry.gitlab.appian-stratus.com/appian/atlas/gam-knowledge-base:latest"],
      "env": {"GITLAB_TOKEN": "glpat-xxxxxxxxxxxxxxxxxxxx"}
    }
  }
}
```

**Note:** All three MCP servers now follow the same Docker-based deployment pattern.

---

## Appendix B: Tool Comparison

### Tools Remain Unchanged

All existing tools maintain their exact interfaces and functionality:

| Tool | Description | Status |
|------|-------------|--------|
| `list_applications` | Lists available GAM applications | ✅ Unchanged |
| `get_app_overview` | Returns comprehensive app map | ✅ Unchanged |
| `search_bundles` | Finds bundles by keyword | ✅ Unchanged |
| `search_objects` | Searches parsed objects by name | ✅ Unchanged |
| `get_bundle` | Loads bundle at requested detail level | ✅ Unchanged |
| `get_dependencies` | Returns dependency subgraph | ✅ Unchanged |
| `get_object_detail` | Returns object info by UUID | ✅ Unchanged |
| `list_orphans` | Lists unreachable objects | ✅ Unchanged |
| `get_orphan` | Returns orphan detail with code | ✅ Unchanged |

**User Impact:** Zero - all queries and workflows remain identical.

---

## Questions & Answers

**Q: Why migrate from GitHub to GitLab?**  
A: To align with Appian's internal infrastructure, improve security, and standardize deployment patterns with existing Appian MCP servers (GitLab MCP, Jira MCP).

**Q: Will users need to reinstall anything?**  
A: Yes, but it's simpler - just update the Power configuration. No Python environment setup needed, Docker handles everything.

**Q: What happens to existing GitHub repositories?**  
A: They'll be archived and marked as deprecated, with clear redirects to GitLab.

**Q: Will this break existing workflows?**  
A: No - all tools and their interfaces remain identical. Only the deployment method changes.

**Q: Do users need new tokens?**  
A: Users likely already have GitLab tokens for the GitLab MCP server. They can reuse the same token.

**Q: What if Docker isn't installed?**  
A: Docker is already required for GitLab MCP and Jira MCP servers, so users should already have it.

**Q: How long will the migration take?**  
A: Estimated 10-11 weeks total (8 weeks execution + 2-3 weeks approval lead time), with minimal user disruption during the transition.

**Q: How risky is the datasource refactor from GitHub to GitLab?**  
A: Low risk. The current `GitHubDataSource` uses only 2 API operations: read a file and list a directory. Both have direct 1:1 equivalents in the GitLab API (`/repository/files/{path}/raw` and `/repository/tree`). These are the same endpoints the existing GitLab MCP server uses internally. The MCP tools themselves are completely decoupled from the data-fetching layer — they consume JSON and don't care about the source. All caching logic (LRU cache, pinned anchor files) stays unchanged.

**Q: Can we reuse the GitLab MCP server's API client directly?**  
A: Not directly (MCP-to-MCP calls aren't how it works), but we replicate the same HTTP pattern. The GitLab MCP server proves the approach works — same auth header (`PRIVATE-TOKEN`), same API surface. Our datasource just needs to swap 2 URL templates and 1 auth header.

**Q: What happens if GitLab goes down after migration?**  
A: All three MCP servers (GitLab, Jira proxy, Atlas) would be unavailable simultaneously since they all depend on GitLab infrastructure. This is a trade-off vs the current setup where GitHub provides infrastructure diversity. Mitigation includes local data caching in the Docker container and clear incident communication.

---

## Conclusion

Migrating Appian Atlas to GitLab and adopting the Docker-based MCP server pattern used by GitLab MCP and Jira MCP will:

1. **Improve security** by keeping all data within Appian's infrastructure
2. **Standardize deployment** across all Appian MCP servers
3. **Simplify maintenance** through containerization and CI/CD
4. **Enhance user experience** with consistent installation patterns
5. **Reduce external dependencies** by eliminating GitHub reliance

The datasource refactor — the core technical change — is low-risk: only 2 API endpoints
and 1 auth header change, with the pattern already proven by the existing GitLab MCP
server. All MCP tools, caching logic, and user-facing interfaces remain identical.

This migration aligns Appian Atlas with Appian's internal tooling ecosystem while maintaining full backward compatibility for end users.

---

**Prepared by:** Appian Atlas Team  
**Review Date:** February 24, 2026  
**Next Review:** Upon approval
