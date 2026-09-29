# commandcode-workers

Command Code API → OpenAI / Anthropic 兼容代理，部署在 Cloudflare Workers（单文件、无构建步骤）。

## Project

- 运行时：Cloudflare Workers（`compatibility_date = 2024-11-12`，`compatibility_flags = ["nodejs_compat"]`）。
- 入口：`src/index.js`（唯一源文件，~1900 行 ESM，`export default { fetch }`）。
- 路由：`POST /v1/chat/completions`（OpenAI）、`POST /v1/messages`（Anthropic）、`GET /v1/models`、`GET /health`（`/` 同）。
- 协议实现移植自 https://github.com/MAXeaglet/commandcode-proxy （MIT，保留其版权声明，勿删除 `src/index.js` 头部与 `LICENSE` 的 credit）。
- 无依赖（`devDependencies` 仅 `wrangler`）、无打包器、无测试框架、无 lint 配置。

## Commands

```bash
npm install
npm run dev        # wrangler dev → http://127.0.0.1:8787
npm run deploy     # wrangler deploy
npm run tail       # wrangler tail（线上日志）

node --check src/index.js   # 唯一的语法/静态检查（仓库没有 lint/test 脚本）
npx wrangler secret put CC_API_KEY   # 可选兜底 key
```

本地开发 key 放 `.dev.vars`（已 gitignore）：`CC_API_KEY=user_xxxx`。真实调用请优先走请求头。

## Architecture

单文件按 `// ── 分区 ──` 注释切块，改动时保持分区归属：

| 区域 | 作用 |
|------|------|
| `loadConfig` / `CFG` | 每次 `fetch` 从 `env` 重载配置（`CC_API_BASE`、`PROJECT_SLUG`、`CC_API_KEY`、`CC_USE_PROVIDER_MODELS`、`MODEL_REFRESH_INTERVAL_MS`） |
| fingerprint / session / keyState | isolate 内存态 `Map`：设备指纹伪装、12h+1h 抖动 session（`ensureSession`）、每 key 状态（`getOrCreateKeyState`）；冷启动重置 |
| `ensureInitialized` | 首次及每 8h+2h 抖动并行打 `/alpha/fingerprint/record` 与 `/alpha/lifecycle-events` 预请求 |
| `buildCcRequest` | OpenAI 请求 → CC `/alpha/generate` 请求体（system 抽取、多模态 image、tool-call/tool-result、`tool_choice` 映射） |
| `createSseTranslator` | CC NDJSON → OpenAI SSE chunk（含 tool_calls、usage、finish_reason 映射） |
| `forwardToCC` | 带伪装头（`x-session-id` / `x-project-slug` / `traceparent` / `x-command-code-version`）请求上游，上游恒定 `stream: true` |
| `convertAnthropicToOpenAI` / `handleMessages` | Anthropic Messages ↔ OpenAI 双向转换与 SSE 输出 |
| `fetchModels` / `handleModels` | `/provider/v1/models`（5 分钟缓存），失败回落硬编码 `MODELS` 列表 |
| `logUsage` / `log` | 结构化 usage 与普通日志，仅 `console`（靠 `[observability.logs]` 采集） |

## Conventions

- 所有逻辑写在 `src/index.js`；新增功能放入对应 `// ── 分区 ──`，不要引入打包产物或多文件拆分（`.gitignore` 的 `index.js` 是旧打包残留）。
- 只用 Web 平台 API + `node:crypto`；不得使用 Node 专有 API（`fs`/`net`/`process.env` 等）。
- 禁止 `setInterval` / 后台定时器；周期性任务一律惰性刷新（`maybeRefreshCCVersion`、`cleanupExpiredSessions`）。
- 上游 CC API 始终以 `stream: true` 调用；非流式响应在 Worker 内聚合后再返回 JSON。
- 客户端流式请求在拿到首个可见 chunk 前先缓冲，以便此时仍能返回 JSON 错误。
- 零输出 / 静默超时按 `429 + retry_after` 返回（SDK 会自动重试），不要改回 502/500；连续 3 次超时才提示压缩上下文（`TIMEOUT_REDUCE_CONTEXT_THRESHOLD`）。
- 错误体统一 `{ error: { message, type } }`；Anthropic 路径用 `anthropicErrorResponse`。所有响应经 `withCors`。
- 注释用中文，日志/错误文案用英文；`log(level, msg, data)` 传对象而非拼字符串。
- API key 必须是请求头里的 `user_` 前缀 key（`Authorization: Bearer` 或 `x-api-key`），`getApiKey` 负责提取，不要在别处重复解析。

## Notes

<!-- 待补充 -->
