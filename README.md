# Codex Relay Service

轻量、可扩展的多供应商 AI 中转服务。开箱即用兼容 OpenAI API（/v1/chat/completions 等），支持 Claude、OpenAI、Gemini、Azure OpenAI、AWS Bedrock 等后端，提供多账号调度、API Key 鉴权、用量/费用统计、速率限制、代理、Webhook、Admin 后台等能力。

— 本文为合并版文档：整合原 README 与 CLAUDE.md，去重并按实际实现梳理 —

如需一步步部署指引，请参见根目录的 INSTALL.md（中文）与 INSTALL_EN.md（English）。

## 核心特性

- 多后端：Claude / OpenAI / Gemini / Azure OpenAI / Bedrock
- 统一协议：OpenAI 兼容接口 + 统一路由 `/v1/unified/chat/completions`
- 多账号调度：基于权限/速率/会话的智能分配与回退
- 安全与控制：API Key 鉴权、速率限制、请求大小限制、模型黑名单
- 观测与成本：用量与费用统计、定价更新、Webhook 通知
- 代理支持：HTTP/HTTPS/SOCKS5，支持账户级代理与 IPv4/IPv6 开关
- 管理界面：`/admin-next` 管理后台（可选构建 `web/admin-spa`）

## 运行要求

- Node.js >= 18
- Redis >= 6（会话、队列、统计）
- Linux/macOS/Docker 均可

## 快速开始

1) 配置环境变量
- 复制 `.env.example` 为 `.env`，至少设置：
  - `PORT`、`HOST`
  - `JWT_SECRET`、`ENCRYPTION_KEY`
  - `REDIS_HOST`、`REDIS_PORT`、`REDIS_PASSWORD`（如需）
  - 更多选项见 `config/config.example.js`

2) 安装与初始化
```bash
npm install
npm run setup            # 初始化必要数据（如有）
npm run install:web      # 可选：安装 Admin SPA 依赖
npm run build:web        # 可选：构建 Admin SPA 到 web/admin-spa/dist
```

3) 运行
```bash
npm run dev              # 开发模式（nodemon）
# 或
npm run service:start    # 以进程方式启动（scripts/manage.js）
npm run service:status   # 查看状态
npm run service:logs     # 查看日志
```

4) Docker（可选）
```bash
npm run docker:build
npm run docker:up        # docker-compose up -d
```

示例 docker-compose 片段（含常用环境变量）：
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
      # 基础
      - NODE_ENV=production
      - PORT=3000
      - HOST=0.0.0.0

      # 安全（必填）
      - JWT_SECRET=${JWT_SECRET}
      - ENCRYPTION_KEY=${ENCRYPTION_KEY}
      - API_KEY_PREFIX=${API_KEY_PREFIX:-cr_}
      - WEB_SESSION_SECRET=${WEB_SESSION_SECRET:-CHANGE_ME_SESSION_SECRET}

      # 管理员（可选）
      - ADMIN_USERNAME=${ADMIN_USERNAME:-}
      - ADMIN_PASSWORD=${ADMIN_PASSWORD:-}

      # Redis
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - REDIS_PASSWORD=${REDIS_PASSWORD:-}
      - REDIS_DB=${REDIS_DB:-0}

      # Gemini OAuth（如需）
      - GEMINI_OAUTH_CLIENT_ID=${GEMINI_OAUTH_CLIENT_ID:-}
      - GEMINI_OAUTH_CLIENT_SECRET=${GEMINI_OAUTH_CLIENT_SECRET:-}

      # 代理与限制（可选）
      - DEFAULT_PROXY_TIMEOUT=${DEFAULT_PROXY_TIMEOUT:-600000}
      - MAX_PROXY_RETRIES=${MAX_PROXY_RETRIES:-3}
      - PROXY_USE_IPV4=${PROXY_USE_IPV4:-true}
      - REQUEST_TIMEOUT=${REQUEST_TIMEOUT:-600000}
      - DEFAULT_TOKEN_LIMIT=${DEFAULT_TOKEN_LIMIT:-1000000}

      # 日志（可选）
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

## API 使用

- OpenAI 兼容示例（鉴权：`Authorization: Bearer <你的API Key>`）
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
- 统一路由（自动选择 Claude/OpenAI/Gemini 后端）：`POST /v1/unified/chat/completions`
- Key 权限会影响可用后端（`all`/`openai`/`claude`/`gemini`）
- 支持 SSE 流式输出（`stream: true`）

## 管理与脚本

- Admin 后台：`/admin-next/`（需先构建前端）
- 常用脚本：
  - `npm run service:start|stop|restart|status|logs`
  - `npm run update:pricing`（更新模型定价）
  - `npm run data:export|import`（Redis 数据迁移）
  - `npm run init:costs`（初始化费用数据）
  - `npm run monitor`、`npm run status:detail`

## 配置总览

- 基础：`PORT`、`HOST`、`NODE_ENV`、`TRUST_PROXY`
- 安全：`JWT_SECRET`、`API_KEY_PREFIX`、`ENCRYPTION_KEY`、会话超时
- Redis：`REDIS_HOST`、`REDIS_PORT`、`REDIS_PASSWORD`、`REDIS_DB`、`REDIS_ENABLE_TLS`
- 会话：`STICKY_SESSION_TTL_HOURS`、`STICKY_SESSION_RENEWAL_THRESHOLD_MINUTES`
- Claude：`CLAUDE_API_URL`、`CLAUDE_API_VERSION`、`CLAUDE_OVERLOAD_HANDLING_MINUTES`
- Bedrock：`CLAUDE_CODE_USE_BEDROCK`、`AWS_REGION`、`ANTHROPIC_MODEL` 等
- 代理：`DEFAULT_PROXY_TIMEOUT`、`MAX_PROXY_RETRIES`、`PROXY_USE_IPV4`
- 限制：`REQUEST_TIMEOUT`、`DEFAULT_TOKEN_LIMIT`
- 日志：`LOG_LEVEL`、`LOG_MAX_SIZE`、`LOG_MAX_FILES`
- Web：`WEB_TITLE`、`WEB_DESCRIPTION`、`ENABLE_CORS`
- LDAP：`LDAP_ENABLED`、`LDAP_URL`、证书与属性映射等
- 用户：`USER_MANAGEMENT_ENABLED`、`MAX_API_KEYS_PER_USER`
- Webhook：`WEBHOOK_ENABLED`、`WEBHOOK_URLS`、`WEBHOOK_RETRIES`

详见 `.env.example` 与 `config/config.example.js`。

## 环境变量要点（务必检查）

- 必填
  - `JWT_SECRET`：JWT 密钥，建议≥32字符随机串
  - `ENCRYPTION_KEY`：32字符加密密钥
- 推荐
  - `WEB_SESSION_SECRET`：Web 会话密钥
- Redis 连接
  - `REDIS_HOST`、`REDIS_PORT`、`REDIS_PASSWORD`（如需）、`REDIS_DB`
- 代理（如需）
  - `DEFAULT_PROXY_TIMEOUT`、`MAX_PROXY_RETRIES`、`PROXY_USE_IPV4`
- Gemini OAuth（如需使用 Gemini 账户）
  - `GEMINI_OAUTH_CLIENT_ID`
  - `GEMINI_OAUTH_CLIENT_SECRET`

以上变量均在 `.env.example` 提供占位，请复制为 `.env` 后按需填写。

## 部署清单（开箱即用）

1) 克隆并进入目录
```bash
git clone git@github.com:xuxinyue18-dot/codex-relay-service.git
cd codex-relay-service
```

2) 准备配置文件
```bash
cp .env.example .env                  # 填写上方“环境变量要点”
cp config/config.example.js config/config.js
```

3) 安装与初始化
```bash
npm install
npm run setup
```

4) 可选：构建管理端（Admin UI）
```bash
npm run install:web
npm run build:web    # 产物位于 web/admin-spa/dist
```

5) 启动服务
```bash
npm run service:start   # 或 npm run dev（开发模式）
```

## 架构总览（合并自 CLAUDE.md）

- 统一调度：`unifiedClaudeScheduler`、`unifiedGeminiScheduler`、`unifiedOpenAIScheduler`、`droidScheduler`
- 账户类型：`claude-official`、`claude-console`、`bedrock`、`ccr`、`droid`、`gemini`、`openai-responses`、`azure-openai`
- 鉴权链路：自建 API Key → 验证与限流 → 统一调度 → 账户 Token → 代理转发
- Token 管理：过期检测与自动刷新（支持提前刷新，走账户级代理）
- 数据安全：敏感凭据 AES 加密存储于 Redis
- 粘性会话：会话级账户绑定，保障上下文连续
- 权限/限制：Key 权限、客户端识别（User-Agent）、模型黑名单

### 主要组件

- 转发服务：`claudeRelayService`、`claudeConsoleRelayService`、`geminiRelayService`、`bedrockRelayService`、`azureOpenaiRelayService`、`droidRelayService`、`ccrRelayService`、`openaiResponsesRelayService`
- 账户管理：`claudeAccountService`、`claudeConsoleAccountService`、`geminiAccountService`、`bedrockAccountService`、`azureOpenaiAccountService`、`droidAccountService`、`ccrAccountService`、`openaiResponsesAccountService`、`openaiAccountService`、`accountGroupService`
- 调度器：`unifiedClaudeScheduler`、`unifiedGeminiScheduler`、`unifiedOpenAIScheduler`、`droidScheduler`
- 核心功能：`apiKeyService`、`userService`、`pricingService`、`costInitService`、`webhookService`、`webhookConfigService`、`ldapService`、`tokenRefreshService`、`rateLimitCleanupService`、`claudeCodeHeadersService`
- 工具：`oauthHelper`（PKCE + 代理）、`workosOAuthHelper`、`openaiToClaude`

### 请求流程（简要）

1) 客户端以 `cr_` Key 请求对应路由（/api、/claude、/gemini、/openai、/droid 等）
2) `authenticateApiKey` 验证有效性、速率、权限、客户端与模型限制
3) 调度器依据模型/会话/权限选择账户；必要时自动回退
4) 检查/刷新账户 Token（经代理）
5) 使用账户凭据转发请求（非客户端 Key），按账户代理访问上游 API
6) 返回（流式/非流式）并记录使用量与成本，更新限流与并发计数

## 日志与排障

- 默认将应用日志写入 `logs/`，按大小/数量轮转
- HTTP 调试：设置 `DEBUG_HTTP_TRAFFIC=true`（输出到 `logs/http-debug-*.log`）
- 常见问题：
  - Redis 连接失败：检查 `REDIS_HOST/PORT/PASSWORD` 与网络
  - 代理异常：确认协议、地址与超时；必要时启用 `PROXY_USE_IPV4`
  - Admin 404：确保先执行 `npm run build:web`

## 安全与合规

- `.gitignore` 已排除日志/数据/密钥等敏感内容
- 请勿提交生产密钥；使用 `.env.example` 和 `web/*/.env.example` 作为模板
- 仅供学习与研究，请遵循服务提供商条款

## 许可证

MIT（见 `LICENSE`）

#### 必填项

- `JWT_SECRET`: JWT密钥，至少32个字符
- `ENCRYPTION_KEY`: 加密密钥，必须是32个字符

#### 可选项

- `ADMIN_USERNAME`: 管理员用户名（不设置则自动生成）
- `ADMIN_PASSWORD`: 管理员密码（不设置则自动生成）
- `LOG_LEVEL`: 日志级别（默认：info）
- 更多配置项请参考 `.env.example` 文件

### 管理员凭据获取方式

1. **查看容器日志**

   ```bash
   docker logs claude-relay-service
   ```

2. **查看挂载的文件**

   ```bash
   cat ./data/init.json
   ```

3. **使用环境变量预设**
   ```bash
   # 在 .env 文件中设置
   ADMIN_USERNAME=cr_admin_custom
   ADMIN_PASSWORD=your-secure-password
   ```

---

## 🎮 开始使用

### 1. 打开管理界面

浏览器访问：`http://你的服务器IP:3000/web`

管理员账号：

- 自动生成：查看 data/init.json
- 环境变量预设：通过 ADMIN_USERNAME 和 ADMIN_PASSWORD 设置
- Docker 部署：查看容器日志 `docker logs claude-relay-service`

### 2. 添加Claude账户

这一步比较关键，需要OAuth授权：

1. 点击「Claude账户」标签
2. 如果你担心多个账号共用1个IP怕被封禁，可以选择设置静态代理IP（可选）
3. 点击「添加账户」
4. 点击「生成授权链接」，会打开一个新页面
5. 在新页面完成Claude登录和授权
6. 复制返回的Authorization Code
7. 粘贴到页面完成添加

**注意**: 如果你在国内，这一步可能需要科学上网。

### 3. 创建API Key

给每个使用者分配一个Key：

1. 点击「API Keys」标签
2. 点击「创建新Key」
3. 给Key起个名字，比如「张三的Key」
4. 设置使用限制（可选）：
   - **速率限制**: 限制每个时间窗口的请求次数和Token使用量
   - **并发限制**: 限制同时处理的请求数
   - **模型限制**: 限制可访问的模型列表
   - **客户端限制**: 限制只允许特定客户端使用（如ClaudeCode、Gemini-CLI等）
5. 保存，记下生成的Key

### 4. 开始使用 Claude Code 和 Gemini CLI

现在你可以用自己的服务替换官方API了：

**Claude Code 设置环境变量：**

默认使用标准 Claude 账号池：

```bash
export ANTHROPIC_BASE_URL="http://127.0.0.1:3000/api/" # 根据实际填写你服务器的ip地址或者域名
export ANTHROPIC_AUTH_TOKEN="后台创建的API密钥"
```

**VSCode Claude 插件配置：**

如果使用 VSCode 的 Claude 插件，需要在 `~/.claude/config.json` 文件中配置：

```json
{
    "primaryApiKey": "crs"
}
```

如果该文件不存在，请手动创建。Windows 用户路径为 `C:\Users\你的用户名\.claude\config.json`。

**Gemini CLI 设置环境变量：**

```bash
GEMINI_MODEL="gemini-2.5-pro"
GOOGLE_GEMINI_BASE_URL="http://127.0.0.1:3000/gemini" # 根据实际填写你服务器的ip地址或者域名
GEMINI_API_KEY="后台创建的API密钥"  # 使用相同的API密钥即可
```
**使用 Claude Code：**

```bash
claude
```

**使用 Gemini CLI：**

```bash
gemini  # 或其他 Gemini CLI 命令
```

**Codex 配置：**

在 `~/.codex/config.toml` 文件**开头**添加以下配置：

```toml
model_provider = "crs"
model = "gpt-5-codex"
model_reasoning_effort = "high"
disable_response_storage = true
preferred_auth_method = "apikey"

[model_providers.crs]
name = "crs"
base_url = "http://127.0.0.1:3000/openai"  # 根据实际填写你服务器的ip地址或者域名
wire_api = "responses"
requires_openai_auth = true
env_key = "CRS_OAI_KEY"
```

在 `~/.codex/auth.json` 文件中配置API密钥为 null：

```json
{
    "OPENAI_API_KEY": null  
}
```

环境变量设置：

```bash
export CRS_OAI_KEY="后台创建的API密钥"
```

> ⚠️ 在通过 Nginx 反向代理 CRS 服务并使用 Codex CLI 时，需要在 http 块中添加 underscores_in_headers on;。因为 Nginx 默认会移除带下划线的请求头（如 session_id），一旦该头被丢弃，多账号环境下的粘性会话功能将失效。

**Droid CLI 配置：**

Droid CLI 读取 `~/.factory/config.json`。可以在该文件中添加自定义模型以指向本服务的新端点：

```json
{
  "custom_models": [
    {
      "model_display_name": "Sonnet 4.5 [crs]",
      "model": "claude-sonnet-4-5-20250929",
      "base_url": "http://127.0.0.1:3000/droid/claude",
      "api_key": "后台创建的API密钥",
      "provider": "anthropic",
      "max_tokens": 8192
    },
    {
      "model_display_name": "GPT5-Codex [crs]",
      "model": "gpt-5-codex",
      "base_url": "http://127.0.0.1:3000/droid/openai",
      "api_key": "后台创建的API密钥",
      "provider": "openai",
      "max_tokens": 16384
    }
  ]
}
```

> 💡 将示例中的 `http://127.0.0.1:3000` 替换为你的服务域名或公网地址，并写入后台生成的 API 密钥（cr_ 开头）。

### 5. 第三方工具API接入

本服务支持多种API端点格式，方便接入不同的第三方工具（如Cherry Studio等）。

#### Cherry Studio 接入示例

Cherry Studio支持多种AI服务的接入，下面是不同账号类型的详细配置：

**1. Claude账号接入：**

```
# API地址
http://你的服务器:3000/claude

# 模型ID示例
claude-sonnet-4-5-20250929 # Claude Sonnet 4.5
claude-opus-4-20250514     # Claude Opus 4
```

配置步骤：
- 供应商类型选择"Anthropic"
- API地址填入：`http://你的服务器:3000/claude`
- API Key填入：后台创建的API密钥（cr_开头）

**2. Gemini账号接入：**

```
# API地址
http://你的服务器:3000/gemini

# 模型ID示例
gemini-2.5-pro             # Gemini 2.5 Pro
```

配置步骤：
- 供应商类型选择"Gemini"
- API地址填入：`http://你的服务器:3000/gemini`
- API Key填入：后台创建的API密钥（cr_开头）

**3. Codex接入：**

```
# API地址
http://你的服务器:3000/openai

# 模型ID（固定）
gpt-5                      # Codex使用固定模型ID
```

配置步骤：
- 供应商类型选择"Openai-Response"
- API地址填入：`http://你的服务器:3000/openai`
- API Key填入：后台创建的API密钥（cr_开头）
- **重要**：Codex只支持Openai-Response标准


**Cherry Studio 地址格式重要说明：**

- ✅ **推荐格式**：`http://你的服务器:3000/claude`（不加结尾 `/`，让 Cherry Studio 自动加上 v1）
- ✅ **等效格式**：`http://你的服务器:3000/claude/v1/`（手动指定 v1 并加结尾 `/`）
- 💡 **说明**：这两种格式在 Cherry Studio 中是完全等效的
- ❌ **错误格式**：`http://你的服务器:3000/claude/`（单独的 `/` 结尾会被 Cherry Studio 忽略 v1 版本）

#### 其他第三方工具接入

**接入要点：**

- 所有账号类型都使用相同的API密钥（在后台统一创建）
- 根据不同的路由前缀自动识别账号类型
- `/claude/` - 使用Claude账号池
- `/droid/claude/` - 使用Droid类型Claude账号池（只建议api调用或Droid Cli中使用）
- `/gemini/` - 使用Gemini账号池  
- `/openai/` - 使用Codex账号（只支持Openai-Response格式）
- `/droid/openai/` - 使用Droid类型OpenAI兼容账号池（只建议api调用或Droid Cli中使用）
- 支持所有标准API端点（messages、models等）

**重要说明：**

- 确保在后台已添加对应类型的账号（Claude/Gemini/Codex）
- API密钥可以通用，系统会根据路由自动选择账号类型
- 建议为不同用户创建不同的API密钥便于使用统计

---

## 🔧 日常维护

### 服务管理

```bash
# 查看服务状态
npm run service:status

# 查看日志
npm run service:logs

# 重启服务
npm run service:restart:daemon

# 停止服务
npm run service:stop
```

### 监控使用情况

- **Web界面**: `http://你的域名:3000/web` - 查看使用统计
- **健康检查**: `http://你的域名:3000/health` - 确认服务正常
- **日志文件**: `logs/` 目录下的各种日志文件

### 升级指南

当有新版本发布时，按照以下步骤升级服务：

```bash
# 1. 进入项目目录
cd claude-relay-service

# 2. 拉取最新代码
git pull origin main

# 如果遇到 package-lock.json 冲突，使用远程版本
git checkout --theirs package-lock.json
git add package-lock.json

# 3. 安装新的依赖（如果有）
npm install

# 4. 安装并构建前端
npm run install:web
npm run build:web

# 5. 重启服务
npm run service:restart:daemon

# 6. 检查服务状态
npm run service:status
```

**注意事项：**

- 升级前建议备份重要配置文件（.env, config/config.js）
- 查看更新日志了解是否有破坏性变更
- 如果有数据库结构变更，会自动迁移

---

## 🔒 客户端限制功能

### 功能说明

客户端限制功能允许你控制每个API Key可以被哪些客户端使用，通过User-Agent识别客户端，提高API的安全性。

### 使用方法

1. **在创建或编辑API Key时启用客户端限制**：
   - 勾选"启用客户端限制"
   - 选择允许的客户端（支持多选）

2. **预定义客户端**：
   - **ClaudeCode**: 官方Claude CLI（匹配 `claude-cli/x.x.x (external, cli)` 格式）
   - **Gemini-CLI**: Gemini命令行工具（匹配 `GeminiCLI/vx.x.x (platform; arch)` 格式）

3. **调试和诊断**：
   - 系统会在日志中记录所有请求的User-Agent
   - 客户端验证失败时会返回403错误并记录详细信息
   - 通过日志可以查看实际的User-Agent格式，方便配置自定义客户端


### 日志示例

认证成功时的日志：

```
🔓 Authenticated request from key: 测试Key (key-id) in 5ms
   User-Agent: "claude-cli/1.0.58 (external, cli)"
```

客户端限制检查日志：

```
🔍 Checking client restriction for key: key-id (测试Key)
   User-Agent: "Mozilla/5.0 (Windows NT 10.0; Win64; x64)"
   Allowed clients: claude_code, gemini_cli
🚫 Client restriction failed for key: key-id (测试Key) from 127.0.0.1, User-Agent: Mozilla/5.0...
```

### 常见问题处理

**Redis连不上？**

```bash
# 检查Redis是否启动
redis-cli ping

# 应该返回 PONG
```

**OAuth授权失败？**

- 检查代理设置是否正确
- 确保能正常访问 claude.ai
- 清除浏览器缓存重试

**API请求失败？**

- 检查API Key是否正确
- 查看日志文件找错误信息
- 确认Claude账户状态正常

---

## 🛠️ 进阶

### 反向代理部署指南

在生产环境中，建议通过反向代理进行连接，以便使用自动 HTTPS、安全头部和性能优化。下面提供两种常用方案： **Caddy** 和 **Nginx Proxy Manager (NPM)**。

---

## Caddy 方案

Caddy 是一款自动管理 HTTPS 证书的 Web 服务器，配置简单、性能优秀，很适合不需要 Docker 环境的部署方案。

**1. 安装 Caddy**

```bash
# Ubuntu/Debian
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy

# CentOS/RHEL/Fedora
sudo yum install yum-plugin-copr
sudo yum copr enable @caddy/caddy
sudo yum install caddy
```

**2. Caddy 配置**

编辑 `/etc/caddy/Caddyfile` ：

```caddy
your-domain.com {
    # 反向代理到本地服务
    reverse_proxy 127.0.0.1:3000 {
        # 支持流式响应或 SSE
        flush_interval -1

        # 传递真实 IP
        header_up X-Real-IP {remote_host}
        header_up X-Forwarded-For {remote_host}
        header_up X-Forwarded-Proto {scheme}

        # 长读/写超时配置
        transport http {
            read_timeout 300s
            write_timeout 300s
            dial_timeout 30s
        }
    }

    # 安全头部
    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains"
        X-Frame-Options "DENY"
        X-Content-Type-Options "nosniff"
        -Server
    }
}
```

**3. 启动 Caddy**

```bash
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl start caddy
sudo systemctl enable caddy
sudo systemctl status caddy
```

**4. 服务配置**

Caddy 会自动管理 HTTPS，因此可以将服务限制在本地进行监听：

```javascript
// config/config.js
module.exports = {
  server: {
    port: 3000,
    host: '127.0.0.1' // 只监听本地
  }
}
```

**Caddy 特点**

* 🔒 自动 HTTPS，零配置证书管理
* 🛡️ 安全默认配置，启用现代 TLS 套件
* ⚡ HTTP/2 和流式传输支持
* 🔧 配置文件简洁，易于维护

---

## Nginx Proxy Manager (NPM) 方案

Nginx Proxy Manager 通过图形化界面管理反向代理和 HTTPS 证书，並以 Docker 容器部署。

**1. 在 NPM 创建新的 Proxy Host**

Details 配置如下：

| 项目                    | 设置                      |
| --------------------- | ----------------------- |
| Domain Names          | relay.example.com       |
| Scheme                | http                    |
| Forward Hostname / IP | 192.168.0.1 (docker 机器 IP) |
| Forward Port          | 3000                    |
| Block Common Exploits | ☑️                      |
| Websockets Support    | ❌ **关闭**                |
| Cache Assets          | ❌ **关闭**                |
| Access List           | Publicly Accessible     |

> 注意：
> - 请确保 Claude Relay Service **监听 host 为 `0.0.0.0` 、容器 IP 或本机 IP**，以便 NPM 实现内网连接。
> - **Websockets Support 和 Cache Assets 必须关闭**，否则会导致 SSE / 流式响应失败。

**2. Custom locations**

無需添加任何内容，保持为空。

**3. SSL 设置**

* **SSL Certificate**: Request a new SSL Certificate (Let's Encrypt) 或已有证书
* ☑️ **Force SSL**
* ☑️ **HTTP/2 Support**
* ☑️ **HSTS Enabled**
* ☑️ **HSTS Subdomains**

**4. Advanced 配置**

Custom Nginx Configuration 中添加以下内容：

```nginx
# 传递真实用户 IP
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;

# 支持 WebSocket / SSE 等流式通信
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
proxy_buffering off;

# 长连接 / 超时设置（适合 AI 聊天流式传输）
proxy_read_timeout 300s;
proxy_send_timeout 300s;
proxy_connect_timeout 30s;

# ---- 安全性设置 ----
# 严格 HTTPS 策略 (HSTS)
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

# 阻挡点击劫持与内容嗅探
add_header X-Frame-Options "DENY" always;
add_header X-Content-Type-Options "nosniff" always;

# Referrer / Permissions 限制策略
add_header Referrer-Policy "no-referrer-when-downgrade" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;

# 隐藏服务器信息（等效于 Caddy 的 `-Server`）
proxy_hide_header Server;

# ---- 性能微调 ----
# 关闭代理端缓存，确保即时响应（SSE / Streaming）
proxy_cache_bypass $http_upgrade;
proxy_no_cache $http_upgrade;
proxy_request_buffering off;
```

**4. 启动和验证**

* 保存后等待 NPM 自动申请 Let's Encrypt 证书（如果有）。
* Dashboard 中查看 Proxy Host 状态，确保显示为 "Online"。
* 访问 `https://relay.example.com`，如果显示绿色锁图标即表示 HTTPS 正常。

**NPM 特点**

* 🔒 自动申请和续期证书
* 🔧 图形化界面，方便管理多服务
* ⚡ 原生支持 HTTP/2 / HTTPS
* 🚀 适合 Docker 容器部署

---

上述两种方案均可用于生产部署。

---

## 💡 使用建议

### 账户管理

- **定期检查**: 每周看看账户状态，及时处理异常
- **合理分配**: 可以给不同的人分配不同的apikey，可以根据不同的apikey来分析用量

### 安全建议

- **使用HTTPS**: 强烈建议使用Caddy反向代理（自动HTTPS），确保数据传输安全
- **定期备份**: 重要配置和数据要备份
- **监控日志**: 定期查看异常日志
- **更新密钥**: 定期更换JWT和加密密钥
- **防火墙设置**: 只开放必要的端口（80, 443），隐藏直接服务端口

---

## 🆘 遇到问题怎么办？

### 自助排查

1. **查看日志**: `logs/` 目录下的日志文件
2. **检查配置**: 确认配置文件设置正确
3. **测试连通性**: 用 curl 测试API是否正常
4. **重启服务**: 有时候重启一下就好了

### 寻求帮助

- **GitHub Issues**: 提交详细的错误信息
- **查看文档**: 仔细阅读错误信息和文档
- **社区讨论**: 看看其他人是否遇到类似问题

---

## 📄 许可证

本项目采用 [MIT许可证](LICENSE)。

---

<div align="center">

**⭐ 觉得有用的话给个Star呗，这是对作者最大的鼓励！**

**🤝 有问题欢迎提Issue，有改进建议欢迎PR**

</div>
