# Navigating v3 Documents

> **Goal:** Understand the v3 model tree's unique structure (top-level operations, channel IDs vs addresses, reply patterns) with code examples.

---

## The Key v3 Structural Difference

In v2, **operations live inside channels**. In v3, **operations are top-level** and reference channels. This affects how you navigate the model:

```
v2 mental model:          v3 mental model:
  channel                   operation
    └─ publish/subscribe       ├─ action: send/receive
        └─ message             ├─ channel → channel
                               └─ message(s) from channel

channels["user/reg"]      operations["onUserRegistered"]
  publish                   action: receive
    message: UserReg          channel: → channels["userRegistered"]
                              messages: → [channels["userRegistered"].messages["UserReg"]]
```

---

## Channel ID vs Channel Address in v3

In v2, the **map key** of `channels:` IS the channel address (the MQTT topic, Kafka topic, etc.).

In v3, the **map key** is the **channel ID** (a short identifier), and the actual topic address is a separate `address` field:

```yaml
# v2: channel key = address
channels:
  user/registered:      ← this IS the address
    publish: ...

# v3: channel key = ID, address is a field
channels:
  userRegistered:       ← this is the ID (used for referencing)
    address: user/registered   ← this is the actual topic
    messages: ...
```

This separation allows:
- Multiple channels with the same address (for different protocols)
- Channels to be named meaningfully without `/` separators
- Operations to reference channels by their meaningful ID

---

## The v3 Model Tree

```
AsyncAPIDocumentV3
├── version()
├── defaultContentType()
├── info() → same as v2
│
├── servers() → ServersInterface
│
├── channels() → ChannelsInterface
│   └── [channel] → ChannelV3
│       ├── id()       ← channel ID (the map key, NOT the address)
│       ├── address()  ← the actual topic path (NEW in v3)
│       ├── description()
│       ├── parameters() → ChannelParametersInterface
│       ├── bindings() → BindingsInterface
│       └── messages() → MessagesInterface (messages listed on the channel)
│           └── [message] → MessageV3
│               ├── id()
│               ├── name()
│               ├── payload() → SchemaInterface
│               ├── headers() → SchemaInterface
│               └── ... (same as v2)
│
├── operations() → OperationsInterface  ← TOP-LEVEL in v3
│   └── [operation] → OperationV3
│       ├── id()         ← operation ID (the map key)
│       ├── action()     ← 'send' | 'receive' (no more 'publish'/'subscribe')
│       ├── channel()    ← reference to ChannelV3  (NEW — reversed relationship)
│       ├── messages()   ← specific messages from the channel this op uses
│       ├── reply()      ← OperationReplyV3 | undefined (NEW in v3)
│       │   ├── channel()       ← the reply channel
│       │   └── address()       ← OperationReplyAddressV3 | undefined
│       │       ├── location()  ← where the reply address is specified
│       │       └── description()
│       ├── tags()
│       ├── bindings()
│       └── traits() → OperationTraitsInterface
│
├── messages()    → derived from channels (unique)
├── schemas()     → active schemas
├── allMessages() → includes component-only
├── allSchemas()  → all schemas
└── components() → ComponentsInterface (same structure)
    ├── channels()   ← NEW: components can have reusable channels
    ├── operations() ← NEW: components can have reusable operations
    ├── messages()
    ├── schemas()
    └── ...
```

---

## Complete Navigation Example

```js
// scratch/navigate-v3.js
const { Parser } = require('../packages/parser/cjs/index.js');

async function main() {
  const parser = new Parser();

  const doc = `
asyncapi: '3.0.0'
info:
  title: Streetlights API v3
  version: '1.0.0'
  description: Smart streetlights with reply support.
defaultContentType: application/json
servers:
  production:
    host: mqtt.example.com
    protocol: mqtt
channels:
  lightingMeasured:
    address: 'smartylighting/streetlights/1/0/event/{streetlightId}/lighting/measured'
    description: The channel for light measurements.
    parameters:
      streetlightId:
        description: ID of the streetlight
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
  lightTurnOn:
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
  lightStatusReply:
    address: 'smartylighting/streetlights/reply/{correlationId}'
    messages:
      statusReply:
        name: StatusReply
        payload:
          type: object
          properties:
            status:
              type: string
              enum: [ok, error]
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
      $ref: '#/channels/lightTurnOn'
    messages:
      - $ref: '#/channels/lightTurnOn/messages/turnOnOff'
    reply:
      channel:
        $ref: '#/channels/lightStatusReply'
      address:
        location: '$message.header#/replyTo'
        description: Reply address from message header
`;

  const { document, diagnostics } = await parser.parse(doc);
  
  const errors = diagnostics.filter(d => d.severity === 0);
  if (errors.length > 0) {
    console.log('Errors:', errors.map(e => e.message));
    return;
  }
  
  if (!document) return;

  // --- v3 Channels ---
  console.log('=== V3 Channels ===');
  document.channels().all().forEach(channel => {
    console.log(`\nChannel ID: ${channel.id()}`);
    console.log('  Address:', channel.address());
    // ↑ Note: id() and address() are different in v3!
    
    if (channel.description()) {
      console.log('  Description:', channel.description());
    }
    
    console.log('  Messages on channel:');
    channel.messages().all().forEach(msg => {
      console.log(`    - ${msg.id()} (name: ${msg.name() ?? 'unnamed'})`);
      console.log(`      Payload type: ${msg.payload()?.type()}`);
    });
  });

  // --- v3 Operations (TOP-LEVEL) ---
  console.log('\n=== V3 Top-Level Operations ===');
  document.operations().all().forEach(op => {
    console.log(`\nOperation: ${op.id()}`);
    console.log('  Action:', op.action()); // 'send' or 'receive'
    
    // Operations reference channels (reversed from v2)
    const ch = op.channel();
    console.log('  Channel ID:', ch.id());
    console.log('  Channel address:', ch.address());
    
    // Specific messages this operation uses
    console.log('  Messages:');
    op.messages().all().forEach(msg => {
      console.log(`    - ${msg.name() ?? msg.id()}`);
    });
    
    // Reply (v3-only!)
    const reply = op.reply();
    if (reply) {
      console.log('  REPLY:');
      const replyChannel = reply.channel();
      console.log('    Reply channel:', replyChannel?.id(), '→', replyChannel?.address());
      
      const replyAddress = reply.address();
      if (replyAddress) {
        console.log('    Reply address location:', replyAddress.location());
        console.log('    Reply address description:', replyAddress.description());
      }
    }
  });

  // --- Looking up a channel by ID ---
  console.log('\n=== Lookup by ID ===');
  const measuredChannel = document.channels().get('lightingMeasured');
  console.log('Found lightingMeasured:', measuredChannel?.address());

  // --- Looking up an operation ---
  const turnOnOp = document.operations().get('sendTurnOn');
  console.log('Found sendTurnOn:', turnOnOp?.action());
}

main().catch(console.error);
```

---

## Iterating Operations and Their Channels

The most common traversal pattern for v3 is operation-first:

```js
// Print every operation with its channel address
document.operations().all().forEach(op => {
  const direction = op.action() === 'send' ? '→' : '←';
  console.log(`${op.id()} ${direction} ${op.channel().address()}`);
});
// receiveLightMeasurement ← smartylighting/streetlights/1/0/event/.../measured
// sendTurnOn → smartylighting/streetlights/1/0/action/.../turn/on
```

---

## Channel-First Iteration in v3

To iterate channels and then find which operations use them:

```js
document.channels().all().forEach(channel => {
  // Find all operations that reference this channel
  const channelOps = document.operations().filterBy(op => 
    op.channel().id() === channel.id()
  );
  
  console.log(`Channel: ${channel.id()} (${channel.address()})`);
  channelOps.forEach(op => {
    console.log(`  Used by operation: ${op.id()} [${op.action()}]`);
  });
});
```

---

## v3 Components (New Keys)

v3 adds `operations` and `channels` to the `components` section (for reusable definitions):

```js
const components = document.components();

// These are NEW in v3:
components.channels().all();     // reusable channel definitions
components.operations().all();   // reusable operation definitions

// These exist in both v2 and v3:
components.messages().all();
components.schemas().all();
components.servers().all();
components.securitySchemes().all();
```

---

## `allOperations()` in v3

```js
// operations() — only top-level operations in the spec
document.operations().all();

// allOperations() — includes component-defined operations
document.allOperations().all();
```

---

## Next Step

Read [04-parser-api-versioning.md](./04-parser-api-versioning.md) to understand how `@asyncapi/multi-parser` handles multiple parser API versions.
