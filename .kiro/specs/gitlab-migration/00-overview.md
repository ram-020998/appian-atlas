# Appian Atlas GitLab Migration — Overview

**Document Version:** 2.3
**Date:** February 25, 2026
**Status:** Proposal — Pending Approval
**Changelog:**
- v1.0 — Initial architecture proposal
- v1.1 — Added API mapping, datasource risk assessment, CI/CD fixes, timeline adjustment
- v2.0 — Complete rewrite after gap analysis against `gitlab-mcp-server` reference repo. Split into multi-document spec.
- v2.1 — Finalized GitLab repo structure: 4 separate repos (atlas-mcp-server, solutions-knowledge-base, atlas-parser, atlas-kiro-powers). Removed all references to gam-knowledge-base monorepo. Updated CI/CD, Docker registry paths, mcp.json, and data pipeline ownership.
- v2.2 — Split `requirements.txt` into prod/dev. Deferred daily data pipeline to separate plan. Marked v1.1 standalone doc as superseded. Pipeline monitoring via GitLab email notifications.
- v2.3 — Local-first development: work in `appian-atlas-gitlab/` folder, push to GitLab after approvals. GitHub repos kept as backup (no archiving). Phase 1 and Phase 6 rewritten. Timeline reduced to ~8–9 weeks.

---

## Document Index

| Document | Contents |
|----------|----------|
| [00-overview.md](00-overview.md) | This file — summary, decisions, timeline |
| [01-architecture.md](01-architecture.md) | Current vs proposed architecture, data flow, repo structure |
| [02-technical-spec.md](02-technical-spec.md) | File-by-file implementation spec — Dockerfile, CI/CD, main.py, config, client, server, models, logging, token validation, datasource, tools |
| [03-development-tooling.md](03-development-tooling.md) | Makefile, tox.ini, docker-compose.yml, Dockerfile.local, .dockerignore, .gitignore, test strategy |
| [04-migration-plan.md](04-migration-plan.md) | Phases, risk assessment, success criteria, approvals |

---

## Executive Summary

This spec proposes migrating the Appian Atlas infrastructure from GitHub to GitLab, following the exact patterns established by Appian's existing `gitlab-mcp-server`. This migration will:

- Reorganize into 4 purpose-built GitLab repositories (MCP server, data, parser, powers)
- Standardize deployment using Docker containers hosted in GitLab's container registry
- Maintain all existing MCP tool functionality with zero user-facing changes
- Align with Appian's internal tooling ecosystem (same Dockerfile, CI/CD, logging, token validation patterns)

---

## GitLab Repository Structure

```
gitlab.appian-stratus.com/appian/atlas/
├── atlas-mcp-server              # MCP server (Docker, CI/CD, registry)
├── solutions-knowledge-base      # Application data (JSON, updated daily by pipeline)
├── atlas-parser                  # Appian app parser (generates data)
└── atlas-kiro-powers             # All Kiro powers (consolidated)
    ├── atlas/                    # Landing page power
    ├── atlas-developer/          # Developer persona
    ├── atlas-product-owner/      # Product Owner persona
    ├── atlas-ux-designer/        # UX Designer persona
    └── appian-reference/         # SAIL language reference
```

Each repo has a single clear responsibility:
- **atlas-mcp-server** — the Docker-containerized MCP server that users interact with. Fetches data at runtime from `solutions-knowledge-base` via GitLab API.
- **solutions-knowledge-base** — parsed application data (JSON files). Updated daily by a scheduled CI pipeline that runs `atlas-parser` against dev environments.
- **atlas-parser** — the Appian application parser tool. Used as a dependency by the data pipeline in `solutions-knowledge-base`.
- **atlas-kiro-powers** — all Kiro power configurations (steering docs + MCP server config) consolidated into one repo.

---

## Key Architectural Decisions

These decisions were made after a gap analysis comparing the original v1.1 proposal against the actual `gitlab-mcp-server` reference repo (`gitlab.appian-stratus.com/appian/prod/gitlab-mcp-server`).

| # | Decision | Rationale |
|---|----------|-----------|
| 1 | **Chainguard multi-stage Dockerfile** | Match reference repo. Required for Appian security compliance. Non-root user (UID 65532). |
| 2 | **Shared CI templates** (k8s-gitlab-runners, stratus-pipeline-tools) | Appian-wide standard. Includes k8s executors, STRATUS_JWT auth, crane tagging, tox testing. |
| 3 | **Token validation at startup** | Reject write-scoped tokens. Same `token_validator.py` pattern as reference. |
| 4 | **Structured logging** (file + stderr) | Full `logging_config.py` — startup/shutdown info, tool call tracing, log file in mcp.json directory. |
| 5 | **Separate `client.py`** | Dedicated HTTP client with session reuse, pagination, error handling, timeouts. |
| 6 | **Separate `models.py`** | Tool schemas and data models extracted from tool implementations. |
| 7 | **Full dev tooling** | Makefile, tox.ini, docker-compose.yml, Dockerfile.local — matching reference. |
| 8 | **.dockerignore + .gitignore** | Proper ignore files matching reference patterns. |
| 9 | **Full test suite** | `tests/` directory with per-module tests, pytest.ini, run_tests.py, 80% coverage threshold. |
| 10 | **Package name: `atlas_mcp/`** | Mirrors `gitlab_mcp/` naming convention. |
| 11 | **Full `main.py` spec** | Async entry point with logging setup, config init, token validation, error handling. |
| 12 | **Ship `mcp.json`** | Reference configuration file at repo root for easy user setup. |
| 13 | **`docs/` directory** | SECURITY.md + tool usage examples. |
| 14 | **Runtime data fetch via GitLab API** | Data NOT baked into Docker image. Daily pipeline updates `solutions-knowledge-base`. Users never need to re-pull image for fresh data. |
| 15 | **`__init__.py` with version metadata** | `__version__`, `__author__`, `__description__` — standard Python packaging. |
| 16 | **4 separate repos** | MCP server, data, parser, and powers each in their own repo. Clean separation of concerns — different change cadences, different owners, different CI needs. |
| 17 | **Split `requirements.txt`** | Production deps in `requirements.txt`, dev/test deps in `requirements-dev.txt`. Deviates from reference repo but keeps production Docker image clean. |
| 18 | **Daily data pipeline deferred** | Pipeline spec for `solutions-knowledge-base` will be a separate implementation plan after core migration is complete. |
| 19 | **Hardcoded data project ID** | The `solutions-knowledge-base` GitLab project ID is hardcoded in `config.py` as `SOLUTIONS_KB_PROJECT_ID` — not passed via env var or Docker args. Users only need `GITLAB_TOKEN`. Env var override (`ATLAS_DATA_PROJECT_ID`) exists for dev/testing only. Placeholder used during local dev, actual ID set in Phase 6. |
| 20 | **Local-first development** | All work done in `appian-atlas-gitlab/` folder locally. GitLab group/repo creation deferred to Phase 6 (after approvals). Phases 1–5 don't require GitLab access. |
| 21 | **GitHub repos kept as backup** | No archiving or deprecation of GitHub repos. They remain as-is and serve as a backup/reference. |

---

## Reference Repo

All patterns are derived from:

- **Repo:** `gitlab.appian-stratus.com/appian/prod/gitlab-mcp-server`
- **Local clone:** `/Users/ramaswamy.u/repo/gitlab-mcp-server`

---

## Target File Structure — `atlas-mcp-server`

```
atlas-mcp-server/
├── main.py                        # Async entry point
├── mcp.json                       # Reference MCP configuration
├── requirements.txt               # Production dependencies
├── requirements-dev.txt           # Development & testing dependencies
├── Dockerfile                     # Production (Chainguard multi-stage)
├── Dockerfile.local               # Local dev (python:3.11-slim)
├── docker-compose.yml             # Local container testing
├── Makefile                       # Dev task runner
├── tox.ini                        # Test/lint orchestration
├── pytest.ini                     # Pytest configuration
├── run_tests.py                   # Standalone test runner
├── .gitlab-ci.yml                 # CI/CD pipeline
├── .dockerignore
├── .gitignore
├── .env.example                   # Environment variable reference for developers
├── atlas_mcp/                     # Main package
│   ├── __init__.py                # Version metadata
│   ├── config.py                  # Configuration management
│   ├── server.py                  # MCP server setup + tool routing
│   ├── client.py                  # GitLab HTTP client
│   ├── models.py                  # Data models + tool schemas
│   ├── datasource.py              # GitLab data source (runtime fetch from solutions-knowledge-base)
│   ├── token_validator.py         # Read-only token enforcement
│   ├── logging_config.py          # Structured logging
│   ├── utils.py                   # Utility functions
│   └── tools/                     # Tool implementations
│       ├── __init__.py
│       ├── application.py         # list_applications, get_app_overview
│       ├── bundle.py              # search_bundles, get_bundle
│       ├── object.py              # search_objects, get_dependencies, get_object_detail
│       └── orphan.py              # list_orphans, get_orphan
├── tests/                         # Test suite
│   ├── __init__.py
│   ├── test_basic.py
│   ├── test_config.py
│   ├── test_client.py
│   ├── test_utils.py
│   ├── test_server.py
│   ├── test_token_validation.py
│   ├── test_datasource.py
│   ├── test_application.py
│   ├── test_bundle.py
│   ├── test_object.py
│   └── test_orphan.py
└── docs/
    ├── SECURITY.md
    └── USAGE_EXAMPLES.md
```

## Target File Structure — `solutions-knowledge-base`

```
solutions-knowledge-base/
├── data/
│   ├── ClauseAutomation/
│   │   ├── app_overview.json
│   │   ├── search_index.json
│   │   ├── bundles/
│   │   ├── objects/
│   │   ├── orphans/
│   │   └── enrichment/
│   └── SourceSelection/
│       ├── app_overview.json
│       ├── search_index.json
│       ├── bundles/
│       ├── objects/
│       ├── orphans/
│       └── enrichment/
├── .gitlab-ci.yml                 # Daily scheduled pipeline (runs atlas-parser)
└── README.md
```

## Target File Structure — `atlas-kiro-powers`

```
atlas-kiro-powers/
├── atlas/                         # Landing page power
├── atlas-developer/               # Developer persona
├── atlas-product-owner/           # Product Owner persona
├── atlas-ux-designer/             # UX Designer persona
├── appian-reference/              # SAIL language reference
└── README.md
```

---

## Data Pipeline Flow

> **Deferred:** The daily data pipeline will be specified in a separate implementation plan once the core migration is complete.

At a high level, the intended flow is:

```
solutions-knowledge-base CI (daily scheduled pipeline)
  → pulls atlas-parser (as pip dependency or Docker image)
  → runs parser against Appian dev environments
  → commits updated JSON files to solutions-knowledge-base repo
```

The pipeline will live in `solutions-knowledge-base` so it writes to its own repo — no cross-repo write permissions needed. `atlas-parser` is consumed as a read-only dependency. Detailed spec (authentication to dev environments, cron schedule, failure alerting, commit strategy) will be covered in the dedicated plan.

---

## Timeline Estimate

| Phase | Duration | Dependencies |
|-------|----------|--------------|
| Phase 0: Approvals & Reviews | 2–3 weeks | Security review, infrastructure team, stakeholder sign-off |
| Phase 1: Repository Setup | 1 week | Phase 0 complete |
| Phase 2: Code Refactoring | 2 weeks | Phase 1 complete |
| Phase 3: Containerization & CI/CD | 1 week | Phase 2 complete |
| Phase 4: Testing | 1 week | Phase 3 complete |
| Phase 5: Documentation & Rollout | 1 week | Phase 4 complete |
| Phase 6: Deprecation | 2 weeks | Phase 5 complete, user adoption |
| **Total** | **~10–11 weeks** | 8 weeks execution + 2–3 weeks approval lead time |

---

## Approval Requirements

### Technical
- [ ] Architecture review by platform team
- [ ] Security review for token handling and Dockerfile
- [ ] Infrastructure team approval for GitLab resources

### Operational
- [ ] GitLab group creation (`appian/atlas`)
- [ ] Container registry access for `atlas-mcp-server`
- [ ] CI/CD pipeline permissions for all 4 repos

### Stakeholder
- [ ] Product owner sign-off
- [ ] User communication plan approval
- [ ] Migration timeline approval

---

**Prepared by:** Appian Atlas Team
**Review Date:** February 24, 2026
**Next Review:** Upon approval
