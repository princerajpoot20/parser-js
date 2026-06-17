# Custom Operations

> **Goal:** Understand the five post-parse transformations that run after the model is created: traits, unique IDs, schema parsing, circular ref resolution, and anonymous naming.

---

## Overview

After the typed model is created, a pipeline of "custom operations" enriches it. These run in a specific order because later operations depend on earlier ones:

```
1. checkCircularRefs       — detect circular $refs before doing anything else
2. applyTraits             — merge trait objects into operations/messages
3. parseSchemas            — invoke custom schema parsers on payloads
4. resolveCircularRefs     — annotate circular schemas in the model
5. anonymousNaming         — assign names to anonymous messages/schemas
```

The order matters: schema parsing must happen after trait merging (traits can add/change `schemaFormat`), and both circular ref steps must happen after schemas are fully resolved.

**File:** `packages/parser/src/custom-operations/index.ts`

---

## 1. `checkCircularRefs`

**File:** `packages/parser/src/custom-operations/check-circular-refs.ts`

### What it does

Walks the entire document JSON looking for objects that appear more than once in the traversal path — indicating a circular reference. If found, sets `x-parser-circular: true` on the root document.

```typescript
checkCircularRefs(document);
// After: document.json()['x-parser-circular'] === true (if circular refs exist)
```

### Why early detection?

Subsequent operations (especially `parseSchemas` and `anonymousNaming`) traverse the document. Without early circular detection, they would infinitely recurse. By detecting early and marking the document, other operations can check this flag and use guards.

### How to check in your code

```typescript
const { document } = await parser.parse(asyncapi);
if (document?.extensions().has('x-parser-circular')) {
  console.log('This document contains circular references');
}
```

---

## 2. `applyTraits`

**File:** `packages/parser/src/custom-operations/apply-traits.ts`

### What traits are

AsyncAPI supports "traits" — reusable fragments that get merged into operations or messages. They avoid copy-pasting:

```yaml
# Without traits
channels:
  user/registered:
    publish:
      bindings:
        kafka:
          clientId: { type: string, enum: [my-app] }
      message:
        headers:
          type: object
          properties:
            correlationId: { type: string }
  order/placed:
    publish:
      bindings:
        kafka:
          clientId: { type: string, enum: [my-app] }
      message:
        headers:
          type: object
          properties:
            correlationId: { type: string }

# With traits (DRY)
components:
  operationTraits:
    kafka:
      bindings:
        kafka:
          clientId: { type: string, enum: [my-app] }
  messageTraits:
    commonHeaders:
      headers:
        type: object
        properties:
          correlationId: { type: string }

channels:
  user/registered:
    publish:
      traits: [{ $ref: '#/components/operationTraits/kafka' }]
      message:
        traits: [{ $ref: '#/components/messageTraits/commonHeaders' }]
```

### How merging works

`applyTraitsToObjectV2()` iterates each trait in the `traits` array and calls `mergePatch()` on each field:

```typescript
function applyTraitsToObjectV2(value: Record<string, unknown>) {
  if (Array.isArray(value.traits)) {
    for (const trait of value.traits) {
      for (const key in trait) {
        value[String(key)] = mergePatch(value[String(key)], trait[String(key)]);
      }
    }
  }
}
```

`mergePatch` is a deep merge: object properties are merged recursively; primitive values from the trait override the object's value.

**The original `traits` array is preserved** in `x-parser-original-traits`:

```typescript
// In the merged result:
// operation.traits is still the original array
// operation.bindings now contains the merged result from the trait
```

### JSONPath traversal

The function uses `jsonpath-plus` to walk multiple paths in the document where traits can appear:

**v2 paths:**
- `$.channels.*.[publish,subscribe]` — channel operations
- `$.channels.*.[publish,subscribe].message` — channel operation messages
- `$.components.messages.*` — component messages

**v3 paths:**
- `$.operations.*` — top-level operations
- `$.channels.*.messages.*` — channel messages
- `$.components.channels.*.messages.*` — component channel messages

### Checking after parsing

After `parse()`, you can see merged traits:

```typescript
const op = document.channels().get('user/registered')!.operations().all()[0];
// op.bindings() now includes the kafka binding from the trait
// op.json()['x-parser-original-traits'] has the original traits array
```

---

## 3. `parseSchemas`

**File:** `packages/parser/src/custom-operations/parse-schema.ts`

### What it does

Walks the document looking for message payloads (and headers) that have a `schemaFormat` field. For each such payload, it calls the appropriate registered `SchemaParser` plugin.

The default `AsyncAPISchemaParser` handles JSON Schema formats — it does validation but no transformation (returns the schema as-is).

Non-default parsers (Avro, Protobuf, etc.) can:
1. **Validate** the payload against the schema format's rules
2. **Convert** the payload to an AsyncAPI Schema Object

After conversion, the original payload is saved in `x-parser-original-payload` and the converted version replaces it.

### v2 paths searched for payloads

```
$.channels.*.[publish,subscribe].message.payload
$.channels.*.[publish,subscribe].message.oneOf.*.payload
$.components.messages.*.payload
```

### v3 paths searched for payloads

```
$.channels.*.messages.*.payload
$.operations.*.messages.*.payload
$.components.messages.*.payload
```

Plus headers at equivalent paths.

### What happens for JSON Schema (no `schemaFormat`)

If `schemaFormat` is not set, the AsyncAPI spec implies JSON Schema Draft-07. `parseSchemasV2` and `parseSchemasV3` find these payloads and call `validateSchema()` using the `AsyncAPISchemaParser`, which validates with Ajv. Validation failures become diagnostics.

### Disabled with `parseSchemas: false`

```typescript
const { document } = await parser.parse(asyncapi, {
  parseSchemas: false  // skip schema parsing, useful for performance
});
```

Use this when you only need the structure of the document and don't need schema validation.

---

## 4. `resolveCircularRefs`

**File:** `packages/parser/src/custom-operations/resolve-circular-refs.ts`

### What it does

Uses the Spectral `documentInventory` (passed from `parse.ts`) to identify which schemas are part of circular reference chains. For each circular schema, sets `x-parser-circular-props` listing the properties that create the cycle.

This happens **after** custom schema parsing because converting schemas (e.g., Avro to JSON Schema) might create new circular structures.

### Why use Spectral's inventory?

The `documentInventory` tracks every `$ref` that Spectral resolved during the `runWithResolved()` call. It knows the full resolution path. By examining this, the parser can determine which schemas are circular (their resolution path contains a reference back to a path already in the chain).

### Reading circular info from the model

```typescript
const schema = doc.schemas().get('SomeSchema')!;
if (schema.isCircular()) {
  // x-parser-circular-props is set
  console.log('Circular schema:', schema.json()['x-parser-circular-props']);
}
```

---

## 5. `anonymousNaming`

**File:** `packages/parser/src/custom-operations/anonymous-naming.ts`

### What it does

Assigns auto-generated names to:
- **Messages** that don't have a `name` field
- **Schemas** that don't have an `$id`

Anonymous objects cannot be looked up by name in collections. Assigning names makes the document more navigable and enables tools to reference these objects consistently.

### Naming strategy for messages

For an anonymous message in a channel operation, the name is derived from:
1. The channel address (e.g., `user/registered`)
2. The operation direction (publish/subscribe)
3. A counter if there are multiple anonymous messages in the same operation

The result is stored in `x-parser-message-name`:

```typescript
// Before naming: message.name() === undefined
// After naming: message.name() === 'anonymous-message-1'
//              message.json()['x-parser-message-name'] === 'anonymous-message-1'
```

### Naming strategy for schemas

Anonymous schemas (those without `$id`) get an `x-parser-schema-id` assigned. This is typically a generated hash-like identifier based on their position in the document.

---

## Controlling Custom Operations

You can disable the trait and schema parsing steps via `ParseOptions`:

```typescript
const { document } = await parser.parse(asyncapiDoc, {
  applyTraits: false,   // skip applyTraits (traits array will remain in json but not merged)
  parseSchemas: false,  // skip parseSchemas (schema validation not run, avro not converted)
});
```

There are no options to disable `checkCircularRefs`, `resolveCircularRefs`, or `anonymousNaming`. These always run.

---

## Summary Table

| Operation | When to skip | Side effects |
|-----------|-------------|-------------|
| `checkCircularRefs` | Never | Adds `x-parser-circular` to root |
| `applyTraits` | `applyTraits: false` | Adds merged fields directly to ops/messages; saves original in `x-parser-original-traits` |
| `parseSchemas` | `parseSchemas: false` | May add `x-parser-original-payload`, `x-parser-original-schema-format` to messages |
| `resolveCircularRefs` | Never | Adds `x-parser-circular-props` to circular schemas |
| `anonymousNaming` | Never | Adds `x-parser-message-name`, `x-parser-schema-id` to unnamed objects |

---

## Next Step

Read [06-schema-parser-system.md](./06-schema-parser-system.md) to understand the pluggable schema parser interface and how non-JSON-Schema payloads are handled.
