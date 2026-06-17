# Use Case: Custom Schema Formats

> **Goal:** Register a schema parser plugin (Avro example), parse a document with a non-JSON-Schema payload, and inspect the converted schema output.

---

## When You Need a Custom Schema Parser

If your AsyncAPI document uses a `schemaFormat` field with a non-default MIME type, the parser needs a plugin to handle it:

```yaml
channels:
  user/registered:
    publish:
      message:
        schemaFormat: 'application/vnd.apache.avro+json;version=1.9.0'
        # ↑ This is NOT JSON Schema — the default parser can't handle it
        payload:
          type: record
          name: UserRegistered
          fields:
            - name: userId
              type: string
```

Without the Avro parser registered:
```
[Error] asyncapi2-schemas: Unknown schema format: "application/vnd.apache.avro+json;version=1.9.0"
```

---

## Installing the Avro Schema Parser

```bash
npm install @asyncapi/avro-schema-parser
```

Or in the dev context of this repo, it is already a devDependency of `packages/parser`.

---

## Example: Parse a Document with Avro Payloads

Create `scratch/use-case-avro.js`:

```js
// scratch/use-case-avro.js
const { Parser } = require('../packages/parser/cjs/index.js');

// Import from the devDependency in packages/parser
const { AvroSchemaParser } = require('../packages/parser/node_modules/@asyncapi/avro-schema-parser');

async function main() {
  const parser = new Parser({
    schemaParsers: [AvroSchemaParser()],
    // ↑ Register the Avro parser at construction time
  });

  const doc = `
asyncapi: '2.6.0'
info:
  title: Avro Schema Example
  version: '1.0.0'
channels:
  user/registered:
    publish:
      operationId: onUserRegistered
      message:
        name: UserRegistered
        schemaFormat: 'application/vnd.apache.avro+json;version=1.9.0'
        payload:
          type: record
          name: UserRegistered
          namespace: com.example
          doc: A user registration event
          fields:
            - name: userId
              type: string
              doc: The unique user identifier
            - name: email
              type: string
            - name: registeredAt
              type:
                type: long
                logicalType: timestamp-millis
            - name: optionalField
              type: ["null", "string"]
              default: null
`;

  const { document, diagnostics } = await parser.parse(doc);

  const errors = diagnostics.filter(d => d.severity === 0);
  if (errors.length > 0) {
    console.log('Errors:');
    errors.forEach(d => console.log(`  ${d.code}: ${d.message}`));
    return;
  }

  if (!document) return;

  const channel = document.channels().get('user/registered')!;
  const msg = channel.operations().all()[0].messages().all()[0];

  console.log('=== Message: UserRegistered ===');
  console.log('Name:', msg.name());

  // After Avro parsing, payload() returns the CONVERTED JSON Schema
  const payload = msg.payload();
  console.log('\nConverted JSON Schema:');
  console.log('  type:', payload?.type());
  console.log('  properties:', Object.keys(payload?.properties() ?? {}));

  // The ORIGINAL Avro schema is preserved in x-parser-original-payload
  const originalPayload = msg.json()['x-parser-original-payload'];
  console.log('\nOriginal Avro schema:');
  console.log('  Avro type:', originalPayload?.type);
  console.log('  Avro name:', originalPayload?.name);
  console.log('  Avro fields:', originalPayload?.fields?.map((f: any) => f.name));

  // The original schemaFormat is preserved in x-parser-original-schema-format
  const originalFormat = msg.json()['x-parser-original-schema-format'];
  console.log('\nOriginal schema format:', originalFormat);
}

main().catch(console.error);
```

Run:

```bash
node scratch/use-case-avro.js
```

Expected output:

```
=== Message: UserRegistered ===
Name: UserRegistered

Converted JSON Schema:
  type: object
  properties: [ 'userId', 'email', 'registeredAt', 'optionalField' ]

Original Avro schema:
  Avro type: record
  Avro name: UserRegistered
  Avro fields: [ 'userId', 'email', 'registeredAt', 'optionalField' ]

Original schema format: application/vnd.apache.avro+json;version=1.9.0
```

---

## Registering Multiple Schema Parsers

```js
const { Parser } = require('../packages/parser/cjs/index.js');
const { AvroSchemaParser } = require('@asyncapi/avro-schema-parser');
const { OpenAPISchemaParser } = require('@asyncapi/openapi-schema-parser');
const { ProtoBuffSchemaParser } = require('@asyncapi/protobuf-schema-parser');

const parser = new Parser({
  schemaParsers: [
    AvroSchemaParser(),
    OpenAPISchemaParser(),
    ProtoBuffSchemaParser(),
  ]
});
```

Each parser registers itself for specific MIME types. When the parser encounters a `schemaFormat`, it looks up the MIME type in the registry.

---

## Registering After Construction

```js
const parser = new Parser();
// Register manually
parser.registerSchemaParser(AvroSchemaParser());

// Now parse Avro documents
const { document } = await parser.parse(avroDoc);
```

---

## Writing a Minimal Custom Schema Parser

If you have an internal schema format, you can write your own parser:

```js
// A minimal schema parser for a hypothetical "Simple Type" format
const simpleTypeParser = {
  getMimeTypes() {
    return ['application/x-simple-type;version=1.0'];
  },
  
  async validate(input) {
    // input.data is the raw payload from the YAML/JSON
    const validTypes = ['text', 'number', 'boolean', 'timestamp'];
    if (!validTypes.includes(input.data)) {
      return [{
        message: `Unknown simple type: "${input.data}". Valid types: ${validTypes.join(', ')}`,
        path: [...input.path, 'payload'],
      }];
    }
    return []; // empty = valid
  },
  
  async parse(input) {
    // Convert to JSON Schema
    const typeMap = {
      text: { type: 'string' },
      number: { type: 'number' },
      boolean: { type: 'boolean' },
      timestamp: { type: 'string', format: 'date-time' },
    };
    return typeMap[input.data] || { type: 'string' };
  }
};

const parser = new Parser({
  schemaParsers: [simpleTypeParser]
});
```

---

## What Happens When `schemaFormat` Is Not Registered

If you parse a document with an unknown `schemaFormat` and haven't registered a parser for it:

```
[Error] asyncapi2-schemas: Unknown schema format: "application/vnd.apache.avro+json;version=1.9.0"
[Error] asyncapi2-schemas: Cannot validate and parse given schema due to unknown schema format...
```

`document` will be `undefined` (these are errors, not warnings). You must register the appropriate schema parser before calling `parse()`.

---

## Disabling Schema Parsing

If you want to skip schema validation entirely (e.g., you're only interested in the channel/operation structure):

```js
const { document } = await parser.parse(doc, {
  parseSchemas: false,  // no schema validation or conversion
});
```

With `parseSchemas: false`, even unknown `schemaFormat` values will not produce errors, and Avro schemas won't be converted — `msg.payload()` will return the raw Avro JSON.

---

## Next Step

Read [07-stringify-and-unstringify.md](./07-stringify-and-unstringify.md) for safe serialization of parsed documents (including those with circular references).
