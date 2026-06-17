# Where parser-js Fits in the AsyncAPI Ecosystem

> **Goal:** Understand the AsyncAPI tooling ecosystem and the specific role `@asyncapi/parser` plays. Understand why the parser exists and what tools depend on it.

---

## The AsyncAPI Tooling Ecosystem

AsyncAPI is a specification, but the real value comes from the ecosystem of tools built around it. Here is how the major tools relate to each other:

```
AsyncAPI Document (.yaml / .json)
          │
          ▼
┌─────────────────────────────────────┐
│          @asyncapi/parser           │  ← YOU ARE HERE
│  • Validates the document           │
│  • Resolves $refs                   │
│  • Creates typed model tree         │
└─────────────────────────────────────┘
          │
          │ AsyncAPIDocumentInterface
          │ (typed, navigable model)
          │
    ┌─────┴──────┬──────────────────┬──────────────────┐
    ▼            ▼                  ▼                  ▼
@asyncapi/   AsyncAPI           @asyncapi/         Community
generator    Studio             modelina            tools
(code gen)  (visual editor)    (data models)      (docs, linters,
                                                   CI validators)
```

### `@asyncapi/parser` (this repo)

The **foundation library**. Everything else in the ecosystem depends on it. It:
- Takes a raw YAML/JSON AsyncAPI document
- Validates it against the official JSON Schemas
- Resolves all `$ref` references
- Runs domain-specific Spectral rules
- Produces a fully typed, navigable `AsyncAPIDocumentInterface` object

Almost every AsyncAPI tool that works with document content uses this library.

### `@asyncapi/generator`

A code generation tool. Given a parsed `AsyncAPIDocumentInterface`, it fills in Handlebars templates to generate code in any language or framework (TypeScript clients, Python producers, Go consumers, Java Spring Boot apps, etc.).

Without the parser, the generator would need to re-implement reference resolution, validation, and all the model traversal logic itself.

### AsyncAPI Studio

A browser-based editor at [studio.asyncapi.com](https://studio.asyncapi.com). It uses the parser (compiled to a browser bundle) to provide real-time validation and preview as you type your AsyncAPI document.

### `@asyncapi/modelina`

A data model generator that takes message schemas from an AsyncAPI document and generates language-specific data classes (TypeScript interfaces, Java classes, Python dataclasses, etc.).

### Community tools

Many community-built tools (documentation generators, CI validators, linters, gateway configs) are built on top of `@asyncapi/parser`.

---

## Why the Parser is Necessary

You might wonder: why not just use a YAML parser and read the fields directly? Here is why a dedicated parser is needed:

### 1. Reference Resolution

A real-world AsyncAPI document has dozens of `$ref` entries that may point to external files or URLs. Before you can traverse the document, you need to fetch and inline those references. This requires:
- File system access (for local refs)
- HTTP/HTTPS client (for remote refs)
- Circular reference detection (a ref can form a cycle)
- Caching (same ref fetched multiple times)

### 2. Spec Validation

The AsyncAPI specification is defined as a JSON Schema. Validating a document against it requires Ajv plus additional domain-specific rules (e.g., "every channel parameter must have a corresponding `{parameter}` in the channel address"). The parser bundles all of this via Spectral.

### 3. Typed Navigation

A raw JavaScript object after YAML parsing is untyped. A generator or tool that traverses the document would need to write `doc.channels?.['user/registered']?.publish?.message?.payload` with no type safety and no knowledge of what fields are valid.

The parser produces strongly-typed model objects like `AsyncAPIDocumentV2` where you call `doc.channels().get('user/registered').operations().all()[0].messages().all()[0].payload()` with full TypeScript type safety.

### 4. Trait Merging

AsyncAPI supports "traits" — reusable fragments that are merged into operations or messages. Without a parser, every tool would need to re-implement this merge logic.

### 5. Spec Version Handling

AsyncAPI 2.x and 3.x have significantly different structures. The parser handles version detection and creates the right model class (`AsyncAPIDocumentV2` vs `AsyncAPIDocumentV3`) so consumers always get a consistent API.

---

## The Two Packages in this Repo

### `@asyncapi/parser`

The main library. It:
- Parses a single AsyncAPI document
- Supports AsyncAPI 2.0.0–2.6.0 and 3.0.0
- Exposes the Parser-API v3 (Intent API) model
- Has a compatibility shim (`convertToOldAPI`) for tools that used older versions

This is what you will spend 95% of your time on as a maintainer.

### `@asyncapi/multi-parser`

A thin wrapper that allows tools to work with **multiple versions of the Parser API** simultaneously.

Background: The parser itself has had multiple major API versions (v0 = old API, v3 = Intent API). Some tools were built against older parser API versions and can't upgrade easily. `@asyncapi/multi-parser` lets a tool use Parser-API v1, v2, or v3 interchangeably, and can convert documents between parser API versions.

This is useful for the AsyncAPI generator which supports multiple template API versions.

---

## Parser-API vs AsyncAPI Spec Version

This distinction is important and often confuses new contributors:

| Concept | What it is | Examples |
|---------|------------|---------|
| **AsyncAPI spec version** | The version of the AsyncAPI *specification* — what the YAML document describes | 2.0.0, 2.6.0, 3.0.0 |
| **Parser-API version** | The version of the *TypeScript API* the parser exposes for navigating parsed documents | v0 (old-api), v3 (Intent API) |

`@asyncapi/parser` v3.x (npm package version) implements Parser-API v3 (Intent API) and supports AsyncAPI spec versions 2.x and 3.x.

The [parser-api repo](https://github.com/asyncapi/parser-api) defines the Parser-API specification — the interface contracts that model classes must implement.

---

## What the Parser Does NOT Do

Understanding the scope helps with issue triage:

| Task | Not this parser's job | Who does it |
|------|----------------------|-------------|
| Generate code from a parsed doc | `@asyncapi/generator` | |
| Convert AsyncAPI 1.x to 2.x | `@asyncapi/converter-js` | |
| Generate data model classes | `@asyncapi/modelina` | |
| Render documentation HTML | `@asyncapi/generator` with templates | |
| Diff two AsyncAPI documents | Community tools | |
| Upgrade 2.x to 3.x | `@asyncapi/converter-js` | |
| Run at the broker level | Not a runtime tool — parse-time only | |

---

## How Issues in This Repo Typically Fall

When you see a GitHub issue, it usually falls into one of these categories:

1. **Validation bug** — the parser accepts an invalid document, or rejects a valid one. Root: `src/ruleset/`.
2. **Model bug** — a model accessor returns the wrong value or throws unexpectedly. Root: `src/models/v2/` or `src/models/v3/`.
3. **Reference resolution bug** — `$ref` to a file or URL is not resolved correctly. Root: `src/resolver.ts`.
4. **Custom operation bug** — traits not applied correctly, circular refs mishandled. Root: `src/custom-operations/`.
5. **Schema parser bug** — Avro/Protobuf payloads not parsed correctly. Root: `src/schema-parser/`.
6. **Browser bundle issue** — parser does not work in a browser environment. Root: `webpack.config.js`.
7. **Performance issue** — large documents parse slowly. Root: usually `$ref` resolution or Spectral rules.

---

## Next Step

You now have all the conceptual background needed. Move to [Chapter 2: Getting Started](../02-getting-started/01-prerequisites-and-installation.md) to install the project and write your first parse.
