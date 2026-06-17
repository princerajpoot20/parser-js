# New API vs Old API

> **Goal:** Understand the two generations of the parser's TypeScript API, why both exist, when to use each, and how to convert between them.

---

## Background

The AsyncAPI parser has gone through multiple major versions. With version 2.0.0 of `@asyncapi/parser`, the team introduced the **Parser-API v3** (also called the "Intent API") — a completely redesigned TypeScript interface for navigating parsed documents.

Before that, there was the "old API" (Parser-API v0), which is still available in `src/old-api/` for backward compatibility.

---

## Detection: `x-parser-api-version`

The parser marks each document with which API version it uses:

| Extension value | API | Document class |
|-----------------|-----|----------------|
| `3` | New API (Intent API, Parser-API v3) | `AsyncAPIDocumentV2` / `AsyncAPIDocumentV3` |
| `0` or absent | Old API (Parser-API v0) | `OldAsyncAPIDocument` |

```js
const { document } = await parser.parse(asyncapiDoc);

// New API detection
const apiVersion = document?.json()['x-parser-api-version'];
console.log('Parser API version:', apiVersion);  // 3

// Type guard functions
const { isAsyncAPIDocument, isOldAsyncAPIDocument } = require('../packages/parser/cjs/index.js');
console.log(isAsyncAPIDocument(document));     // true (new API)
console.log(isOldAsyncAPIDocument(document));  // false
```

---

## Side-by-Side Comparison

Given this document:

```yaml
asyncapi: '2.6.0'
info:
  title: Example
  version: '1.0.0'
channels:
  user/registered:
    publish:
      operationId: onUserRegistered
      message:
        name: UserRegistered
        payload:
          type: object
```

### Accessing channels

**New API (default after `parse()`):**
```typescript
const channels = document.channels();              // ChannelsInterface
const channel = channels.get('user/registered');   // ChannelInterface | undefined
console.log(channel?.id());                        // 'user/registered'
```

**Old API (after `convertToOldAPI()`):**
```typescript
const oldDoc = convertToOldAPI(document);
const channels = oldDoc.channels();                // Record<string, OldChannel>
const channel = channels['user/registered'];       // OldChannel | undefined (direct object access)
console.log(channel?.name());                      // 'user/registered'
```

### Accessing operations

**New API:**
```typescript
const channel = document.channels().get('user/registered')!;
const ops = channel.operations().all();            // OperationInterface[]
console.log(ops[0].action());                     // 'send' (mapped from 'publish')
console.log(ops[0].id());                         // 'onUserRegistered'
```

**Old API:**
```typescript
const channel = oldDoc.channel('user/registered')!;
const publish = channel.publish();                 // OldOperation | undefined
const subscribe = channel.subscribe();             // OldOperation | undefined
console.log(publish?.operationId());              // 'onUserRegistered'
```

### Accessing messages

**New API:**
```typescript
const op = document.channels().get('user/registered')!.operations().all()[0];
const messages = op.messages().all();              // MessageInterface[]
console.log(messages[0].name());                   // 'UserRegistered'
console.log(messages[0].payload()?.type());        // 'object'
```

**Old API:**
```typescript
const publish = oldDoc.channel('user/registered')!.publish()!;
const message = publish.message();                 // OldMessage (single, not array)
console.log(message.name());                       // 'UserRegistered'
console.log(message.payload()?.type());            // 'object'
```

---

## Converting Between APIs

**File:** `packages/parser/src/old-api/converter.ts`

### `convertToOldAPI(newDocument)` → `OldAsyncAPIDocument`

Use this when you have code that was written against the old API and cannot be upgraded yet:

```js
const { Parser, convertToOldAPI } = require('../packages/parser/cjs/index.js');

async function main() {
  const parser = new Parser();
  const { document } = await parser.parse(asyncapiDoc);
  
  if (!document) return;
  
  // Convert to old API
  const oldDoc = convertToOldAPI(document);
  
  // Now use old API methods
  console.log(oldDoc.version());           // '2.6.0' (same)
  console.log(oldDoc.title());             // 'Example'
  const channels = oldDoc.channels();      // plain object, not Collection
  console.log(Object.keys(channels));      // ['user/registered']
  
  // Old API uses map() with the channel name, not get()
  const channel = channels['user/registered'];
  const publish = channel.publish();
  console.log(publish?.operationId());     // 'onUserRegistered'
}
```

### What `convertToOldAPI()` Does Internally

1. Deep-copies the document JSON (to avoid mutating the new document)
2. Removes traits from operations/messages (old API expects pre-merged state, but stores original traits)
3. Sets `x-parser-original-schema-format` and `x-parser-message-parsed` flags
4. Creates an `OldAsyncAPIDocument` wrapping the same JSON
5. Sets `x-parser-api-version: 0`

### `convertToNewAPI(oldDocument)` → `AsyncAPIDocumentInterface`

Going the other direction:

```js
const { convertToNewAPI } = require('../packages/parser/cjs/index.js');

const newDoc = convertToNewAPI(oldDoc);
console.log(newDoc.channels().get('user/registered')?.id()); // 'user/registered'
```

---

## When to Use Each API

| Situation | Recommendation |
|-----------|---------------|
| New code, new project | Use new API (default) |
| Existing code using `@asyncapi/parser` v1 | Use `convertToOldAPI()` as bridge while migrating |
| AsyncAPI Generator templates (old templates) | Templates may use old API — check template's parser-api version |
| Writing new tools / integrations | Use new API |
| Reading issues/PRs about "old model API" | Refers to v0 API |

---

## Key Differences Summary

| Aspect | Old API (v0) | New API (v3) |
|--------|-------------|--------------|
| Channel access | `doc.channels()['user/registered']` | `doc.channels().get('user/registered')` |
| Operations | Per-channel `.publish()` / `.subscribe()` | `channel.operations().all()` or `doc.operations().all()` |
| Operation direction | `.publish()` / `.subscribe()` methods | `.action()` returning `'send'` or `'receive'` |
| Message access | `.message()` (single) | `.messages().all()` (array, for `oneOf`) |
| Collections | Plain objects/arrays | `Collection<T>` with `.get()`, `.all()`, `.filterBy()` |
| Type safety | Limited | Full TypeScript types via `BaseModel` |
| API version marker | `x-parser-api-version: 0` | `x-parser-api-version: 3` |
| AsyncAPI v3 support | No (v2 only) | Yes |

---

## Migration Guide (Old → New API)

See the existing migration docs:
- `packages/parser/docs/migrations/v1-to-v2.md`
- `packages/parser/docs/migrations/v2-to-v3.md`

---

## Next Step

Read [02-navigating-v2-documents.md](./02-navigating-v2-documents.md) for a complete walkthrough of the v2 model tree with code examples.
