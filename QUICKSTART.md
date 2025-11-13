# 快速上手（极简）

要求
- Node.js ≥ 18、Redis ≥ 6（本地或远端）

步骤
```bash
# 1) 获取代码
git clone git@github.com:xuxinyue18-dot/codex-relay-service.git
cd codex-relay-service

# 2) 配置环境
cp .env.example .env      # 至少填写 JWT_SECRET、ENCRYPTION_KEY、(可选) WEB_SESSION_SECRET
cp config/config.example.js config/config.js

# 3) 安装与初始化
npm install
npm run setup

# 4) 启动（可选构建管理端：npm run install:web && npm run build:web）
npm run service:start
```

访问
- 管理界面：`http://<主机>:3000/admin-next/`
- 健康检查：`http://<主机>:3000/health`

排障速记
- admin 404：先执行 `npm run build:web`
- Redis 失败：确认 Redis 正常、`.env` 连接正确
- Termux 环境：见 INSTALL.md 的 Termux 小节

更多说明：见 README / INSTALL
