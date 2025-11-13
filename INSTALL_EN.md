# Quick Deployment Guide (English)

This guide helps you deploy Codex Relay Service from scratch with clear, unambiguous steps.

## 1. Prerequisites
- Node.js ≥ 18 (LTS recommended)
- Redis ≥ 6 available
- Linux/macOS or Docker

### Termux notes (Android)
- Paths and working directory: Termux home is `~/` (absolute: `/data/data/com.termux/files/home`).
  - Example: `cd ~/codex-relay-service`
- Node & Redis: install via `pkg` (e.g., `pkg install nodejs-lts redis`), or point to an external Redis by changing `REDIS_HOST` in `.env`.
- Docker: not available on Termux by default — prefer the native Node workflow (skip the Docker section below).
- Ports: default 3000 is reachable locally; to access from outside, use port forwarding/VPN/reverse proxy as appropriate.
- Keep process awake (optional): `termux-wake-lock` if you need long-running services.

## 2. Get the project
```bash
git clone git@github.com:xuxinyue18-dot/codex-relay-service.git
cd codex-relay-service   # On Termux this equals /data/data/com.termux/files/home/codex-relay-service
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

Sample docker-compose:
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
      - NODE_ENV=production
      - PORT=3000
      - HOST=0.0.0.0
      - JWT_SECRET=${JWT_SECRET}
      - ENCRYPTION_KEY=${ENCRYPTION_KEY}
      - API_KEY_PREFIX=${API_KEY_PREFIX:-cr_}
      - WEB_SESSION_SECRET=${WEB_SESSION_SECRET:-CHANGE_ME_SESSION_SECRET}
      - ADMIN_USERNAME=${ADMIN_USERNAME:-}
      - ADMIN_PASSWORD=${ADMIN_PASSWORD:-}
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - REDIS_PASSWORD=${REDIS_PASSWORD:-}
      - REDIS_DB=${REDIS_DB:-0}
      - GEMINI_OAUTH_CLIENT_ID=${GEMINI_OAUTH_CLIENT_ID:-}
      - GEMINI_OAUTH_CLIENT_SECRET=${GEMINI_OAUTH_CLIENT_SECRET:-}
      - DEFAULT_PROXY_TIMEOUT=${DEFAULT_PROXY_TIMEOUT:-600000}
      - MAX_PROXY_RETRIES=${MAX_PROXY_RETRIES:-3}
      - PROXY_USE_IPV4=${PROXY_USE_IPV4:-true}
      - REQUEST_TIMEOUT=${REQUEST_TIMEOUT:-600000}
      - DEFAULT_TOKEN_LIMIT=${DEFAULT_TOKEN_LIMIT:-1000000}
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
