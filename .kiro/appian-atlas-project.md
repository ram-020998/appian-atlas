# Appian Atlas Project

## Project Overview

Appian Atlas is a comprehensive system for parsing, storing, and exploring Appian applications through an AI-powered interface. It consists of three main components:

1. **Parser** - Converts Appian ZIP packages to structured JSON
2. **Knowledge Base** - Stores parsed data and provides MCP server for querying
3. **Powers** - Persona-specific AI assistants for different user roles

## Repository Structure

This is the parent repository that aggregates all sub-repositories as Git submodules. GitHub Pages is hosted from the separate `atlas-docs` repo to avoid build failures caused by the private `appian-parser` submodule.

```
appian-atlas/
├── appian-parser/                      # Core parsing engine (private)
├── atlas-docs/                         # GitHub Pages site (public)
├── gam-knowledge-base/                 # Data repository + MCP server
├── power-appian-atlas/                 # Landing page for powers
├── power-appian-atlas-developer/       # Developer persona power
├── power-appian-atlas-product-owner/   # Product Owner persona power
└── power-appian-atlas-ux-designer/     # UX Designer persona power
```

## Component Details

### 1. appian-parser

**Purpose**: Core Python package that parses Appian application ZIP files into structured JSON.

**Key Features**:
- Extracts all Appian objects (Interfaces, Expression Rules, Process Models, etc.)
- Resolves dependencies and creates dependency graphs
- Organizes objects into functional bundles (actions, processes, pages, etc.)
- Generates search indexes for fast lookup

**Main Modules**:
- `appian_parser/parser/` - Core parsing logic
- `appian_parser/output/` - Output generation (bundles, indexes, etc.)
- `appian_parser/models/` - Data models for Appian objects
- `mcp_server/` - MCP server implementation (also in gam-knowledge-base)

**CLI Usage**:
```bash
cd appian-parser
python -m appian_parser dump <package.zip> <output_dir>
```

**Output Structure**:
```
output_dir/
├── app_overview.json       # Complete application map
├── search_index.json       # Fast object lookup
├── bundles/                # Functional bundles
│   └── <bundle_id>/
│       ├── structure.json  # Flow + relationships
│       └── code.json       # SAIL code
├── objects/                # Per-object dependencies
│   └── <uuid>.json
└── orphans/                # Unbundled objects
    ├── _index.json
    └── <uuid>.json
```

**Development**:
- Language: Python 3.10+
- Main entry: `appian_parser/__main__.py`
- Tests: `tests/`

---

### 2. gam-knowledge-base

**Purpose**: Data repository containing parsed Appian applications and the MCP server for querying them.

**Key Features**:
- Stores parsed application data in `data/` directory
- Provides MCP server (`appian-atlas`) for AI-powered querying
- Fetches data from GitHub at runtime (no local storage needed by users)

**Directory Structure**:
```
gam-knowledge-base/
├── data/                   # Parsed applications
│   ├── SourceSelection/
│   └── CaseManagementStudio/
├── mcp_server/             # MCP server package
│   ├── server.py           # FastMCP server implementation
│   └── pyproject.toml      # Package configuration
└── README.md               # User installation guide
```

**MCP Server**:
- Package name: `appian-atlas`
- Command: `appian-atlas --github ram-020998/gam-knowledge-base`
- Requires: `GITHUB_TOKEN` environment variable

**MCP Tools** (9 total):
1. `list_applications` - List all available apps
2. `get_app_overview` - Get complete app map
3. `search_bundles` - Find bundles by keyword
4. `search_objects` - Find objects by name
5. `get_bundle` - Load bundle with detail level (summary/structure/full)
6. `get_dependencies` - Get object dependency graph
7. `get_object_detail` - Get object info by UUID
8. `list_orphans` - List unbundled objects
9. `get_orphan` - Get orphan details

**Installation**:
```bash
pip install "appian-atlas @ git+https://github.com/ram-020998/gam-knowledge-base.git#subdirectory=mcp_server"
```

**Development**:
- Language: Python 3.10+
- Framework: FastMCP
- Main file: `mcp_server/server.py`

---

### 3. atlas-docs

**Purpose**: Public repository hosting the GitHub Pages site for Appian Atlas.

**Key Details**:
- Separated from `appian-atlas` to avoid GitHub Pages build failures caused by the private `appian-parser` submodule
- Contains manually maintained static HTML content
- Published at: https://ram-020998.github.io/atlas-docs/

**Directory Structure**:
```
atlas-docs/
├── index.html              # Landing page
└── installation.html       # Installation guide
```

**Development**:
- Edit HTML files directly
- Push to `main` branch — Pages deploys automatically

---

### 4. Powers (3 persona-specific repositories)

**Purpose**: Kiro Powers that provide persona-specific steering for exploring Appian applications.

#### power-appian-atlas-developer
- **Focus**: Technical implementation details
- **Shows**: UUIDs, SAIL code, dependencies, technical debt
- **Language**: Technical terminology (Expression Rules, CDTs, etc.)
- **URL**: `https://github.com/ram-020998/power-appian-atlas-developer`

#### power-appian-atlas-product-owner
- **Focus**: Business capabilities and workflows
- **Shows**: Plain language descriptions, user capabilities, business value
- **Language**: Business-friendly (no UUIDs or jargon)
- **URL**: `https://github.com/ram-020998/power-appian-atlas-product-owner`

#### power-appian-atlas-ux-designer
- **Focus**: User interfaces and interaction patterns
- **Shows**: SAIL UI components, user flows, interface layouts
- **Language**: UI/UX terminology
- **URL**: `https://github.com/ram-020998/power-appian-atlas-ux-designer`

**Structure** (each power):
```
power-appian-atlas-<persona>/
├── POWER.md        # Frontmatter + steering instructions
├── mcp.json        # MCP server configuration
└── README.md       # Installation guide
```

**MCP Configuration** (same for all):
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

---

## Development Workflows

### Adding a New Appian Application

1. **Parse the application**:
   ```bash
   cd appian-atlas/appian-parser
   source .venv/bin/activate
   python -m appian_parser dump <package.zip> ../gam-knowledge-base/data/<AppName>
   ```

2. **Commit to knowledge base**:
   ```bash
   cd ../gam-knowledge-base
   git add data/<AppName>
   git commit -m "Add <AppName> application"
   git push origin main
   ```

3. **Test with MCP server**:
   ```bash
   appian-atlas --github ram-020998/gam-knowledge-base
   # In Kiro, ask: "What applications are available?"
   ```

---

### Updating the Parser

1. **Make changes** in `appian-atlas/appian-parser/`
2. **Test locally**:
   ```bash
   cd appian-parser
   python -m appian_parser dump test_package.zip test_output/
   ```
3. **Update MCP server** if tool signatures changed:
   - Update `appian-parser/mcp_server/server.py`
   - Update `gam-knowledge-base/mcp_server/server.py`
4. **Commit and push**:
   ```bash
   git add -A
   git commit -m "Description of changes"
   git push origin main
   ```

---

### Updating a Power

1. **Edit POWER.md** in the specific power repository
2. **Test locally**:
   - In Kiro: Powers tab → Add Power from Local Path
   - Select the power directory
   - Test with various queries
3. **Commit and push**:
   ```bash
   cd power-appian-atlas-<persona>
   git add POWER.md
   git commit -m "Update steering instructions"
   git push origin main
   ```
4. **Users update**: They'll need to reinstall or update the power in Kiro

---

### Updating MCP Server

1. **Update server code**:
   - `gam-knowledge-base/mcp_server/server.py`
   - Also update `appian-parser/mcp_server/server.py` (keep in sync)

2. **Test locally**:
   ```bash
   cd gam-knowledge-base/mcp_server
   python server.py --github ram-020998/gam-knowledge-base
   ```

3. **Commit and push**:
   ```bash
   git add mcp_server/
   git commit -m "Update MCP server"
   git push origin main
   ```

4. **Users update**:
   ```bash
   pip install --upgrade "appian-atlas @ git+https://github.com/ram-020998/gam-knowledge-base.git#subdirectory=mcp_server"
   ```

---

## Key Design Principles

### 1. Separation of Concerns
- **Parser**: Pure parsing logic, no MCP dependencies
- **Knowledge Base**: Data storage + MCP server
- **Powers**: Persona-specific steering only

### 2. On-Demand Loading
- MCP server fetches data from GitHub at runtime
- Bundle structure allows loading summary → structure → full code progressively
- No need to download entire knowledge base locally

### 3. Persona-Specific Responses
- Same MCP tools, different steering instructions
- Developer sees technical details
- Product Owner sees business language
- UX Designer sees UI/UX focus

### 4. Bundle-Based Organization
- Each bundle represents a complete functional flow
- Entry points: actions, processes, pages, sites, dashboards, web APIs
- Transitive dependencies included in each bundle

---

## Common Tasks

### Rename MCP Server
If you need to rename the MCP server (e.g., from `gam-appian-kb` to `appian-atlas`):

1. Update `gam-knowledge-base/mcp_server/pyproject.toml` - `name` field
2. Update `gam-knowledge-base/mcp_server/server.py` - `FastMCP("name")`
3. Update `appian-parser/mcp_server/pyproject.toml` - `name` field
4. Update `appian-parser/mcp_server/server.py` - `FastMCP("name")`
5. Update all power `mcp.json` files - server name in `mcpServers`
6. Update all README files with new installation commands
7. Commit and push all repositories

### Add New MCP Tool
1. Add tool function in `gam-knowledge-base/mcp_server/server.py`
2. Add corresponding function in `appian-parser/mcp_server/server.py`
3. Update power POWER.md files to document the new tool
4. Test with all three personas to ensure appropriate responses

### Create New Persona Power
1. Create new repository: `power-appian-atlas-<persona>`
2. Copy `mcp.json` from existing power
3. Create `POWER.md` with:
   - Frontmatter (name, displayName, description, keywords)
   - Onboarding section
   - Steering instructions for the persona
   - Tool reference with persona-specific examples
4. Create `README.md` with installation instructions
5. Test locally, then push to GitHub

---

## GitHub Repositories

All repositories are under `ram-020998`:

- **appian-parser**: https://github.com/ram-020998/appian-parser (private)
- **atlas-docs**: https://github.com/ram-020998/atlas-docs (GitHub Pages site)
- **gam-knowledge-base**: https://github.com/ram-020998/gam-knowledge-base
- **power-appian-atlas**: https://github.com/ram-020998/power-appian-atlas (landing page)
- **power-appian-atlas-developer**: https://github.com/ram-020998/power-appian-atlas-developer
- **power-appian-atlas-product-owner**: https://github.com/ram-020998/power-appian-atlas-product-owner
- **power-appian-atlas-ux-designer**: https://github.com/ram-020998/power-appian-atlas-ux-designer

---

## Environment Setup

### Prerequisites
- Python 3.10+
- Git
- GitHub account with access to repositories
- GitHub Personal Access Token (for MCP server)

### Initial Setup
```bash
# Clone with all submodules
git clone --recurse-submodules git@github.com:ram-020998/appian-atlas.git
cd appian-atlas

# Set up parser environment
cd appian-parser
python3 -m venv .venv
source .venv/bin/activate
pip install -e .

# Set up MCP server
cd ../gam-knowledge-base
python3 -m venv .venv
source .venv/bin/activate
pip install -e mcp_server/

# Set GitHub token
export GITHUB_TOKEN="your_token_here"
```

---

## Testing

### Test Parser
```bash
cd appian-parser
source .venv/bin/activate
python -m appian_parser dump test_data/sample.zip test_output/
# Verify output structure
ls -la test_output/
```

### Test MCP Server
```bash
cd gam-knowledge-base
source .venv/bin/activate
appian-atlas --github ram-020998/gam-knowledge-base
# Server runs on stdio - appears to hang (normal)
# Test in Kiro IDE
```

### Test Powers
1. Open Kiro IDE
2. Powers tab → Add Power from Local Path
3. Select `power-appian-atlas-developer/`
4. Test queries:
   - "What applications are available?"
   - "Give me an overview of SourceSelection"
   - "How does the Add Vendors action work?"

---

## Troubleshooting

### Parser Issues
- **Import errors**: Ensure virtual environment is activated
- **Missing dependencies**: `pip install -e .` in appian-parser directory
- **Parse errors**: Check Appian package structure, review error logs

### MCP Server Issues
- **401 errors**: GitHub token not set or invalid
- **Server not found**: Check pip installation, verify PATH
- **Data not loading**: Verify GitHub repo access, check branch name

### Power Issues
- **Power not activating**: Check keywords in frontmatter
- **Wrong responses**: Review POWER.md steering instructions
- **MCP tools not available**: Verify MCP server is connected in Kiro

---

## Best Practices

1. **Keep MCP servers in sync**: Both `appian-parser` and `gam-knowledge-base` have MCP server code - keep them identical
2. **Test all personas**: When updating tools or data, test with all three persona powers
3. **Document changes**: Update README files when making significant changes
4. **Version control**: Commit frequently with descriptive messages
5. **Progressive loading**: Use bundle detail levels (summary → structure → full) to minimize data transfer
6. **Consistent naming**: Follow established naming conventions for objects, bundles, and tools
