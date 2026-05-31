# Secure Hermes Sandbox

Run [NousResearch Hermes Agent](https://github.com/NousResearch/hermes-agent) inside a [Docker Sandbox](https://docs.docker.com/ai/sandboxes/) (`sbx`) microVM, with web access forced through a local sanitizer that strips PII with [Microsoft Presidio](https://microsoft.github.io/presidio/) before forwarding to [self-hosted Firecrawl](https://github.com/firecrawl/firecrawl/blob/main/SELF_HOST.md).

```
Hermes (sbx microVM)
  │  FIRECRAWL_API_URL = http://host.docker.internal:5050
  ▼
sanitizer-proxy  (Presidio analyze + anonymize)
  │
  ▼
Firecrawl
```

LLM credentials stay on the host and are injected through sbx's credential proxy; keys never enter the VM.

## Prerequisites

- macOS with Docker Desktop running
- Docker Sandboxes CLI:
  ```sh
  brew install docker/tap/sbx
  sbx login
  ```
- ~6 GB free disk for the Firecrawl stack

## Quick Start

```sh
./setup.sh
```

This builds the backend (Presidio + Firecrawl + sanitizer), creates the `secure-hermes` sandbox, and drops you into a shell. Type `hermes` to launch the agent.

The workspace defaults to `~/secure-hermes-workspace/` (override with `HERMES_WORKSPACE=...`). It's seeded on first run from the repo's `workspace-template/` — including `AGENTS.md`, which Hermes auto-loads into it's system prompt. Edit `AGENTS.md` in the workspace to add project-specific conventions; seeding never overwrites existing files.

Re-attach later (preserving installs, memory, and history):

```sh
sbx run secure-hermes      # or pick it from the sbx TUI, or re-run ./setup.sh
```

> `sbx rm secure-hermes` wipes Hermes' internal state inside the VM (`~/.hermes`, OAuth tokens, memory, history). Your workspace directory on the host is left alone — neither `sbx rm` nor `docker compose down -v` touch it.

## Credentials

LLM provider keys live in your host keychain and are injected into outbound requests by sbx's [credential proxy](https://docs.docker.com/ai/sandboxes/security/credentials/) — they never enter the VM. Inside the sandbox the corresponding env vars read `proxy-managed`; that is expected.

Any [built-in service](https://docs.docker.com/ai/sandboxes/security/credentials/#built-in-services) (`anthropic`, `openai`, `google`, `groq`, `mistral`, `xai`, `aws`, …) works out of the box:

```sh
sbx secret set -g anthropic   # ANTHROPIC_API_KEY
sbx secret set -g openai      # OPENAI_API_KEY
sbx secret set -g google      # GEMINI_API_KEY / GOOGLE_API_KEY
```

**Global vs sandbox-scoped.** `-g` secrets apply to every sandbox but only bind at sandbox creation — recreate to pick up a new one (`sbx rm secure-hermes && ./setup.sh`). Sandbox-scoped secrets take effect immediately, no recreation needed:

```sh
sbx secret set secure-hermes anthropic
```

**Adding a non-built-in provider** (OpenRouter, self-hosted endpoints, …) requires two changes:

1. **Credential** — use [`sbx secret set-custom`](https://docs.docker.com/ai/sandboxes/security/credentials/#custom-secrets) for an ad-hoc binding, or declare the service in `sandbox/spec.yaml` under `credentials.sources` (see [Kits → Authenticate to external services](https://docs.docker.com/ai/sandboxes/customize/kits/#authenticate-to-external-services)).
2. **Network** — add the provider's domain to `network.allowedDomains` in `sandbox/spec.yaml`.

Recreate the sandbox after either change.

**OAuth.** Run `hermes setup` inside the sandbox and pick Nous Portal.

## Web Search Sanitization

Hermes talks to Firecrawl through `FIRECRAWL_API_URL=http://host.docker.internal:5050`. The proxy intercepts `/v1/*` and `/v2/*`, runs the user-controlled fields through Presidio, and forwards the scrubbed request:

| Route      | Fields scrubbed  |
| ---------- | ---------------- |
| `/search`  | `query`          |
| `/scrape`  | `url`, `prompt`  |
| `/extract` | `urls`, `prompt` |
| `/crawl`   | `url`, `prompt`  |

Default entities: credit cards, API keys, emails, phone numbers, IP addresses, SSNs, IBANs, people, locations.

Hermes is also locked down at the config layer: `web.backend: firecrawl` pins the search backend and `agent.disabled_toolsets: [browser]` removes the in-VM browser tools, so the agent cannot bypass the proxy via an alternative web provider or direct Chromium call.

Every redaction is logged with the original and scrubbed values so you can audit what would have left the VM:

```sh
docker compose logs -f sanitizer-proxy
```

## Network Policy

Web search has it's own dedicated path through the sanitizer proxy (above). Everything else — package managers, code hosts, AI provider APIs, OAuth — leaves the sandbox via sbx's HTTP/HTTPS policy proxy, with three allowlist layers composed together:

1. **Host baseline** — chosen on first `sbx` run. The default _Balanced_ baseline pre-allows package managers (npm, PyPI, crates.io, …), code hosts (GitHub, GitLab, …), and major AI APIs so common workflows just work. _Open_ allows everything; _Locked Down_ denies everything not explicitly allowed. Reset with `sbx policy reset`.
2. **Kit additions** — `sandbox/spec.yaml` → `network.allowedDomains`, applied at sandbox creation. Currently covers the sanitizer proxy and install-time mirrors (Nous Portal, GitHub, PyPI, Debian) so setup works under any baseline.
3. **Host overrides** — `sbx policy allow/deny network`, layered at runtime, machine- or sandbox-scoped.

Deny always beats allow. Inspect the effective policy with `sbx policy ls`; tail allowed/blocked requests with `sbx policy log`.

To allow a new host **permanently**, add it to the kit and recreate the sandbox:

```yaml
# sandbox/spec.yaml
network:
  allowedDomains:
    - 'example.com:443'
    - '*.example.com:443' # *.foo does NOT match apex foo — list both if you need both
```

```sh
sbx rm secure-hermes && ./setup.sh
```

To allow a new host **ad-hoc** (no recreation, scoped to this sandbox):

```sh
sbx policy allow network secure-hermes example.com:443
sbx policy deny  network secure-hermes telemetry.example.com
```

Only HTTP/HTTPS go through the policy proxy. Raw TCP (e.g. SSH) can be allowed by IP:port (`sbx policy allow network -g 1.2.3.4:22`); UDP and ICMP are blocked at the kernel and cannot be unblocked.

## Endpoints

- Sanitizer health: <http://localhost:5050/health>
- Firecrawl API: <http://localhost:3002>

Port 5050 is used on the host because macOS AirPlay binds 5000.

## Teardown

```sh
docker compose down        # stop backend
sbx rm secure-hermes       # destroy sandbox
docker compose down -v     # also drop backend volumes
```

## Troubleshooting

- **`agent "hermes" not found`** — Hermes is not an sbx built-in. Use `./setup.sh` (or `sbx run --name secure-hermes --kit ./sandbox shell`).
- **Search returns 404** — rebuild the proxy: `docker compose up -d --build sanitizer-proxy`.
- **Provider call returns 401** — set the host secret (`sbx secret set -g <provider>`) and recreate the sandbox.
- **Weak/no search results** — set `SEARXNG_ENDPOINT` in `.env`; otherwise Firecrawl falls back to rate-limited Google scraping.
