# 快速部署指南（中文）

本指南帮助你从零开始部署 Codex Relay Service，确保步骤清晰、无歧义。

## 1. 环境准备
- Node.js ≥ 18（建议 LTS）
- 可用的 Redis ≥ 6
- Linux/macOS 或 Docker 环境

## 2. 获取项目
```bash
git clone git@github.com:xuxinyue18-dot/codex-relay-service.git
cd codex-relay-service
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

