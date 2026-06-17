# Parsing Pipeline Deep Dive

> **Goal:** Walk through every step of `parse.ts` and `validate.ts` with actual code references, explaining what happens at each stage and why.

---

## Step 0: Entry — `Parser.parse()`

**File:** `packages/parser/src/parser.ts`

```typescript
async parse(asyncapi: Input, options?: ParseOptions): Promise<ParseOutput> {
  // Short-circuit: if input is already a parsed document, return it
  const maybeDocument = toAsyncAPIDocument(asyncapi);
  if (maybeDocument) {
    return { document: maybeDocument, diagnostics: [] };
  }
  // Otherwise, run the full pipeline
  return parse(this, this.spectral, asyncapi, options);
}
```

The short-circuit check calls `toAsyncAPIDocument()` which looks for `x-parser-spec-parsed` extension:

```typescript
// document.ts
export function toAsyncAPIDocument(asyncapi: Input): AsyncAPIDocumentInterface | undefined {
  if (isAsyncAPIDocument(asyncapi)) return asyncapi;
  if (typeof asyncapi === 'object' && asyncapi !== null) {
    // Check if it's a plain object that was previously parsed
    if ((asyncapi as any)[xParserSpecParsed]) {
      // Reconstruct from raw JSON
      ...
    }
  }
  return undefined;
}
```

---

## Step 1: Option Merging

**File:** `packages/parser/src/parse.ts`

```typescript
options = mergePatch<ParseOptions>(defaultOptions, options);
```

The `mergePatch` function does a deep merge of user options over defaults:

```typescript
const defaultOptions: ParseOptions = {
  applyTraits: true,
  parseSchemas: true,
  validateOptions: {},
  __unstable: {},
};
```

If a user passes `{ applyTraits: false }`, the result is:

```typescript
{
  applyTraits: false,    // overridden
  parseSchemas: true,    // kept from default
  validateOptions: {},
  __unstable: {}
}
```

---

## Step 2: Validate

**File:** `packages/parser/src/validate.ts`

`parse()` immediately calls `validate()`:

```typescript
const { validated, diagnostics, extras } = await validate(
  parser, spectral, asyncapi, { ...options.validateOptions, source: options.source }
);
```

Inside `validate()`:

### 2a. Normalize Input

```typescript
const stringifiedDocument = normalizeInput(asyncapi);
```

`normalizeInput()` in `utils.ts` converts the input to a string:
- If `string` → returns as-is
- If `object` → `JSON.stringify()` (Spectral's Document requires a string)

### 2b. Create a Spectral Document

```typescript
document = new Document(stringifiedDocument, Yaml, options.source);
```

`@stoplight/spectral-core`'s `Document` wraps the string with a parser (`Yaml` handles both YAML and JSON). The `source` parameter provides the file path or URL for:
- Range-accurate error locations
- Relative `$ref` resolution base path

A private field is added for later use by rules:
```typescript
(document as any).__parserInput = asyncapi;
```

### 2c. Run Spectral

```typescript
let { resolved: validated, results } = await spectral.runWithResolved(document, {});
```

`runWithResolved()` is Spectral's method that:
1. **Resolves** all `$ref` entries using the configured resolver
2. **Runs** all rules against both the resolved and unresolved document
3. Returns `resolved` (the fully dereferenced document object) and `results` (array of diagnostics)

The resolved document is a plain JavaScript object — all `$ref` strings have been replaced with the actual referenced content.

### 2d. Severity Gating

```typescript
if (
  (!allowedSeverity?.error   && hasErrorDiagnostic(results))   ||
  (!allowedSeverity?.warning && hasWarningDiagnostic(results)) ||
  (!allowedSeverity?.info    && hasInfoDiagnostic(results))    ||
  (!allowedSeverity?.hint    && hasHintDiagnostic(results))
) {
  validated = undefined;  // Block model construction
}
```

Default `allowedSeverity`:
- `error: false` → errors block (validated = undefined)
- `warning: true` → warnings allowed (don't block)
- `info: true`
- `hint: true`

If `validated` is `undefined`, `parse()` returns immediately:
```typescript
if (validated === undefined) {
  return { document: undefined, diagnostics, extras };
}
```

---

## Step 3: Unfreeze the Resolved Document

**File:** `packages/parser/src/parse.ts`

```typescript
const validatedDoc = copy(validated as Record<string, any>);
```

Spectral's resolved object is **frozen** (via `Object.freeze()`) to prevent accidental mutation during rule execution. Since our custom operations need to add `x-parser-*` extensions to the document, we use `copy()` from `stringify.ts` to create an unfrozen deep copy.

`copy()` is not a naive `JSON.parse(JSON.stringify(...))` — it handles circular references by tracking them and replacing with `$ref` string references.

---

## Step 4: Apply Unique IDs

**File:** `packages/parser/src/custom-operations/apply-unique-ids.ts`

```typescript
applyUniqueIds(validatedDoc);
```

This walks the document and assigns `x-parser-unique-object-id` to channels, operations, and messages. These IDs are deterministic — based on the position in the document — and are used to cross-reference objects across the model tree.

For example, a channel at path `channels.user/registered` gets an ID like `user/registered`. An operation's message references back to the same message object in components via these IDs.

---

## Step 5: Create `DetailedAsyncAPI`

**File:** `packages/parser/src/utils.ts`

```typescript
const detailed = createDetailedAsyncAPI(validatedDoc, asyncapi as DetailedAsyncAPI['input'], options.source);
```

`DetailedAsyncAPI` is the internal context object passed everywhere in the model layer:

```typescript
interface DetailedAsyncAPI {
  source: string | undefined;   // 'file:///path/to/doc.yaml' or 'https://...'
  input: Input;                 // original input (before normalization)
  parsed: v2.AsyncAPIObject | v3.AsyncAPIObject;  // resolved JSON
  semver: AsyncAPISemver;       // { version: '2.6.0', major: 2, minor: 6, patch: 0 }
}
```

The `semver.major` field is used throughout to branch between v2 and v3 logic.

---

## Step 6: Create the Document Model

**File:** `packages/parser/src/document.ts`

```typescript
const document = createAsyncAPIDocument(detailed);
```

```typescript
export function createAsyncAPIDocument(asyncapi: DetailedAsyncAPI): AsyncAPIDocumentInterface {
  switch (asyncapi.semver.major) {
  case 2:
    return new AsyncAPIDocumentV2(asyncapi.parsed as v2.AsyncAPIObject, { asyncapi, pointer: '/' });
  case 3:
    return new AsyncAPIDocumentV3(asyncapi.parsed as v3.AsyncAPIObject, { asyncapi, pointer: '/' });
  default:
    throw new Error(`Unsupported AsyncAPI version: ${asyncapi.semver.version}`);
  }
}
```

`AsyncAPIDocumentV2` and `AsyncAPIDocumentV3` extend `BaseModel`. They are **lazy** — they do not pre-compute the entire model tree on construction. Each accessor (`.channels()`, `.servers()`, etc.) creates child model instances on demand.

The second argument `{ asyncapi, pointer: '/' }` is the `ModelMetadata`:
- `asyncapi`: reference to the `DetailedAsyncAPI` context (passed down to all child models)
- `pointer`: JSON Pointer string (`/channels/user~1registered` for `channels["user/registered"]`)

---

## Step 7: Set Parser Extension Markers

```typescript
setExtension(xParserSpecParsed, true, document);   // x-parser-spec-parsed: true
setExtension(xParserApiVersion, ParserAPIVersion, document);  // x-parser-api-version: 3
```

These are how tools (and the parser itself) detect that a document has been parsed and which API version it uses:
- `isAsyncAPIDocument()` checks for `x-parser-api-version === 3`
- `isOldAsyncAPIDocument()` checks for `x-parser-api-version === 0`

---

## Step 8: Custom Operations

**File:** `packages/parser/src/custom-operations/index.ts`

```typescript
await customOperations(parser, document, detailed, inventory, options);
```

This dispatches to either `operationsV2()` or `operationsV3()` based on `detailed.semver.major`. Both run the same set of operations in the same order:

```typescript
async function operationsV2(parser, document, detailed, inventory, options) {
  checkCircularRefs(document);

  if (options.applyTraits) {
    applyTraitsV2(detailed.parsed as v2.AsyncAPIObject);
  }
  if (options.parseSchemas) {
    await parseSchemasV2(parser, detailed);
  }

  if (inventory) {
    resolveCircularRefs(document, inventory);
  }
  anonymousNaming(document);
}
```

### `checkCircularRefs`

Walks the document JSON to detect any circular references (where a `$ref` eventually points back to itself). Sets `x-parser-circular: true` on the root document if found.

### `applyTraits`

Merges trait objects into operations and messages. After this step, an operation or message has all trait fields merged directly into its JSON — the original `traits` array is preserved in `x-parser-original-traits`.

### `parseSchemas`

Walks the document looking for message payloads and invokes the appropriate registered `SchemaParser` based on the `schemaFormat` MIME type. For JSON Schema (the default), the payload is returned as-is. For Avro, the Avro-specific parser converts the Avro schema to an AsyncAPI Schema Object.

### `resolveCircularRefs`

Uses the Spectral `documentInventory` (passed through from the validate step) to resolve circular references in the model. The inventory tracks which `$ref` paths have been visited. Circular schemas get `x-parser-circular-props` markers.

### `anonymousNaming`

Assigns auto-generated names to anonymous messages (those without a `name` field). Names are based on the channel address or operation ID. This enables tools to reference messages consistently even without explicit names.

---

## Step 9: Return Output

```typescript
return { 
  document,        // AsyncAPIDocumentV2 | AsyncAPIDocumentV3 | undefined
  diagnostics,     // Diagnostic[]
  extras,          // { document: SpectralDocument } — raw Spectral internals
};
```

The `extras.document` is the raw Spectral `Document` object. This is useful for advanced debugging (inspecting the JSON path resolution, examining the pre-model-wrap state). Most users ignore `extras`.

---

## Error Handling

The entire pipeline is wrapped in a `try/catch`:

```typescript
} catch (err: unknown) {
  return {
    document: undefined,
    diagnostics: createUncaghtDiagnostic(err, 'Error thrown during AsyncAPI document parsing', spectralDocument),
    extras: undefined
  };
}
```

This means unexpected exceptions (bugs in the parser, memory issues, etc.) are returned as `diagnostics` rather than thrown. The diagnostic has code `'uncaught-error'` and includes the error stack trace.

---

## `ParseOptions` Reference

| Option | Type | Default | Effect |
|--------|------|---------|--------|
| `source` | `string` | `undefined` | File path / URL for `$ref` base resolution and diagnostic source labels |
| `applyTraits` | `boolean` | `true` | Run `applyTraits` custom operation |
| `parseSchemas` | `boolean` | `true` | Run `parseSchemas` custom operation |
| `validateOptions.allowedSeverity` | object | `{ error: false, ... }` | Control which severity levels block model creation |
| `__unstable.resolver` | `ResolverOptions` | — | Override the `$ref` resolver for this parse call |

---

## Next Step

Read [03-validation-layer.md](./03-validation-layer.md) to understand how Spectral runs validation rules and how Ajv validates the document structure.
