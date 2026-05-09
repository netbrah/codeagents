# 23 — Orchestrator 多租户与配额

> [← 上一篇：单 vs 多 Session 设计深度对比](./22-single-vs-multi-session-design.md) · [回到 README](./README.md)

> [§03 §2](./03-architectural-decisions.md#2-状态进程模型) "1 Daemon Instance = 1 Session" 模型下，daemon 进程不感知租户——multi-tenancy 概念全部在 orchestrator 层。本章设计 orchestrator 的 Tenant 抽象、AuthN/AuthZ、配额引擎、审计日志、SaaS 路线图。Shell sandbox 见 [§11](./11-multi-tenancy-and-sandbox.md)。

## 一、TL;DR

| Layer | 职责 | 在哪 |
|---|---|---|
| **Orchestrator** | tenant 路由 / spawn daemon / quota / audit / OIDC | `qwen-coordinator` 进程 |
| **Daemon Instance** | 单一 session 状态 + tool 执行 + Shell sandbox | 每 session 一个 process |
| **Shell Sandbox** | 命令执行隔离 | daemon 进程内（[§11](./11-multi-tenancy-and-sandbox.md)）|

**核心设计**：tenant 信息**不进入 daemon 进程内部**——orchestrator 在 spawn daemon 时绑定 tenant id，daemon 只服务那个 tenant 的一个 session，进程级隔离自然消除 daemon 内 cross-tenant 问题。Tenant ACL / quota / audit 等都在 orchestrator 层做。

| 维度 | 设计 |
|---|---|
| Tenant 抽象 | `Tenant` 类在 orchestrator，daemon 不感知 |
| 认证 | Bearer token / OIDC / mTLS（在 orchestrator 入口）|
| 授权 | tenant → workspaces 映射 + scope 路由策略 |
| 配额 | per-tenant LLM token quota + 并发 daemon 数 |
| 审计 | 每 daemon spawn / kill / shell 调用都写 audit log（聚合到 orchestrator）|
| SaaS 路线图 | Stage 4（多租户）→ Stage 5（沙箱）→ Stage 6（SaaS）|

## 二、为什么 multi-tenancy 在 orchestrator 而非 daemon

[§03 §2 决策](./03-architectural-decisions.md#2-状态进程模型) "1 daemon = 1 session" 模型把 multi-tenancy 复杂度从 daemon 内挤出到 orchestrator 层：

| 维度 | 单 daemon 多 tenant 模型（旧）| Orchestrator + 单 session daemon（当前）|
|---|---|---|
| Tenant 隔离 | 应用层（AsyncLocalStorage `tenantId`）| **OS process 边界**（每 daemon 一个 tenant 的一个 session）|
| 跨 tenant 数据泄漏 | session/cache key 必须加 tenantId 前缀 | **不可能**（不同 daemon 进程互不可见）|
| Shell sandbox | 与 tenant 绑定，per-tenant 配置 | sandbox 在 daemon 内，tenant 信息已固化 |
| Quota | daemon 内 quota engine + Redis | orchestrator 层统计 + 决策 |
| Audit log | daemon 内写入 | daemon 写入本地 + orchestrator 聚合 |
| OIDC / SSO | daemon 入口 ACL middleware | orchestrator 入口（daemon 不感知）|
| Daemon 复杂度 | 高（cross-tenant 状态管理）| **低**（每 daemon 自给自足）|

**直接收益**：daemon 代码不需要处理 tenantId / ACL / quota——它只服务一个用户。orchestrator 是新加层，问题域清晰，与现有 core 业务逻辑解耦。

## 三、Tenant 抽象（在 orchestrator 中）

```ts
// packages/coordinator/src/tenant/Tenant.ts （新建）
export interface Tenant {
  id: string                          // 'alice@company.com' / OIDC sub
  displayName: string
  
  // 授权
  workspaces: WorkspaceAccess[]       // 哪些 workspaces 可访问
  scopes: SessionScope[]              // 'single' / 'user' / 'thread'
  
  // 配额
  quota: TenantQuota
  
  // Sandbox 配置（[§11](./11-multi-tenancy-and-sandbox.md) ShellSandbox 类型）
  sandbox: {
    type: 'none' | 'os-user' | 'namespace' | 'container' | 'remote'
    osUid?: number; osGid?: number     // os-user 时
    cgroupPath?: string                // namespace 时
    image?: string                     // container 时
    remote?: RemoteSandboxConfig       // remote 时
  }
  
  // Audit
  auditChannel: 'jsonl-local' | 'syslog' | 'opentelemetry' | 'kafka'
}

export interface WorkspaceAccess {
  workspaceId: string
  permissions: ('read' | 'write' | 'spawn-shell' | 'spawn-monitor')[]
}

export interface TenantQuota {
  maxConcurrentDaemons: number       // 同时跑几个 daemon
  llmTokensPerHour: number            // LLM 调用 token 上限
  llmTokensPerDay: number
  shellCallsPerHour: number
  storageBytes: number                // workspace + transcript 总大小
}
```

orchestrator 启动时从 settings.json / Postgres / Identity Provider 加载 Tenant 列表。

## 四、Authentication & Authorization

### 4.1 认证（AuthN）

```ts
// orchestrator HTTP middleware
app.use(async (req, res, next) => {
  const tenant = await authenticateRequest(req)
  if (!tenant) return res.status(401).json({ error: 'unauthenticated' })
  req.tenant = tenant
  next()
})

async function authenticateRequest(req): Promise<Tenant | null> {
  // 优先级：mTLS > Bearer token > OIDC cookie
  const auth = req.headers['authorization']
  
  if (req.connection.encrypted && req.client.authorized) {
    // mTLS 已验证（企业部署）
    return await tenantFromClientCert(req.client.getPeerCertificate())
  }
  
  if (auth?.startsWith('Bearer ')) {
    const token = auth.slice(7)
    return await tenantFromBearerToken(token)
  }
  
  if (req.cookies?.session_token) {
    return await tenantFromOidcSession(req.cookies.session_token)
  }
  
  return null
}
```

### 4.2 授权（AuthZ）

```ts
// orchestrator 在 spawn daemon 前检查
async function authorizeSpawn(tenant: Tenant, req: SpawnDaemonRequest): Promise<void> {
  // 1. workspace 访问权限
  const ws = tenant.workspaces.find(w => w.workspaceId === req.workspaceId)
  if (!ws) throw new ForbiddenError('workspace not authorized')
  
  // 2. scope 是否允许
  if (!tenant.scopes.includes(req.scope)) {
    throw new ForbiddenError(`scope ${req.scope} not allowed for tenant`)
  }
  
  // 3. 配额检查
  const usage = await quotaEngine.getUsage(tenant.id)
  if (usage.concurrentDaemons >= tenant.quota.maxConcurrentDaemons) {
    throw new QuotaExceededError('max concurrent daemons reached')
  }
  if (usage.llmTokensThisHour >= tenant.quota.llmTokensPerHour) {
    throw new QuotaExceededError('hourly LLM token quota exceeded')
  }
  
  // 4. 写 audit log
  await auditLog.record(tenant.id, 'spawn', req)
}
```

### 4.3 OIDC / SSO 集成

```yaml
# orchestrator settings.yaml
auth:
  modes: ['oidc', 'bearer', 'mtls']  # 多种并存
  
  oidc:
    issuer: https://auth.company.com
    clientId: qwen-coordinator
    clientSecret: ${OIDC_CLIENT_SECRET}
    tenantIdClaim: 'sub'            # OIDC claim → Tenant.id 映射
    workspaceClaim: 'qwen:workspaces' # custom claim
  
  bearer:
    tokenStore: postgres            # 多 token 存储
    
  mtls:
    caCert: /etc/qwen/ca.pem
    tenantIdFromCN: true           # cert CN → Tenant.id
```

## 五、Per-tenant Quota 引擎

### 5.1 Quota 维度

```ts
interface QuotaUsage {
  concurrentDaemons: number          // 当前活跃 daemon 数
  llmTokensThisHour: number
  llmTokensThisDay: number
  shellCallsThisHour: number
  storageBytesUsed: number
}
```

### 5.2 Quota 存储与原子性

**Redis** 存 sliding-window counters（per-tenant）：

```ts
class RedisQuotaEngine {
  async incrementLlmTokens(tenantId: string, tokens: number): Promise<QuotaCheck> {
    // Lua 脚本：原子读取 + 检查 + 递增 + 设置 TTL
    const result = await redis.eval(LUA_INCR_AND_CHECK, [
      `quota:${tenantId}:llm:hour:${currentHourBucket()}`,
      `quota:${tenantId}:llm:day:${currentDayBucket()}`,
    ], [
      tokens,
      tenant.quota.llmTokensPerHour,
      tenant.quota.llmTokensPerDay,
      3600,    // hour TTL
      86400,   // day TTL
    ])
    
    return parseQuotaResult(result)
  }
}
```

### 5.3 daemon 端配额上报

每 daemon 在 LLM 调用前向 orchestrator 报"我要用 N tokens"：

```ts
// daemon 内 LLM client wrapper
class QuotaAwareLlmClient {
  async generate(messages, options) {
    const estimatedTokens = estimateTokens(messages)
    
    // 预扣
    const reservation = await this.coordinator.reserveQuota({
      sessionId: this.sessionId,
      tokens: estimatedTokens,
    })
    
    if (!reservation.granted) {
      throw new QuotaExceededError(reservation.reason)
    }
    
    try {
      const response = await this.upstreamLlm.generate(messages, options)
      
      // 实扣（差额结算）
      await this.coordinator.confirmQuota({
        reservationId: reservation.id,
        actualTokens: response.usage.totalTokens,
      })
      
      return response
    } catch (err) {
      await this.coordinator.cancelReservation(reservation.id)
      throw err
    }
  }
}
```

### 5.4 优雅降级

quota 接近上限时 orchestrator 可以：

| 行为 | 触发条件 |
|---|---|
| **拒绝新 spawn** | concurrentDaemons ≥ quota | 返回 429 + retry-after |
| **拒绝新 LLM 调用** | tokens 超限 | session-internal error，让 LLM 决策（如 truncate context）|
| **降级到便宜模型** | tokens 接近上限 | orchestrator 在 spawn 时强制 model = 便宜版 |
| **冻结新 shell 调用** | shellCalls 超限 | 仅允许 read-only 命令 |

## 六、Audit Log

### 6.1 Audit log 内容

每个关键事件写入 audit log（orchestrator 维护，daemon 异步推送）：

```ts
interface AuditEvent {
  id: string
  timestamp: ISO8601
  tenantId: string
  sessionId?: string
  daemonId?: string
  
  type: 'auth.login' | 'auth.token-issued' | 'auth.token-revoked'
       | 'session.created' | 'session.terminated' | 'session.idle-evicted'
       | 'tool.shell-spawn' | 'tool.shell-exit' | 'tool.file-write' | 'tool.permission-granted'
       | 'quota.exceeded' | 'quota.warning'
  
  details: Record<string, unknown>   // 事件具体内容
  outcome: 'success' | 'denied' | 'error'
  
  // 不记敏感数据：tool args 中的 file content 不写 / LLM prompt 不写 / token 不写
}
```

### 6.2 Audit channels

```yaml
auditChannels:
  # JSONL 本地文件（默认 / self-host）
  - kind: jsonl-local
    path: /var/log/qwen/audit-{date}.jsonl
    rotate: daily
    retention: 90d
  
  # Syslog（企业 SIEM）
  - kind: syslog
    facility: AUTHPRIV
    server: syslog.company.com:514
    format: rfc5424
  
  # OpenTelemetry（云原生）
  - kind: opentelemetry
    endpoint: otel-collector:4317
    exporter: otlp-grpc
    attributes:
      service.name: qwen-coordinator
  
  # Kafka（高吞吐 SaaS）
  - kind: kafka
    brokers: [kafka1:9092, kafka2:9092]
    topic: qwen-audit-events
    schema: avro+confluent-registry
```

### 6.3 GDPR / 合规要求

| 要求 | 实现 |
|---|---|
| Right to Access（用户取自己的数据）| `GET /coordinator/users/:id/audit?since=...` |
| Right to Erasure（删除）| `DELETE /coordinator/users/:id` → 标记 audit 为 redacted（不真删，符合监管）|
| 数据最小化 | tool args / file content / LLM prompts **不进 audit**——仅记 metadata |
| 数据驻留 | per-tenant 配置 audit 写到哪个 region 的存储 |
| 加密 | audit channel 必须 TLS（syslog → TLS / kafka → TLS）|

## 七、Stage 4-6 SaaS 路线图

```
Stage 4 (~1-2 周)：Orchestrator 多租户基础
  └─ Tenant 抽象 + 配置加载
  └─ Bearer token 认证 + 单 token 多 user
  └─ 基础 quota（concurrent daemons + llm tokens）
  └─ Audit JSONL local
  └─ workspace allowlist
  └─ 适合：同公司团队 trusted multi-user

Stage 5 (~2-3 周)：+ Shell sandbox
  └─ §11 ShellSandbox 4 种本地实现
  └─ Monitor tool 走 sandbox 接口
  └─ 适合：半信任多用户（学校 / 大型团队 / consulting）

Stage 5.5 (~2 周)：+ 远程 sandbox
  └─ RemoteSandbox（SSH-based 起步）
  └─ Workspace 同步策略（NFS / rsync）
  └─ 适合：daemon vs shell 不同机器场景

Stage 6 (~1-2 月)：完整 SaaS
  └─ OIDC / SSO 集成
  └─ Quota engine（Redis + Lua 原子）+ 优雅降级
  └─ Audit channels（syslog / opentelemetry / kafka）
  └─ k8s native deployment（daemon per pod + sandbox worker pool）
  └─ Postgres state + Redis cache + S3 transcript backup
  └─ 多 region / 跨地理调度
  └─ Console / billing dashboard
  └─ 适合：完全不信任 SaaS 用户
```

## 八、与 OpenCode / Claude Code 多租户对比

| 维度 | OpenCode | Claude Code | Qwen daemon |
|---|---|---|---|
| 多租户支持 | ❌（单用户）| ❌（单用户）| ✅ Stage 4+ |
| 认证模式 | 单 password | API key | Bearer / OIDC / mTLS |
| Quota | ❌ | Anthropic API rate-limit（被动）| 主动 quota engine |
| Audit | ❌ | Anthropic console | 4 通道可选 |
| Sandbox | ❌ | Linux PID namespace | 4 本地 + 4 远程方案（§11）|
| SaaS 模式 | ❌ | Anthropic Managed Agents（闭源）| Stage 6 自托管 SaaS |

## 九、关键权衡

### 9.1 Orchestrator 单点 vs 高可用

orchestrator 是 SaaS 部署的关键路径——挂了所有用户无法 spawn 新 daemon。设计上：

- **现有 daemon 不受影响**——已 spawn 的 daemon instance 直接服务现有 client（沿用 [§17 §discovery 协议](./17-remote-cli-mode.md)）
- **新建 session 阻塞**——orchestrator 不可用时 client 短期重试 + UI 提示
- **HA 设计**：orchestrator 自身多副本 + Postgres 主备 + Redis Sentinel + 负载均衡（参考 [§16 5 层 HA 架构](./16-high-availability.md)）

### 9.2 Quota 准确性 vs 性能

| 模式 | 准确性 | 性能 |
|---|---|---|
| Synchronous Redis check + atomic incr | ✅ 强一致 | ~1ms / 调用 |
| Async batch upload（每 N 秒）| ⚠️ 滞后 N 秒 | ~0.01ms |
| Token bucket local + sync to Redis | ⚠️ 半异步 | ~0.1ms |

**推荐**：LLM token 配额走 sync atomic（必须强一致防止巨额超额）；shell calls / file writes 走 async batch（容忍秒级滞后）。

### 9.3 Daemon 端 vs Orchestrator 端 Quota 决策

| 决策位置 | 优势 | 劣势 |
|---|---|---|
| Daemon 端 | 延迟低 | 各 daemon 必须 sync state（推回 orchestrator）|
| Orchestrator 端 | 单点决策 | 每次 LLM 调用多一跳 RPC |

**推荐**：orchestrator 端集中决策 + reservation 模式（daemon 预扣 + 实际消耗后差额结算）—— 单一真相源，daemon 实现简单。

## 十、一句话总结

[§03 §2](./03-architectural-decisions.md#2-状态进程模型) "1 daemon = 1 session" 模型把 multi-tenancy 从 daemon 内挤出到 orchestrator 层——daemon 进程级隔离消除 daemon 内 cross-tenant 问题，daemon 代码 0 tenant 感知。  
**Orchestrator 承担 4 件事**：(1) 认证（Bearer / OIDC / mTLS）+ 授权（tenant → workspace 映射）；(2) Quota 引擎（reservation 模式 + Redis 原子）；(3) Audit log（4 通道）；(4) Daemon 生命周期 + sandbox 配置 spawn。  
**Stage 4-6 SaaS 路线图**：Stage 4 多租户基础（~1-2w）→ Stage 5 加沙箱（~2-3w）→ Stage 5.5 远程沙箱（~2w）→ Stage 6 完整 SaaS（~1-2m）。  
**与 OpenCode / Claude Code 差异**：本设计是唯一原生支持多租户 SaaS 的架构（OpenCode / Claude Code 均单用户）。

---

[← 上一篇：单 vs 多 Session 设计深度对比](./22-single-vs-multi-session-design.md) · [回到 README](./README.md)
