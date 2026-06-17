# Understanding GitHub Issues

> **Goal:** Learn how to read, classify, and reproduce GitHub issues for `asyncapi/parser-js`. By the end you will have a repeatable process for going from "new issue" to "I understand the root cause".

---

## Issue URL

All issues are at: [github.com/asyncapi/parser-js/issues](https://github.com/asyncapi/parser-js/issues)

---

## Label Taxonomy

Issues are labeled to help route them:

| Label | Meaning |
|-------|---------|
| `bug` | Something doesn't work as expected |
| `enhancement` | New feature or improvement |
| `question` | Usage question (may belong in Discussions) |
| `good first issue` | Small, well-defined, good for new contributors |
| `help wanted` | Maintainers want community help |
| `wontfix` | Intentional behavior or out of scope |
| `duplicate` | Already reported elsewhere |
| `invalid` | Not a valid bug (user error, etc.) |
| `stale` | Inactive for 60+ days |
| `triaged` | Maintainer has assessed and categorized |

---

## Issue Classification Decision Flowchart

When you read a new issue, run through this decision tree:

```
1. Does the user report a crash / wrong output from parse() or validate()?
   ├─ Yes → PARSER BUG — go to step 2
   └─ No → Is it a validation rule producing wrong results?
            ├─ Yes → RULE BUG — go to step 3
            └─ No → Is it about the typed model API (wrong accessor, wrong value)?
                     ├─ Yes → MODEL BUG — go to step 4
                     └─ No → Is it a feature request?
                              ├─ Yes → ENHANCEMENT — assess feasibility
                              └─ No → Is it a usage question?
                                       └─ Yes → Redirect to Discussions or answer directly

2. PARSER BUG: Which component is likely at fault?
   ├─ $ref resolution fails / wrong? → src/resolver.ts
   ├─ Traits not merged? → src/custom-operations/apply-traits.ts
   ├─ Circular refs not handled? → src/custom-operations/check-circular-refs.ts
   ├─ Schema format unknown/wrong? → src/schema-parser/
   ├─ Uncaught exception? → Likely a bug in src/parse.ts error handling
   └─ Browser-only issue? → webpack.config.js, browser polyfills

3. RULE BUG:
   ├─ False positive (valid doc rejected)? → Rule condition too strict
   ├─ False negative (invalid doc accepted)? → Rule missing or condition too loose
   ├─ Wrong JSONPath in given:? → Need to trace JSONPath against example doc
   └─ Which rule fires? → Read diagnostic code, find in src/ruleset/

4. MODEL BUG:
   ├─ v2 model? → src/models/v2/
   ├─ v3 model? → src/models/v3/
   └─ Which accessor? → Find the model file matching the reported method
```

---

## Reproducing an Issue Locally: Template Script

Use this template for every issue. Fill in the AsyncAPI document and options from the issue report:

```js
// scratch/reproduce-issue-NNN.js
const { Parser } = require('./packages/parser/cjs/index.js');

async function main() {
  const parser = new Parser();

  // Paste the AsyncAPI document from the issue here:
  const doc = `
asyncapi: '2.6.0'
info:
  title: Reproduce Issue #NNN
  version: '1.0.0'
channels:
  # ... paste channels from issue
`;

  const { document, diagnostics } = await parser.parse(doc);

  // Print ALL diagnostics
  console.log('=== Diagnostics ===');
  diagnostics.forEach(d => {
    const sev = ['Error', 'Warning', 'Info', 'Hint'][d.severity];
    console.log(`[${sev}] ${d.code}: ${d.message}`);
    console.log(`  Path: ${d.path.join(' > ') || 'root'}`);
  });

  // Print document state
  console.log('\n=== Document ===');
  if (document) {
    console.log('Version:', document.version());
    // Add specific accessors the issue mentions
  } else {
    console.log('Document is UNDEFINED');
  }
}

main().catch(err => {
  console.error('UNCAUGHT ERROR:', err);
  console.error(err.stack);
});
```

Then run:
```bash
# Build first (required for CJS imports)
npm run parser:build

# Run the reproduction
node scratch/reproduce-issue-NNN.js
```

---

## Reading Diagnostics to Understand Failures

When a user reports "the parser rejects my document", the first step is reading diagnostics carefully.

Each diagnostic has:
- **`code`**: The rule name. Look this up in the ruleset tables ([Chapter 6](../06-validation-ruleset/02-core-rules.md)) to understand what rule triggered.
- **`path`**: Where in the document the problem is. `['channels', 'user/registered', 'publish', 'message']` tells you the exact location.
- **`message`**: What the rule says is wrong.
- **`range`**: The character position in the YAML source.
- **`source`**: Which file (for multi-file documents with `$ref`).

### Example: Tracing a False Positive

User reports: "My valid document gets an `asyncapi2-channel-parameters` error but all parameters are defined."

1. **Reproduce**: Copy their document, run it through the parser.
2. **Read the diagnostic**: Check `d.path` — does it point to the right channel?
3. **Find the rule**: `asyncapi2-channel-parameters` → `src/ruleset/v2/functions/channelParameters.ts`
4. **Read the function**: What JSONPath does it use? What does it extract from `parameters:`?
5. **Trace**: Run the JSONPath against the user's document manually. Does it produce the expected output?
6. **Identify**: Maybe the channel address has special characters that break the JSONPath, or the parameters are nested under a `$ref`.

---

## Key Areas of Complexity to Know

### 1. Circular References

Issues around circular refs are tricky because:
- `checkCircularRefs` detects them early
- `resolveCircularRefs` annotates them after schema parsing
- `stringify` handles them during serialization
- Circular detection may miss indirect cycles (A→B→C→A vs A→A)

Look for issues mentioning `x-parser-circular`, `JSON.stringify` errors, or infinite loops.

### 2. v2 vs v3 Model Differences

The most common source of confusion. Users sometimes try v3 API patterns on v2 documents or vice versa:
- "Why does `channel.address()` return `undefined`?" → Because in v2, the address IS the channel ID. `channel.address()` = `channel.id()` in v2.
- "Why doesn't `document.operations()` work?" → It does, but in v2 it derives operations from channels; in v3 they're top-level.

### 3. `$ref` Resolution and `source`

Many issues come from parsing documents with external `$ref`s without providing `source`:
- "References not resolving" → Almost always missing `source` option.
- "Wrong line numbers in errors" → May be related to `source` being wrong.

### 4. Schema Format Not Registered

Many "parse fails" issues turn out to be: user's document has `schemaFormat: 'application/vnd.apache.avro...'` but they haven't registered the Avro parser.

Look for `asyncapi2-schemas` errors mentioning "Unknown schema format".

### 5. Traits Not Merged as Expected

The traits merge order is: trait fields → applied to operation/message. Later traits override earlier ones. If a user reports "my trait binding isn't showing up", check:
- Is `applyTraits: false` in their `ParseOptions`?
- Is the trait using a deep-merge-able structure?

---

## Checking if a Bug is Already Known

Before diving deep:

```bash
# Search existing issues via GitHub CLI
gh issue list --repo asyncapi/parser-js --search "circular reference" --state all

# Or via web search
# github.com/asyncapi/parser-js/issues?q=circular+reference
```

---

## Useful Metadata to Request from Issue Reporters

When an issue lacks enough context:

1. **What AsyncAPI spec version?** (2.x vs 3.x)
2. **What `@asyncapi/parser` version?** (`npm ls @asyncapi/parser`)
3. **Minimal reproduction document** (the smallest YAML that triggers the issue)
4. **Full diagnostics output** (all fields including code, path, range)
5. **Node.js version?** (`node --version`)
6. **Browser or Node.js?**

---

## Next Step

Read [02-debugging-and-tracing.md](./02-debugging-and-tracing.md) for techniques to trace through the parser pipeline and find root causes.
