# Cloudflare Workers MVP

将本代理以 **Cloudflare Workers** 运行的最小可用版本。

- 入口：`src/index.js`
- 配置：`wrangler.toml` + env vars
- 状态：进程/isolate 内 `Map`（**ephemeral**，冷启动会丢 session/fingerprint）
- Node 版（`proxy.mjs` + Docker）保持不变

## 要求

- Node ≥ 18
- Cloudflare 账号 + `wrangler` 登录

```bash
npm install
npx wrangler login
```

## 本地开发

```bash
npm run worker:dev
# 默认 http://127.0.0.1:8787
```

```bash
curl http://127.0.0.1:8787/health

curl http://127.0.0.1:8787/v1/chat/completions \
  -H "Authorization: Bearer user_xxxxxxxxx" \
  -H "Content-Type: application/json" \
  -d '{"model":"deepseek/deepseek-v4-flash","messages":[{"role":"user","content":"hi"}],"stream":true}'
```

## 部署

```bash
npm run worker:deploy
```

部署后 base URL 形如：

```text
https://commandcode-proxy.<your-subdomain>.workers.dev/v1
```

可在 Cursor / OpenAI SDK 里把 `base_url` 指到该地址。

## 环境变量

| 变量 | 默认 | 说明 |
|------|------|------|
| `CC_API_BASE` | `https://api.commandcode.ai` | 上游 API |
| `PROJECT_SLUG` | `cc-proxy` | 配置项（project slug 实际由 session 派生） |
| `CC_USE_PROVIDER_MODELS` | `true` | 是否拉 Provider 模型列表 |
| `CC_API_KEY` | — | 可选兜底 Key（推荐仍用请求头） |
| `LOG_LEVEL` | `info` | 日志级别（目前主要走 console） |
| `MODEL_REFRESH_INTERVAL_MS` | `300000` | 模型列表缓存间隔 |

可选 secret：

```bash
npx wrangler secret put CC_API_KEY
```

本地可用 `.dev.vars`（已 gitignore）：

```text
CC_API_KEY=user_xxxxxxxxx
```

## 与 Node 版差异（MVP）

| 点 | Node (`proxy.mjs`) | Workers MVP |
|----|--------------------|-------------|
| 配置 | `config.json` + env | `wrangler.toml` / secrets |
| 日志文件 | `logFile` | 仅 `console`（可用 Logpush） |
| Session / 指纹 | 长生命周期进程内 Map | isolate 级 Map，冷启动重置 |
| 定时任务 | `setInterval` | 请求时惰性刷新 |
| 环境字段 | `process.platform` 等 | 固定伪装 `win32-x64, Node.js 22.0.0` |
| 客户端断连 | `res.on('close')` | `request.signal` / stream `cancel` |

## 已知限制

1. **状态不稳定**：同一 API Key 跨 isolate 可能换 session/fingerprint。个人用通常可接受；要稳定需后续上 KV。
2. **时长**：超长非流式生成可能触达 Workers CPU/wall-clock 限制；优先用 `stream: true`。
3. **免费计划**：子请求与 CPU 有限额，重度使用建议 Paid。

## 脚本

```bash
npm run worker:dev      # 本地
npm run worker:deploy   # 部署
npm run worker:tail     # 实时日志
```
