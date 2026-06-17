# Prerequisites and Installation

> **Goal:** Get the repository cloned, dependencies installed, and the project built so you can run tests and experiment locally.

---

## Prerequisites

| Requirement | Minimum | Notes |
|-------------|---------|-------|
| **Node.js** | 18.x | The `package.json` engines field specifies `>=18`. |
| **npm** | 8.x | Ships with Node 18. Workspaces support required. |
| **Git** | Any recent version | For cloning and contributing. |

Check your versions:

```bash
node --version   # should print v18.x.x or higher
npm --version    # should print 8.x.x or higher
```

---

## Step 1: Clone the Repository

```bash
git clone https://github.com/asyncapi/parser-js.git
cd parser-js
```

If you are working on a fork (recommended for contributions):

```bash
git clone https://github.com/YOUR_USERNAME/parser-js.git
cd parser-js
git remote add upstream https://github.com/asyncapi/parser-js.git
```

---

## Step 2: Install Dependencies

```bash
npm install
```

This installs dependencies for all workspaces (both `packages/parser` and `packages/multi-parser`) because this is an npm workspaces monorepo. The `node_modules` folder at the root is shared, with workspace-specific packages hoisted automatically.

> **What gets installed:** ~300MB of dependencies including TypeScript, Jest, Webpack, Spectral, Ajv, and all schema parser plugins used in tests.

---

## Step 3: Build the Project

```bash
npm run build
```

This is required before running tests because the test suite imports the compiled output (CJS/ESM), not the TypeScript source directly.

What the build command does (via Turborepo):

```
turbo run build
├── packages/parser build
│   ├── build:esm  → tsc              → packages/parser/esm/
│   ├── build:cjs  → tsc --project tsconfig.cjs.json → packages/parser/cjs/
│   └── build:browser → webpack      → packages/parser/browser/
└── packages/multi-parser build
    ├── build:esm  → tsc              → packages/multi-parser/esm/
    └── build:cjs  → tsc --project tsconfig.cjs.json → packages/multi-parser/cjs/
```

After a successful build you should see:

```
packages/parser/
├── cjs/        ← CommonJS build (Node.js require())
├── esm/        ← ES Module build (import)
└── browser/    ← Webpack UMD bundle for browsers

packages/multi-parser/
├── cjs/
└── esm/
```

---

## Step 4: Verify Everything Works

```bash
npm run parser:test:unit
```

This runs the Jest unit test suite (163 spec files) for `@asyncapi/parser`. A successful run looks like:

```
Test Suites: 163 passed, 163 total
Tests:       1247 passed, 1247 total
Snapshots:   0 total
Time:        ~30s
```

If you see failures here before making any changes, there may be CI issues already present in the branch. Check the [project issues](https://github.com/asyncapi/parser-js/issues).

---

## Build Outputs Explained

### `cjs/` (CommonJS)

The format used by `require()` in Node.js. The parser's `package.json` sets `"main": "cjs/index.js"`.

```js
const { Parser } = require('@asyncapi/parser');
```

### `esm/` (ES Modules)

The format used by `import` in modern Node.js or bundlers. The parser's `package.json` sets `"module": "esm/index.js"`.

```js
import { Parser } from '@asyncapi/parser';
```

### `browser/`

A Webpack-bundled UMD file for direct browser use. It:
- Bundles all dependencies (except native browser APIs)
- Polyfills Node.js APIs (`path`, `buffer`, etc.)
- Replaces `node-fetch` with the browser's native `fetch`
- Exports `window.AsyncAPIParser` globally

```html
<script src="https://unpkg.com/@asyncapi/parser@latest/browser/index.js"></script>
<script>
  const parser = new window.AsyncAPIParser();
  parser.parse('asyncapi: "2.6.0"\n...');
</script>
```

---

## Using the Parser in Your Own Project

Once you understand the source, you may want to experiment in a separate directory. Install from npm:

```bash
mkdir my-asyncapi-experiment
cd my-asyncapi-experiment
npm init -y
npm install @asyncapi/parser
```

For the examples in this documentation, you can also use the local build by linking:

```bash
cd packages/parser
npm link
cd /path/to/my-asyncapi-experiment
npm link @asyncapi/parser
```

---

## Development Workflow

The most common development loop:

```bash
# 1. Make a change in packages/parser/src/

# 2. Rebuild (required since tests run against compiled output)
npm run parser:build

# 3. Run tests to verify
npm run parser:test:unit

# 4. (Optional) Run a single test file for speed
npx jest --testPathPattern="parse.spec.ts" --rootdir packages/parser
```

For changes to ruleset rules, rebuilding is not strictly required since Jest's SWC transform compiles TypeScript on-the-fly. However, browser tests always require a fresh build.

---

## Useful npm Scripts at a Glance

Run these from the **repository root** (where the root `package.json` lives):

| Script | What it does |
|--------|-------------|
| `npm test` | Full build + all tests (parser + multi-parser + browser) |
| `npm run build` | Build all packages |
| `npm run parser:build` | Build only `@asyncapi/parser` |
| `npm run parser:test` | Run all parser tests (unit + browser) |
| `npm run parser:test:unit` | Fast: unit tests only, no browser |
| `npm run parser:test:browser` | Browser bundle tests (requires Playwright + Chromium) |
| `npm run multi-parser:build` | Build only `@asyncapi/multi-parser` |
| `npm run multi-parser:test` | Run multi-parser tests |
| `npm run lint` | ESLint across all packages |
| `npm run lint:fix` | ESLint with auto-fix |

---

## Browser Test Setup

Browser tests use Playwright to launch a real Chromium browser and test the bundled parser. On first run, install Chromium:

```bash
npm run parser:test:browser
# This script automatically calls: playwright install chromium
```

The browser test:
1. Builds the webpack bundle
2. Starts a local HTTP server serving `test/browser/sample-page.html`
3. Launches Playwright Chromium
4. Runs `test/browser/browser.spec.ts`

---

## Next Step

Read [02-your-first-parse.md](./02-your-first-parse.md) to write and run your first AsyncAPI document parse.
