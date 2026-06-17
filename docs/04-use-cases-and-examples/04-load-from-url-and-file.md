# Use Case: Load from URL and File

> **Goal:** Use `fromFile()` and `fromURL()` to load and parse AsyncAPI documents from the filesystem or HTTP, with proper source tracking and error handling.

---

## Why `fromFile()` and `fromURL()`?

You could manually read a file and pass the content to `parser.parse()`. The `fromFile()` and `fromURL()` helpers make this more convenient by:
1. Reading the file/URL for you
2. Automatically setting the `source` option (required for relative `$ref` resolution)
3. Supporting both `parse()` and `validate()` in one call

---

## `fromFile(parser, filePath)`

**File:** `packages/parser/src/from.ts`

```typescript
export function fromFile(parser: Parser, source: string, options?): FromResult {
  return {
    async parse(options?) {
      const schema = await readFile(source, options);
      return parser.parse(schema, { ...options, source });
      //                                          ↑ source is set automatically
    },
    async validate(options?) { ... }
  };
}
```

### Example

Create `scratch/streetlights-v2.yaml` (copy from [Chapter 1](../01-foundations/03-asyncapi-spec-walkthrough.md)) and then:

```js
// scratch/use-case-fromfile.js
const { Parser, fromFile } = require('../packages/parser/cjs/index.js');
const path = require('path');

async function main() {
  const parser = new Parser();

  const filePath = path.join(__dirname, 'streetlights-v2.yaml');

  // fromFile returns an object with parse() and validate() methods
  const result = fromFile(parser, filePath);

  // Parse the file
  const { document, diagnostics } = await result.parse();

  if (document) {
    console.log('Parsed from file:', filePath);
    console.log('Title:', document.info().title());
    console.log('Channels:', document.channels().all().map(c => c.id()));
  }

  // Or just validate (faster, no model)
  // const diagnostics = await result.validate();
}

main().catch(console.error);
```

### Error Handling for Missing Files

```js
const { Parser, fromFile } = require('../packages/parser/cjs/index.js');

async function main() {
  const parser = new Parser();
  
  try {
    const { document, diagnostics } = await fromFile(
      parser, 
      '/path/to/nonexistent.yaml'
    ).parse();
    
    if (!document) {
      console.log('Failed to parse:', diagnostics[0]?.message);
    }
  } catch (err) {
    // fromFile throws a Node.js ENOENT error if the file doesn't exist
    if (err.code === 'ENOENT') {
      console.error('File not found:', err.path);
    } else {
      throw err;
    }
  }
}

main().catch(console.error);
```

> **Note:** `fromFile` uses Node.js `fs.readFile`. If the file doesn't exist, it throws `ENOENT` directly (not as a diagnostic). If the file exists but the content is invalid AsyncAPI, errors come back as `diagnostics`.

---

## `fromURL(parser, url)`

```typescript
export function fromURL(parser: Parser, source: string, fetchOptions?: RequestInit): FromResult {
  return {
    async parse(options?) {
      const schema = await fetch(source, fetchOptions).then(r => r.text());
      return parser.parse(schema, { ...options, source });
    },
    async validate(options?) { ... }
  };
}
```

In a Node.js environment before v18, `fromURL` falls back to `node-fetch`. In Node.js 18+ and the browser, it uses the native `fetch` API.

### Example: Parse from GitHub

```js
// scratch/use-case-fromurl.js
const { Parser, fromURL } = require('../packages/parser/cjs/index.js');

async function main() {
  const parser = new Parser();

  const url = 'https://raw.githubusercontent.com/asyncapi/spec/v2.6.0/examples/streetlights-kafka-asyncapi.yml';

  console.log('Fetching and parsing:', url);
  
  const { document, diagnostics } = await fromURL(parser, url).parse();

  if (document) {
    console.log('Title:', document.info().title());
    console.log('AsyncAPI version:', document.version());
    console.log('Number of channels:', document.channels().all().length);
    console.log('Operations:');
    document.operations().all().forEach(op => {
      console.log(`  ${op.id()} [${op.action()}]`);
    });
  } else {
    const errors = diagnostics.filter(d => d.severity === 0);
    console.log('Parse failed:', errors.map(e => e.message).join(', '));
  }
}

main().catch(console.error);
```

### Error Handling for Bad URLs

```js
try {
  const { document, diagnostics } = await fromURL(
    parser,
    'https://example.com/does-not-exist.yaml'
  ).parse();

  if (!document) {
    // HTTP 404 or other issues appear as uncaught-error diagnostics
    console.log('Failed:');
    diagnostics.forEach(d => console.log(` ${d.code}: ${d.message}`));
  }
} catch (err) {
  // Network errors (DNS failure, no connection) throw here
  console.error('Network error:', err.message);
}
```

---

## Validating a URL Without Building a Model

```js
// Validate without full parse
const diagnostics = await fromURL(parser, url).validate();

const errors = diagnostics.filter(d => d.severity === 0);
if (errors.length === 0) {
  console.log('Document at URL is valid.');
} else {
  console.log('Errors:', errors.map(e => e.message));
}
```

---

## Passing Custom HTTP Options

`fromURL` accepts the second argument as `RequestInit` options (same interface as the Fetch API):

```js
// With authentication header
const { document } = await fromURL(
  parser,
  'https://private.example.com/api/asyncapi.yaml',
  {
    headers: {
      Authorization: 'Bearer mytoken123',
    }
  }
).parse();

// With timeout (using AbortController, Node 16+)
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), 5000);

try {
  const { document } = await fromURL(
    parser,
    'https://slow.example.com/asyncapi.yaml',
    { signal: controller.signal }
  ).parse();
} finally {
  clearTimeout(timeout);
}
```

---

## Manual Approach (When You Need More Control)

If you need to handle the file/URL reading yourself:

```js
const { Parser } = require('../packages/parser/cjs/index.js');
const fs = require('fs');
const path = require('path');

const parser = new Parser();
const filePath = '/path/to/asyncapi.yaml';

// Read manually
const content = fs.readFileSync(filePath, 'utf-8');

// Pass source explicitly
const { document } = await parser.parse(content, {
  source: filePath,  // IMPORTANT: needed for external $ref resolution
});
```

The `source` value should be:
- An absolute file path for local files (e.g., `/Users/me/project/asyncapi.yaml`)
- A full URL for remote documents (e.g., `https://example.com/asyncapi.yaml`)
- `undefined` for inline documents with no external refs

---

## Next Step

Read [05-circular-references.md](./05-circular-references.md) to understand how circular `$ref` chains are detected and handled.
