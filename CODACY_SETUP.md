# Codacy Configuration Guide

This document explains the Codacy setup for the google-maps-timeline-viewer project.

## Overview

The project targets **Node.js 20+** with **ES Modules (ESM)**. This is configured in `package.json`:

```json
{
  "type": "module",
  "engines": {
    "node": ">=20.0.0"
  }
}
```

The Codacy configuration (`.codacy.yml`) aligns linting rules with this target.

## Configuration Files

### `.codacy.yml`
The primary Codacy configuration file that:
- Disables ES5 compatibility checks (not applicable for Node 20+)
- Enables ES2020+ features (const, arrow functions, async/await, etc.)
- Targets Node.js environment
- Disables overly strict rules that conflict with modern JavaScript

### `.eslintrc.json`
ESLint configuration for local development:
- Targets ES2020+ features
- Configured as ESM (`"sourceType": "module"`)
- Validates against undefined globals
- Warns on code style issues

### `.eslintignore`
Files and directories to exclude from linting:
- node_modules
- Build artifacts (dist, build, coverage)
- Environment files

## Key Decisions

### Why Disable ES5 Compatibility Checks?

The project requires Node.js 20+, which fully supports:
- ES2015+ features (const/let, arrow functions, classes)
- ES2017+ features (async/await)
- ES2018+ features (rest/spread properties)
- ES2020+ features (optional chaining, nullish coalescing)

Enforcing ES5-only compatibility would:
- Significantly reduce code quality
- Prevent use of modern, safer patterns
- Require verbose workarounds
- Contradict the project's stated requirements

### Why Allow null/undefined?

JavaScript requires null/undefined for:
- Optional database values
- API responses with missing fields
- Conditional logic and error handling

Disallowing these would be impractical.

### Why Allow Async/Await in Loops?

For this project, sequential async operations (with explicit await in loops) are intentional for:
- Rate-limited API calls
- Database transactions
- Prefetch operations that need ordering

ESLint's warning is still enabled to draw attention to the pattern.

## Customizing Rules

To modify Codacy behavior:

1. **Update `.codacy.yml`**:
   - Add/remove rules from `disabled-rules`
   - Adjust quality levels
   - Enable/disable specific tools

2. **Update `.eslintrc.json`**:
   - Modify rule severity levels
   - Add new rules
   - Adjust environment settings

3. **Commit changes** and Codacy will apply them on next scan

## Tool-Specific Notes

### ESLint
- Checks code quality and style
- Catches potential bugs
- Validates against runtime globals

### PMD
- Disabled by default (too strict for Node.js)
- Can be re-enabled for specific Java/XML linting if needed

### Trivy
- Scans for security vulnerabilities
- Checks dependencies for known CVEs
- Enabled for production safety

## Validation

Verify the configuration with:

```bash
# Validate YAML syntax
python3 -c "import yaml; yaml.safe_load(open('.codacy.yml'))"

# Check ESLint config
npx eslint --print-config . | head -50

# Run local linting
npm run lint  # or: npx eslint .
```

## References

- [Codacy Configuration Documentation](https://docs.codacy.com/administration/codacy-configuration-file/)
- [ESLint Configuration](https://eslint.org/docs/user-guide/configuring/)
- [Node.js ECMAScript Compatibility](https://node.green/)
- [ECMAScript 2020+ Features](https://github.com/tc39/proposals)
