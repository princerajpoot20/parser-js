# Navigating v2 Documents

> **Goal:** Complete walkthrough of the AsyncAPIDocumentV2 model tree — every major accessor with code examples and expected output.

---

## The v2 Model Tree

```
AsyncAPIDocumentV2
├── version()
├── defaultContentType()
├── info() → InfoInterface
│   ├── title()
│   ├── version()
│   ├── description()
│   ├── license() → LicenseInterface
│   │   ├── name()
│   │   └── url()
│   └── contact() → ContactInterface
│       ├── name()
│       ├── url()
│       └── email()
│
├── servers() → ServersInterface (Collection<ServerInterface>)
│   └── [server] → ServerInterface
│       ├── id()
│       ├── url()
│       ├── protocol()
│       ├── description()
│       ├── variables() → ServerVariablesInterface
│       ├── security() → SecurityRequirementsInterface
│       └── bindings() → BindingsInterface
│
├── channels() → ChannelsInterface (Collection<ChannelInterface>)
│   └── [channel] → ChannelV2
│       ├── id()          (= channel address in v2)
│       ├── address()     (= channel address in v2)
│       ├── description()
│       ├── parameters() → ChannelParametersInterface
│       ├── bindings() → BindingsInterface
│       └── operations() → OperationsInterface
│           └── [operation] → OperationV2
│               ├── id()   (= operationId)
│               ├── action() → 'send' | 'receive'
│               ├── summary()
│               ├── description()
│               ├── tags() → TagsInterface
│               ├── externalDocs()
│               ├── bindings() → BindingsInterface
│               ├── security() → SecurityRequirementsInterface
│               ├── traits() → OperationTraitsInterface
│               └── messages() → MessagesInterface
│                   └── [message] → MessageV2
│                       ├── id()
│                       ├── name()
│                       ├── title()
│                       ├── summary()
│                       ├── description()
│                       ├── contentType()
│                       ├── headers() → SchemaInterface
│                       ├── payload() → SchemaInterface
│                       ├── correlationId() → CorrelationIdInterface
│                       ├── bindings() → BindingsInterface
│                       ├── examples() → MessageExamplesInterface
│                       └── traits() → MessageTraitsInterface
│
├── operations()    → OperationsInterface (derived: all channel ops)
├── messages()      → MessagesInterface   (derived: unique messages from operations)
├── schemas()       → SchemasInterface    (derived: active schemas)
├── securitySchemes() → SecuritySchemesInterface
├── components()    → ComponentsInterface
├── allServers()    → includes component servers
├── allChannels()   → includes component channels
├── allOperations() → includes component operations
├── allMessages()   → includes component-only messages
└── allSchemas()    → includes all schemas incl. unused components
```

---

## Complete Navigation Example

This script walks the full v2 model tree and prints everything:

```js
// scratch/navigate-v2.js
const { Parser } = require('../packages/parser/cjs/index.js');

async function main() {
  const parser = new Parser();

  const doc = `
asyncapi: '2.6.0'
id: 'urn:streetlights'
info:
  title: Streetlights API
  version: '1.0.0'
  description: The Smartylighting Streetlights API.
  license:
    name: Apache 2.0
    url: https://www.apache.org/licenses/LICENSE-2.0
  contact:
    name: Support
    email: support@example.com
defaultContentType: application/json
servers:
  production:
    url: mqtt.example.com:{port}
    protocol: mqtt
    description: Production MQTT broker
    variables:
      port:
        enum: [1883, 8883]
        default: '1883'
    security:
      - apiKey: []
channels:
  smartylighting/streetlights/1/0/event/{streetlightId}/lighting/measured:
    description: Lighting measurement channel.
    parameters:
      streetlightId:
        description: ID of the streetlight.
        schema:
          type: string
    bindings:
      mqtt:
        qos: 1
    publish:
      operationId: receiveLightMeasurement
      summary: Receive a light measurement.
      tags:
        - name: measurements
      message:
        $ref: '#/components/messages/LightMeasured'
  smartylighting/streetlights/1/0/action/{streetlightId}/turn/on:
    parameters:
      streetlightId:
        schema:
          type: string
    subscribe:
      operationId: turnOn
      message:
        $ref: '#/components/messages/TurnOnOff'
components:
  messages:
    LightMeasured:
      name: LightMeasured
      title: Light measured
      summary: Environmental lighting measurement.
      contentType: application/json
      headers:
        type: object
        properties:
          correlationId:
            type: string
      payload:
        $ref: '#/components/schemas/LightMeasuredPayload'
    TurnOnOff:
      name: TurnOnOff
      payload:
        type: object
        properties:
          command:
            type: string
            enum: [on, off]
  schemas:
    LightMeasuredPayload:
      type: object
      required: [lumens, sentAt]
      properties:
        lumens:
          type: integer
          minimum: 0
        sentAt:
          type: string
          format: date-time
  securitySchemes:
    apiKey:
      type: apiKey
      in: user
`;

  const { document } = await parser.parse(doc);
  if (!document) return;

  // --- Root Fields ---
  console.log('=== Root ===');
  console.log('Version:', document.version());
  console.log('Default content type:', document.defaultContentType());

  // --- Info ---
  const info = document.info();
  console.log('\n=== Info ===');
  console.log('Title:', info.title());
  console.log('API version:', info.version());
  console.log('Description:', info.description());
  console.log('License:', info.license()?.name(), '-', info.license()?.url());
  console.log('Contact:', info.contact()?.name(), '/', info.contact()?.email());

  // --- Servers ---
  console.log('\n=== Servers ===');
  document.servers().all().forEach(server => {
    console.log(`[${server.id()}]`);
    console.log('  URL:', server.url());
    console.log('  Protocol:', server.protocol());
    console.log('  Description:', server.description());
    
    if (!server.variables().isEmpty()) {
      console.log('  Variables:');
      server.variables().all().forEach(v => {
        console.log(`    {${v.id()}}: default=${v.defaultValue()}, enum=${v.enum()}`);
      });
    }
  });

  // --- Channels and Operations ---
  console.log('\n=== Channels ===');
  document.channels().all().forEach(channel => {
    console.log(`\n[${channel.id()}]`);
    if (channel.description()) console.log('  Desc:', channel.description());
    
    // Channel parameters
    if (!channel.parameters().isEmpty()) {
      console.log('  Parameters:');
      channel.parameters().all().forEach(p => {
        console.log(`    {${p.id()}} type: ${p.schema()?.type()}`);
      });
    }
    
    // Channel bindings
    const bindings = channel.bindings();
    if (!bindings.isEmpty()) {
      console.log('  Bindings:', bindings.all().map(b => b.protocol()).join(', '));
    }

    // Operations
    channel.operations().all().forEach(op => {
      console.log(`\n  Operation: ${op.id()}`);
      console.log('  Action:', op.action()); // 'send' or 'receive'
      if (op.summary()) console.log('  Summary:', op.summary());
      
      // Tags
      if (!op.tags().isEmpty()) {
        console.log('  Tags:', op.tags().all().map(t => t.name()).join(', '));
      }

      // Messages
      op.messages().all().forEach(msg => {
        console.log(`\n    Message: ${msg.name() ?? '(anonymous)'}`);
        console.log('    Title:', msg.title());
        console.log('    Content type:', msg.contentType());
        
        // Headers schema
        const headers = msg.headers();
        if (headers) {
          console.log('    Headers type:', headers.type());
          const headerProps = headers.properties();
          if (headerProps) {
            Object.keys(headerProps).forEach(k => console.log(`      .${k}`));
          }
        }
        
        // Payload schema
        const payload = msg.payload();
        if (payload) {
          console.log('    Payload type:', payload.type());
          const required = payload.required();
          if (required) console.log('    Required:', required);
          const props = payload.properties();
          if (props) {
            Object.entries(props).forEach(([k, s]) => {
              console.log(`    .${k}: ${s.type()}${s.format() ? ` (${s.format()})` : ''}`);
            });
          }
        }
      });
    });
  });

  // --- Components ---
  const components = document.components();
  console.log('\n=== Components ===');
  console.log('Messages:', components.messages().all().map(m => m.id()).join(', '));
  console.log('Schemas:', components.schemas().all().map(s => s.id()).join(', '));
  console.log('Security schemes:', components.securitySchemes().all().map(s => s.id()).join(', '));

  // --- allMessages: includes component-only messages ---
  console.log('\n=== All Messages (including component-only) ===');
  document.allMessages().all().forEach(m => {
    console.log(' ', m.name() ?? m.id());
  });

  // --- allSchemas: includes all schemas ---
  console.log('\n=== All Schemas ===');
  document.allSchemas().all().forEach(s => {
    console.log(' ', s.id() ?? '(anonymous)', '-', s.type());
  });
}

main().catch(console.error);
```

---

## Working with Schemas (The `SchemaInterface`)

The payload and headers of a message are `SchemaInterface` instances. Key methods:

```js
const payload = message.payload();

payload.type()           // string | string[] | undefined
payload.properties()     // Record<string, SchemaInterface> | undefined
payload.items()          // SchemaInterface | undefined (for arrays)
payload.required()       // string[] | undefined
payload.enum()           // any[] | undefined
payload.allOf()          // SchemaInterface[] | undefined
payload.anyOf()          // SchemaInterface[] | undefined
payload.oneOf()          // SchemaInterface[] | undefined
payload.not()            // SchemaInterface | undefined
payload.additionalProperties() // SchemaInterface | boolean | undefined
payload.minimum()        // number | undefined
payload.maximum()        // number | undefined
payload.format()         // string | undefined
payload.default()        // any
payload.description()    // string | undefined
payload.title()          // string | undefined
payload.isCircular()     // boolean
payload.id()             // $id value
payload.extensions()     // x-* extension fields
payload.json()           // raw JSON Schema object
```

---

## The `allMessages()` vs `messages()` Distinction

```js
// messages() — only messages reachable from channels/operations
document.messages().all();
// → LightMeasured, TurnOnOff (both are used in operations)

// allMessages() — includes component-only messages (not referenced from operations)
document.allMessages().all();
// → LightMeasured, TurnOnOff + any messages defined in components but not $ref'd

// Use allMessages() when building documentation or generators that need
// complete coverage. Use messages() when only processing "active" messages.
```

---

## Next Step

Read [03-navigating-v3-documents.md](./03-navigating-v3-documents.md) for the v3 model tree where operations are top-level and channels reference-based.
