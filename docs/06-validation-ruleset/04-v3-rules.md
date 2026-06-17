# AsyncAPI 3.x Rules

> **Goal:** Complete reference for all v3-specific validation rules, their unique constraints that reflect the v3 spec redesign, and the JSONPath patterns they use.

---

## V3 Rule Philosophy

AsyncAPI 3.0 introduced a new structural model where:
- Operations are top-level (not inside channels)
- Operations reference channels (not channels containing operations)
- A channel can be referenced from components OR directly from root channels
- This means cross-referencing must be unambiguous

Most v3 rules enforce **unambiguity** — making sure references point to the right place.

> **Note:** The v3 recommended ruleset is currently work-in-progress (WIP) per comments in `src/ruleset/ruleset.ts`. Many v2 recommended rules have not yet been ported to v3. This is a contribution opportunity.

---

## All V3 Core Rules

All v3 rules are in `v3CoreRuleset` (always active for v3 documents).

### 1. `asyncapi3-operation-messages-from-referred-channel`

| Property | Value |
|----------|-------|
| Severity | Error |
| `resolved` | `false` (uses JSON pointers) |
| Given | `$.operations.*`, `$.components.operations.*` |

**What it checks:** Every message listed in `operations.*.messages` must be a message that exists on the channel referenced by that operation.

**Why:** In v3, each operation references a channel AND specifies which messages from that channel it handles. An operation cannot "claim" a message from a different channel.

```yaml
# INVALID: operation claims a message that's not on its channel
channels:
  channelA:
    messages:
      MessageA:
        payload: { type: object }
  channelB:
    messages:
      MessageB:
        payload: { type: string }

operations:
  myOp:
    action: receive
    channel:
      $ref: '#/channels/channelA'   # references channelA
    messages:
      - $ref: '#/channels/channelB/messages/MessageB'  # ← Error! MessageB is on channelB, not channelA
```

**Implementation:** `operationMessagesUnambiguity` function extracts the `$ref` path of the channel and compares the prefix of each message `$ref` to ensure they come from the same channel.

---

### 2. `asyncapi3-required-operation-channel-unambiguity`

| Property | Value |
|----------|-------|
| Severity | Error |
| `resolved` | `false` |
| Given | `$.operations.*` |
| Field | `channel.$ref` |
| Function | `pattern` (must match `#/channels/`) |

**What it checks:** The `channel` field of a top-level operation must reference a channel defined in the root `channels` object (not in `components.channels`).

```yaml
# VALID
operations:
  myOp:
    channel:
      $ref: '#/channels/userRegistered'  # ← root channels

# INVALID: references a component channel from an operation
operations:
  myOp:
    channel:
      $ref: '#/components/channels/someChannel'  # ← must be from root channels
```

**Why:** Operations in the root `operations` object must be concrete (unambiguous). They cannot reference "template" channels from components directly — components are meant for reuse patterns.

---

### 3. `asyncapi3-required-channel-servers-unambiguity`

| Property | Value |
|----------|-------|
| Severity | Error |
| `resolved` | `false` |
| Given | `$.channels.*` |
| Field | `$.servers.*.$ref` |
| Function | `pattern` (must match `#/servers/`) |

**What it checks:** If a root channel specifies `servers:`, those server references must point to the root `servers` object (not `components.servers`).

```yaml
# INVALID: channel server references a component server
channels:
  userRegistered:
    servers:
      - $ref: '#/components/servers/myServer'  # ← must be from root servers

# VALID
channels:
  userRegistered:
    servers:
      - $ref: '#/servers/production'  # ← root servers
```

---

### 4. `asyncapi3-channel-servers`

| Property | Value |
|----------|-------|
| Severity | Error |
| Given | `$` (root) |
| Function | `channelServers` (shared with v2) |

**What it checks:** If a channel specifies `servers:`, all referenced servers must exist in the root `servers` object.

This is the v3 equivalent of `asyncapi2-channel-servers`. The shared `channelServers` function handles both v2 and v3.

```yaml
# INVALID
servers:
  production:
    host: mqtt.example.com
    protocol: mqtt

channels:
  userRegistered:
    servers:
      - $ref: '#/servers/production'   # valid
      - $ref: '#/servers/staging'      # invalid — staging doesn't exist
```

---

### 5. `asyncapi3-channel-no-query-nor-fragment`

| Property | Value |
|----------|-------|
| Severity | Error |
| Given | `$.channels` (the channels map) |
| Field | `@key` (the channel ID/key) |
| Function | `pattern` (not match `[?#]`) |

**What it checks:** Channel IDs (the map keys in `channels:`) must not contain `?` or `#` characters.

In v3, the channel key is the **ID** (not the address), but this rule still prevents potentially confusing IDs.

```yaml
# INVALID channel IDs
channels:
  'user?registered':    # ← contains '?'
  '#userUpdated':       # ← contains '#'
```

---

## V3 Rules Summary Table

| Rule | Severity | What triggers it |
|------|---------|-----------------|
| `asyncapi3-operation-messages-from-referred-channel` | Error | Operation message `$ref` doesn't match operation's channel |
| `asyncapi3-required-operation-channel-unambiguity` | Error | Operation's `channel.$ref` doesn't point to root channels |
| `asyncapi3-required-channel-servers-unambiguity` | Error | Channel's server `$ref` doesn't point to root servers |
| `asyncapi3-channel-servers` | Error | Channel references a server not in root servers |
| `asyncapi3-channel-no-query-nor-fragment` | Error | Channel ID contains `?` or `#` |

---

## Running V3 Rule Tests

```bash
# All v3 rule tests
npx jest "test/ruleset/rules/v3" --rootdir packages/parser

# Specific rules
npx jest "asyncapi3-operation-messages-from-referred-channel" --rootdir packages/parser
npx jest "asyncapi3-required-operation-channel-unambiguity" --rootdir packages/parser
```

---

## What's Missing / WIP in V3 Rules

The comment in `src/ruleset/ruleset.ts`:
```typescript
// Recommended validation for AsyncAPI v3 is still WIP.
formats: AsyncAPIFormats.filterByMajorVersions(['2']).formats()
```

This means the recommended ruleset (server URL patterns, operation IDs, message IDs, etc.) only runs for v2. V3 equivalents of these rules need to be written. This is a contribution opportunity — see [08-contributor-guide/03-adding-new-validation-rules.md](../08-contributor-guide/03-adding-new-validation-rules.md).

---

## Next Step

Read [Chapter 7: Multi-Parser](../07-multi-parser/01-multi-parser-guide.md) for information on `@asyncapi/multi-parser`.
