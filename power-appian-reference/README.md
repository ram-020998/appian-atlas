# Appian Development Reference Power

A comprehensive SAIL language reference and Appian development best practices power for Kiro CLI.

## Overview

This power provides complete SAIL language documentation including grammar, functions, common mistakes, and best practices. It is designed to be queried by other Appian Atlas powers (Developer, Product Owner, UX Designer) when generating or reviewing SAIL code.

## What's Included

### SAIL Language Reference
- **Complete SAIL Grammar** - Full BNF grammar with syntax rules, operators, and type system
- **Common Functions** - Quick reference for 50+ most-used SAIL functions with examples
- **Common Mistakes** - Critical anti-patterns and errors to avoid

### Appian Best Practices
- **Design Best Practices** - Solutions design principles, SAIL coding standards, naming conventions, code reusability, interface patterns, performance optimization
- **Accessibility Guide** - A11Y guidelines for Appian components with proper labeling, ARIA attributes, and screen reader support

## Installation

### From Local Path
1. Clone this repository
2. Open Kiro → Powers panel → **Add power from Local Path**
3. Select the `power-appian-reference` directory

### From GitHub (Once Published)
```bash
# In Kiro Powers panel
Add power from GitHub → https://github.com/ram-020998/power-appian-reference
```

## Usage

This power is designed to be queried via **subagent delegation** from other powers.

### Example Queries

From Developer Power:
```
"Query power-appian-reference: How do I format dates in SAIL?"
"Query power-appian-reference: Show me array manipulation functions"
"Query power-appian-reference: What are common SAIL mistakes with local variables?"
```

### Integration with Appian Atlas

This power complements the Appian Atlas ecosystem:

- **Appian Atlas MCP Server** - Queries actual application data (objects, bundles, dependencies)
- **This Power** - Provides SAIL language knowledge and best practices
- **Persona Powers** - Use both to generate correct, well-structured SAIL code

## Structure

```
power-appian-reference/
├── POWER.md                          # Main power configuration
├── README.md                         # This file
└── steering/
    ├── sail-grammar.md               # Complete SAIL BNF grammar (157KB)
    ├── sail-common-functions.md      # Top 50 functions with examples
    ├── sail-common-mistakes.md       # Critical anti-patterns
    ├── appian-design-best-practices.md  # Solutions design best practices (66KB)
    └── appian-accessibility-guide.md    # A11Y guidelines (9KB)
```

## Steering Files

The power uses focused steering files for efficient context loading:

| File | Purpose | When to Load |
|------|---------|--------------|
| `sail-grammar.md` | Complete BNF grammar | Syntax questions, grammar rules |
| `sail-common-functions.md` | Function reference | Function usage, examples |
| `sail-common-mistakes.md` | Anti-patterns | Code review, error debugging |
| `appian-design-best-practices.md` | Design standards | Code structure, naming, patterns, performance |
| `appian-accessibility-guide.md` | A11Y guidelines | Accessibility requirements, component labeling |

## Contributing

To add Appian best practices documentation:

1. Create new steering file in `steering/` directory
2. Update `POWER.md` with steering file mapping
3. Submit pull request

## Related Projects

- [appian-atlas](https://github.com/ram-020998/appian-atlas) - Main repository
- [appian-parser](https://github.com/ram-020998/appian-parser) - Appian application parser
- [gam-knowledge-base](https://github.com/ram-020998/gam-knowledge-base) - Data + MCP server
- [power-appian-atlas-developer](https://github.com/ram-020998/power-appian-atlas-developer) - Developer persona
- [power-appian-atlas-product-owner](https://github.com/ram-020998/power-appian-atlas-product-owner) - Product Owner persona
- [power-appian-atlas-ux-designer](https://github.com/ram-020998/power-appian-atlas-ux-designer) - UX Designer persona

## License

MIT

## Version

1.0.0 - Initial release with SAIL language reference
