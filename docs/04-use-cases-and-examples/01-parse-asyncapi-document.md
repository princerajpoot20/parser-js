# Use Case: Parse an AsyncAPI Document

> **Goal:** Parse real AsyncAPI 2.x and 3.x documents, walk the entire typed model tree, and inspect channels, operations, messages, and schemas.

---

## The Document

Save the following as `scratch/streetlights-v2.yaml` (same as in [Chapter 1](../01-foundations/03-asyncapi-spec-walkthrough.md)):

```yaml
asyncapi: '2.6.0'
info:
  title: Streetlights API
  version: '1.0.0'
  description: The Smartylighting Streetlights API.
  license:
    name: Apache 2.0
    url: https://www.apache.org/licenses/LICENSE-2.0
defaultContentType: application/json
servers:
  production:
    url: test.mosquitto.org
    protocol: mqtt
channels:
  smartylighting/streetlights/1/0/event/{streetlightId}/lighting/measured:
    description: The topic on which measured values may be produced.
    parameters:
      streetlightId:
        schema:
          type: string
    publish:
      operationId: receiveLightMeasurement
      summary: Inform about environmental lighting conditions.
      message:
        name: LightMeasured
        payload:
          type: object
          properties:
            lumens:
              type: integer
              minimum: 0
            sentAt:
              type: string
              format: date-time
  smartylighting/streetlights/1/0/action/{streetlightId}/turn/on:
    parameters:
      streetlightId:
        schema:
          type: string
    subscribe:
      operationId: turnOn
      message:
        name: TurnOnOff
        payload:
          type: object
          properties:
            command:
              type: string
              enum: [on, off]
            sentAt:
              type: string
              format: date-time
```

---

## Example: Walk the v2 Model Tree

Create `scratch/use-case-parse-v2.js`:

```js
// scratch/use-case-parse-v2.js
const { Parser } = require('../packages/parser/cjs/index.js');
const fs = require('fs');
const path = require('path');

async function main() {
  const parser = new Parser();

  // Load from file (note: pass source so $refs in the file resolve correctly)
  const filePath = path.join(__dirname, 'streetlights-v2.yaml');
  const asyncapiDoc = fs.readFileSync(filePath, 'utf-8');

  const { document, diagnostics } = await parser.parse(asyncapiDoc, {
    source: filePath,
  });

  if (!document) {
    console.log('Parse failed:');
    diagnostics.filter(d => d.severity === 0).forEach(d => console.log(' ', d.code, d.message));
    return;
  }

  // --- Top-Level Info ---
  console.log('=== Document Info ===');
  console.log('AsyncAPI version:', document.version());
  console.log('Title:', document.info().title());
  console.log('API version:', document.info().version());
  console.log('Description:', document.info().description());
  console.log('Default content type:', document.defaultContentType());

  // --- Servers ---
  console.log('\n=== Servers ===');
  document.servers().all().forEach(server => {
    console.log(`  [${server.id()}]`);
    console.log('    URL:', server.url());
    console.log('    Protocol:', server.protocol());
  });

  // --- Channels ---
  console.log('\n=== Channels ===');
  document.channels().all().forEach(channel => {
    console.log(`\n  Channel: ${channel.id()}`);
    if (channel.description()) {
      console.log('  Description:', channel.description());
    }

    // Parameters (URL template variables like {streetlightId})
    if (!channel.parameters().isEmpty()) {
      console.log('  Parameters:');
      channel.parameters().all().forEach(param => {
        console.log(`    {${param.id()}} - schema type: ${param.schema()?.type()}`);
      });
    }

    // Operations on this channel
    channel.operations().all().forEach(operation => {
      console.log(`\n  Operation: ${operation.id()}`);
      console.log('    Action:', operation.action()); // 'send' or 'receive'

      // Messages in this operation
      operation.messages().all().forEach(message => {
        console.log('    Message:', message.name() ?? '(anonymous)');

        // Access payload schema
        const payload = message.payload();
        if (payload) {
          console.log('    Payload type:', payload.type());
          const props = payload.properties();
          if (props) {
            Object.entries(props).forEach(([key, schema]) => {
              console.log(`      .${key}: ${schema.type()}`);
            });
          }
        }
      });
    });
  });

  // --- All Operations (convenience accessor) ---
  console.log('\n=== All Operations ===');
  document.operations().all().forEach(op => {
    console.log(`  ${op.id()} [${op.action()}]`);
  });

  // --- All Messages (unique, deduplicated) ---
  console.log('\n=== All Messages ===');
  document.messages().all().forEach(msg => {
    console.log(`  ${msg.name() ?? '(anonymous)'}`);
  });

  // --- Components ---
  const components = document.components();
  console.log('\n=== Components ===');
  console.log('  Component messages:', components.messages().all().map(m => m.id()).join(', ') || '(none)');
  console.log('  Component schemas:', components.schemas().all().map(s => s.id()).join(', ') || '(none)');

  // --- Diagnostics summary ---
  const warnings = diagnostics.filter(d => d.severity === 1);
  const errors = diagnostics.filter(d => d.severity === 0);
  console.log(`\n=== Diagnostics: ${errors.length} error(s), ${warnings.length} warning(s) ===`);
  diagnostics.forEach(d => {
    const severity = ['Error', 'Warning', 'Info', 'Hint'][d.severity];
    if (d.severity <= 1) { // show errors and warnings
      console.log(`  [${severity}] ${d.code}: ${d.message}`);
    }
  });
}

main().catch(console.error);
```

Run:

```bash
node scratch/use-case-parse-v2.js
```

Expected output:

```
=== Document Info ===
AsyncAPI version: 2.6.0
Title: Streetlights API
API version: 1.0.0
Description: The Smartylighting Streetlights API.
Default content type: application/json

=== Servers ===
  [production]
    URL: test.mosquitto.org
    Protocol: mqtt

=== Channels ===

  Channel: smartylighting/streetlights/1/0/event/{streetlightId}/lighting/measured
  Description: The topic on which measured values may be produced.
  Parameters:
    {streetlightId} - schema type: string

  Operation: receiveLightMeasurement
    Action: send
    Message: LightMeasured
    Payload type: object
      .lumens: integer
      .sentAt: string

  Channel: smartylighting/streetlights/1/0/action/{streetlightId}/turn/on
  Parameters:
    {streetlightId} - schema type: string

  Operation: turnOn
    Action: receive
    Message: TurnOnOff
    Payload type: object
      .command: string
      .sentAt: string

=== All Operations ===
  receiveLightMeasurement [send]
  turnOn [receive]

=== All Messages ===
  LightMeasured
  TurnOnOff

=== Components ===
  Component messages: (none)
  Component schemas: (none)

=== Diagnostics: 0 error(s), 3 warning(s) ===
  [Warning] asyncapi-id: AsyncAPI document should have "id" field.
  [Warning] asyncapi-info-contact: Info object should have "contact" object.
  [Warning] asyncapi-servers: ...
```

> **Note on action values:** In AsyncAPI v2, channels have `publish` and `subscribe`. The parser's model maps these to the `action` concept: `publish` → `'send'` (the app sends), `subscribe` → `'receive'` (the app receives). This aligns with v3 terminology.

---

## Example: Walk the v3 Model Tree

Create `scratch/use-case-parse-v3.js`:

```js
// scratch/use-case-parse-v3.js
const { Parser } = require('../packages/parser/cjs/index.js');

async function main() {
  const parser = new Parser();

  const asyncapiDoc = `
asyncapi: '3.0.0'
info:
  title: Streetlights API
  version: '1.0.0'
channels:
  lightingMeasured:
    address: 'smartylighting/streetlights/1/0/event/{streetlightId}/lighting/measured'
    messages:
      lightMeasured:
        name: LightMeasured
        payload:
          type: object
          properties:
            lumens:
              type: integer
              minimum: 0
            sentAt:
              type: string
              format: date-time
  turnOn:
    address: 'smartylighting/streetlights/1/0/action/{streetlightId}/turn/on'
    messages:
      turnOnOff:
        name: TurnOnOff
        payload:
          type: object
          properties:
            command:
              type: string
              enum: [on, off]
operations:
  receiveLightMeasurement:
    action: receive
    channel:
      $ref: '#/channels/lightingMeasured'
    messages:
      - $ref: '#/channels/lightingMeasured/messages/lightMeasured'
  sendTurnOn:
    action: send
    channel:
      $ref: '#/channels/turnOn'
    messages:
      - $ref: '#/channels/turnOn/messages/turnOnOff'
`;

  const { document } = await parser.parse(asyncapiDoc);
  if (!document) return;

  // KEY DIFFERENCE in v3: operations are top-level, not inside channels
  console.log('=== V3 Top-Level Operations ===');
  document.operations().all().forEach(op => {
    console.log(`\n  Operation: ${op.id()}`);
    console.log('  Action:', op.action());  // 'send' or 'receive'
    
    // In v3, each operation references a channel
    const ch = op.channel();
    console.log('  Channel:', ch.id(), '→ address:', ch.address());
    
    // Messages for this operation
    op.messages().all().forEach(msg => {
      console.log('  Message:', msg.name() ?? msg.id());
    });
  });

  console.log('\n=== V3 Channels ===');
  document.channels().all().forEach(channel => {
    console.log(`\n  Channel id: ${channel.id()}`);
    console.log('  Address:', channel.address());
    
    // In v3, channels have messages listed directly
    console.log('  Messages on channel:');
    channel.messages().all().forEach(msg => {
      console.log(`    - ${msg.id()} (${msg.name() ?? 'no name'})`);
    });
  });
}

main().catch(console.error);
```

---

## Key API Differences: v2 vs v3

| What you want | v2 code | v3 code |
|---------------|---------|---------|
| All channels | `doc.channels().all()` | `doc.channels().all()` |
| Channel address | `channel.id()` | `channel.address()` |
| Operations in a channel | `channel.operations().all()` | N/A — operations are top-level |
| All top-level operations | `doc.operations().all()` (reads from channels) | `doc.operations().all()` (direct) |
| Operation's channel | N/A — `channel.operations()` is the relationship | `op.channel()` |
| Operation direction | `op.action()` → `'send'` or `'receive'` | Same |
| Messages in operation | `op.messages().all()` | `op.messages().all()` |
| Messages on channel | `channel.messages().all()` (derived from ops) | `channel.messages().all()` (direct) |

---

## Accessing Raw JSON

When you need a field that is not exposed by the typed API, use `.json()`:

```js
// Get the raw channel object
const rawChannel = channel.json();
console.log(rawChannel); // Full JSON, including x-parser-* extensions

// Get a specific raw field
const rawBindings = channel.json('bindings');

// Get the JSON pointer path (useful for diagnostics)
console.log(channel.jsonPath()); // '/channels/smartylighting~1...'
```

---

## Next Step

Read [02-validate-only-no-model.md](./02-validate-only-no-model.md) to learn how to use the parser purely for validation without building a model.
