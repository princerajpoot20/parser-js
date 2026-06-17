# Multi-Parser Guide

> **Goal:** Understand `@asyncapi/multi-parser`, when to use it, how `NewParser()` works, and how `ConvertDocumentParserAPIVersion()` converts documents between parser API versions.

---

## Package Overview

**Package:** `@asyncapi/multi-parser` (v2.3.0)  
**Location:** `packages/multi-parser/` in this monorepo  
**NPM:** `npm install @asyncapi/multi-parser`

`@asyncapi/multi-parser` is a thin wrapper around multiple versions of `@asyncapi/parser`. It lets a single codebase work with Parser-API v1, v2, and v3 simultaneously, and convert documents between those API versions.

---

## When to Use It

| Scenario | Use `@asyncapi/parser` | Use `@asyncapi/multi-parser` |
|----------|----------------------|---------------------------|
| New tool, new code | ✓ | — |
| Need to navigate a parsed AsyncAPI doc | ✓ | — |
| Building a tool that supports multiple generator template API versions | — | ✓ |
| Need Parser-API v1 or v2 specifically | — | ✓ |
| Convert a doc from Parser-API v3 to v1 for a legacy system | — | ✓ |

Most maintainers will only ever use `@asyncapi/parser` directly. `@asyncapi/multi-parser` is primarily for the AsyncAPI Generator and tools that must support multiple template/tooling API versions.

---

## `NewParser(version?, options?)`

**File:** `packages/multi-parser/src/parse.ts`

```typescript
export function NewParser(parserAPIMajorVersion?: number, options?: Options): Parser
```

Returns a parser configured for the specified Parser-API major version.

| `parserAPIMajorVersion` | Returns | npm package alias |
|------------------------|---------|-------------------|
| `1` | `Parser` from `parserapiv1` | `@asyncapi/parser@^2.1.0` |
| `2` | `Parser` from `parserapiv2` | `@asyncapi/parser@3.0.0-next-major-spec.8` |
| `3` or `undefined` (default) | `Parser` from `@asyncapi/parser` | Latest |

### Basic Usage

```js
const { NewParser } = require('@asyncapi/multi-parser');

// Get Parser-API v3 (same as new Parser() from @asyncapi/parser)
const parser = NewParser(3);
const { document } = await parser.parse(asyncapiDoc);

// Check API version
console.log(document?.json()['x-parser-api-version']); // 3
```

### With All Schema Parsers

```js
const parser = NewParser(3, {
  includeSchemaParsers: true,
  // Automatically includes: Avro, OpenAPI, RAML DT, Protobuf parsers
});
```

### With Custom Parser Options

```js
const parser = NewParser(3, {
  parserOptions: {
    ruleset: {
      recommended: false, // disable recommended rules
    }
  }
});
```

The `parserOptions` type is a union of the options types for all parser versions. Pass options compatible with your chosen version.

---

## Bundled Schema Parsers

When `includeSchemaParsers: true`, multi-parser bundles these schema parsers:

| Package | Schema format | MIME type prefix |
|---------|--------------|------------------|
| `@asyncapi/avro-schema-parser` | Apache Avro | `application/vnd.apache.avro` |
| `@asyncapi/openapi-schema-parser` | OpenAPI Schema Object | `application/vnd.oai.openapi` |
| `@asyncapi/raml-dt-schema-parser` | RAML Data Types | `application/raml+yaml` |
| `@asyncapi/protobuf-schema-parser` | Protocol Buffers | `application/vnd.google.protobuf` |

When a user-provided parser in `parserOptions.schemaParsers` handles the same MIME types as a bundled parser, the user's parser takes precedence.

---

## `ConvertDocumentParserAPIVersion(doc, toVersion)`

**File:** `packages/multi-parser/src/convert.ts`

Converts a parsed document from one Parser-API version to another:

```typescript
export function ConvertDocumentParserAPIVersion(
  doc: AsyncAPIDocument,
  toParserAPIMajorVersion: number
): AsyncAPIDocument
```

### How It Works

1. Reads `x-parser-api-version` from the document's extensions to determine current version
2. If current version === target version, returns doc unchanged
3. Otherwise, calls `createAsyncAPIDocument()` from the target version's parser package, passing the shared `DetailedAsyncAPI` (raw parsed JSON + metadata)

The key insight: all parser API versions share the same underlying `DetailedAsyncAPI` (the raw resolved JSON). Only the model wrapper classes differ. Conversion is essentially "unwrap the model, wrap with a different model class".

### Example

```js
const { NewParser, ConvertDocumentParserAPIVersion } = require('@asyncapi/multi-parser');

async function main() {
  // Parse with v3 API
  const parserV3 = NewParser(3);
  const { document: docV3 } = await parserV3.parse(asyncapiDoc);

  // Convert to v1 API (for legacy consumer)
  const docV1 = ConvertDocumentParserAPIVersion(docV3, 1);
  console.log(docV1.json()['x-parser-api-version']); // may be 1 or undefined

  // Convert to v3 API (from v1)
  const backToV3 = ConvertDocumentParserAPIVersion(docV1, 3);
  console.log(backToV3.json()['x-parser-api-version']); // 3

  // Use v3 API on the converted document
  const channels = backToV3.channels().all();
  console.log(channels.map(c => c.id()));
}
```

### No-Op Case

```js
const docV3 = await NewParser(3).parse(asyncapiDoc).then(r => r.document);

// Same version — returns doc unchanged
const sameDoc = ConvertDocumentParserAPIVersion(docV3, 3);
console.log(docV3 === sameDoc); // true
```

---

## Internal Package Structure

```
packages/multi-parser/
├── src/
│   ├── index.ts      ← exports NewParser, ConvertDocumentParserAPIVersion
│   ├── parse.ts      ← NewParser() implementation
│   └── convert.ts    ← ConvertDocumentParserAPIVersion() implementation
├── test/
│   ├── parse.spec.ts   ← Tests for NewParser()
│   └── convert.spec.ts ← Tests for ConvertDocumentParserAPIVersion()
└── package.json
```

The `package.json` `dependencies` aliases:

```json
{
  "parserapiv1": "npm:@asyncapi/parser@^2.1.0",
  "parserapiv2": "npm:@asyncapi/parser@3.0.0-next-major-spec.8",
  "@asyncapi/parser": "*"
}
```

This is how npm workspace aliasing works — `parserapiv1` is just `@asyncapi/parser@^2.1.0` installed under a different name.

---

## Running Multi-Parser Tests

```bash
# From repo root
npm run multi-parser:test

# From packages/multi-parser
cd packages/multi-parser
npm test
```

The tests verify:
- `NewParser(1)` returns a parser that produces v1-API documents
- `NewParser(3)` returns a parser that produces v3-API documents
- `ConvertDocumentParserAPIVersion(doc, 3)` upgrades old docs
- Schema parsers work across all versions when `includeSchemaParsers: true`

---

## Next Step

Move to [Chapter 8: Contributor Guide](../08-contributor-guide/01-understanding-github-issues.md) — everything you need to start triaging and resolving GitHub issues.
