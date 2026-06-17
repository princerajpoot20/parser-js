# Spectral Integration

> **Goal:** Understand how Spectral rules are structured and wired in this project, how to write custom rules, and how to test them using the `testRule()` helper.

---

## Spectral Rule Anatomy

Every Spectral rule is a JavaScript object with these fields:

```typescript
const myRule = {
  // Human-readable description
  description: 'Something must be true.',
  
  // The error message template (supports {{error}} for function-returned messages)
  message: '{{error}}',
  
  // How severe is this? 'error' | 'warn' | 'info' | 'hint'
  severity: 'error',
  
  // Is this rule in the recommended set?
  recommended: true,
  
  // Which document formats does this apply to?
  // (use AsyncAPIFormats.formats(), or filterByMajorVersions, etc.)
  formats: [asyncapi2],
  
  // JSONPath expression selecting the node(s) to validate
  given: '$.channels.*',
  
  // What to do with each selected node
  then: {
    // Optional: access a specific field of the selected node
    field: 'publish',
    
    // The validation function (built-in or custom)
    function: myCustomFunction,
    
    // Options passed to the function
    functionOptions: {
      someOption: true,
    },
  },
  
  // If true, runs against the UNRESOLVED document (with $ref strings)
  // If false/absent, runs against the RESOLVED document
  resolved: false,
};
```

---

## Rule Functions

Rule functions have this signature:

```typescript
type RuleFunction = (
  input: unknown,
  options: Record<string, unknown>,
  context: IFunctionContext,
) => IFunctionResult[] | void | Promise<IFunctionResult[] | void>;
```

- `input` — the node matched by `given` + `then.field`
- `options` — the `functionOptions` from the rule
- `context` — provides `context.path`, `context.document`, `context.documentInventory`

Return an empty array or `void` for a passing check, or `IFunctionResult[]` for violations:

```typescript
interface IFunctionResult {
  message: string;
  path?: string[];  // relative path from the given node to the problematic field
}
```

---

## Built-in Spectral Functions

From `@stoplight/spectral-functions`:

| Function | What it checks | Usage |
|----------|---------------|-------|
| `truthy` | Value is truthy (non-null, non-empty) | `then: { field: 'title', function: truthy }` |
| `schema` | Value matches a JSON Schema | `then: { function: schema, functionOptions: { schema: { type: 'string' } } }` |
| `pattern` | String matches/doesn't match a regex | `then: { function: pattern, functionOptions: { match: '^[a-z]' } }` |
| `length` | Array/string length within bounds | `then: { function: length, functionOptions: { min: 1 } }` |
| `enumeration` | Value is in a list | `then: { function: enumeration, functionOptions: { values: ['a', 'b'] } }` |
| `alphabetical` | Array items are alphabetically sorted | |
| `xor` | Exactly one of two fields present | |
| `unreferencedReusableObject` | Object not `$ref`'d anywhere | |

---

## How `createRuleset()` Builds the Combined Ruleset

**File:** `packages/parser/src/ruleset/index.ts`

```typescript
export function createRuleset(parser: Parser, options?: RulesetOptions) {
  // Always include core rules
  let ruleset = { extends: [coreRuleset] };
  
  // Include recommended rules by default
  if (options?.recommended !== false) {
    ruleset.extends.push(recommendedRuleset);
  }
  
  // Include v2 and v3 rulesets
  ruleset.extends.push(v2CoreRuleset, v2SchemasRuleset(parser), v2RecommendedRuleset);
  ruleset.extends.push(v3CoreRuleset);
  
  // Add user-provided custom rules
  if (options?.extends) {
    ruleset.extends.push(...options.extends);
  }
  
  return new Ruleset(ruleset, { severity: 'recommended' });
}
```

---

## The `testRule()` Helper

**File:** `packages/parser/test/ruleset/tester.ts` (in tests)

This is the standard way to test a single Spectral rule in isolation. It creates a parser with only the specified rule enabled:

```typescript
// Example from test/ruleset/rules/v2/asyncapi2-channel-servers.spec.ts
import { testRule, DiagnosticSeverity } from '../../tester';

describe('asyncapi2-channel-servers', function() {
  const { document, diagnostics } = await testRule(
    'asyncapi2-channel-servers',
    {
      asyncapi: '2.6.0',
      info: { title: 'Test', version: '1.0' },
      servers: {
        production: { url: 'localhost', protocol: 'mqtt' }
      },
      channels: {
        'user/registered': {
          servers: ['nonexistent'],  // ← This server doesn't exist
          publish: { message: { payload: {} } }
        }
      }
    }
  );
  
  expect(diagnostics).toHaveLength(1);
  expect(diagnostics[0].severity).toBe(DiagnosticSeverity.Error);
  expect(diagnostics[0].message).toContain('nonexistent');
});
```

The `testRule(ruleName, document)` function:
1. Creates a `Parser` with all rules disabled
2. Enables only the named rule
3. Runs `parse()` on the document
4. Returns `{ document, diagnostics }`

---

## Finding Custom Rule Functions

All custom function files are in:

```
packages/parser/src/ruleset/functions/
├── channelServers.ts         ← Shared: used by v2 and v3
├── documentStructure.ts      ← Core: JSON Schema validation via Ajv
├── internal.ts               ← Core: stores documentInventory
├── isAsyncAPIDocument.ts     ← Core: version check
├── uniquenessTags.ts         ← Shared: tag uniqueness
└── unusedComponent.ts        ← Recommended: unused component check

packages/parser/src/ruleset/v2/functions/
├── channelParameters.ts
├── checkId.ts
├── messageExamples.ts
├── messageExamples-spectral-rule-v2.ts
├── messageIdUniqueness.ts
├── operationIdUniqueness.ts
├── schemaValidation.ts
├── security.ts
├── serverVariables.ts
└── unusedSecuritySchemes.ts

packages/parser/src/ruleset/v3/functions/
└── operationMessagesUnambiguity.ts
```

---

## Writing a Custom Rule: Example

Here is how you would write a new rule that warns when a channel address starts with a slash (a common mistake):

**Step 1: The rule function** (`src/ruleset/v2/functions/channelNoLeadingSlash.ts`):

```typescript
import type { IFunctionContext } from '@stoplight/spectral-core';

export function channelNoLeadingSlash(
  channels: Record<string, unknown>,
  _options: unknown,
  context: IFunctionContext,
): ReturnType<IFunctionContext['path']['join']> {
  const results = [];
  for (const channelAddress of Object.keys(channels)) {
    if (channelAddress.startsWith('/')) {
      results.push({
        message: `Channel address "${channelAddress}" should not start with "/".`,
        path: [channelAddress],
      });
    }
  }
  return results;
}
```

**Step 2: Add to the ruleset** (`src/ruleset/v2/ruleset.ts`):

```typescript
import { channelNoLeadingSlash } from './functions/channelNoLeadingSlash';

// In v2RecommendedRuleset.rules:
'asyncapi2-channel-no-leading-slash': {
  description: 'Channel address should not start with a leading slash.',
  severity: 'warn',
  recommended: true,
  given: '$.channels',
  then: {
    function: channelNoLeadingSlash,
  },
},
```

**Step 3: Write the test**:

```typescript
// test/ruleset/rules/v2/asyncapi2-channel-no-leading-slash.spec.ts
import { testRule, DiagnosticSeverity } from '../../tester';

describe('asyncapi2-channel-no-leading-slash', function() {
  it('passes for normal channel address', async () => {
    const { diagnostics } = await testRule('asyncapi2-channel-no-leading-slash', {
      asyncapi: '2.6.0',
      info: { title: 'Test', version: '1.0.0' },
      channels: { 'user/registered': { publish: { message: { payload: {} } } } }
    });
    expect(diagnostics).toHaveLength(0);
  });

  it('fails for channel with leading slash', async () => {
    const { diagnostics } = await testRule('asyncapi2-channel-no-leading-slash', {
      asyncapi: '2.6.0',
      info: { title: 'Test', version: '1.0.0' },
      channels: { '/user/registered': { publish: { message: { payload: {} } } } }
    });
    expect(diagnostics).toHaveLength(1);
    expect(diagnostics[0].severity).toBe(DiagnosticSeverity.Warning);
  });
});
```

---

## Running Rule Tests

```bash
# Run all ruleset tests
npx jest "test/ruleset" --rootdir packages/parser

# Run tests for a specific rule
npx jest "asyncapi2-channel-servers" --rootdir packages/parser

# Run your new rule test
npx jest "asyncapi2-channel-no-leading-slash" --rootdir packages/parser
```

---

## Next Step

Read [02-core-rules.md](./02-core-rules.md) for the complete table of core rules and a deep dive into `documentStructure`.
