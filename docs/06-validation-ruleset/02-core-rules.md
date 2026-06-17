# Core Rules

> **Goal:** Complete reference for all core and recommended rules — what they check, their severity, and when they trigger.

---

## Core Rules (Always Active)

These rules run for all AsyncAPI versions (2.x and 3.x).

### `asyncapi-is-asyncapi`

| Property | Value |
|----------|-------|
| Severity | Error |
| Formats | All (even non-AsyncAPI docs) |
| Given | `$` (root) |
| Function | `isAsyncAPIDocument` |

Checks that:
1. The `asyncapi` field is present
2. Its value is a supported version (2.0.0–2.6.0, 3.0.0)
3. The document structure is recognizable as AsyncAPI

This is the first gate — if it fails, no other rules run.

```typescript
// isAsyncAPIDocument function (simplified)
function isAsyncAPIDocument(input, options, context) {
  if (!input?.asyncapi) {
    return [{ message: 'Not an AsyncAPI document' }];
  }
  if (!specVersions.includes(input.asyncapi)) {
    return [{ message: `Unsupported AsyncAPI version "${input.asyncapi}"` }];
  }
}
```

---

### `asyncapi-latest-version`

| Property | Value |
|----------|-------|
| Severity | Info |
| Given | `$.asyncapi` |
| Function | `schema` (built-in: check `const: lastVersion`) |

Informs users when they are not using the latest supported AsyncAPI version. The `lastVersion` constant is derived from the last key in `@asyncapi/specs`.

This is deliberately `info` level — it never blocks parsing.

---

### `asyncapi-document-resolved`

| Property | Value |
|----------|-------|
| Severity | Error |
| Given | `$` (root) |
| Function | `documentStructure` |
| Options | `{ resolved: true }` |

Validates the **resolved** (post-`$ref`) document against the official AsyncAPI JSON Schema for that version using Ajv.

"Resolved" means all `$ref` pointers have been replaced with the referenced content. Validates the merged document.

---

### `asyncapi-document-unresolved`

| Property | Value |
|----------|-------|
| Severity | Error |
| Given | `$` (root) |
| Function | `documentStructure` |
| Options | `{ resolved: false }` |
| `resolved` flag | `false` (runs against raw document) |

Validates the **unresolved** (raw, with `$ref` strings intact) document against the official AsyncAPI JSON Schema.

Why run both? The unresolved rule catches errors in how `$ref` strings are structured (e.g., wrong path). The resolved rule catches errors in the merged content.

---

### `asyncapi-internal`

| Property | Value |
|----------|-------|
| Severity | None (informational) |
| Given | `$` (root) |
| Function | `internal` |

This is an internal plumbing rule. It stores Spectral's `documentInventory` in the raw Document object so `parse.ts` can access it for circular reference resolution.

It never produces diagnostics. Never disable this rule.

---

## `documentStructure` Function Deep Dive

**File:** `packages/parser/src/ruleset/functions/documentStructure.ts`

This is the most important validation function — it's what gives you errors like "Object must have required property 'title'".

```typescript
export function documentStructure(input, { resolved }, context) {
  const document = context.document;
  const version = document.data?.asyncapi;
  
  if (!version) return; // asyncapi-is-asyncapi handles this
  
  // Get the JSON Schema for this version from @asyncapi/specs
  const schema = resolved 
    ? getResolvedSchema(version) 
    : getUnresolvedSchema(version);
  
  const ajvInstance = getAjvInstance();
  const validate = ajvInstance.compile(schema);
  
  const valid = validate(input);
  if (!valid && validate.errors) {
    return validate.errors
      .filter(err => !shouldIgnoreError(err))
      .map(err => ({
        message: err.message,
        path: err.instancePath.split('/').filter(Boolean),
      }));
  }
}
```

### Why some Ajv errors are filtered

```typescript
function shouldIgnoreError(error: ErrorObject): boolean {
  return (
    error.keyword === 'oneOf' ||
    (error.keyword === 'required' && error.params.missingProperty === '$ref')
  );
}
```

- `oneOf` errors: When a value fails a `oneOf` (exactly one schema must match), Ajv generates one error per branch of the `oneOf` plus one for the `oneOf` itself. Only the branch errors are useful. The aggregate `oneOf` error is suppressed.
- `$ref` required: The AsyncAPI spec uses `$ref` internally as part of schema definitions, but `$ref` is not something users provide as a required field. These false positives are suppressed.

---

## Recommended Rules

Recommended rules are active by default but can be disabled:

```typescript
const parser = new Parser({
  ruleset: {
    recommended: false  // disable all recommended rules
  }
});
```

### Root Object Rules

| Rule | Severity | What it checks |
|------|---------|---------------|
| `asyncapi-id` | Warning | Document has `id` field (URN identifying the API) |
| `asyncapi-defaultContentType` | Warning | Document has `defaultContentType` field |

### Info Object Rules

| Rule | Severity | What it checks |
|------|---------|---------------|
| `asyncapi-info-description` | Warning | `info.description` is present and non-empty |
| `asyncapi-info-contact` | Warning | `info.contact` object is present |
| `asyncapi-info-contact-properties` | Warning | Contact has `name`, `url`, and `email` |
| `asyncapi-info-license` | Warning | `info.license` object is present |
| `asyncapi-info-license-url` | Warning (not recommended) | License has `url` field |

### Server Rules

| Rule | Severity | What it checks |
|------|---------|---------------|
| `asyncapi-servers` | Warning | Document has non-empty `servers` object |

### Component Rules

| Rule | Severity | What it checks |
|------|---------|---------------|
| `asyncapi-unused-component` | Info | A component is defined but never `$ref`'d (v2 only; v3 WIP) |

The `asyncapi-unused-component` function walks the document looking for each component's JSON Pointer path in all `$ref` values. If a component's path never appears in a `$ref`, it produces an info diagnostic.

---

## Disabling Individual Rules

To disable specific rules while keeping everything else:

```typescript
const parser = new Parser({
  ruleset: {
    extends: [
      {
        rules: {
          'asyncapi-servers': 'off',           // disable this rule entirely
          'asyncapi-id': 'hint',               // downgrade from warning to hint
          'asyncapi-info-contact': 'warn',     // keep as warning (default)
        }
      }
    ]
  }
});
```

---

## Recommended Rule Severity Defaults

Rules tagged `recommended: false` in the ruleset (like `asyncapi-info-license-url`) run but at a lower severity by default. Spectral's `severity: 'recommended'` setting in `createRuleset()` handles this mapping.

---

## Next Step

Read [03-v2-rules.md](./03-v2-rules.md) for all 25+ AsyncAPI 2.x-specific validation rules.
