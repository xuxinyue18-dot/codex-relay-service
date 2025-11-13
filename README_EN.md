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

See `.env.example` and `config/config.example.js` for defaults and details.

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

