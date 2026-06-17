# Use Case: Validate Only (No Model)

> **Goal:** Use `parser.validate()` for CI linting, pre-commit hooks, and quality gates — without building a typed model tree.

---

## When to Use `validate()` Instead of `parse()`

| Scenario | Use |
|----------|-----|
| CI lint step: "is this AsyncAPI doc valid?" | `validate()` |
| Pre-commit hook | `validate()` |
| Need to traverse the document | `parse()` |
| Generating code from the document | `parse()` |
| Checking if a doc is valid before doing expensive processing | `validate()` |

`validate()` is faster because it skips the model construction and all custom operations (traits, schema parsing, etc.).

---

## Example 1: Basic Validation

Create `scratch/use-case-validate.js`:

```js
// scratch/use-case-validate.js
const { Parser } = require('../packages/parser/cjs/index.js');

async function main() {
  const parser = new Parser();

  // Document with multiple intentional problems
  const doc = `
asyncapi: '2.6.0'
info:
  version: '1.0.0'
  # MISSING: title (required field)
channels:
  user/registered:
    publish:
      operationId: onUserRegistered
      message:
        payload:
          type: object
          properties:
            userId:
              type: string
  notification/sent:
    publish:
      operationId: onUserRegistered
      # DUPLICATE operationId — should be unique
      message:
        payload:
          type: string
          minimum: 0
          # INVALID: minimum is not valid for type:string in JSON Schema
`;

  const diagnostics = await parser.validate(doc);

  console.log('Total diagnostics:', diagnostics.length);
  console.log();

  // Group by severity
  const bySeverity = {
    0: diagnostics.filter(d => d.severity === 0),  // Error
    1: diagnostics.filter(d => d.severity === 1),  // Warning
    2: diagnostics.filter(d => d.severity === 2),  // Info
    3: diagnostics.filter(d => d.severity === 3),  // Hint
  };

  const severityNames = ['Error', 'Warning', 'Info', 'Hint'];

  [0, 1, 2, 3].forEach(sev => {
    if (bySeverity[sev].length > 0) {
      console.log(`--- ${severityNames[sev]}s (${bySeverity[sev].length}) ---`);
      bySeverity[sev].forEach(d => {
        const pathStr = d.path.join(' → ') || '(root)';
        console.log(`  [${d.code}]`);
        console.log(`    Message: ${d.message}`);
        console.log(`    Path: ${pathStr}`);
        console.log(`    Location: line ${d.range.start.line + 1}`);
        console.log();
      });
    }
  });

  // Exit with error code if there are errors (useful for CI)
  const hasErrors = bySeverity[0].length > 0;
  if (hasErrors) {
    console.log('Validation FAILED');
    process.exit(1);
  } else {
    console.log('Validation PASSED (warnings present but not errors)');
  }
}

main().catch(console.error);
```

Run:

```bash
node scratch/use-case-validate.js
echo "Exit code: $?"
```

Expected output:

```
Total diagnostics: 8

--- Errors (2) ---
  [asyncapi-document-unresolved]
    Message: Object must have required property "title"
    Path: info
    Location: line 3

  [asyncapi2-operation-operationId-uniqueness]
    Message: "onUserRegistered" operationId must be unique across all the operations.
    Path: channels → notification/sent → publish → operationId
    Location: line 18

--- Warnings (4) ---
  [asyncapi-id] ...
  [asyncapi-defaultContentType] ...
  [asyncapi-info-description] ...
  [asyncapi-servers] ...

--- Infos (1) ---
  [asyncapi-latest-version] ...

Validation FAILED
Exit code: 1
```

---

## Example 2: CI-Style Validation Script

A minimal script suitable for use in CI pipelines or npm scripts:

Create `scratch/ci-validate.js`:

```js
// scratch/ci-validate.js
// Usage: node ci-validate.js <path-to-asyncapi.yaml>

const { Parser } = require('../packages/parser/cjs/index.js');
const fs = require('fs');
const path = require('path');

async function main() {
  const filePath = process.argv[2];
  if (!filePath) {
    console.error('Usage: node ci-validate.js <path-to-asyncapi.yaml>');
    process.exit(2);
  }

  const absolutePath = path.resolve(filePath);
  const content = fs.readFileSync(absolutePath, 'utf-8');

  const parser = new Parser();
  const diagnostics = await parser.validate(content, { source: absolutePath });

  const errors = diagnostics.filter(d => d.severity === 0);
  const warnings = diagnostics.filter(d => d.severity === 1);

  if (errors.length > 0) {
    console.error(`\n✗ ${errors.length} error(s) found:\n`);
    errors.forEach(d => {
      console.error(`  Line ${d.range.start.line + 1}: [${d.code}] ${d.message}`);
    });
    if (warnings.length > 0) {
      console.warn(`\n  (${warnings.length} warning(s) also present)`);
    }
    process.exit(1);
  }

  if (warnings.length > 0) {
    console.warn(`\n⚠ ${warnings.length} warning(s):`);
    warnings.forEach(d => {
      console.warn(`  Line ${d.range.start.line + 1}: [${d.code}] ${d.message}`);
    });
  }

  console.log('\n✓ AsyncAPI document is valid.');
  process.exit(0);
}

main().catch(err => {
  console.error('Unexpected error:', err);
  process.exit(2);
});
```

Test with the streetlights document:

```bash
node scratch/ci-validate.js scratch/streetlights-v2.yaml
```

---

## Example 3: Custom Ruleset

Disable specific rules or the entire recommended ruleset:

```js
// scratch/use-case-custom-ruleset.js
const { Parser } = require('../packages/parser/cjs/index.js');

async function main() {
  // Disable all recommended rules — only run core (structure) validation
  const strictParser = new Parser({
    ruleset: {
      recommended: false,
    }
  });

  const doc = `
asyncapi: '2.6.0'
info:
  title: Minimal API
  version: '1.0.0'
channels: {}
`;

  const diagnostics = await strictParser.validate(doc);
  
  // With recommended: false, no warnings about missing 'id', 'servers', etc.
  console.log('Diagnostics with recommended: false:');
  diagnostics.forEach(d => {
    const sev = ['Error', 'Warning', 'Info', 'Hint'][d.severity];
    console.log(`  [${sev}] ${d.code}`);
  });

  // Only asyncapi-latest-version (info) and maybe asyncapi-is-asyncapi remain
}

main().catch(console.error);
```

---

## Example 4: Making Warnings Also Block

By default only errors block validation. Use `allowedSeverity` to also fail on warnings:

```js
const diagnostics = await parser.validate(doc, {
  allowedSeverity: {
    error: false,    // errors block (default)
    warning: false,  // warnings NOW also block
    info: true,
    hint: true,
  }
});
```

This is useful for enforcing stricter standards in a CI environment.

---

## How to Add Validation to Your npm Scripts

In `package.json`:

```json
{
  "scripts": {
    "validate:asyncapi": "node scripts/validate-asyncapi.js asyncapi.yaml"
  }
}
```

Or with the AsyncAPI CLI (which also uses this parser under the hood):

```bash
npm install --save-dev @asyncapi/cli
npx asyncapi validate asyncapi.yaml
```

---

## Understanding Diagnostic Codes

Common codes you will see:

| Code | Severity | What it means |
|------|---------|---------------|
| `asyncapi-is-asyncapi` | Error | `asyncapi` field missing or not a supported version |
| `asyncapi-document-unresolved` | Error | Missing required field in the raw document |
| `asyncapi-document-resolved` | Error | Missing required field after ref resolution |
| `asyncapi2-operation-operationId-uniqueness` | Error | Duplicate `operationId` |
| `asyncapi2-channel-parameters` | Warning | Channel parameter referenced but not defined |
| `asyncapi-latest-version` | Info | Not using the latest AsyncAPI version |
| `asyncapi-servers` | Warning | No servers defined |
| `asyncapi-id` | Warning | No `id` field in the document |
| `asyncapi-unused-component` | Info | Component defined but never `$ref`'d |

---

## Next Step

Read [03-working-with-refs.md](./03-working-with-refs.md) to understand how `$ref` resolution works and how to work with documents that use internal and external references.
