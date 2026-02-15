# Appian Atlas

AI-powered system for parsing, storing, and exploring Appian applications.

## Quick Start

This is the parent repository that aggregates all Appian Atlas sub-repositories as Git submodules:

```
appian-atlas/
├── appian-parser/                      # Appian application parser (private)
├── atlas-docs/                         # GitHub Pages site (public)
├── gam-knowledge-base/                 # Data + MCP server
├── power-appian-atlas/                 # Powers landing page
├── power-appian-atlas-developer/       # Developer persona
├── power-appian-atlas-product-owner/   # Product Owner persona
├── power-appian-atlas-ux-designer/     # UX Designer persona
└── power-appian-reference/             # SAIL language reference & best practices
```

### Clone with all submodules
```bash
git clone --recurse-submodules git@github.com:ram-020998/appian-atlas.git
```

### If already cloned, initialize submodules
```bash
git submodule update --init --recursive
```

## Documentation

See `.kiro/appian-atlas-project.md` for complete documentation including:
- Repository structure and purpose
- Development workflows
- Common tasks
- Testing procedures
- Troubleshooting guide

## Quick Links

- **Knowledge Base**: [gam-knowledge-base/README.md](gam-knowledge-base/README.md)
- **Powers**: [power-appian-atlas/README.md](power-appian-atlas/README.md)
- **Website**: https://ram-020998.github.io/atlas-docs/

## GitHub Repositories

All under `ram-020998`:
- https://github.com/ram-020998/appian-parser (private)
- https://github.com/ram-020998/atlas-docs (GitHub Pages site)
- https://github.com/ram-020998/gam-knowledge-base
- https://github.com/ram-020998/power-appian-atlas
- https://github.com/ram-020998/power-appian-atlas-developer
- https://github.com/ram-020998/power-appian-atlas-product-owner
- https://github.com/ram-020998/power-appian-atlas-ux-designer
- https://github.com/ram-020998/power-appian-reference
