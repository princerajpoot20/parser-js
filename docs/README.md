# AsyncAPI parser-js — Knowledge Base

This documentation suite takes you from zero background in event-driven architecture to a full working knowledge of the `asyncapi/parser-js` monorepo. It is structured as a book: each chapter builds on the previous one, and every section includes runnable code examples you can verify locally.

> **Who this is for:** New maintainers, contributors, and anyone who wants to deeply understand how `@asyncapi/parser` works internally — not just how to call its API.

---

## Reading Order

If you are completely new, read front-to-back. If you have a specific goal, jump to the relevant section.

| # | Section | What you will learn |
|---|---------|---------------------|
| 1 | [Foundations](#1-foundations) | Event-driven architecture, AsyncAPI spec, where the parser fits |
| 2 | [Getting Started](#2-getting-started) | Install, build, first parse, run tests |
| 3 | [Architecture](#3-architecture) | Internal design, pipeline, validation, models |
| 4 | [Use Cases & Examples](#4-use-cases--examples) | Hands-on scenarios with runnable code |
| 5 | [Model API](#5-model-api) | Navigating parsed documents, old vs new API |
| 6 | [Validation Ruleset](#6-validation-ruleset) | Spectral rules deep dive |
| 7 | [Multi-Parser](#7-multi-parser) | Handling multiple parser API versions |
| 8 | [Contributor Guide](#8-contributor-guide) | Triaging issues, debugging, adding features |

---

## 1. Foundations

Start here if you have never worked with event-driven systems or AsyncAPI before.

| File | Description |
|------|-------------|
| [01-event-driven-architecture.md](./01-foundations/01-event-driven-architecture.md) | What EDA is, how it differs from REST, key vocabulary (broker, channel, producer, consumer) |
| [02-what-is-asyncapi.md](./01-foundations/02-what-is-asyncapi.md) | AsyncAPI spec purpose, analogy to OpenAPI, spec 2.x vs 3.x overview |
| [03-asyncapi-spec-walkthrough.md](./01-foundations/03-asyncapi-spec-walkthrough.md) | Fully annotated real AsyncAPI documents (v2 and v3), `$ref`, `schemaFormat` explained |
| [04-where-parser-js-fits.md](./01-foundations/04-where-parser-js-fits.md) | The AsyncAPI tooling ecosystem, why a parser is needed, what this library does |

---

## 2. Getting Started

Get the project running locally and parse your first document.

| File | Description |
|------|-------------|
| [01-prerequisites-and-installation.md](./02-getting-started/01-prerequisites-and-installation.md) | Node ≥ 18, clone, install, build, browser bundle |
| [02-your-first-parse.md](./02-getting-started/02-your-first-parse.md) | Complete runnable scripts for v2 and v3, inspecting output, intentional errors |
| [03-monorepo-project-structure.md](./02-getting-started/03-monorepo-project-structure.md) | Annotated directory tree, key source files, `turbo.json`, CI workflows |
| [04-running-tests-locally.md](./02-getting-started/04-running-tests-locally.md) | All test commands, single-test runs, browser tests, coverage |

---

## 3. Architecture

Understand how the parser works internally from input to output model.

| File | Description |
|------|-------------|
| [01-high-level-architecture.md](./03-architecture/01-high-level-architecture.md) | Full system diagram, `parse()` vs `validate()`, Parser constructor options |
| [02-parsing-pipeline-deep-dive.md](./03-architecture/02-parsing-pipeline-deep-dive.md) | Step-by-step code walkthrough of `parse.ts`, all intermediate states |
| [03-validation-layer.md](./03-architecture/03-validation-layer.md) | Spectral integration, Ajv, severity gating, diagnostic structure |
| [04-model-layer.md](./03-architecture/04-model-layer.md) | `BaseModel`, `Collection`, `AsyncAPIDocumentInterface`, v2 vs v3 differences |
| [05-custom-operations.md](./03-architecture/05-custom-operations.md) | Traits, unique IDs, schema parsing, circular ref detection/resolution |
| [06-schema-parser-system.md](./03-architecture/06-schema-parser-system.md) | `SchemaParser` interface, built-in JSON Schema parser, pluggable formats |

---

## 4. Use Cases & Examples

Each document explains one scenario with a fully annotated AsyncAPI document, working code, and expected output.

| File | Description |
|------|-------------|
| [01-parse-asyncapi-document.md](./04-use-cases-and-examples/01-parse-asyncapi-document.md) | Parse v2 and v3 docs, walk the model tree, inspect channels/operations/messages |
| [02-validate-only-no-model.md](./04-use-cases-and-examples/02-validate-only-no-model.md) | Use `validate()` for CI linting, intentional errors, custom ruleset |
| [03-working-with-refs.md](./04-use-cases-and-examples/03-working-with-refs.md) | Internal `$ref`, external file refs, HTTP refs, inspecting resolved vs raw |
| [04-load-from-url-and-file.md](./04-use-cases-and-examples/04-load-from-url-and-file.md) | `fromFile()`, `fromURL()`, error handling |
| [05-circular-references.md](./04-use-cases-and-examples/05-circular-references.md) | What circular refs are, detection, `x-parser-circular`, handling strategies |
| [06-custom-schema-formats.md](./04-use-cases-and-examples/06-custom-schema-formats.md) | Register Avro parser, parse Avro payload, inspect converted schema |
| [07-stringify-and-unstringify.md](./04-use-cases-and-examples/07-stringify-and-unstringify.md) | Serialize/deserialize parsed docs, caching, circular-ref-safe serialization |

---

## 5. Model API

Deep dive into navigating the parsed document tree.

| File | Description |
|------|-------------|
| [01-new-api-vs-old-api.md](./05-model-api/01-new-api-vs-old-api.md) | History of Parser API versions, old-api vs Intent API, conversion utilities |
| [02-navigating-v2-documents.md](./05-model-api/02-navigating-v2-documents.md) | Full v2 model method tree, channels → operations → messages → schema |
| [03-navigating-v3-documents.md](./05-model-api/03-navigating-v3-documents.md) | Top-level operations model, replies, channel references |
| [04-parser-api-versioning.md](./05-model-api/04-parser-api-versioning.md) | Parser-API spec, version matrix, `@asyncapi/multi-parser` context |

---

## 6. Validation Ruleset

How Spectral rules validate AsyncAPI documents.

| File | Description |
|------|-------------|
| [01-spectral-integration.md](./06-validation-ruleset/01-spectral-integration.md) | Spectral internals, `createSpectral()`, resolver wiring, writing custom rules |
| [02-core-rules.md](./06-validation-ruleset/02-core-rules.md) | All core rules with severity table, `documentStructure` deep dive |
| [03-v2-rules.md](./06-validation-ruleset/03-v2-rules.md) | All 25+ v2 validation rules, 3 representative deep dives |
| [04-v3-rules.md](./06-validation-ruleset/04-v3-rules.md) | All v3 validation rules, unique v3 constraints |

---

## 7. Multi-Parser

Handle multiple Parser API versions in a single tool.

| File | Description |
|------|-------------|
| [01-multi-parser-guide.md](./07-multi-parser/01-multi-parser-guide.md) | `@asyncapi/multi-parser`, `NewParser()`, `ConvertDocumentParserAPIVersion()` |

---

## 8. Contributor Guide

Everything you need to triage issues and contribute effectively.

| File | Description |
|------|-------------|
| [01-understanding-github-issues.md](./08-contributor-guide/01-understanding-github-issues.md) | Issue taxonomy, decision flowchart, reproducing issues locally |
| [02-debugging-and-tracing.md](./08-contributor-guide/02-debugging-and-tracing.md) | Debug logging, inspecting intermediate state, CI failure patterns |
| [03-adding-new-validation-rules.md](./08-contributor-guide/03-adding-new-validation-rules.md) | Step-by-step guide to adding a Spectral rule |
| [04-extending-schema-parsers.md](./08-contributor-guide/04-extending-schema-parsers.md) | Build a custom `SchemaParser`, register it, test it |

---

## 9. Open GitHub Issues

Deep-dive analysis of every currently open issue — what it is, why it happens, how to reproduce it, and how to fix it.

| File | Description |
|------|-------------|
| [README.md](./09-open-issues/README.md) | All 20 open issues explained, from beginner to expert, with reproduction scripts and fix guidance |

---

## Quick Reference

### Key Source Files

| File | Role |
|------|------|
| `packages/parser/src/parser.ts` | `Parser` class — public entry point |
| `packages/parser/src/parse.ts` | Parse pipeline orchestration |
| `packages/parser/src/validate.ts` | Spectral validation wrapper |
| `packages/parser/src/document.ts` | Document factory and type guards |
| `packages/parser/src/models/` | All Intent API model classes |
| `packages/parser/src/ruleset/` | Spectral rules and functions |
| `packages/parser/src/custom-operations/` | Post-parse transformations |
| `packages/parser/src/schema-parser/` | Pluggable schema format parsers |
| `packages/parser/src/old-api/` | Legacy API compatibility layer |

### Key Test Commands

```bash
# Full build + all tests
npm test

# Parser unit tests only (fastest iteration)
npm run parser:test:unit

# Single spec file
npx jest --testPathPattern="parse.spec.ts" --rootdir packages/parser

# Single test by name
npx jest --testPathPattern="parse.spec.ts" -t "should parse valid document" --rootdir packages/parser
```

### Packages

| Package | npm | Purpose |
|---------|-----|---------|
| `@asyncapi/parser` | [npmjs.com](https://www.npmjs.com/package/@asyncapi/parser) | Parse, validate, and model AsyncAPI documents |
| `@asyncapi/multi-parser` | [npmjs.com](https://www.npmjs.com/package/@asyncapi/multi-parser) | Support multiple Parser API versions in one tool |

### External Links

- [AsyncAPI Specification](https://www.asyncapi.com/docs/reference/specification/latest)
- [Parser-API (Intent API spec)](https://github.com/asyncapi/parser-api)
- [Spectral (validation engine)](https://github.com/stoplightio/spectral)
- [GitHub Issues](https://github.com/asyncapi/parser-js/issues)
- [AsyncAPI Studio](https://studio.asyncapi.com) — uses this parser
