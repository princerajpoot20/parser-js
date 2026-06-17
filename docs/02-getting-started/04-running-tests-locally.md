# Running Tests Locally

> **Goal:** Know how to run the full test suite, individual files, and specific tests. Understand what each test category covers and how to iterate quickly during development.

---

## Test Suite Overview

| Suite | Command | Count | What it covers |
|-------|---------|-------|---------------|
| Parser unit | `npm run parser:test:unit` | ~163 files | Everything except browser |
| Parser browser | `npm run parser:test:browser` | 1 file | Browser bundle via Playwright |
| Multi-parser unit | `npm run multi-parser:test:unit` | 2 files | Multi-version wrapper |
| All (build + test) | `npm test` | All | Full CI equivalent |

---

## Running the Full Suite

```bash
# Full CI-equivalent run: build + all tests
npm test
```

This is equivalent to what CI runs. It:
1. Builds all packages (parser ESM, CJS, browser; multi-parser ESM, CJS)
2. Runs parser unit tests
3. Installs Playwright Chromium (if not installed)
4. Runs parser browser tests
5. Runs multi-parser tests

Expected duration: **2–5 minutes** on a modern machine.

---

## Running Unit Tests Only (Fastest)

For day-to-day development, skip the build step and browser tests:

```bash
npm run parser:test:unit
```

This directly runs Jest on the TypeScript source (via `@swc/jest` transform). No build needed.

Expected duration: **20–40 seconds**.

---

## Running a Single Test File

Use Jest's `--testPathPattern` flag to run one spec file:

```bash
# From the repo root:
npx jest --testPathPattern="parse.spec.ts" --rootdir packages/parser

# Or navigate to the package first:
cd packages/parser
npx jest parse.spec.ts
```

Common single-file runs:

```bash
# Core parsing
npx jest parse.spec.ts --rootdir packages/parser

# Validation only
npx jest validate.spec.ts --rootdir packages/parser

# Reference resolution
npx jest resolver.spec.ts --rootdir packages/parser

# A specific model
npx jest "models/v2/channel.spec.ts" --rootdir packages/parser

# A specific validation rule
npx jest "asyncapi2-operation-operationId.spec.ts" --rootdir packages/parser

# Traits custom operation
npx jest "custom-operations/apply-traits" --rootdir packages/parser
```

---

## Running a Single Test by Name

Use `-t` (or `--testNamePattern`) to run one specific `it()` or `test()` block:

```bash
cd packages/parser
npx jest parse.spec.ts -t "should parse valid document"
```

The pattern is matched as a substring (regex), so partial names work:

```bash
npx jest parse.spec.ts -t "circular"
```

---

## Running Tests with Coverage

```bash
cd packages/parser
npx jest --coverage --testPathIgnorePatterns=test/browser/*
```

Coverage is collected from `src/**`. After the run, an HTML report is at `packages/parser/coverage/lcov-report/index.html`.

---

## Running Browser Tests

Browser tests require Playwright and a browser. They are slower (~30 seconds) and are usually only needed when:
- Modifying the webpack configuration
- Changing browser-specific polyfills
- Checking that a new feature works in the browser bundle

```bash
npm run parser:test:browser
```

This script:
1. Runs `playwright install chromium` (one-time setup)
2. Builds the browser bundle (`webpack`)
3. Runs `jest ./test/browser/*`

The browser test starts a local HTTP server on port 8080 and uses Playwright to run assertions in a real Chromium browser.

---

## Running Multi-Parser Tests

```bash
npm run multi-parser:test:unit
```

Only 2 test files: `parse.spec.ts` and `convert.spec.ts`. These test:
- Selecting the correct parser version (`NewParser(1)`, `NewParser(3)`)
- Converting documents between parser API versions

---

## How to Read Jest Output

A passing run:

```
 PASS  test/parse.spec.ts (4.2s)
   parse()
     ✓ should parse valid document (203ms)
     ✓ should not parse valid v3 document (88ms)
     ✓ should return diagnostics for invalid document (55ms)
     ...

Test Suites: 163 passed, 163 total
Tests:       1247 passed, 1247 total
Snapshots:   0 total
Time:        28.4s
```

A failing run:

```
 FAIL  test/parse.spec.ts
   parse()
     ✗ should parse valid document (205ms)
       ● Expected AsyncAPIDocumentV2, received undefined
         
         expect(received).toBeInstanceOf(expected)
         
         Expected: AsyncAPIDocumentV2
         Received: undefined
         
           at Object.<anonymous> (test/parse.spec.ts:25:5)
```

Key columns:
- `PASS` / `FAIL` — file-level result
- `✓` / `✗` — individual test result
- Time in parentheses — how long the test took
- Stack trace shows the exact line in the test file

---

## Test Categories Explained

### Core tests (`test/*.spec.ts`)

End-to-end tests using the full `Parser` class. These are the closest to real-world usage:

| File | What it tests |
|------|--------------|
| `parse.spec.ts` | `Parser.parse()` — valid/invalid docs, options, v2 vs v3 |
| `validate.spec.ts` | `Parser.validate()` — severity gating, allowedSeverity option |
| `parser.spec.ts` | `Parser` constructor, `registerSchemaParser()` |
| `from.spec.ts` | `fromURL()`, `fromFile()` — network/filesystem access |
| `resolver.spec.ts` | `$ref` resolution — internal, file, HTTP, custom protocols |
| `stringify.spec.ts` | `stringify()` / `unstringify()` round-trips |
| `iterator.spec.ts` | Schema iteration utilities |
| `document.spec.ts` | `createAsyncAPIDocument()`, type guards |
| `spectral.spec.ts` | Spectral instance creation, custom rulesets |

### Model tests (`test/models/`)

Unit tests that construct model instances directly without going through `Parser.parse()`. Each model class has its own spec file.

```bash
# Test all v2 models
npx jest "test/models/v2" --rootdir packages/parser

# Test a specific v2 model
npx jest "test/models/v2/channel.spec.ts" --rootdir packages/parser
```

### Ruleset tests (`test/ruleset/rules/`)

Each Spectral rule has its own test file. Tests use the `testRule()` helper which enables one rule at a time:

```bash
# Test all rules
npx jest "test/ruleset" --rootdir packages/parser

# Test a specific rule
npx jest "asyncapi2-channel-servers.spec.ts" --rootdir packages/parser
```

### Custom operation tests (`test/custom-operations/`)

Tests for the post-parse pipeline (traits, unique IDs, schema parsing, circular refs):

```bash
npx jest "test/custom-operations" --rootdir packages/parser
```

### Old API tests (`test/old-api/`)

Tests for the legacy API compatibility layer:

```bash
npx jest "test/old-api" --rootdir packages/parser
```

---

## YAML Test Fixtures

The `test/mocks/` folder contains YAML AsyncAPI documents used by multiple tests:

| File | Purpose |
|------|---------|
| `simple.yaml` | Basic v2 doc, used in resolver tests |
| `simple-message.yaml` | Standalone message, used as an external ref target |
| `simple-with-refs.yaml` | Multi-level external refs chain |
| `refs-1.yaml`, `refs-2.yaml` | Chained ref targets |
| `nested-schemas.yaml` | Deeply nested schema structures |
| `circular-refs.yaml` | Document with circular `$ref` |
| `parse/circular-ref.yaml` | Circular ref used in parse tests |
| `parse/circular-ref-deep.yaml` | Deep circular ref chain |

---

## Common Issues

### `Cannot find module '@asyncapi/parser'`

You need to run a build before some tests. Try:

```bash
npm run parser:build
```

### `jest: command not found`

Use `npx jest` instead of `jest`.

### ESM/CJS resolution errors with `nimma`

The `jest.config.ts` already has `moduleNameMapper` for this. If you see these errors, check that `jest.config.ts` is being used (not a custom config override).

### Browser tests fail: `Chromium not found`

Run `npx playwright install chromium` manually.

### Tests time out (10s default)

If a test fetches from a real URL (like `from.spec.ts`), it may time out in a network-restricted environment. CI has full network access; local may not. Use `--testTimeout=30000` to extend:

```bash
npx jest from.spec.ts --testTimeout=30000 --rootdir packages/parser
```

---

## Watching Mode During Development

Run tests automatically when files change:

```bash
cd packages/parser
npx jest --watch
```

Then press:
- `p` — filter by filename
- `t` — filter by test name
- `a` — run all tests
- `q` — quit

---

## Next Step

You are ready to dive into the architecture. Read [Chapter 3: Architecture](../03-architecture/01-high-level-architecture.md) starting with the high-level system diagram.
