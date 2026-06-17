# Debugging and Tracing

> **Goal:** Know how to add debug logging to the parse pipeline, inspect intermediate states, use `extras.document`, and interpret common CI failure patterns.

---

## Adding Debug Logging to the Pipeline

The parser does not have a built-in debug mode, but you can add `console.log` statements to the source files during debugging. Since Jest compiles TypeScript via `@swc/jest`, you don't need to rebuild for unit test debugging — but for scratch scripts you must rebuild.

**Strategy:** Add temporary logging to `src/parse.ts` at each stage:

```typescript
// src/parse.ts (temporary debug addition)
export async function parse(...) {
  // ...
  const { validated, diagnostics, extras } = await validate(...);
  
  // DEBUG: what did Spectral produce?
  console.log('[DEBUG parse.ts] validated defined:', validated !== undefined);
  console.log('[DEBUG parse.ts] diagnostics:', diagnostics.map(d => d.code));
  
  if (validated === undefined) {
    return { document: undefined, diagnostics, extras };
  }

  const validatedDoc = copy(validated as Record<string, any>);
  
  // DEBUG: what does the resolved JSON look like?
  console.log('[DEBUG parse.ts] validatedDoc keys:', Object.keys(validatedDoc));
  console.log('[DEBUG parse.ts] asyncapi version:', validatedDoc.asyncapi);
  
  applyUniqueIds(validatedDoc);
  // ...
}
```

Remember to remove debug logs before committing.

---

## Inspecting `extras.document`

`parse()` returns `extras.document` — the raw Spectral `Document` object before model wrapping. This is useful for:

### 1. Seeing the fully resolved JSON

```js
const { document, diagnostics, extras } = await parser.parse(asyncapiDoc);

// The resolved (all $refs inlined) JavaScript object
const resolved = extras?.document.data;
console.log(JSON.stringify(resolved, null, 2).slice(0, 2000));
```

### 2. Getting range info for a JSON path

```js
const range = extras?.document.getRangeForJsonPath(['channels', 'user/registered', 'publish'], true);
console.log('Location:', range);
// → { start: { line: 10, character: 4 }, end: { line: 20, character: 0 } }
```

### 3. Accessing the document inventory (resolved refs map)

```js
// __documentInventory is set by the asyncapi-internal rule
const inventory = (extras?.document as any).__documentInventory;
// inventory tracks which $refs were resolved and to what
```

---

## Inspecting Intermediate Parsing States

### After `validate()` (pre-model stage)

```js
// src/validate.ts
const { validated, diagnostics, extras } = await validate(parser, spectral, asyncapi, options);

// validated = fully resolved JSON object OR undefined (if errors blocked it)
// diagnostics = all Spectral diagnostic results
// extras.document = raw Spectral Document
```

To isolate validation from the rest of the pipeline, call `validate()` directly:

```js
const { Parser } = require('./packages/parser/cjs/index.js');

// Access the internal validate function (not exported publicly)
// Best approach: add a temporary export to validate.ts
// Or use the parser.validate() public method and compare with parse() results

const parser = new Parser();
const validateDiags = await parser.validate(asyncapiDoc);
const { document, diagnostics: parseDiags } = await parser.parse(asyncapiDoc);

// Compare: parseDiags includes all validate diagnostics plus any from custom ops
console.log('Validate only diags:', validateDiags.length);
console.log('Parse diags:', parseDiags.length);
```

### After `applyUniqueIds`

To see what unique IDs were assigned, add a log after `applyUniqueIds` in `src/parse.ts`:

```typescript
applyUniqueIds(validatedDoc);
console.log('[DEBUG] channels after applyUniqueIds:', Object.keys(validatedDoc.channels ?? {}));
const firstChannel = Object.values(validatedDoc.channels ?? {})[0] as any;
console.log('[DEBUG] first channel x-parser-unique-object-id:', firstChannel?.['x-parser-unique-object-id']);
```

### After each custom operation

Add logs in `src/custom-operations/index.ts`:

```typescript
checkCircularRefs(document);
console.log('[DEBUG] after checkCircularRefs:', document.json()['x-parser-circular']);

if (options.applyTraits) {
  applyTraitsV2(detailed.parsed as v2.AsyncAPIObject);
  console.log('[DEBUG] traits applied');
}

if (options.parseSchemas) {
  await parseSchemasV2(parser, detailed);
  console.log('[DEBUG] schemas parsed');
}
```

---

## Running a Failing Test with Verbose Output

```bash
# Run a single test file with full verbose output
npx jest parse.spec.ts --verbose --rootdir packages/parser

# Run a specific test case
npx jest parse.spec.ts -t "should parse valid document" --verbose --rootdir packages/parser

# Run with increased timeout (useful for async tests)
npx jest parse.spec.ts --testTimeout=30000 --rootdir packages/parser

# Run and see console output (normally suppressed)
npx jest parse.spec.ts --verbose --no-silent --rootdir packages/parser
```

---

## Inspecting What Spectral Does Internally

Spectral has its own internal state. To see what rules run and in what order:

```js
// scratch/debug-spectral.js
const { Parser } = require('./packages/parser/cjs/index.js');
const { Spectral } = require('@stoplight/spectral-core');

// Create a parser and access its spectral instance
const parser = new Parser();

// Access the private spectral instance (for debugging only)
const spectral = (parser as any).spectral;

// See what rules are registered
const ruleset = spectral.ruleset;
console.log('Rules:', Object.keys(ruleset.rules));
```

---

## Common CI Failure Patterns

### Pattern 1: "Unknown schema format"

```
[Error] asyncapi2-schemas: Unknown schema format: "application/vnd.apache.avro+json;version=1.9.0"
```

**Cause:** Test or scenario uses Avro format without registering the Avro parser.  
**Fix:** Add `AvroSchemaParser()` to the parser or look at whether the test fixture should use JSON Schema instead.

### Pattern 2: "Cannot find module 'parserapiv1'"

```
Error: Cannot find module 'parserapiv1'
```

**Cause:** `packages/multi-parser` dependencies not installed, or `npm install` wasn't run.  
**Fix:** Run `npm install` from the repo root.

### Pattern 3: Jest "out of memory"

```
FATAL ERROR: CALL_AND_RETRY_LAST Allocation failed - JavaScript heap out of memory
```

**Cause:** A test is creating an infinitely recursive structure (likely a missed circular ref guard).  
**Fix:** Run `node --max-old-space-size=4096 node_modules/.bin/jest` to give more memory, find the leaking test.

### Pattern 4: "Cannot convert circular structure to JSON"

```
TypeError: Converting circular structure to JSON
```

**Cause:** Someone called `JSON.stringify()` directly on a parsed document with circular schemas.  
**Fix:** Use `stringify()` from `@asyncapi/parser` instead.

### Pattern 5: Browser test fails with "asyncapi is not defined"

**Cause:** The webpack bundle didn't build before the browser test ran.  
**Fix:** Ensure `npm run parser:build:browser` runs before the browser test.

### Pattern 6: ESM/CJS module resolution errors

```
require() of ES Module .../node_modules/nimma/... not supported
```

**Cause:** A dependency uses ESM-only exports. The `jest.config.ts` has `moduleNameMapper` for known cases.  
**Fix:** Add the offending package to `moduleNameMapper` in `packages/parser/jest.config.ts`.

---

## Reading Test Fixture Files

When a test fails with an unexpected diagnostic or model state, check the fixture YAML:

```bash
# View a specific fixture
cat packages/parser/test/mocks/simple.yaml
cat packages/parser/test/mocks/circular-refs.yaml

# Find which tests use a fixture
grep -r "simple.yaml" packages/parser/test/
grep -r "circular-refs" packages/parser/test/
```

---

## Using `--runInBand` for Sequential Debugging

By default Jest runs tests in parallel. For debugging, run sequentially:

```bash
npx jest parse.spec.ts --runInBand --rootdir packages/parser
```

This ensures `console.log` output appears in the right order relative to test output.

---

## Next Step

Read [03-adding-new-validation-rules.md](./03-adding-new-validation-rules.md) for a step-by-step guide to contributing a new validation rule.
