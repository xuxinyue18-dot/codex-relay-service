# 快速部署指南（中文）

本指南帮助你从零开始部署 Codex Relay Service，确保步骤清晰、无歧义。

## 1. 环境准备
- Node.js ≥ 18（建议 LTS）
- 可用的 Redis ≥ 6
- Linux/macOS 或 Docker 环境

### Termux 使用提示（Android）
- 路径与工作目录：Termux 的家目录为 `~/`（绝对路径：`/data/data/com.termux/files/home`）。
  - 例如：`cd ~/codex-relay-service`
- Node 与 Redis：可通过 `pkg` 安装（如：`pkg install nodejs-lts redis`），或连接到远端/外部 Redis（在 `.env` 里改 `REDIS_HOST`）。
  - 本地 Redis 启动示例：`redis-server --daemonize yes`（Termux 无 systemd）。
- Docker：Termux 默认不可用 Docker，建议使用“原生 Node 方式”运行（下面步骤 4 的 Docker 可跳过）。
- 端口：默认 3000 本地可访问；若需外网访问，需额外端口映射/内网穿透/反向代理。
- 保持进程不休眠（可选）：`termux-wake-lock` 在需要长时间运行时保持唤醒。

## 2. 获取项目
```bash
git clone git@github.com:xuxinyue18-dot/codex-relay-service.git
cd codex-relay-service   # Termux 下等价于 /data/data/com.termux/files/home/codex-relay-service
```

## 3. 配置文件
- 复制并填写环境变量
```bash
cp .env.example .env
```
- 复制并调整应用配置
```bash
cp config/config.example.js config/config.js
```

建议优先检查以下关键变量：
- 必填：`JWT_SECRET`（≥32字符）、`ENCRYPTION_KEY`（32字符）
- 推荐：`WEB_SESSION_SECRET`
- Redis：`REDIS_HOST`、`REDIS_PORT`、`REDIS_PASSWORD`（如需）、`REDIS_DB`
- 代理（可选）：`DEFAULT_PROXY_TIMEOUT`、`MAX_PROXY_RETRIES`、`PROXY_USE_IPV4`
- Gemini（可选）：`GEMINI_OAUTH_CLIENT_ID`、`GEMINI_OAUTH_CLIENT_SECRET`

全部变量见 `.env.example` 对应注释。

### 最小 .env 示例（可直接复制粘贴后修改）

```env
# 基础
NODE_ENV=production
HOST=0.0.0.0
PORT=3000

# 安全（必填）
JWT_SECRET=please-change-to-a-random-string-at-least-32-chars
ENCRYPTION_KEY=32-characters-long-encryption-secret-please-change
WEB_SESSION_SECRET=change-me-session-secret
API_KEY_PREFIX=cr_

# Redis（按你的环境调整）
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_DB=0
```

## 4. 安装与初始化
```bash
npm install
npm run setup
```

## 5.（可选）构建管理后台
```bash
npm run install:web
npm run build:web   # 产物在 web/admin-spa/dist
```

## 6. 启动服务
```bash
npm run service:start    # 后台运行
# 或开发模式
npm run dev
```

## 7. 访问与验证
- 管理界面（如已构建）：`http://<服务器IP或域名>:3000/admin-next/`
- 健康检查：`http://<服务器IP或域名>:3000/health`

## 8. 常见问题
- Redis 连接失败：检查 `REDIS_HOST/PORT/PASSWORD` 与网络
- 管理后台 404：先执行 `npm run build:web`
- 代理问题：确认代理协议/地址/超时；必要时开启 `PROXY_USE_IPV4`

## 9. Docker 方式（可选）
```bash
npm run docker:build
npm run docker:up
```
在 `docker-compose.yml` 中通过环境变量传入 `JWT_SECRET`、`ENCRYPTION_KEY` 等必填项。

示例 docker-compose：
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

---

## 常见错误速查（Cheat Sheet）

- 现象：启动时报 Redis 连接失败（ECONNREFUSED/ETIMEDOUT）
  - 排查：
    - 本地 Redis 是否运行：`redis-cli ping` 应返回 `PONG`
    - `.env` 中 `REDIS_HOST/REDIS_PORT/REDIS_PASSWORD/REDIS_DB` 是否正确
    - Termux 本地启动示例：`redis-server --daemonize yes`
    - 远端 Redis：确认网络可达、防火墙与白名单
  - 处理：修正 `.env`，确保 Redis 正常运行后重启服务

- 现象：访问 `/admin-next/` 返回 404
  - 原因：未构建前端管理界面
  - 处理：`npm run install:web && npm run build:web`，然后重启服务

- 现象：Nginx 反代后会话不粘连/偶发 401
  - 原因：Nginx 默认丢弃带下划线的请求头（如 `session_id`）
  - 处理：在 http 块加入 `underscores_in_headers on;` 并保留相关头部

- 现象：SSE（流式响应）在代理后卡住或中断
  - 排查：反代是否开启了压缩/缓冲，或超时太短
  - 处理：关闭对 SSE 的 gzip/缓冲，适当放宽 `proxy_read_timeout`

- 现象：端口占用（EADDRINUSE）
  - 排查：`lsof -i:3000` 或 `ss -lntp | grep 3000`
  - 处理：结束占用进程或修改 `.env` 的 `PORT`

- 现象：跨域（CORS）错误
  - 处理：开启 `ENABLE_CORS=true` 或在反代层正确设置 `Access-Control-Allow-*`

- 现象：请求体过大（413）
  - 处理：增大反代/网关与应用层的限制（反代的 `client_max_body_size`、应用的请求大小限制）

- 现象：管理后台登录失败
  - 排查：`data/init.json` 中的管理员是否存在；Redis 会话是否正常
  - 处理：
    - 使用环境变量预设管理员：`ADMIN_USERNAME`、`ADMIN_PASSWORD`
    - 或清空初始化文件 `data/init.json` 后重启，按日志提示重新初始化

- 现象：Health `/health` 显示 unhealthy
  - 排查：看 `redis` 一项是否 healthy；检查日志 `logs/`
  - 处理：先修复 Redis 可用性，再重启应用

- 现象：Termux 中服务意外停止/休眠
  - 处理：执行 `termux-wake-lock` 保持唤醒；使用 `npm run service:start` 守护；避免清理后台
