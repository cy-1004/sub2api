# Sub2API 微服务拆分与分布式部署 PRD

| 项目 | 内容 |
|---|---|
| 文档版本 | v0.1（草案，待评审） |
| 编写日期 | 2026-10-19 |
| 基线代码 | `main` 分支，提交 `3a6fd1c9d`，版本 0.2.15 |
| 参考方案 | `C:\workspace\yagent-service\agent-hub`（usercenter + codex-pool） |
| 关联文档 | `PROJECT_OVERVIEW_CN.md`（功能说明） |
| 状态 | 草案。第 11 节"待决问题"需要评审时确认 |

> **本文的前提假设**（评审时如果不成立，相关章节要调整）：
> 1. 生产环境运行在 Kubernetes 上；测试环境和小规模部署用 docker-compose 多副本方案。
> 2. 技术栈保持 Go，单仓库、单 `go.mod`，不改写成其他语言。
> 3. 前端 Vue 应用调用的 HTTP 接口（`/api/v1/**`）和对外网关协议（`/v1/**` 等）**保持兼容，不改动**。
> 4. agent-hub 只作为模式参考，不和它合并。是否把用户中心并入 agent-hub usercenter，放在待决问题里。

---

## 目录

1. [背景、目标与非目标](#1-背景目标与非目标)
2. [现状分析](#2-现状分析)
3. [参考方案 agent-hub：借鉴与差异](#3-参考方案-agent-hub借鉴与差异)
4. [设计原则](#4-设计原则)
5. [目标架构](#5-目标架构)
6. [关键设计](#6-关键设计)
7. [部署方案](#7-部署方案)
8. [实施路线图](#8-实施路线图)
9. [非功能需求与验收指标](#9-非功能需求与验收指标)
10. [风险与应对](#10-风险与应对)
11. [待决问题](#11-待决问题)
12. [附录](#12-附录)

---

## 1. 背景、目标与非目标

### 1.1 背景

Sub2API 目前是**一个 Go 单体进程**，做三件事：

- **AI 网关数据面**：鉴权、调度、协议转换、转发、故障转移、计费。
- **运营控制面**：管理后台、用户中心、支付、订阅、监控告警。
- **全部后台任务**：Token 刷新、聚合、对账、备份、探测等。

前端编译后嵌入同一个二进制。随着业务增长，暴露出这些问题：

| 痛点 | 表现 |
|---|---|
| 数据面和控制面互相影响 | 管理后台的大查询（使用记录、仪表盘）、批量导入、备份等操作，和网关请求抢同一进程的 CPU、内存、DB 连接池和 Redis 连接池，网关延迟会抖动 |
| 扩容粒度太粗 | 网关需要横向扩容时，只能把控制面和全部后台任务一起复制。每个副本都会启动约 45 个后台服务（见 2.3） |
| 发布风险集中 | 改一个后台页面也要重启整个网关，长连接（SSE/WebSocket）会被打断。目前停机等待只有 5 秒（`backend/cmd/server/main.go`） |
| 故障域太大 | 任何一个模块的 panic 风暴、内存泄漏或 goroutine 泄漏，都会直接影响对外 API |
| 多副本部署不完整 | 代码已经部分支持多副本，但还有多处进程内状态会出错（见 2.4） |
| 团队协作成本高 | `internal/service` 是一个扁平的包（约 573 个文件、24 万行），模块之间没有编译期边界 |

### 1.2 目标

| 编号 | 目标 | 衡量方式 |
|---|---|---|
| G1 | **能多副本、分布式部署**：任何服务都能跨节点、跨可用区运行 N 个副本，没有单点、不重复执行任务、不丢状态 | 第 9 节的验收指标 |
| G2 | **数据面和控制面隔离**：网关单独部署、单独扩缩容、单独发布 | 控制面压测时，网关 P99 附加延迟不受影响 |
| G3 | **按领域拆成微服务**：每个领域服务独立拥有自己的数据，通过契约（gRPC 和事件）协作 | 跨服务没有直接访问对方表的情况；契约有 CI 校验 |
| G4 | **计费零丢失、可对账**：异步化之后，用量事件不丢、不重复扣费，可以追溯 | 每日对账差异为 0 |
| G5 | **可观测**：有链路追踪、指标、结构化日志，以及健康检查和就绪检查 | 一个请求能从 Ingress 一直追踪到上游 |
| G6 | **渐进可回滚**：每个阶段都能单独上线、单独回滚，并保留一体化部署能力 | `--services=all` 能运行完整功能，现有测试全部通过 |

### 1.3 非目标

- 不重写业务逻辑，不改变调度、计费、协议转换的**业务行为**。
- 不改变对外网关协议，不改变前端 API 契约，不重写前端。
- 不在本期引入多地域多活（只保留扩展能力，见 P4）。
- 不追求"一个表一个服务"式的过度拆分。服务粒度按数据归属和扩缩容特征来定。

---

## 2. 现状分析

> 以下结论来自代码梳理，文件路径相对于仓库根目录。

### 2.1 架构与规模

```
客户端 (Claude Code / Codex CLI / Gemini CLI / SDK)
        │
        ▼
┌──────────────────────── sub2api 单进程 (Gin, :8080) ─────────────────────────┐
│ 路由: /v1 /v1beta /backend-api/codex /antigravity  (网关)                     │
│       /api/v1/auth /api/v1/* /api/v1/admin/* /api/v1/payment/*  (控制面)      │
│       /  (嵌入的 Vue 前端)   /setup (安装向导)                                 │
│ Wire DI: config → repository(~40 仓储 + ~33 缓存) → service(~130 provider)    │
│          → handler(20 用户 + 36 管理) → server                                │
│ 后台任务: 约 45 个 Start()/Stop() 服务, 都在 Provide* 中启动                  │
└───────────────────────────────────────────────────────────────────────────────┘
        │                      │                       │
   PostgreSQL               Redis (单节点)          S3 (可选) / SMTP / 支付 / 上游 AI
```

| 维度 | 现状 | 位置 |
|---|---|---|
| 入口 | `main.go`、`wire.go`、`wire_gen.go`（758 行，自动生成） | `backend/cmd/server/` |
| 业务包 | `internal/service`：573 个非测试文件，约 24.3 万行（含测试约 53.5 万行） | `backend/internal/service/` |
| DI | 一个大 `service.ProviderSet`（1082 行，约 130 个 provider） | `backend/internal/service/wire.go` |
| 数据模型 | 39 个 Ent schema，另有约 60 张表只存在于 SQL 迁移中 | `backend/ent/schema/`、`backend/migrations/`（315 个文件） |
| 外键 | `REFERENCES users` 31 处、`groups` 14 处、`accounts` 6 处、`api_keys` 5 处 | `backend/migrations/` |
| 路由 | gateway、admin、user、auth、payment、common、model_plaza | `backend/internal/server/routes/` |
| Redis | 单节点客户端，约 60 段 Lua 脚本，6 个 Pub/Sub 频道，没有用 Streams | `backend/internal/repository/redis.go`、`*_cache.go` |
| gRPC | 已经是直接依赖（插件系统、datamanagementd） | `backend/pkg/pluginapi/v1/plugin.proto`、`backend/internal/service/data_management_grpc.go` |
| 可观测 | zap 日志；内置 Ops 子系统，数据存在 PG 的 `ops_*` 表；没有 Prometheus、OTel 和 pprof；`/health` 恒定返回 ok | `backend/internal/pkg/logger/`、`backend/internal/server/routes/common.go` |
| 部署 | 单个 Dockerfile（前端嵌入，`-tags embed`）；compose 写死了 `container_name`；有 systemd 和 Caddy；没有 K8s | `Dockerfile`、`deploy/` |

### 2.2 已有的分布式基础（可以直接复用）

代码已经为多副本做了不少工作，拆分时应当**复用这些机制，不要另起炉灶**：

| 机制 | 说明 | 位置 |
|---|---|---|
| 领导者锁 | 优先用 Redis `SET NX` 加 Lua 比较后删除；Redis 不可用时退回 PG `pg_try_advisory_lock` | `backend/internal/service/leader_lock.go`（`tryAcquireSingletonLeaderLock`）、`backend/internal/repository/leader_lock_cache.go`、`backend/internal/service/ops_advisory_lock.go` |
| 调度快照和事务性 Outbox | 账号写入时同时写 `scheduler_outbox`。各副本约每秒轮询一次，重建 Redis 中按分组 × 平台 × 模式分桶的版本化快照（`sched:*`） | `backend/internal/service/scheduler_snapshot_service.go`、`backend/internal/repository/scheduler_outbox_repo.go` |
| 鉴权缓存失效 Outbox | PG 触发器写入 `auth_cache_invalidation_outbox`，Worker 认领后执行 DEL 和 PUBLISH，并做二次失效 | `backend/migrations/184_auth_cache_invalidation_outbox.sql`、`backend/internal/service/auth_cache_invalidation_outbox.go` |
| L1 加 Pub/Sub 失效 | API Key 鉴权、订阅、渠道、错误透传规则、TLS 指纹、提示词审计配置 | `backend/internal/service/api_key_auth_cache_impl.go`、`subscription_service.go`、`channel_service.go` 等 |
| 认领和租约 | `FOR UPDATE SKIP LOCKED`（用量清理）；带租约的任务认领（提示词审计）；Redis 队列（批量生图） | `backend/internal/repository/usage_cleanup_repo.go`、`backend/internal/securityaudit/prompt_worker.go`、`backend/internal/repository/batch_image_queue.go` |
| 幂等 | `usage_billing_dedup` 按 request_id 去重；`idempotency_record` | `backend/internal/repository/usage_billing_repo.go` |
| 共享运行时状态 | 三级并发槽、等待队列、粘性会话、RPM、用户消息队列、余额缓存、临时不可调度标记都已放在 Redis | `backend/internal/repository/concurrency_cache.go` 等 |
| 预扣（in-flight 预留） | `billing:inflight:*` | `backend/internal/service/billing_inflight_reservation.go` |
| 迁移加锁 | 迁移在启动时执行，用 advisory lock 互斥，带 checksum | `backend/internal/repository/migrations_runner.go` |

### 2.3 后台任务现状

所有后台任务都在 `backend/internal/service/wire.go` 的 `Provide*` 里直接调用 `Start()`，**每个副本都会启动全部任务**，也没有按角色开关。

| 类别 | 任务举例 | 多副本是否安全 |
|---|---|---|
| 已加领导者锁 | 仪表盘聚合，Ops 指标采集、聚合、告警、清理、报表，订阅到期提醒，支付订单对账，定时备份，上游计费探测，Ollama/OpenCode 用量轮询等，约 15 个 | 安全 |
| 认领/租约队列 | 用量清理、鉴权失效 Outbox、批量生图、提示词审计、平台配额刷写、调度快照 | 安全，可以横向扩展 |
| 去重执行 | OAuth Token 刷新：各副本都扫描，实际刷新由 `oauth:refresh_lock:*` 去重 | 能用，但扫描是重复的 |
| **没有协调** | `scheduled_test_runner_service.go`（定时测试）、`channel_monitor_runner.go`（渠道探测）、`cn_provider_balance_check_service.go`、两个版本同步服务、`pricing_service.go`（各副本各自下载价格表） | **会重复请求上游**，必须修复 |
| 进程内缓冲 | 用量记录 Worker 池（`usage_record_worker_pool.go`）、usage_log 批量写、邮件队列（`email_queue_service.go`，容量 100，没有持久化）、审计日志、系统日志 Sink | **进程崩溃会丢数据** |

### 2.4 多副本部署的阻碍（P0 必须解决）

| # | 问题 | 位置 | 影响 |
|---|---|---|---|
| B1 | Claude、OpenAI、Gemini、Antigravity 的 OAuth 授权会话放在进程内 map 里，只有 xAI 用了 Redis | `backend/internal/pkg/oauth/oauth.go`、`backend/internal/pkg/openai/oauth.go`、`backend/internal/pkg/geminicli/oauth.go`、`backend/internal/pkg/antigravity/oauth.go`；可复用 `backend/internal/pkg/redissession/store.go` | 管理员在副本 A 生成授权链接、回调落到副本 B 时会失败 |
| B2 | 定时测试和渠道探测没有跨副本协调 | 见 2.3 | 对上游的请求数随副本数成倍增加，可能触发上游风控 |
| B3 | 嵌入前端的 HTML 缓存（注入了公共设置）没有 TTL，只在处理更新请求的那个副本上失效 | `backend/internal/web/html_cache.go`、`backend/internal/server/router.go` | 其他副本一直返回旧的站点配置，直到重启 |
| B4 | 设置只靠 TTL（30–60 秒）同步，`SetOnUpdateCallback` 只在本副本执行 | `backend/internal/service/setting_service.go`、`setting_gateway_runtime.go` | 配置生效时间不一致 |
| B5 | `TOTP_ENCRYPTION_KEY` 没配置时，每个进程随机生成一个 | `backend/internal/config/config.go` | 各副本之间无法解密 TOTP 和 S3 密钥 |
| B6 | 自定义页面、`data/public` 前端覆盖、价格文件、插件包写在本地磁盘 | `backend/internal/handler/page_handler.go`、`backend/internal/service/plugin_manager.go` | 副本之间数据不一致 |
| B7 | 用量和邮件放在进程内队列，停机只等 5 秒 | `backend/cmd/server/main.go`、`backend/internal/service/usage_record_worker_pool.go` | 滚动发布时打断长流、丢用量 |
| B8 | `/health` 不检查任何依赖，没有区分 liveness 和 readiness | `backend/internal/server/routes/common.go` | K8s 会把流量导到还没就绪的实例 |
| B9 | 在线升级用 `os.Rename` 替换磁盘上的二进制 | `backend/internal/service/update_service.go` | 和不可变镜像、多副本冲突 |
| B10 | 无效凭据滥用限流器放在进程内 | `backend/internal/service/invalid_auth_abuse_limiter.go`、`deploy/EDGE_SECURITY.md` | 限流阈值会随副本数被放大 |
| B11 | OpenAI 的启发式数据（EWMA 错误率、TTFT）只在本副本统计 | `backend/internal/service/openai_account_scheduler.go` | 各副本调度判断有偏差（可以接受，但要知道） |

### 2.5 耦合热点（拆分的主要成本）

1. **一个横跨四个领域的事务**：`usageBillingRepository.Apply` → `applyUsageBillingEffects`（`backend/internal/repository/usage_billing_repo.go`）在同一个 SQL 事务里依次做这些事：
   - 写入 `usage_billing_dedup`（去重）
   - 更新 `user_subscriptions`（订阅用量）
   - 扣减 `users` 的余额
   - 更新 `api_keys` 的额度和 5 小时、1 天、7 天窗口
   - 更新 `accounts.extra` 的账号配额

   这是拆分计费时**最大的阻碍**。
2. **构造函数依赖过多**：`NewGatewayService` 有 28 个参数，`NewOpenAIGatewayService` 22 个，`NewAdminService` 23 个，`ProvideOpsService` 14 个（它依赖全部 4 个网关服务），`ProvideAuthService` 15 个。
3. **反向依赖和环**：账号测试、Token 刷新、用量查询服务都反过来依赖 `OpenAIGatewayService` 和 `AntigravityGatewayService`。代码用 setter 注入来绕开循环依赖（`SetOpenAIGatewayService`、`SetPluginManager`、`SetScheduler` 等）。
4. **进程级全局钩子**：`logger.SetSink`、`middleware.SetIngressRejectRecorder`、`service.SetWebSearchManager`、`claude.SetCLIVersionResolver` 等。
5. **`users` 和 `groups` 是被几乎所有领域引用的中心表**。`usage_log` 关联 user、api_key、account、group、subscription。
6. **`SettingService`、`AccountRepository`、`*config.Config` 各自被 20 多个 provider 注入**。

### 2.6 网关热路径（拆分时必须保持零额外跨服务调用）

```
RequestBodyLimit → ClientRequestID → OpsErrorLogger → InboundEndpoint
→ APIKeyAuth（L1 ristretto → Redis apikey:auth: → DB；状态、IP、用户、分组、订阅/余额检查）
→ GroupModelAllowlist → CompositeTarget → RequireGroupAssignment
→ Handler：解析请求、渠道映射、内容审核、提示词审计、用户并发槽、CheckBillingEligibility
→ 故障转移循环：粘性会话 → SelectAccountWithLoadAwareness（Redis sched:* 快照、负载、并发槽）
             → 利润控制 → 用户消息队列 → Forward（指纹、TLS、Token、上游）→ 出错时 RateLimitService 处理并换号
→ 成功后：RPM、粘性绑定 → UsageRecordWorkerPool → RecordUsage → 计算费用 → Apply（大事务）
         → Redis 缓存更新 → 写 usage_log → 余额或配额通知
```

关键文件：`backend/internal/server/middleware/api_key_auth.go`、`backend/internal/handler/gateway_handler.go`、`backend/internal/handler/failover_loop.go`、`backend/internal/service/gateway_scheduling.go`、`backend/internal/service/gateway_forward.go`、`backend/internal/service/gateway_usage_billing.go`。

**结论**：热路径上的读操作**基本已经走缓存和 Redis 快照**，写操作集中在最后的计费落账。所以合适的拆分方式是：**读走各服务发布的快照，写变成事件**。

---

## 3. 参考方案 agent-hub：借鉴与差异

### 3.1 agent-hub 实际是什么

agent-hub 是同一个仓库里的两个 Python FastAPI 服务，**彼此之间没有调用**（`agent-hub/PROJECT_OVERVIEW.md`）：

- **usercenter**：账号、API Key、兼容 OpenAI 和 Anthropic 的 LLM 网关、计量、计费、套餐、管理后台。使用 PG 和 Redis。
- **codex-pool**：Codex CLI 账号池，SSE 生图，用 SQLite 存储，只能单实例运行。

它没有 gRPC/IDL、服务发现、配置中心、独立 API 网关、应用容器化、K8s、CI，也没有链路追踪和指标。所以对本项目来说，**它提供的是一组可复用的工程模式，而不是一套现成的分布式基础设施**。

### 3.2 借鉴清单

| 模式 | agent-hub 实现 | 在本方案中的用法 |
|---|---|---|
| Redis Streams 消费组 | 计量事件用 `XADD stream:usage`（`maxlen`）写入；消费组 `metering`，消费者名为 `hostname-pid`；`XREADGROUP`、`XAUTOCLAIM`（空闲 60 秒的消息重新认领）、`XACK`（`usercenter/app/metering.py`） | 事件总线的首个实现（6.1）。消费者名改成 Pod 名，**另外增加死信流**，避免 agent-hub 那种"解析失败就 ack 丢弃"的处理 |
| 基于 request_id 的幂等 | `usage_logs` 用 `ON CONFLICT (request_id, ...) DO NOTHING`，结算用 `SET NX` 标记 | 与 sub2api 已有的 `usage_billing_dedup` 合并，作为消费端统一的幂等规范 |
| 预留 → 结算加 TTL 补偿 | 配额先预留，结算时扣减，预留超时自动释放；Redis 计数器丢失时从 `usage_daily` 重建；每日对账（`usercenter/app/quota.py`、`tasks.py`） | 计费异步化后的一致性模型（6.3）。复用 sub2api 的 `billing_inflight_reservation`，**新增对账任务** |
| 分层严格配置 | default.yaml → local.yaml → 环境变量里的密钥；`extra="forbid"`，未知配置项让启动失败；`settings check` 命令（`usercenter/app/settings.py`） | 各服务分段配置、`s2a config check`、密钥只从环境变量或 Secret 注入（6.5） |
| 统一错误包 | `{"error":{"code","message"}}`（`usercenter/app/context.py`） | sub2api 的控制面已经有 `{code,message,reason,metadata,data}`（`backend/internal/pkg/response/response.go`）。内部 gRPC 用 `google.rpc.ErrorInfo{reason, metadata}` 一一对应 |
| JWT 分受众 | `iss=usercenter`，`aud=user/admin`，访问令牌 15 分钟；刷新令牌是不透明令牌，换发时 GETDEL，可以整体吊销（`usercenter/app/auth.py`、`security.py`） | identity 服务签发令牌时带 `aud` 区分 user、admin、internal（6.4） |
| API Key 只存哈希加短 TTL 缓存 | `ak:{hash}` 缓存 300 秒，状态变化时删除（`usercenter/app/keys.py`） | sub2api 已经有更完整的 L1、L2 和 Outbox 机制，保持不变 |
| 用户状态即时生效 | 每个请求都回查用户行 | 改成从 identity 发布的快照读取用户状态和 token_version（6.4），避免每次都查库 |

### 3.3 不能照搬的地方（agent-hub 的短板）

| agent-hub 的做法 | 问题 | 本方案的做法 |
|---|---|---|
| `workers.embedded: true`，多实例时要手动只在一个实例上开启 | 容易配错，导致任务重复或没人执行 | 按角色部署，加领导者锁或认领机制，配置写错也不会重复执行 |
| 用量事件发布失败只记日志，靠对账发现（`gateway.py`） | 计费会有缺口 | 先写流，失败时同步降级，再失败写本地 spool，最后靠对账兜底（6.3） |
| 没有链路追踪、指标、结构化日志 | 分布式环境下无法排障 | OTel、Prometheus、JSON 日志（6.9） |
| codex-pool 的调度状态放在进程内存 | 只能单实例运行 | 调度状态全部放在 Redis（sub2api 已经做到） |
| 对称密钥 JWT，其他服务要校验只能共享密钥 | 密钥扩散 | 阶段性改为非对称签名加 JWKS（6.4） |
| 没有容器化和编排 | 无法分布式部署 | Helm、K8s、HPA、PDB（第 7 节） |

---

## 4. 设计原则

1. **渐进式（绞杀者模式）**：路径是模块化单体 → 按角色部署 → 领域服务独立。每一步都能单独发布、单独回滚，**不做大爆炸式重写**。
2. **一个镜像，多种组合**：同一个二进制用 `s2a serve --services=<列表>` 启用任意模块组合。同进程的模块之间用本地适配器调用，跨进程的用 gRPC。**`--services=all` 永远可用**，用于小规模部署、简易模式，以及继续合并上游代码。
3. **网关热路径零同步跨服务调用**：网关只读快照（L1 → Redis → 未命中才走 gRPC），写操作都以事件发出。冷路径（例如 Token 过期刷新）允许同步 RPC，但必须有超时、熔断和单飞（singleflight）。
4. **数据归属唯一**：每张表只有一个拥有者服务，其他服务通过 API、事件或快照访问。先在逻辑上划清归属（PG 角色授权），再做物理拆分。
5. **至少一次投递，消费端幂等**：所有事件带 `event_id` 和业务幂等键，消费端必须幂等。涉及钱的流程同时有预留、结算和对账三层保障。
6. **兼容优先**：外部协议、前端契约和数据库迁移都遵循 expand/contract，新旧版本能同时运行（滚动发布的前提）。
7. **出站一致**：凡是对上游 AI 发起请求的组件，出口 IP、TLS 指纹和代理策略都必须一致，避免分布式部署触发上游账号风控（6.8）。
8. **够用就好**：服务粒度按数据归属、扩缩容特征和变更频率来定。能合并部署的就合并部署，等真有需要再拆开。

---

## 5. 目标架构

### 5.1 总体架构图

```
                                   ┌──────────────────────────────┐
  Claude Code / Codex / SDK ──────►│  边缘: CDN/WAF + Ingress      │◄────── 浏览器 (管理后台/用户中心)
  支付回调 (支付宝/微信/Stripe…) ──►│  (Nginx Ingress / Envoy GW)   │
                                   └──┬──────────┬───────────┬────┘
              /v1 /v1beta /backend-api│          │/api/v1/** │/api/v1/payment/webhook/*
              /antigravity /api/v3 …  │          │  + index  │
                                      ▼          ▼           ▼
                         ┌────────────────┐ ┌─────────┐ ┌───────────┐
       数据面 ────────── │  gateway (×N)  │ │ console │ │ commerce  │
                         │  HPA, 无状态     │ │  (BFF)  │ │ (webhook) │
                         └──┬───┬─────┬───┘ └────┬────┘ └─────┬─────┘
           读快照(L1→Redis)  │   │事件  │内部gRPC    │gRPC         │gRPC/事件
          ┌─────────────────┘   │     │(探测/测试)  ▼             ▼
          │      ┌──────────────┼─────┴────► ┌──────────────────────────────────────────┐
          │      │              │            │          控制面领域服务 (gRPC :9090)          │
          │      │              │            │ identity │ resource │ billing │ commerce   │
          │      │              │            │ platform │ ops      │ media(worker)         │
          │      │              │            └───┬──────────┬──────────┬─────────────────┘
          ▼      ▼              ▼                │ 发布快照   │ Outbox   │ 消费事件
   ┌───────────────────┐  ┌──────────────────────┴──────────┴──────────┴───────┐
   │ redis-state (HA)  │  │ 事件总线: Redis Streams (首选) → Kafka (升级路径)      │
   │ 快照/并发/计费/流   │  └─────────────────────────────────────────────────────┘
   │ redis-cache       │  ┌──────────────────────┐  ┌──────────────────────────────┐
   └───────────────────┘  │ PostgreSQL HA        │  │ S3: 图片/备份/页面/插件/前端覆盖 │
                          │ 按服务分 schema/角色   │  └──────────────────────────────┘
                          │ (热点库后续独立)       │  ┌──────────────────────────────┐
                          └──────────────────────┘  │ OTel Collector / Prometheus / │
                                                    │ Loki / Grafana / Alertmanager │
                                                    └──────────────────────────────┘

   上游 AI ◄── 固定出口 (NAT / Egress Gateway / 账号代理) ◄── gateway、media 的全部上游出站流量
```

### 5.2 服务清单

| 服务 | 平面 | 职责 | 拥有的数据（详见附录 A） | 扩缩容特征 | 主要来源代码 |
|---|---|---|---|---|---|
| **gateway** | 数据面 | 所有对外 AI 端点：API Key 鉴权与准入、模型白名单和组合路由、内容审核（同步）、账号调度、三级并发槽、粘性会话、协议转换、转发、故障转移、预扣、发出用量事件。另外提供**内部上游探测 RPC**（账号测试、渠道探测）。托管插件进程 | **不拥有业务表**，只拥有 Redis 运行时状态：并发、粘性、RPM、健康计数、用户消息队列 | CPU 和连接数密集；按并发数和 CPU 做 HPA；副本最多 | `gateway_*.go`、`openai_*`、`gemini_*`、`antigravity_*`、`grok_*`、`internal/pkg/apicompat`、`internal/handler/gateway_*` |
| **console** | 接入层（BFF） | 承接前端全部 `/api/v1/**`（支付回调除外）：JWT 校验、管理员 API Key、面板限流、审计、二次验证、合规确认、聚合查询；托管 `index.html`（注入公共设置）；首次安装 | 不拥有业务表 | 流量低，可以 2–3 副本固定 | `internal/server/routes/{admin,user,auth}.go`、`internal/server/middleware/`、`internal/handler/admin/` |
| **identity** | 控制面 | 注册、登录（密码、TOTP、Passkey、第三方 OAuth、验证码）、刷新令牌、签发 JWT 和 JWKS、用户资料与属性、API Key 管理、发布鉴权快照 | 用户、身份、凭据、API Key 配置 | 流量低；登录有突发 | `auth_service.go`、`user_service.go`、`api_key_service.go`、`totp_service.go`、`passkey.go` |
| **resource** | 控制面 | 号池与路由配置：账号、代理、TLS 指纹、错误透传规则、分组、渠道与定价配置、组合路由、模型映射、远程价格表；上游 OAuth 授权向导、Token 刷新、账号测试计划、配额和余额探测、版本同步；**构建调度快照** | 账号、代理、分组、渠道、定价配置、调度 Outbox | 中等；Worker 用领导者锁 | `account_*`、`admin_account.go`、`proxy_*`、`group_*`、`channel_*`、`token_refresh_service.go`、`scheduler_snapshot_service.go`、`pricing_service.go` |
| **billing** | 控制面 | 账本：钱包余额（带流水）、订阅权益与用量、API Key 用量窗口、用户平台配额、账号配额计数；消费用量事件并落账；usage_log 的存储与查询；仪表盘聚合；对账 | 钱包、流水、订阅、用量日志、聚合表、配额计数 | 写入密集（用量）；消费者可以水平扩展 | `billing_*`、`gateway_usage_billing.go`、`usage_*`、`subscription_service.go`、`dashboard_*`、`user_platform_quota_*` |
| **commerce** | 控制面 | 交易：支付订单、支付服务商实例、回调、退款、套餐、兑换码、优惠码、邀请返佣；用 Outbox 加幂等命令完成发货 | 订单、套餐、兑换码、优惠码、返佣 | 流量低；回调要求高可用 | `payment_*`、`internal/payment/`、`redeem_service.go`、`promo_service.go`、`affiliate_service.go` |
| **media** | 数据面 Worker | 异步生图、批量生图（Gemini/Vertex Batch）、视频任务轮询（Grok、Seedance）、S3 结果存储、媒体计费的冻结、扣款和释放 | 批量任务表、Redis 任务队列 | Worker 密集；按队列长度扩缩 | `image_task.go`、`batch_image_*.go`、`grok_media.go`、`seedance.go` |
| **ops** | 支撑 | 运维监控（指标、错误日志、告警规则和事件、静默、报表、实时大盘）、渠道监控 V1/V2、风控（内容审核日志、关键词）、提示词审计 Worker、管理员审计日志 | `ops_*`、渠道监控、审计、审核相关表 | 写入密集（日志）；消费者可以水平扩展 | `ops_*.go`、`channel_monitor_*`、`content_moderation*.go`、`internal/securityaudit/`、`audit_log_service.go` |
| **platform** | 支撑 | 系统设置中心（按命名空间发布快照并广播变更）、公告、自定义页面、通知与邮件（持久化队列、模板）、备份与恢复、版本信息 | 设置、公告、页面元数据、邮件队列 | 流量低 | `setting_*.go`、`announcement_service.go`、`email_*`、`notification_email_service.go`、`backup_*` |

> **为什么 media 归在数据面**：媒体任务要调用上游，需要复用 gateway 的上游执行内核（调度、Token、代理、TLS 指纹）。它和 gateway **在同一个代码模块里，但以独立角色 `media-worker` 部署**。任务数据和队列归 media 所有。
>
> **为什么账号测试和渠道探测要经过 gateway**：所有对上游的出站流量都从 gateway 或 media 的 Pod 发出，这样出口 IP、指纹和代理策略才一致（原则 7），也不需要在 resource 和 ops 里再复制一份转发代码。

### 5.3 部署分组（同一镜像，按规模组合）

| 规模 | Deployment 组合 | 适用场景 |
|---|---|---|
| **S（3 个部署）** | `gateway`（×N）；`control` = console + identity + resource + billing + commerce + platform + ops 的 API；`worker` = 全部 Worker（含 billing 消费者、media、ops） | 中小站点；P1 阶段的落地形态 |
| **M（6–7 个部署）** | `gateway`、`console`、`core`（identity + resource + platform）、`billing`、`commerce`、`ops`、`media-worker` | 有独立团队维护计费或支付；日请求在千万级 |
| **L（全拆）** | 9 个服务各自独立，每个服务再拆成 `-api` 和 `-worker` | 大规模、多团队、需要独立 SLO |

```bash
# 示例：同一镜像不同启动参数
s2a serve --services=gateway
s2a serve --services=console,identity,resource,billing,commerce,platform,ops --no-workers
s2a serve --services=billing,ops,media,resource,platform --workers-only
s2a serve --services=all          # 一体化（兼容现状、简易模式）
s2a migrate up                    # 迁移（K8s Job）
s2a config check                  # 配置校验
```

---

## 6. 关键设计

### 6.1 通信与契约

**同步调用用 gRPC**：

- **契约**：放在 `backend/api/proto/<service>/v1/*.proto`，用 `buf` 做 lint、生成代码和 breaking change 检查（CI 强制）。
- **服务发现**：用 K8s DNS。客户端使用 `dns:///<svc>-headless.<ns>.svc.cluster.local:9090` 加 `round_robin`，配合 headless Service，避免 HTTP/2 长连接导致负载不均。compose 环境用 Docker DNS。**不引入 Nacos 或 Consul**。
- **调用规范**：
  - 必须设置 deadline，并沿调用链传递。
  - 只有幂等的 RPC 允许自动重试，最多 2 次，带指数退避和抖动。
  - 写操作类 RPC 必须带 `idempotency_key`。
- **错误模型**：使用 `google.rpc.Status` 加 `ErrorInfo{reason, metadata}`，与现有 HTTP 响应包 `{code,message,reason,metadata,data}`（`backend/internal/pkg/response/response.go`）一一对应，console 透传 reason，前端无感知。
- **元数据透传**：`x-request-id`、W3C `traceparent`、`x-s2a-actor`（user_id、admin_id、role）、`x-s2a-caller`（调用方服务名）。
- **本地适配器**：每个领域对外暴露一个 Go 接口（`internal/modules/<svc>/api`），同时有本地实现和 gRPC 客户端两种实现。目标模块在同一进程时自动走本地实现，所以 `--services=all` 的部署没有网络开销。

**异步通信用事件总线**：

- **抽象接口**：`internal/kit/eventbus`，提供 `Publish(ctx, topic, key, msg)` 和 `Subscribe(group, topic, handler)`。
- **首个实现用 Redis Streams**（不新增中间件，延续 agent-hub 的模式）：
  - 每个 topic 一个 stream，`XADD ... MAXLEN ~ N`。
  - 消费组按服务命名，消费者名用 Pod 名。
  - `XAUTOCLAIM` 认领空闲超过 60 秒的消息。
  - 失败超过 N 次的消息转入 `<topic>:dlq` 并告警。
  - 积压（lag）作为指标导出。
- **升级路径是 Kafka（或 Redpanda）**，满足以下任一条件时切换：用量事件持续超过约 3k/s；需要按天回放或保留；需要多个分析系统订阅；Redis 内存压力过大。因为有抽象接口，业务代码不用改。
- **事件封装**（protobuf，`api/proto/events/v1/envelope.proto`）：`event_id`(ULID)、`type`、`schema_version`、`occurred_at`、`producer`、`partition_key`、`trace_context`、`payload`。
- **通用 Outbox**：新增 `outbox_events` 表，业务事务内写入，Relay Worker 认领后投递到总线。现有的 `scheduler_outbox` 和 `auth_cache_invalidation_outbox` 保持原有语义继续运行，P3 阶段再逐步并入。
- **广播类消息**（配置变更、缓存失效）**继续用 Pub/Sub**，以 TTL 兜底。它们的特点是允许丢失、只需要最终一致。

**跨服务调用矩阵**（目标形态，"—"表示没有直接依赖）：

| 调用方 → 被调方 | identity | resource | billing | commerce | platform | ops | gateway(内部) |
|---|---|---|---|---|---|---|---|
| **gateway** | 快照（未命中时 gRPC） | 快照，冷路径 RPC 取 Token | 快照加事件 | — | 快照 | 事件 | — |
| **console** | gRPC | gRPC | gRPC | gRPC | gRPC | gRPC | — |
| **identity** | — | 快照（分组可见性） | 事件 `user_registered` | gRPC（校验优惠码和邀请码） | 事件（邮件） | 事件（审计） | — |
| **commerce** | gRPC（用户校验） | — | gRPC 命令（入账、开通订阅） | — | 事件（通知） | 事件 | — |
| **billing** | — | 快照（定价、分组规则） | — | — | 事件（余额提醒） | — | — |
| **resource** | — | — | — | — | 快照 | 事件 | gRPC（账号测试） |
| **ops** | gRPC（查询展示） | gRPC | — | — | 事件（告警邮件） | — | gRPC（渠道探测） |
| **media** | 快照 | 快照 | gRPC 命令（冻结、扣款、释放） | — | — | 事件 | 进程内复用 |

### 6.2 网关读模型：快照发布协议

网关需要的所有控制面数据，都由**拥有该数据的服务**以"快照"形式发布到 Redis。网关只读，不直接查控制面的表。这把现有的 `apikey:auth:`、`sched:*`、`billing:*` 做法统一成一个正式契约。

**协议规定**：

1. **键**：`snap:{<domain>}:<entity>:<id>`。值是 protobuf 编码，带 `version`（单调递增，取自 DB 序列或 `updated_at` 加序号）和 `schema_version`。
2. **更新**：拥有者服务在业务事务中写 Outbox，Relay 写入快照后 `PUBLISH snap:invalidate:<domain>`，消息内容为 `{entity, id, version}`。
3. **读取**：网关依次查 L1（ristretto，带随机 TTL 抖动），再查 Redis（singleflight），都没有时调用拥有者的 gRPC `GetSnapshot`，结果顺手回填 Redis。也支持负缓存。
4. **版本保护**：L1 只接受版本号更高的值，防止乱序覆盖。
5. **兜底**：Pub/Sub 消息丢了，靠 L1 的短 TTL 修正；拥有者定期全量重建，修正 Redis 中的快照。
6. **敏感字段**：账号凭据（OAuth Token、API Key）在快照里**用信封加密**，只有 gateway 和 media 持有数据密钥（KMS 或 K8s Secret）。

**网关读取的快照清单**：

| 快照 | 拥有者 | 现状对应 | 内容 |
|---|---|---|---|
| API Key 鉴权 | identity | `apikey:auth:*`（L1、L2 和 Outbox） | key → 状态、IP 规则、过期时间、额度配置、user_id、group_id |
| 用户状态 | identity | 包含在鉴权快照中 | 状态、并发上限、token_version、允许的分组 |
| 分组、渠道、组合路由、模型映射、定价 | resource | `channel_cache_updated`、go-cache | 分组规则、倍率、白名单、媒体价格、利润控制参数 |
| 调度快照 | resource | `sched:*`（已存在） | 按分组 × 平台 × 模式分桶的可调度账号 |
| 错误透传规则、TLS 指纹 | resource | Pub/Sub 加本地副本 | 规则集 |
| 余额、订阅用量、Key 窗口、平台配额 | billing | `billing:*`、`apikey:rate:*`、`billing:upq:*` | 资格判断用的实时计数 |
| 网关运行时设置、功能开关 | platform | `SettingService` 的 TTL | 按命名空间（`gateway.*`、`features.*`） |
| 内容审核和提示词审计配置 | ops | `sub2api:prompt_guard:config:invalidate` | 审核策略 |

**网关写出的数据**：

| 写入 | 方式 | 消费方 |
|---|---|---|
| 并发槽、粘性会话、RPM、用户消息队列、临时不可调度、健康计数 | 直接写 Redis（网关自己拥有的运行时状态） | 网关各副本 |
| 用量 `usage.recorded` | 事件 | billing（落账）、ops（被动监控 V2） |
| 账号健康信号 `account.health_signal`（429 及重置时间、401、403、529、超时、恢复） | 先在 Redis 立即标记，再发事件 | resource 持久化账号状态 → 调度 Outbox → 快照 |
| 请求错误、入口拒绝 | 事件 | ops |
| 提示词审计任务 | 事件 | ops（风控） |
| Token 过期刷新 | **冷路径同步 RPC** `resource.EnsureFreshToken`（singleflight，加 `oauth:refresh_lock:`） | resource |

### 6.3 计费账本与一致性

#### 6.3.1 归属调整

为了把 `usageBillingRepository.Apply` 变成 **billing 服务内部的本地事务**，把"会被计费修改的计数"迁到 billing 名下：

| 现在的位置 | 迁移后（billing 拥有） | 说明 |
|---|---|---|
| `users.balance`（及冻结余额） | `billing.wallets`，另加只追加的 `billing.wallet_ledger` | 余额作为流水的物化值；充值、扣费、退款、调账、返佣转入、兑换都记流水 |
| `user_subscriptions` 的用量字段 | `billing.user_subscriptions`（订阅权益整体归 billing） | commerce 和兑换只通过命令"开通、续期、撤销" |
| `api_keys` 的额度用量和 5 小时、1 天、7 天窗口 | `billing.api_key_usage_counters` | Key 的**额度配置**仍归 identity，**用量**归 billing |
| `accounts.extra` 的配额用量 | `billing.account_quota_counters` | 超额时发 `billing.quota_exhausted` 事件，resource 据此标记账号 |
| `usage_logs`、`usage_billing_dedup`、`billing_usage_entries`、仪表盘聚合表 | billing | 用量日志和落账**放在同一个事务里**（现状是落账后再尽力写入） |

**列迁移步骤**（每一步都可回滚）：

1. 新建 billing 表，**双写**：旧事务里同时写新表。
2. 回填历史数据并校验，生成差异报告。
3. 切换读路径（用功能开关），影子比对一个周期。
4. 停止写旧列。
5. 下一个版本删除旧列（contract）。

#### 6.3.2 用量结算流程

```
gateway                                   redis-state                billing (consumer)
  │ 1. CheckBillingEligibility ───────────► billing:* 快照读取
  │ 2. 预留 billing:inflight:<req> (TTL) ──►
  │ 3. Forward 上游 ... 完成
  │ 4. 按定价快照版本计算费用（共享 pricing 库，行为与现状一致）
  │ 5. 更新 Redis 实时计数（余额缓存、Key 窗口、订阅用量）用于快速限流
  │ 6. XADD usage.recorded {request_id, user, key, account, group, tokens, cost, pricing_ver}
  │      ├─ 失败 → 同步降级 gRPC billing.RecordUsage
  │      └─ 再失败 → 本地有界 spool（磁盘）+ 告警；预留不释放，等待对账
  │                                                                │ 7. XREADGROUP 批量拉取
  │                                                                │ 8. 每条事件一个事务：
  │                                                                │    INSERT usage_billing_dedup（冲突即跳过）
  │                                                                │    钱包扣减 + 流水 / 订阅用量 / Key 计数 / 账号配额
  │                                                                │    INSERT usage_log
  │                                                                │ 9. COMMIT → XACK → 释放预留
  │                                                                │10. 刷新 billing:* 快照；必要时发
  │                                                                │    wallet_changed / quota_exhausted / balance_low
  │                                                                │11. 解析失败或重试超过上限 → DLQ + 告警
```

- **超额消费控制**：在步骤 1、2 用预留和实时计数控制并发请求导致的超额，保持现有的 fail-closed 开关（计费熔断）。异步只影响"落账可见的延迟"，不影响"能不能发起请求"。
- **定价一致性**：费用仍由网关按快照版本计算并写进事件（与现状一致，避免行为变化）。billing 校验 `pricing_ver`，对账任务抽样复算。
- **对账**（借鉴 agent-hub `tasks.py`）：
  - 每小时：超时未结算的预留与 usage_log 对比，补偿或释放。
  - 每日：`wallet_ledger` 汇总与 `wallets.balance` 对比；stream 的 MAXLEN 截断与消费进度对比。
  - 差异进入 Ops 告警。
- **批量生图冻结、扣款、释放**（`ReserveBatchImageBalance` 等）改为 billing 的幂等 gRPC 命令，幂等键为 `batch:<job_id>:<phase>`。

#### 6.3.3 交易发货（Saga-lite）

```
commerce: 回调验签 → 订单置为 PAID 并写 outbox(fulfill)（同一事务）
       → Fulfill Worker 认领 → 调用命令（幂等键 order:<order_no>:<step>）
            ├─ billing.CreditWallet / GrantSubscription
            ├─ identity.UpdateUserLimits（例如兑换并发）
            └─ commerce 内部：返佣入账
       → 全部成功 → FULFILLED → 事件 order_fulfilled（通知、统计）
       → 失败 → 指数退避重试 → 超过上限标记 FULFILL_FAILED → 后台"重试发货"（已有功能）
```

退款反向执行：先调用支付服务商退款，再调用 `billing.DebitWallet` 或撤销订阅，同样幂等。

### 6.4 认证与鉴权

| 场景 | 现状 | 目标 |
|---|---|---|
| 用户和管理员 JWT | HS256，密钥在 DB（`backend/internal/repository/security_secret_bootstrap.go`） | identity 签发。**P1–P2**：共享密钥由 K8s Secret 注入（从 DB 导出），console 本地校验。**P3**：改为 EdDSA，提供 JWKS（`/internal/.well-known/jwks.json`），密钥可以轮换；`aud` 区分 `user`、`admin` |
| 用户状态即时生效 | 中间件查询 | console 从 identity 的用户状态快照读取 `status` 和 `token_version`；吊销全部会话时 token_version 加一，立即生效 |
| 管理员 API Key | 中间件查询 | console 调用 `identity.VerifyAdminKey`（结果短 TTL 缓存） |
| 二次验证（TOTP） | 中间件 | console 调用 `identity.VerifyStepUp` |
| 网关 API Key | L1、L2 和 Outbox | 不变，快照拥有者改为 identity |
| 服务间调用 | 无 | P2：服务令牌（静态，按服务签发，放在 Secret 中）加 NetworkPolicy。P4：可以选择服务网格 mTLS（Linkerd 或 Istio）。被调方按 `x-s2a-caller` 和方法做白名单 |
| 支付回调 | 由 payment handler 验签 | Ingress 直接路由到 commerce，**不经过 console**；验签逻辑不变 |

### 6.5 配置与密钥

- **静态配置**：继续使用 viper 和 `config.yaml`，新增按服务划分的配置段，例如 `services.gateway.*`、`services.billing.consumer.*`、`transport.<svc>.addr`。
- **严格校验**（借鉴 agent-hub）：`s2a config check` 检查未知配置项、类型和依赖关系。`cluster_mode: true` 时强制以下要求：
  - 必须配置 `TOTP_ENCRYPTION_KEY` 等固定密钥。
  - 禁用在线升级、重启和安装向导。
  - 共享文件必须走 S3。
  - OAuth 会话必须放在 Redis。
- **运行时配置**：继续使用 DB 的 `settings` 表，归 platform 所有，按命名空间发布快照，变更时广播 `platform.settings_changed`。各副本收到后刷新缓存，并执行原来的 `SetOnUpdateCallback`（HTML 缓存、frame-src、websearch 管理器）。这同时修复 B3 和 B4。
- **密钥**：只从环境变量或 K8s Secret 注入（可以对接外部 KMS 或 Vault），镜像和 ConfigMap 里不出现明文。需要固定的密钥有：JWT 密钥、TOTP 加密密钥、支付配置加密密钥、账号凭据信封加密密钥、服务令牌、DB 和 Redis 密码。
- **安装向导**：集群模式下不使用。改为 `s2a migrate up` 加 `s2a admin init`（Job），或者沿用 `AUTO_SETUP` 加环境变量的方式。

### 6.6 后台任务治理

1. **生命周期改造**：`Provide*` 只负责构造，不再调用 `Start()`。组件注册到 `lifecycle.Registry` 时声明所属模块和角色，例如 `{module: "billing", kind: worker, singleton: true}`。由 `s2a serve` 根据 `--services` 和 `--workers-only/--no-workers` 决定启动哪些。
2. **单例任务**：统一使用 `tryAcquireSingletonLeaderLock`。没有协调的任务（见 2.3）一律接入，其中定时测试改为按计划行 `FOR UPDATE SKIP LOCKED` 认领，并设置 `next_run_at`。
3. **价格表**：由 resource 的领导者下载并校验，写入 Redis 或 DB 并广播，其他副本不再各自访问 GitHub。
4. **心跳**：所有任务上报 `ops_job_heartbeats`（已存在），由 Ops 告警"任务停摆"。
5. **定时类任务**：备份（pg_dump）和保留期清理可以改成 K8s CronJob，不常驻进程。
6. **任务归属**：

| 任务 | 归属 | 协调方式 |
|---|---|---|
| Token 刷新、账号到期、代理到期、配额和余额探测、版本同步、定时测试、调度快照构建、价格表 | resource | 领导者锁或认领，加 `oauth:refresh_lock` |
| 用量消费、平台配额刷写、仪表盘聚合、用量清理、对账、订阅到期提醒 | billing | 消费组或认领，加领导者锁 |
| 订单到期和对账、发货 Worker | commerce | 领导者锁或认领 |
| Ops 采集、聚合、告警、报表、清理，渠道监控，提示词审计，审计日志保留 | ops | 领导者锁或认领 |
| 邮件队列、公告投放、备份 | platform | 消费组或领导者锁 |
| 异步生图、批量生图、视频轮询、媒体清理 | media | Redis 队列，加租约和心跳（已有） |
| Outbox Relay | 每个拥有 Outbox 的服务 | 认领 |

### 6.7 长连接与优雅下线

网关有大量长时间运行的 SSE 流、OpenAI Responses WebSocket、Realtime 和音频 WebSocket。滚动发布时必须做到**不主动打断还在进行的请求**。

1. 收到 SIGTERM 后，先让 `/readyz` 返回 503，并用 preStop `sleep 10s` 等待端点摘除传播完成。
2. 停止接受新连接，`Server.Shutdown` 等待在途请求结束，等待上限 `server.drain_timeout`：网关默认 600 秒，控制面 30 秒。**把现在写死的 5 秒改成可配置**。
3. 超过上限仍未结束的 WebSocket，发送 close frame（1012 Service Restart），由客户端重连。Responses 续接靠 Redis 中的 `openai:response:*` 映射回到同一个账号（现有能力）。
4. 依次排空用量事件、spool、Worker 池，再停止 Worker，最后关闭 Redis 和 DB。
5. K8s 的 `terminationGracePeriodSeconds` 要比 `drain_timeout` 多 30 秒。网关的 PDB 设置 `maxUnavailable: 10%`，滚动策略设置 `maxSurge: 25%`。
6. Ingress 配置：`proxy-read-timeout` 和 `proxy-send-timeout` ≥ 3600 秒，`proxy-buffering off`，开启 WebSocket upgrade，打开 HTTP/2 和 h2c（现有的 h2c 回退逻辑保留）。
7. Responses WebSocket 的连接池（`backend/internal/service/openai_ws_pool.go`）在本副本内复用。可选：Ingress 按会话头做一致性哈希，提高复用率，这不是必需的。

### 6.8 出站与上游风控（分布式部署特有的风险）

上游（尤其是 Claude 和 ChatGPT 的 OAuth 账号）对**同一账号的出口 IP 变化和 TLS 指纹变化**很敏感。单体部署时出口固定，分布式部署后，同一个账号的请求可能从多个节点发出。

- **出口固定**（三选一或组合使用）：
  - (a) 集群统一 NAT 网关，所有节点共享固定的出口 IP 池。
  - (b) Egress Gateway（例如 Cilium 或 Istio egress），按命名空间或标签固定出口。
  - (c) `cluster_mode` 下要求 OAuth 类账号**必须绑定代理**（代理与回退能力已有，见 `proxy_fallback.go`）。
- **只有 gateway 和 media 对上游发起业务请求**。resource 的账号测试、ops 的渠道探测通过 gateway 的内部 RPC 执行。Token 刷新从 resource 发出，必须遵循同样的出口策略（经过代理或 NAT）。
- **TLS 指纹模板**是 resource 发布的快照，所有 gateway 副本使用一致的指纹（`backend/internal/pkg/tlsfingerprint/dialer.go`）。
- **SSRF 防护**（上游 URL 白名单）和响应体大小限制保持不变，并作为所有出站组件的公共库。

### 6.9 可观测性

| 维度 | 方案 |
|---|---|
| 链路追踪 | OpenTelemetry SDK：otelgin（HTTP 入口）、otelgrpc、otelhttp（包装上游 transport，**不记录请求体、提示词和凭据**）。W3C traceparent 跨服务传递，事件封装里也带 trace 上下文。采样：父级优先加 5%，错误和慢请求在 Collector 侧做尾部采样 |
| 指标 | 每个服务在独立端口 `:9100/metrics` 提供 Prometheus 指标。通用 RED 指标（gin、grpc）。网关专项：上游延迟和 TTFT（按平台、模型、账号类型）、故障转移次数、并发槽占用、排队时长、快照命中率和版本延迟、事件发布失败和 spool 积压。billing：消费 lag、落账延迟、DLQ 数量、对账差异。资源：DB 连接池、Redis 连接池 |
| 日志 | zap JSON 输出到 stdout，采集到 Loki 或 ELK。每条日志带 `request_id`、`trace_id`、`service`、`pod`。内置的 `ops_system_logs` 落库保留为产品功能，但要限制采样和级别 |
| 健康检查 | `/livez`：进程存活。`/readyz`：DB、Redis 可达；迁移版本满足要求；网关的关键快照已预热；不在排空中 |
| 产品内运维大盘 | 保留现有 Ops 功能，数据源改为事件和 Redis 聚合。实时 QPS 由各网关副本按秒 `INCRBY` 到 `ops:rt:*`，由 ops 汇总推送，不再依赖进程内的网关服务对象 |
| 平台告警 | 基础设施告警（Pod、DB、Redis、lag）用 Alertmanager。业务告警继续用内置的告警规则和邮件 |
| 诊断 | 内部端口提供 pprof，只在集群内可访问 |

### 6.10 数据库演进

**PostgreSQL 分四步**：

1. **逻辑归属**（P2）：输出"表 → 拥有者"清单（附录 A），为每个服务创建 PG 角色，只授予自己表的读写权限（迁移期间可以给少数共享视图只读权限）。在 staging 环境用最小权限运行，以此发现跨域访问。
2. **按服务分 schema**（P3，逐个服务进行）：`ALTER TABLE ... SET SCHEMA <svc>`，去掉跨域外键，改为应用层校验加事件。迁移文件按服务分目录（`backend/migrations/<svc>/`），各自记录 `schema_migrations`。
3. **独立实例**（P4，只针对热点）：
   - billing：`usage_logs` 按月声明式分区，热数据保留 N 个月，冷数据归档到 S3；分析需求可以引入 ClickHouse。
   - ops：`ops_*` 日志写入量大，独立成库或改为 ClickHouse。
   - 其余服务继续共用一个集群，以 schema 隔离。
4. **读写分离**：console 和 ops 的列表、统计查询走只读副本（可以容忍秒级延迟）。

**注意事项**：

- **PgBouncer 与 advisory lock**：代码里用的是会话级 `pg_try_advisory_lock`（`ops_advisory_lock.go`、`migrations_runner.go`、`scheduler_outbox_repo.go`），在 transaction pooling 模式下会失效。需要加锁的组件要直连或使用 session pooling，或者改用 `pg_try_advisory_xact_lock`。
- **连接预算**：总连接数约等于各角色"副本数 × 连接池大小"之和，必须小于 `max_connections` 的 80%。网关的 DB 连接池应当缩小（它主要读 Redis），控制面和 Worker 按需配置。
- **跨域列表查询**（例如使用记录要显示用户邮箱和 Key 名）：由 console 做 API 组合，用 `identity.BatchGetUsers` 和 `BatchGetKeys` 批量获取并加短缓存；或者在 usage_log 中冗余存一份展示字段的快照。

**Redis 分两步**：

1. **按用途拆实例**（P4）：
   - `redis-state`：开启 AOF everysec，`noeviction`，使用 Sentinel 或托管高可用。存放流、计费计数、并发槽、调度快照、领导者锁、刷新令牌、OAuth 会话、幂等标记。
   - `redis-cache`：允许 LRU 淘汰。存放鉴权 L2、代理延迟、指纹、仪表盘缓存、模型列表等。
   - 按仓储注入不同的客户端即可，改动集中在 `backend/internal/repository/wire.go`。
2. **Redis Cluster 暂缓**：现有约 60 段 Lua 脚本里有大量多键操作（`concurrency_cache.go`、`scheduler_cache.go` 等），上 Cluster 需要统一加 hash tag，例如 `concurrency:{acc:123}:*`。建议先纵向扩容加读写分离，等单个主节点成为瓶颈再改造。

### 6.11 代码结构与 DI 改造

**目标结构**（单仓库、单 `go.mod`）：

```
backend/
├── api/proto/                      # 契约（buf）
│   ├── identity/v1/  resource/v1/  billing/v1/  commerce/v1/
│   ├── platform/v1/  ops/v1/       gateway/v1/(内部探测)  events/v1/
├── cmd/
│   ├── s2a/                        # 统一入口：serve | migrate | config check | admin init
│   └── server/                     # 兼容旧入口 = s2a serve --services=all
├── internal/
│   ├── kit/                        # 公共底座：lifecycle, grpcx, eventbus, outbox, snapshot,
│   │                               #           leaderlock, health, otelx, errorsx, configx
│   ├── shared/                     # 纯库：domain 常量、pricing 计算、apicompat、协议、tlsfingerprint…
│   └── modules/
│       ├── gateway/                # handler / scheduling / forward / upstream / protocol / probe
│       ├── console/                # BFF：路由、中间件、聚合
│       ├── identity/   resource/   billing/   commerce/
│       ├── media/      ops/        platform/
│       │   └── 每个模块: api/(对外接口+DTO) app/(用例) repo/ grpc/(server+client) module.go
├── ent/                            # 按模块拆分 schema 目录（P3）
└── migrations/<svc>/               # 按服务拆分（P3）
```

**模块接口**：

```go
type Module interface {
    Name() string
    Provide(c *kit.Container) error          // 注册本模块的组件（wire 子 ProviderSet）
    RegisterHTTP(r *gin.Engine)               // 仅对外暴露 HTTP 的模块实现
    RegisterGRPC(s *grpc.Server)
    Workers() []lifecycle.Worker              // 由 --workers-only / --no-workers 控制
}
```

**改造要点**：

1. **按依赖方向从外往里搬**：先搬叶子领域（依赖少、被依赖少），依次是 platform（通知）→ commerce → ops → identity → billing → resource → gateway。每次搬迁用 `gopls rename` 或 `gomvpkg` 脚本化完成，搬完立刻合并，缩短与上游的冲突窗口。
2. **拆解依赖环**：
   - `OpsService` 依赖网关服务：改为读 Redis 和指标。
   - `AdminService` 依赖 `OpenAIGatewayService`：改为调用 resource 或 gateway 的接口。
   - 账号工具类服务依赖网关：改为 gateway 内部探测 RPC。
   - setter 注入改为接口加延迟绑定，或者用事件。
3. **去掉全局钩子**：`logger.SetSink`、`SetIngressRejectRecorder`、`SetWebSearchManager` 等改为显式注入；版本解析器改为从 platform 快照读取。
4. **约束边界**：用 `depguard` 或 `go-arch-lint` 规定 `modules/A` 只能 import `modules/B/api` 和 `kit/shared`，CI 强制执行。
5. **上帝对象**：`GatewayService`（28 个参数）、`AdminService`（23 个参数）按用例拆分，构造参数改为按端口（port）分组的结构体。

---

## 7. 部署方案

### 7.1 Kubernetes 拓扑（生产）

```
namespace: sub2api
├── Deployment gateway            replicas 3–50, HPA(CPU 60% + 自定义指标 inflight_streams), PDB 10%
│     ports: 8080(http) 9090(grpc 内部探测) 9100(metrics)
│     topologySpreadConstraints: zone / node;  terminationGracePeriodSeconds: 630
├── Deployment media-worker       replicas 2–10, KEDA(batch_image 队列长度)
├── Deployment console            replicas 2–3
├── Deployment identity           replicas 2–3
├── Deployment resource-api / resource-worker
├── Deployment billing-api / billing-worker   (worker 按 stream lag 扩缩)
├── Deployment commerce           replicas 2 (回调高可用)
├── Deployment ops-api / ops-worker
├── Deployment platform           replicas 2
├── Deployment web (nginx)        静态资源 /assets/*，也可以直接放 CDN
├── Job        migrate            Helm pre-install / pre-upgrade hook
├── CronJob    backup             pg_dump → S3（替代常驻进程里的定时备份）
├── Service    *-headless         gRPC 客户端负载均衡
├── Ingress    见 7.2
├── NetworkPolicy                 只允许必要的服务间端口
├── Secret / ConfigMap / ServiceMonitor / PrometheusRule
└── 外部依赖 (建议托管):  PostgreSQL HA(+PgBouncer, 只读副本)  redis-state(HA)  redis-cache
                         S3 兼容存储   OTel Collector   Loki   Grafana
```

- **Helm Chart**：`deploy/helm/sub2api`。`values` 中用 `deploymentProfile: S|M|L` 选择 5.3 节的部署分组，同一个 Chart 覆盖三种规模。
- **镜像**：继续使用同一个镜像（多角色）。`Dockerfile` 保留前端嵌入，供 console 渲染 index 使用，静态资源另外打包成 web 镜像或上传 CDN。`pg_dump` 只在 backup 或 platform 镜像中需要。

### 7.2 入口路由

| 路径 | 目标 | 说明 |
|---|---|---|
| `/v1/*`、`/v1beta/*`、`/backend-api/codex/*`、`/antigravity/*`、`/api/v3/contents/*`、`/v3/contents/*` | gateway | 长超时，关闭缓冲，支持 WebSocket |
| 根路径别名：`/responses*`、`/chat/completions`、`/messages*`、`/embeddings`、`/images/*`、`/videos*`、`/models`、`/contents/*`、`/tts`、`/stt`、`/custom-voices*`、`/realtime`，以及 `/api/event_logging/batch` | gateway | 以 `backend/internal/server/routes/gateway.go` 为准，**逐条列出**，避免和前端 SPA 路由冲突 |
| `/api/v1/payment/webhook/*` | commerce | 不经过 console，使用稳定的公网地址 |
| `/api/v1/**`（其余全部）、`/setup/*` | console | 面板限流和审计都在 console 内完成 |
| `/assets/*` 等静态资源 | web / CDN | 强缓存 |
| `/`（SPA 入口和 index.html） | console | 需要注入公共设置和 CSP nonce（`backend/internal/web/html_cache.go`） |
| `/livez`、`/readyz`、`/metrics` | 不对外暴露 | 只在集群内访问 |

边缘安全（WAF、无效凭据限流、IP 信誉）沿用 `deploy/EDGE_SECURITY.md` 的建议，在 Ingress 或 CDN 层实施。

### 7.3 docker-compose 集群方案（测试、小规模）

新增 `deploy/docker-compose.cluster.yml`：

- 去掉 `container_name`，用 `deploy.replicas` 设置副本数。
- 按 S 方案组合：`gateway`×2、`control`×1、`worker`×1，另加 `postgres`、`redis`、`minio`。
- 用 Caddy 或 Nginx 做负载均衡（沿用 `deploy/Caddyfile`，加 `flush_interval -1` 和长超时）。
- 迁移由 `migrate` 一次性容器执行，其他服务通过 `depends_on: condition: service_completed_successfully` 等待迁移完成。

### 7.4 CI/CD 与发布

- **CI**（在 `.github/workflows/backend-ci.yml` 基础上新增）：
  - `buf lint` 和 `buf breaking`。
  - `go-arch-lint`（模块边界检查）。
  - 契约测试（各服务的 gRPC 客户端对着 server 做 conformance 测试）。
  - 迁移 expand/contract 检查：禁止在同一个版本里既删列又改读路径。
  - 镜像构建与扫描（沿用 `release.yml` 和 `.goreleaser.yaml`）。
- **发布**：
  - 先运行 migrate Job，再滚动更新各 Deployment。
  - 网关使用金丝雀发布（Argo Rollouts，按 5% → 25% → 100% 放量），根据错误率和 TTFT 指标自动回滚。
  - 控制面普通滚动更新即可。
- **兼容窗口**：任意相邻两个版本的服务必须能共存，包括 gRPC 字段只增不删、事件 `schema_version` 向后兼容、快照多版本解码。
- **迁移启动方式**：保留"启动时执行迁移"作为一体化部署的默认行为；集群模式下设置 `--skip-migrate`，只检查迁移版本是否满足要求。

---

## 8. 实施路线图

> 工期是粗略估算，前提是 3–4 名后端工程师加 1 名 SRE，并且各阶段内部可以并行推进。每个阶段结束都是**可发布、可回滚**的状态。

```
P0 多副本就绪 ──► P1 角色化部署 ──► P2 异步化与契约化 ──► P3 领域服务独立 ──► P4 生产级高可用
 (3–4 周)          (2–3 周)           (6–8 周)             (12–20 周, 可并行)     (4–6 周)
 单体 ×N            gateway/control/worker   事件总线 + 模块边界      9 个服务逐个独立            多可用区, 热点库拆分
```

### P0 多副本就绪（单体 × N）

**目标**：在不拆代码的前提下，单体能安全地以 N 个副本部署在负载均衡后面。这就是**分布式部署的第一种形态**。

| # | 工作项 | 涉及位置 |
|---|---|---|
| 0.1 | Claude、OpenAI、Gemini、Antigravity 的 OAuth 授权会话迁到 Redis（复用 `redissession`） | `backend/internal/pkg/{oauth,openai,geminicli,antigravity}/oauth.go`、`backend/internal/pkg/redissession/store.go` |
| 0.2 | 定时测试改为认领（`SKIP LOCKED` 加 `next_run_at`）；渠道探测、余额检查、版本同步、价格表下载接入领导者锁 | `scheduled_test_runner_service.go`、`channel_monitor_runner.go`、`cn_provider_balance_check_service.go`、`*_version_sync_service.go`、`pricing_service.go` |
| 0.3 | 设置变更通过 Pub/Sub 广播；各副本刷新设置缓存，并执行 `SetOnUpdateCallback`（修复 HTML 缓存） | `setting_service.go`、`backend/internal/web/html_cache.go`、`backend/internal/server/router.go` |
| 0.4 | 引入 `cluster_mode`：强制固定密钥（`TOTP_ENCRYPTION_KEY` 等），禁用在线升级、重启和安装向导 | `backend/internal/config/config.go`、`update_service.go`、`backend/internal/setup/` |
| 0.5 | 共享文件迁到 S3：自定义页面、`data/public` 覆盖文件；日志只输出到 stdout | `backend/internal/handler/page_handler.go`、`backend/internal/web/embed_on.go` |
| 0.6 | 无效凭据限流器改用 Redis 实现（或明确交给边缘层） | `invalid_auth_abuse_limiter.go` |
| 0.7 | 邮件队列持久化（DB 表或 Redis Stream），崩溃不丢邮件 | `email_queue_service.go` |
| 0.8 | 优雅下线：`drain_timeout` 可配置，就绪状态切换，排空用量 Worker 池和批量写缓冲 | `backend/cmd/server/main.go`、`usage_record_worker_pool.go`、`backend/internal/repository/usage_log_repo.go` |
| 0.9 | `/livez`、`/readyz`，在独立端口提供 `/metrics`（Prometheus），日志带 trace_id 和 request_id | `backend/internal/server/routes/common.go` |
| 0.10 | `deploy/docker-compose.cluster.yml`；Helm Chart v0（单个 Deployment，多副本） | `deploy/` |

**验收标准**：

- 3 个副本运行在负载均衡后面，完整回归测试通过。
- 跨副本完成 OAuth 授权向导。
- 设置变更 2 秒内在所有副本生效。
- 单例任务不重复执行，定时测试和探测的请求数不随副本数增加。
- 滚动发布期间，持续 5 分钟的 SSE 流不中断，用量记录不丢失。
- 强杀一个 Pod 后，只影响它上面的在途请求。

### P1 角色化部署（同一代码库，同一数据库）

**目标**：数据面、控制面、Worker 分开部署和扩缩容（即 S 方案）。

| # | 工作项 |
|---|---|
| 1.1 | 生命周期改造：`Provide*` 不再调用 `Start()`，引入 `lifecycle.Registry`，组件声明自己的角色（见 6.6） |
| 1.2 | 新增 `s2a serve --roles=gateway\|control\|worker\|all`。第一步保持同一个 DI 图，只按角色控制路由注册和 Worker 启动（改动最小）；之后再按角色裁剪 Wire 图 |
| 1.3 | 按角色配置 DB 和 Redis 连接池、超时、`drain_timeout` |
| 1.4 | 入口路由表（7.2），回调直接打到 control |
| 1.5 | 网关 HPA，Worker 设置 `replicas: 2`（依靠领导者锁实现主备） |
| 1.6 | 基线压测：记录网关附加延迟、单 Pod 并发流上限，作为后续阶段的对比基准 |

**验收标准**：

- gateway Pod 上没有管理路由。
- Worker 全部宕机时，网关和控制面 API 不受影响，只有后台任务暂停并告警。
- 控制面大查询压测期间，网关 P99 附加延迟变化 ≤ 5%。
- `--roles=all` 行为与现状一致，现有单元、集成、E2E 测试全部通过。

### P2 异步化与契约化

**目标**：建立事件总线、Outbox 和快照协议，划清模块边界，为物理拆分扫清障碍。

| # | 工作项 |
|---|---|
| 2.1 | `internal/kit/eventbus`（Redis Streams 实现：消费组、XAUTOCLAIM、DLQ、lag 指标）和事件封装 proto |
| 2.2 | 通用 `outbox_events` 表加 Relay |
| 2.3 | **用量链路异步化**：网关发出 `usage.recorded` → billing 消费者（worker 角色）调用 `Apply`，并把 usage_log 写入同一事务；网关热路径去掉进程内 Worker 池（保留为降级缓冲）；新增对账任务 |
| 2.4 | 错误日志、入口拒绝、审计日志、提示词审计、邮件通知改为事件，由 ops 和 platform 消费 |
| 2.5 | 快照发布协议的 kit 实现（6.2），用它重写设置快照，并统一已有的鉴权、渠道等缓存 |
| 2.6 | 代码重构：搬出 `internal/kit` 和 `internal/shared`；领域按 6.11 的顺序逐个搬到 `internal/modules/*`；拆解依赖环，去掉全局钩子 |
| 2.7 | 为各领域定义 proto 和 Go 接口，先用本地适配器实现 |
| 2.8 | 引入 `go-arch-lint` 并在 CI 强制执行；PG 按服务建角色，staging 环境用最小权限运行，暴露跨域访问 |
| 2.9 | OTel 链路追踪全量接入 |

**验收标准**：

- 停掉 billing 消费者 10 分钟，网关不受影响；恢复后积压在 SLA 内消费完，对账差异为 0。
- 架构 lint 零违规。
- staging 在最小权限下运行，没有越权访问的报错。
- 一个请求能完整追踪 Ingress → gateway → 上游 → 事件 → billing。

### P3 领域服务独立

**目标**：9 个服务逐个独立部署，并拥有各自的数据（即 M 或 L 方案）。

**每个服务的拆分步骤（统一模板）**：

1. 模块边界已在 P2 完成，确认对外只暴露 `api/` 接口。
2. 实现 gRPC server 和 client 适配器，配置 `transport.<svc>: local|grpc`，**可以随时切回本地调用**。
3. 新建独立 Deployment，影子流量或按比例切换，观察错误率和延迟。
4. 把表迁到独立 schema，去掉跨域外键，迁移文件按服务分目录。
5. 发布网关需要的快照，补齐契约测试。
6. 稳定一个发布周期后，删除旧路径代码。

**拆分顺序与要点**：

| 顺序 | 服务 | 选它先拆的理由 | 要点 |
|---|---|---|---|
| 1 | **ops** | 主要消费事件，不在热路径上，写入量大，拆出后对主库减压明显 | 实时大盘改为读 Redis 聚合；渠道探测改为 gateway 内部 RPC；审计和提示词审计 Worker 一并迁出 |
| 2 | **commerce** | 边界清晰，涉及外部回调，安全上需要隔离（支付密钥） | Saga-lite 发货（6.3.3）；回调独立入口；返佣转余额走 billing 命令 |
| 3 | **platform** | 设置中心是其他服务的基础，越早独立，后续服务越早读快照 | 设置快照和广播、邮件队列、公告、页面（S3）、备份改为 CronJob |
| 4 | **identity** | 对应 agent-hub 的 usercenter，边界相对清晰 | JWT 改为 EdDSA 加 JWKS；鉴权快照拥有者切换；console 改为 gRPC 调用 |
| 5 | **billing** | 最复杂，要先完成 P2 的用量异步化 | 钱包和计数列迁移（6.3.1），流水，usage_log 分区，聚合 |
| 6 | **resource** | 与网关耦合最深（快照、健康信号、Token） | 调度快照构建器、健康信号消费、`EnsureFreshToken`、账号测试走 gateway 探测 |
| 7 | **media** | 复用网关内核，最后拆风险最小 | 独立的 `media-worker` 角色，计费冻结和扣款走 billing 命令 |

**验收标准（每个服务）**：

- 能独立发布、独立回滚（切换 transport 开关即可回退）。
- 服务自己的 SLO 达标，网关附加延迟不退化。
- 不存在跨 schema 写入。

### P4 生产级高可用

| # | 工作项 |
|---|---|
| 4.1 | 多可用区部署，`topologySpread`，PDB；PG 高可用（Patroni 或托管）加只读副本，开启 PITR |
| 4.2 | Redis 拆分为 state 和 cache，state 使用高可用部署；评估 Lua 脚本加 hash tag 的改造成本 |
| 4.3 | `usage_logs` 按月分区，冷数据归档；ops 日志独立存储（可选 ClickHouse） |
| 4.4 | 根据实际 QPS 评估是否把用量事件迁到 Kafka |
| 4.5 | 网关金丝雀发布（Argo Rollouts）、混沌演练（杀 Pod、断 Redis、PG 主从切换、上游 5xx 风暴） |
| 4.6 | 容量模型和压测报告、SLO 和错误预算、Runbook |
| 4.7 | （可选）多地域主备：数据面就近接入，控制面单主 |

**验收标准**：

- 一个可用区故障时，网关可用性不低于 99.95%。
- 演练中 PG 主从切换 RTO ≤ 30 秒（应用侧），Redis 主从切换 ≤ 30 秒，期间计费不丢失。
- 达到第 9 节的全部指标。

---

## 9. 非功能需求与验收指标

> 具体数值在 P1 基线压测后确认，下面是建议的目标值。

| 类别 | 指标 | 目标 |
|---|---|---|
| 性能 | 网关附加延迟（不含上游时间，用 Server-Timing 测量） | P99 ≤ 基线 + 5ms |
| 性能 | 快照命中率（L1 加 Redis） | ≥ 99.9% |
| 扩展 | 网关吞吐随副本数的扩展 | 1→10 副本时近似线性（≥ 85%） |
| 可用性 | 网关月可用性 / 控制面月可用性 | 99.95% / 99.9% |
| 发布 | 滚动发布打断长流的比例 | drain 窗口（默认 10 分钟）内结束的流打断率为 0 |
| 计费 | 用量事件丢失 / 重复扣费 | 0 / 0（每日对账差异为 0） |
| 计费 | 落账延迟 | P99 ≤ 5 秒；消费者故障恢复后 ≤ 15 分钟追平 |
| 一致性 | 设置、配置变更的传播 | ≤ 2 秒（广播）；最差 ≤ 60 秒（TTL 兜底） |
| 一致性 | 账号状态变更反映到调度快照 | ≤ 3 秒 |
| 容灾 | 计费数据 RPO / RTO | ≤ 1 分钟 / ≤ 30 分钟 |
| 安全 | 服务间调用都经过认证；镜像和 ConfigMap 中没有明文密钥；凭据快照加密 | 100% |
| 运维 | 所有服务都有 livez、readyz、metrics 和链路追踪 | 100% |

---

## 10. 风险与应对

| # | 风险 | 影响 | 应对 |
|---|---|---|---|
| R1 | **上游开源项目还在快速迭代**（Wei-Shaw/sub2api，版本发布频繁），大规模搬动包结构会导致合并冲突成本极高 | 高 | P0 和 P1 的改动尽量小、可以贡献回上游（多副本修复本身就有通用价值）；P2 的包搬迁按领域用脚本完成，每次搬迁前先合并一次上游，缩短冲突窗口；**需要明确长期 fork 策略**（见待决问题 Q3） |
| R2 | `internal/service` 有 24 万行、依赖环多，重构工作量可能被低估 | 高 | 先做 P0 和 P1，它们不依赖重构就能拿到分布式部署的大部分收益；P2 和 P3 按服务逐个推进，随时可以停在 M 方案 |
| R3 | 计费异步化或钱包迁移导致资金差错 | 高 | 双写、影子比对、功能开关；预留加实时计数控制超额消费；每日对账；保留 fail-closed |
| R4 | 网关延迟退化 | 中 | 遵守原则 3（热路径零同步 RPC）；把基准压测作为发布门禁 |
| R5 | Redis 成为单点和容量瓶颈（承担的职责越来越多） | 中 | state 和 cache 分开部署、高可用；流设置 MAXLEN；监控内存和 lag；必要时把流迁到 Kafka |
| R6 | 9 个服务带来的运维复杂度 | 中 | 部署分组（S、M、L）可以选择；统一 Helm 和 kit 底座；不需要的就不拆 |
| R7 | 分布式出口导致上游账号风控 | 中 | 6.8 的固定出口方案；`cluster_mode` 下强制 OAuth 账号绑定代理 |
| R8 | PgBouncer 与会话级 advisory lock 不兼容；连接数爆炸 | 中 | 6.10 的注意事项；制定连接预算 |
| R9 | 新旧版本共存时契约不兼容 | 中 | `buf breaking`；事件和快照的多版本解码；expand/contract 迁移检查 |
| R10 | 许可证合规（LGPL-3.0-or-later） | 低 | 分发修改后的版本时，需要按 LGPL 提供对应源码；建议法务评估 fork 的分发方式 |

---

## 11. 待决问题

| # | 问题 | 选项和影响 | 建议 |
|---|---|---|---|
| Q1 | identity 是否和 agent-hub 的 usercenter **合并成统一的用户中心**？ | 合并：一套账号体系，跨产品单点登录，但要统一手机号短信登录与邮箱和第三方登录、统一 JWT 签发方，迁移成本高。不合并：边界清晰，互不影响 | P3 拆 identity 时预留"外部 IdP"适配（OIDC），先不合并 |
| Q2 | 生产平台：自建 K8s、云托管 K8s，还是其他平台（例如 Nomad 或 compose 多机）？ | 影响 Helm、Ingress、Egress、托管 PG 和 Redis 的选型 | 云托管 K8s，PG 和 Redis 用托管服务 |
| Q3 | 是否继续跟进上游 sub2api？合并频率多高？ | 跟进：重构节奏要配合上游；不跟进：可以放手重构，但失去上游的新功能 | P1 之前保持每周跟进；P2 开始评估是否转为长期 fork |
| Q4 | 事件总线什么时候引入 Kafka？ | Redis Streams 的运维成本低，但保留期和回放能力有限 | 先用 Redis Streams，按 6.1 的阈值触发切换 |
| Q5 | 用量和运维分析是否引入 ClickHouse？ | 能缓解 PG 的压力，但多了一套组件 | P4 根据数据量决定 |
| Q6 | 是否继续保留简易模式和一体化发行？ | 保留会增加测试矩阵 | 保留（`--services=all`），这是开源分发的基础 |
| Q7 | 团队规模和时间窗口 | 决定 P3 是全拆（L）还是停在 M | 先确定 P0–P2 的排期，P3 结束前再评估 |
| Q8 | 是否需要多地域部署？ | 影响数据面就近接入和计费的一致性模型 | 本期不做，P4 评估 |

---

## 12. 附录

### 附录 A：表归属清单（目标形态）

> 实体名以 `backend/ent/schema/` 和 `backend/migrations/` 为准；标"新"的是新建表；标"拆"的表需要把一部分列迁到其他服务。

| 服务 | 拥有的表 / 数据 |
|---|---|
| **identity** | `users`（拆：余额迁往 billing）、`auth_identity`、`auth_identity_channel`、`pending_auth_session`、`identity_adoption_decision`、`user_attribute_definition`、`user_attribute_value`、`passkey_credentials`、`passkey_user_handles`、`user_avatars`、`api_keys`（拆：用量迁往 billing）、`deleted_api_key_audits`、`security_secret`（JWT 密钥）；Redis：`refresh_token:`、`token_family:`、`user_refresh_tokens:`、`totp:*`、`verify_code:`、`password_reset*:`、`passkey:session:` |
| **resource** | `accounts`（拆：配额用量迁往 billing）、`account_groups`、`proxies`、`tls_fingerprint_profile`、`error_passthrough_rule`、`groups`、`user_allowed_groups`、`user_group_rate_multipliers`、`composite_model_route`、`channels`、`channel_groups`、`channel_model_pricing`、`channel_pricing_intervals`、`channel_account_stats_*`、`scheduled_test_plans`、`scheduled_test_results`、`scheduler_outbox`、`sub2api_plugin_installations`、`sub2api_plugin_bindings`；Redis：`sched:*`、`oauth:token:`、`oauth:refresh_lock:`、`oauth:session:*`、`proxy:latency:` |
| **billing** | `wallets`（新）、`wallet_ledger`（新）、`user_subscriptions`、`api_key_usage_counters`（新）、`account_quota_counters`（新）、`user_platform_quota`、`usage_logs`、`usage_billing_dedup`（及 `_archive`）、`billing_usage_entries`、`usage_cleanup_task`、`usage_dashboard_*`、`usage_group_daily_rollups`、`usage_group_rollup_state`、`user_provider_default_grants`（待确认）；Redis：`billing:*`、`apikey:rate:`、`billing:upq:dirty`、`billing:inflight*` |
| **commerce** | `payment_order`、`payment_audit_log`、`payment_provider_instance`、`subscription_plan`、`redeem_code`、`promo_code`、`promo_code_usage`、`user_affiliates`、`user_affiliate_ledger`；Redis：`redeem:*` |
| **media** | `batch_image_job`、`batch_image_item`、`batch_image_event`；Redis：`image_task:`、`batch_image:*`、`grok_video_*` |
| **ops** | `ops_*`（error_logs、retry_attempts、system_metrics、metrics_hourly/daily、alert_rules/events/silences、job_heartbeats、system_logs、ingress_reject_aggregates）、`channel_monitor`、`channel_monitor_history`、`channel_monitor_daily_rollup`、`channel_monitor_request_template`、`channel_monitor_v2_*`、`audit_logs`、`content_moderation_logs`、`prompt_audit_jobs`、`prompt_audit_events`；Redis：`content_moderation:flagged_hashes`、`cyber_session_*`、`ops:*` |
| **platform** | `setting`、`announcement`、`announcement_read`、自定义页面元数据（文件存 S3）、邮件队列（新）；Redis：`update:latest`、`notify_*` |
| **gateway** | 不拥有表；Redis 运行时状态：`concurrency:*`、`wait:account:`、`sticky_session:`、`session_limit:`、`window_cost:`、`rpm:*`、`umq:*`、`temp_unsched:`、各类健康计数、`openai:response:`、`openai_responses_session_window:`、`live:call:`、`fingerprint:`、`websearch:quota:`、`plugin:kv:v1:` |
| **每个服务各自一份** | `idempotency_record`、`outbox_events`（新）、`schema_migrations` |

### 附录 B：事件清单（初版）

| 事件 | 生产者 | 消费者 | 分区键 | 语义 |
|---|---|---|---|---|
| `usage.recorded.v1` | gateway、media | billing（落账）、ops（被动监控） | user_id | 至少一次投递；按 request_id 幂等 |
| `account.health_signal.v1` | gateway | resource | account_id | 429（含 reset_at）、401、403、529、超时、恢复 |
| `resource.account_changed.v1` | resource | gateway（经调度快照）、ops | account_id | 现有 `scheduler_outbox` 的演进 |
| `resource.group_changed.v1` | resource | gateway、billing、identity | group_id | 分组规则和可见性变化 |
| `identity.user_registered.v1` | identity | billing（建钱包、发默认余额、开默认订阅）、commerce（优惠码、邀请关系）、platform（欢迎邮件） | user_id | — |
| `identity.user_changed.v1` / `api_key_changed.v1` | identity | gateway（鉴权快照失效）、billing | user_id / key_id | 现有鉴权 Outbox 的演进 |
| `billing.wallet_changed.v1` | billing | gateway（余额快照）、platform（余额不足提醒） | user_id | — |
| `billing.quota_exhausted.v1` | billing | gateway、resource（账号配额）、platform | 实体 id | Key、订阅、平台配额或账号配额耗尽 |
| `commerce.order_fulfilled.v1` | commerce | platform（通知）、ops（统计） | order_no | — |
| `ops.request_error.v1` / `ops.ingress_rejected.v1` | gateway | ops | — | 替代现在进程内的 Ops 写库 |
| `audit.admin_action.v1` | console | ops | admin_id | 管理员操作审计 |
| `risk.prompt_audit_requested.v1` | gateway | ops | user_id | 异步提示词审计 |
| `notify.email_requested.v1` | 任意服务 | platform | 收件人 | 持久化邮件队列 |
| `platform.settings_changed`（Pub/Sub 广播） | platform | 所有服务 | — | 配置快照失效 |

### 附录 C：gRPC 接口草案（节选）

```protobuf
// api/proto/billing/v1/billing.proto
service BillingService {
  rpc RecordUsage(RecordUsageRequest) returns (RecordUsageResponse);          // 事件降级通道, 幂等: request_id
  rpc CreditWallet(CreditWalletRequest) returns (WalletMutationResponse);     // 充值/返佣/兑换, 幂等: idempotency_key
  rpc DebitWallet(DebitWalletRequest) returns (WalletMutationResponse);       // 退款/调账
  rpc HoldBalance(HoldRequest) returns (HoldResponse);                        // 批量生图冻结
  rpc CaptureHold(CaptureRequest) returns (HoldResponse);
  rpc ReleaseHold(ReleaseRequest) returns (HoldResponse);
  rpc GrantSubscription(GrantSubscriptionRequest) returns (Subscription);    // 开通/续期, 幂等
  rpc RevokeSubscription(RevokeSubscriptionRequest) returns (Subscription);
  rpc GetBillingSnapshot(GetBillingSnapshotRequest) returns (BillingSnapshot); // 网关快照未命中
  rpc QueryUsageLogs(QueryUsageLogsRequest) returns (QueryUsageLogsResponse);  // console 列表
}

// api/proto/identity/v1/identity.proto
service IdentityService {
  rpc GetAuthSnapshot(GetAuthSnapshotRequest) returns (AuthSnapshot);         // key_hash → 鉴权快照
  rpc BatchGetUsers(BatchGetUsersRequest) returns (BatchGetUsersResponse);
  rpc VerifyAdminKey(VerifyAdminKeyRequest) returns (AdminPrincipal);
  rpc VerifyStepUp(VerifyStepUpRequest) returns (VerifyStepUpResponse);
  rpc UpdateUserLimits(UpdateUserLimitsRequest) returns (User);               // 兑换并发等, 幂等
}

// api/proto/resource/v1/resource.proto
service ResourceService {
  rpc GetGroupSnapshot(GetGroupSnapshotRequest) returns (GroupSnapshot);
  rpc EnsureFreshToken(EnsureFreshTokenRequest) returns (AccountToken);       // 冷路径, singleflight
  rpc RebuildSchedulerSnapshot(RebuildRequest) returns (RebuildResponse);
}

// api/proto/gateway/v1/probe.proto  (仅集群内)
service UpstreamProbeService {
  rpc TestAccount(TestAccountRequest) returns (stream TestAccountEvent);     // 账号测试/定时测试
  rpc ProbeChannel(ProbeChannelRequest) returns (ProbeChannelResult);         // 渠道监控
}

// api/proto/events/v1/envelope.proto
message EventEnvelope {
  string event_id = 1;            // ULID
  string type = 2;                // e.g. "usage.recorded"
  uint32 schema_version = 3;
  google.protobuf.Timestamp occurred_at = 4;
  string producer = 5;            // service/pod
  string partition_key = 6;
  map<string, string> trace_context = 7;
  google.protobuf.Any payload = 8;
}
```

### 附录 D：代码位置索引（拆分时的关键入口）

| 主题 | 位置 |
|---|---|
| 启动、DI、清理 | `backend/cmd/server/main.go`、`backend/cmd/server/wire.go`、`backend/internal/service/wire.go`、`backend/internal/repository/wire.go`、`backend/internal/handler/wire.go` |
| 路由 | `backend/internal/server/router.go`、`backend/internal/server/routes/` |
| 网关鉴权中间件 | `backend/internal/server/middleware/api_key_auth.go` |
| 调度与故障转移 | `backend/internal/service/gateway_scheduling.go`、`backend/internal/service/openai_account_scheduler.go`、`backend/internal/handler/failover_loop.go` |
| 转发 | `backend/internal/service/gateway_forward.go`、`backend/internal/service/openai_gateway_service.go`、`backend/internal/service/gateway_service.go` |
| 账号健康 | `backend/internal/service/ratelimit_service.go` |
| 计费落账（大事务） | `backend/internal/service/gateway_usage_billing.go`、`backend/internal/repository/usage_billing_repo.go` |
| 预扣与计费缓存 | `backend/internal/service/billing_inflight_reservation.go`、`backend/internal/service/billing_cache_service.go` |
| 调度快照与 Outbox | `backend/internal/service/scheduler_snapshot_service.go`、`backend/internal/repository/scheduler_outbox_repo.go` |
| 鉴权缓存与 Outbox | `backend/internal/service/api_key_auth_cache_impl.go`、`backend/internal/service/auth_cache_invalidation_outbox.go` |
| 领导者锁 | `backend/internal/service/leader_lock.go`、`backend/internal/repository/leader_lock_cache.go` |
| 迁移 | `backend/internal/repository/migrations_runner.go`、`backend/internal/repository/ent.go`、`backend/migrations/` |
| 配置与安装向导 | `backend/internal/config/config.go`、`backend/internal/setup/setup.go`、`deploy/config.example.yaml` |
| 前端嵌入与 HTML 缓存 | `backend/internal/web/embed_on.go`、`backend/internal/web/html_cache.go` |
| 支付回调 | `backend/internal/server/routes/payment.go`、`backend/internal/handler/payment_webhook_handler.go`、`backend/internal/payment/provider/` |
| 插件 | `backend/internal/service/plugin_manager.go`、`backend/pkg/pluginapi/v1/plugin.proto` |
| 出站网络 | `backend/internal/pkg/tlsfingerprint/dialer.go`、`backend/internal/pkg/httpclient/pool.go`、`backend/internal/repository/req_client_pool.go` |
| 部署 | `Dockerfile`、`deploy/docker-compose.yml`、`deploy/Caddyfile`、`deploy/EDGE_SECURITY.md`、`.github/workflows/backend-ci.yml`、`.github/workflows/release.yml`、`.goreleaser.yaml` |
| agent-hub 参考 | `agent-hub/usercenter/app/metering.py`（Streams 消费组）、`quota.py`（预留和结算）、`tasks.py`（对账）、`settings.py`（分层配置）、`auth.py`（令牌）、`context.py`（错误包） |
