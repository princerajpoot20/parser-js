# Use Case: Stringify and Unstringify

> **Goal:** Safely serialize a parsed `AsyncAPIDocumentInterface` to a JSON string and later reconstruct it — especially when the document contains circular references.

---

## The Problem

After `parser.parse()`, the `document` object has resolved all `$ref`s. The resolved JSON may contain **circular references** (e.g., a schema that references itself). Standard `JSON.stringify` throws:

```js
const { document } = await parser.parse(circularDoc);
JSON.stringify(document.json()); 
// TypeError: Converting circular structure to JSON
```

The parser provides `stringify()` and `unstringify()` that handle this correctly.

---

## `stringify(document)`

**File:** `packages/parser/src/stringify.ts`

Converts a parsed `AsyncAPIDocumentInterface` to a JSON string. Circular references are replaced with special `$ref:` JSON pointer strings, making the output a standard, serializable JSON string.

```typescript
function stringify(document: AsyncAPIDocumentInterface, options?: StringifyOptions): string | undefined
```

The output JSON has an additional marker: `"x-parser-spec-stringified": true`.

---

## `unstringify(jsonString)`

Takes the stringified JSON (either a string or a parsed object) and reconstructs the circular references, returning a full `AsyncAPIDocumentInterface`.

```typescript
function unstringify(document: string | Record<string, unknown>): AsyncAPIDocumentInterface | undefined
```

---

## Example: Round-Trip Serialization

Create `scratch/use-case-stringify.js`:

```js
// scratch/use-case-stringify.js
const { Parser, stringify, unstringify } = require('../packages/parser/cjs/index.js');

async function main() {
  const parser = new Parser();

  // A document with circular refs
  const circularDoc = `
asyncapi: '2.6.0'
info:
  title: Circular Stringify Example
  version: '1.0.0'
channels:
  tree/update:
    publish:
      message:
        payload:
          $ref: '#/components/schemas/TreeNode'
components:
  schemas:
    TreeNode:
      type: object
      properties:
        value:
          type: string
        children:
          type: array
          items:
            $ref: '#/components/schemas/TreeNode'
`;

  const { document } = await parser.parse(circularDoc);
  if (!document) {
    console.log('Parse failed');
    return;
  }

  // === STRINGIFY ===
  console.log('=== Stringifying ===');
  
  // This would throw: JSON.stringify(document.json())
  
  // Use stringify() instead:
  const serialized = stringify(document);
  if (!serialized) {
    console.log('stringify() returned undefined');
    return;
  }

  console.log('Serialized length:', serialized.length, 'chars');
  
  // The output is valid JSON
  const parsed = JSON.parse(serialized);
  console.log('Is valid JSON: true');
  console.log('Has x-parser-spec-stringified marker:', parsed['x-parser-spec-stringified']); // true
  console.log('Has x-parser-spec-parsed marker:', parsed['x-parser-spec-parsed']); // true

  // Circular refs are replaced with $ref: pointers
  const schemaStr = JSON.stringify(parsed['components']['schemas']['TreeNode'], null, 2);
  console.log('\nTreeNode schema in serialized form:');
  console.log(schemaStr.slice(0, 500));

  // === UNSTRINGIFY ===
  console.log('\n=== Unstringifying ===');
  
  const restored = unstringify(serialized);
  if (!restored) {
    console.log('unstringify() returned undefined');
    return;
  }

  console.log('Restored document title:', restored.info().title());
  console.log('Restored channel count:', restored.channels().all().length);

  // Circular refs are reconnected
  const treeSchema = restored.allSchemas().get('TreeNode');
  console.log('TreeNode schema type:', treeSchema?.type());
  console.log('TreeNode is circular:', treeSchema?.isCircular());
}

main().catch(console.error);
```

Run:

```bash
node scratch/use-case-stringify.js
```

---

## Example: Non-Circular Document

`stringify()` and `unstringify()` also work on documents without circular refs — you can use them as a universal safe serialization mechanism:

```js
const { Parser, stringify, unstringify } = require('../packages/parser/cjs/index.js');

const parser = new Parser();

const simpleDoc = `
asyncapi: '2.6.0'
info:
  title: Simple API
  version: '1.0.0'
channels:
  user/registered:
    publish:
      message:
        payload:
          type: object
`;

const { document } = await parser.parse(simpleDoc);

// Works on non-circular docs too
const serialized = stringify(document);
const restored = unstringify(serialized);
console.log(restored?.info().title()); // 'Simple API'
```

---

## Use Cases for Stringify/Unstringify

### 1. Caching parsed documents

Parsing is expensive (Spectral validation, $ref resolution, custom operations). Cache the serialized result:

```js
const fs = require('fs');
const path = require('path');
const { Parser, stringify, unstringify } = require('@asyncapi/parser');

const CACHE_DIR = '.asyncapi-cache';

async function getParsedDocument(filePath) {
  const cachePath = path.join(CACHE_DIR, path.basename(filePath) + '.json');
  
  // Try to load from cache
  if (fs.existsSync(cachePath)) {
    const cached = fs.readFileSync(cachePath, 'utf-8');
    return unstringify(cached);
  }
  
  // Parse and cache
  const parser = new Parser();
  const { document } = await fromFile(parser, filePath).parse();
  
  if (document) {
    fs.mkdirSync(CACHE_DIR, { recursive: true });
    fs.writeFileSync(cachePath, stringify(document) ?? '');
  }
  
  return document;
}
```

### 2. Sending parsed docs over HTTP or Worker threads

Node.js Worker threads cannot pass class instances directly. Stringify before sending, unstringify on the other side:

```js
// In main thread:
const { document } = await parser.parse(asyncapiDoc);
worker.postMessage({ serialized: stringify(document) });

// In worker thread:
const { unstringify } = require('@asyncapi/parser');
worker.on('message', ({ serialized }) => {
  const document = unstringify(serialized);
  // Now use document normally
});
```

### 3. Storing in databases

```js
// Store
const serialized = stringify(document);
await db.collection('parsedDocs').insertOne({ id: 'streetlights', doc: serialized });

// Retrieve
const row = await db.collection('parsedDocs').findOne({ id: 'streetlights' });
const document = unstringify(row.doc);
```

---

## How `refReplacer` Works

Under the hood, `stringify()` uses a custom JSON replacer function called `refReplacer()`. When `JSON.stringify` encounters an object it has seen before (circular), `refReplacer` emits a special string like:

```
"$ref:$.components.schemas.TreeNode"
```

When `unstringify()` reads this back, it finds strings starting with `$ref:` and replaces them with the actual object reference, restoring the circular structure.

This is why `stringify()` output is **not** a standard JSON Schema or AsyncAPI document — it has internal `$ref:` strings that only `unstringify()` knows how to interpret.

---

## `stringify()` Options

```typescript
const serialized = stringify(document, {
  space: 2,  // pretty-print with 2-space indentation (default is 2)
});

const compact = stringify(document, {
  space: 0,  // compact output (no whitespace)
});
```

---

## What `unstringify()` Returns

`unstringify()` returns:
- An `AsyncAPIDocumentInterface` if the input is valid stringified content
- `undefined` if the input is not a stringified AsyncAPI document (e.g., plain JSON without the `x-parser-spec-stringified` marker)

---

## Next Step

You have completed the Use Cases chapter. Move to [Chapter 5: Model API](../05-model-api/01-new-api-vs-old-api.md) for a deep dive into navigating the typed document model.
