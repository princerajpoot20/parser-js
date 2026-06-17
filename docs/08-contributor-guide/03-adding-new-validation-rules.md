# Adding New Validation Rules

> **Goal:** Step-by-step guide to writing, testing, and integrating a new Spectral validation rule for `@asyncapi/parser`.

---

## Before You Start

Ask yourself:
1. Is this rule about AsyncAPI 2.x structure → add to `v2CoreRuleset` or `v2RecommendedRuleset`
2. Is this rule about AsyncAPI 3.x structure → add to `v3CoreRuleset`
3. Is this rule about both versions → add to `coreRuleset` or `recommendedRuleset` with appropriate `formats`

Check if the rule already exists:
```bash
# Search existing rule names
grep -r "asyncapi2-" packages/parser/src/ruleset/ --include="*.ts" | grep "description:"
grep -r "asyncapi3-" packages/parser/src/ruleset/ --include="*.ts" | grep "description:"
```

---

## Step 1: Write the Rule Function

Create a file in the appropriate functions directory.

**Example:** A new rule that warns when a server URL uses `localhost` (probably not production-ready):

**File:** `packages/parser/src/ruleset/v2/functions/serverNotLocalhost.ts`

```typescript
import type { IFunctionContext, RuleFunction } from '@stoplight/spectral-core';

// The function receives the node matched by the rule's `given` path
export const serverNotLocalhost: RuleFunction = (
  serverUrl: string,
  _options: unknown,
  _context: IFunctionContext,
) => {
  if (typeof serverUrl !== 'string') return;
  
  // Check if URL contains localhost or 127.0.0.1
  const localhostPattern = /^(localhost|127\.0\.0\.1|0\.0\.0\.0)(:|$|\s)/i;
  
  if (localhostPattern.test(serverUrl)) {
    return [{
      message: `Server URL "${serverUrl}" appears to use localhost. This is likely a development configuration.`,
    }];
  }
  
  // Return undefined or empty array for no violations
};
```

### Function Signature Details

```typescript
type RuleFunction = (
  input: unknown,         // the node matched by given+field
  options: Record<string, unknown>,  // functionOptions from the rule
  context: IFunctionContext,  // gives access to path, document, etc.
) => IFunctionResult[] | void | Promise<IFunctionResult[] | void>;

interface IFunctionResult {
  message: string;
  path?: string[];  // path from the matched node to the problematic sub-node
}
```

**Using `context.path`:** The full JSON path to the current node:
```typescript
export const myFunction: RuleFunction = (input, _options, context) => {
  const fullPath = context.path;  // ['servers', 'production', 'url']
  // ...
};
```

**Returning sub-paths:** When the violation is on a child field:
```typescript
return [{
  message: 'Something is wrong with the url',
  path: ['url'],  // adds 'url' to the context path
}];
```

---

## Step 2: Add the Rule to the Ruleset

**File:** `packages/parser/src/ruleset/v2/ruleset.ts`

In `v2RecommendedRuleset.rules` (since this is a best-practice, not a structural error):

```typescript
import { serverNotLocalhost } from './functions/serverNotLocalhost';

// In v2RecommendedRuleset:
'asyncapi2-server-not-localhost': {
  description: 'Server URL should not use localhost.',
  message: '{{error}}',
  severity: 'warn',
  recommended: true,
  given: '$.servers[*].url',
  then: {
    function: serverNotLocalhost,
  },
},
```

### Choosing `given`

The `given` JSONPath selects the nodes to check. Common patterns:

| What to select | JSONPath |
|----------------|---------|
| All server URLs | `$.servers[*].url` |
| All channel operations | `$.channels.*.[publish,subscribe]` |
| All message payloads (v2) | `$.channels.*.[publish,subscribe].message.payload` |
| Root document | `$` |
| Channel map keys | `$.channels` (then use `field: '@key'`) |
| Operations in root (v3) | `$.operations.*` |
| Component messages | `$.components.messages.*` |

### Choosing `resolved`

- If your rule uses `$ref` strings to check referencing patterns → `resolved: false`
- If your rule validates merged content → omit or `resolved: true` (default)

---

## Step 3: Add the Import

Make sure your function is imported in the ruleset file:

```typescript
// packages/parser/src/ruleset/v2/ruleset.ts (top imports)
import { serverNotLocalhost } from './functions/serverNotLocalhost';
```

---

## Step 4: Write Tests

Create a test file following the existing pattern.

**File:** `packages/parser/test/ruleset/rules/v2/asyncapi2-server-not-localhost.spec.ts`

```typescript
import { testRule, DiagnosticSeverity } from '../../tester';

describe('asyncapi2-server-not-localhost', function() {
  
  it('should pass for non-localhost server URL', async () => {
    const { document, diagnostics } = await testRule('asyncapi2-server-not-localhost', {
      asyncapi: '2.6.0',
      info: { title: 'Test', version: '1.0.0' },
      servers: {
        production: {
          url: 'mqtt.example.com',
          protocol: 'mqtt',
        }
      },
      channels: {}
    });
    
    expect(document).toBeDefined();
    expect(diagnostics).toHaveLength(0);
  });

  it('should warn for localhost URL', async () => {
    const { diagnostics } = await testRule('asyncapi2-server-not-localhost', {
      asyncapi: '2.6.0',
      info: { title: 'Test', version: '1.0.0' },
      servers: {
        dev: {
          url: 'localhost',
          protocol: 'mqtt',
        }
      },
      channels: {}
    });
    
    expect(diagnostics).toHaveLength(1);
    expect(diagnostics[0].severity).toBe(DiagnosticSeverity.Warning);
    expect(diagnostics[0].message).toContain('localhost');
    expect(diagnostics[0].path).toEqual(['servers', 'dev', 'url']);
  });

  it('should warn for 127.0.0.1 URL', async () => {
    const { diagnostics } = await testRule('asyncapi2-server-not-localhost', {
      asyncapi: '2.6.0',
      info: { title: 'Test', version: '1.0.0' },
      servers: {
        dev: {
          url: '127.0.0.1:1883',
          protocol: 'mqtt',
        }
      },
      channels: {}
    });
    
    expect(diagnostics).toHaveLength(1);
    expect(diagnostics[0].severity).toBe(DiagnosticSeverity.Warning);
  });

  it('should not warn for server with port variable', async () => {
    const { diagnostics } = await testRule('asyncapi2-server-not-localhost', {
      asyncapi: '2.6.0',
      info: { title: 'Test', version: '1.0.0' },
      servers: {
        production: {
          url: 'mqtt.example.com:{port}',
          protocol: 'mqtt',
          variables: { port: { default: '1883' } }
        }
      },
      channels: {}
    });
    
    expect(diagnostics).toHaveLength(0);
  });
});
```

### Understanding `testRule()`

The `testRule()` helper:
1. Creates a `Parser` with only the named rule enabled
2. Parses the document object passed as the second argument
3. Returns `{ document, diagnostics }`

This isolation is critical — it ensures only YOUR rule fires, not every other rule. Without it, you'd see warnings from `asyncapi-servers` and other rules mixed in.

---

## Step 5: Run the Tests

```bash
# Run only your new test
npx jest "asyncapi2-server-not-localhost" --rootdir packages/parser

# Run all ruleset tests to ensure nothing broke
npx jest "test/ruleset" --rootdir packages/parser
```

---

## Step 6: Integration Test

Test that the rule fires correctly when using the full `Parser`:

```js
// scratch/test-new-rule.js
const { Parser } = require('./packages/parser/cjs/index.js');

async function main() {
  const parser = new Parser();

  const doc = `
asyncapi: '2.6.0'
info:
  title: Test
  version: '1.0.0'
servers:
  dev:
    url: localhost
    protocol: mqtt
channels: {}
`;

  const diagnostics = await parser.validate(doc);
  const localhostWarning = diagnostics.find(d => d.code === 'asyncapi2-server-not-localhost');
  
  if (localhostWarning) {
    console.log('Rule fires correctly:', localhostWarning.message);
  } else {
    console.log('Rule did NOT fire (possible issue)');
    console.log('All diagnostics:', diagnostics.map(d => d.code));
  }
}

main().catch(console.error);
```

---

## Checklist Before Opening a PR

- [ ] New function file in `src/ruleset/v2/functions/` or `src/ruleset/v3/functions/` (or shared `functions/`)
- [ ] Rule added to the correct ruleset in `ruleset.ts`
- [ ] Function imported in `ruleset.ts`
- [ ] Test file in `test/ruleset/rules/v2/` or `test/ruleset/rules/v3/`
- [ ] At least one passing case and one failing case in tests
- [ ] Test verifies the `severity`, `message`, and `path` of the diagnostic
- [ ] All existing tests still pass: `npm run parser:test:unit`
- [ ] ESLint passes: `npm run lint`

---

## Real Example to Study

Look at the `asyncapi2-channel-servers` rule as a reference implementation:

- Function: `packages/parser/src/ruleset/functions/channelServers.ts`
- Rule definition: `packages/parser/src/ruleset/v2/ruleset.ts` → `asyncapi2-channel-servers`
- Test: `packages/parser/test/ruleset/rules/v2/asyncapi2-channel-servers.spec.ts`

---

## Next Step

Read [04-extending-schema-parsers.md](./04-extending-schema-parsers.md) for how to build and test a custom schema parser plugin.
