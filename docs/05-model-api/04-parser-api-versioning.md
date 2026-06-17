# Parser API Versioning

> **Goal:** Understand the difference between AsyncAPI spec versions and Parser API versions, why multiple parser API versions coexist, and how `@asyncapi/multi-parser` handles this.

---

## Two Different Version Numbers

This is a common source of confusion:

| Concept | What it versions | Current | Example |
|---------|-----------------|---------|---------|
| **AsyncAPI spec version** | The YAML document format | 3.0.0 | `asyncapi: '2.6.0'` |
| **Parser API version** | The TypeScript/JavaScript API for navigating parsed documents | v3 (Intent API) | `x-parser-api-version: 3` |
| **`@asyncapi/parser` npm version** | The npm package release | 3.6.0 | `"@asyncapi/parser": "^3.6.0"` |

They are independent:
- `@asyncapi/parser` v3.6.0 (npm package) implements Parser-API v3 (Intent API) and supports AsyncAPI spec 2.x and 3.x documents.

---

## Parser API History

The Parser-API is a separate specification maintained at [github.com/asyncapi/parser-api](https://github.com/asyncapi/parser-api). It defines the interface contracts that model classes must implement.

| Parser-API version | What changed |
|--------------------|-------------|
| v0 (old-api) | Original model API — direct object property access, channel-centric, no `Collection` classes |
| v1 | (`parserapiv1` package) — intermediate version |
| v2 | (`parserapiv2` package) — another intermediate |
| v3 (current) | Intent API — `BaseModel`, `Collection<T>`, `operations()` at top level for v3 spec, full TypeScript types, circular-ref-safe |

Most tools today should use Parser-API v3 (the default from `@asyncapi/parser`).

---

## Why `@asyncapi/multi-parser` Exists

Some tools were built against older parser API versions:
- AsyncAPI Generator templates use a specific parser API version
- Older community tools may have been built against v0 or v1
- Some organizations have internal code that uses the old API

These tools cannot always be upgraded immediately. `@asyncapi/multi-parser` provides a single package that can instantiate any parser API version, allowing tools to specify exactly which version they need.

---

## `@asyncapi/multi-parser` — How It Works

**File:** `packages/multi-parser/src/parse.ts`

```typescript
import { Parser as ParserV1 } from 'parserapiv1';
import { Parser as ParserV2 } from 'parserapiv2';
import { Parser as ParserV3 } from '@asyncapi/parser';

export function NewParser(parserAPIMajorVersion?: number, options?: Options): Parser {
  switch (parserAPIMajorVersion) {
  case 1: return new ParserV1(parserOptions);
  case 2: return new ParserV2(parserOptions);
  default:
  case 3: return new ParserV3(parserOptions);
  }
}
```

The npm packages `parserapiv1` and `parserapiv2` are aliases:
- `parserapiv1` → `@asyncapi/parser@^2.1.0`
- `parserapiv2` → `@asyncapi/parser@3.0.0-next-major-spec.8`
- (default) `@asyncapi/parser` → current latest (v3 / Intent API)

---

## Using `NewParser()`

```js
const { NewParser } = require('@asyncapi/multi-parser');

// Get a Parser-API v3 parser (default)
const parserV3 = NewParser(3);
const { document: docV3 } = await parserV3.parse(asyncapiDoc);
// docV3 is AsyncAPIDocumentInterface (new API)

// Get a Parser-API v1 parser (for legacy tool compatibility)
const parserV1 = NewParser(1);
const { document: docV1 } = await parserV1.parse(asyncapiDoc);
// docV1 uses old API (OldAsyncAPIDocument equivalent)
```

### With all bundled schema parsers

```js
const parser = NewParser(3, {
  includeSchemaParsers: true,
  // ↑ Automatically includes: Avro, OpenAPI, RAML, Protobuf schema parsers
});
```

---

## `ConvertDocumentParserAPIVersion()`

**File:** `packages/multi-parser/src/convert.ts`

Converts a parsed document from one Parser-API version to another:

```js
const { NewParser, ConvertDocumentParserAPIVersion } = require('@asyncapi/multi-parser');

const parserV3 = NewParser(3);
const { document: docV3 } = await parserV3.parse(asyncapiDoc);

// Convert to Parser-API v1 (old API) format
const docV1 = ConvertDocumentParserAPIVersion(docV3, 1);
// docV1 uses old API accessors

// Convert back to v3
const docV3again = ConvertDocumentParserAPIVersion(docV1, 3);
```

---

## When You Need `@asyncapi/multi-parser`

Most developers should use `@asyncapi/parser` directly (always gets the latest API v3). Use `@asyncapi/multi-parser` only if:

1. You are building a tool that must support multiple AsyncAPI Generator template API versions
2. You are maintaining a library that other tools depend on, and those tools were built against older parser API versions
3. You are the AsyncAPI Generator team

For regular parser usage, issue debugging, and new feature development — use `@asyncapi/parser` directly.

---

## `x-parser-api-version` Quick Reference

After calling `parser.parse()`:

```js
const { document } = await parser.parse(asyncapiDoc);

// Check which Parser API version was used:
const apiVersion = document?.json()['x-parser-api-version'];
// → 3 (for new API)
// → 0 (for old API, after convertToOldAPI())
// → undefined (very old documents)
```

---

## Next Step

Move to [Chapter 6: Validation Ruleset](../06-validation-ruleset/01-spectral-integration.md) for a deep dive into how Spectral rules validate AsyncAPI documents.
