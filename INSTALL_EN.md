# Quick Deployment Guide (English)

This guide helps you deploy Codex Relay Service from scratch with clear, unambiguous steps.

## 1. Prerequisites
- Node.js ≥ 18 (LTS recommended)
- Redis ≥ 6 available
- Linux/macOS or Docker

## 2. Get the project
```bash
git clone git@github.com:xuxinyue18-dot/codex-relay-service.git
cd codex-relay-service
```

## 3. Configuration
- Copy and fill environment variables
```bash
cp .env.example .env
```
- Copy and adjust app config
```bash
cp config/config.example.js config/config.js
```

Check these first:
- Required: `JWT_SECRET` (≥ 32 chars), `ENCRYPTION_KEY` (32 chars)
- Recommended: `WEB_SESSION_SECRET`
- Redis: `REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD` (if any), `REDIS_DB`
- Proxy (optional): `DEFAULT_PROXY_TIMEOUT`, `MAX_PROXY_RETRIES`, `PROXY_USE_IPV4`
- Gemini (optional): `GEMINI_OAUTH_CLIENT_ID`, `GEMINI_OAUTH_CLIENT_SECRET`

All variables are documented in `.env.example`.

## 4. Install and initialize
```bash
npm install
npm run setup
```

## 5. (Optional) build Admin UI
```bash
npm run install:web
npm run build:web   # output at web/admin-spa/dist
```

## 6. Start service
```bash
npm run service:start    # background process
# or development mode
npm run dev
```

## 7. Access and verification
- Admin UI (if built): `http://<server-ip-or-domain>:3000/admin-next/`
- Health: `http://<server-ip-or-domain>:3000/health`

## 8. Troubleshooting
- Redis connection: verify `REDIS_HOST/PORT/PASSWORD` and networking
- Admin 404: run `npm run build:web` first
- Proxy issues: verify protocol/address/timeouts; enable `PROXY_USE_IPV4` if needed

## 9. Docker (optional)
```bash
npm run docker:build
npm run docker:up
```
Provide `JWT_SECRET`, `ENCRYPTION_KEY`, and other required variables via environment in `docker-compose.yml`.

