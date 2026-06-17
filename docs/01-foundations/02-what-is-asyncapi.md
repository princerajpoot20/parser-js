# What is AsyncAPI?

> **Goal:** Understand the AsyncAPI specification, its purpose, and how spec 2.x differs from 3.x.

---

## The Problem AsyncAPI Solves

After reading about event-driven architecture, you understand that producers and consumers are decoupled — the producer does not know who is listening. That decoupling is a feature, but it creates a documentation and tooling problem:

- **How do consumers know what messages a producer sends?**
- **What is the shape of the payload?**
- **Which broker URL do I connect to?**
- **What authentication is needed?**

Without a standard answer to these questions, every team writes documentation in a different format (or not at all). Code generation, validation tools, and documentation generators all need to reinvent the wheel.

**AsyncAPI is the answer:** a specification that defines a standard, machine-readable format for describing event-driven APIs.

---

## The OpenAPI Analogy

The fastest way to understand AsyncAPI:

> AsyncAPI is to event-driven APIs what OpenAPI (Swagger) is to REST APIs.

| Concept | OpenAPI (REST) | AsyncAPI (Event-Driven) |
|---------|---------------|------------------------|
| Describes | HTTP endpoints | Channels / topics |
| Operations | GET, POST, PUT, DELETE | publish, subscribe (v2) / send, receive (v3) |
| Data shape | JSON Schema for request/response bodies | JSON Schema (or Avro, Protobuf, etc.) for message payloads |
| Servers | Base URLs | Broker URLs + protocol |
| Standard | OpenAPI 3.x | AsyncAPI 2.x / 3.x |
| Tools | Swagger UI, code generators | AsyncAPI Studio, code generators |

Both specifications produce a document (YAML or JSON) that:
1. Machines can read to generate code, documentation, and validation
2. Humans can read to understand the API contract

---

## A Minimal AsyncAPI Document

Here is the simplest valid AsyncAPI 2.x document:

```yaml
asyncapi: '2.6.0'

info:
  title: Streetlights API
  version: '1.0.0'
  description: A sample API for the smart streetlights system.

channels:
  smartylighting/streetlights/1/0/event/lighting/measured:
    publish:
      message:
        payload:
          type: object
          properties:
            lumens:
              type: integer
              description: Light intensity measured in lumens.
            sentAt:
              type: string
              format: date-time
```

This document says:
- The API is described by AsyncAPI version 2.6.0
- It has a channel named `smartylighting/streetlights/1/0/event/lighting/measured`
- Something **publishes** to that channel (i.e., sends messages to it)
- The message payload is a JSON object with `lumens` and `sentAt` fields

---

## Anatomy of an AsyncAPI Document

### `asyncapi` (required)
The version of the AsyncAPI specification this document conforms to. Currently `2.x.x` or `3.0.0`.

### `info` (required)
Human-readable metadata: `title`, `version`, `description`, `contact`, `license`.

### `servers` (optional)
The broker(s) the API connects to. Each server has a URL, a protocol, and optionally security requirements:

```yaml
servers:
  production:
    url: mqtt://broker.example.com:1883
    protocol: mqtt
    description: Production MQTT broker
  development:
    url: mqtt://localhost:1883
    protocol: mqtt
```

### `channels` (v2 — required; v3 — required)
In **v2**: channels are addressed by their channel name (the topic path). Each channel has `publish` and/or `subscribe` operations. The terminology is from the application's perspective — if the application *publishes*, it sends; if it *subscribes*, it receives.

In **v3**: channels are just channel definitions. Operations are top-level and reference channels (see [v2 vs v3](#asyncapi-2x-vs-30) below).

### `components` (optional)
Reusable definitions: messages, schemas, servers, parameters, security schemes, etc. Components are referenced using `$ref`.

### `operations` (v3 only)
Top-level list of all operations (send/receive). Each operation references a channel.

---

## `$ref` — The Building Block

AsyncAPI documents can use JSON Reference (`$ref`) to reuse definitions:

```yaml
channels:
  user/registered:
    publish:
      message:
        $ref: '#/components/messages/UserRegistered'

components:
  messages:
    UserRegistered:
      payload:
        type: object
        properties:
          userId:
            type: string
```

`$ref: '#/components/messages/UserRegistered'` means "replace this with the definition at that path in the same document." References can also point to external files or URLs:

```yaml
$ref: './messages/user-registered.yaml'
$ref: 'https://schemas.example.com/user-registered.json'
```

**This is why the parser needs a `$ref` resolver.** Before the document can be modeled or validated, all `$ref`s must be followed and replaced. That process is called **dereferencing** or **resolution**.

---

## Schema Formats

By default, AsyncAPI uses **JSON Schema Draft-07** to describe message payloads. But many teams already use other schema languages — Avro, Protobuf, RAML Data Types, OpenAPI Schema Objects.

AsyncAPI supports these via the `schemaFormat` field:

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

When `schemaFormat` is present, a parser needs a **schema parser plugin** to validate and convert the payload. The `@asyncapi/parser` provides a plugin interface for this (see [06-schema-parser-system.md](../03-architecture/06-schema-parser-system.md)).

---

## AsyncAPI 2.x vs 3.0

AsyncAPI 3.0 was a significant redesign. Here are the key differences:

### Channel vs Operation separation

**v2:** Operations (`publish` / `subscribe`) are defined *inside* channels. A channel can have at most one `publish` and one `subscribe`.

```yaml
# AsyncAPI 2.x
channels:
  user/registered:
    publish:
      operationId: onUserRegistered
      message:
        $ref: '#/components/messages/UserRegistered'
```

**v3:** Operations are top-level and *reference* channels. Multiple operations can reference the same channel.

```yaml
# AsyncAPI 3.x
channels:
  userRegistered:
    address: user/registered
    messages:
      UserRegistered:
        $ref: '#/components/messages/UserRegistered'

operations:
  onUserRegistered:
    action: receive
    channel:
      $ref: '#/channels/userRegistered'
```

### Terminology

| Concept | v2 | v3 |
|---------|----|----|
| Application sends a message | `publish` | `send` |
| Application receives a message | `subscribe` | `receive` |
| Channel address | Channel name (map key) | `address` field inside channel |

The v2 `publish`/`subscribe` terminology caused confusion — "does `publish` mean the application publishes, or the broker publishes?". v3 uses unambiguous `send`/`receive` from the application's point of view.

### Replies

v3 adds first-class support for request/reply patterns via `reply` and `replyAddress` on operations. v2 had no standard way to model this.

### Components

v3 adds `operations` to the `components` section (reusable operations). v2 only had messages, schemas, servers, parameters, etc.

---

## Supported Versions in parser-js

The `@asyncapi/parser` supports AsyncAPI versions **2.0.0 through 2.6.0** and **3.0.0**. Version 1.x is not supported (the [asyncapi/converter-js](https://github.com/asyncapi/converter-js) project can upgrade 1.x docs).

---

## Next Step

Read [03-asyncapi-spec-walkthrough.md](./03-asyncapi-spec-walkthrough.md) for fully annotated real-world AsyncAPI documents that you can use to test the parser locally.
