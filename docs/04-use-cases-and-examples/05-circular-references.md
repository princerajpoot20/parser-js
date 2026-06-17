# Use Case: Circular References

> **Goal:** Understand what circular references are in AsyncAPI documents, how the parser detects and handles them, and what extensions the parser adds to mark them.

---

## What is a Circular Reference?

In JSON Schema, schemas can reference each other. If schema A references schema B, and schema B references schema A (directly or through a chain), the reference is **circular**:

```yaml
components:
  schemas:
    Category:
      type: object
      properties:
        subcategories:
          type: array
          items:
            $ref: '#/components/schemas/Category'
          # ↑ Category contains a list of Category — circular!
```

Circular references are valid in JSON Schema and are used to model recursive data structures like trees, graphs, and nested categories. The parser must handle them specially because:

1. Naive `JSON.stringify()` throws `"Converting circular structure to JSON"`
2. Naive traversal algorithms infinite-loop

---

## Detection: `x-parser-circular`

After parsing, the root document will have `x-parser-circular: true` if any circular refs were found:

```js
// scratch/use-case-circular.js
const { Parser } = require('../packages/parser/cjs/index.js');

async function main() {
  const parser = new Parser();

  const doc = `
asyncapi: '2.6.0'
info:
  title: Circular Example
  version: '1.0.0'
channels:
  category/updated:
    publish:
      message:
        payload:
          $ref: '#/components/schemas/Category'
components:
  schemas:
    Category:
      type: object
      properties:
        name:
          type: string
        subcategories:
          type: array
          items:
            $ref: '#/components/schemas/Category'
`;

  const { document } = await parser.parse(doc);
  if (!document) return;

  // Check if any circular refs exist in the document
  const hasCircular = document.json()['x-parser-circular'];
  console.log('Has circular refs:', hasCircular);  // true

  // Or use the extensions API:
  const circularExtension = document.extensions().get('x-parser-circular');
  console.log('x-parser-circular:', circularExtension?.value());  // true
}

main().catch(console.error);
```

---

## `x-parser-circular-props`: Per-Schema Circular Properties

For each schema with circular properties, the parser sets `x-parser-circular-props` — an array of property names that are circular:

```js
// Continuing from above...
const schema = document.allSchemas().get('Category');
if (schema) {
  console.log('Schema:', schema.id());
  console.log('Is circular:', schema.isCircular());

  const circularProps = schema.json()['x-parser-circular-props'];
  console.log('Circular props:', circularProps);  // ['subcategories']
}
```

The `isCircular()` method on `SchemaInterface` checks for this extension.

---

## A More Complex Example: Deep Circular Chain

```yaml
components:
  schemas:
    Node:
      type: object
      properties:
        value:
          type: string
        left:
          $ref: '#/components/schemas/Node'   # Node → Node (direct self-reference)
        right:
          $ref: '#/components/schemas/Node'
```

After parsing:

```js
const nodeSchema = document.allSchemas().get('Node');
console.log(nodeSchema?.isCircular());  // true
// nodeSchema.json()['x-parser-circular-props'] → ['left', 'right']
```

---

## Using `checkCircularRefs` Option

You can check for circular refs without doing the full parse using the `checkCircularRefs` utility exposed in the custom operations:

```js
// Check after parsing
const { document } = await parser.parse(doc);

// Simple check on the model
function hasCircularRefs(doc) {
  return Boolean(doc?.json()['x-parser-circular']);
}

if (hasCircularRefs(document)) {
  console.warn('Document has circular references — JSON.stringify will fail');
}
```

---

## Serializing Documents with Circular Refs

Standard `JSON.stringify` throws on circular structures. Use the parser's `stringify()` function instead:

```js
// scratch/use-case-circular-stringify.js
const { Parser, stringify, unstringify } = require('../packages/parser/cjs/index.js');

async function main() {
  const parser = new Parser();

  const doc = `
asyncapi: '2.6.0'
info:
  title: Circular Example
  version: '1.0.0'
channels:
  tree/node:
    publish:
      message:
        payload:
          $ref: '#/components/schemas/TreeNode'
components:
  schemas:
    TreeNode:
      type: object
      properties:
        value: { type: string }
        children:
          type: array
          items:
            $ref: '#/components/schemas/TreeNode'
`;

  const { document } = await parser.parse(doc);
  if (!document) return;

  // This WOULD throw: JSON.stringify(document.json())
  // Use stringify() instead:
  const serialized = stringify(document);
  console.log('Serialized length:', serialized?.length);
  console.log('Has x-parser-spec-stringified:', serialized?.includes('x-parser-spec-stringified'));

  // Reconstruct the document from the serialized string
  const restored = unstringify(serialized!);
  console.log('Restored document title:', restored?.info().title());
}

main().catch(console.error);
```

See [07-stringify-and-unstringify.md](./07-stringify-and-unstringify.md) for full details on serialization.

---

## Walking Schemas with Circular Refs Safely

When traversing schemas, guard against circular refs:

```js
function walkSchema(schema, visited = new Set()) {
  // Guard: if we've seen this schema object before, stop
  if (visited.has(schema.json())) return;
  visited.add(schema.json());

  console.log('Schema type:', schema.type());

  // Recurse into properties
  const props = schema.properties();
  if (props) {
    for (const [name, propSchema] of Object.entries(props)) {
      if (!propSchema.isCircular()) {
        walkSchema(propSchema, visited);
      } else {
        console.log(`Property ${name} is circular — skipping`);
      }
    }
  }
}

// Usage
const schema = document.allSchemas().get('Category');
if (schema) walkSchema(schema);
```

---

## Circular Refs in the Test Suite

The test fixture `packages/parser/test/mocks/circular-refs.yaml` is used in multiple tests. Looking at it helps understand what patterns the parser handles:

```bash
cat packages/parser/test/mocks/circular-refs.yaml
```

And the corresponding tests:

```bash
npx jest "check-circular-refs" --rootdir packages/parser
npx jest "resolve-circular-refs" --rootdir packages/parser
```

---

## Next Step

Read [06-custom-schema-formats.md](./06-custom-schema-formats.md) to learn how to handle Avro or other schema formats beyond JSON Schema.
