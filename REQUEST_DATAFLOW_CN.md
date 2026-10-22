# Sub2API 请求全链路数据流

> 基于 `main` 分支（提交 `3a6fd1c9d`，版本 0.2.15）的代码整理，2026-10-20。
> 文件路径相对于 `backend/`。函数名与行为以代码为准。

---

## 0. 先回答核心问题：上游是不是"转给 CLI 执行"？

**不是。Sub2API 不启动、不调用任何 Claude Code / Codex / Gemini CLI 进程。**

它的做法是：**在 HTTP 协议层面"扮演"官方 CLI 客户端**，拿着订阅账号的 OAuth Token，自己用 Go 的 `net/http` 直接向厂商的 HTTPS 接口发请求，然后把上游返回的 SSE 流**逐个事件**转写给客户端。

| 误解 | 实际情况 |
|---|---|
| 收到请求 → 启动 `claude` / `codex` 进程 → 等进程输出 → 返回 | 收到请求 → 改写请求体和请求头，让它"长得像"官方 CLI 发出的 → 直接 HTTPS 调用 `api.anthropic.com` / `chatgpt.com/backend-api/codex` / `cloudcode-pa.googleapis.com` → 上游每吐一个 SSE 事件就转发一个 |
| 网关里跑着智能体循环（读文件、执行命令） | 网关**只转发单轮模型调用**。工具调用（读文件、跑命令）由用户电脑上的 Claude Code / Codex 客户端执行，执行结果作为下一轮请求再发过来 |
| "CLI" 指网关后端 | "CLI" 出现在两头：**客户端**是用户本机的 CLI；**上游凭据**是 CLI 专用的 OAuth 授权。中间的 Sub2API 是一个纯 HTTP 代理 |

**代码证据**：全仓非测试代码中 `exec.Command` 只有三类用途，没有一处启动 AI CLI：

| 位置 | 启动的进程 | 用途 |
|---|---|---|
| `internal/repository/backup_pg_dumper.go` | `pg_dump` / `psql` | 数据库备份与恢复 |
| `internal/service/plugin_runtime.go` | 插件二进制 | `.s2plugin` 插件（本机 gRPC 子进程） |
| `internal/platform/liveattestation/attestation_darwin.go` | `node` / `osascript` | macOS 上的设备认证辅助 |

> 对比：你们 agent-hub 的 **codex-pool 才是真正的"转给 CLI"方案**（见第 7 节），它用 `asyncio.create_subprocess_exec` 启动 `codex exec --json`，逐行读 stdout 的 JSON 事件再转成 SSE。两者架构完全不同。

---

## 1. 全景图

```
用户电脑                                   Sub2API (Go 单进程)                                   厂商
┌──────────────┐                ┌──────────────────────────────────────────────────┐
│ Claude Code  │ POST /v1/messages│ ① 中间件：请求体限制 → 请求ID → Ops 错误记录         │
│ Codex CLI    │ Bearer sk-xxx    │ ② API Key 鉴权（L1 → Redis → DB）+ 余额/订阅检查      │
│ Gemini CLI   │ ───────────────► │ ③ 分组准入：模型白名单 → 组合路由 → 分组校验            │
│ OpenAI SDK   │                  │ ④ Handler 预检：解析请求 → 渠道映射 → 内容审核          │
│ ...          │                  │    → 用户并发槽 → 计费资格                            │
│              │                  │ ⑤ 故障转移循环：                                     │
│ (工具在这里   │                  │    粘性会话 → 选账号(Redis快照/负载/并发槽) → 利润控制  │
│  本地执行)    │                  │ ⑥ 改写请求：伪装官方客户端(头/系统提示/指纹/beta)       │
│              │                  │    或协议转换(Messages⇄Responses⇄Chat⇄Gemini)        │
│              │                  │ ⑦ 取 OAuth Token(Redis 缓存, 临期刷新)               │  HTTPS (utls 指纹, 可走代理)
│              │                  │ ⑧ DoWithTLS ──────────────────────────────────────┼──────────────► api.anthropic.com
│              │   SSE 逐事件       │ ⑨ 读上游 SSE → 改写/转换 → 写回 + Flush ◄──────────┼─────────────── chatgpt.com/backend-api/codex
│              │ ◄─────────────── │    失败(首字节前) → 标记账号 → 换号重试 → 回到⑤       │                cloudcode-pa.googleapis.com
│              │                  │ ⑩ 结束：提取 usage → 计算费用 → 落账(事务) → 写日志    │                api.openai.com / api.x.ai ...
└──────────────┘                └──────────────────────────────────────────────────┘
                                      │ Redis：鉴权缓存、并发槽、粘性会话、调度快照、Token、计费缓存
                                      │ PostgreSQL：账号、用户、Key、分组、用量日志、余额
```

---

## 2. 分阶段详解（以 `POST /v1/messages` 流式请求为例）

### ① 路由与基础中间件

- 路由注册：`internal/server/routes/gateway.go`。同一端点按分组平台分发：`domain.UsesOpenAIGateway(platform)` 为真走 `OpenAIGatewayHandler.Messages`，否则走 `GatewayHandler.Messages`。
- 中间件顺序：`RequestBodyLimit` → `ClientRequestID` → `OpsErrorLoggerMiddleware` → `InboundEndpointMiddleware`。

### ② API Key 鉴权

位置：`internal/server/middleware/api_key_auth.go`（`apiKeyAuthWithSubscription`）

1. 先过无效凭据滥用限流（`rejectInvalidAuthAbuse`）。
2. 从 `Authorization: Bearer`、`x-api-key` 或 `x-goog-api-key` 取出 Key。
3. `APIKeyService.GetByKey`：L1（ristretto 进程内缓存）→ L2（Redis `apikey:auth:*`）→ DB，带 singleflight 和负缓存。
4. 校验：Key 状态、IP 黑白名单、用户是否启用、分组是否可用。
5. 简易模式直接放行；标准模式下：订阅分组查订阅额度（日/周/月），余额分组查余额阈值。
6. 把 Key、用户、分组写进 gin 上下文。

### ③ 分组准入

- `GroupModelAllowlist`：模型不在分组白名单直接拒绝。
- `compositeTargetPlatformMiddleware`：组合分组按模型名解析目标平台，可能改写请求体里的模型名。
- `RequireGroupAssignment`：确保 Key 绑定了分组。

### ④ Handler 预检

位置：`internal/handler/gateway_handler.go`（`Messages`）

1. `ParseGatewayRequest` 解析请求体（模型、stream、metadata.user_id、system 等）。
2. 推理强度策略；`ResolveChannelMappingAndRestrict` 做渠道级模型映射和限制。
3. 识别是不是**真实的 Claude Code 客户端**（`internal/service/claude_code_validator.go`：看 UA、system 提示词、metadata 等），并做版本检查。分组可以设"仅限 Claude Code"。
4. `checkSecurityAudit`：内容审核与提示词审计（同步拦截或异步审计）。
5. `AcquireUserSlotWithWait`：用户级并发槽（Redis `concurrency:user:*`），满了就排队等待。
6. `CheckBillingEligibility`：余额、订阅、Key 消费窗口、RPM、平台配额。

### ⑤ 账号调度（故障转移循环）

位置：`internal/handler/failover_loop.go`、`internal/service/gateway_scheduling.go`

1. 计算会话哈希 → 查粘性会话（Redis `sticky_session:*`，1 小时），同一会话尽量落到同一账号，提高 Prompt Cache 命中率。
2. `SelectAccountWithLoadAwareness`：
   - 从 Redis 调度快照（`sched:*`）取出该分组可调度的账号；
   - 按优先级、负载率、最近最少使用排序；
   - `tryAcquireAccountSlot` 抢账号并发槽；都满了就进等待队列。
3. 利润控制否决（亏本的调度会被拒）、绑定粘性会话、用户消息串行队列。
4. OpenAI 系分组用 `openai_account_scheduler.go` 的加权打分（错误率、首字延迟、配额余量等）。

### ⑥ 改写请求：伪装官方客户端

这一步是"像 CLI"的关键，**完全是字节级的 HTTP 改写**，没有任何进程调用。详见第 4 节。

位置：`internal/service/gateway_forward.go`（`Forward`）、`internal/service/gateway_upstream_request.go`（`buildUpstreamRequest`）、`internal/service/gateway_claude_oauth_body.go`

```go
// gateway_forward.go
isClaudeCode := IsClaudeCodeClient(ctx) || isClaudeCodeClient(clientUserAgent, parsed.MetadataUserID)
shouldMimicClaudeCode := account.IsOAuth() && !isClaudeCode   // OAuth 账号 + 非官方客户端 → 伪装
```

- 真实 Claude Code 客户端：基本**透传**它自己的头和请求体（只做必要的清洗）。
- 非 Claude Code 客户端（例如 OpenAI SDK 转过来的、Cherry Studio 等）走 OAuth 账号：**补齐**整套 Claude Code 特征。

### ⑦ 获取上游凭据

位置：`internal/service/gateway_service.go`（`GetAccessToken`）、`internal/service/claude_token_provider.go`

- OAuth / Setup Token 账号：先查 Redis Token 缓存（`oauth:token:*`），临近过期则刷新（`oauth:refresh_lock:*` 分布式锁，避免多副本同时刷新）。
- API Key 账号：直接用账号里的 Key。
- Bedrock：SigV4 签名；Vertex：Service Account。

### ⑧ 发往上游

```go
// gateway_forward.go
upstreamReq, wireBody, err := s.buildUpstreamRequest(upstreamCtx, c, account, body, token, tokenType, reqModel, reqStream, shouldMimicClaudeCode)
resp, err = s.httpUpstream.DoWithTLS(upstreamReq, proxyURL, account.ID, account.Concurrency, tlsProfile)
```

- 目标地址：`claudeAPIURL = "https://api.anthropic.com/v1/messages?beta=true"`（`gateway_service.go`），或账号配置的 Base URL。
- `DoWithTLS`：按账号绑定的代理（HTTP/SOCKS5）发出，并用 utls 模拟 Node/Claude Code 的 TLS 握手指纹（`internal/pkg/tlsfingerprint/dialer.go`）。
- **流式请求的上游上下文与客户端解绑**：

```go
// gateway_usage_billing.go
func detachStreamUpstreamContext(ctx context.Context, stream bool) (context.Context, context.CancelFunc) {
	...
	return context.WithoutCancel(ctx), func() {}
}
```

客户端中途断开，上游请求**不会被取消**，网关会继续把上游读完，拿到完整 usage 用于计费。

### ⑨ 回传响应（同步流式）

详见第 5 节。要点：上游每产生一个 SSE 事件，网关就解析、改写、写回、`Flush`，**不等全部生成完**。

### ⑩ 结束与计费

位置：`internal/service/gateway_usage_billing.go`（`RecordUsage` → `recordUsageCore`）、`internal/repository/usage_billing_repo.go`（`Apply`）

1. 成功后：账号 RPM 加一、绑定粘性会话，提交到 `UsageRecordWorkerPool`（异步，不阻塞响应）。
2. 计算费用：模型定价（分组 → 渠道 → 远程价格表 → 内置兜底）× 分组倍率 × 高峰倍率等。
3. `Apply` 在**一个数据库事务**里：按 request_id 去重 → 订阅用量 → 扣余额 → Key 额度和窗口 → 账号配额。
4. 更新 Redis 计费缓存、写 `usage_logs`、必要时发余额不足 / 配额告警邮件。

---

## 3. 流式请求时序图

```
Client(Claude Code)        Sub2API Gateway                         Redis/PG          Anthropic
      │ POST /v1/messages (stream:true) │                                │                 │
      │────────────────────────────────►│ 鉴权/准入/预检 ───────────────►│ 缓存/并发槽       │
      │                                 │ 选账号 ───────────────────────►│ sched:* 快照      │
      │                                 │ 取 Token ─────────────────────►│ oauth:token:*    │
      │                                 │ 改写请求(伪装 CLI)               │                 │
      │                                 │ HTTPS POST /v1/messages?beta=true (utls/代理) ──►│
      │                                 │◄──────────── 200 text/event-stream ──────────────│
      │◄─ event: message_start ─────────│ 记录 usage(input/cache) 改写模型名 Flush          │
      │◄─ event: content_block_delta ───│◄──────────── content_block_delta ───────────────│
      │◄─ event: content_block_delta ───│◄──────────── ... (逐 token)                      │
      │◄─ event: ping (keepalive) ──────│ 上游空闲时网关自己补 ping                         │
      │◄─ event: message_delta ─────────│ 记录 usage(output)                               │
      │◄─ event: message_stop ──────────│◄──────────── message_stop ──────────────────────│
      │                                 │ 释放并发槽 → 异步计费 ─────────►│ Apply 事务        │
```

---

## 4. "像 CLI"具体是怎么做到的

### 4.1 Claude（Anthropic OAuth 账号）

Claude Pro/Max 的 OAuth 凭据**只允许官方 Claude Code 使用**。上游会根据请求特征判断来源，判定为第三方会被降级到额外用量计费甚至拒绝。代码注释里写得很明白（`internal/pkg/claude/constants.go`）：

> Anthropic 上游会基于 anthropic-beta 的完整集合判定请求来源；缺少任何"官方 Claude Code 请求才会带"的 beta，都会被降级到第三方额度，对应报错：`Third-party apps now draw from your extra usage, not your plan limits.`

所以网关要逐项对齐官方 CLI 的流量特征：

| 特征 | 网关怎么做 | 位置 |
|---|---|---|
| 认证 | `authorization: Bearer <OAuth access token>` | `gateway_upstream_request.go` |
| User-Agent | `claude-cli/<版本> (external, cli)`，版本号内置基线 `2.1.258`，可由版本同步服务自动更新或用 `SUB2API_CLAUDE_CLI_VERSION` 覆盖 | `internal/pkg/claude/cli_version_runtime.go`、`constants.go`、`claude_code_version_sync_service.go` |
| SDK 头 | `X-Stainless-Lang: js`、`X-Stainless-Runtime: node`、`X-Stainless-Runtime-Version`、`X-Stainless-Package-Version`、`X-App: cli`、`Anthropic-Dangerous-Direct-Browser-Access: true` 等 | `constants.go`（`DefaultHeaders`） |
| anthropic-beta | 与真实 CLI 抓包一致的完整列表：`claude-code-20250219`、`oauth-2025-04-20`、`interleaved-thinking-...`、`prompt-caching-scope-...`、`effort-...`、`context-management-...` 等，顺序也对齐 | `constants.go`（`FullClaudeCodeMimicryBetas`） |
| 系统提示词 | 非官方客户端的请求，在 system 最前面注入 `You are Claude Code, Anthropic's official CLI for Claude.` | `gateway_service.go`、`gateway_claude_oauth_body.go` |
| metadata.user_id | 按账号指纹生成与官方客户端格式一致的 user_id | `gateway_forward.go`（`buildOAuthMetadataUserID`）、`identity_service.go` |
| 设备指纹 | 每个账号固定一套指纹（缓存在 Redis `fingerprint:*`），同一账号对上游始终呈现同一"设备" | `identity_service.go`（`GetOrCreateFingerprint`） |
| TLS 指纹 | utls 模拟 Node 的 ClientHello，模板可在后台管理 | `internal/pkg/tlsfingerprint/dialer.go`、`tls_fingerprint_profile_service.go` |
| cache_control | 统一管理缓存断点（最多 4 个）、TTL | `gateway_forward.go`（`rewriteMessageCacheControlIfEnabled`） |

**结论**：从 Anthropic 服务器的视角，它收到的是"一个 Claude Code CLI 发来的普通 HTTPS 请求"，而实际发送方是 Sub2API 的 Go 代码。

### 4.2 OpenAI（ChatGPT / Codex OAuth 账号）

```go
// openai_gateway_forward.go
switch account.Type {
case AccountTypeOAuth:
	// OAuth accounts use ChatGPT internal API
	targetURL = chatgptCodexURL          // https://chatgpt.com/backend-api/codex/responses
case AccountTypeAPIKey:
	// API Key accounts use Platform API or custom base URL
	...                                   // https://api.openai.com/v1/responses 或自定义
```

| 特征 | 网关怎么做 | 位置 |
|---|---|---|
| 端点 | ChatGPT 内部接口 `chatgpt.com/backend-api/codex/responses`（Codex CLI 用的就是它） | `openai_gateway_forward.go` |
| 身份头 | `originator`（Codex 客户端标识）、`version`（Codex 版本，自动同步）、`user-agent`、`OpenAI-Beta: responses=experimental` | `openai_codex_identity.go`（`ensureCodexIdentityHeaders`） |
| 账号头 | `chatgpt-account-id` | `openai_chatgpt_headers.go` |
| Prompt Cache | 自动注入 `prompt_cache_key` | `openai_gateway_*` |
| WebSocket 模式 | Codex 的 Responses WebSocket，网关维护上游 WS 连接池 | `openai_ws_pool.go`、`openai_ws_forwarder*.go` |

### 4.3 Gemini（Code Assist / Google One OAuth 账号）

| 特征 | 网关怎么做 | 位置 |
|---|---|---|
| 端点 | `https://cloudcode-pa.googleapis.com/v1internal:<action>`（Gemini CLI 使用的 Code Assist 内部接口） | `internal/pkg/geminicli/constants.go`、`gemini_messages_compat_service.go` |
| User-Agent | `GeminiCLI/...` | `internal/pkg/geminicli/constants.go` |
| 协议 | Claude / OpenAI 格式请求 → Gemini 原生格式（`contents`、`generationConfig`），响应再转回来 | `gemini_messages_compat_service.go`、`gemini_chat_completions_compat_service.go` |

### 4.4 API Key 账号

OpenAI / Anthropic / Grok / DeepSeek / Kimi / 智谱等的 API Key 账号**不需要伪装**，直接调官方开放平台（`api.openai.com`、`api.anthropic.com`、`api.x.ai` ……）。同协议零转换直通，跨协议才转换。

---

## 5. "同步流式返回"的实现细节

位置：`internal/service/gateway_upstream_response.go`（`handleStreamingResponse`）

### 5.1 结构：一个读协程 + 一个 select 主循环

```go
// 设置 SSE 响应头
c.Header("Content-Type", "text/event-stream")
c.Header("Cache-Control", "no-cache")
c.Header("Connection", "keep-alive")
c.Header("X-Accel-Buffering", "no")        // 告诉 Nginx 不要缓冲

scanner := bufio.NewScanner(resp.Body)      // 按行读上游 SSE（64K 起步的复用缓冲区）
// 独立 goroutine 读取上游，避免读取阻塞导致超时/keepalive无法处理
events := make(chan scanEvent, 16)
go func(...) { for scanner.Scan() { events <- ... } }()

for {
	select {
	case ev, ok := <-events:   // 上游来了一行/一个事件 → 处理 → 写回 → Flush
	case <-intervalCh:         // 上游太久没数据（stream_data_interval_timeout）→ 判超时、冷却账号
	case <-keepaliveCh:        // 下游空闲（stream_keepalive_interval）→ 主动写 event: ping，防代理/CF 断连
	}
}
```

### 5.2 每个事件做什么

1. 解析 `event:` 和 `data:` 行，组装成完整事件块。
2. **改写**：把上游实际模型名换回客户端请求的模型名（模型映射时），伪装模式下清理掉不该让客户端看到的内容。
3. **采集 usage**：`message_start` 里取输入 token 和缓存读写，`message_delta` 里取输出 token。按账号配置调整缓存 TTL 的计费归属。
4. **记录首字延迟**（firstTokenMs），用于调度打分和运维统计。
5. `fmt.Fprintf(w, ...)` 写回客户端，`flusher.Flush()` 立刻推送。

### 5.3 异常处理

| 情况 | 行为 |
|---|---|
| 上游在**首字节写出之前**返回 429 / 401 / 403 / 5xx / 网络错误 | `RateLimitService.HandleUpstreamError` 标记账号（冷却、刷新 Token、停用等），回到第 ⑤ 步**换号重试**（默认最多 10 次，Gemini 3 次）。客户端无感知 |
| 上游 400 签名错误、thinking 预算错误等 | 在同账号上修正请求体后重试（`gateway_forward.go` 里的 rectify 分支） |
| **已经开始向客户端写数据**后上游出错 | 不能再换号（HTTP 头已发出），只能在流内写 `event: error` 告知客户端 |
| 客户端中途断开 | 标记 `clientDisconnected`，**继续读完上游**以拿到完整 usage，照常计费 |
| 上游流结束但缺少终止事件 | 视为不完整，按部分 usage 结算（`partialStreamUsageResult`）并报错 |
| 上游长时间无数据 | 间隔超时触发，账号进入流超时冷却 |

### 5.4 跨协议的流式转换

客户端协议和上游协议不同时，网关用**流式状态机**逐事件翻译，仍然是"来一个转一个"，不会攒成完整响应再转：

```go
// gateway_forward_as_responses.go：上游 Anthropic SSE → 客户端 OpenAI Responses SSE
state := apicompat.NewAnthropicEventToResponsesState()
...
events := apicompat.AnthropicEventToResponsesEvents(event, state)   // 1 个上游事件 → 0..N 个下游事件
fmt.Fprintf(c.Writer, "event: %s\ndata: %s\n\n", eventType, restored)
c.Writer.Flush()
...
apicompat.FinalizeAnthropicResponsesStream(state)                    // 流结束补齐收尾事件
```

转换库在 `internal/pkg/apicompat/`：`anthropic_to_responses*.go`、`chatcompletions_responses_bridge.go`、`chatcompletions_anthropic_bridge.go` 等。状态机要处理的难点有：文本块与工具调用块的开始/增量/结束对齐、工具参数 JSON 分片、thinking 与签名、usage 汇总、各协议不同的结束标记。

### 5.5 非流式请求

`handleNonStreamingResponse`：读完整个响应体（有大小上限，防止异常响应撑爆内存）→ 解析 usage → 必要时转换协议 → 一次性返回 JSON。

---

## 6. 三条典型链路

### 链路 A：Claude Code → Claude OAuth 账号（最常见）

```
Claude Code ──/v1/messages──► 网关 识别为真实 Claude Code
  → 基本透传头和请求体（保留客户端 beta、system、metadata）
  → 换成号池里某个 Claude Max 账号的 OAuth Token
  → HTTPS api.anthropic.com/v1/messages?beta=true
  → SSE 原样逐事件回传（仅改模型名/采集 usage）
```

### 链路 B：Claude Code → OpenAI/Codex OAuth 账号（"让 Claude Code 用 GPT"）

```
Claude Code ──/v1/messages──► 网关 分组平台是 OpenAI → OpenAIGatewayHandler.Messages
  → Anthropic Messages 请求 → 转成 OpenAI Responses 请求（工具、system、thinking 映射）
  → 补 Codex 身份头（originator/version/chatgpt-account-id）
  → HTTPS chatgpt.com/backend-api/codex/responses
  → Responses SSE → 状态机转回 Anthropic SSE（message_start / content_block_* / message_delta）
  → Claude Code 以为自己在和 Claude 对话
```

### 链路 C：OpenAI SDK / 第三方客户端 → Claude OAuth 账号

```
SDK ──/v1/chat/completions──► 网关 → Chat Completions 转 Anthropic Messages
  → 非 Claude Code 客户端 + OAuth 账号 → 开启伪装：
    注入 "You are Claude Code..." 系统提示、完整 beta 列表、CLI UA、metadata.user_id、TLS 指纹
  → HTTPS api.anthropic.com
  → Anthropic SSE → 转回 Chat Completions 的 chunk 流
```

---

## 7. 对比：agent-hub 的两种做法

你们 agent-hub 里**恰好有两种模式**，可以直接对照：

| | **codex-pool（真·转给 CLI）** | **usercenter 网关 / Sub2API（HTTP 代理）** |
|---|---|---|
| 调用方式 | `asyncio.create_subprocess_exec("codex", "exec", "--json", ...)`（`agent-hub/codex/app/runner.py`） | `httpx.AsyncClient` 直连上游（`agent-hub/usercenter/app/gateway.py`）；Sub2API 用 Go `net/http` + utls |
| 输出读取 | 逐行读进程 stdout 的 JSON 事件（只有 4 类事件），解析后通过 `asyncio.Queue` 推成 SSE | 直接读上游 HTTP 响应的 SSE 字节流，逐事件转发 |
| 流式粒度 | CLI 输出什么粒度就是什么粒度（通常是"一步/一条消息"，不是逐 token） | 逐 token（上游 SSE 的原始粒度） |
| 每个请求的成本 | 启动一个进程（几十到几百 MB 内存、秒级启动），独占工作目录 | 一个 goroutine + 一个复用的 HTTP 连接，KB 级 |
| 并发 | 受进程数限制（每账号 `max_concurrent: 2`） | 受账号并发槽限制，单实例可承载大量并发流 |
| 智能体循环 | **在服务端**：CLI 自己执行工具（沙箱 `workspace-write`）、读写文件、多轮迭代后才结束 | **在客户端**：网关只转发一次模型调用，工具由用户本机执行 |
| 能力 | 能用 CLI 独有能力（如 codex 生图、文件产出），但只能拿到 CLI 暴露的结果 | 完整的协议能力（tools、thinking、cache_control、多模态），可做协议转换 |
| 适用场景 | 需要服务端代为"干活"（生图、跑任务） | 给已有客户端提供模型 API |
| 多实例 | 进程内状态 + 本地工作目录，只能单实例 | Sub2API 的状态在 Redis/PG，可多副本 |

**一句话**：codex-pool 把 CLI 当"执行器"用；Sub2API 把 CLI 当"身份"用——借用它的协议特征和 OAuth 授权，但完全绕过 CLI 程序本身。

---

## 8. 常见问题

**Q1：既然不跑 CLI，为什么要那么多"CLI 版本号"？**
上游会校验客户端版本和特征。版本过旧可能被拒（代码里有 `claude-cli >= 2.1.251` 之类的门槛判断，见 `internal/pkg/claude/cli_version.go`）。所以网关要跟着官方 CLI 的发版同步伪装版本号（`claude_code_version_sync_service.go`、`openai_codex_version_sync_service.go`）。

**Q2：Claude Code 里的工具调用（Bash、Read、Edit）在哪执行？**
在用户自己的电脑上。流程是：模型返回 `tool_use` → Claude Code 本地执行 → 把 `tool_result` 放进下一轮 `/v1/messages` 请求发给网关 → 网关再转发。每一轮对网关来说都是独立的一次请求，靠粘性会话尽量落到同一个上游账号，以复用 Prompt Cache。

**Q3：首字延迟主要花在哪？**
网关自身开销（鉴权、调度、改写）通常是毫秒级，因为几乎全走内存和 Redis。大头是上游模型的排队和推理时间。可以打开 `ENABLE_SERVER_TIMING` 看分段耗时（`internal/server/middleware/server_timing.go`）。

**Q4：如果想做 codex-pool 那样"服务端执行任务"，Sub2API 能做吗？**
不能直接做，它的定位是模型 API 网关。这类需求应该像 agent-hub 那样单独做一个执行器服务，然后让执行器本身通过 Sub2API 调模型。
