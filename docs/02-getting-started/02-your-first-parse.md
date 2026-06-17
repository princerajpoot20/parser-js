# Your First Parse

> **Goal:** Write and run a complete script that parses an AsyncAPI document, inspects the typed model, and handles diagnostics. By the end you will have run the parser locally and seen real output.

---

## Setup

Create a working directory for experiments. You can put this anywhere — we will use a `scratch/` folder inside the repo (it is already gitignored via `node_modules` proximity):

```bash
mkdir -p scratch
cd scratch
```

The examples below use CommonJS (`require`) so they run with plain `node`. The built package is at `packages/parser/cjs/index.js`. Make sure you have run `npm run build` first.

---

## Example 1: Parse a Valid AsyncAPI 2.x Document

Create `scratch/example1-parse-v2.js`:

```js
// scratch/example1-parse-v2.js
const { Parser } = require('../packages/parser/cjs/index.js');

async function main() {
  const parser = new Parser();

  // A minimal but valid AsyncAPI 2.6.0 document as a JavaScript string
  const asyncapiDoc = `
asyncapi: '2.6.0'
info:
  title: Streetlights API
  version: '1.0.0'
defaultContentType: application/json
channels:
  smartylighting/streetlights/1/0/event/lighting/measured:
    publish:
      operationId: receiveLightMeasurement
      message:
        payload:
          type: object
          properties:
            lumens:
              type: integer
              minimum: 0
            sentAt:
              type: string
              format: date-time
`;

  const { document, diagnostics } = await parser.parse(asyncapiDoc);

  // document is an AsyncAPIDocumentV2 instance (or undefined if invalid)
  if (document) {
    console.log('=== Parse Successful ===');
    console.log('Title:', document.info().title());
    console.log('Version:', document.info().version());
    console.log('AsyncAPI version:', document.version());

    console.log('\n=== Channels ===');
    document.channels().all().forEach(channel => {
      console.log('  Channel:', channel.id());
      channel.operations().all().forEach(op => {
        console.log('    Operation:', op.id(), '- action:', op.action());
        op.messages().all().forEach(msg => {
          console.log('      Message payload type:', msg.payload()?.type());
        });
      });
    });
  }

  // diagnostics is always an array (never undefined), even for valid documents
  // For a valid doc, it will have warnings/hints but no errors
  if (diagnostics.length > 0) {
    console.log('\n=== Diagnostics ===');
    diagnostics.forEach(d => {
      const severity = ['Error', 'Warning', 'Info', 'Hint'][d.severity];
      console.log(`  [${severity}] ${d.code}: ${d.message}`);
    });
  }
}

main().catch(console.error);
```

Run it:

```bash
# From repo root:
node scratch/example1-parse-v2.js
```

Expected output:

```
=== Parse Successful ===
Title: Streetlights API
Version: 1.0.0
AsyncAPI version: 2.6.0

=== Channels ===
  Channel: smartylighting/streetlights/1/0/event/lighting/measured
    Operation: receiveLightMeasurement - action: send
      Message payload type: object

=== Diagnostics ===
  [Hint] asyncapi-latest-version: Update to the latest AsyncAPI version...
```

> **Why do diagnostics appear for a valid document?** Because Spectral runs all configured rules, and some are informational (Info/Hint level). Only `DiagnosticSeverity.Error` blocks parsing. The "latest version" hint tells you a newer AsyncAPI spec version is available — it is not an error.

---

## Example 2: Parse an AsyncAPI 3.x Document

Create `scratch/example2-parse-v3.js`:

```js
// scratch/example2-parse-v3.js
const { Parser } = require('../packages/parser/cjs/index.js');

async function main() {
  const parser = new Parser();

  const asyncapiDoc = `
asyncapi: '3.0.0'
info:
  title: Streetlights API
  version: '1.0.0'
defaultContentType: application/json
channels:
  lightingMeasured:
    address: 'smartylighting/streetlights/1/0/event/lighting/measured'
    messages:
      lightMeasured:
        payload:
          type: object
          properties:
            lumens:
              type: integer
              minimum: 0
            sentAt:
              type: string
              format: date-time
operations:
  receiveLightMeasurement:
    action: receive
    channel:
      $ref: '#/channels/lightingMeasured'
    messages:
      - $ref: '#/channels/lightingMeasured/messages/lightMeasured'
`;

  const { document, diagnostics } = await parser.parse(asyncapiDoc);

  if (document) {
    console.log('=== Parse Successful (v3) ===');
    console.log('AsyncAPI version:', document.version());

    console.log('\n=== Channels ===');
    document.channels().all().forEach(channel => {
      console.log('  Channel id:', channel.id(), '-> address:', channel.address());
    });

    console.log('\n=== Operations (v3 top-level) ===');
    document.operations().all().forEach(op => {
      console.log('  Operation:', op.id());
      console.log('    Action:', op.action());  // 'receive' or 'send'
      console.log('    Channel:', op.channel().id());
      op.messages().all().forEach(msg => {
        console.log('    Message:', msg.id());
      });
    });
  }

  const errors = diagnostics.filter(d => d.severity === 0); // 0 = Error
  if (errors.length > 0) {
    console.log('\n=== Errors ===');
    errors.forEach(d => console.log(`  ${d.code}: ${d.message}`));
  }
}

main().catch(console.error);
```

Run it:

```bash
node scratch/example2-parse-v3.js
```

Expected output:

```
=== Parse Successful (v3) ===
AsyncAPI version: 3.0.0

=== Channels ===
  Channel id: lightingMeasured -> address: smartylighting/streetlights/1/0/event/lighting/measured

=== Operations (v3 top-level) ===
  Operation: receiveLightMeasurement
    Action: receive
    Channel: lightingMeasured
    Message: lightMeasured
```

---

## Example 3: Inspect Diagnostics from an Invalid Document

Create `scratch/example3-invalid.js`:

```js
// scratch/example3-invalid.js
const { Parser } = require('../packages/parser/cjs/index.js');

async function main() {
  const parser = new Parser();

  // This document has multiple intentional errors:
  // 1. Missing required 'info.title'
  // 2. Duplicate operationId (same id used twice)
  // 3. Channel parameter {streetlightId} not defined in parameters
  const invalidDoc = `
asyncapi: '2.6.0'
info:
  version: '1.0.0'
channels:
  smartylighting/streetlights/{streetlightId}/measured:
    publish:
      operationId: duplicateOp
      message:
        payload:
          type: object
  other/channel:
    publish:
      operationId: duplicateOp
      message:
        payload:
          type: string
`;

  const { document, diagnostics } = await parser.parse(invalidDoc);

  console.log('Document:', document ? 'created' : 'UNDEFINED (errors blocked parsing)');
  console.log('Total diagnostics:', diagnostics.length);
  console.log();

  const severityNames = ['Error', 'Warning', 'Info', 'Hint'];
  diagnostics.forEach(d => {
    console.log(`[${severityNames[d.severity]}] ${d.code}`);
    console.log(`  Message: ${d.message}`);
    console.log(`  Path: ${d.path.join(' > ') || '(root)'}`);
    if (d.range) {
      const { start, end } = d.range;
      console.log(`  Location: line ${start.line + 1}, col ${start.character + 1}`);
    }
    console.log();
  });
}

main().catch(console.error);
```

Run it:

```bash
node scratch/example3-invalid.js
```

Expected output (abbreviated):

```
Document: UNDEFINED (errors blocked parsing)
Total diagnostics: 4

[Error] asyncapi-document-unresolved
  Message: Object must have required property "title"
  Path: info
  Location: line 4, col 1

[Error] asyncapi2-operation-operationId-uniqueness
  Message: "duplicateOp" operationId must be unique across all the operations.
  Path: channels > other/channel > publish > operationId
  Location: line 14, col 17

[Warning] asyncapi2-channel-parameters
  Message: ...
```

> **Key insight:** When there are errors, `document` is `undefined`. The parser cannot construct a reliable typed model from an invalid document. Warnings and hints do not block document creation.

---

## Understanding the Diagnostic Object

Each item in the `diagnostics` array has this shape:

```typescript
{
  code: string;          // rule name, e.g. 'asyncapi-document-unresolved'
  message: string;       // human-readable description
  path: string[];        // JSON path to the offending element
  severity: number;      // 0=Error, 1=Warning, 2=Info, 3=Hint
  range: {
    start: { line: number; character: number };
    end:   { line: number; character: number };
  };
  source?: string;       // file path if parsing from a file
}
```

The `DiagnosticSeverity` enum re-exported from the parser maps numbers to names:

```js
const { DiagnosticSeverity } = require('../packages/parser/cjs/index.js');

console.log(DiagnosticSeverity.Error);    // 0
console.log(DiagnosticSeverity.Warning);  // 1
console.log(DiagnosticSeverity.Information); // 2
console.log(DiagnosticSeverity.Hint);     // 3
```

---

## Example 4: Using `validate()` Instead of `parse()`

If you only want to check if a document is valid (CI linting, pre-commit hook) without building the model tree, use `validate()`:

```js
// scratch/example4-validate-only.js
const { Parser } = require('../packages/parser/cjs/index.js');

async function main() {
  const parser = new Parser();

  const doc = `
asyncapi: '2.6.0'
info:
  title: My API
  version: '1.0.0'
channels: {}
`;

  // validate() returns Diagnostic[] only — no document model created
  const diagnostics = await parser.validate(doc);

  const errors = diagnostics.filter(d => d.severity === 0);
  if (errors.length === 0) {
    console.log('Document is valid (no errors).');
  } else {
    console.log('Validation errors:');
    errors.forEach(d => console.log(`  ${d.code}: ${d.message}`));
    process.exit(1);
  }
}

main().catch(console.error);
```

---

## What to Try Next

- Modify the streetlights YAML in Example 1 — add a server, add a second channel, nest schemas
- Intentionally break the schema (e.g., set `type: notavalidtype`) and see which rule fires
- Try parsing the YAML files from the spec walkthrough: copy `streetlights-v2.yaml` from [Chapter 1](../01-foundations/03-asyncapi-spec-walkthrough.md) and pass it as a string

---

## Next Step

Read [03-monorepo-project-structure.md](./03-monorepo-project-structure.md) to understand exactly how the repository is organized before you start reading source code.
