# High-Level Architecture

> **Goal:** Understand the complete system in one diagram, then know what each major component does and how they relate.

---

## Complete System Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                           Public API                                 │
│                                                                      │
│   new Parser(options?)                                               │
│     .parse(input, options?)  ──────────────────────────────┐        │
│     .validate(input, options?) ────────────────────┐       │        │
│     .registerSchemaParser(parser)                  │       │        │
└────────────────────────────────────────────────────│───────│────────┘
                                                     │       │
                 ┌───────────────────────────────────┘       │
                 │                                           │
                 ▼                                           ▼
    ┌────────────────────────┐              ┌────────────────────────┐
    │     validate.ts        │              │       parse.ts         │
    │                        │              │                        │
    │  1. normalizeInput()   │◄─────────────│  Calls validate() then │
    │  2. new Document()     │              │  continues if valid    │
    │  3. spectral.run()     │              │                        │
    │  4. severity gating    │              └──────────┬─────────────┘
    └──────────┬─────────────┘                         │
               │                                       │
               │ validated JSON (resolved)             │
               │                                       │
               ▼                                       ▼
    ┌────────────────────────┐   ┌────────────────────────────────────┐
    │   Spectral Engine      │   │  Parse Pipeline (parse.ts cont.)   │
    │   (spectral.ts)        │   │                                    │
    │                        │   │  1. copy(validated)                │
    │  Rules from:           │   │     (unfreeze Spectral's result)   │
    │  ├─ ruleset.ts         │   │  2. applyUniqueIds()               │
    │  ├─ v2/ruleset.ts      │   │  3. createDetailedAsyncAPI()       │
    │  ├─ v3/ruleset.ts      │   │  4. createAsyncAPIDocument()       │
    │  └─ custom ruleset     │   │     (v2 or v3 model)               │
    │                        │   │  5. setExtension(x-parser-*)       │
    │  $ref Resolver:        │   │  6. customOperations()             │
    │  ├─ file://            │   │                                    │
    │  ├─ http://            │   └──────────────┬─────────────────────┘
    │  └─ custom             │                  │
    └────────────────────────┘                  │
                                                ▼
                              ┌────────────────────────────────────────┐
                              │      Custom Operations (post-parse)    │
                              │                                        │
                              │  checkCircularRefs()                   │
                              │  applyTraits() [if applyTraits:true]   │
                              │  parseSchemas() [if parseSchemas:true] │
                              │  resolveCircularRefs()                 │
                              │  anonymousNaming()                     │
                              └──────────────┬─────────────────────────┘
                                             │
                                             ▼
                              ┌────────────────────────────────────────┐
                              │         Output                         │
                              │                                        │
                              │  {                                     │
                              │    document: AsyncAPIDocumentV2|V3     │
                              │             | undefined,               │
                              │    diagnostics: Diagnostic[],          │
                              │    extras: { document: SpectralDoc }   │
                              │  }                                     │
                              └────────────────────────────────────────┘
```

---

## The `Parser` Class

**File:** `packages/parser/src/parser.ts`

The `Parser` class is the only object users instantiate directly. It is a thin facade over the internal pipeline.

```typescript
export class Parser {
  public readonly parserRegistry = new Map<string, SchemaParser>();
  protected readonly spectral: Spectral;

  constructor(private readonly options: ParserOptions = {}) {
    // 1. Create a Spectral instance with configured rules + resolver
    this.spectral = createSpectral(this, options);
    // 2. Register the default JSON Schema parser
    this.registerSchemaParser(AsyncAPISchemaParser());
    // 3. Register any additional schema parsers from options
    this.options.schemaParsers?.forEach(parser => this.registerSchemaParser(parser));
  }
}
```

### Constructor Options (`ParserOptions`)

| Option | Type | Purpose |
|--------|------|---------|
| `ruleset` | `RulesetOptions` | Customize which Spectral rules run (disable core/recommended, add custom) |
| `schemaParsers` | `SchemaParser[]` | Additional schema parsers to register (Avro, Protobuf, etc.) |
| `__unstable.resolver` | `ResolverOptions` | Override the `$ref` resolver (custom protocols, caching) |

### `parse(input, options?)` vs `validate(input, options?)`

| Method | Returns | Creates model? | Use case |
|--------|---------|---------------|---------|
| `parse()` | `{ document, diagnostics }` | Yes (if no errors) | Normal usage — need to traverse the document |
| `validate()` | `Diagnostic[]` | No | CI linting, pre-commit hooks |

Both methods have a short-circuit: if `input` is already a parsed `AsyncAPIDocumentInterface`, they return immediately without re-processing.

---

## Input Types

The `Input` type accepts multiple forms:

```typescript
type Input = string | Record<string, unknown> | AsyncAPIDocumentInterface;
```

| Input type | Example | What happens |
|------------|---------|-------------|
| YAML string | `"asyncapi: '2.6.0'\n..."` | Parsed by `@stoplight/yaml` |
| JSON string | `'{"asyncapi":"2.6.0",...}'` | Parsed by `@stoplight/yaml` (handles both) |
| Plain object | `{ asyncapi: '2.6.0', ... }` | Serialized to string first via `JSON.stringify` |
| `AsyncAPIDocumentInterface` | A previously parsed doc | Short-circuit: returned as-is |

---

## Parsing vs Validation Difference

This is a common point of confusion:

```
validate():
  Input → Spectral → Diagnostics
  
parse():
  Input → Spectral → (if no blocking errors) → Model Pipeline → Document + Diagnostics
```

`parse()` calls `validate()` internally. If validation produces **error-severity** diagnostics (severity=0), the model pipeline is skipped and `document` is `undefined`.

By default, warnings/info/hints do NOT block parsing. This is controlled by `allowedSeverity`:

```typescript
const diagnostics = await parser.validate(doc, {
  allowedSeverity: {
    error: false,    // errors block (default)
    warning: false,  // warnings also block (non-default)
    info: true,
    hint: true,
  }
});
```

---

## Data Flow Summary

```
1. Input (string/object/doc)
        │
        ▼
2. normalizeInput()
   └─ string → stays string
   └─ object → JSON.stringify
   └─ AsyncAPIDoc → short-circuit return
        │
        ▼
3. Spectral Document created
   └─ Parsed as YAML (handles both YAML and JSON)
   └─ $refs resolved (file, http, custom)
        │
        ▼
4. Spectral rules run
   ├─ asyncapi-is-asyncapi (version check)
   ├─ asyncapi-document-resolved/unresolved (JSON Schema via Ajv)
   ├─ recommended rules (info, servers, unused components)
   ├─ v2-specific rules OR v3-specific rules
   └─ Custom schema format validation (if schemaFormat found)
        │
        ├─── [errors found] → validated=undefined → return { document: undefined, diagnostics }
        │
        ▼ [no blocking errors]
5. copy(validated)
   └─ Unfreeze Spectral's resolved JSON object
        │
        ▼
6. applyUniqueIds(doc)
   └─ Assigns x-parser-unique-object-id to channels, operations, messages
        │
        ▼
7. createDetailedAsyncAPI(doc, input, source)
   └─ Wraps parsed JSON with semver info, input reference, source path
        │
        ▼
8. createAsyncAPIDocument(detailed)
   └─ AsyncAPIDocumentV2 if semver.major === 2
   └─ AsyncAPIDocumentV3 if semver.major === 3
        │
        ▼
9. setExtension(x-parser-spec-parsed, true)
   setExtension(x-parser-api-version, 3)
        │
        ▼
10. customOperations():
    ├─ checkCircularRefs
    ├─ applyTraits (if options.applyTraits)
    ├─ parseSchemas (if options.parseSchemas)
    ├─ resolveCircularRefs
    └─ anonymousNaming
        │
        ▼
11. Return { document, diagnostics, extras }
```

---

## Key Internal Types

### `DetailedAsyncAPI`

The internal representation that carries everything needed for model construction:

```typescript
interface DetailedAsyncAPI {
  source: string | undefined;   // file path or URL of the source document
  input: Input;                 // original input as provided by the user
  parsed: v2.AsyncAPIObject | v3.AsyncAPIObject;  // fully resolved JSON
  semver: AsyncAPISemver;      // { version: '2.6.0', major: 2, minor: 6, patch: 0 }
}
```

### `AsyncAPISemver`

```typescript
interface AsyncAPISemver {
  version: string;
  major: number;
  minor: number;
  patch: number;
}
```

The `major` field is the key branching point used throughout the code to dispatch between v2 and v3 behavior.

### `Diagnostic`

Every validation finding is a `Diagnostic`:

```typescript
interface Diagnostic {
  code: string;              // rule name (e.g., 'asyncapi2-channel-servers')
  message: string;           // human-readable description
  path: Array<string>;       // JSON pointer path (e.g., ['channels', 'user/registered'])
  severity: DiagnosticSeverity; // 0=Error, 1=Warning, 2=Info, 3=Hint
  range: {
    start: { line: number; character: number };
    end:   { line: number; character: number };
  };
  source?: string;           // file path of the document containing this issue
}
```

---

## `x-parser-*` Extension Keys

The parser adds non-spec fields (prefixed `x-`) to track internal state. These are defined in `constants.ts`:

| Extension key | Where added | What it marks |
|---------------|-------------|---------------|
| `x-parser-spec-parsed` | Root document | Document was successfully parsed |
| `x-parser-api-version` | Root document | Parser API version (currently `3`) |
| `x-parser-unique-object-id` | Every channel/operation/message | Deterministic unique ID for cross-referencing |
| `x-parser-message-name` | Anonymous messages | Auto-generated name |
| `x-parser-schema-id` | Schemas | Schema identifier |
| `x-parser-original-schema-format` | Messages with custom format | Original `schemaFormat` value before conversion |
| `x-parser-original-payload` | Messages with custom format | Original payload before schema parser conversion |
| `x-parser-original-traits` | Operations/messages | Original traits before merging |
| `x-parser-circular` | Root document | Document contains circular refs |
| `x-parser-circular-props` | Schemas | Schema properties that are circular |

---

## Next Step

Read [02-parsing-pipeline-deep-dive.md](./02-parsing-pipeline-deep-dive.md) for a step-by-step walkthrough of the code inside each stage.
