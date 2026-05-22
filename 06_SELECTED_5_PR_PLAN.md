# 06 — SELECTED 5 PR PLAN

## Selected PRs (by priority)
1. **Add MIT License** — legal foundation for open contribution
2. **Add GitHub Actions CI** — catch regressions on every push
3. **Add Unit Tests for Sanitizer Proxy** — cover the core PII-scrubbing component
4. **Fix `info()` Logging Bug** — prevent noisy logs under INFO log level
5. **Add `.env` Explicitly to `.gitignore`** — prevent credential accidents

---

## PR 1: Add MIT License
**File**: `LICENSE` (new)

### Rationale
No OSS license = default copyright. An MIT license is the standard for developer tools and removes all ambiguity about permitted use and contribution.

### Implementation
```
MIT License

Copyright (c) 2026-present, sebbmn and contributors

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

Add to README after title: `[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)`

---

## PR 2: Add GitHub Actions CI
**File**: `.github/workflows/ci.yml` (new)

### Rationale
No CI exists. TypeScript type-checks pass today, but any future regression goes undetected.

### Implementation
```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  sanitizer-proxy:
    name: Sanitizer Proxy (Node.js)
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: sanitizer_proxy
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'
          cache-dependency-path: sanitizer_proxy/package-lock.json
      - run: npm ci
      - run: npm run typecheck
      - run: npm run build
```

### Verification
- `npm run typecheck` → 0 errors
- `npm run build` → dist/ produced
- No Docker required for TypeScript layer

---

## PR 3: Add Unit Tests for Sanitizer Proxy
**Files**: `sanitizer_proxy/src/server.test.ts` (new), `sanitizer_proxy/package.json` update

### Rationale
The sanitizer proxy is the security-critical component. No tests exist. Adding vitest-based unit tests covers the core redaction logic.

### Implementation
Add to `sanitizer_proxy/package.json`:
```json
"scripts": {
  "test": "vitest run",
  "test:watch": "vitest"
},
"devDependencies": {
  "vitest": "^2.0.0"
}
```

Tests to write:
```typescript
// scrubString with PII → redaction
// scrubString with clean text → unchanged
// scrubField skips null/undefined/non-string
// scrubField handles string arrays
// isAbortError correctly identifies timeout
// health endpoint returns { status: "ok" }
// 400 for non-JSON body
// scrubbedRoutes covers search/scrape/extract/crawl
```

### Verification
`npm test` → all tests pass

---

## PR 4: Fix `info()` Logging Bug
**File**: `sanitizer_proxy/src/server.ts` — `info()` function (line 88-92)

### Rationale
Bug: `info()` suppresses output only when logLevel is ERROR or WARN, but info messages should print when logLevel is INFO or DEBUG. Under the default `INFO` log level, `info()` calls silently do nothing, making the health-check and scan-log messages disappear.

### Current (buggy):
```ts
function info(message: string, ...args: unknown[]): void {
  if (!["ERROR", "WARN", "WARNING"].includes(logLevel)) {
    console.log(message, ...args);
  }
}
```

### Fix:
```ts
function info(message: string, ...args: unknown[]): void {
  if (["INFO", "DEBUG"].includes(logLevel)) {
    console.log(message, ...args);
  }
}
```

### Verification
- Before fix: `LOG_LEVEL=INFO node dist/server.js` → health check logs disappear
- After fix: health check and scan endpoint logs appear at INFO level

---

## PR 5: Add `.env` Explicitly to `.gitignore`
**File**: `.gitignore`

### Rationale
`.env` is created at setup time from `.env.example` but is not in `.gitignore`. If a user populates `.env` with live Firecrawl credentials and accidentally commits it, secrets could leak. Adding `.env` to `.gitignore` is a belt-and-suspenders measure.

### Current `.gitignore`:
```
dist/
node_modules/
package-lock.json
```

### Fix:
```
dist/
node_modules/
.env
```

### Verification
`echo "SECRET=123" >> .env && git status` → `.env` not shown as untracked

---

## Execution Order
1. **PR 1** (License) — independent, no code changes, safest to merge first
2. **PR 2** (CI) — infrastructure, validates build pipeline
3. **PR 3** (Tests) — quality, adds test coverage
4. **PR 4** (Bug fix) — correctness, one-line fix
5. **PR 5** (gitignore) — safety, trivial change

Each PR is independent and can be reviewed and merged separately without conflicts.