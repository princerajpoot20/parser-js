# Validation Layer

> **Goal:** Understand how Spectral validates AsyncAPI documents, how the JSON Schema validation via Ajv works, what severity levels mean, and how rules are organized.

---

## Overview

The validation layer has two tiers:

```
Tier 1: Spectral (rule engine)
├── Core rules (asyncapi-is-asyncapi, asyncapi-document-resolved, asyncapi-internal)
├── Recommended rules (info, servers, unused components)
├── Version-specific rules (v2: 25+ rules, v3: 5 rules)
└── Custom rules (user-configured)

Tier 2: Custom schema format validation (inside Tier 1)
└── Invoked by asyncapi2-schemas rule
    └── Uses registered SchemaParser.validate()
    └── Default: AsyncAPISchemaParser (Ajv)
    └── Optional: Avro, Protobuf, OpenAPI parsers
```

---

## Spectral: The Rule Engine

[Spectral](https://github.com/stoplightio/spectral) is a generic JSON/YAML linting framework from Stoplight. The AsyncAPI parser uses it as its validation backbone.

### Key Spectral Concepts

**Document**
A wrapper around the raw YAML/JSON string. Spectral parses it and tracks character positions for accurate diagnostic ranges.

**Rule**
A named validation check with:
- `given`: JSONPath expression selecting the nodes to check
- `then.function`: a function that receives each matched node and returns violations
- `severity`: `error`, `warn`, `info`, `hint`
- `recommended`: whether it's in the recommended ruleset (can be disabled)
- `resolved`: whether to run against the resolved or unresolved document

**Ruleset**
A collection of rules. The AsyncAPI parser builds a combined ruleset from core + recommended + version-specific + user-configured rulesets.

**Function**
A TypeScript function with signature `(input, options, context) => IFunctionResult[] | void`. Built-in functions from `@stoplight/spectral-functions` include `truthy`, `schema`, `pattern`, `length`, etc. Custom functions are in `src/ruleset/functions/`.

---

## How `createSpectral()` Works

**File:** `packages/parser/src/spectral.ts`

```typescript
export function createSpectral(parser: Parser, options: ParserOptions = {}): Spectral {
  const resolverOptions = options.__unstable?.resolver;
  // Create Spectral with $ref resolver
  const spectral = new Spectral({ resolver: createResolver(resolverOptions) });
  // Build and attach the combined ruleset
  const ruleset = createRuleset(parser, options.ruleset);
  spectral.setRuleset(ruleset);
  return spectral;
}
```

The `Spectral` instance is created once per `Parser` instance (at construction time) and reused for all `parse()` / `validate()` calls. The only exception is if `__unstable.resolver` is passed per-call — then a new Spectral instance is created just for that call.

---

## The Ruleset Stack

**File:** `packages/parser/src/ruleset/index.ts` and `ruleset/ruleset.ts`

The combined ruleset is built by `createRuleset()`:

```
createRuleset(parser, userOptions)
├── Core Ruleset (always active)
│   ├── asyncapi-is-asyncapi      [error]
│   ├── asyncapi-latest-version   [info]
│   ├── asyncapi-document-resolved    [error]
│   ├── asyncapi-document-unresolved  [error]
│   └── asyncapi-internal         [special — no severity]
│
├── Recommended Ruleset (active for v2; v3 WIP)
│   ├── asyncapi-id               [warning]
│   ├── asyncapi-defaultContentType [warning]
│   ├── asyncapi-info-description [warning]
│   ├── asyncapi-info-contact     [warning]
│   ├── asyncapi-info-contact-properties [warning]
│   ├── asyncapi-info-license     [warning]
│   ├── asyncapi-info-license-url [warning, not recommended by default]
│   ├── asyncapi-servers          [warning]
│   └── asyncapi-unused-component [info]
│
├── v2 Ruleset (active when document is asyncapi 2.x)
│   └── 25+ rules (see Chapter 6)
│
└── v3 Ruleset (active when document is asyncapi 3.x)
    └── 5 rules (see Chapter 6)
```

### Customizing the Ruleset

Users can pass `ruleset` option to the `Parser` constructor:

```typescript
const parser = new Parser({
  ruleset: {
    core: false,        // disable all core rules (rarely needed)
    recommended: false, // disable all recommended rules
    extends: [myCustomRuleset],  // add your own rules
  }
});
```

---

## The Most Important Rule: `documentStructure`

**File:** `packages/parser/src/ruleset/functions/documentStructure.ts`

This function is used by both `asyncapi-document-resolved` and `asyncapi-document-unresolved`. It validates the entire document against the official AsyncAPI JSON Schema using Ajv.

### How it works

1. Extract the AsyncAPI spec version from the document (`$.asyncapi` field)
2. Get the official JSON Schema for that version from `@asyncapi/specs`
3. Compile an Ajv validator for that schema
4. Validate the entire document (or resolved document) against it
5. Convert Ajv errors to Spectral diagnostics

```typescript
// documentStructure.ts (simplified)
function documentStructure(input, options, context) {
  const version = context.document.data?.asyncapi;
  const schema = specs.schemas[version];
  const validator = ajv.compile(schema);
  
  if (!validator(input)) {
    return validator.errors
      .filter(err => !shouldIgnoreError(err))
      .map(err => ({ message: err.message, path: err.instancePath }));
  }
}
```

### Why two rules (resolved vs unresolved)?

- `asyncapi-document-resolved`: validates the `$ref`-expanded document. Catches errors in the final structure after all refs are inlined.
- `asyncapi-document-unresolved` (`resolved: false`): validates the raw document with `$ref` strings intact. Catches errors in how refs are structured.

Running both catches different classes of errors.

### Error filtering

Some Ajv errors are deliberately suppressed:

```typescript
function shouldIgnoreError(error: ErrorObject): boolean {
  return (
    error.keyword === 'oneOf' ||   // oneOf errors are often redundant/confusing
    (error.keyword === 'required' && error.params.missingProperty === '$ref')
    // '$ref' is not a real required field, it's a JSON Reference indicator
  );
}
```

---

## The `asyncapi-internal` Rule

**File:** `packages/parser/src/ruleset/functions/internal.ts`

This rule does not produce diagnostics. It is an internal mechanism:

```typescript
function internal(input, options, context) {
  // Store Spectral's document inventory in the document object
  // so parse.ts can access it for circular ref resolution
  (context.document as any).__documentInventory = context.documentInventory;
}
```

The `documentInventory` is Spectral's internal tracker for all resolved `$ref` paths. The parser needs this to properly detect and handle circular references in the post-parse custom operations. Without this rule, the document inventory would not be accessible from `parse.ts`.

---

## Diagnostic Severity

Spectral uses the Stoplight `DiagnosticSeverity` enum:

| Numeric | Name | Spectral string | What it means |
|---------|------|----------------|---------------|
| 0 | `Error` | `'error'` | Invalid document; blocks model creation by default |
| 1 | `Warning` | `'warn'` | Document is valid but has issues |
| 2 | `Information` | `'info'` | Informational, e.g., "newer version available" |
| 3 | `Hint` | `'hint'` | Minor suggestions |

### Severity Gating

By default, only `Error`-severity diagnostics prevent `document` from being returned:

```typescript
// validate.ts — default allowedSeverity
const defaultOptions: ValidateOptions = {
  allowedSeverity: {
    error: false,    // errors are NOT allowed → block parsing
    warning: true,   // warnings ARE allowed → don't block
    info: true,
    hint: true,
  },
};
```

You can make warnings also block by setting `allowedSeverity.warning: false` in `ValidateOptions`.

---

## `$ref` Resolver

**File:** `packages/parser/src/resolver.ts`

The Spectral instance is created with a custom resolver that handles the protocols used in AsyncAPI documents:

```typescript
// resolver.ts (simplified)
export function createResolver(options?: ResolverOptions) {
  return new Resolver({
    resolvers: {
      file: createFileResolver(),    // handles file:// and relative paths
      http: createHttpResolver(),    // handles http://
      https: createHttpResolver(),   // handles https://
      ...options?.resolvers          // user-provided custom resolvers
    }
  });
}
```

The `source` option in `ParseOptions` is critical for file resolvers — without knowing the document's location, relative references like `./other.yaml` cannot be resolved.

### Custom Protocol Resolvers

Users can add resolvers for custom protocols:

```typescript
const parser = new Parser({
  __unstable: {
    resolver: {
      resolvers: {
        'myschema': {
          resolve(uri) {
            return fetch(`https://my-registry.example.com/${uri.path()}`).then(r => r.text());
          }
        }
      }
    }
  }
});
```

---

## Format Detection

**File:** `packages/parser/src/ruleset/formats.ts`

Spectral rules are scoped to specific document formats. AsyncAPI format detectors identify which version of AsyncAPI a document is:

```typescript
// formats.ts (simplified)
export const AsyncAPIFormats = {
  formats: () => [asyncapi2, asyncapi3],
  filterByMajorVersions: (majors) => {
    return {
      formats: () => majors.map(major => major === '2' ? asyncapi2 : asyncapi3)
    };
  }
};

const asyncapi2 = createFormat('asyncapi2', /^2\./);
const asyncapi3 = createFormat('asyncapi3', /^3\./);
```

v2-specific rules use `formats: [asyncapi2]` and v3-specific rules use `formats: [asyncapi3]`. Core rules use all formats.

---

## `extras.document` — Accessing Spectral Internals

`parse()` and `validate()` return `extras.document`, the raw Spectral `Document` object:

```typescript
const { document, diagnostics, extras } = await parser.parse(asyncapi);
// extras.document is the Spectral Document before model wrapping
```

This is useful for:
- Accessing the resolved JSON directly: `extras.document.data`
- Inspecting JSON paths from the document: `extras.document.getRangeForJsonPath()`
- Getting the full `documentInventory` for advanced circular ref analysis

---

## Next Step

Read [04-model-layer.md](./04-model-layer.md) to understand how the parsed JSON is wrapped in typed model classes.
