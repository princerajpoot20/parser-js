# Model Layer

> **Goal:** Understand how parsed JSON is wrapped in typed model classes, how the class hierarchy works, and how v2 and v3 models differ.

---

## The Model Hierarchy

```
BaseModel<J, M>                         ← root class for all model objects
├── AsyncAPIDocumentV2 (models/v2/)     ← created for asyncapi 2.x docs
├── AsyncAPIDocumentV3 (models/v3/)     ← created for asyncapi 3.x docs
├── InfoV2 / InfoV3
├── ChannelV2 / ChannelV3
├── OperationV2 / OperationV3
├── MessageV2 / MessageV3
├── SchemaV2
├── ServerV2 / ServerV3
├── ComponentsV2 / ComponentsV3
└── ... (80+ model classes)

Collection<T>                           ← root class for model collections
├── Channels
├── Operations
├── Messages
├── Schemas
├── Servers
└── ... (15+ collection classes)
```

---

## `BaseModel` — The Foundation

**File:** `packages/parser/src/models/base.ts`

Every model class extends `BaseModel`:

```typescript
export abstract class BaseModel<J extends any = any, M extends Record<string, any> = {}> {
  constructor(
    protected readonly _json: J,
    protected readonly _meta: ModelMetadata & M = {} as ModelMetadata & M,
  ) {}
```

`J` is the TypeScript type of the underlying raw JSON object (e.g., `v2.ChannelObject`).
`M` is additional metadata specific to this model type (e.g., `{ id: string; address: string }`).

### Three Core Methods

**`json(key?)`** — access the raw underlying JSON:

```typescript
const channel = doc.channels().get('user/registered')!;

// Get the entire raw JSON object
const rawChannel = channel.json();
// → { publish: { message: { payload: { type: 'object' } } }, ... }

// Get a specific field
const rawPublish = channel.json('publish');
// → { message: { ... } }
```

This is useful when you need to access fields not exposed by the typed API, or when debugging.

**`meta(key?)`** — access model metadata:

```typescript
channel.meta();
// → { asyncapi: DetailedAsyncAPI, pointer: '/channels/user~1registered', id: 'user/registered' }

channel.meta('pointer');
// → '/channels/user~1registered'
```

**`jsonPath(field?)`** — get the JSON Pointer string for this model's position:

```typescript
channel.jsonPath();
// → '/channels/user~1registered'

channel.jsonPath('publish');
// → '/channels/user~1registered/publish'
```

JSON Pointers use `~1` to encode `/` in path segments (e.g., channel key `user/registered` → `user~1registered`).

### `createModel()` — the Protected Factory

All model classes use `createModel()` to instantiate child models. This propagates the `asyncapi` context (the `DetailedAsyncAPI` reference) down the tree:

```typescript
// Inside AsyncAPIDocumentV2:
info(): InfoInterface {
  return this.createModel(Info, this._json.info, { pointer: '/info' });
}

// createModel implementation in BaseModel:
protected createModel<T extends BaseModel>(
  Model: Constructor<T>,
  value: InferModelData<T>,
  meta: Omit<ModelMetadata, 'asyncapi'> & InferModelMetadata<T>
): T {
  return new Model(value, { ...meta, asyncapi: this._meta.asyncapi });
}
```

The `asyncapi` reference is never passed explicitly — it is automatically propagated. This means every model at any depth in the tree can access the full `DetailedAsyncAPI` context.

---

## `Collection<T>` — Model Arrays

**File:** `packages/parser/src/models/collection.ts`

Collections are typed array wrappers:

```typescript
export abstract class Collection<T extends BaseModel> extends Array<T> {
  constructor(protected readonly collections: T[], protected readonly _meta) {
    super(...collections);  // Makes it a real Array
  }

  abstract get(id: string): T | undefined;  // Must be implemented per collection
  has(id: string): boolean;
  all(): T[];
  isEmpty(): boolean;
  filterBy(filter: (item: T) => boolean): T[];
  meta(): CollectionMetadata;
}
```

Usage example:

```typescript
const channels = doc.channels();

// These are equivalent:
channels.all().forEach(ch => { ... });
channels.forEach(ch => { ... }); // works because Collection extends Array

// Look up by ID:
const channel = channels.get('user/registered');
if (!channel) throw new Error('Channel not found');

// Filter:
const publishChannels = channels.filterBy(ch =>
  ch.operations().some(op => op.action() === 'send')
);
```

---

## The Interface Layer

Between `BaseModel` and the concrete classes, there is a layer of **interfaces** that define the API contract. These live in `src/models/` (not in `v2/` or `v3/`).

Example: `AsyncAPIDocumentInterface` (in `src/models/asyncapi.ts`):

```typescript
export interface AsyncAPIDocumentInterface extends BaseModel<...>, ExtensionsMixinInterface {
  version(): string;
  defaultContentType(): string | undefined;
  hasDefaultContentType(): boolean;
  info(): InfoInterface;
  servers(): ServersInterface;
  channels(): ChannelsInterface;
  operations(): OperationsInterface;
  messages(): MessagesInterface;
  schemas(): SchemasInterface;
  securitySchemes(): SecuritySchemesInterface;
  components(): ComponentsInterface;
  allServers(): ServersInterface;    // includes servers defined in components
  allChannels(): ChannelsInterface;  // includes channels defined in components
  allOperations(): OperationsInterface;
  allMessages(): MessagesInterface;  // all messages including component-only ones
  allSchemas(): SchemasInterface;
}
```

The `all*()` variants include objects that are defined only in `components` (not referenced from channels). The non-`all` variants only include "active" objects (reachable from channels/operations).

---

## Mixin Interfaces

**File:** `packages/parser/src/models/mixins.ts`

Common capabilities are defined as mixins that multiple model interfaces extend:

| Mixin | Methods | Used by |
|-------|---------|---------|
| `ExtensionsMixinInterface` | `extensions()` | AsyncAPIDocument, Channel, Operation, Message, Server, ... |
| `DescriptionMixinInterface` | `description()`, `hasDescription()` | Info, Channel, Message, Server, ... |
| `TagsMixinInterface` | `tags()` | Info, Channel, Message, Operation, ... |
| `ExternalDocumentationMixinInterface` | `externalDocs()`, `hasExternalDocs()` | Info, Channel, Message, ... |
| `BindingsMixinInterface` | `bindings()` | Channel, Message, Operation, Server |
| `SummaryMixinInterface` | `summary()`, `hasSummary()` | Channel, Message, Operation |
| `TitleMixinInterface` | `title()`, `hasTitle()` | Info, Message, Schema |
| `CoreMixinInterface` | All of the above | Complex objects |

---

## `AsyncAPIDocumentV2` Concrete Implementation

**File:** `packages/parser/src/models/v2/asyncapi.ts`

A representative slice showing how accessors work:

```typescript
export class AsyncAPIDocument extends BaseModel<v2.AsyncAPIObject> 
  implements AsyncAPIDocumentInterface {

  channels(): ChannelsInterface {
    return new Channels(
      Object.entries(this._json.channels || {}).map(([channelAddress, channel]) => 
        this.createModel(Channel, channel, {
          id: channelAddress,
          address: channelAddress,
          pointer: `/channels/${tilde(channelAddress)}`
        })
      )
    );
  }

  operations(): OperationsInterface {
    const operations: OperationInterface[] = [];
    // In v2, operations live inside channels, not at the top level
    this.channels().forEach(channel => operations.push(...channel.operations()));
    return new Operations(operations);
  }

  messages(): MessagesInterface {
    const messages: MessageInterface[] = [];
    // De-duplicate: same JSON object reference = same message
    this.operations().forEach(operation =>
      operation.messages().forEach(message =>
        !messages.some(m => m.json() === message.json()) && messages.push(message)
      )
    );
    return new Messages(messages);
  }
}
```

Key observations:
- **Lazy construction**: `channels()` creates new model instances every call. The raw JSON (`_json`) is the single source of truth.
- **Identity by reference**: `m.json() === message.json()` checks object identity (same resolved object), not deep equality.
- **`tilde()`**: URL-encodes path segments (`user/registered` → `user~1registered` for JSON Pointer).

---

## v2 vs v3 Model Differences

The most important structural difference:

### v2: Operations inside Channels

```
AsyncAPIDocumentV2
└── channels()
    └── ChannelV2 (id = channel address)
        └── operations()
            └── OperationV2
                └── action: 'send' | 'receive'  (mapped from publish/subscribe)
                └── messages()
                    └── MessageV2
```

In v2, `doc.operations()` iterates all channels and collects their operations.

### v3: Operations at Top Level

```
AsyncAPIDocumentV3
├── channels()
│   └── ChannelV3 (id = channel key, NOT the address)
│       └── address()  ← the channel address is a field
│       └── messages() ← messages are listed on the channel
└── operations()
    └── OperationV3
        └── action: 'send' | 'receive'
        └── channel()  ← reference to a ChannelV3
        └── messages() ← specific messages from that channel
        └── reply()    ← optional OperationReplyV3 (v3-only)
```

In v3, `doc.operations()` directly returns the top-level operations array.

### `OperationReply` and `OperationReplyAddress` (v3 only)

```typescript
const op = doc.operations().get('requestLightMeasurement')!;
const reply = op.reply();
if (reply) {
  const replyChannel = reply.channel();
  const replyAddress = reply.address();
}
```

These classes only exist in `models/v3/`. They model the first-class reply mechanism added in v3.

---

## The Schema Model

**File:** `packages/parser/src/models/v2/schema.ts`

The `Schema` model wraps a JSON Schema object with typed methods for every JSON Schema keyword. It is one of the most complex models.

```typescript
schema.type()            // string | string[] | undefined
schema.properties()      // Record<string, SchemaInterface> | undefined
schema.items()           // SchemaInterface | SchemaInterface[] | undefined
schema.allOf()           // SchemaInterface[] | undefined
schema.anyOf()           // SchemaInterface[] | undefined
schema.oneOf()           // SchemaInterface[] | undefined
schema.not()             // SchemaInterface | undefined
schema.additionalProperties()  // SchemaInterface | boolean | undefined
schema.required()        // string[] | undefined
schema.minimum()         // number | undefined
schema.maximum()         // number | undefined
schema.enum()            // any[] | undefined
schema.isCircular()      // boolean
schema.extensions()      // ExtensionsInterface
```

The `isCircular()` method checks the `x-parser-circular-props` extension set by `resolveCircularRefs`.

---

## `extensions()` — Accessing `x-` Fields

Any model that implements `ExtensionsMixinInterface` exposes `extensions()`:

```typescript
const doc = await parser.parse(asyncapiDoc);
const extensions = doc.document?.extensions();

// Iterate all x-* extensions
extensions?.all().forEach(ext => {
  console.log(ext.id(), '=', ext.value());
});

// Get a specific extension
const circular = doc.document?.extensions().get('x-parser-circular');
```

---

## `spec-types/` — Raw TypeScript Types

**Files:** `packages/parser/src/spec-types/v2.ts`, `spec-types/v3.ts`

These TypeScript interfaces mirror the raw JSON structure from `@asyncapi/specs`. They are used as the generic type parameter `J` in `BaseModel<J>`.

```typescript
// spec-types/v2.ts (simplified)
export interface AsyncAPIObject {
  asyncapi: string;
  info: InfoObject;
  servers?: ServersObject;
  channels: ChannelsObject;
  components?: ComponentsObject;
  'x-parser-spec-parsed'?: boolean;
  'x-parser-api-version'?: number;
  // ... all other fields
}
```

These types are **not** model classes — they are TypeScript compile-time types only.

---

## Next Step

Read [05-custom-operations.md](./05-custom-operations.md) to understand the post-parse transformations that run after the model is created.
