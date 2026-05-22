# 01 — REPO MAP

## File Tree
```
secure-hermes-sandbox/
├── .env.example                 # Host-side config template (SearXNG, logging level)
├── .gitignore                   # Ignores dist/, node_modules/, .env
├── README.md                    # Full project documentation (146 lines)
├── docker-compose.yml           # Full stack: Presidio-analyzer, Presidio-anonymizer,
│                                #   Firecrawl (redis/rabbitmq/postgres/playwright),
│                                #   sanitizer-proxy
├── sandbox/
│   └── spec.yaml                # Docker Sandbox kit (mixin for `shell` agent);
│                                #   network allowlist, Hermes config YAML, install command
├── sanitizer_proxy/
│   ├── .dockerignore
│   ├── Dockerfile               # Builds sanitizer-proxy from dist/ (Node.js)
│   ├── package.json             # TypeScript/ESM, hono + @hono/node-server
│   ├── tsconfig.json            # ES2022, NodeNext, strict
│   ├── src/
│   │   └── server.ts            # Main proxy: Presidio scrub + Firecrawl forward (305 lines)
│   └── dist/                    # Compiled JS output (gitignored)
├── setup.sh                     # One-shot setup: workspace + backend + sandbox launch (247 lines)
└── workspace-template/
    └── AGENTS.md                # Injected into workspace on first run; documents
                                 #   sandbox constraints for the Hermes agent
```

## Key Code Components

### sanitizer_proxy/src/server.ts
Core PII-scrubbing HTTP proxy. Handles `/v1/search`, `/v1/scrape`, `/v1/extract`, `/v1/crawl` and same for v2.
- Scrub fields: query, url, prompt, urls
- Entities: CREDIT_CARD, EMAIL_ADDRESS, PHONE_NUMBER, US_SSN, IBAN_CODE, IP_ADDRESS, PERSON, LOCATION, API_KEY
- Custom API_KEY recognizer for `sk-` and `sk-ant-` prefixed secrets
- Uses Microsoft Presidio (analyzer → anonymizer) for redaction
- Logs every redaction with original + scrubbed value at WARN level
- Timeout: 30s default, configurable via REQUEST_TIMEOUT_SECONDS

### docker-compose.yml
7 services on `secure-search-net` bridge:
1. `presidio-analyzer` — PII detection (port 5001→3000)
2. `presidio-anonymizer` — PII redaction (port 5002→3000)
3. `firecrawl-redis` — cache/queue
4. `firecrawl-rabbitmq` — job queue
5. `firecrawl-postgres` — persistence (no auth)
6. `firecrawl-playwright` — headless browser scraping
7. `firecrawl` — main API (port 3002)
8. `sanitizer-proxy` — PII scrubber (port 5050→5000), built from `./sanitizer_proxy`

### sandbox/spec.yaml
Docker Sandbox kit (mixin on `shell`):
- Allowed domains: host.docker.internal:5050, localhost:5050, Nous Portal/API, GitHub, PyPI, Debian mirrors
- `FIRECRAWL_API_URL=http://host.docker.internal:5050`
- Hermes config (`~/.hermes/config.yaml`): terminal backend=local, browser toolset disabled, web backend=firecrawl
- Install: `curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash`

### setup.sh
- Validates Docker, docker compose, sbx CLI
- Interactive workspace picker (default `~/secure-hermes-workspace/`)
- Seeds workspace from `workspace-template/` (preserves existing files)
- Creates `.env` from `.env.example` if missing
- Brings up full docker compose stack (backend)
- Polls sanitizer proxy health (up to 120s) and Firecrawl (best-effort)
- Validates kit with `sbx kit validate`
- Creates or attaches `secure-hermes` sandbox

## CI / Automation
None found — no GitHub Actions, no GitHub workflows directory.

## Dependencies
- `sanitizer_proxy/package.json`: hono, @hono/node-server (latest), TypeScript (dev)
- `docker-compose.yml`: Microsoft Presidio images, Redis, RabbitMQ, Firecrawl (ghcr.io), Playwright
- `sandbox/spec.yaml`: Hermes Agent (install script from NousResearch/hermes-agent)