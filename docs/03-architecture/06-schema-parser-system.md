# Schema Parser System

> **Goal:** Understand how the pluggable schema parser system works — the `SchemaParser` interface, how schemas get dispatched to parsers, what the built-in JSON Schema parser does, and how to add support for Avro, Protobuf, or other formats.

---

## Why a Pluggable System?

AsyncAPI documents can describe message payloads using different schema languages:

| Schema format | Example use case |
|--------------|-----------------|
| JSON Schema Draft-07 | Most common, the default |
| Avro | Kafka-heavy systems |
| Protobuf | gRPC systems |
| OpenAPI Schema Object | REST + events mixed codebases |
| RAML Data Types | RAML users |

The parser cannot bundle validators for every format — some are large (Protobuf parser is substantial) and new formats emerge over time. The solution: a **plugin interface** called `SchemaParser`.

---

## The `SchemaParser` Interface

**File:** `packages/parser/src/schema-parser/index.ts`

```typescript
export interface SchemaParser<D = unknown, M = unknown> {
  // Validate the payload against the schema format's rules
  // Returns an array of validation errors (empty = valid)
  validate: (input: ValidateSchemaInput<D, M>) => 
    void | SchemaValidateResult[] | Promise<void | SchemaValidateResult[]>;

  // Convert/parse the payload into an AsyncAPI Schema Object
  // For JSON Schema parsers, this is typically identity (return input.data)
  // For Avro/Protobuf, this converts the format-specific schema to JSON Schema
  parse: (input: ParseSchemaInput<D, M>) => AsyncAPISchema | Promise<AsyncAPISchema>;

  // Returns the list of MIME type strings this parser handles
  getMimeTypes: () => Array<string>;
}
```

### `ValidateSchemaInput` and `ParseSchemaInput`

Both share the same structure:

```typescript
interface ValidateSchemaInput<D = unknown, M = unknown> {
  readonly asyncapi: DetailedAsyncAPI;    // full document context
  readonly data: D;                       // the payload/schema data
  readonly meta: M;                       // message metadata
  readonly path: Array<string | number>;  // JSON path to this payload in the document
  readonly schemaFormat: string;          // the MIME type string
  readonly defaultSchemaFormat: string;   // fallback MIME type for this AsyncAPI version
}
```

### `SchemaValidateResult`

```typescript
interface SchemaValidateResult {
  message: string;
  path: Array<string | number>;
}
```

---

## How Parsers Are Registered

**File:** `packages/parser/src/schema-parser/index.ts`

```typescript
export function registerSchemaParser(parser: Parser, schemaParser: SchemaParser) {
  if (
    typeof schemaParser !== 'object' || 
    typeof schemaParser.validate !== 'function' || 
    typeof schemaParser.parse !== 'function' || 
    typeof schemaParser.getMimeTypes !== 'function'
  ) {
    throw new Error('Custom parser must have "parse()", "validate()" and "getMimeTypes()" functions.');
  }

  // Register each MIME type this parser handles
  schemaParser.getMimeTypes().forEach(schemaFormat => {
    parser.parserRegistry.set(schemaFormat, schemaParser);
  });
}
```

The registry is a simple `Map<string, SchemaParser>` keyed by MIME type string.

Registration happens in three ways:
1. **Automatic**: `AsyncAPISchemaParser` is always registered in the `Parser` constructor
2. **Constructor option**: pass `schemaParsers: [myParser]` to the `Parser` constructor
3. **Manual**: call `parser.registerSchemaParser(myParser)` after construction

---

## The Default: `AsyncAPISchemaParser`

**File:** `packages/parser/src/schema-parser/asyncapi-schema-parser.ts`

This is the built-in parser for JSON Schema. It handles these MIME types:

```
application/schema;version=draft-07
application/schema+json;version=draft-07
application/schema+yaml;version=draft-07
application/vnd.aai.asyncapi;version=2.0.0
application/vnd.aai.asyncapi;version=2.1.0
... (one per AsyncAPI version)
application/vnd.aai.asyncapi;version=3.0.0
application/vnd.aai.asyncapi+json;version=3.0.0
application/vnd.aai.asyncapi+yaml;version=3.0.0
```

When no `schemaFormat` is specified in the AsyncAPI document, the default format is:
```
application/vnd.aai.asyncapi;version=<doc-version>
```

So every payload gets validated, even without an explicit `schemaFormat`.

### How the JSON Schema validator works

```typescript
async function validate(input: ValidateSchemaInput): Promise<SchemaValidateResult[]> {
  const version = input.asyncapi.semver.version;  // e.g., '2.6.0'
  const validator = getSchemaValidator(version);  // cached Ajv validator

  const valid = validator(input.data);
  if (!valid && validator.errors) {
    return ajvToSpectralResult(input.path, [...validator.errors]);
  }
  return [];
}
```

The Ajv validator is built from the official `@asyncapi/specs` JSON Schema for the matching version. It validates the payload against the AsyncAPI Schema Object definition (a subset of the full AsyncAPI schema).

```typescript
function preparePayloadSchema(asyncapiSchema: JSONSchema7, version: string): JSONSchema7 {
  const payloadSchema = `http://asyncapi.com/definitions/${version}/schema.json`;
  const definitions = asyncapiSchema.definitions;
  // Remove duplicate meta schemas that Ajv already has
  delete definitions['http://json-schema.org/draft-07/schema'];
  delete definitions['http://json-schema.org/draft-04/schema'];
  return {
    $ref: payloadSchema,
    definitions
  };
}
```

### `parse()` for JSON Schema is identity

```typescript
async function parse(input: ParseSchemaInput): Promise<AsyncAPISchema> {
  return input.data as AsyncAPISchema;  // no conversion needed
}
```

JSON Schema doesn't need to be converted — it is already in the right format. Other parsers like Avro convert to JSON Schema here.

---

## How `schemaFormat` Selects the Parser

During the `parseSchemas` custom operation, for each payload found:

```typescript
// parse-schema.ts (simplified)
const schemaFormat = getSchemaFormat(message.schemaFormat, asyncapi.semver.version);
const errors = await validateSchema(parser, { ...input, schemaFormat });
const convertedSchema = await parseSchema(parser, { ...input, schemaFormat });
```

`getSchemaFormat()` returns:
- The `schemaFormat` field value if present
- Otherwise `application/vnd.aai.asyncapi;version=<version>` (default JSON Schema)

`validateSchema()` and `parseSchema()` look up the MIME type in `parser.parserRegistry`:

```typescript
export async function validateSchema(parser: Parser, input: ValidateSchemaInput) {
  const schemaParser = parser.parserRegistry.get(input.schemaFormat);
  if (schemaParser === undefined) {
    return [
      { message: `Unknown schema format: "${schemaFormat}"`, path: [...path, 'schemaFormat'] },
      { message: `Cannot validate...`, path: [...path, 'payload'] }
    ];
  }
  return schemaParser.validate(input);
}
```

**If the MIME type is not in the registry, validation fails** with an "Unknown schema format" error. This is intentional — you must explicitly register a parser for any non-default format.

---

## Example: Registering the Avro Parser

```typescript
import { Parser } from '@asyncapi/parser';
import { AvroSchemaParser } from '@asyncapi/avro-schema-parser';

const parser = new Parser({
  schemaParsers: [AvroSchemaParser()],
});
```

Or manually:

```typescript
const parser = new Parser();
parser.registerSchemaParser(AvroSchemaParser());
```

Now documents with Avro payloads will be validated and converted:

```yaml
channels:
  user/registered:
    publish:
      message:
        schemaFormat: 'application/vnd.apache.avro+json;version=1.9.0'
        payload:
          type: record
          name: UserRegistered
          fields:
            - name: userId
              type: string
```

After parsing:
- `message.payload()` returns the converted JSON Schema
- `message.json()['x-parser-original-payload']` has the original Avro schema
- `message.json()['x-parser-original-schema-format']` has the original MIME type

---

## Community Schema Parsers

| Package | `schemaFormat` MIME type prefix | GitHub |
|---------|--------------------------------|--------|
| `@asyncapi/avro-schema-parser` | `application/vnd.apache.avro` | [asyncapi/avro-schema-parser](https://github.com/asyncapi/avro-schema-parser) |
| `@asyncapi/openapi-schema-parser` | `application/vnd.oai.openapi` | [asyncapi/openapi-schema-parser](https://github.com/asyncapi/openapi-schema-parser) |
| `@asyncapi/protobuf-schema-parser` | `application/vnd.google.protobuf` | [asyncapi/protobuf-schema-parser](https://github.com/asyncapi/protobuf-schema-parser) |
| `@asyncapi/raml-dt-schema-parser` | `application/raml+yaml` | [asyncapi/raml-dt-schema-parser](https://github.com/asyncapi/raml-dt-schema-parser) |

---

## Writing Your Own Schema Parser

Minimum implementation:

```typescript
import type { SchemaParser, ParseSchemaInput, ValidateSchemaInput } from '@asyncapi/parser';

export function MySchemaParser(): SchemaParser {
  return {
    getMimeTypes() {
      return ['application/x-my-format;version=1.0'];
    },
    
    async validate(input: ValidateSchemaInput) {
      // Validate input.data against your schema format
      // Return [] if valid, or [{ message, path }] for each error
      const errors = mySchemaLib.validate(input.data);
      return errors.map(err => ({
        message: err.message,
        path: [...input.path, ...err.path],
      }));
    },
    
    async parse(input: ParseSchemaInput) {
      // Convert your schema format to AsyncAPI/JSON Schema
      const jsonSchema = mySchemaLib.toJsonSchema(input.data);
      return jsonSchema;
    },
  };
}
```

Register it:

```typescript
const parser = new Parser({
  schemaParsers: [MySchemaParser()],
});
```

---

## Next Step

You have completed the Architecture chapter. Move to [Chapter 4: Use Cases & Examples](../04-use-cases-and-examples/01-parse-asyncapi-document.md) for hands-on scenarios.
