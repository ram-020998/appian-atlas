# Appian Atlas GitLab Migration — Migration Plan

**Parent document:** [00-overview.md](00-overview.md)

---

## Migration Phases

### Phase 0: Approvals & Reviews (2–3 weeks)

Run in parallel with planning. Do not block on this before starting design work.

- [ ] Security review for token handling and Dockerfile pattern
- [ ] Infrastructure team approval for GitLab group creation
- [ ] Architecture review by platform team
- [ ] Stakeholder sign-off on timeline
- [ ] User communication plan approval

### Phase 1: Local Project Setup (1 week)

> **Note:** GitLab group and repository creation requires approvals (Phase 0). All Phase 1 work is done locally in `appian-atlas-gitlab/` and pushed to GitLab later once approvals are granted.

1. Create local project structure under `appian-atlas-gitlab/`:
   - `atlas-mcp-server/` — MCP server (populated in Phase 2)
   - `solutions-knowledge-base/` — application data
   - `atlas-parser/` — Appian app parser
   - `atlas-kiro-powers/` — consolidated Kiro powers
2. Copy data from existing GitHub repos into local folders:
   - `gam-knowledge-base/data/` → `solutions-knowledge-base/data/`
   - `appian-parser` → `atlas-parser`
   - `power-appian-atlas` → `atlas-kiro-powers/atlas/`
   - `power-appian-atlas-developer` → `atlas-kiro-powers/atlas-developer/`
   - `power-appian-atlas-product-owner` → `atlas-kiro-powers/atlas-product-owner/`
   - `power-appian-atlas-ux-designer` → `atlas-kiro-powers/atlas-ux-designer/`
   - `power-appian-reference` → `atlas-kiro-powers/appian-reference/`
3. Verify data integrity (compare file counts, checksums)
4. Initialize git repos locally for each folder

### Phase 2: Code Refactoring (2 weeks)

Build the `atlas-mcp-server` repo following `gitlab-mcp-server` patterns.

**Week 1 — Core infrastructure:**
1. Create `atlas_mcp/` package with `__init__.py`
2. Implement `config.py` — configuration management (hardcode `SOLUTIONS_KB_PROJECT_ID` — use placeholder value locally, update after GitLab repo is created)
3. Implement `client.py` — GitLab HTTP client
4. Implement `token_validator.py` — read-only token enforcement
5. Implement `logging_config.py` — structured logging
6. Implement `utils.py` — utility functions
7. Implement `main.py` — async entry point

**Week 2 — Server and tools:**
1. Implement `models.py` — data models and tool schemas
2. Implement `datasource.py` — GitLab data source (reads from `solutions-knowledge-base` via API)
3. Implement `server.py` — MCP server with tool routing
4. Refactor tools into `tools/` directory:
   - `application.py`
   - `bundle.py`
   - `object.py`
   - `orphan.py`
5. Implement `tools/__init__.py`

### Phase 3: Containerization & CI/CD (1 week)

1. Create `Dockerfile` (production — Chainguard multi-stage)
2. Create `Dockerfile.local` (local dev — python:3.11-slim)
3. Create `docker-compose.yml`
4. Create `.gitlab-ci.yml` for `atlas-mcp-server` (shared templates, tox, crane)
5. Create `Makefile`
6. Create `tox.ini`
7. Create `.dockerignore`, `.gitignore`, and `.env.example`
8. Create `mcp.json` reference configuration
9. Create `requirements.txt` (production) and `requirements-dev.txt` (development/testing)
10. Configure container registry access for `atlas-mcp-server`
11. Test CI pipeline end-to-end (lint → test → build → tag)

> **Deferred:** The `solutions-knowledge-base` daily data pipeline (scheduled CI that runs `atlas-parser` to refresh application data) is out of scope for this migration. It will be specified in a separate implementation plan once the core migration is complete and all 4 repos are operational.

### Phase 4: Testing (1 week)

1. Create `pytest.ini` and `run_tests.py`
2. Write test suite:
   - `test_basic.py` — imports, sanity checks
   - `test_config.py` — config initialization, data project config
   - `test_client.py` — HTTP client, errors, pagination
   - `test_utils.py` — utility functions
   - `test_server.py` — tool routing
   - `test_token_validation.py` — scope validation
   - `test_datasource.py` — data fetching from `solutions-knowledge-base`, caching
   - `test_application.py`, `test_bundle.py`, `test_object.py`, `test_orphan.py` — tool tests
3. Achieve 80% minimum coverage
4. Test Docker container locally with `make docker-build-local && make docker-run-local`
5. Validate token authentication end-to-end
6. Verify MCP server correctly reads data from `solutions-knowledge-base` repo
7. Performance testing — compare response times with current GitHub-based implementation

### Phase 5: Documentation & Rollout (1 week)

1. Create `docs/SECURITY.md` in `atlas-mcp-server`
2. Create `docs/USAGE_EXAMPLES.md` in `atlas-mcp-server`
3. Update `README.md` for all 4 repos
4. Update Power configurations in `atlas-kiro-powers` to point to GitLab Docker image
5. Create user migration guide (step-by-step)
6. Announce migration timeline to users
7. Provide support channel during transition

### Phase 6: GitLab Push & Go-Live (2 weeks)

> **Note:** GitHub repositories are intentionally left as-is — they serve as a backup. No archiving or deprecation notices needed.

1. Create GitLab group: `gitlab.appian-stratus.com/appian/atlas` (requires Phase 0 approvals)
2. Create 4 GitLab repositories and push local code:
   - `atlas-mcp-server`
   - `solutions-knowledge-base`
   - `atlas-parser`
   - `atlas-kiro-powers`
3. Update `SOLUTIONS_KB_PROJECT_ID` in `config.py` with actual GitLab project ID
4. Verify CI/CD pipelines run successfully on GitLab
5. Verify Docker image builds and publishes to container registry
6. Communicate migration to users with installation guide
7. Provide support channel during transition
8. Monitor adoption — ensure users can connect successfully

---

## Risk Assessment

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| **User disruption during migration** | Medium | Medium | Provide clear migration guide, maintain GitHub temporarily, support channel |
| **Token management confusion** | Low | Low | Users already have GitLab tokens for other MCP servers |
| **Docker not installed** | Low | Low | Docker already required for GitLab MCP and Jira MCP servers |
| **Data migration issues** | Low | Low | Simple file copy from GitHub to GitLab, verify with file counts and checksums |
| **Tool compatibility regression** | Medium | Low | Comprehensive test suite with 80% coverage, tool interfaces unchanged |
| **Network access to internal GitLab** | Low | Low | Users already access internal GitLab for other work |
| **Datasource refactor** | Low | Low | Only 2 API endpoints change; same pattern proven by GitLab MCP server; all caching unchanged |
| **Cross-repo data access** | Low | Low | `atlas-mcp-server` reads from `solutions-knowledge-base` via standard GitLab API; same token works for both |
| **Daily data pipeline** | Medium | Low | *(Deferred to separate implementation plan)* Pipeline in `solutions-knowledge-base` uses `atlas-parser` as read-only dependency; no cross-repo write permissions needed. Will be specified after core migration is complete. |
| **GitLab single point of failure** | Medium | Low | If GitLab goes down, all three MCP servers unavailable. Mitigate with local data caching and clear incident communication. Trade-off: consistency over infrastructure diversity. |
| **Approval lead time** | Medium | Medium | Security review and infrastructure approval may add 2–3 weeks. Engage reviewers early in parallel with planning. |
| **Chainguard image access** | Low | Low | Same image used by gitlab-mcp-server; already approved. Dockerfile.local provides fallback for local dev. |
| **CI shared template changes** | Low | Low | Templates are stable and used across Appian. Pin to specific versions if needed. |
| **Power consolidation** | Low | Low | 5 power repos → 1 repo with subdirectories. Content unchanged, only organization changes. |

---

## Rollback Strategy

If critical issues are discovered post-migration:

1. **Immediate:** Users revert their `mcp.json` to the GitHub-based configuration (old config preserved in migration guide)
2. **Short-term:** GitHub repositories remain accessible (archived but not deleted) for at least 4 weeks after migration
3. **Server code:** GitLab container registry retains all image versions — roll back to any previous SHA tag

---

## Success Criteria

1. ✅ All 4 project folders created locally in `appian-atlas-gitlab/` and pushed to GitLab
2. ✅ Docker image builds and runs successfully via CI/CD pipeline in `atlas-mcp-server`
3. ✅ All 9 MCP tools function identically to current implementation
4. ✅ MCP server correctly reads data from `solutions-knowledge-base` repo via GitLab API
5. ✅ CI/CD pipeline passes: lint, test (80%+ coverage), build, tag
6. ✅ Users can install and use with same ease as GitLab MCP server
7. ✅ Token validation rejects write-scoped tokens
8. ✅ Logging works (file + stderr, startup/shutdown, tool calls)
9. ✅ All 5 powers consolidated in `atlas-kiro-powers` and functional
10. ✅ Documentation complete: README (all repos), SECURITY.md, USAGE_EXAMPLES.md, migration guide
11. ✅ Zero data loss during migration
12. ✅ Performance equal to or better than current implementation
13. ✅ Production Docker image contains only production dependencies (`requirements.txt`)

> **Deferred success criterion:** Daily data pipeline in `solutions-knowledge-base` — will be tracked in the separate data pipeline implementation plan.

---

## Timeline Summary

| Phase | Duration | Dependencies |
|-------|----------|--------------|
| Phase 0: Approvals & Reviews | 2–3 weeks | Security review, infrastructure team, stakeholder sign-off |
| Phase 1: Local Project Setup | 1 week | Can start immediately (parallel with Phase 0) |
| Phase 2: Code Refactoring | 2 weeks | Phase 1 complete |
| Phase 3: Containerization & CI/CD | 1 week | Phase 2 complete |
| Phase 4: Testing | 1 week | Phase 3 complete |
| Phase 5: Documentation & Rollout | 1 week | Phase 4 complete |
| Phase 6: GitLab Push & Go-Live | 2 weeks | Phase 0 + Phase 5 complete |
| **Total** | **~8–9 weeks** | Phases 0–1 run in parallel, reducing wall-clock time |

---

## Approval Checklist

### Technical
- [ ] Architecture review by platform team
- [ ] Security review for token handling and Dockerfile
- [ ] Infrastructure team approval for GitLab resources

### Operational
- [ ] GitLab group creation (`appian/atlas`) — needed before Phase 6
- [ ] 4 repositories created on GitLab (`atlas-mcp-server`, `solutions-knowledge-base`, `atlas-parser`, `atlas-kiro-powers`) — needed before Phase 6
- [ ] Container registry access for `atlas-mcp-server`
- [ ] CI/CD pipeline permissions for all repos

### Stakeholder
- [ ] Product owner sign-off
- [ ] User communication plan approval
- [ ] Migration timeline approval

---

## Q&A

**Q: Why migrate from GitHub to GitLab?**
A: To align with Appian's internal infrastructure, improve security, and standardize deployment patterns with existing Appian MCP servers (GitLab MCP, Jira MCP).

**Q: Why 4 repos instead of the current 8?**
A: Clean separation of concerns. The MCP server code, application data, parser, and powers all change at different cadences and for different reasons. The 5 power repos are consolidated into 1 since they're all lightweight steering docs. `atlas-docs` (GitHub Pages) is dropped since GitLab has its own Pages feature if needed.

**Q: Will users need to reinstall anything?**
A: Yes, but it's simpler — just update the MCP configuration in `mcp.json`. No Python environment setup needed, Docker handles everything.

**Q: What happens to existing GitHub repositories?**
A: All 8 GitHub repos will be archived and marked as deprecated, with clear redirects to GitLab. Kept accessible for at least 4 weeks post-migration.

**Q: Will this break existing workflows?**
A: No — all 9 tools and their interfaces remain identical. Only the deployment method changes.

**Q: Do users need new tokens?**
A: Users likely already have GitLab tokens for the GitLab MCP server. They can reuse the same token.

**Q: What if Docker isn't installed?**
A: Docker is already required for GitLab MCP and Jira MCP servers, so users should already have it.

**Q: How long will the migration take?**
A: ~10–11 weeks total (8 weeks execution + 2–3 weeks approval lead time), with minimal user disruption during the transition.

**Q: How risky is the datasource refactor?**
A: Low risk. Only 2 API endpoints change (file read + directory list). Same pattern already proven by GitLab MCP server. All caching logic unchanged. Tool layer completely decoupled from data-fetching layer.

**Q: How does the MCP server know where to find the data?**
A: The `solutions-knowledge-base` GitLab project ID is **hardcoded** in the server's `config.py` as `SOLUTIONS_KB_PROJECT_ID`. During local development (Phases 1–5), a placeholder value is used. The actual project ID is set in Phase 6 when the GitLab repo is created. Users do not need to know or pass this value — they only provide `GITLAB_TOKEN`. This keeps the `mcp.json` configuration simple (one env var, matching the GitLab MCP server pattern). An env var override (`ATLAS_DATA_PROJECT_ID`) exists for development/testing only.

**Q: How does the daily data pipeline work?**
A: This is deferred to a separate implementation plan. At a high level: a scheduled CI pipeline in `solutions-knowledge-base` will pull `atlas-parser` as a dependency, run it against Appian dev environments, and commit updated JSON files to its own repo. The detailed spec (authentication, scheduling, failure alerting) will be created once the core migration is complete.

**Q: What happens if GitLab goes down after migration?**
A: All three MCP servers would be unavailable simultaneously. This is a trade-off for consistency. Mitigation includes local data caching and clear incident communication.

**Q: Can we roll back if something goes wrong?**
A: Yes. Users revert their `mcp.json` to the old GitHub config. GitHub repos remain accessible (archived) for at least 4 weeks. Docker images are versioned by commit SHA for server rollback.

---

## GitHub → GitLab Repository Mapping

| GitHub (Current) | GitLab (Proposed) | Notes |
|-------------------|-------------------|-------|
| `gam-knowledge-base` (data + server) | `atlas-mcp-server` (server only) | Server code extracted, renamed |
| `gam-knowledge-base` (data + server) | `solutions-knowledge-base` (data only) | Data extracted, renamed, daily pipeline added |
| `appian-parser` | `atlas-parser` | Renamed |
| `atlas-docs` | *(dropped)* | Not needed for GitLab |
| `power-appian-atlas` | `atlas-kiro-powers/atlas/` | Consolidated |
| `power-appian-atlas-developer` | `atlas-kiro-powers/atlas-developer/` | Consolidated |
| `power-appian-atlas-product-owner` | `atlas-kiro-powers/atlas-product-owner/` | Consolidated |
| `power-appian-atlas-ux-designer` | `atlas-kiro-powers/atlas-ux-designer/` | Consolidated |
| `power-appian-reference` | `atlas-kiro-powers/appian-reference/` | Consolidated |
