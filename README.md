# commandcode-workers

Command Code API → OpenAI / Anthropic 兼容代理，运行在 **Cloudflare Workers**。

- 入口：`src/index.js`
- 配置：`wrangler.toml` + secrets / env
- 状态：isolate 内 ephemeral `Map`（冷启动会重置 session / fingerprint）

## Credits

本项目基于 [MAXeaglet/commandcode-proxy](https://github.com/MAXeaglet/commandcode-proxy) 的协议实现与代理逻辑，移植并裁剪为 Cloudflare Workers 部署形态。

- 原仓库：https://github.com/MAXeaglet/commandcode-proxy  
- 原作者：[@MAXeaglet](https://github.com/MAXeaglet)  
- 原项目 License：MIT（见本仓库 `LICENSE`）

感谢原作者开源的 Command Code → OpenAI/Anthropic 兼容代理。

## 功能

OpenAI Chat Completions · Anthropic Messages · 流式/非流式 · tool_use · 多模态图片 · reasoning_effort · 动态模型列表 · 设备指纹伪装 · 零输出 / 超时 → 429

## 快速开始

```bash
npm install
npx wrangler login   # 首次
npm run dev          # http://127.0.0.1:8787
npm run deploy       # 部署到 Cloudflare
```

API Key 通过请求头传入（须以 `user_` 开头）：

```bash
curl https://<your-worker>.workers.dev/v1/chat/completions \
  -H "Authorization: Bearer user_xxxxxxxxx" \
  -H "Content-Type: application/json" \
  -d '{"model":"deepseek/deepseek-v4-flash","messages":[{"role":"user","content":"hi"}],"stream":false}'
```

OpenAI SDK / Cursor：

```text
base_url = https://<your-worker>.workers.dev/v1
api_key  = user_xxxxxxxxx
```

## 接口

| 方法 | 路径 | 说明 |
|------|------|------|
| `POST` | `/v1/chat/completions` | OpenAI 兼容 |
| `POST` | `/v1/messages` | Anthropic 兼容 |
| `GET` | `/v1/models` | 模型列表 |
| `GET` | `/health` | 健康检查 |

## 配置

### `wrangler.toml` `[vars]`

| 变量 | 默认 | 说明 |
|------|------|------|
| `CC_API_BASE` | `https://api.commandcode.ai` | 上游 API |
| `PROJECT_SLUG` | `cc-proxy` | 项目标识（实际 slug 由 session 派生） |
| `CC_USE_PROVIDER_MODELS` | `true` | 是否从 Provider 拉模型列表 |

### Secrets（可选兜底 Key，推荐仍用请求头）

```bash
npx wrangler secret put CC_API_KEY
```

本地开发用 `.dev.vars`（已 gitignore）：

```text
CC_API_KEY=user_xxxxxxxxx
```

## 脚本

```bash
npm run dev       # 本地 wrangler dev
npm run deploy    # 部署
npm run tail      # 线上日志
```

## 限制

1. Session / 指纹为 isolate 内存态，冷启动会变；要稳定可后续加 KV。
2. 超长非流式可能触达 Workers 时长限制；优先 `stream: true`。
3. 日志仅 `console`（可用 Cloudflare Logpush）。

## License

MIT — 版权声明见 `LICENSE`（保留原作者 [MAXeaglet](https://github.com/MAXeaglet) 版权）。
