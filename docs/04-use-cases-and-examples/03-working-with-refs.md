# Use Case: Working with $refs

> **Goal:** Understand how `$ref` resolution works for internal refs, external file refs, and HTTP refs. Learn how to inspect the resolved vs original document.

---

## What is a `$ref`?

A `$ref` (JSON Reference) is a string value that acts as a pointer to another location:

```yaml
message:
  $ref: '#/components/messages/UserRegistered'
#          ↑ '#' means "in this same document"
#            '/components/messages/UserRegistered' is the JSON path
```

The parser resolves all `$ref`s before validation and model construction. After resolution, the `$ref` string is replaced with the actual value it points to.

---

## Internal References

Internal refs (`#/...`) point to another part of the same document. They are resolved automatically with no extra configuration:

```js
// scratch/use-case-internal-refs.js
const { Parser } = require('../packages/parser/cjs/index.js');

async function main() {
  const parser = new Parser();

  const doc = `
asyncapi: '2.6.0'
info:
  title: Internal Refs Example
  version: '1.0.0'
channels:
  user/registered:
    publish:
      operationId: onUserRegistered
      message:
        $ref: '#/components/messages/UserRegistered'
      # ↑ Internal ref: resolved to the message defined in components

components:
  messages:
    UserRegistered:
      name: UserRegistered
      payload:
        $ref: '#/components/schemas/UserRegisteredPayload'

  schemas:
    UserRegisteredPayload:
      type: object
      required: [userId, email]
      properties:
        userId:
          type: string
        email:
          type: string
          format: email
`;

  const { document } = await parser.parse(doc);
  if (!document) return;

  // After resolution, message is the full object (no $ref)
  const channel = document.channels().get('user/registered')!;
  const op = channel.operations().all()[0];
  const message = op.messages().all()[0];

  console.log('Message name:', message.name());  // 'UserRegistered'
  console.log('Payload type:', message.payload()?.type());  // 'object'
  console.log('Required fields:', message.payload()?.required());  // ['userId', 'email']
  
  // The JSON path shows where this object is in the ORIGINAL document structure
  console.log('Message JSON path:', message.jsonPath());  
  // '/channels/user~1registered/publish/message' (even though it was a $ref, 
  // the path reflects its defined location after resolution)
}

main().catch(console.error);
```

---

## External File References

External refs point to other files. You **must** provide the `source` option so the parser knows the base path for relative resolution.

Create two files:

**`scratch/main.yaml`:**
```yaml
asyncapi: '2.6.0'
info:
  title: External Refs Example
  version: '1.0.0'
channels:
  user/registered:
    publish:
      operationId: onUserRegistered
      message:
        $ref: './messages/user-registered.yaml'
```

**`scratch/messages/user-registered.yaml`:**
```yaml
name: UserRegistered
payload:
  type: object
  properties:
    userId:
      type: string
    email:
      type: string
      format: email
```

```js
// scratch/use-case-external-refs.js
const { Parser } = require('../packages/parser/cjs/index.js');
const fs = require('fs');
const path = require('path');

async function main() {
  const parser = new Parser();

  const filePath = path.join(__dirname, 'main.yaml');
  const content = fs.readFileSync(filePath, 'utf-8');

  // source is REQUIRED for external file refs to resolve
  const { document, diagnostics } = await parser.parse(content, {
    source: filePath,
    // ↑ Without this, './messages/user-registered.yaml' cannot be resolved
    //   and you'll get: "Error: Could not resolve reference"
  });

  if (!document) {
    console.log('Parse failed:');
    diagnostics.filter(d => d.severity === 0).forEach(d => console.log(d.message));
    return;
  }

  const channel = document.channels().get('user/registered');
  const msg = channel?.operations().all()[0].messages().all()[0];
  console.log('Message name:', msg?.name());     // 'UserRegistered'
  console.log('Payload type:', msg?.payload()?.type()); // 'object'
}

main().catch(console.error);
```

### What happens without `source`?

```js
// Missing source: external refs will fail
const { document, diagnostics } = await parser.parse(content);
// diagnostics will contain: "Error resolving $ref ./messages/user-registered.yaml: ..."
// document will be undefined
```

---

## HTTP/HTTPS References

HTTP refs work the same as file refs, but use a URL as the source:

```js
// scratch/use-case-http-refs.js
const { Parser, fromURL } = require('../packages/parser/cjs/index.js');

async function main() {
  const parser = new Parser();

  // fromURL handles fetching + parsing in one call
  const { document, diagnostics } = await fromURL(
    parser,
    'https://raw.githubusercontent.com/asyncapi/spec/v2.6.0/examples/streetlights-kafka-asyncapi.yml'
  );

  if (document) {
    console.log('Title:', document.info().title());
    console.log('Version:', document.version());
    console.log('Channels:', document.channels().all().map(c => c.id()));
  } else {
    console.log('Failed:', diagnostics.filter(d => d.severity === 0).map(d => d.message));
  }
}

main().catch(console.error);
```

The `fromURL` helper is covered more in [04-load-from-url-and-file.md](./04-load-from-url-and-file.md).

---

## Inspecting the Resolved vs Unresolved Document

The parsed `document` model always represents the **resolved** state — all `$ref`s have been replaced. But you can also access the unresolved input:

```js
const { document, extras } = await parser.parse(content, { source: filePath });

// The resolved document (all $refs replaced):
console.log('Resolved JSON:', JSON.stringify(document?.json(), null, 2).slice(0, 200));

// The raw Spectral document (before model wrapping):
// extras.document.data is the resolved JavaScript object
// extras.document.source is the source string
console.log('Spectral doc source:', extras?.document.source);
```

To access the ORIGINAL (unresolved) input:

```js
// document.meta().asyncapi.input is the original input
const originalInput = document?.meta().asyncapi.input;
```

---

## Ref Resolution Order and Caching

The resolver processes refs breadth-first and caches results. If the same file is referenced from multiple places:

```yaml
components:
  schemas:
    SentAt:
      $ref: './schemas/sent-at.yaml'
    AnotherSchema:
      properties:
        sentAt:
          $ref: './schemas/sent-at.yaml'  # same file, referenced twice
```

The file is only fetched once. The second reference uses the cached result. This is why parsing the same document twice (or referencing the same external resource repeatedly) is not as expensive as it seems.

---

## Handling Resolution Errors

If a `$ref` cannot be resolved (file not found, HTTP 404, etc.), it becomes a diagnostic error:

```js
const doc = `
asyncapi: '2.6.0'
info:
  title: Bad Ref
  version: '1.0.0'
channels:
  user/registered:
    publish:
      message:
        $ref: './does-not-exist.yaml'
`;

const { document, diagnostics } = await parser.parse(doc, { source: __filename });
// document === undefined
// diagnostics has an error with code 'asyncapi-document-resolved' or similar
console.log(diagnostics.filter(d => d.severity === 0).map(d => d.message));
// Something like: "Error resolving $ref: ENOENT: no such file or directory './does-not-exist.yaml'"
```

---

## Custom Protocol Resolver

Add resolvers for non-standard protocols:

```js
const parser = new Parser({
  __unstable: {
    resolver: {
      resolvers: {
        'myschema': {
          async resolve(uri) {
            const schemaId = uri.path().replace(/^\//, '');
            // Fetch from your internal schema registry
            const response = await fetch(`https://registry.internal/schemas/${schemaId}`);
            return response.text();
          }
        }
      }
    }
  }
});
```

Now refs like `myschema://user-registered` will be resolved via this function.

---

## Next Step

Read [04-load-from-url-and-file.md](./04-load-from-url-and-file.md) for the `fromURL()` and `fromFile()` convenience helpers.
