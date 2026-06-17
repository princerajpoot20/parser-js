# AsyncAPI 2.x Rules

> **Goal:** Complete reference for all v2-specific validation rules — what each rule checks, its severity, and the JSONPath it operates on.

---

## Rule Sets for v2

The v2 rules are split into three rulesets (all in `src/ruleset/v2/ruleset.ts`):

| Ruleset | Contents |
|---------|----------|
| `v2CoreRuleset` | Always-on structural rules (mostly errors) |
| `v2SchemasRuleset` | Schema validation rules (errors) |
| `v2RecommendedRuleset` | Optional best-practice rules (warnings/info) |

---

## Core Rules (`v2CoreRuleset`)

### Server Rules

| Rule | Severity | What it checks |
|------|---------|---------------|
| `asyncapi2-server-security` | Error | Every security scheme referenced in `servers.*.security` must exist in `components.securitySchemes` |
| `asyncapi2-server-variables` | Error | All `{variables}` in server URL are defined in `variables:`, and no extra variables are defined that aren't in the URL |

**Deep dive: `asyncapi2-server-variables`**

```yaml
# INVALID: {port} is in the URL but not in variables
servers:
  production:
    url: mqtt.example.com:{port}
    protocol: mqtt
    variables:
      host: { default: 'mqtt.example.com' }  # 'host' isn't in the URL!
```

The `serverVariables` function:
1. Extracts all `{variable}` placeholders from the server URL
2. Compares with the defined `variables:` keys
3. Reports missing variables (in URL but not defined) as errors
4. Reports redundant variables (defined but not in URL) as warnings

---

### Channel Rules

| Rule | Severity | What it checks |
|------|---------|---------------|
| `asyncapi2-channel-parameters` | Error | All `{parameters}` in the channel address are defined in `parameters:`, no extra defined |
| `asyncapi2-channel-servers` | Error | If channel specifies `servers:`, all must exist in the document's `servers:` object |
| `asyncapi2-channel-no-query-nor-fragment` | Error | Channel address must not contain `?` or `#` (query/fragment delimiters) |

**Deep dive: `asyncapi2-channel-parameters`**

```yaml
# INVALID: {streetlightId} in address but not in parameters
channels:
  'streetlights/{streetlightId}/status':
    publish:
      message:
        payload: { type: object }
    # Missing: parameters: { streetlightId: ... }
```

**Deep dive: `asyncapi2-channel-servers`**

```yaml
# INVALID: 'edge' server doesn't exist in servers:
servers:
  cloud:
    url: mqtt.example.com
    protocol: mqtt
channels:
  sensor/data:
    servers: [cloud, edge]   # 'edge' doesn't exist
```

---

### Operation Rules

| Rule | Severity | What it checks |
|------|---------|---------------|
| `asyncapi2-operation-operationId-uniqueness` | Error | All `operationId` values are unique across every operation in the document |
| `asyncapi2-operation-security` | Error | Security schemes referenced in operations must exist in `components.securitySchemes` |

**Deep dive: `asyncapi2-operation-operationId-uniqueness`**

```yaml
# INVALID: same operationId used twice
channels:
  user/registered:
    publish:
      operationId: onEvent  # ← duplicate!
  order/placed:
    publish:
      operationId: onEvent  # ← duplicate!
```

The `operationIdUniqueness` function collects all `operationId` values from:
- `$.channels.*.[publish,subscribe].operationId`
- `$.components.channels.*.[publish,subscribe].operationId`

And reports duplicates.

---

### Message Rules

| Rule | Severity | What it checks |
|------|---------|---------------|
| `asyncapi2-message-examples` | Error | Message examples must validate against the `payload` and `headers` schemas |
| `asyncapi2-message-messageId-uniqueness` | Error | All `messageId` values are unique across all messages |
| `asyncapi2-tags-uniqueness` | Error | Tag names are unique within each tags array (operations, messages, traits, etc.) |

---

## Schema Rules (`v2SchemasRuleset`)

| Rule | Severity | What it checks |
|------|---------|---------------|
| `asyncapi2-schemas` | Error | Payload schemas are valid according to the registered schema parsers |
| `asyncapi2-schema-default` | Error | `default` values in schemas must validate against the schema itself |
| `asyncapi2-schema-examples` | Error | `examples` in schemas must validate against the schema |
| `asyncapi2-message-examples-custom-format` | Error | Message examples validate for custom schema formats too |

**Deep dive: `asyncapi2-schema-default`**

```yaml
# INVALID: default value doesn't match the schema
schemas:
  Port:
    type: integer
    minimum: 1024
    maximum: 65535
    default: "8080"   # string, not integer — invalid!
```

The `schemaValidation` function validates the `default` field against the parent schema using Ajv.

**Deep dive: `asyncapi2-schemas`**

This is the Spectral rule that invokes the pluggable schema parser system. For each message payload found in the document, it:
1. Determines the `schemaFormat` (explicit or default JSON Schema)
2. Looks up the registered `SchemaParser` in the parser registry
3. Calls `schemaParser.validate(input)`
4. Converts results to Spectral diagnostics

---

## Recommended Rules (`v2RecommendedRuleset`)

### Root Rules

| Rule | Severity | What it checks |
|------|---------|---------------|
| `asyncapi2-tags` | Warning | Document root should have a non-empty `tags` array |

### Server Rules

| Rule | Severity | What it checks |
|------|---------|---------------|
| `asyncapi2-server-no-empty-variable` | Warning | Server URL should not have empty `{}` variable patterns |
| `asyncapi2-server-no-trailing-slash` | Warning | Server URL should not end with `/` |

### Channel Rules

| Rule | Severity | What it checks |
|------|---------|---------------|
| `asyncapi2-channel-no-empty-parameter` | Warning | Channel address should not have empty `{}` parameter patterns |
| `asyncapi2-channel-no-trailing-slash` | Warning | Channel address should not end with `/` |

### Operation Rules

| Rule | Severity | What it checks |
|------|---------|---------------|
| `asyncapi2-operation-operationId` | Warning | Every operation should have an `operationId` |

### Message Rules

| Rule | Severity | What it checks |
|------|---------|---------------|
| `asyncapi2-message-messageId` | Warning | Every message should have a `messageId` (available from AsyncAPI 2.4.0+) |

### Component Rules

| Rule | Severity | What it checks |
|------|---------|---------------|
| `asyncapi2-unused-securityScheme` | Info | A security scheme is defined in components but never referenced |

---

## Common Error Scenarios and Their Rules

| Mistake | Rule that catches it |
|---------|---------------------|
| Forgot to add `operationId` | `asyncapi2-operation-operationId` (warning) |
| Two operations with same `operationId` | `asyncapi2-operation-operationId-uniqueness` (error) |
| Channel address contains `?foo=bar` | `asyncapi2-channel-no-query-nor-fragment` (error) |
| Channel URL template `{id}` has no `parameters:` entry | `asyncapi2-channel-parameters` (error) |
| Server URL has `{port}` but no `variables.port:` | `asyncapi2-server-variables` (error) |
| `schemaFormat: 'application/vnd.apache.avro...'` but no Avro parser registered | `asyncapi2-schemas` (error) |
| Message `default` fails the schema | `asyncapi2-schema-default` (error) |
| Two messages with same `messageId` | `asyncapi2-message-messageId-uniqueness` (error) |
| Payload example doesn't match schema | `asyncapi2-message-examples` (error) |

---

## Running Specific v2 Rule Tests

```bash
# All v2 rule tests
npx jest "test/ruleset/rules/v2" --rootdir packages/parser

# A specific rule
npx jest "asyncapi2-operation-operationId" --rootdir packages/parser
npx jest "asyncapi2-channel-parameters" --rootdir packages/parser
npx jest "asyncapi2-server-variables" --rootdir packages/parser
```

---

## Next Step

Read [04-v3-rules.md](./04-v3-rules.md) for the AsyncAPI 3.x-specific validation rules.
