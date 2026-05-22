# 00 — STATE

## Repository Identity
- **Name**: secure-hermes-sandbox
- **Owner/Origin**: sebbmn (upstream), okwn (fork clone)
- **URL**: https://github.com/sebbmn/secure-hermes-sandbox
- **Fork**: yes (okwn/secure-hermes-sandbox)
- **License**: NONE (no license file present)
- **Archived**: false
- **Language**: TypeScript (sanitizer proxy), Shell (setup.sh), YAML (compose/spec)

## Upstream Status
- Upstream has **0 open issues**, **0 open PRs**
- Fork is even with upstream (same commit history, fork-of-fork from okwn)
- Last push: 2026-05-19

## Local Clone
- Path: `/root/oss-pr-campaign/repos/secure-hermes-sandbox`
- Branch: `main`
- Remotes: `origin` (okwn fork), `upstream` (sebbmn original)

## Sandbox Kit Compatibility
- `sandbox/spec.yaml` kind: `mixin` — requires Docker Sandboxes CLI (`sbx`) and the built-in `shell` agent
- Not macOS-exclusive at the file level, but `sbx` CLI only runs on macOS with Docker Desktop

## Test Baseline
- **TypeScript**: `npm run typecheck` → ✅ PASS (0 errors)
- **Build**: `npm run build` → ✅ PASS (compiles to dist/)
- **Tests**: ❌ NONE FOUND — no test suite present

## Issues Found
1. No license — default copyright applies, unclear what OSS usage is permitted
2. No test suite — no unit/integration tests for sanitizer proxy or setup script
3. No CI workflow — no GitHub Actions for lint/typecheck/build/test
4. PII redaction logs use `warn()` level (console.warn) even at INFO log level — may cause noise in logs
5. No `.env` file committed — only `.env.example`; first-run requires manual setup