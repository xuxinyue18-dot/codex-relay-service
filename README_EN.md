# Codex Relay Service

Lightweight and extensible AI relay with multi‑vendor backends. It is OpenAI‑compatible out of the box (/v1/chat/completions, etc.) and supports Claude, OpenAI, Gemini, Azure OpenAI, and AWS Bedrock. It provides multi‑account scheduling, API key auth, usage/cost tracking, rate limiting, proxying, webhooks, and an optional Admin UI.

This is the consolidated English guide (merged and de‑duplicated from the Chinese README and CLAUDE.md) aligned with the current implementation.

## Features

- Backends: Claude / OpenAI / Gemini / Azure OpenAI / Bedrock
- Unified protocol: OpenAI‑compatible APIs + unified route `/v1/unified/chat/completions`
- Multi‑account scheduling: permission/rate/session‑aware allocation with graceful fallback
- Safety & control: API key auth, global/endpoint rate limits, request size guard, model blocklist
- Observability & cost: usage and cost tracking, pricing updater, webhooks
- Proxy support: HTTP/HTTPS/SOCKS5, account‑level proxies, IPv4/IPv6 toggle
- Admin UI: `/admin-next` (optional build in `web/admin-spa`)

## Requirements

- Node.js >= 18
- Redis >= 6 (sessions, queues, metrics)
- Linux/macOS/Docker

Note: Termux (Android) users — see the "Termux notes (Android)" section in INSTALL_EN.md for path, Redis, and runtime tips.

## Quick Start

1) Environment variables
- Copy `.env.example` to `.env` and set at minimum:
  - `PORT`, `HOST`
  - `JWT_SECRET`, `ENCRYPTION_KEY`
  - `REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD` (if applicable)
  - More options: `config/config.example.js`

2) Install and initialize
```bash
npm install
npm run setup            # initialize required data if applicable
npm run install:web      # optional: install Admin SPA deps
npm run build:web        # optional: build Admin SPA to web/admin-spa/dist
```

3) Run
```bash
npm run dev              # development (nodemon)
# or
npm run service:start    # start via scripts/manage.js
npm run service:status   # check status
npm run service:logs     # tail logs
```

4) Docker (optional)
```bash
npm run docker:build
npm run docker:up        # docker-compose up -d
```

Sample docker-compose snippet (with common environment variables):
```yaml
version: '3.8'
services:
  codex-relay:
    build: .
    image: codex-relay-service:latest
    restart: unless-stopped
    ports:
      - "0.0.0.0:${PORT:-3000}:3000"
    environment:
      # Basics
      - NODE_ENV=production
      - PORT=3000
      - HOST=0.0.0.0

      # Security (required)
      - JWT_SECRET=${JWT_SECRET}
      - ENCRYPTION_KEY=${ENCRYPTION_KEY}
      - API_KEY_PREFIX=${API_KEY_PREFIX:-cr_}
      - WEB_SESSION_SECRET=${WEB_SESSION_SECRET:-CHANGE_ME_SESSION_SECRET}

      # Admin (optional)
      - ADMIN_USERNAME=${ADMIN_USERNAME:-}
      - ADMIN_PASSWORD=${ADMIN_PASSWORD:-}

      # Redis
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - REDIS_PASSWORD=${REDIS_PASSWORD:-}
      - REDIS_DB=${REDIS_DB:-0}

      # Gemini OAuth (optional)
      - GEMINI_OAUTH_CLIENT_ID=${GEMINI_OAUTH_CLIENT_ID:-}
      - GEMINI_OAUTH_CLIENT_SECRET=${GEMINI_OAUTH_CLIENT_SECRET:-}

      # Proxy & limits (optional)
      - DEFAULT_PROXY_TIMEOUT=${DEFAULT_PROXY_TIMEOUT:-600000}
      - MAX_PROXY_RETRIES=${MAX_PROXY_RETRIES:-3}
      - PROXY_USE_IPV4=${PROXY_USE_IPV4:-true}
      - REQUEST_TIMEOUT=${REQUEST_TIMEOUT:-600000}
      - DEFAULT_TOKEN_LIMIT=${DEFAULT_TOKEN_LIMIT:-1000000}

      # Logs (optional)
      - LOG_LEVEL=${LOG_LEVEL:-info}
      - LOG_MAX_SIZE=${LOG_MAX_SIZE:-10m}
      - LOG_MAX_FILES=${LOG_MAX_FILES:-5}

    volumes:
      - ./logs:/app/logs
      - ./data:/app/data
    depends_on:
      - redis

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    expose:
      - "6379"
    volumes:
      - ./redis_data:/data
    command: redis-server --save 60 1 --appendonly yes --appendfsync everysec
```

## API Usage

- OpenAI‑compatible example (auth: `Authorization: Bearer <YOUR_API_KEY>`)
```bash
curl -X POST http://<host>:<port>/v1/chat/completions \
  -H 'Authorization: Bearer cr_xxx' \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "Hello"}],
    "stream": false
  }'
```
- Unified route (auto‑select Claude/OpenAI/Gemini backend): `POST /v1/unified/chat/completions`
- Key permissions affect accessible backends (`all`/`openai`/`claude`/`gemini`)
- Supports SSE streaming (`stream: true`)

## Admin & Scripts

- Admin UI: `/admin-next/` (build the SPA first)
- Handy scripts:
  - `npm run service:start|stop|restart|status|logs`
  - `npm run update:pricing` (update model pricing)
  - `npm run data:export|import` (Redis data migration)
  - `npm run init:costs` (initialize costs)
  - `npm run monitor`, `npm run status:detail`

## Configuration Overview

- Basics: `PORT`, `HOST`, `NODE_ENV`, `TRUST_PROXY`
- Security: `JWT_SECRET`, `API_KEY_PREFIX`, `ENCRYPTION_KEY`, session timeout
- Redis: `REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD`, `REDIS_DB`, `REDIS_ENABLE_TLS`
- Session: `STICKY_SESSION_TTL_HOURS`, `STICKY_SESSION_RENEWAL_THRESHOLD_MINUTES`
- Claude: `CLAUDE_API_URL`, `CLAUDE_API_VERSION`, `CLAUDE_OVERLOAD_HANDLING_MINUTES`
- Bedrock: `CLAUDE_CODE_USE_BEDROCK`, `AWS_REGION`, `ANTHROPIC_MODEL`, etc.
- Proxy: `DEFAULT_PROXY_TIMEOUT`, `MAX_PROXY_RETRIES`, `PROXY_USE_IPV4`
- Limits: `REQUEST_TIMEOUT`, `DEFAULT_TOKEN_LIMIT`
- Logs: `LOG_LEVEL`, `LOG_MAX_SIZE`, `LOG_MAX_FILES`
- Web: `WEB_TITLE`, `WEB_DESCRIPTION`, `ENABLE_CORS`
- LDAP: `LDAP_ENABLED`, `LDAP_URL`, certificates and attribute mapping
- Users: `USER_MANAGEMENT_ENABLED`, `MAX_API_KEYS_PER_USER`
- Webhook: `WEBHOOK_ENABLED`, `WEBHOOK_URLS`, `WEBHOOK_RETRIES`

See [`.env.example`](./.env.example) and [`config/config.example.js`](./config/config.example.js) for defaults and details.

## Environment Variables (must‑check)

- Required
  - `JWT_SECRET`: JWT secret (recommend ≥ 32 chars random)
  - `ENCRYPTION_KEY`: 32‑char encryption key
- Recommended
  - `WEB_SESSION_SECRET`: web session secret
- Redis connection
  - `REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD` (if any), `REDIS_DB`
- Proxy (optional)
  - `DEFAULT_PROXY_TIMEOUT`, `MAX_PROXY_RETRIES`, `PROXY_USE_IPV4`
- Gemini OAuth (optional, if you use Gemini accounts)
  - `GEMINI_OAUTH_CLIENT_ID`
  - `GEMINI_OAUTH_CLIENT_SECRET`

All variables have placeholders in `.env.example`. Copy it to `.env` and fill accordingly.

### Minimal .env example (copy, then adjust)

```env
# Basics
NODE_ENV=production
HOST=0.0.0.0
PORT=3000

# Security (required)
JWT_SECRET=please-change-to-a-random-string-at-least-32-chars
ENCRYPTION_KEY=32-characters-long-encryption-secret-please-change
WEB_SESSION_SECRET=change-me-session-secret
API_KEY_PREFIX=cr_

# Redis (adjust to your environment)
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_DB=0
```

## Deployment Checklist (ready to run)

1) Clone and enter the project
```bash
git clone git@github.com:xuxinyue18-dot/codex-relay-service.git
cd codex-relay-service
```

2) Prepare configuration
```bash
cp .env.example .env                   # fill variables listed above
cp config/config.example.js config/config.js
```

3) Install and initialize
```bash
npm install
npm run setup
```

4) Optional: build Admin UI
```bash
npm run install:web
npm run build:web    # output under web/admin-spa/dist
```

5) Start service
```bash
npm run service:start   # or npm run dev for development
```

## Architecture Overview

- Unified schedulers: `unifiedClaudeScheduler`, `unifiedGeminiScheduler`, `unifiedOpenAIScheduler`, `droidScheduler`
- Account types: `claude-official`, `claude-console`, `bedrock`, `ccr`, `droid`, `gemini`, `openai-responses`, `azure-openai`
- Auth chain: self‑issued API key → validation & rate limits → unified scheduling → account tokens → proxy forwarding
- Token lifecycle: expiry detection + auto refresh (can refresh ahead of time via account‑level proxy)
- Data security: sensitive credentials are AES‑encrypted and stored in Redis
- Sticky session: account pinning per session to preserve context
- Guardrails: key permissions, client identification (User‑Agent), model blocklist

### Main Components

- Relay services: `claudeRelayService`, `claudeConsoleRelayService`, `geminiRelayService`,
  `bedrockRelayService`, `azureOpenaiRelayService`, `droidRelayService`, `ccrRelayService`,
  `openaiResponsesRelayService`
- Account services: `claudeAccountService`, `claudeConsoleAccountService`, `geminiAccountService`,
  `bedrockAccountService`, `azureOpenaiAccountService`, `droidAccountService`, `ccrAccountService`,
  `openaiResponsesAccountService`, `openaiAccountService`, `accountGroupService`
- Schedulers: `unifiedClaudeScheduler`, `unifiedGeminiScheduler`, `unifiedOpenAIScheduler`, `droidScheduler`
- Core services: `apiKeyService`, `userService`, `pricingService`, `costInitService`, `webhookService`,
  `webhookConfigService`, `ldapService`, `tokenRefreshService`, `rateLimitCleanupService`, `claudeCodeHeadersService`
- Utilities: `oauthHelper` (PKCE + proxy), `workosOAuthHelper`, `openaiToClaude`

### Request Flow (TL;DR)

1) Client calls `/api`, `/claude`, `/gemini`, `/openai`, or `/droid` with a `cr_` API key
2) `authenticateApiKey` validates key, rate limits, permissions, client and model constraints
3) Scheduler picks an account based on model/session/permissions; falls back when needed
4) Check/refresh the account token (via configured proxy)
5) Forward request using account credentials (not the client’s key) and the account’s proxy
6) Stream/return upstream response; record usage and cost; update rate/concurrency counters

## Logging & Troubleshooting

- App logs rotate by size/count under `logs/`
- HTTP debug: set `DEBUG_HTTP_TRAFFIC=true` to write `logs/http-debug-*.log`
- Common issues
  - Redis connection: verify `REDIS_HOST/PORT/PASSWORD` and network reachability
  - Proxy errors: confirm protocol/address/timeouts; toggle `PROXY_USE_IPV4` if needed
  - Admin 404: ensure `npm run build:web` was executed

## Security & Compliance

- `.gitignore` excludes logs/data/dumps/keys and other sensitive artifacts
- Never commit production secrets; use `.env.example` and `web/*/.env.example` as templates
- For learning and research only; follow all provider terms of service

## License

MIT (see `LICENSE`)
