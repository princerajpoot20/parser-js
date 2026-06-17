# Extending Schema Parsers

> **Goal:** Build, register, and test a custom `SchemaParser` plugin. Understand how the plugin system works end-to-end so you can debug existing parsers and add new ones.

---

## Recap: The Plugin Interface

A schema parser must implement three methods:

```typescript
interface SchemaParser<D = unknown, M = unknown> {
  getMimeTypes(): string[];
  validate(input: ValidateSchemaInput<D, M>): void | SchemaValidateResult[] | Promise<void | SchemaValidateResult[]>;
  parse(input: ParseSchemaInput<D, M>): AsyncAPISchema | Promise<AsyncAPISchema>;
}
```

**`getMimeTypes()`**: Returns the list of MIME type strings that this parser handles. These are the values that appear in the `schemaFormat` field of an AsyncAPI message.

**`validate()`**: Validates `input.data` against the schema format's rules. Returns `[]` or `void` if valid; returns an array of `{ message, path }` objects for each error.

**`parse()`**: Converts `input.data` to an AsyncAPI/JSON Schema object. For non-JSON-Schema formats, this is where you convert the schema to JSON Schema. For JSON Schema, this is an identity function.

---

## Full Working Example: A "Version String" Schema Parser

This parser handles a hypothetical `application/vnd.semver+yaml;version=1.0` format where payloads are semantic version strings:

```yaml
channels:
  app/deployed:
    publish:
      message:
        schemaFormat: 'application/vnd.semver+yaml;version=1.0'
        payload: ">= 1.0.0"   # just a semver range string
```

**File:** `scratch/semver-schema-parser.js`

```js
const semverSchemaParser = {
  getMimeTypes() {
    return [
      'application/vnd.semver+yaml;version=1.0',
      'application/vnd.semver+json;version=1.0',
    ];
  },

  async validate(input) {
    // input.data is the payload value from the AsyncAPI doc
    // (could be any type — string, object, array)
    const { data } = input;
    
    if (typeof data !== 'string') {
      return [{
        message: `Semver schema must be a string, got ${typeof data}`,
        path: [...input.path],
      }];
    }
    
    // Validate it's a valid semver range string
    // (simplified: just check it contains digits)
    const semverPattern = /^[><=~^]?[\s]*\d+\.\d+\.\d+/;
    if (!semverPattern.test(data)) {
      return [{
        message: `Invalid semver range: "${data}". Expected format like ">= 1.0.0"`,
        path: [...input.path],
      }];
    }
    
    return []; // Valid
  },

  async parse(input) {
    // Convert the semver range string to a JSON Schema
    // that validates version strings matching the range
    const { data } = input;
    
    return {
      type: 'string',
      description: `A semantic version string satisfying the range: ${data}`,
      pattern: '^\\d+\\.\\d+\\.\\d+(-[\\w.]+)?(\\+[\\w.]+)?$',
      // x-semver-range: data (could store original for tooling)
    };
  },
};

module.exports = { semverSchemaParser };
```

**Usage:**

```js
// scratch/use-semver-parser.js
const { Parser } = require('../packages/parser/cjs/index.js');
const { semverSchemaParser } = require('./semver-schema-parser.js');

async function main() {
  const parser = new Parser({
    schemaParsers: [semverSchemaParser],
  });

  const doc = `
asyncapi: '2.6.0'
info:
  title: App Deployments
  version: '1.0.0'
channels:
  app/deployed:
    publish:
      message:
        schemaFormat: 'application/vnd.semver+yaml;version=1.0'
        payload: ">= 1.0.0"
`;

  const { document, diagnostics } = await parser.parse(doc);
  
  const errors = diagnostics.filter(d => d.severity === 0);
  console.log('Errors:', errors.map(e => `${e.code}: ${e.message}`));
  
  if (document) {
    const channel = document.channels().get('app/deployed');
    const msg = channel?.operations().all()[0].messages().all()[0];
    
    // payload() returns the CONVERTED JSON Schema
    console.log('\nConverted JSON Schema:');
    console.log(JSON.stringify(msg?.payload()?.json(), null, 2));
    
    // Original semver string preserved in x-parser-original-payload
    console.log('\nOriginal payload:', msg?.json()?.['x-parser-original-payload']);
  }
}

main().catch(console.error);
```

---

## How `parseSchemas` Invokes Your Parser

**File:** `packages/parser/src/custom-operations/parse-schema.ts`

The traversal looks for all payloads and headers:

```typescript
async function parseSchemasV2(parser: Parser, detailed: DetailedAsyncAPI) {
  const version = detailed.semver.version;
  
  // Walk all message payloads in the document
  JSONPath({
    path: '$.channels.*.[publish,subscribe].message.payload',
    json: detailed.parsed,
    resultType: 'all',
    callback({ value, path }) {
      const schemaFormat = getSchemaFormatFromPath(detailed.parsed, path);
      const defaultSchemaFormat = getDefaultSchemaFormat(version);
      
      const errors = await validateSchema(parser, {
        asyncapi: detailed,
        data: value,
        meta: {},
        path: path,
        schemaFormat,
        defaultSchemaFormat,
      });
      
      // Convert errors to diagnostics...
      
      const converted = await parseSchema(parser, {
        asyncapi: detailed,
        data: value,
        meta: {},
        path: path,
        schemaFormat,
        defaultSchemaFormat,
      });
      
      // Replace payload with converted schema
      // Save original in x-parser-original-payload
    }
  });
}
```

---

## Testing Your Schema Parser

### Unit Test: `validate()`

```js
// scratch/test-semver-parser-validate.js
const { semverSchemaParser } = require('./semver-schema-parser.js');

async function testValidate() {
  // Mock input
  const baseInput = {
    asyncapi: { semver: { version: '2.6.0' } },
    meta: {},
    path: ['channels', 'app/deployed', 'publish', 'message', 'payload'],
    schemaFormat: 'application/vnd.semver+yaml;version=1.0',
    defaultSchemaFormat: 'application/vnd.aai.asyncapi;version=2.6.0',
  };
  
  // Test valid
  const validResult = await semverSchemaParser.validate({ ...baseInput, data: '>= 1.0.0' });
  console.log('Valid semver errors:', validResult); // should be []

  // Test invalid
  const invalidResult = await semverSchemaParser.validate({ ...baseInput, data: 'not-a-version' });
  console.log('Invalid semver errors:', invalidResult); // should have one error
  
  // Test wrong type
  const wrongTypeResult = await semverSchemaParser.validate({ ...baseInput, data: { type: 'string' } });
  console.log('Wrong type errors:', wrongTypeResult); // should have one error
}

testValidate().catch(console.error);
```

### Unit Test: `parse()`

```js
async function testParse() {
  const baseInput = {
    asyncapi: { semver: { version: '2.6.0' } },
    meta: {},
    path: ['channels', 'app/deployed', 'publish', 'message', 'payload'],
    schemaFormat: 'application/vnd.semver+yaml;version=1.0',
    defaultSchemaFormat: 'application/vnd.aai.asyncapi;version=2.6.0',
  };
  
  const converted = await semverSchemaParser.parse({ ...baseInput, data: '>= 1.0.0' });
  console.log('Converted schema:', JSON.stringify(converted, null, 2));
  // Should be: { type: 'string', description: '...', pattern: '...' }
}

testParse().catch(console.error);
```

---

## How Validation Errors Become Diagnostics

When your `validate()` function returns errors, `parse-schema.ts` converts them to Spectral diagnostics. Each error's `path` is relative to the payload node in the document — the absolute path is computed by prepending the payload's full JSONPath.

```typescript
// In parse-schema.ts (simplified)
const errors = await validateSchema(parser, input);
for (const error of errors) {
  diagnostics.push({
    code: 'asyncapi2-schemas',  // always this code for schema validation
    message: error.message,
    path: [...absolutePayloadPath, ...error.path],
    severity: DiagnosticSeverity.Error,
    // ...
  });
}
```

This is why your `validate()` should return paths relative to the schema root, not absolute paths.

---

## Checking Your Parser Is Registered

```js
const { Parser } = require('./packages/parser/cjs/index.js');
const { semverSchemaParser } = require('./scratch/semver-schema-parser.js');

const parser = new Parser({ schemaParsers: [semverSchemaParser] });

// Check what MIME types are registered
console.log('Registered MIME types:');
for (const [mimeType] of parser.parserRegistry) {
  console.log(' ', mimeType);
}
// Should include: 'application/vnd.semver+yaml;version=1.0'
```

---

## Integration Into the Official Schema Parsers

If you are adding a new officially supported schema parser (like a new format the community wants):

1. Create a new package (e.g., `@asyncapi/myformat-schema-parser`) following the pattern of `@asyncapi/avro-schema-parser`
2. Add it to `multi-parser`'s `includeSchemaParsers` list in `packages/multi-parser/src/parse.ts`
3. Add it as a devDependency in `packages/parser/package.json` for testing
4. Add integration tests using `devDependencies` import

---

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| `getMimeTypes()` returns wrong MIME string | "Unknown schema format" error | Check MIME type matches exactly what's in the `schemaFormat` field |
| `validate()` returns path to the parent, not the issue | Error path is off by one | Use `path: [...input.path]` for root issues, add sub-paths for nested issues |
| `parse()` returns `undefined` | TypeError on payload access | Always return a valid JSON Schema object (at minimum `{}`) |
| `validate()` throws instead of returning errors | Uncaught error diagnostic | Wrap in try/catch, return `[{ message: err.message, path: [] }]` |
| Parser not registered at parse time | "Unknown schema format" | Register before calling `parse()`, not after |

---

## Next Step

You have completed the contributor guide! You are now equipped to:
- Classify and reproduce any GitHub issue
- Debug through the parsing pipeline
- Add new validation rules
- Build and test custom schema parser plugins

Return to the [main README](../README.md) for the full documentation index.
