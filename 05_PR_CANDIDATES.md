# 05 — PR CANDIDATES

## Rationale
The repo has 0 issues and 0 PRs. With no existing community activity, candidates focus on infrastructure quality, security hardening, and operational robustness — improvements any maintainer would welcome without debate.

---

## Candidate 1: Add MIT License
**File**: `LICENSE` (new)

**Why**: The repo has `license: null` — no OSS license declared. Default copyright law applies, making the repo effectively proprietary. An MIT license is the standard choice for developer tooling and enables open contribution.

**Change**: Add a `LICENSE` file with MIT terms, update README badge.

---

## Candidate 2: Add GitHub Actions CI Workflow
**File**: `.github/workflows/ci.yml` (new)

**Why**: The project has no CI. TypeScript compiles cleanly today, but any future regression would go unnoticed. A minimal CI gate prevents that.

**Change**: On push/PR to `main`:
- Checkout + Node.js 22
- `cd sanitizer_proxy && npm ci`
- `npm run typecheck`
- `npm run build`
- (Docker Compose validation is out of scope for a fork PR, but compose config validation could be added)

---

## Candidate 3: Add Unit Tests for Sanitizer Proxy
**Files**: `sanitizer_proxy/src/server.test.ts` (new), `sanitizer_proxy/package.json` (update)

**Why**: No tests exist. The proxy is the core security component — at minimum `scrubString`, `scrubField`, `anonymize`, and the route handlers need test coverage.

**Change**: Add `vitest` as devDependency, write tests for:
- `scrubString`: PII redaction (email, SSN, API key patterns)
- `scrubString`: no-op when no PII present
- `scrubField`: skips non-string/non-array fields
- Route handler: returns 400 for non-JSON body
- Health endpoint returns `{ status: "ok" }`
- `isAbortError` correctly identifies timeout errors

---

## Candidate 4: Fix Verbose `info()` Logging Always Emitting
**File**: `sanitizer_proxy/src/server.ts` — `info()` function

**Why**: The `info()` function only suppresses output for `ERROR` log level, not `WARN`. Since `WARN`-level redaction logs (the `warn()` calls) always fire and include the original PII value as a console arg, they can leak to stdout under default `INFO` logging. The condition `!["ERROR", "WARN", "WARNING"].includes(logLevel)` is backwards — info messages should only print when logLevel is INFO or DEBUG.

**Current (buggy)**:
```ts
function info(message: string, ...args: unknown[]): void {
  if (!["ERROR", "WARN", "WARNING"].includes(logLevel)) {
    console.log(message, ...args);
  }
}
```
**Fix**: `if (["INFO", "DEBUG"].includes(logLevel))`

---

## Candidate 5: Add `.env` to `.gitignore` Explicitly
**File**: `.gitignore`

**Why**: While `dist/` and `node_modules/` are gitignored, `.env` is not listed. The project already creates `.env` from `.env.example` at setup time, but adding `.env` explicitly prevents accidental commits of live credentials.

**Change**: Add `.env` line to `.gitignore`.

---

## Candidate 6: Add Docker Healthcheck to Sanitizer Proxy Dockerfile
**File**: `sanitizer_proxy/Dockerfile`

**Why**: The `docker-compose.yml` healthcheck for `sanitizer-proxy` relies on a node inline script. Adding a proper `HEALTHCHECK` instruction to the Dockerfile itself makes the image self-describing and usable outside compose.

**Change**: Add `HEALTHCHECK --interval=5s --timeout=3s --retries=20 CMD node -e "fetch('http://localhost:5000/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"` to Dockerfile.

---

## Candidate 7: Add `package-lock.json` to `.gitignore` Cleanup
**File**: `.gitignore` — remove `package-lock.json` from gitignore, add to git

**Why**: `package-lock.json` is currently gitignored. For a multi-component project, the lock file should be committed so `npm ci` produces deterministic installs in CI and across contributors. Only `sanitizer_proxy/package-lock.json` matters (no root lockfile).

**Change**: Remove `package-lock.json` from `.gitignore`, commit the existing lock file under `sanitizer_proxy/`.

---

## Candidate 8: Add `network.allowedDomains` Entry for Firecrawl
**File**: `sandbox/spec.yaml`

**Why**: The sandbox only allows `host.docker.internal:5050` for web access, but Firecrawl itself (running as a Docker Compose service on the host network) also needs to reach external hosts (e.g., for crawling). If Firecrawl makes outbound HTTP calls (e.g., to model providers for extraction), they may be blocked by the sandbox network policy. However, Firecrawl's actual egress is from the **host** network interface (not the VM), so this may not apply. Worth auditing and potentially documenting.

**Change**: Audit and clarify in README; possibly add `api.firecrawl.dev` or model provider domains if needed.

---

## Candidate 9: Add `sbx kit validate` to CI
**File**: `.github/workflows/ci.yml`

**Why**: The `setup.sh` script calls `sbx kit validate "$KIT_PATH"`. If the kit spec is malformed, setup fails at runtime. Adding kit validation to CI catches spec errors early.

**Note**: Requires `sbx` CLI in the CI environment. The `sbx` CLI is macOS-only, so this would only run on a future macOS runner or as a separate job.

---

## Candidate 10: Harden `setup.sh` Error Handling for `poll_url` Timeout
**File**: `setup.sh` — `poll_url` function

**Why**: If `poll_url` exhausts retries, the function returns 1 but `set -e` may not propagate it cleanly in all zsh contexts. Also, the function name conflicts with the Docker service name "sanitizer-proxy" in log output.

**Change**: Rename to `wait_for_url`, improve error message clarity, add retry counter to abort message.

---

## Summary Table

| # | Candidate | Type | Effort | Impact |
|---|-----------|------|--------|--------|
| 1 | Add MIT License | legal/infra | low | high |
| 2 | GitHub Actions CI | infra | low | high |
| 3 | Unit tests (vitest) | quality | medium | high |
| 4 | Fix info() logging bug | bug | low | medium |
| 5 | Explicit `.env` in gitignore | safety | low | medium |
| 6 | Dockerfile HEALTHCHECK | ops | low | medium |
| 7 | Commit package-lock.json | reproducibility | low | medium |
| 8 | Firecrawl network audit | docs/ops | low | low |
| 9 | sbx kit validate in CI | infra | low | medium |
| 10 | poll_url error handling | robustness | low | low |