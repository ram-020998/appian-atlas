---
name: "appian-reference"
displayName: "Appian Development Reference"
description: "Complete SAIL language grammar, functions, and Appian development best practices for generating correct SAIL code"
keywords: ["sail", "appian", "expression", "function", "syntax", "interface", "component", "rule", "record", "process", "a!"]
---

# Appian Development Reference Power

This power provides comprehensive SAIL language reference and Appian development best practices. It is designed to be queried by other powers (Developer, Product Owner, UX Designer) when generating or reviewing SAIL code.

## Purpose

This is a **reference-only power** - it does not modify code directly. Other powers should use subagent delegation to query this power for:
- SAIL syntax and grammar rules
- Function signatures and usage
- Common mistakes and anti-patterns
- Appian best practices

## When to Use This Power

Load this power when you need to:
- Generate SAIL expressions or interface code
- Validate SAIL syntax
- Look up function signatures and parameters
- Understand SAIL type system and operators
- Avoid common SAIL mistakes
- Apply Appian development best practices

## How to Query This Power

When another power needs SAIL reference information, use subagent delegation with specific queries:

**Examples:**
- "What is the syntax for date formatting in SAIL?"
- "Show me text manipulation functions"
- "How do I use a!localVariables correctly?"
- "What are common SAIL mistakes to avoid?"
- "Show me UI component patterns"

## Steering Files

This power uses focused steering files for efficient context loading:

### SAIL Language Reference
- **sail-grammar.md** - Complete SAIL BNF grammar, type system, operators, precedence rules
- **sail-common-functions.md** - Most frequently used SAIL functions with examples
- **sail-common-mistakes.md** - Critical anti-patterns and common errors to avoid

### Appian Best Practices
- **appian-design-best-practices.md** - Solutions design best practices: SAIL principles, code reusability, naming conventions, interface patterns, performance optimization
- **appian-accessibility-guide.md** - Accessibility (A11Y) guidelines for Appian components: proper labeling, ARIA attributes, keyboard navigation, screen reader support

## Response Guidelines

When responding to queries:
1. **Be specific** - Return exact function signatures and examples
2. **Include context** - Explain parameter types and return values
3. **Show examples** - Provide working SAIL code snippets
4. **Highlight mistakes** - Point out common errors related to the query
5. **Be concise** - Return only relevant information, not entire sections

## Integration with Appian Atlas

This power complements the Appian Atlas MCP server:
- **Atlas MCP** - Queries actual application data (objects, bundles, dependencies)
- **This Power** - Provides SAIL language knowledge and best practices

Use both together for complete Appian development assistance.
