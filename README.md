# 微信代码助手（Cloudflare Workers AI）

一个部署在 Cloudflare Workers 的中文代码问答助手，支持：
- 直接粘贴代码、错误日志并分析；
- 调用 Cloudflare Workers AI；
- 查询 GitHub 仓库元数据并尝试搜索相关代码；
- 支持公开仓库；私有仓库需要 GitHub token；
- 提供网页界面与通用 `POST /webhook` 消息适配端点。

> **微信说明：** 本项目已经提供通用 webhook 接口，但它本身不是微信官方机器人，也没有内置可保证长期可用的个人微信登录/收发协议。要连接普通个人微信，需要另行部署并配置一个兼容的消息桥（例如你实际使用的 iLink/ClawBot 网关），将收到的文本转发到本项目的 `/webhook`，再把响应的 `text` 发回微信。不同网关的协议和可用性不同，请勿把微信账号密码或二维码登录凭据交给第三方。

## 部署到 Cloudflare

1. 在 Cloudflare Dashboard 打开 **Workers & Pages**，选择创建应用并连接 GitHub。
2. 选择 `8aot/wechat-code-assistant`。框架选 **None / Worker**（如果系统识别 wrangler 配置，可直接使用）。
3. 构建命令可留空；部署命令设为 `npx wrangler deploy`。项目根目录包含 `wrangler.jsonc`。
4. 在 Worker 的 **Settings → Bindings** 确认添加 Workers AI binding，变量名必须是 `AI`。配置文件也已声明此 binding。
5. 部署后打开 Worker 的 `workers.dev` 地址测试网页与 `/health`。

## 私有 GitHub 仓库

在 Worker 的 **Settings → Variables and Secrets** 添加 Secret：
- `GITHUB_TOKEN`: GitHub fine-grained personal access token。仅授予需要读取的仓库，至少有 Metadata: Read 和 Contents: Read；如果要使用代码搜索，请根据 GitHub 当前要求授予相应权限。绝不要把 token 提交到仓库。
- 可选 `GITHUB_DEFAULT_REPO`: 默认仓库，格式 `owner/repo`。
- 可选 `AI_MODEL`: 覆盖默认模型名。
- 可选 `WEBHOOK_SECRET`: 保护 webhook 的随机长密钥。
- 可选 `ALLOWED_ORIGINS`: 允许的前端 Origin，逗号分隔；不设置时用于简单演示。

若要让助手搜索多个私有仓库，请在每次请求的 `repo` 字段中传入 `owner/repo`，且 token 必须有该仓库的读取权限。

## API

### `POST /api/chat`

请求：
```json
{
  "message": "这个 TypeError 为什么出现？",
  "repo": "owner/repo",
  "history": [
    { "role": "user", "content": "之前的问题" },
    { "role": "assistant", "content": "之前的回答" }
  ]
}
```

`repo` 和 `history` 都是可选字段。返回 `{ "reply": "..." }`，错误时返回 `{ "error": "..." }`。

### `POST /webhook`

可供消息桥调用。请求 JSON：
```json
{ "text": "帮我解释这段代码", "repo": "owner/repo", "user_id": "optional" }
```

如果配置了 `WEBHOOK_SECRET`，请求需在 `X-Webhook-Secret` 或 `Authorization: Bearer <secret>` 头中携带密钥。响应包含 `text` 字段，消息桥可以将它发回聊天。

## 本地运行

需要 Node.js 20+：
```bash
npm install
npx wrangler login
npm run dev
``

## 注意事项

- 免费额度和可用模型以 Cloudflare 当前 Workers AI 计划及账户页面为准；不同模型的可用性可能不同。
- GitHub 代码搜索可能因 token 权限、仓库索引或 API 限制而失败。失败时可以直接粘贴相关代码和报错。
- 不要把私有代码、凭据或用户数据发送给你不信任的 AI 服务或消息桥。
- 目前没有数据库会话存储，历史记录由调用方可选传入。
