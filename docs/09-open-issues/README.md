# Open GitHub Issues — Complete Guide

This document explains every currently open issue in [asyncapi/parser-js](https://github.com/asyncapi/parser-js/issues), from the simplest to the most complex.

Each issue follows this structure:
- **What is the problem** — plain English
- **Why it happens** — which part of the code causes it
- **How to reproduce** — a script you can run locally
- **How to fix it** — specific files and changes

---

## Reading Order

| Difficulty | Issues |
|-----------|--------|
| Beginner (missing methods) | [#1075](#1075), [#1076](#1076), [#1067](#1067), [#1137](#1137) |
| Intermediate (validation gaps) | [#875](#875), [#876](#876), [#878](#878), [#877](#877), [#793](#793) |
| Intermediate (runtime bugs) | [#874](#874), [#1063](#1063) |
| Advanced (pipeline bugs) | [#1099](#1099), [#924](#924), [#1098](#1098) |
| Expert (architecture) | [#403](#403), [#761](#761), [#1065](#1065), [#1131](#1131) |
| Meta | [#1167](#1167), [#1181](#1181) |

---

## Quick Links

| # | Title | Type | Difficulty |
|---|-------|------|-----------|
| [#1075](#1075) | Missing `server.title()` function | Bug | Beginner |
| [#1076](#1076) | Missing `server.summary()` function | Bug | Beginner |
| [#1067](#1067) | Missing `channel.title()` function | Feature | Beginner |
| [#1137](#1137) | Missing `title`/`summary` in v3 ServerObject spec types | Bug | Beginner |
| [#875](#875) | No error when channel has parameters but `address` has none | Bug | Intermediate |
| [#876](#876) | No error when reply.address.location used but channel address is not null | Bug | Intermediate |
| [#878](#878) | `parse()` returns `undefined` with no explanation for invalid doc | Bug | Intermediate |
| [#877](#877) | Parser returns `undefined` for docs with message bindings in v3 | Bug | Intermediate |
| [#793](#793) | False "messageId must be unique" error when reusing same message | Bug | Intermediate |
| [#874](#874) | `SecurityRequirements.map()` throws a crash error | Bug | Intermediate |
| [#1063](#1063) | Wrong `x-parser-schema-id` when schemas are in external files | Bug | Intermediate |
| [#1099](#1099) | Custom schema parsers not triggered for payloads via `operation.channel` | Bug | Advanced |
| [#924](#924) | v3 rules don't catch invalid cross-file external references | Bug | Advanced |
| [#1098](#1098) | No way to disable `$ref` dereferencing | Bug | Advanced |
| [#403](#403) | JSON Schema `$id`-based references not resolved correctly | Bug | Expert |
| [#761](#761) | Parser uses a deprecated `$ref` resolver library | Enhancement | Expert |
| [#1065](#1065) | `@asyncapi/multi-parser` depends on vulnerable `jsonpath-plus` | Bug | Expert |
| [#1131](#1131) | Browser bundle doesn't work in GraalVM (Java) | Enhancement | Expert |
| [#1167](#1167) | Microgrant Program 2026-06 aggregated issue | Meta | N/A |
| [#1181](#1181) | Microgrant Program 2026-07 aggregated issue | Meta | N/A |

---

## Individual Issue Details

---

<a name="1075"></a>
## Issue #1075 — Missing `server.title()` function

**Link:** https://github.com/asyncapi/parser-js/issues/1075  
**Labels:** bug, good first issue  
**Difficulty:** Beginner

### What is the problem?

The AsyncAPI spec allows servers to have a `title` field (especially in v3):

```yaml
servers:
  production:
    host: mqtt.example.com
    protocol: mqtt
    title: Production MQTT Broker
```

But when you parse this and try to call `server.title()`, you get a `TypeError: server.title is not a function`. The method simply doesn't exist in the model class.

### Why it happens

The `ServerInterface` and concrete `Server` model classes are missing the `title()` method. The `TitleMixinInterface` (which provides `title()` and `hasTitle()`) is not mixed into the server interface.

Look at these files:
- **`packages/parser/src/models/server.ts`** — the `ServerInterface` that `Server` must implement
- **`packages/parser/src/models/v2/server.ts`** — concrete v2 Server class
- **`packages/parser/src/models/v3/server.ts`** — concrete v3 Server class

### How to reproduce

```js
// scratch/reproduce-1075.js
const { Parser } = require('./packages/parser/cjs/index.js');

async function main() {
  const parser = new Parser();
  const { document } = await parser.parse(`
asyncapi: '3.0.0'
info:
  title: Test
  version: '1.0.0'
servers:
  production:
    host: mqtt.example.com
    protocol: mqtt
    title: My Production Server
`);

  const server = document?.servers().get('production');
  console.log('Server id:', server?.id());
  
  // This crashes:
  try {
    console.log('Server title:', server?.title());
  } catch (e) {
    console.error('ERROR:', e.message); // TypeError: server.title is not a function
  }
  
  // Raw JSON works (confirms the data IS there):
  console.log('Raw title from JSON:', server?.json()?.title); // "My Production Server"
}

main().catch(console.error);
```

### How to fix it

**Step 1:** Add `title` and `hasTitle` to `ServerInterface`

In `packages/parser/src/models/server.ts`:
```typescript
// Add TitleMixinInterface to the import
import type { TitleMixinInterface, ... } from './mixins';

// Add to ServerInterface:
export interface ServerInterface extends BaseModel<...>, 
  TitleMixinInterface,  // ← ADD THIS
  ... {
  // ... existing methods
}
```

**Step 2:** Implement `title()` in both v2 and v3 Server classes

In `packages/parser/src/models/v2/server.ts` AND `packages/parser/src/models/v3/server.ts`:
```typescript
title(): string | undefined {
  return this._json.title;
}

hasTitle(): boolean {
  return !!this._json.title;
}
```

**Step 3:** Write tests in `packages/parser/test/models/v2/server.spec.ts` and `test/models/v3/server.spec.ts`:
```typescript
it('should return title when defined', () => {
  const server = new Server({ title: 'My Server', url: 'mqtt.example.com', protocol: 'mqtt' }, { asyncapi, pointer: '/servers/test', id: 'test' });
  expect(server.title()).toBe('My Server');
  expect(server.hasTitle()).toBe(true);
});
it('should return undefined when title is missing', () => {
  const server = new Server({ url: 'mqtt.example.com', protocol: 'mqtt' }, { asyncapi, pointer: '/servers/test', id: 'test' });
  expect(server.title()).toBeUndefined();
  expect(server.hasTitle()).toBe(false);
});
```

---

<a name="1076"></a>
## Issue #1076 — Missing `server.summary()` function

**Link:** https://github.com/asyncapi/parser-js/issues/1076  
**Labels:** bug, good first issue  
**Difficulty:** Beginner

### What is the problem?

Very similar to #1075. The AsyncAPI spec allows servers to have a `summary` field:

```yaml
servers:
  production:
    host: mqtt.example.com
    protocol: mqtt
    summary: "The main broker for production traffic"
```

But `server.summary()` doesn't exist — the method is missing.

### Why it happens

`SummaryMixinInterface` (which provides `summary()` and `hasSummary()`) is not implemented in the `Server` model. You'll notice that channels and operations already have `.summary()` — servers are just missing it.

### How to reproduce

```js
// scratch/reproduce-1076.js
const { Parser } = require('./packages/parser/cjs/index.js');

async function main() {
  const parser = new Parser();
  const { document } = await parser.parse(`
asyncapi: '3.0.0'
info:
  title: Test
  version: '1.0.0'
servers:
  production:
    host: mqtt.example.com
    protocol: mqtt
    summary: "Main production broker"
`);
  const server = document?.servers().get('production');
  
  try {
    console.log('Summary:', server?.summary()); // crashes
  } catch (e) {
    console.error('ERROR:', e.message);
  }
  
  // The data is there in raw JSON:
  console.log('Raw summary:', server?.json()?.summary); // "Main production broker"
}
main().catch(console.error);
```

### How to fix it

Same pattern as #1075 but for `SummaryMixinInterface`:

**In `packages/parser/src/models/server.ts`:**
```typescript
import type { SummaryMixinInterface, ... } from './mixins';

export interface ServerInterface extends BaseModel<...>, 
  SummaryMixinInterface,  // ← ADD THIS
  ... {
```

**In both v2 and v3 Server classes:**
```typescript
summary(): string | undefined {
  return this._json.summary;
}

hasSummary(): boolean {
  return !!this._json.summary;
}
```

> **Note:** Issues #1075 and #1076 can be fixed together in a single PR since they touch the same files and are very similar changes. Issue #1137 (below) is also related.

---

<a name="1067"></a>
## Issue #1067 — Missing `channel.title()` function

**Link:** https://github.com/asyncapi/parser-js/issues/1067  
**Labels:** enhancement  
**Difficulty:** Beginner

### What is the problem?

The AsyncAPI v3 spec allows channels to have an optional `title` field:

```yaml
channels:
  userRegistered:
    address: user/registered
    title: "User Registration Channel"
    messages:
      UserRegistered:
        payload:
          type: object
```

But `channel.title()` doesn't work — the channel model doesn't expose this field.

### Why it happens

The `ChannelInterface` (in `packages/parser/src/models/channel.ts`) does not include `TitleMixinInterface`. Same root cause as #1075 and #1076.

### How to reproduce

```js
// scratch/reproduce-1067.js
const { Parser } = require('./packages/parser/cjs/index.js');

async function main() {
  const parser = new Parser();
  const { document } = await parser.parse(`
asyncapi: '3.0.0'
info:
  title: Test
  version: '1.0.0'
channels:
  userRegistered:
    address: user/registered
    title: "User Registration Channel"
    messages:
      m1:
        payload:
          type: object
operations:
  onUserRegistered:
    action: receive
    channel:
      $ref: '#/channels/userRegistered'
`);

  const channel = document?.channels().get('userRegistered');
  console.log('Channel id:', channel?.id());
  
  // This won't work:
  try {
    console.log('Channel title:', (channel as any)?.title?.());
  } catch (e) {
    console.error('No title() method');
  }
  
  // Raw JSON confirms data is there:
  console.log('Raw title:', channel?.json()?.title); // "User Registration Channel"
}
main().catch(console.error);
```

### How to fix it

**In `packages/parser/src/models/channel.ts`:**
```typescript
export interface ChannelInterface extends BaseModel<...>,
  TitleMixinInterface,  // ← ADD
  ...
```

**In `packages/parser/src/models/v3/channel.ts`:**
```typescript
title(): string | undefined {
  return this._json.title;
}
hasTitle(): boolean {
  return !!this._json.title;
}
```

Also check `v2/channel.ts` — though AsyncAPI 2.x channels don't have `title` in the spec, you may want to return `undefined` for completeness.

---

<a name="1137"></a>
## Issue #1137 — Missing `title` and `summary` fields in v3 ServerObject spec-types

**Link:** https://github.com/asyncapi/parser-js/issues/1137  
**Labels:** bug  
**Difficulty:** Beginner

### What is the problem?

This is related to #1075 and #1076 but at the TypeScript type level. The `ServerObject` interface in `packages/parser/src/spec-types/v3.ts` is missing the `title` and `summary` fields.

This means even if you implement the model methods, TypeScript will show a type error when you try to access `this._json.title` because the type doesn't include it.

### Why it happens

The spec-types files are supposed to mirror the official AsyncAPI specification. Someone forgot to include `title` and `summary` when writing the v3 spec-types.

The [AsyncAPI 3.0 spec](https://www.asyncapi.com/docs/reference/specification/v3.0.0#serverObject) explicitly lists `title` and `summary` as valid fields on Server Object.

### How to reproduce

Look at the source file:

```bash
grep -A 20 "interface ServerObject" packages/parser/src/spec-types/v3.ts
```

You'll see something like:
```typescript
export interface ServerObject {
  host: string;
  protocol: string;
  protocolVersion?: string;
  pathname?: string;
  description?: string;
  // title and summary are MISSING here
  variables?: Record<string, ServerVariableObject>;
  security?: SecurityRequirementObject[];
  tags?: TagObject[];
  externalDocs?: ExternalDocumentationObject;
  bindings?: Record<string, unknown>;
}
```

### How to fix it

**In `packages/parser/src/spec-types/v3.ts`**, add `title` and `summary` to `ServerObject`:

```typescript
export interface ServerObject {
  host: string;
  protocol: string;
  protocolVersion?: string;
  pathname?: string;
  description?: string;
  title?: string;    // ← ADD
  summary?: string;  // ← ADD
  variables?: Record<string, ServerVariableObject>;
  security?: SecurityRequirementObject[];
  tags?: TagObject[];
  externalDocs?: ExternalDocumentationObject;
  bindings?: Record<string, unknown>;
}
```

> **All three issues #1075, #1076, and #1137 should be fixed together in one PR**: fix the spec-types, add the methods to the model interface, and implement them in v2 and v3 server classes.

---

<a name="875"></a>
## Issue #875 — No error when channel has `parameters:` but `address` is null

**Link:** https://github.com/asyncapi/parser-js/issues/875  
**Labels:** bug, good first issue  
**Difficulty:** Intermediate

### What is the problem?

In AsyncAPI v3, a channel can have `address: null` — this means the channel address is dynamic and will be determined at runtime. In that case, it makes no sense to also define URL parameters (like `{streetlightId}`), because those parameters only work when there's an actual address template.

The parser should reject this document but currently it doesn't.

### Why it happens

There is no Spectral rule that checks: "if `address` is null, the `parameters` field must also be absent (or empty)."

The v3 JSON Schema from `@asyncapi/specs` doesn't seem to catch this semantic constraint, and no custom Spectral rule covers it either.

### How to reproduce

```js
// scratch/reproduce-875.js
const { Parser } = require('./packages/parser/cjs/index.js');

async function main() {
  const parser = new Parser();
  
  // This document is INVALID but the parser accepts it (the bug)
  const { document, diagnostics } = await parser.parse(`
asyncapi: '3.0.0'
info:
  title: Account Service
  version: '1.0.0'
channels:
  userSignedup:
    address: user/signedup
    parameters:
      test:
        description: >
          I provide a parameter but there are no {} in the address!
          This should be an error but isn't.
    messages:
      UserSignedUp:
        payload:
          type: object
operations:
  sendUserSignedup:
    action: send
    channel:
      $ref: '#/channels/userSignedup'
    messages:
      - $ref: '#/channels/userSignedup/messages/UserSignedUp'
`);

  console.log('Document defined:', document !== undefined);
  // BUG: document IS defined — no error!
  
  const errors = diagnostics.filter(d => d.severity === 0);
  console.log('Errors:', errors.length);
  // BUG: 0 errors — it should have 1 error about unused parameters
}
main().catch(console.error);
```

### How to fix it

Add a new v3 Spectral rule. 

**Step 1: Create the rule function** — `packages/parser/src/ruleset/v3/functions/channelParametersWithAddress.ts`:

```typescript
import type { RuleFunction, IFunctionContext } from '@stoplight/spectral-core';

export const channelParametersWithAddress: RuleFunction = (
  channel: Record<string, unknown>,
  _options: unknown,
  context: IFunctionContext,
) => {
  const { address, parameters } = channel;
  
  // If address is null or undefined but parameters are defined → error
  if (
    (address === null || address === undefined) &&
    parameters !== undefined &&
    Object.keys(parameters as object).length > 0
  ) {
    return [{
      message: 'Channel must not have "parameters" when "address" is null or undefined.',
      path: ['parameters'],
    }];
  }
  
  // Also check: parameters defined but no {param} in the address
  if (
    typeof address === 'string' &&
    parameters !== undefined
  ) {
    const paramsInAddress = (address.match(/\{([^}]+)\}/g) ?? []).map(p => p.slice(1, -1));
    const definedParams = Object.keys(parameters as object);
    const unusedParams = definedParams.filter(p => !paramsInAddress.includes(p));
    
    if (unusedParams.length > 0) {
      return unusedParams.map(p => ({
        message: `Parameter "${p}" is defined but "{${p}}" does not appear in address "${address}".`,
        path: ['parameters', p],
      }));
    }
  }
};
```

**Step 2: Add the rule to v3 ruleset** — `packages/parser/src/ruleset/v3/ruleset.ts`:

```typescript
import { channelParametersWithAddress } from './functions/channelParametersWithAddress';

// In v3CoreRuleset.rules:
'asyncapi3-channel-parameters-with-address': {
  description: 'Channel parameters must correspond to address template parameters.',
  message: '{{error}}',
  severity: 'error',
  recommended: true,
  given: ['$.channels.*', '$.components.channels.*'],
  then: {
    function: channelParametersWithAddress,
  },
},
```

---

<a name="876"></a>
## Issue #876 — No error when `reply.address.location` is set but channel address is not null

**Link:** https://github.com/asyncapi/parser-js/issues/876  
**Labels:** bug, good first issue  
**Difficulty:** Intermediate

### What is the problem?

In AsyncAPI v3, when an operation has a `reply`, you can either:
1. Hardcode the reply address on the reply channel (`channel.address: "some/fixed/address"`)
2. Use a dynamic reply address via `reply.address.location` — which reads the reply address from a message header

The spec says: if you use `reply.address.location` (dynamic address), the reply **channel must have `address: null`**. This makes sense — if the address is dynamic (from the message), the channel shouldn't have a fixed address.

The parser doesn't enforce this constraint.

### How to reproduce

```js
// scratch/reproduce-876.js
const { Parser } = require('./packages/parser/cjs/index.js');

async function main() {
  const parser = new Parser();
  
  // BUG: This should be INVALID but the parser accepts it
  // The replyChannel has a non-null address, but we're using
  // reply.address.location (dynamic address) — contradiction!
  const { document, diagnostics } = await parser.parse(`
asyncapi: '3.0.0'
info:
  title: Account Service
  version: '1.0.0'
channels:
  replyChannel:
    address: user/signedup   # ← HAS an address
    messages:
      UserSignedUp:
        $ref: '#/components/messages/UserSignedUp'
  userSignedup:
    address: user/signedup
    messages:
      UserSignedUp:
        $ref: '#/components/messages/UserSignedUp'
operations:
  sendUserSignedup:
    action: send
    channel:
      $ref: '#/channels/userSignedup'
    messages:
      - $ref: '#/channels/userSignedup/messages/UserSignedUp'
    reply:
      channel: 
        $ref: '#/channels/replyChannel'
      address:
        location: '$message.header#/REPLY_TOPIC'
        # ↑ Dynamic address — but replyChannel has a fixed address! Bug!
components:
  messages:
    UserSignedUp:
      payload:
        type: object
`);

  const errors = diagnostics.filter(d => d.severity === 0);
  console.log('Errors:', errors.length); // 0 — should be 1!
  console.log('Document defined:', document !== undefined); // true — should be false!
}
main().catch(console.error);
```

### How to fix it

Add a v3 Spectral rule in `packages/parser/src/ruleset/v3/functions/operationReplyAddressLocation.ts`:

```typescript
import type { RuleFunction } from '@stoplight/spectral-core';

export const operationReplyAddress: RuleFunction = (
  operation: Record<string, unknown>,
  _options: unknown,
  context: IFunctionContext,
) => {
  const reply = operation?.reply as Record<string, unknown>;
  if (!reply) return;
  
  const replyAddress = reply?.address as Record<string, unknown>;
  const replyChannel = reply?.channel as Record<string, unknown>;
  
  // If reply has a dynamic address location...
  if (replyAddress?.location) {
    // ...then the reply channel MUST have address: null
    const channelAddress = replyChannel?.address;
    if (channelAddress !== null && channelAddress !== undefined) {
      return [{
        message: 'When "reply.address.location" is set, the reply channel must have "address: null".',
        path: ['reply', 'channel'],
      }];
    }
  }
};
```

> **Note:** This rule operates on resolved documents, where `$ref` to the channel is replaced with the actual channel object.

---

<a name="878"></a>
## Issue #878 — `parse()` returns `undefined` with no explanation for invalid doc

**Link:** https://github.com/asyncapi/parser-js/issues/878  
**Labels:** bug  
**Difficulty:** Intermediate

### What is the problem?

When you parse an invalid AsyncAPI document, `parse()` returns `{ document: undefined, diagnostics: [...] }`. The user reported that they expect an "error" (exception thrown) rather than a silent `undefined`.

The actual behavior is: `document` is `undefined` AND `diagnostics` has errors. But new users often forget to check `diagnostics` and only check `document`, which gives no information.

### How to reproduce

```js
// scratch/reproduce-878.js
const { Parser } = require('./packages/parser/cjs/index.js');

async function main() {
  const parser = new Parser();
  
  const { document, diagnostics } = await parser.parse(`
asyncapi: 3.0.0
test: hello
info:
  title: Account Service
  version: 1.0.0
channels:
  user/signedup:
    address: user/signedup
    messages:
      subscribe.message:
        $ref: '#/components/messages/UserSignedUp'
operations:
  user/signedup.subscribe:
    action: send
    channel:
      $ref: '#/channels/user~1signedup'
    messages:
      - $ref: '#/components/messages/UserSignedUp'
components:
  messages:
    UserSignedUp:
      payload:
        type: object
`);

  // New users often only check document:
  if (!document) {
    console.log('document is undefined'); // They see this and wonder why
    // But they miss that diagnostics has the answer!
  }
  
  // The REAL information is in diagnostics:
  const errors = diagnostics.filter(d => d.severity === 0);
  console.log('Errors:', errors.map(e => `[${e.code}] ${e.message}`));
  // → asyncapi-document-unresolved: Property test is not expected to be here
}
main().catch(console.error);
```

### Understanding the behavior

This is partly a design debate (should the parser throw vs return undefined) but the consensus is that the current design is intentional — it returns a structured result with diagnostics. The issue is really about **documentation and user experience**.

The fix could be:
1. Better error messaging in the README (not a code change)
2. Adding a helper function that throws if there are errors

### Possible code improvement

Add a helper that throws if errors are present:

```typescript
// Usage pattern:
const { document, diagnostics } = await parser.parse(asyncapiDoc);

// Helper (doesn't exist yet, would be a contribution):
function assertValidDocument(document, diagnostics) {
  const errors = diagnostics.filter(d => d.severity === 0);
  if (errors.length > 0) {
    const messages = errors.map(e => `[${e.code}] ${e.message}`).join('\n');
    throw new Error(`AsyncAPI document is invalid:\n${messages}`);
  }
  if (!document) {
    throw new Error('Document is undefined with no errors — unexpected state');
  }
  return document;
}
```

---

<a name="877"></a>
## Issue #877 — Parser returns `undefined` for v3 docs with message bindings

**Link:** https://github.com/asyncapi/parser-js/issues/877  
**Labels:** bug  
**Difficulty:** Intermediate

### What is the problem?

When an AsyncAPI v3 document has bindings on messages, the parser fails silently (returns `document: undefined`) instead of parsing correctly. The parser should either parse successfully or return proper error diagnostics.

### How to reproduce

```js
// scratch/reproduce-877.js
const { Parser } = require('./packages/parser/cjs/index.js');

async function main() {
  const parser = new Parser();
  
  // This uses the official adeo-kafka example that has message bindings
  const doc = `
asyncapi: '3.0.0'
info:
  title: Adeo AsyncAPI Case Study
  version: '0.1.0'
channels:
  costingRequestChannel:
    address: adeo-{env}-case-study-costing-request-v1
    messages:
      CostingRequest:
        $ref: '#/components/messages/CostingRequest'
components:
  messages:
    CostingRequest:
      bindings:
        kafka:
          key:
            type: string
            description: Message key
      payload:
        type: object
operations:
  sendCostingRequest:
    action: send
    channel:
      $ref: '#/channels/costingRequestChannel'
    messages:
      - $ref: '#/channels/costingRequestChannel/messages/CostingRequest'
`;

  const { document, diagnostics } = await parser.parse(doc);
  
  const errors = diagnostics.filter(d => d.severity === 0);
  console.log('Errors:', errors.map(e => `${e.code}: ${e.message}`));
  console.log('Document:', document ? 'DEFINED' : 'UNDEFINED');
}
main().catch(console.error);
```

### Why it happens

This might be caused by the Ajv JSON Schema validation rejecting the document because the v3 spec types for message bindings are not fully defined, causing the `asyncapi-document-resolved` rule to fail with a schema validation error.

Check if the error code is `asyncapi-document-resolved` or `asyncapi-document-unresolved`. If it is, the fix is in `@asyncapi/specs` (the JSON Schema definition), not in this repo.

If it's an uncaught exception, check `diagnostics[0].code === 'uncaught-error'`.

---

<a name="793"></a>
## Issue #793 — False "messageId must be unique" error when same message is reused

**Link:** https://github.com/asyncapi/parser-js/issues/793  
**Labels:** (none)  
**Difficulty:** Intermediate

### What is the problem?

In AsyncAPI v2, you can have both `publish` and `subscribe` on the same channel, both pointing to the **same** message (via `$ref`). This is a valid pattern — the same message format is used for both incoming and outgoing events.

But the `asyncapi2-message-messageId-uniqueness` rule incorrectly flags this as an error: "messageId must be unique across all messages".

This is wrong because it's literally the same message object (same `$ref`), just referenced twice.

### How to reproduce

```js
// scratch/reproduce-793.js
const { Parser } = require('./packages/parser/cjs/index.js');

async function main() {
  const parser = new Parser();
  
  const { document, diagnostics } = await parser.parse(`
asyncapi: '2.6.0'
info:
  title: Event Store
  version: '1.0.0'
channels:
  someChannel:
    publish:
      message:
        $ref: '#/components/messages/someMessage'
    subscribe:
      message:
        $ref: '#/components/messages/someMessage'
    bindings:
      eventstore:
        streamName: someStream
components:
  messages:
    someMessage:
      messageId: SomeEvent
      name: SomeEvent
      payload:
        type: object
        properties:
          data:
            type: string
`);

  const errors = diagnostics.filter(d => d.severity === 0);
  console.log('Errors:', errors.map(e => `${e.code}: ${e.message}`));
  // BUG: asyncapi2-message-messageId-uniqueness fires
  // But this is wrong — it's the SAME message, just referenced twice!
}
main().catch(console.error);
```

### Why it happens

The `messageIdUniqueness` function in `packages/parser/src/ruleset/v2/functions/messageIdUniqueness.ts` collects all message objects and checks if any `messageId` is duplicated. However, it doesn't account for the case where the same message object is referenced from multiple places via `$ref`.

After `$ref` resolution, both `publish.message` and `subscribe.message` point to the **same JavaScript object** (same reference). But the uniqueness check probably compares `messageId` values as strings, counting the same message twice.

### How to fix it

In `packages/parser/src/ruleset/v2/functions/messageIdUniqueness.ts`, add a check to skip duplicate **object references** (same JSON object pointer), not just duplicate IDs:

```typescript
// Simplified concept of the fix:
function messageIdUniqueness(document) {
  const seen = new Map(); // messageId → first path it was seen
  const seenObjects = new Set(); // track by object identity, not just value
  
  // Walk all messages
  for (const { message, path } of getAllMessages(document)) {
    if (!message.messageId) continue;
    
    // If we've seen this exact object before, skip it
    if (seenObjects.has(message)) continue;
    seenObjects.add(message);
    
    if (seen.has(message.messageId)) {
      // This is a TRUE duplicate (different objects, same messageId) → error
      yield { message: `"${message.messageId}" messageId must be unique`, path };
    } else {
      seen.set(message.messageId, path);
    }
  }
}
```

The key is: **track seen objects by reference** (`Set`), not just by value.

---

<a name="874"></a>
## Issue #874 — `SecurityRequirements.map()` throws a crash error

**Link:** https://github.com/asyncapi/parser-js/issues/874  
**Labels:** bug  
**Difficulty:** Intermediate

### What is the problem?

The `SecurityRequirements` collection class extends JavaScript's `Array`. When you call `.map()` on it, JavaScript creates a new `SecurityRequirements` instance internally (because `Array.map` uses `Symbol.species` to create the result). But `SecurityRequirements` constructor doesn't know how to handle this, causing a crash.

Specifically: `for...of` and `.forEach()` work. But `.map()`, `.filter()`, `.slice()`, and other array methods that return a **new array** crash.

### How to reproduce

```js
// scratch/reproduce-874.js
const { Parser } = require('./packages/parser/cjs/index.js');

async function main() {
  const parser = new Parser();
  const { document } = await parser.parse(`
asyncapi: '2.6.0'
info:
  title: Test
  version: '1.0.0'
servers:
  production:
    url: mqtt.example.com
    protocol: mqtt
    security:
      - apiKey: []
components:
  securitySchemes:
    apiKey:
      type: apiKey
      in: user
channels: {}
`);

  const server = document?.servers().get('production');
  const securityRequirements = server?.security()?.all()?.[0];
  
  if (!securityRequirements) {
    console.log('No security requirements found');
    return;
  }
  
  // Works fine:
  for (const req of securityRequirements) {
    console.log('for...of:', req.jsonPath());
  }
  
  // Works fine:
  securityRequirements.forEach(req => {
    console.log('forEach:', req.jsonPath());
  });
  
  // CRASHES:
  try {
    const paths = securityRequirements.map(req => req.jsonPath());
    console.log('map result:', paths);
  } catch (e) {
    console.error('map() crashed:', e.message);
    // TypeError: Spread syntax requires ...iterable[Symbol.iterator] to be a function
  }
}
main().catch(console.error);
```

### Why it happens

JavaScript's `Array` class uses `Symbol.species` to determine what type of array to create when methods like `.map()` are called. By default, `Array.map()` creates an instance of the subclass (here `SecurityRequirements`) using `new SecurityRequirements(result)`.

But `SecurityRequirements` constructor (inherited from `Collection`) expects `collections: T[]` as its first argument — not the spread result from `.map()`.

The issue is deep in how `Collection` extends `Array`:

```typescript
// packages/parser/src/models/collection.ts
export abstract class Collection<T extends BaseModel> extends Array<T> {
  constructor(
    protected readonly collections: T[],  // ← expects T[] as first arg
    protected readonly _meta = {}
  ) {
    super(...collections);  // spreads collections into Array
  }
}
```

When `.map()` internally tries to create `new SecurityRequirements(mappedResults)`, it doesn't match this constructor signature.

### How to fix it

Add a static getter for `Symbol.species` to return plain `Array`:

```typescript
// In packages/parser/src/models/collection.ts
export abstract class Collection<T extends BaseModel> extends Array<T> {
  // This tells Array methods to create a plain Array, not a Collection subclass
  static get [Symbol.species]() { return Array; }
  
  // ... rest of the class
}
```

This ensures `.map()`, `.filter()`, `.slice()` etc. return plain arrays instead of trying to construct new `Collection` instances.

---

<a name="1063"></a>
## Issue #1063 — Wrong `x-parser-schema-id` when schemas are in external files

**Link:** https://github.com/asyncapi/parser-js/issues/1063  
**Labels:** bug  
**Difficulty:** Intermediate

### What is the problem?

When your AsyncAPI document uses `$ref` to pull in schemas from external files, the parser assigns wrong (empty or meaningless) `x-parser-schema-id` values to those schemas. This causes code generators (like `@asyncapi/modelina`) to generate classes with names like `AnonymousSchema_1`, `AnonymousSchema_2` instead of the actual schema names.

### How to reproduce

Create two files:

**`scratch/multi-file-main.yaml`:**
```yaml
asyncapi: '2.6.0'
info:
  title: Multi-file Test
  version: '1.0.0'
channels:
  user/registered:
    publish:
      message:
        payload:
          $ref: './schemas.yaml#/UserPayload'
```

**`scratch/schemas.yaml`:**
```yaml
UserPayload:
  type: object
  properties:
    userId:
      type: string
    email:
      type: string
```

```js
// scratch/reproduce-1063.js
const { Parser } = require('./packages/parser/cjs/index.js');
const path = require('path');
const fs = require('fs');

async function main() {
  const parser = new Parser();
  const filePath = path.join(__dirname, 'scratch/multi-file-main.yaml');
  const content = fs.readFileSync(filePath, 'utf-8');
  
  const { document } = await parser.parse(content, { source: filePath });
  
  if (!document) return;
  
  // Print all schema IDs
  document.allSchemas().all().forEach(schema => {
    console.log('Schema ID:', schema.id(), '| type:', schema.type());
    // BUG: schema.id() returns '' or undefined for external schemas
    // Expected: 'UserPayload'
  });
}
main().catch(console.error);
```

### Why it happens

The `anonymous-naming.ts` custom operation assigns `x-parser-schema-id` based on the schema's JSON pointer path in the document. For schemas that come from external files via `$ref`, after resolution the pointer path information may be lost or incorrect — the external file path isn't reflected in the ID assignment.

The relevant code is in `packages/parser/src/custom-operations/anonymous-naming.ts`. The ID is likely computed from the JSON pointer, but external refs become their target path (e.g., the path inside `schemas.yaml`) not the calling document's path.

### Possible fix

When assigning `x-parser-schema-id`, if the schema came from an external file (detectable via `$ref` source tracking in `documentInventory`), use the component name from that external file rather than an auto-generated ID.

---

<a name="1099"></a>
## Issue #1099 — Custom schema parsers not triggered for payloads accessed via `operation.channel`

**Link:** https://github.com/asyncapi/parser-js/issues/1099  
**Labels:** bug  
**Difficulty:** Advanced

### What is the problem?

This is a v3-specific bug. When an operation references a channel that's defined in an **external file** (via `$ref`), and that channel's messages have custom schema formats (Avro, Protobuf, etc.), the custom schema parser never gets called. The payloads remain in their original format without conversion.

### Why it happens

After `$ref` resolution, Spectral inlines external file content. An operation that references `otherFile.yaml#/channels/BarChannel` ends up with the channel merged at path:

```
$.operations.FooOperation.channel.messages.AMessage.payload
```

But `parseSchemasV3` in `packages/parser/src/custom-operations/parse-schema.ts` only walks these paths:

```
$.channels.*.messages.*.payload
$.components.messages.*.payload
$.operations.*.messages.*.payload
```

Notice what's missing: `$.operations.*.channel.messages.*.payload`

The path where the externally-referenced channel's messages end up is NOT searched.

### How to reproduce

Create two files:

**`scratch/main-v3.yaml`:**
```yaml
asyncapi: '3.0.0'
info:
  title: Test
  version: '1.0.0'
operations: 
  FooOperation:
    action: send
    channel:
       $ref: 'other.yaml#/channels/BarChannel'
```

**`scratch/other.yaml`:**
```yaml
asyncapi: '3.0.0'
info:
  title: Other
  version: '1.0.0'
channels:
  BarChannel:
    address: foo/bar
    messages:
      AMessage:
        payload:
          schemaFormat: 'application/vnd.apache.avro+json;version=1.9.0'
          schema:
            type: record
            name: AMessage
            fields:
              - name: id
                type: string
```

```js
// scratch/reproduce-1099.js
const { Parser } = require('./packages/parser/cjs/index.js');
const { AvroSchemaParser } = require('./packages/parser/node_modules/@asyncapi/avro-schema-parser');
const path = require('path');
const fs = require('fs');

async function main() {
  const parser = new Parser({ schemaParsers: [AvroSchemaParser()] });
  const filePath = path.join(__dirname, 'scratch/main-v3.yaml');
  const content = fs.readFileSync(filePath, 'utf-8');
  
  const { document, diagnostics } = await parser.parse(content, { source: filePath });
  
  const errors = diagnostics.filter(d => d.severity === 0);
  console.log('Errors:', errors.map(e => `${e.code}: ${e.message}`));
  
  // The message payload should be converted from Avro to JSON Schema
  const op = document?.operations().get('FooOperation');
  const msg = op?.messages().all()[0];
  const payload = msg?.payload();
  
  // BUG: payload is still the Avro schema, not converted JSON Schema
  console.log('Payload type:', payload?.type()); // Might be undefined instead of 'object'
  console.log('Original payload:', msg?.json()?.['x-parser-original-payload']); // Should be set
}
main().catch(console.error);
```

### How to fix it

In `packages/parser/src/custom-operations/parse-schema.ts`, add the missing path to `customSchemasPathsV3`:

```typescript
const customSchemasPathsV3 = [
  // ... existing paths ...
  '$.channels.*.messages.*.payload',
  '$.operations.*.messages.*.payload',
  '$.components.messages.*.payload',
  // ADD THIS:
  '$.operations.*.channel.messages.*.payload',
];
```

---

<a name="924"></a>
## Issue #924 — v3 rules don't catch invalid cross-file external references

**Link:** https://github.com/asyncapi/parser-js/issues/924  
**Labels:** bug, keep-open  
**Difficulty:** Advanced

### What is the problem?

Two v3 Spectral rules check that:
- `asyncapi3-required-operation-channel-unambiguity`: operation's channel must point to root `#/channels/`
- `asyncapi3-required-channel-servers-unambiguity`: channel's servers must point to root `#/servers/`

These rules work by checking the **text of the `$ref` string** — does it contain `#/channels/`?

But what if an external file (`b.yaml`) has an invalid cross-reference (channel server pointing to `#/components/servers/`), and the main file (`a.yaml`) just includes `b.yaml`? The external file reference passes as `b.yaml#/channels/test` — which doesn't contain the invalid pattern. The cross-validation never happens.

### The specific scenario

```yaml
# a.yaml — SHOULD be invalid (references an invalid external channel)
asyncapi: 3.0.0
info:
  title: FileA
  version: 1.0.0
channels:
  test:
    $ref: './b.yaml#/channels/test'
```

```yaml
# b.yaml — this IS invalid on its own
asyncapi: 3.0.0
info:
  title: FileB
  version: 1.0.0
channels:
  test: 
    servers:
      - $ref: '#/components/servers/serverA'  # ← violates unambiguity rule!
components:
  servers:
    serverA:
      host: localhost
      protocol: http
```

`b.yaml` alone fails validation. But `a.yaml` passes — even though it includes the invalid channel.

### Why it happens

The two rules use `resolved: false` — they run against the **raw** document with `$ref` strings intact. At that stage, they pattern-match against `$ref` strings. When the ref is to an external file (`./b.yaml#/channels/test`), the pattern `#/channels/` doesn't even appear in the string, so the rule never fires.

After resolution, the invalid `#/components/servers/serverA` becomes the actual server object (no `$ref` string left), so pattern matching can't work either.

### The challenge

The issue comment says: "the only way to achieve this is to make the rules pass when the document is unresolved." This means the rules need to be refactored to work on the resolved document using a different approach — maybe checking the resolved structure rather than `$ref` string patterns.

This is a non-trivial architectural change and explains why it's labeled `keep-open`.

---

<a name="1098"></a>
## Issue #1098 — No way to disable `$ref` dereferencing

**Link:** https://github.com/asyncapi/parser-js/issues/1098  
**Labels:** bug, stale  
**Difficulty:** Advanced

### What is the problem?

When `parser.validate()` is called on a document with external HTTP/HTTPS `$ref`s, the parser automatically makes network requests to fetch and dereference those references. There's no built-in way to disable this.

Some users need to validate documents in air-gapped environments (no internet) or in security-sensitive contexts where outbound network calls must be blocked.

### What users tried (didn't work)

```js
// These approaches were tried but don't work:
parser.validate(spec, {
  resolve: { http: false, https: false, file: false }
  // → This option doesn't exist
});

parser.validate(spec, {
  resolve: { external: false }
  // → Also not a real option
});
```

### Why there's no easy fix

Spectral's resolver is configured at `Parser` construction time, not per-validate call. The `__unstable.resolver` option exists per parse/validate call, but it requires providing your own complete resolver implementation — there's no simple `{ allowNetwork: false }` flag.

### Potential fix approaches

**Option A:** Add a `__unstable.resolver.disableExternalRefs` boolean:

```typescript
// Conceptual API addition to validate.ts / parse.ts:
const parser = new Parser({
  __unstable: {
    resolver: {
      disableExternalRefs: true,  // block all HTTP/file resolvers
    }
  }
});
```

**Option B:** Allow passing `null` resolvers per-call via `__unstable`:

```typescript
const result = await parser.validate(doc, {
  __unstable: {
    resolver: {
      resolvers: {
        http: null,   // disables http resolver
        https: null,  // disables https resolver
        file: null,   // disables file resolver
      }
    }
  }
});
```

**Workaround today:**

Users can provide a resolver that throws immediately:

```typescript
const parser = new Parser({
  __unstable: {
    resolver: {
      resolvers: {
        http: {
          resolve: () => { throw new Error('HTTP resolution not allowed'); }
        },
        https: {
          resolve: () => { throw new Error('HTTPS resolution not allowed'); }
        },
      }
    }
  }
});
```

---

<a name="403"></a>
## Issue #403 — JSON Schema `$id`-based references not resolved correctly

**Link:** https://github.com/asyncapi/parser-js/issues/403  
**Labels:** bug, keep-open  
**Difficulty:** Expert

### What is the problem?

JSON Schema Draft-07 allows schemas to define a base URI using `$id`. Other schemas in the same document can then use `$ref` relative to that base URI. This is called `$id`-based dereferencing.

The parser uses `@stoplight/json-ref-resolver` to resolve references, but this library does not implement `$id`-based resolution (it treats `$id` as just a regular field, not as a URI base).

So the following is broken:

```yaml
payload:
  $id: 'http://localhost.com/'   # ← sets the base URI for this schema
  type: object
  properties:
    sentAt:
      $ref: "/components/schemas/sentAt"
      # ↑ This should resolve to http://localhost.com/components/schemas/sentAt
      # But the parser tries http:///components/schemas/sentAt — wrong!
```

### Why it's hard to fix

This requires either:
1. Switching to a different resolver that supports `$id` resolution (like `@apidevtools/json-schema-ref-parser` — which is ironically what the parser used to use before switching to Spectral's resolver)
2. Implementing `$id` resolution on top of the current Spectral resolver

This is labeled `keep-open` because the fix would be a significant change to the resolver infrastructure.

---

<a name="761"></a>
## Issue #761 — Parser uses a deprecated `$ref` resolver library

**Link:** https://github.com/asyncapi/parser-js/issues/761  
**Labels:** enhancement, help wanted, stale  
**Difficulty:** Expert

### What is the problem?

The parser currently uses `@stoplight/json-ref-resolver` (a Stoplight-maintained library) to resolve `$ref` references. Stoplight has deprecated this library and recommends migrating to `@apidevtools/json-schema-ref-parser`.

Interestingly, the old v1 parser used `@apidevtools/json-schema-ref-parser`. The v2 rewrite switched to Spectral's resolver — which uses the now-deprecated `@stoplight/json-ref-resolver` under the hood.

### Why this matters

- The deprecated library may stop receiving security fixes
- Issue #403 (`$id` resolution) might be fixable by switching back to `@apidevtools/json-schema-ref-parser`
- The deprecated warning in library code is confusing for contributors

### Challenges

Switching resolver libraries would be a significant refactor because:
1. The resolver is tightly integrated with how Spectral runs
2. Spectral's `runWithResolved()` expects a Spectral-compatible resolver
3. Any switch would need extensive testing across all `$ref` resolution scenarios (local, remote, circular)

---

<a name="1065"></a>
## Issue #1065 — `@asyncapi/multi-parser` depends on vulnerable `jsonpath-plus` version

**Link:** https://github.com/asyncapi/parser-js/issues/1065  
**Labels:** bug  
**Difficulty:** Expert

### What is the problem?

`jsonpath-plus` versions before `10.0.7` have a known security vulnerability. The main `@asyncapi/parser` was already patched to use `jsonpath-plus@>=10.0.7`.

But `@asyncapi/multi-parser` bundles two older parser versions as npm aliases:
- `parserapiv1` = `@asyncapi/parser@^2.1.0` (uses `jsonpath-plus@7.2.0`)
- `parserapiv2` = `@asyncapi/parser@3.0.0-next-major-spec.8` (uses `jsonpath-plus@7.2.0`)

These old versions are pinned and still use the vulnerable `jsonpath-plus@7.2.0`.

### How to verify

```bash
# Install multi-parser in a test project
mkdir test-multiparser && cd test-multiparser
npm init -y
npm install @asyncapi/multi-parser

# Check what versions of jsonpath-plus got installed
npm why jsonpath-plus
```

You'll see two entries for `jsonpath-plus@7.2.0` from `parserapiv1` and `parserapiv2`.

### Why it's hard to fix

To fix this, the maintainers need to:
1. Release patch versions of `@asyncapi/parser@2.1.x` and `@asyncapi/parser@3.0.0-next-major-spec.x` that use `jsonpath-plus@>=10.0.7`
2. Update `@asyncapi/multi-parser` to point to these patched versions

This requires careful testing since upgrading `jsonpath-plus` from v7 to v10 had breaking API changes that needed adaptation in the main parser.

---

<a name="1131"></a>
## Issue #1131 — Browser bundle doesn't work in GraalVM (Java)

**Link:** https://github.com/asyncapi/parser-js/issues/1131  
**Labels:** (none)  
**Difficulty:** Expert

### What is the problem?

A Java developer wants to use `@asyncapi/parser` inside a Java application via GraalVM's JavaScript engine. GraalVM runs JavaScript without Node.js and without browser APIs like `fetch`, `Headers`, etc.

The parser's browser bundle (webpack) assumes either a browser environment (with `fetch`) or Node.js. GraalVM is neither.

### Current failure

When bundling `@asyncapi/parser` for GraalVM:
```
TypeError: Cannot set property 'Headers' of undefined
```

The webpack config uses `browserify-shim` to replace `node-fetch` with the browser's global `fetch`. In GraalVM, neither Node's `fetch` nor the browser's `fetch` exists.

### This is an enhancement request, not a bug

The library was never designed for GraalVM. The requester is asking for:
1. A bundle that works without any fetch API
2. Or documentation on how to create such a bundle
3. Or polyfill guidance for GraalVM environments

### Potential approach

The requester would need to:
1. Provide a `fetch` polyfill for GraalVM (wrap Java's `HttpClient` in a JavaScript shim)
2. Create a custom webpack build that uses their polyfill
3. Or use the `__unstable.resolver` option to provide custom resolvers that don't use `fetch`

---

<a name="1167"></a>
## Issue #1167 — Microgrant Program 2026-06 (Aggregated)

**Link:** https://github.com/asyncapi/parser-js/issues/1167  
**Labels:** microgrant  

### What this is

This is an administrative issue that groups multiple bugs under the AsyncAPI Microgrant Program 2026-06. Contributors can earn a microgrant (financial reward) by fixing the grouped issues:

- **#877** — Parser returns `undefined` when bindings are in message (v3)
- **#1137** — Missing `title` and `summary` in v3 ServerObject spec-types
- **#1067** — Missing `channel.title()` function

If you want to contribute and potentially earn a microgrant, these are priority issues. Check the linked issue for program details.

---

<a name="1181"></a>
## Issue #1181 — Microgrant Program 2026-07 (Aggregated)

**Link:** https://github.com/asyncapi/parser-js/issues/1181

### What this is

Same as above, for the July 2026 Microgrant Program cycle. Groups:

- **#875** — No error when channel has `parameters` but `address` is null
- **#1076** — Missing `server.summary()` function
- **#1075** — Missing `server.title()` function

---

## Summary: Where to Start Contributing

If you're new and want to fix your first issue, start here in order:

| Issue | Why it's good to start | Expected effort |
|-------|----------------------|----------------|
| #1075 | Clear spec, clear fix, one model file | 1–2 hours |
| #1076 | Same pattern as #1075, fix both together | 30 mins extra |
| #1067 | Same pattern, channel instead of server | 1 hour |
| #1137 | One-line spec-types fix, needed for #1075/#1076 | 30 mins |
| #874 | One-line `Symbol.species` fix | 1 hour |
| #875 | Write a new rule — good learning exercise | 2–4 hours |
| #876 | Write a new rule — slightly more complex | 3–5 hours |

All of #1075, #1076, #1067, and #1137 are naturally grouped and can be submitted as a single PR.
