# Node Modules: How They Work, Making Changes, Patching & Best Practices

> **Target Audience:** Senior Front End Developer Interview Preparation

## Table of Contents

1. [Understanding node_modules](#understanding-node_modules)
2. [Making Changes to node_modules (Patching)](#making-changes-to-node_modules-patching)
3. [Dependency Management Best Practices](#dependency-management-best-practices)
4. [Advanced Topics](#advanced-topics)
5. [Common Issues & Troubleshooting](#common-issues--troubleshooting)
6. [Package Manager Comparison](#package-manager-comparison)

---

## Understanding node_modules

### What is node_modules and how does it work?

`node_modules` is the directory where npm/yarn/pnpm installs all dependencies listed in your `package.json`. It contains:

- **Direct dependencies** — packages you explicitly installed
- **Transitive dependencies** — dependencies of your dependencies
- **Binary executables** — in `node_modules/.bin/`

```
my-project/
├── node_modules/
│   ├── .bin/              # Executable binaries (react-scripts, vite, etc.)
│   ├── react/             # Direct dependency
│   │   ├── index.js
│   │   └── package.json
│   ├── lodash/
│   └── ...                # Hundreds of transitive dependencies
├── package.json
├── package-lock.json
└── src/
```

**Why is node_modules so large?** Each package can have its own dependencies, which can have their own, leading to deep dependency trees. A typical React app can have 800-1500+ packages.

---

### How does npm/yarn/pnpm install work (dependency resolution)?

**npm (v7+) — Hoisting with flat structure:**

```
Installing: react-router-dom (depends on react@>=16)
            react@18

npm resolves:
- Place react@18 at top level (hoisted)
- react-router-dom's react peer dep satisfied by top-level react
- Result: flat node_modules
```

```
node_modules/
├── react/                   (v18 — hoisted to top)
├── react-router-dom/
│   └── node_modules/        (only if version conflict)
│       └── some-package/
└── react-dom/               (v18 — hoisted)
```

**Before npm v3 — Nested (no hoisting):**
```
node_modules/
├── package-a/
│   └── node_modules/
│       └── lodash/          (v3)
└── package-b/
    └── node_modules/
        └── lodash/          (v4)
```

**Hoisting pros/cons:**

| Aspect | Hoisted (npm/yarn) | Nested (old npm) |
|---|---|---|
| Disk usage | Smaller | Very large |
| Phantom deps | Possible | Not possible |
| Version conflicts | Rare | Each package gets its version |
| Install speed | Faster | Slower |

---

### What's the difference between package.json and lockfiles?

```json
// package.json — INTENT (what you want)
{
  "dependencies": {
    "react": "^18.0.0",        // any 18.x.x
    "lodash": "~4.17.0",       // 4.17.x only
    "axios": "1.6.0",          // exactly 1.6.0
    "express": ">= 4.0.0"      // 4.0.0 or higher
  }
}
```

```json
// package-lock.json — EXACT STATE (what was actually installed)
{
  "lockfileVersion": 3,
  "packages": {
    "node_modules/react": {
      "version": "18.2.0",     // exact version
      "resolved": "https://registry.npmjs.org/react/-/react-18.2.0.tgz",
      "integrity": "sha512-...", // integrity hash
      "dependencies": {
        "loose-envify": "^1.1.0"
      }
    }
  }
}
```

**Key rule:** Always commit your lockfile. This ensures every developer and CI gets the same exact versions.

---

### What is Semantic Versioning (semver)?

```
Version: MAJOR.MINOR.PATCH
         1.2.3

MAJOR — Breaking changes (incompatible API changes)
MINOR — New features (backward compatible)
PATCH — Bug fixes (backward compatible)
```

**Range operators in package.json:**

```
Exact:    "1.2.3"          → only 1.2.3
Caret:    "^1.2.3"         → >=1.2.3 <2.0.0 (allows minor+patch)
Tilde:    "~1.2.3"         → >=1.2.3 <1.3.0 (allows patch only)
Greater:  ">1.2.3"         → any version above 1.2.3
Range:    ">=1.2.3 <2.0.0" → explicit range
Wildcard: "1.x"            → any 1.x.x
Latest:   "*"              → any version (dangerous!)
```

```bash
# Check what a range resolves to
npm semver "^1.2.3" 1.3.0    # outputs: 1.3.0 (matches)
npm semver "^1.2.3" 2.0.0    # empty (doesn't match)
```

---

### How does Node.js module resolution work?

```js
// require('./utils') resolution algorithm:
// 1. Try exact file: ./utils (if no extension, try .js, .json, .node)
// 2. Try ./utils.js
// 3. Try ./utils.json
// 4. Try ./utils/index.js
// 5. Throw MODULE_NOT_FOUND

// require('lodash') resolution:
// 1. Is it a core module? (fs, path, http) → return
// 2. Walk up directories looking for node_modules/lodash:
//    ./node_modules/lodash
//    ../node_modules/lodash
//    ../../node_modules/lodash
//    ... (up to filesystem root)
// 3. Use 'main' field from lodash/package.json to find entry point
```

**Modern ESM resolution (`import`):**

```js
// ESM with "exports" field in package.json
// package.json of a library:
{
  "exports": {
    ".": {
      "import": "./dist/index.esm.js",   // ESM import
      "require": "./dist/index.cjs.js",  // CommonJS require
      "types": "./dist/index.d.ts"       // TypeScript
    },
    "./utils": "./dist/utils.js"          // named subpath
  }
}

// Consumer:
import { helper } from 'my-lib/utils';  // resolves via exports map
```

---

### What are the different types of dependencies?

```json
{
  "dependencies": {
    "react": "^18.0.0",
    "axios": "^1.6.0"
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "vitest": "^1.0.0",
    "eslint": "^8.0.0"
  },
  "peerDependencies": {
    "react": ">=16.8.0"
  },
  "peerDependenciesMeta": {
    "react": { "optional": false }
  },
  "optionalDependencies": {
    "fsevents": "^2.3.0"
  }
}
```

| Type | Installed When | When to Use |
|---|---|---|
| `dependencies` | Always (prod + dev) | Required at runtime by your app/lib |
| `devDependencies` | Local dev only | Build tools, testing, linting |
| `peerDependencies` | NOT auto-installed (warning shown) | Libraries that need host to provide deps |
| `optionalDependencies` | If possible, continue if fails | Native modules, platform-specific |

---

### What is the .npmrc configuration file?

```ini
# .npmrc — project-level npm configuration
registry=https://registry.npmjs.org/

# Use private registry for @company scope
@company:registry=https://npm.pkg.github.com/

# Authentication for private registry
//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}

# Save exact versions (no ^ or ~)
save-exact=true

# Node version compatibility
engine-strict=true

# Cache configuration
cache=~/.npm

# Proxy settings (corporate environments)
proxy=http://proxy.company.com:8080
https-proxy=http://proxy.company.com:8080
```

---

## Making Changes to node_modules (Patching)

### Why would you ever need to modify node_modules?

Common reasons:
1. **Bug in a dependency** — the maintainer hasn't released a fix yet
2. **Missing feature** — need a small addition not worth waiting for
3. **Security vulnerability** — need to patch before maintainer releases fix
4. **Fork diverged** — dependency has abandoned maintenance
5. **Corporate workaround** — need to modify for internal compliance

**Important:** Direct edits to `node_modules` are wiped on every `npm install`. You need a proper patching strategy.

---

### How do you use patch-package to patch a dependency?

```bash
# Step 1: Install patch-package
npm install --save-dev patch-package
# or
yarn add --dev patch-package
```

```bash
# Step 2: Make your changes directly in node_modules
# (Edit the file you need to fix)
nano node_modules/some-package/dist/index.js
```

```bash
# Step 3: Create the patch file
npx patch-package some-package
# Creates: patches/some-package+1.2.3.patch
```

```diff
# patches/some-package+1.2.3.patch (auto-generated)
diff --git a/node_modules/some-package/dist/index.js b/node_modules/some-package/dist/index.js
index abc123..def456 100644
--- a/node_modules/some-package/dist/index.js
+++ b/node_modules/some-package/dist/index.js
@@ -42,7 +42,7 @@ function processData(input) {
-  return input.filter(x => x !== null);
+  return input.filter(x => x !== null && x !== undefined);
 }
```

```json
// Step 4: Add postinstall script to package.json
{
  "scripts": {
    "postinstall": "patch-package"
  }
}
```

```bash
# Step 5: Commit the patches directory
git add patches/
git commit -m "patch: fix null handling in some-package"
```

**Now every time someone runs `npm install`, the patch is automatically applied via `postinstall`.**

---

### How do you patch a dependency with Yarn Berry?

```bash
# Yarn 2+ (Berry) has built-in patching

# Step 1: Create a patch
yarn patch some-package

# This creates a temporary copy at:
# /tmp/xfs-xyz/user/

# Step 2: Edit the temporary copy
nano /tmp/xfs-xyz/user/dist/index.js

# Step 3: Commit the patch
yarn patch-commit /tmp/xfs-xyz/user/
```

```yaml
# package.json / .yarnrc.yml — patch stored as:
# .yarn/patches/some-package-npm-1.2.3-abc123.patch

# yarn.lock will reference:
# "some-package@patch:some-package@1.2.3#./.yarn/patches/some-package-npm-1.2.3.patch"
```

---

### How do you patch a dependency with pnpm?

```bash
# pnpm 6.25+ has built-in patching

# Step 1: Create the patch
pnpm patch some-package@1.2.3
# Opens an editable copy at: /tmp/pnpm-patch-xyz/

# Step 2: Edit the copy
nano /tmp/pnpm-patch-xyz/dist/index.js

# Step 3: Commit the patch
pnpm patch-commit /tmp/pnpm-patch-xyz/
```

```yaml
# pnpm-lock.yaml will reference the patch:
# patchedDependencies:
#   some-package@1.2.3:
#     hash: abc123
#     path: patches/some-package@1.2.3.patch
```

---

### When should you fork a package instead of patching?

**Fork when:**
- Changes are extensive (more than a few lines)
- You need ongoing maintenance
- The library is abandoned
- You want to submit changes upstream as a PR

```bash
# Step 1: Fork on GitHub
# Go to github.com/original-author/some-package → Fork

# Step 2: Clone and make changes
git clone https://github.com/your-org/some-package
cd some-package
# Make your changes
git commit -am "fix: handle undefined values"
git push origin main
```

```json
// Step 3: Reference via GitHub URL
{
  "dependencies": {
    "some-package": "github:your-org/some-package#main"
  }
}
```

```json
// Or via npm alias (clean version)
{
  "dependencies": {
    "some-package": "npm:@your-org/some-package@^1.2.3"
  }
}
```

```bash
# Or publish to npm as a scoped package
npm publish --access public
```

---

### How do npm link / yarn link work for local package development?

```bash
# Scenario: Developing a local package and testing in another project

# Step 1: In your local package directory
cd /path/to/my-library
npm link
# Creates a global symlink: ~/.nvm/versions/node/vX/lib/node_modules/my-library → /path/to/my-library

# Step 2: In your project that uses the library
cd /path/to/my-project
npm link my-library
# Creates: node_modules/my-library → global symlink → your local source

# Step 3: Changes in my-library are immediately reflected in my-project

# Cleanup
npm unlink my-library      # in project
npm unlink                  # in library
```

```bash
# pnpm workspace approach (better for monorepos)
# pnpm-workspace.yaml
packages:
  - 'packages/*'

# Then in your app's package.json:
{
  "dependencies": {
    "my-library": "workspace:*"
  }
}
```

---

### How do you override nested dependency versions?

```json
// npm overrides (npm v8.3+)
{
  "overrides": {
    // Force all packages to use react@18
    "react": "18.2.0",
    
    // Only override react within a specific package
    "some-package": {
      "lodash": "4.17.21"
    }
  }
}
```

```json
// Yarn resolutions (Yarn Classic)
{
  "resolutions": {
    "lodash": "4.17.21",
    "**/moment": "2.29.4"
  }
}
```

```yaml
# pnpm overrides
# package.json
{
  "pnpm": {
    "overrides": {
      "lodash": "4.17.21",
      "some-package>lodash": "4.17.21"
    }
  }
}
```

**Use case:** A transitive dependency has a security vulnerability. You can override it to a patched version while waiting for the upstream to update.

---

## Dependency Management Best Practices

### How do you audit your dependencies for security vulnerabilities?

```bash
# npm built-in audit
npm audit
npm audit --audit-level=high   # fail only for high/critical
npm audit fix                   # auto-fix safe updates
npm audit fix --force           # upgrade breaking changes (careful!)

# More detailed output
npm audit --json | npx better-npm-audit
```

```bash
# Snyk — more powerful vulnerability scanner
npm install -g snyk
snyk auth
snyk test                    # scan current project
snyk monitor                 # continuous monitoring
snyk fix                     # auto-remediation
```

```bash
# Socket.dev — detects malicious packages
npx socket npm install react  # security-aware install
```

---

### How do you keep dependencies updated?

```bash
# Check for outdated packages
npm outdated

# Update all to latest minor/patch
npm update

# Update to latest major (breaking changes possible)
npm install react@latest react-dom@latest

# Interactive update with npx npm-check-updates
npx ncu                  # show what can be updated
npx ncu -u              # update package.json
npm install             # install updated versions
```

**Automated dependency updates:**

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: npm
    directory: /
    schedule:
      interval: weekly
    groups:
      react:
        patterns:
          - "react"
          - "react-dom"
          - "@types/react*"
    ignore:
      - dependency-name: "some-stable-dep"
        update-types: ["version-update:semver-major"]
```

---

### How do you evaluate a new dependency before adding it?

**Checklist:**

```bash
# 1. Check bundle size impact
npx bundlephobia <package-name>

# 2. Check download stats / popularity
npm info <package-name>

# 3. Check last publish date and maintenance status
npm view <package-name> time.modified

# 4. Check for vulnerabilities
npm audit  # after install
snyk test

# 5. Check alternative lighter options
# bundlephobia.com → "Similar packages"
```

**Questions to ask:**
- Does it have TypeScript support?
- What's the bundle size (gzip)?
- When was it last updated?
- How many open issues/PRs?
- Is there a smaller alternative?
- Does it have a peer dependency on a different version of React/etc.?

---

### How does Tree Shaking work and how do you enable it?

Tree shaking removes unused code from your bundle. It requires:
1. **ES modules** (import/export, not require/module.exports)
2. **`sideEffects` field** in package.json

```json
// package.json of a library — declare side effects
{
  "sideEffects": false,        // no files have side effects (all can be tree-shaken)
  // or:
  "sideEffects": ["*.css", "./src/polyfills.js"]  // these files have side effects
}
```

```js
// Library: only import what you need
// BAD — prevents tree shaking in some bundlers
import _ from 'lodash';
const result = _.pick(obj, ['a', 'b']);

// GOOD — specific import, easier to tree-shake
import pick from 'lodash/pick';
// or with lodash-es (ESM version):
import { pick } from 'lodash-es';
```

```ts
// Vite/Rollup tree shakes by default
// Webpack needs mode: 'production' and usedExports: true
module.exports = {
  mode: 'production',
  optimization: {
    usedExports: true,
  },
};
```

---

## Advanced Topics

### How do you create and publish your own npm package?

```bash
# Initialize
mkdir my-package && cd my-package
npm init

# Add build tooling
npm install --save-dev typescript tsup
```

```json
// package.json
{
  "name": "@yourscope/my-package",
  "version": "1.0.0",
  "description": "My awesome package",
  "main": "./dist/index.js",
  "module": "./dist/index.esm.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.esm.js",
      "require": "./dist/index.js",
      "types": "./dist/index.d.ts"
    }
  },
  "files": ["dist"],
  "scripts": {
    "build": "tsup src/index.ts --format cjs,esm --dts",
    "prepublishOnly": "npm run build"
  },
  "peerDependencies": {
    "react": ">=16.8.0"
  }
}
```

```bash
# Publish to npm
npm login
npm publish --access public   # for scoped packages

# Publish to GitHub Packages
npm publish --registry=https://npm.pkg.github.com/
```

---

### How do workspaces manage monorepo dependencies?

```json
// root package.json
{
  "private": true,
  "workspaces": ["packages/*", "apps/*"],
  "scripts": {
    "build": "npm run build --workspaces",
    "test": "npm run test --workspaces"
  }
}
```

```
monorepo/
├── package.json          (root workspace)
├── node_modules/         (shared dependencies hoisted here)
├── packages/
│   ├── ui-components/
│   │   ├── package.json  { "name": "@company/ui" }
│   │   └── src/
│   └── utils/
│       ├── package.json  { "name": "@company/utils" }
│       └── src/
└── apps/
    └── web/
        ├── package.json  { "dependencies": { "@company/ui": "*" } }
        └── src/
```

```bash
# Install package in specific workspace
npm install lodash --workspace=packages/utils

# Run script in specific workspace
npm run build --workspace=apps/web

# Run script in all workspaces
npm run test --workspaces
```

---

### What is pnpm's content-addressable storage?

pnpm uses a **global content-addressable store** — each package version is stored once globally and **hard-linked** into project `node_modules`.

```
~/.pnpm-store/v3/         ← Global store (shared across ALL projects)
  files/
    ab/cdef1234.../       ← File content addressed by hash
    
project/
  node_modules/
    .pnpm/
      react@18.2.0/       ← Virtual store
        node_modules/
          react/          ← Hard link to global store
    react/                ← Symlink to .pnpm/react@18.2.0/node_modules/react
```

**Benefits:**
- 60-80% less disk space vs npm
- Faster installs (copy via hard links)
- Strict dependency isolation (can't accidentally use unlisted deps)
- Non-flat structure prevents phantom dependencies

---

### What are Phantom Dependencies?

**Phantom dependencies** occur when a package uses a module that is available in `node_modules` because it was hoisted, but isn't listed in `package.json`.

```json
// your package.json
{
  "dependencies": {
    "some-lib": "^1.0.0"
  }
  // Note: lodash is NOT listed
}
```

```js
// your code — works because some-lib's lodash is hoisted!
import { pick } from 'lodash'; // phantom dep — could break anytime

// When some-lib upgrades and no longer uses lodash,
// or starts using lodash@5, this import BREAKS.
```

**pnpm prevents this:** Its non-flat structure means your code can only import what's in your `package.json`.

---

### What is Yarn Berry's Plug'n'Play (PnP)?

Yarn PnP eliminates `node_modules` entirely:

```yaml
# .yarnrc.yml
nodeLinker: pnp
```

```
Instead of node_modules/:
.yarn/
  cache/                  ← Packages stored as zip files
  pnp.cjs                 ← PnP runtime loader
.pnp.loader.mjs
```

**How it works:**
- Packages stored as zip files in `.yarn/cache/`
- `.pnp.cjs` maps package names to locations
- Node.js patched to use the PnP resolution instead of filesystem traversal

**Pros:** Faster installs, no phantom deps, smaller repo size, instant caching
**Cons:** Compatibility issues with some tools that expect `node_modules`, requires IDE plugins

---

### How should you handle node_modules in Docker?

```dockerfile
# Multi-stage build to minimize image size
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production  # install only production deps

FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production image — minimal size
FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production

# Copy only what's needed
COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY package.json ./

EXPOSE 3000
CMD ["node", "dist/server.js"]
```

```yaml
# .dockerignore — prevent copying node_modules from host
node_modules
.npm
.yarn
dist
.env*
*.log
```

---

### How do you cache node_modules in CI/CD?

```yaml
# GitHub Actions — cache node_modules
- name: Cache dependencies
  uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-

- name: Install dependencies
  run: npm ci
```

```yaml
# With pnpm — cache the global store
- uses: pnpm/action-setup@v3
  with:
    version: 8

- name: Get pnpm store directory
  id: pnpm-cache
  run: echo "STORE_PATH=$(pnpm store path)" >> $GITHUB_OUTPUT

- uses: actions/cache@v4
  with:
    path: ${{ steps.pnpm-cache.outputs.STORE_PATH }}
    key: ${{ runner.os }}-pnpm-${{ hashFiles('**/pnpm-lock.yaml') }}

- run: pnpm install --frozen-lockfile
```

---

### What are Import Maps and CDN-based dependency loading?

```html
<!-- Import maps — map bare specifiers to URLs (no bundler needed) -->
<script type="importmap">
{
  "imports": {
    "react": "https://esm.sh/react@18",
    "react-dom/client": "https://esm.sh/react-dom@18/client",
    "lodash-es": "https://esm.sh/lodash-es@4.17"
  }
}
</script>

<script type="module">
import React from 'react';
import { createRoot } from 'react-dom/client';
import { pick } from 'lodash-es';
// No node_modules needed!
</script>
```

**CDN options:**
- **esm.sh** — ESM-compatible npm packages
- **Skypack** — Optimized ESM CDN
- **unpkg** — Raw npm package files
- **jsDelivr** — Fast global CDN

**Use cases:** Quick prototypes, Stackblitz projects, micro-frontends, edge deployments

---

## Common Issues & Troubleshooting

### How do you fix "Module not found" errors?

```bash
# Common causes and fixes:

# 1. Package not installed
npm install missing-package

# 2. Wrong import path (case sensitivity on Linux)
# Wrong:  import Button from './components/button'
# Right:  import Button from './components/Button'

# 3. Missing file extension in some configs
# Wrong:  import utils from './utils'
# Right:  import utils from './utils.js'

# 4. TypeScript path aliases not configured in bundler
# Add to vite.config.ts or webpack.config.js:
resolve: { alias: { '@': './src' } }

# 5. Peer dependency missing
npm install react react-dom  # install peer deps

# 6. ESM vs CJS conflict
# Add to package.json:
{ "type": "module" }  # or remove for CJS
```

---

### How do you handle version conflicts between packages?

```bash
# 1. Identify the conflict
npm ls react        # show all installed versions of react

# 2. Use overrides to force a specific version
# package.json:
{ "overrides": { "react": "18.2.0" } }

# 3. Check peer dependency warnings
npm install --legacy-peer-deps  # skip strict peer dep checking (last resort)

# 4. Deduplicate
npm dedupe          # removes duplicate packages where possible
```

---

### How do you properly clear node_modules and caches?

```bash
# Full clean reinstall
rm -rf node_modules package-lock.json
npm cache clean --force
npm install

# yarn
rm -rf node_modules yarn.lock
yarn cache clean
yarn install

# pnpm
rm -rf node_modules pnpm-lock.yaml
pnpm store prune    # clean global store
pnpm install

# Nuclear option — clear everything
rm -rf node_modules .npm .yarn/cache .pnpm-store
npm cache clean --force
```

```bash
# macOS — free up disk from all node_modules globally
find ~ -name "node_modules" -type d -prune | xargs du -sh | sort -h
# Remove old ones:
npx npkill  # interactive tool to clean node_modules from disk
```

---

### What should your .gitignore contain for node_modules?

```gitignore
# Dependencies
node_modules/
.pnp
.pnp.js
.pnp.loader.mjs

# Yarn v2+ cache (commit if using PnP, ignore if using node-modules linker)
.yarn/cache
.yarn/unplugged
.yarn/build-state.yml
.yarn/install-state.gz

# Lock files — COMMIT THESE! Don't gitignore
# package-lock.json    ← commit
# yarn.lock            ← commit
# pnpm-lock.yaml       ← commit
```

---

## Package Manager Comparison

### npm vs yarn vs pnpm — Comprehensive Comparison

| Feature | npm | Yarn Classic (v1) | Yarn Berry (v2+) | pnpm |
|---|---|---|---|---|
| **Speed** | Moderate | Faster | Fastest | Very Fast |
| **Disk usage** | Large (flat) | Large (flat) | Small (zip cache) | Smallest (symlinks) |
| **Phantom deps** | Possible | Possible | Prevented (PnP) | Prevented (strict) |
| **Workspaces** | Yes (v7+) | Yes | Yes | Yes (best-in-class) |
| **PnP support** | No | No | Yes (default) | Optional |
| **Patch built-in** | No | No | Yes | Yes |
| **Lockfile** | package-lock.json | yarn.lock | yarn.lock | pnpm-lock.yaml |
| **Compatibility** | Universal | Good | Some issues | Very Good |
| **Security** | Good | Good | Good | Excellent (strict) |
| **Learning curve** | Lowest | Low | Medium | Low-Medium |
| **CI caching** | cache ~/.npm | cache ~/.yarn | cache .yarn/cache | cache pnpm store |
| **Monorepo** | Basic | Good | Excellent | Excellent |

---

### patch-package vs forking vs npm overrides — When to use which?

| Approach | When to Use | Pros | Cons |
|---|---|---|---|
| **patch-package** | Quick bug fix, small change | Fast, patches stay in repo, transparent | Breaks on package major version update |
| **Yarn patch/pnpm patch** | Same as patch-package but native | No extra dependency | Yarn/pnpm specific |
| **Fork on GitHub** | Large changes, abandoned package | Full control, can PR upstream | Maintenance burden, drift from upstream |
| **npm overrides** | Force specific version (security) | Simple, native npm feature | Doesn't change code, just version |
| **npm link / workspace** | Local development of a package | No publish needed, instant feedback | Local only, not for CI fixes |
| **Custom npm alias** | Use your fork but with original name | Transparent to consumers | Must publish your fork |

```json
// npm alias example — use your fork with the original package name
{
  "dependencies": {
    "some-package": "npm:@your-org/some-package-fork@^1.2.3"
  }
}
```

---

### What are npm lifecycle hooks and how do you use them?

```json
// package.json lifecycle scripts (run automatically)
{
  "scripts": {
    "preinstall": "echo 'Before npm install'",
    "install": "node scripts/setup.js",
    "postinstall": "patch-package && node scripts/post-install.js",
    
    "prebuild": "npm run clean",
    "build": "tsc && vite build",
    "postbuild": "node scripts/copy-assets.js",
    
    "pretest": "npm run lint",
    "test": "vitest",
    "posttest": "npm run coverage",
    
    "prepare": "husky install",        // runs on npm install
    "prepublishOnly": "npm run test",  // runs before npm publish
    "prepack": "npm run build",        // runs before npm pack
  }
}
```

---

[← Back to README](./README.md)
