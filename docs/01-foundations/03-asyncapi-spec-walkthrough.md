# AsyncAPI Spec Walkthrough

> **Goal:** Walk through two fully annotated real-world AsyncAPI documents (one v2, one v3) that you will use throughout this documentation to test the parser locally.

---

## Part 1: AsyncAPI 2.6.0 — Streetlights

This is the official AsyncAPI example used in the specification docs. Save this as `streetlights-v2.yaml` and use it with the examples in [Chapter 4](../04-use-cases-and-examples/).

```yaml
asyncapi: '2.6.0'
# ↑ Required. Tells tools (and the parser) which spec version to use.
#   The parser uses this to select the correct JSON Schema validator and model class.

info:
  title: Streetlights Kafka API
  version: '1.0.0'
  description: |
    The Smartylighting Streetlights API allows you to remotely manage the city lights.
  license:
    name: Apache 2.0
    url: 'https://www.apache.org/licenses/LICENSE-2.0'
  contact:
    name: Smartylighting Support
    email: support@streetlights.example.com
# ↑ info is required. title and version are required fields within it.

defaultContentType: application/json
# ↑ If a message doesn't specify its contentType, assume JSON.
#   The parser marks this as a warning if missing (asyncapi-defaultContentType rule).

servers:
  production:
    url: test.mosquitto.org
    protocol: mqtt
    description: Test MQTT broker
    variables:
      port:
        description: Secure connection (TLS) is available through port 8883
        default: '1883'
        enum:
          - '1883'
          - '8883'
    security:
      - apiKey: []
# ↑ servers describe where the API can be reached.
#   Each server has a url, protocol, and optional security.
#   The parser validates that security schemes referenced here exist in components.

channels:
  smartylighting/streetlights/1/0/event/{streetlightId}/lighting/measured:
  # ↑ The channel address. {streetlightId} is a URL template parameter.
    description: The topic on which measured values may be produced and consumed.
    parameters:
      streetlightId:
        $ref: '#/components/parameters/streetlightId'
    # ↑ $ref points to a reusable parameter definition in components.
    #   The parser resolves this reference before validation.
    publish:
    # ↑ "publish" means: the application SENDS messages to this channel.
    #   In v2, the perspective is from the server/broker side.
    #   This caused confusion — v3 renamed it to "send" from the app's perspective.
      summary: Inform about environmental lighting conditions of a particular streetlight.
      operationId: receiveLightMeasurement
      # ↑ operationId must be unique across all operations in the document.
      #   The parser validates this with asyncapi2-operation-operationId rule.
      traits:
        - $ref: '#/components/operationTraits/kafka'
      # ↑ Traits are merged into the operation during the parse() custom operation
      #   phase (apply-traits.ts). After parsing, the operation has all trait fields.
      message:
        $ref: '#/components/messages/LightMeasured'

  smartylighting/streetlights/1/0/action/{streetlightId}/turn/on:
    parameters:
      streetlightId:
        $ref: '#/components/parameters/streetlightId'
    subscribe:
    # ↑ "subscribe" means: the application RECEIVES messages from this channel.
      operationId: turnOn
      traits:
        - $ref: '#/components/operationTraits/kafka'
      message:
        $ref: '#/components/messages/TurnOnOff'

  smartylighting/streetlights/1/0/action/{streetlightId}/turn/off:
    parameters:
      streetlightId:
        $ref: '#/components/parameters/streetlightId'
    subscribe:
      operationId: turnOff
      traits:
        - $ref: '#/components/operationTraits/kafka'
      message:
        $ref: '#/components/messages/TurnOnOff'

  smartylighting/streetlights/1/0/action/{streetlightId}/dim:
    parameters:
      streetlightId:
        $ref: '#/components/parameters/streetlightId'
    subscribe:
      operationId: dimLight
      traits:
        - $ref: '#/components/operationTraits/kafka'
      message:
        $ref: '#/components/messages/DimLight'

components:
# ↑ components holds reusable definitions. Nothing in components is "active"
#   until referenced from channels/operations.

  messages:
    LightMeasured:
      name: LightMeasured
      title: Light measured
      summary: Inform about environmental lighting conditions of a particular streetlight.
      contentType: application/json
      traits:
        - $ref: '#/components/messageTraits/commonHeaders'
      payload:
        $ref: '#/schemas/LightMeasuredPayload'
      # ↑ The payload schema. By default, this is JSON Schema Draft-07.
      #   If schemaFormat were set here, a custom schema parser would be invoked.

    TurnOnOff:
      name: TurnOnOff
      title: Turn on/off
      summary: Command a particular streetlight to turn the lights on or off.
      traits:
        - $ref: '#/components/messageTraits/commonHeaders'
      payload:
        $ref: '#/schemas/TurnOnOffPayload'

    DimLight:
      name: DimLight
      title: Dim light
      summary: Command a particular streetlight to dim the lights.
      traits:
        - $ref: '#/components/messageTraits/commonHeaders'
      payload:
        $ref: '#/schemas/DimLightPayload'

  schemas:
    LightMeasuredPayload:
      type: object
      properties:
        lumens:
          type: integer
          minimum: 0
          description: Light intensity measured in lumens.
        sentAt:
          $ref: '#/components/schemas/SentAt'
    TurnOnOffPayload:
      type: object
      properties:
        command:
          type: string
          enum:
            - 'on'
            - 'off'
          description: Whether to turn on or off the light.
        sentAt:
          $ref: '#/components/schemas/SentAt'
    DimLightPayload:
      type: object
      properties:
        percentage:
          type: integer
          description: Percentage to which the light should be dimmed to.
          minimum: 0
          maximum: 100
        sentAt:
          $ref: '#/components/schemas/SentAt'
    SentAt:
      type: string
      format: date-time
      description: Date and time when the message was sent.

  securitySchemes:
    apiKey:
      type: apiKey
      in: user
      description: Provide your API key as the user and leave the password empty.
  # ↑ The parser validates that any security scheme referenced in servers or
  #   operations actually exists here (asyncapi2-server-security rule).

  parameters:
    streetlightId:
      description: The ID of the streetlight.
      schema:
        type: string

  messageTraits:
    commonHeaders:
      headers:
        type: object
        properties:
          my-app-header:
            type: integer
            minimum: 0
            maximum: 100
  # ↑ Message traits are merged into messages by apply-traits.ts.
  #   After parsing: every message that uses this trait will have `headers`.

  operationTraits:
    kafka:
      bindings:
        kafka:
          clientId:
            type: string
            enum:
              - my-app-id
  # ↑ Operation traits are merged into operations by apply-traits.ts.
```

### What the parser produces from this document

When you call `parser.parse(streetlightsYaml)`:

1. **All `$ref`s are resolved** — `#/components/messages/LightMeasured` becomes the full message object inline.
2. **Traits are merged** — `commonHeaders` trait is merged into each message; `kafka` trait is merged into each operation.
3. **Unique IDs are assigned** — channels, operations, messages get `x-parser-unique-object-id` values.
4. **A v2 document model is created** — `AsyncAPIDocumentV2` with full typed accessors.

---

## Part 2: AsyncAPI 3.0.0 — Streetlights

The same system described in v3. Notice how operations become top-level and channels are simpler:

```yaml
asyncapi: '3.0.0'

info:
  title: Streetlights Kafka API
  version: '1.0.0'
  description: The Smartylighting Streetlights API.
  license:
    name: Apache 2.0
    url: 'https://www.apache.org/licenses/LICENSE-2.0'

defaultContentType: application/json

servers:
  scram-connections:
    host: test.mosquitto.org:{port}
    protocol: mqtt
    description: Test MQTT broker secured with SCRAM
    variables:
      port:
        description: Secure connection (TLS) is available through port 8883.
        default: '1883'
        enum:
          - '1883'
          - '8883'
    security:
      - $ref: '#/components/securitySchemes/saslScram'

channels:
  lightingMeasured:
  # ↑ In v3, the channel KEY is the channel ID (not the address).
  #   The address is a field inside the channel.
    address: 'smartylighting/streetlights/1/0/event/{streetlightId}/lighting/measured'
    messages:
      lightMeasured:
        $ref: '#/components/messages/LightMeasured'
    description: The topic on which measured values may be produced and consumed.
    parameters:
      streetlightId:
        $ref: '#/components/parameters/streetlightId'

  lightTurnOn:
    address: 'smartylighting/streetlights/1/0/action/{streetlightId}/turn/on'
    messages:
      turnOnOff:
        $ref: '#/components/messages/TurnOnOff'
    parameters:
      streetlightId:
        $ref: '#/components/parameters/streetlightId'

  lightTurnOff:
    address: 'smartylighting/streetlights/1/0/action/{streetlightId}/turn/off'
    messages:
      turnOnOff:
        $ref: '#/components/messages/TurnOnOff'
    parameters:
      streetlightId:
        $ref: '#/components/parameters/streetlightId'

  lightsDim:
    address: 'smartylighting/streetlights/1/0/action/{streetlightId}/dim'
    messages:
      dimLight:
        $ref: '#/components/messages/DimLight'
    parameters:
      streetlightId:
        $ref: '#/components/parameters/streetlightId'

operations:
# ↑ In v3, ALL operations are defined here at the top level.
#   Each operation says what the APPLICATION does (send or receive).

  receiveLightMeasurement:
    action: receive
    # ↑ The APPLICATION receives from this channel (i.e., it is a consumer here).
    #   "receive" is unambiguous — v2's "subscribe" confused many people.
    channel:
      $ref: '#/channels/lightingMeasured'
    summary: Inform about environmental lighting conditions of a particular streetlight.
    traits:
      - $ref: '#/components/operationTraits/kafka'
    messages:
      - $ref: '#/channels/lightingMeasured/messages/lightMeasured'

  turnOn:
    action: send
    # ↑ The APPLICATION sends to this channel (it is a producer here).
    channel:
      $ref: '#/channels/lightTurnOn'
    traits:
      - $ref: '#/components/operationTraits/kafka'
    messages:
      - $ref: '#/channels/lightTurnOn/messages/turnOnOff'

  turnOff:
    action: send
    channel:
      $ref: '#/channels/lightTurnOff'
    traits:
      - $ref: '#/components/operationTraits/kafka'
    messages:
      - $ref: '#/channels/lightTurnOff/messages/turnOnOff'

  dimLight:
    action: send
    channel:
      $ref: '#/channels/lightsDim'
    traits:
      - $ref: '#/components/operationTraits/kafka'
    messages:
      - $ref: '#/channels/lightsDim/messages/dimLight'

components:
  messages:
    LightMeasured:
      name: LightMeasured
      title: Light measured
      summary: Inform about environmental lighting conditions of a particular streetlight.
      contentType: application/json
      traits:
        - $ref: '#/components/messageTraits/commonHeaders'
      payload:
        $ref: '#/components/schemas/LightMeasuredPayload'

    TurnOnOff:
      name: TurnOnOff
      title: Turn on/off
      summary: Command a particular streetlight to turn the lights on or off.
      traits:
        - $ref: '#/components/messageTraits/commonHeaders'
      payload:
        $ref: '#/components/schemas/TurnOnOffPayload'

    DimLight:
      name: DimLight
      title: Dim light
      summary: Command a particular streetlight to dim the lights.
      traits:
        - $ref: '#/components/messageTraits/commonHeaders'
      payload:
        $ref: '#/components/schemas/DimLightPayload'

  schemas:
    LightMeasuredPayload:
      type: object
      properties:
        lumens:
          type: integer
          minimum: 0
          description: Light intensity measured in lumens.
        sentAt:
          $ref: '#/components/schemas/SentAt'
    TurnOnOffPayload:
      type: object
      properties:
        command:
          type: string
          enum: ['on', 'off']
        sentAt:
          $ref: '#/components/schemas/SentAt'
    DimLightPayload:
      type: object
      properties:
        percentage:
          type: integer
          minimum: 0
          maximum: 100
        sentAt:
          $ref: '#/components/schemas/SentAt'
    SentAt:
      type: string
      format: date-time
      description: Date and time when the message was sent.

  securitySchemes:
    saslScram:
      type: scramSha256
      description: Provide your username and password for SASL/SCRAM authentication.

  parameters:
    streetlightId:
      description: The ID of the streetlight.

  messageTraits:
    commonHeaders:
      headers:
        type: object
        properties:
          my-app-header:
            type: integer
            minimum: 0
            maximum: 100

  operationTraits:
    kafka:
      bindings:
        kafka:
          clientId:
            type: string
            enum:
              - my-app-id
```

---

## Key `$ref` Rules

| Pattern | Meaning |
|---------|---------|
| `#/components/messages/Foo` | Same-document reference to `components.messages.Foo` |
| `./other.yaml` | External file reference (relative path) |
| `./other.yaml#/components/messages/Foo` | External file, specific path within it |
| `https://example.com/schema.json` | Remote HTTP reference |

When the parser sees a `$ref`:
1. It fetches the target (file system or HTTP)
2. It substitutes the referenced value inline
3. It tracks the original pointer for diagnostics

You must pass `source` option (the file path or URL of the root document) when parsing documents that use external file refs, so the parser knows the base path for resolving relative references.

---

## The `schemaFormat` Field

Without `schemaFormat`, the parser assumes JSON Schema Draft-07 for all payloads:

```yaml
# Implicit JSON Schema
message:
  payload:
    type: object
    properties:
      userId: { type: string }
```

With `schemaFormat`, you declare the format explicitly, and a registered schema parser plugin handles validation and conversion:

```yaml
# Explicit Avro schema
message:
  schemaFormat: 'application/vnd.apache.avro+json;version=1.9.0'
  payload:
    type: record
    name: UserRegistered
    fields:
      - name: userId
        type: string
```

The MIME type string in `schemaFormat` is used by the parser to look up the right plugin in its registry. See [06-custom-schema-formats.md](../04-use-cases-and-examples/06-custom-schema-formats.md) for a working example.

---

## Next Step

Read [04-where-parser-js-fits.md](./04-where-parser-js-fits.md) to understand the role of `@asyncapi/parser` in the broader AsyncAPI tooling ecosystem.
