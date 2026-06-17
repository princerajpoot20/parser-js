# Monorepo Project Structure

> **Goal:** Know exactly where everything lives in this repository so you can navigate to the right file when reading an issue or debugging a problem.

---

## Top-Level Layout

```
parser-js/
│
├── packages/
│   ├── parser/           ← @asyncapi/parser (the main library)
│   └── multi-parser/     ← @asyncapi/multi-parser (multi-version wrapper)
│
├── docs/                 ← This documentation (you are here)
│
├── assets/
│   └── logo.png
│
├── .github/
│   ├── pull-request-template.md
│   └── workflows/        ← 27 CI workflow files
│
├── .changeset/           ← Changesets for version management
├── package.json          ← Monorepo root (workspaces, turbo scripts)
├── turbo.json            ← Turborepo pipeline configuration
├── README.md             ← Main project README
├── CONTRIBUTING.md       ← How to contribute
├── CODEOWNERS            ← Code ownership rules
├── .eslintrc             ← Shared ESLint configuration
├── .eslintignore
├── .gitignore
└── .releaserc            ← Semantic release config
```

---

## `packages/parser/` — The Main Library

This is where you will spend almost all your time as a maintainer.

```
packages/parser/
│
├── src/                  ← All TypeScript source code
│   ├── parser.ts         ← Parser class (public API entry point)
│   ├── parse.ts          ← parse() pipeline
│   ├── validate.ts       ← validate() via Spectral
│   ├── document.ts       ← Document factory and type guards
│   ├── spectral.ts       ← Spectral instance factory
│   ├── resolver.ts       ← $ref resolution (file/http)
│   ├── from.ts           ← fromURL() / fromFile() helpers
│   ├── stringify.ts      ← serialize/deserialize parsed docs
│   ├── iterator.ts       ← Schema traversal utilities
│   ├── types.ts          ← Core shared types (Input, Diagnostic, etc.)
│   ├── constants.ts      ← x-parser-* extension keys, spec versions
│   ├── utils.ts          ← Shared helper functions
│   ├── index.ts          ← Public exports (the library's surface area)
│   │
│   ├── models/           ← New (Intent) API model classes
│   │   ├── base.ts       ← BaseModel — root class for all models
│   │   ├── asyncapi.ts   ← AsyncAPIDocumentInterface
│   │   ├── collection.ts ← Collection<T> base class
│   │   ├── mixins.ts     ← Shared mixins (extensions, tags, etc.)
│   │   ├── channel.ts    ← ChannelInterface
│   │   ├── message.ts    ← MessageInterface
│   │   ├── operation.ts  ← OperationInterface
│   │   ├── schema.ts     ← SchemaInterface (JSON Schema wrapper)
│   │   ├── server.ts     ← ServerInterface
│   │   ├── components.ts ← ComponentsInterface
│   │   ├── info.ts       ← InfoInterface
│   │   ├── ... (40+ interface files)
│   │   ├── v2/           ← AsyncAPI 2.x concrete implementations
│   │   │   ├── asyncapi.ts     ← AsyncAPIDocumentV2
│   │   │   ├── channel.ts      ← ChannelV2
│   │   │   ├── operation.ts    ← OperationV2
│   │   │   └── ... (35 files)
│   │   └── v3/           ← AsyncAPI 3.x concrete implementations
│   │       ├── asyncapi.ts     ← AsyncAPIDocumentV3
│   │       ├── channel.ts      ← ChannelV3
│   │       ├── operation.ts    ← OperationV3
│   │       ├── operation-reply.ts ← OperationReplyV3
│   │       └── ... (45 files)
│   │
│   ├── custom-operations/ ← Post-parse transformations
│   │   ├── index.ts           ← Dispatcher (v2 vs v3 pipeline)
│   │   ├── apply-traits.ts    ← Merge traits into operations/messages
│   │   ├── apply-unique-ids.ts ← Assign x-parser-unique-object-id
│   │   ├── anonymous-naming.ts ← Auto-name anonymous messages/schemas
│   │   ├── parse-schema.ts    ← Invoke custom schema parsers on payloads
│   │   ├── check-circular-refs.ts   ← Detect circular $ref chains
│   │   └── resolve-circular-refs.ts ← Mark circular refs in model
│   │
│   ├── ruleset/           ← Spectral validation rules
│   │   ├── ruleset.ts         ← Core + recommended rules (version-agnostic)
│   │   ├── formats.ts         ← Format detection (which spec version)
│   │   ├── index.ts
│   │   ├── functions/         ← Custom rule functions
│   │   │   ├── documentStructure.ts  ← Ajv-based spec validation
│   │   │   ├── unusedComponent.ts
│   │   │   └── ... (15+ functions)
│   │   ├── utils/             ← Rule utility helpers
│   │   ├── v2/                ← AsyncAPI 2.x-specific rules
│   │   │   ├── ruleset.ts
│   │   │   └── functions/     ← v2 rule functions
│   │   └── v3/                ← AsyncAPI 3.x-specific rules
│   │       ├── ruleset.ts
│   │       └── functions/
│   │
│   ├── schema-parser/     ← Pluggable schema format system
│   │   ├── index.ts           ← SchemaParser interface, registerSchemaParser
│   │   ├── asyncapi-schema-parser.ts ← Built-in JSON Schema (Ajv) parser
│   │   └── spectral-rule-v2.ts ← Spectral rule for v2 schema validation
│   │
│   ├── old-api/           ← Legacy Parser API compatibility
│   │   ├── index.ts
│   │   ├── converter.ts   ← convertToOldAPI / convertToNewAPI
│   │   ├── asyncapi.ts    ← OldAsyncAPIDocument
│   │   └── ... (20+ files)
│   │
│   └── spec-types/        ← TypeScript types from @asyncapi/specs
│       ├── v2.ts          ← AsyncAPI 2.x spec type definitions
│       └── v3.ts          ← AsyncAPI 3.x spec type definitions
│
├── test/                  ← All test files
│   ├── parse.spec.ts       ← Core parse() tests
│   ├── validate.spec.ts    ← Core validate() tests
│   ├── parser.spec.ts      ← Parser class tests
│   ├── from.spec.ts        ← fromURL/fromFile tests
│   ├── resolver.spec.ts    ← $ref resolution tests
│   ├── stringify.spec.ts   ← stringify/unstringify tests
│   ├── iterator.spec.ts    ← Schema traversal tests
│   ├── document.spec.ts    ← Document creation tests
│   ├── spectral.spec.ts    ← Spectral integration tests
│   ├── utils.spec.ts       ← Utility function tests
│   ├── models/
│   │   ├── v2/             ← 35 v2 model unit tests
│   │   └── v3/             ← 23 v3 model unit tests
│   ├── ruleset/
│   │   └── rules/
│   │       ├── *.spec.ts   ← Core rule tests
│   │       ├── v2/         ← v2 rule tests (25+ files)
│   │       └── v3/         ← v3 rule tests (5 files)
│   ├── custom-operations/  ← 9 custom operation tests
│   ├── old-api/            ← 23 legacy API tests
│   ├── schema-parser/      ← 2 schema parser tests
│   ├── mocks/              ← YAML fixture files
│   │   ├── simple.yaml
│   │   ├── simple-with-refs.yaml
│   │   ├── circular-refs.yaml
│   │   └── ... (11 files total)
│   ├── browser/            ← Browser bundle tests (Playwright)
│   └── utils.ts            ← Test utilities shared across suites
│
├── docs/                  ← Package-level docs (ruleset + migration)
│   ├── migrations/
│   │   ├── v1-to-v2.md
│   │   └── v2-to-v3.md
│   └── ruleset/           ← Ruleset documentation
│
├── package.json           ← Package manifest (@asyncapi/parser)
├── tsconfig.json          ← TypeScript config (ESM output)
├── tsconfig.cjs.json      ← TypeScript config (CJS output)
├── jest.config.ts         ← Jest configuration
└── webpack.config.js      ← Browser bundle configuration
```

---

## `packages/multi-parser/` — Multi-Version Wrapper

Simpler structure:

```
packages/multi-parser/
├── src/
│   ├── index.ts           ← NewParser(), ConvertDocumentParserAPIVersion()
│   └── parser.ts
├── test/
│   ├── parse.spec.ts
│   └── convert.spec.ts
├── package.json
├── tsconfig.json
├── tsconfig.cjs.json
└── jest.config.ts
```

---

## Key Source File Index

Quick lookup when you know what you're looking for:

| Question | File |
|----------|------|
| Where is the `Parser` class? | `packages/parser/src/parser.ts` |
| Where does the parse pipeline run? | `packages/parser/src/parse.ts` |
| Where does Spectral validation happen? | `packages/parser/src/validate.ts` |
| Where are `$ref`s resolved? | `packages/parser/src/spectral.ts` + `resolver.ts` |
| Where is the document model created? | `packages/parser/src/document.ts` |
| Where is the v2 document model class? | `packages/parser/src/models/v2/asyncapi.ts` |
| Where is the v3 document model class? | `packages/parser/src/models/v3/asyncapi.ts` |
| Where are traits applied? | `packages/parser/src/custom-operations/apply-traits.ts` |
| Where are unique IDs assigned? | `packages/parser/src/custom-operations/apply-unique-ids.ts` |
| Where are circular refs handled? | `packages/parser/src/custom-operations/check-circular-refs.ts` |
| Where is the JSON Schema validator? | `packages/parser/src/schema-parser/asyncapi-schema-parser.ts` |
| Where are core Spectral rules? | `packages/parser/src/ruleset/ruleset.ts` |
| Where are v2-specific rules? | `packages/parser/src/ruleset/v2/ruleset.ts` |
| Where are v3-specific rules? | `packages/parser/src/ruleset/v3/ruleset.ts` |
| Where is the old API? | `packages/parser/src/old-api/` |
| Where are all exports? | `packages/parser/src/index.ts` |
| Where are extension key constants? | `packages/parser/src/constants.ts` |

---

## `turbo.json` — Pipeline Configuration

Turborepo orchestrates build and test tasks across packages. The key pipeline:

```json
{
  "pipeline": {
    "test": {
      "dependsOn": ["@asyncapi/parser#build"]  // Always build parser before any test
    },
    "build": {
      "dependsOn": ["@asyncapi/parser#build"]  // multi-parser build depends on parser build
    },
    "@asyncapi/parser#build": {}               // Parser build has no dependencies
  }
}
```

This means:
- `npm test` triggers: build parser → build multi-parser → run all tests
- `npm run parser:test:unit` does NOT trigger a build (runs Jest directly)

---

## `.github/workflows/` — CI Workflows

| Workflow | Trigger | What it does |
|----------|---------|-------------|
| `if-nodejs-pr-testing.yml` | PR opened/updated | Runs `npm ci && npm test` on Ubuntu, macOS, Windows. Lints on Ubuntu. |
| `if-nodejs-release.yml` | Push to master | Test + release |
| `release-with-changesets.yml` | PR to master | Changeset-based release process |
| `automerge.yml` | Dependabot PRs | Auto-merges passing dependency updates |
| `lint-pr-title.yml` | PR opened | Validates Conventional Commits format |
| `stale-issues-prs.yml` | Schedule | Marks and closes stale issues |
| `bump.yml` | After release | Bumps parser version in dependent AsyncAPI repos |

Most of these workflows come from the centralized [asyncapi/.github](https://github.com/asyncapi/.github) repo and should **not** be edited in this repo.

---

## `packages/parser/docs/` — Package-level Documentation

Distinct from this documentation (`/docs/` at repo root), the package has its own docs:

```
packages/parser/docs/
├── migrations/
│   ├── v1-to-v2.md    ← How to upgrade from parser v1 to v2
│   └── v2-to-v3.md    ← How to upgrade from parser v2 to v3 (Intent API)
└── ruleset/
    ├── core-ruleset.md
    └── recommended-ruleset.md
```

These focus on the user-facing upgrade path. Our `/docs/` focuses on internal architecture and maintainer knowledge.

---

## Next Step

Read [04-running-tests-locally.md](./04-running-tests-locally.md) to learn how to run, filter, and interpret the test suite.
