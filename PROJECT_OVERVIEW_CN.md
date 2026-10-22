# Sub2API 项目功能说明

> 基于 `main` 分支（提交 `3a6fd1c9d`，版本 0.2.15）的代码整理，整理日期 2026-10-16。
> 内容来自路由定义、数据模型、服务层代码、前端页面和 `docs/` 文档。上游平台、端点等清单以代码为准，可能随版本变化。

---

## 一、项目定位

Sub2API 是一个**可自行部署的 AI API 网关和运营平台**，主要做这几件事：

1. **聚合上游账号**：把 Claude、ChatGPT/Codex、Gemini、Grok 等 AI 订阅账号（OAuth 登录）和各厂商的 API Key 汇总成"号池"。
2. **对外发放统一的 API Key**：给每个用户发 `sk-` 开头的 Key。用户可以用 Claude Code、Codex CLI、Gemini CLI，或任何兼容 OpenAI/Anthropic/Gemini 协议的客户端直接调用。
3. **处理每个请求**：鉴权、选择上游账号、协议转换、并发和限流控制、故障转移、按 Token 精确计费。
4. **提供完整的运营后台**：用户注册登录、充值支付、订阅套餐、兑换码、分销返佣、公告、监控告警、风控审计。

一句话概括：**把 AI 订阅变成可分发、可计费、可运营的 API 服务。**

---

## 二、能完成哪些任务（典型场景）

| 场景 | 怎么实现 |
|---|---|
| **搭建 AI 中转站或多人拼车** | 管理员录入多个上游账号组成号池，给用户发 Key，按 Token 计费，用户自助充值或购买套餐 |
| **团队内部统一 AI 出口** | 用"简易模式"（`RUN_MODE=simple`）跳过计费和 SaaS 功能，只用来发 Key、限并发、看用量；可以给每个 Key 设 5 小时、1 天、7 天的消费窗口 |
| **让 Claude Code 用 GPT 或 Gemini，或让 OpenAI 客户端用 Claude** | 网关内置多种协议互转（Anthropic Messages、OpenAI Chat Completions、OpenAI Responses、Gemini 原生） |
| **用一个 Key 调用所有厂商的模型** | "组合分组"（Composite）按请求的模型名自动路由到 Anthropic、OpenAI、Gemini、Grok、Kimi、智谱、DeepSeek、MiniMax 等平台 |
| **商业化运营** | 内置支付（易支付、支付宝、微信、Stripe、Airwallex）、订阅套餐、余额充值赠送、兑换码、优惠码、邀请返佣、退款 |
| **多模态服务** | 生图（同步、异步、Gemini 批量）、视频（Grok、火山 Seedance）、语音（Grok TTS、STT、自定义音色、Realtime） |
| **稳定性运维** | 账号健康检测、限流自动冷却、故障转移、渠道监控、运维大盘、告警规则、邮件通知 |
| **合规与风控** | 内容审核（OpenAI Moderation、TypeSafe、关键词）、提示词安全审计（Qwen3Guard）、管理员操作审计日志 |
| **用脚本或 AI Agent 管理平台** | 管理员 API Key 加 `skills/sub2api-admin` 中的 CLI，可批量导入导出账号、管理兑换码、分组、代理等 |

---

## 三、整体架构

```
客户端：Claude Code / Codex CLI / Gemini CLI / OpenAI SDK / 任意兼容客户端
          │   Authorization: Bearer sk-xxxx
          ▼
┌───────────────────────────── Sub2API（单个 Go 二进制）─────────────────────────────┐
│ API Key 鉴权 → 分组与模型白名单准入 → 内容审核 / 提示词审计                         │
│ → 账号调度（粘性会话、负载感知、优先级） → 协议转换 → 发往上游（代理 / TLS 指纹） │
│ → 失败时切换账号重试 → 用量统计 → 计费扣费                                         │
│                                                                                    │
│ 内嵌 Web 界面（Vue 3）：管理后台 + 用户中心 + 安装向导                              │
└────────────────────────────────────────────────────────────────────────────────────┘
          │
          ▼
上游：Anthropic / OpenAI / Gemini / Antigravity / Grok / Kimi / 智谱 / DeepSeek / MiniMax / ...
存储：PostgreSQL（业务数据）+ Redis（缓存、并发槽、队列）+ 可选 S3（备份、异步图片结果）
```

**技术栈**：后端 Go、Gin、Ent ORM、Wire；前端 Vue 3、Vite、TailwindCSS；数据库 PostgreSQL 15+；缓存和队列 Redis 7+。前端编译后嵌入 Go 二进制（`-tags embed`），部署时只有一个文件。

### 核心概念

| 概念 | 含义 |
|---|---|
| **账号（Account）** | 一个上游凭据，例如某个 Claude Pro 的 OAuth 授权，或某个厂商的 API Key。属于某个平台，可以绑定代理，设置并发、优先级、模型映射 |
| **分组（Group）** | 账号池加计费规则。用户的 API Key 绑定到分组；分组决定平台、费率倍率、计费方式（余额或订阅）、模型白名单、媒体权限等 |
| **API Key** | 发给用户的 `sk-` 密钥。绑定分组，可以设置 IP 黑白名单、总额度、过期时间和消费窗口 |
| **渠道（Channel）** | 绑定若干分组，配置按平台的模型定价和模型映射 |
| **组合分组（Composite）** | 一种特殊分组：一个 Key 按请求的模型名自动路由到不同平台的账号 |
| **订阅（Subscription）** | 用户购买的套餐。在指定分组内按日、周、月的美元额度使用，不扣余额 |

---

## 四、上游接入：支持的平台与账号类型

代码中定义了 13 个具体平台，另有一个虚拟的 `composite` 平台（`backend/internal/domain/constants.go`、`platforms.go`）。

| 平台 | 支持的账号类型和授权方式 |
|---|---|
| **Anthropic（Claude）** | OAuth（PKCE 授权码，或用 Cookie/sessionKey 自动授权）；Setup Token（只有推理权限的长期令牌）；API Key（可开启直通，可自定义 Base URL）；AWS Bedrock（SigV4 签名或 API Key）；Vertex AI（Service Account） |
| **OpenAI（ChatGPT / Codex）** | ChatGPT/Codex OAuth（自动刷新）；Codex 个人访问令牌（PAT）；API Key（Responses 或 Chat Completions） |
| **Gemini** | OAuth，分 Code Assist、Google One、AI Studio 三种；AI Studio API Key；Vertex Service Account |
| **Antigravity** | Google OAuth（可以导入 refresh token，自动补全 project_id）；上游透传（Base URL + Key） |
| **Grok（xAI）** | xAI OAuth；SSO Token 转换；账号密码授权；API Key |
| **Kimi、智谱 GLM、DeepSeek、MiniMax** | API Key。可选按量付费或 Coding Plan 模式；上游协议可选 Chat Completions、Anthropic、Responses 或自适应 |
| **OpenCode Go** | Zen（按量）或 Go（订阅），按模型选择协议 |
| **Command Code、Cline** | API Key，按模型选择协议，可以从上游 `/models` 自动获取各模型支持的协议 |
| **TypeSafe（Jev）** | 只支持 API Key，专用于 `/v1/systemone` 评估接口 |

其他接入能力：

- **火山方舟 Seedance 视频**：挂在 OpenAI 平台的 API Key 账号上，需要在账号的端点能力里勾选 Seedance。
- **Ollama Cloud**：Base URL 指向 ollama.com 的 API Key 账号，支持查询用量。
- **Token 自动刷新**：后台定时为 Claude、OpenAI、Gemini、Antigravity、Grok 的 OAuth 账号刷新令牌。重试耗尽后，账号自动进入"临时不可调度"状态。
- **余额和配额查询**：Kimi、智谱、DeepSeek 可以查余额；OpenAI 可以查 Codex 配额；可以同步上游模型列表、探测上游计费倍率。

---

## 五、对外 API：端点与协议兼容

大部分端点同时提供 `/v1/...` 和不带前缀的根路径版本（`backend/internal/server/routes/gateway.go`）。

### 1. 文本对话

| 端点 | 说明 |
|---|---|
| `POST /v1/messages` | Anthropic Messages 协议（Claude Code 使用） |
| `POST /v1/messages/count_tokens` | 计算 Token 数 |
| `POST /v1/chat/completions` | OpenAI Chat Completions 协议 |
| `POST /v1/responses`、`/v1/responses/*` | OpenAI Responses 协议（Codex CLI 使用），包含 compact、input_tokens 等子路径 |
| `GET /v1/responses` | Responses 的 WebSocket 模式 |
| `POST /v1/embeddings` | 向量嵌入（仅 OpenAI 平台） |
| `GET /v1/models`、`/v1/models/:model` | 模型列表；Codex 客户端请求时返回 Codex 模型清单格式 |
| `/v1beta/models/{model}:generateContent` 等 | Gemini 原生协议（Gemini CLI 使用），包括流式和 countTokens |
| `/backend-api/codex/*` | Codex 直连路径 |
| `/antigravity/v1/messages`、`/antigravity/v1beta/*` | Antigravity 专用端点，只调度 Antigravity 账号 |
| `POST /v1/live`、`/backend-api/codex/realtime/calls` | OpenAI 实时通话，需要分组开启 |
| `POST /v1/alpha/search` | Codex 独立网页搜索，按次计费 |
| `POST /v1/systemone` | TypeSafe Jev 评估协议 |
| `GET /v1/usage`、`/v1/sub2api/billing` | 查询当前 Key 的用量和计费信息 |

### 2. 协议转换

客户端用哪种协议，和上游账号是哪个平台，可以不一致。网关会自动转换：

| 客户端发送 | 上游实际调用 | 典型用途 |
|---|---|---|
| Anthropic Messages | OpenAI Responses 或 Chat Completions | **让 Claude Code 使用 GPT/Codex 模型**，分组可以配置模型映射 |
| OpenAI Chat Completions | Anthropic Messages | 让 OpenAI SDK 调用 Claude |
| OpenAI Responses | Anthropic Messages | 让 Codex 类客户端调用 Claude |
| Responses 和 Chat Completions 互转 | 双向 | 适配只支持其中一种协议的上游 |
| Anthropic Messages、Chat Completions | Gemini 原生 | 用 Claude 或 OpenAI 格式调用 Gemini |
| Claude、Chat、Responses | Antigravity 内部协议 | 通过 Antigravity 使用 Claude 和 Gemini |
| 国产及多协议厂商 | 按厂商协议决定 | 同协议零转换直通，跨协议才转换 |

### 3. 多模态

| 能力 | 端点 | 支持的平台 |
|---|---|---|
| 同步生图、改图 | `/v1/images/generations`、`/v1/images/edits` | OpenAI（经 Responses 的 image_generation 工具）、Grok |
| 异步生图 | `/v1/images/generations/async`、`/v1/images/tasks/:id` | OpenAI、Grok。需要先配置对象存储，用来避免 Cloudflare 524 等长连接超时 |
| 批量生图 | `/v1/images/batches*`（提交、查询、下载、取消、删除） | Gemini API 和 Vertex 的 Batch 接口，支持幂等、预扣费和结算 |
| 视频生成、编辑、扩展 | `/v1/videos*` | Grok；组合分组也可以使用 |
| Seedance 视频 | `/api/v3/contents/generations/tasks` 等 | 火山方舟 Ark 原生协议，直接透传 |
| 语音 | `/v1/tts`、`/v1/stt`、`/v1/custom-voices*`、`/v1/realtime`（WebSocket） | Grok |
| 搜索 | `/v1/web_search`、`/v1/x_search` | Grok |

---

## 六、智能调度与高可用

### 账号选择

- **Claude 系**：先看粘性会话，再按分组的模型路由规则选优先账号，然后做负载感知选择（依次比较优先级、负载率，可选优先"最早重置"的账号，最后按最近最少使用）。都不可用时进入排队。Claude OAuth 账号还有窗口费用、RPM 和最大会话数等限制。
- **OpenAI 系**：先按 `previous_response_id` 找原账号，再看会话粘性，最后加权打分（优先级、负载、排队、错误率、首字延迟，可选配额余量和上游成本），在得分最高的几个账号里加权随机。粘性账号不健康时会主动换号。
- **粘性会话**：同一会话固定到同一账号，以提高 Prompt Cache 命中率。会话标识依次取自 `metadata.user_id`、cache_control 内容和全文摘要，有效期 1 小时。

### 故障转移和健康管理

- **自动切换账号**：请求失败时自动换号重试，默认最多 10 次，Gemini 为 3 次；支持同账号重试和退避。
- **按错误码处理**：401 时刷新令牌或停用账号；429 时解析重置时间并冷却，Claude 的 5 小时、7 天窗口单独处理；529 标记为过载；403、402 等按平台分别处理。
- **可配置规则**：
  - 临时不可调度规则：按"状态码 + 关键词 + 时长"把账号暂时移出调度。
  - 错误透传规则：把上游错误原样或改写后返回给客户端。
  - 自定义错误码、流式超时冷却。
- **三级并发控制**：账号、用户、API Key 三级并发槽位，外加等待队列。还有用户消息串行队列，同一账号上的真实用户消息按 RPM 自适应排队。
- **网络层**：
  - 每个账号可以绑定代理，代理失效时自动改用备用代理或直连。
  - 代理支持连通性测试和质量检测。
  - TLS 指纹模拟（基于 utls，模拟 Claude Code/Node 的握手特征），指纹模板可以在后台管理。
- **混合调度**：Antigravity 账号开启混合调度后，也可以服务通用的 `/v1/messages` 和 `/v1beta` 请求。

### 模型控制

- 模型映射可以在账号、渠道、分组三个层级配置，分组还可以设置默认映射模型。
- 分组模型白名单：不在白名单里的模型直接拒绝。
- 组合分组路由表：对外的模型别名可以映射到"目标平台 + 上游模型 + 端点 + 优先级"，后台可以预览路由结果。
- 推理强度（reasoning effort）可以设上限和映射规则。

### 客户端专项适配

- **Claude Code**：
  - 识别真实的 Claude Code 客户端，分组可以设为"仅限 Claude Code"，并指定降级分组。
  - 对 OAuth 账号，按官方客户端的请求特征重写请求。
  - 自动同步 Claude Code 版本号。
  - 拦截预热请求。
  - 配置 anthropic-beta 请求头策略。
- **Codex**：
  - Fast/Flex 策略：对 `service_tier` 可以选择透传、移除、拒绝或强制 priority，可以按用户、模型、账号类型设置规则。
  - 分组可以设置"Fast 按标准价收费"。
  - 支持模型清单、版本同步、仅限 Codex CLI、Responses WebSocket。
- **Prompt Cache**：
  - 网关统一管理 cache_control 断点，最多 4 个。
  - 可以按账号覆写缓存 TTL。
  - OpenAI 请求自动注入 `prompt_cache_key`。
  - Grok 生成租户隔离的缓存标识。
- **网页搜索模拟**：上游不支持 web_search 工具时，网关可以调用 Brave 或 Tavily 搜索，再合成 Anthropic 格式的响应。

---

## 七、计费体系

- **两种计费方式**：
  - 余额计费：按量扣费，支持冻结余额。
  - 订阅计费：分组设为订阅类型，按日、周、月的美元额度使用。
- **价格来源**（优先级从高到低）：分组定价 → 渠道定价 → 远程价格表（LiteLLM 格式，带哈希校验）→ 内置价格兜底。
- **计费维度**：
  - 文本：输入、输出、缓存读写 Token（包括 1 小时缓存和长上下文阶梯）。
  - 按次计费：图片（按 1K、2K、4K 分档，批量有折扣）、视频（按分辨率或按秒）、音频、搜索。
  - **推理强度倍率**：不同 reasoning effort 使用不同价格。
- **多层倍率**：分组倍率、高峰时段倍率、针对特定用户的分组倍率；账号还有成本倍率，用来核算实际成本。
- **用户平台配额**：每个用户在每个平台上可以设置日、周、月的消费上限。
- **API Key 额度**：可以给单个 Key 设置总额度、过期时间，以及 5 小时、1 天、7 天的消费窗口。
- **利润控制**：
  - 网关层可以设置最低毛利和安全缓冲，亏本的调度会被拒绝。
  - 配套离线工具 `cmd/profit-preview`，可以预览分组的利润情况。
- **计费熔断**：计费出现异常时可以配置为直接拒绝请求（fail-closed）。
- **用量日志**：每次请求都有明细记录；管理员可以按条件异步批量清理历史日志。

---

## 八、用户端功能（用户中心）

| 页面 | 功能 |
|---|---|
| **仪表盘** | 用量统计、趋势图、最近请求、各平台配额、快捷入口 |
| **API 密钥** | 创建和管理 Key；绑定分组；IP 白名单和黑名单；总额度；过期时间；消费窗口；查看每个 Key 的每日用量 |
| **使用记录** | 请求日志、错误详情、统计 |
| **我的订阅** | 当前套餐，以及日、周、月额度的使用进度 |
| **充值和购买** | 余额充值（支持阶梯赠送）、购买订阅套餐；支付方式包括易支付、支付宝、微信支付、Stripe（页面或弹窗）、Airwallex；支持扫码支付页、结果页、取消订单、申请退款 |
| **我的订单** | 订单历史 |
| **兑换码** | 兑换余额、并发、订阅等，查看兑换记录 |
| **邀请返佣** | 邀请码、返佣记录，返佣额度可以转入余额 |
| **个人资料** | 头像、密码、TOTP 双因素认证、Passkey 通行密钥、第三方账号绑定（邮箱、LinuxDo、微信、钉钉、OIDC 等）、余额提醒 |
| **渠道状态** | 查看各渠道和模型的可用性 |
| **可用渠道** | 查看可以使用的分组、倍率和模型 |
| **模型广场** | 公开页面，列出模型的官方参考价和各分组的实际价格 |
| **批量生图指南** | 批量生图任务的使用说明，以及任务列表和下载 |
| **Key 用量查询** | 公开页面，不登录也能凭 Key 查询用量 |
| **自定义页面** | 管理员通过自定义菜单嵌入的外部页面（iframe 或 HTML），例如工单系统 |
| **其他** | 公告、法律文档（用户协议等）、首页 |

---

## 九、管理后台功能

所有管理接口都经过管理员鉴权（JWT 或管理员 API Key）、面板限流、操作审计和合规确认。导出、备份恢复、插件变更、清空审计日志等敏感操作，还需要再输入一次 TOTP 验证码。

### 资源管理

| 模块 | 功能 |
|---|---|
| **仪表盘** | 今日请求、Token、活跃用户；实际成本和账号成本对比；平均响应时间；Token 趋势；模型分布；分组统计；用户消费排行 |
| **账号管理** | 新建、复制、批量创建；JSON 导入导出；从 CRS 同步；导入 Codex 会话；各平台 OAuth 授权向导；测试账号，以及定时测试计划；刷新令牌（支持批量）；清除错误、限流或不可调度状态；切换是否参与调度；重置配额；批量修改；批量更新凭据；同步上游模型 |
| **分组管理** | 费率倍率、高峰倍率、用户专属倍率和 RPM；余额或订阅类型；日、周、月额度；模型路由、降级分组、组合路由；模型白名单；媒体权限和按次价格；推理强度策略；Fast 策略；利润控制；仅限 Claude Code、仅限 OAuth 等限制 |
| **渠道定价** | 渠道绑定分组，按平台配置模型价格、模型映射、计费模型来源、模型限制、网页搜索模拟等 |
| **渠道监控** | V1 主动探测：定期用请求模板测试，保留历史和日汇总；V2 被动监控：根据真实流量生成模型矩阵、错误和用户统计。两种模式二选一 |
| **用户管理** | 增删改查；调整余额（保留历史）；批量修改并发和限额；各平台配额；自定义用户属性；更换分组；绑定第三方身份；查看用户的 Key 和用量 |
| **订阅管理** | 分配、批量分配、续期、重置额度、撤销和恢复 |
| **使用记录** | 全站请求日志、统计、按用户或 Key 搜索、按条件异步清理 |
| **兑换码** | 类型有余额、并发、订阅、邀请码；生成、导出、批量修改、作废；"创建并兑换"一步完成，可以对接外部发卡系统 |
| **优惠码** | 赠送金额、使用次数、有效期、使用记录 |
| **代理管理** | 增删改查、导入导出、连通性测试、质量检测、查看绑定的账号 |
| **公告** | 按条件投放、通知方式、生效时间、已读统计 |
| **错误透传规则 / TLS 指纹模板** | 见第六节 |

### 商业化

| 模块 | 功能 |
|---|---|
| **支付大盘** | 收入统计 |
| **订单管理** | 取消、重试发货、退款、查询退款状态 |
| **套餐管理** | 价格、原价、有效期、权益说明、是否上架 |
| **支付服务商实例** | 每种支付方式可以配置多个实例，支持负载均衡、限额、退款开关 |
| **分销管理** | 邀请、返佣、转账记录；按用户设置专属返佣比例；登记线下提现 |

### 运维与安全

| 模块 | 功能 |
|---|---|
| **运维监控（Ops）** | 实时并发；账号可用性；实时 QPS（WebSocket 推送）；吞吐、延迟、错误趋势和错误分布图；OpenAI Token 统计；请求错误、上游错误、入口拒绝三类错误日志（可标记已处理）；请求详情；系统日志和运行时日志级别；**告警规则、告警事件、静默**；**邮件告警和定期报告** |
| **风险控制** | 内容审核（OpenAI 或 TypeSafe 引擎）；关键词和哈希拦截；用户解封；网络安全类会话拦截；白名单。默认关闭 |
| **提示词审计** | 通过兼容 OpenAI 协议的 Qwen3Guard 扫描用户输入，覆盖暴力、隐私信息、越狱等 9 类风险；可以选择异步审计或同步拦截 |
| **审计日志** | 记录管理员操作，可以设置保留期；清空日志需要 TOTP |
| **备份** | 备份到 S3，支持定时备份、下载、恢复；在这里配置异步生图使用的对象存储 |
| **插件** | 上传 `.s2plugin` 签名包，启用或停用、配置、测试、按比例灰度 |
| **系统** | 查看版本、检测更新、在线升级、回滚、重启 |

### 系统设置（9 个标签页）

常规、协议（法律文档）、功能开关、安全、用户、网关、支付、邮件、备份。主要配置项：

- **站点**：名称、Logo 等品牌信息；"后台模式"（只开放管理后台）；自定义菜单（嵌入外部系统）。
- **注册与登录**：注册开关、默认余额和并发、验证码、各种第三方登录。
- **安全**：API Key 访问控制；管理员 API Key；CDN 场景下的客户端 IP 识别。
- **网关**：调度参数、过载、429 和流式超时的冷却时间、anthropic-beta 策略、网页搜索模拟、Claude Code 相关设置、Codex Fast/Flex 策略、上游计费探测、用量记录设置。
- **通知与邮件**：余额、配额、订阅到期提醒；SMTP 配置，支持发测试邮件和编辑邮件模板。

---

## 十、认证与账户安全

- **注册**：
  - 邮箱注册，可以开启验证码验证。
  - 可以限定允许的邮箱后缀。
  - 可以开启"仅限邀请注册"（使用邀请类兑换码）。
  - 注册时可以填写优惠码。
- **登录**：
  - 密码登录，可以要求 TOTP 双因素认证。
  - Passkey（WebAuthn）登录。
  - 可以撤销所有会话。
- **人机验证**：支持 Cloudflare Turnstile、腾讯验证码、阿里云验证码。登录、注册、发送验证码等接口在服务端也有限流，Redis 故障时直接拒绝请求。
- **第三方登录**：
  - 支持 LinuxDo、微信、钉钉、OIDC、GitHub、Google。
  - 首次登录时可以选择新建账号，或绑定到已有账号。
  - GitHub 和 Google 用已验证的邮箱自动注册或登录。
- **找回密码**：可以开启或关闭。
- **部署安全配置**（`config.yaml`）：
  - CORS 白名单、上游 URL 白名单（防 SSRF）、响应头过滤、CSP。
  - 可信代理和 CDN 客户端 IP 请求头。
  - 限制非流式上游响应的读取大小，防止内存被异常响应撑爆。

---

## 十一、扩展与集成

- **插件系统**：
  - 插件是独立进程，打包成签名的 `.s2plugin` 文件，通过本机 gRPC 与 Sub2API 通信。
  - 多实例部署时由数据库自动同步插件包，插件还可以使用 Redis 支撑的键值存储。
  - 目前只开放一种能力：接管 OpenAI OAuth 账号的出站 HTTP 请求。
  - 开发文档：`docs/PLUGIN_DEVELOPMENT.md`、`backend/pkg/pluginapi/`。
- **管理 API 和 AI Agent Skill**：`skills/sub2api-admin` 是一个给 AI Agent 用的 Skill，附带 Node CLI，用管理员 API Key 或 JWT 批量管理账号、兑换码、分组、代理、错误规则、TLS 指纹，也能发起任意管理 API 调用。
- **外部支付和发卡对接**：外部系统可以调用"一步创建并兑换"和余额调整接口，购买页可以透传 URL 参数（见 `docs/ADMIN_PAYMENT_INTEGRATION_API.md`）。
- **外部系统嵌入**：通过自定义菜单，以 iframe 方式把工单等外部系统嵌入后台或用户中心。
- **移动端**：社区项目 [sub2api-mobile](https://github.com/ckken/sub2api-mobile) 提供 iOS、Android 管理端。
- **命令行工具**（`backend/cmd/`）：
  - `jwtgen`：从数据库签发管理员 JWT。
  - `profit-preview`：离线预览分组利润。
  - `cleanup-ingress-reject-logs`：清理历史入口拒绝日志，默认只预演不删除。
- **数据管理守护进程 `datamanagementd`**：可选，部署在宿主机上，通过 Unix Socket 与主程序通信，提供数据源、S3 配置和备份任务管理（见 `deploy/DATAMANAGEMENTD_CN.md`）。

---

## 十二、部署与运行模式

| 方式 | 说明 |
|---|---|
| **一键脚本** | `deploy/install.sh`：从 GitHub Releases 下载二进制，自动安装 systemd 服务 |
| **Docker Compose** | `deploy/docker-deploy.sh` 自动生成密钥；数据可以放在本地目录或命名卷 |
| **Apple container** | 在 macOS 26 和 Apple 芯片上本地运行 |
| **源码编译** | 先编译前端，再用 `go build -tags embed` 把前端打进二进制 |

- **安装向导**：首次启动时在浏览器里配置数据库、Redis 和管理员账号，带安装锁。
- **在线升级**：后台一键升级，带校验和原子替换，支持回滚。
- **简易模式**（`RUN_MODE=simple`）：适合个人或团队内部使用，隐藏 SaaS 功能，跳过计费。生产环境需要额外设置 `SIMPLE_MODE_CONFIRM=true` 才能启动。
- **HTTP/2**：明文端口支持 h2c，并保留 HTTP/1.1 回退，用于 WebSocket 和旧客户端。
- **界面语言**：管理后台和用户中心有中文、英文两种；README 有中、英、日三种。

---

## 十三、使用风险提示

README 中的原文提醒：

- 使用本项目**可能违反 Anthropic 等上游服务商的服务条款**，账号封禁等风险由使用者自行承担。
- 项目仅供技术学习与研究，**作者未授权任何商业化运营**。
- 许可证为 **LGPL v3.0** 或更高版本。

---

## 附：关键代码位置

| 内容 | 位置 |
|---|---|
| 网关路由 | `backend/internal/server/routes/gateway.go` |
| 管理、用户、认证、支付路由 | `backend/internal/server/routes/{admin,user,auth,payment}.go` |
| 平台和账号类型常量 | `backend/internal/domain/constants.go`、`platforms.go` |
| 数据模型 | `backend/ent/schema/*.go` |
| 调度与故障转移 | `backend/internal/service/gateway_scheduling.go`、`openai_account_scheduler.go`、`backend/internal/handler/failover_loop.go` |
| 限流与账号健康 | `backend/internal/service/ratelimit_service.go` |
| 协议转换 | `backend/internal/pkg/apicompat/`、`backend/internal/service/gateway_forward_as_*.go`、`openai_gateway_*.go` |
| 计费 | `backend/internal/service/billing_service.go`、`model_pricing_resolver.go` |
| 支付服务商 | `backend/internal/payment/provider/` |
| 前端页面 | `frontend/src/views/{admin,user,auth}/` |
| 专题文档 | `docs/`（支付、组合分组、异步生图、批量生图、Seedance、插件开发、渠道监控） |
