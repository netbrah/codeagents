# 05 — Security & Permission

> [← 上一篇：Deployment & Client](./04-deployment-and-client.md) · [下一篇：Roadmap & Ecosystem →](./06-roadmap.md)

## TL;DR

daemon（`qwen serve`）的安全模型分三层：

1. **传输层** —— Bearer token 认证（timing-safe 比较、统一 401）、Host 白名单防 DNS rebinding、CORS 默认拒绝浏览器、可选三档限流。loopback 开发默认免 token；**绑定 `0.0.0.0` 而不配 token 时 daemon 直接拒绝启动**。
2. **应用层** —— 复用整个 CLI 统一的 permission flow（`evaluatePermissionFlow()`）：工具默认权限 → PermissionManager 规则 → 调用方模式覆盖。daemon 是同一决策逻辑的又一个接入面，不引入第二套权限语义。
3. **多 client 协调层** —— `PermissionMediator` 把"谁可以批准这次工具调用"做成 4 种可配置策略（first-responder / designated / consensus / local-only），并维护 512 条 in-memory 审计环。

OAuth 登录由 daemon 代办 RFC 8628 device-flow：浏览器开在 client 侧，`device_code` 等机密用 `BrandedSecret` 包裹（任何序列化路径都输出 `[redacted]`），凭据以 `0o600` 权限落盘 daemon host。

**多租户立场：1 daemon = 1 tenant。** 租户隔离靠 OS 进程边界；同一 daemon 内的 N 个 session 共享信任域，**不是**安全边界。

---

## 一、三层权限模型总览

```
client 请求
   ↓
┌────────────────────────────────────────────────────────────┐
│ Layer 1 传输层                                               │
│   Bearer token（SHA-256 + timingSafeEqual，统一 401）         │
│   Host 白名单（防 DNS rebinding）· CORS 默认拒绝 Origin        │
│   mutation gate · clientId 归因 · 限流（可选）                 │
└────────────────────────────────────────────────────────────┘
   ↓
┌────────────────────────────────────────────────────────────┐
│ Layer 2 应用层 permission flow                               │
│   L3 工具默认权限 → L4 PermissionManager 规则                  │
│   → L5 调用方覆盖（YOLO / AUTO_EDIT / PLAN）                   │
│   → finalPermission: allow / deny / ask                      │
└────────────────────────────────────────────────────────────┘
   ↓ ask 时
┌────────────────────────────────────────────────────────────┐
│ Layer 3 多 client 协调（PermissionMediator）                  │
│   permission_request 经 SSE 广播 → client HTTP 投票           │
│   策略：first-responder / designated / consensus / local-only │
│   512 条 audit ring（审计面与 wire 面分离）                     │
└────────────────────────────────────────────────────────────┘
   ↓
 tool 执行 / 拒绝 / 取消
```

各机制的源码位置与出处（均已在 qwen-code main）：

| 机制 | 源码 | 来源 PR |
|---|---|---|
| Bearer token + Host 白名单 + CORS 默认拒绝 | `packages/cli/src/serve/auth.ts` | #3889 |
| clientId 归因 + 事件 stamping | `packages/cli/src/serve/acpHttp/connectionRegistry.ts` | #4231 |
| `--require-auth` mutation gate | `packages/cli/src/serve/auth.ts` | #4236 |
| `--allow-origin` CORS 白名单 | `packages/cli/src/serve/auth.ts` | #4527 |
| `--rate-limit` 三档令牌桶 | `packages/cli/src/serve/rateLimit.ts` | #4861 |
| 统一 permission flow | `packages/core/src/core/permissionFlow.ts` | #3723 |
| session-scoped 投票路由 | `packages/cli/src/serve/server.ts` | #4232 |
| PermissionMediator 4 策略 + audit ring | `packages/acp-bridge/src/permissionMediator.ts`、`packages/cli/src/serve/permissionAudit.ts` | #4335 |
| `permission_resolved` 带 `voterClientId` | `packages/acp-bridge/src/permissionMediator.ts` | #4539 |
| OAuth device-flow + BrandedSecret | `packages/cli/src/serve/auth/deviceFlow.ts` | #4255 |
| 文件边界（symlink / 原子写 / content-hash） | `packages/cli/src/serve/fs/` | #4250 / #4280 |

---

## 二、传输层

### Token 来源与默认策略

```bash
# 方式 A：环境变量（推荐 production —— /proc/<pid>/environ 仅 owner 可读）
QWEN_SERVER_TOKEN=$(openssl rand -hex 32) qwen serve

# 方式 B：CLI flag（手动 / 测试；优先级高于环境变量）
qwen serve --token=$(openssl rand -hex 32)
```

- 优先级：`--token` > `QWEN_SERVER_TOKEN`；两个来源都会 trim（防 `$(cat file)` 带尾换行导致永远 401）。
- `--token` 会在 stderr 提示：flag 值对本机任意用户经 `/proc/<pid>/cmdline` 可见，非临时用途应改用环境变量。
- loopback 绑定（`127.0.0.1` / `localhost` / `::1` / `[::1]`）默认不启用 bearer，stderr 会提示如何开启。

### 三条 boot 拒绝规则（源码: packages/cli/src/serve/runQwenServe.ts）

| 组合 | 行为 |
|---|---|
| 非 loopback 绑定（如 `--hostname 0.0.0.0`）+ 无 token | **拒绝启动** |
| `--require-auth` + 无 token | **拒绝启动** |
| `--allow-origin '*'` + 无 token | **拒绝启动** |

设计依据：对比 OpenCode（无 token 也启动、仅打 "unsecured" 警告），Qwen 选择 fail-loud——daemon 的服务对象包含 IM bot / web / 远端 IDE，一旦暴露非 loopback 接口，风险不止于本机开发者；与其允许"开发时没配 token 直接带上线"，不如让错误配置根本起不来。`--allow-origin '*'` 同理：无 token 时任意网页（恶意站点、广告 iframe）都能跨域驱动 daemon，必须先有 bearer 作为安全边界。

### 认证中间件关键特性（源码: packages/cli/src/serve/auth.ts）

| 特性 | 实现 |
|---|---|
| timing-safe 比较 | token 先 SHA-256 再 `crypto.timingSafeEqual`，防时序侧信道 |
| 统一 401 | 缺 header / scheme 错 / token 错三种情况返回完全一致的 `{"error":"Unauthorized"}`，不给探测者区分信息 |
| Host 白名单 | loopback 绑定时校验 Host 头（`localhost` / `127.0.0.1` / `[::1]` / `host.docker.internal` 加端口），防 DNS rebinding |
| `--require-auth` | bearer 对**所有**路由强制，包括 loopback 下的 `/health`；`/capabilities` 公示 `require_auth` feature |

### clientId：归因，不是认证

- ACP 连接注册时由 daemon 生成 clientId（`crypto.randomUUID()`），client 不能自挑。
- REST 调用方通过 `X-Qwen-Client-Id` 头自带 clientId；语法校验（≤128 字符、`[A-Za-z0-9._:-]+`），不合法返回 400 `invalid_client_id`；workspace 级 mutation 路由还会校验该 id 在 `knownClientIds()` 注册表内。
- daemon 把**校验后的** clientId stamping 到 fan-out 事件的 `originatorClientId` 字段——事件里的身份由 daemon 写入，不信任 client 在 payload 里自报。
- **威胁模型定位（源码注释原文意涵）**：clientId 是 best-effort attribution，不是 authentication——**bearer token 才是认证边界**。clientId 防的是多 SDK 善意共存时的串号，不防恶意伪造。

### mutation gate（`--require-auth` 与 strict 路由）

写路由共享一个 gate 中间件（`createMutationGate`），行为矩阵：

| daemon 配置 | 路由属性 | 结果 |
|---|---|---|
| `--require-auth` 或已配 token | 任意 | 透传（全局 bearer 中间件已拦截未认证请求） |
| 无 token（loopback 开发默认） | 普通 mutation | 开放（保持本地开发体验） |
| 无 token（loopback 开发默认） | **strict** 路由 | 401 + `code: "token_required"` |

strict 路由 = 即使本地开发也不该免认证的高危写操作：memory 写入、文件编辑、tool enable、MCP server 重启、device-flow 认证。401 body 用独立的 `token_required` code（区别于普通 `Unauthorized`），SDK 可以据此提示"重启 daemon 并配置 token"而非笼统报认证失败。

### CORS

- **默认**：任何带 `Origin` 头的请求一律 403（附 `Vary: Origin`）。CLI/SDK 不发 Origin，只有浏览器发——等于默认拒绝一切浏览器跨域。
- `--allow-origin <origin>`：白名单逐项严格校验，必须等于 `new URL(x).origin` 的规范形（无尾斜杠、无路径、无 userinfo、无 query），否则 boot 报错并指出坏在哪一项。**不做自动归一化**——静默改写模糊输入不如显式让运维改配置。
- 永不下发 `Access-Control-Allow-Credentials`；`*` 通配必须配 token（见 boot 拒绝规则）。

### 限流（`--rate-limit`）

默认关闭，开启后按令牌桶三档（窗口默认 60s，均可单独覆盖）：

| 档位 | 默认额度 | 覆盖 |
|---|---|---|
| prompt | 10/min | `--rate-limit-prompt` |
| mutation（其余非 GET/HEAD） | 30/min | `--rate-limit-mutation` |
| read（GET/HEAD） | 120/min | `--rate-limit-read` |

- 限流 key：非 loopback = `ip[:clientId]`；loopback = `cid:<clientId>`（无 clientId 则共享 `anonymous` 桶）。
- 豁免：`/health`、heartbeat、SSE 流、`/acp`。
- 命中日志采样输出（每 100 次一条），防止限流本身刷爆日志。

---

## 三、应用层 permission flow

### 一份决策逻辑，所有执行路径共用

`evaluatePermissionFlow()`（源码: packages/core/src/core/permissionFlow.ts）实现 L3→L4 决策：

- **L3**：工具自身的默认权限（`invocation.getDefaultPermission()`）
- **L4**：PermissionManager 规则覆盖（allowlist / deny / 强制 ask）
- **L5**：调用方模式覆盖（YOLO / AUTO_EDIT / PLAN），由各调用方处理（部分模式需要先拿到 confirmation 类型）

CLI 交互模式、non-interactive、ACP stdio、daemon HTTP 四条执行路径共用这同一份函数——daemon 经由 ACP session bridge 接入，而非另写一套。一致性意义：权限判定的 bug 修一处，全部入口同时受益；不存在"某个入口绕过了规则"的审计盲区。

### daemon HTTP 与 stdio ACP 的差异

| 维度 | ACP stdio | daemon HTTP |
|---|---|---|
| client 数 | 1 | N（同 session 多订阅者） |
| `ask` 的传递 | stdio 双向 RPC，client 阻塞应答 | SSE 推 `permission_request`，client 经 HTTP 投票应答 |
| 应答仲裁 | 无需仲裁 | PermissionMediator 按策略仲裁（见下节） |
| 超时 | client 端处理 | daemon 端计时，**默认 5 分钟**，超时按 `cancelled`（reason `timeout`）解决——工具不执行 |

### 决策持久化 scope 与并发写

| Scope | 存储 | 生命周期 |
|---|---|---|
| `session` | 内存（per-session） | session 结束即失效 |
| `workspace` | workspace 级配置文件 | 启动加载，daemon 内同 workspace 共享 |
| `global` | 用户级配置文件 | 启动加载，daemon 进程全局 |

> **并发写提示**：同 workspace 的 N 个 session 共享 `workspace` / `global` scope 的 decisions 文件，并发保存规则时需 per-file 串行化（in-memory 锁），否则后写覆盖先写（lost update）。`session` scope 各自私有，不冲突。跨 daemon 进程的协调不在 daemon 职责内，由编排层处理。

---

## 四、多 client 投票与 PermissionMediator

### 投票流程

```
1. 工具调用走到 finalPermission = 'ask'
2. mediator 登记 pending，audit 记 permission.requested，
   SSE 向该 session 所有订阅者广播 permission_request
3. 任一 client 投票：POST /session/:id/permission/:requestId
   body: { outcome: "selected", optionId } 或 { outcome: "cancelled" }
4. mediator 按策略仲裁 → 解决后 SSE 广播 permission_resolved
   （带 voterClientId = 投出决定票的 client）
5. 工具继续执行 / 取消；超时（默认 5 分钟）按 cancelled 解决
```

投票路由是 session-scoped 的（`POST /session/:id/permission/:requestId`），requestId 只在所属 session 内有效——A session 的请求不会被 B session 的订阅者看到或应答，关闭 cross-session 串号面。另保留全局 `POST /permission/:requestId` 作为兼容路由。

### 4 种仲裁策略

通过 settings `policy.permissionStrategy`（合法值集合从 capability registry 派生，经 `GET /capabilities` 的 `permission_mediation.modes` 公示）配置：

| 策略 | 语义 | 适用场景 |
|---|---|---|
| `first-responder`（默认） | 任一 client 先投先得 | 单人多端，自己的哪个端先看到先批 |
| `designated` | 仅指定 client 的投票有效，他人 403（`designated_mismatch`） | 人机混合：bot 只读旁观，human 独占审批权 |
| `consensus` | N-of-M 法定票：quorum 默认 `floor(M/2)+1`，`policy.consensusQuorum` 可覆盖（上限封顶至 M）；voter 集合在请求发出时快照 | 多人协作，敏感操作需多数同意 |
| `local-only` | 只接受 loopback 投票，远端投票 403（`remote_not_allowed`） | 远端只读旁观，审批权留在 daemon 本机 |

策略实现细节中的几个安全不变式：

- **loopback 判定看内核，不看 header**：`local-only` 依据 `req.socket.remoteAddress`（`127.0.0.0/8`、`::1`、IPv4-mapped 形式）判定，**绝不**读 `X-Forwarded-For` 等可伪造头；无法识别的地址形态按非 loopback 处理（fail-closed）。
- **consensus 空 voter 集合**：请求发出时若没有任何合格 voter，该请求只能等超时解决；daemon 会在 stderr 留 breadcrumb，运维不用从"5 分钟沉默"反推原因。
- **配置了本 build 未实现的策略** → 投票返回 501 `permission_policy_not_implemented`（而非 500），SDK 可提示"daemon 版本低于配置预期"。策略枚举是为扩展预留的封闭集合，新增策略走同一 501 升级路径。
- **取消是跨策略逃生门（有意为之）**：`{outcome:"cancelled"}` 在策略分发**之前**路由——`local-only` 下的远端 client、consensus 下不在 voter 集合内的 client，都可以取消（但不能批准）。这是 agent 侧 abort 路径；源码注释明确写明该"绕过"是契约，防止后人误当 bug 修掉。配套防御：取消经内部 sentinel 表达，若 agent 声明的选项集合撞上 sentinel 字面量则直接 500（防止真实选项伪装成取消）。

### bounded record：迟到 / 重复投票

mediator 维护一个**有界 FIFO**（512 条）的已解决记录：

- 上界防止长寿 daemon 的 resolved 记录无界增长（每条仅 requestId / sessionId / outcome，512 条 < 100KB）。
- 对已解决的 requestId 再投票：HTTP 返回 404（**有意不区分**"requestId 从未存在"与"已被别人解决"，不给枚举者反馈）；同时 SSE 重放 `permission_already_resolved`——第二个 voter 必须明确得知"已被解决、结果是什么"，而不是误以为自己投赢。UI 据此标记"已被 client X 应答"。
- `permission_resolved` 事件的 `voterClientId` 是"谁投出决定票"的规范字段（事件级 `originatorClientId` 在此事件上是同值兼容别名）；超时 / session 关闭 / 匿名 loopback voter 解决时两者都省略。
- 伪造选项防御：投票的 `optionId` 必须在 agent 当初提供的选项集合内——例如 prompt 按策略隐藏了"Always Allow"时，client 伪造 `ProceedAlways` 会得到 400 `invalid_option_id`（requestId 存在但选项不合法，语义与 404 区分）。

### Audit ring：审计面与 wire 面分离

`permissionAudit.ts` 维护 512 条 in-memory FIFO 审计环，记录五类事件：`permission.requested` / `voted` / `forbidden` / `resolved` / `timeout`（含 voter 快照、被拒原因、超时计时细节等 wire 上没有的取证粒度）。

两条设计边界：

- **audit 记录不上 SSE**——审计是 operator 的取证工具，与推给 client 的 wire 事件刻意分离；wire 上 `agent-cancelled` 与 `voter-cancelled` 折叠为同一形状，审计里保留区分。
- 当前不提供查询路由；环挂在 bridge 闭包内，为未来 `GET /workspace/permission/audit` 预留。
- 文件边界的访问审计（`fs.access` / `fs.denied` 事件）对路径做 SHA-256 截断哈希存 `pathHash`，不存原始路径——日志转发到第三方 SIEM 也不泄漏目录结构。

---

## 五、OAuth device-flow 与 secret 处理

daemon 常运行在容器 / SSH 服务器上，没有 GUI——所以 OAuth 由 daemon 代办 RFC 8628 device-flow，浏览器开在 client 侧：

```
1. client → daemon:  POST /workspace/auth/device-flow
   daemon → IdP:     请求 device_code
   daemon → client:  { user_code, verification_uri, intervalMs, expiresAt }
2. 用户在 client 侧浏览器打开 verification_uri，输入 user_code 登录
3. client 轮询 daemon（间隔默认 5s，daemon 侧夹紧上限）
   daemon → IdP:     grant_type=device_code 换 token
4. 成功后凭据持久化在 daemon host（0o600）
```

防回归与边界控制：

- **浏览器必须开在 client 侧**：测试断言 daemon 侧代码不含 `xdg-open` 等 spawn-browser 调用，防止回归成"在无 GUI 的 daemon 上弹浏览器"。
- 并发 device-flow 有上限，超出返回 409 `too_many_active_flows`；IdP 侧故障映射 502（与 daemon 自身 5xx 区分）。
- 查询 flow 状态时，非发起 client 看到的 verification 字段被脱敏——同样是 clientId 归因（防善意串号），bearer 才是认证边界。

### BrandedSecret：让机密无法被序列化

`device_code`、PKCE verifier 等机密在 daemon 内存中以 `BrandedSecret` 包裹（源码: packages/cli/src/serve/auth/deviceFlow.ts），目标是**任何**序列化 / 拼接路径都只能得到 `[redacted]`：

| 误用路径 | 实际输出 |
|---|---|
| `JSON.stringify({ s: secret })` | `{"s":"[redacted]"}` |
| `String(secret)` | `[redacted]` |
| `'x=' + secret` | `x=[redacted]` |
| `` `s=${secret}` `` | `s=[redacted]` |

实现上的关键决策：**frozen plain object + 模块级 WeakMap，而不是 `new String(value)` wrapper**。早期草案用 String wrapper 覆写 `toString`/`toJSON`，但 `+` 拼接与模板字符串走 `Symbol.toPrimitive → valueOf` 路径，会绕过覆写直接泄漏原值（实测踩过的坑）。现行实现把 `toString` / `toJSON` / `Symbol.toPrimitive` 全部钉死为 `[redacted]`，原值只存 WeakMap，唯一取出路径是 `unsafeRevealSecret(secret)`——`unsafe` 前缀让所有取值调用点天然可 grep、可 lint allowlist、难以误用。

### 凭据落盘：0o600 + 原子替换

凭据写盘走"先以 `0o600` 建临时文件、再原子 rename"两步（源码: packages/core/src/qwen/qwenOAuth2.ts）——不是"先写后 chmod"：后者在写入与收权之间存在文件以宽权限暴露的窗口；临时文件创建即 `0o600`，不受 umask 影响，rename 原子替换后任何时刻都不存在宽权限副本。

### 文件边界（daemon 文件路由通用）

| 机制 | 行为 |
|---|---|
| workspace 根限定 | 所有 daemon 文件路由的路径解析限定在 workspace 根内 |
| symlink 防逃逸 | 路径逐段做 symlink 规范化，逃逸 workspace → `symlink_escape` 错误（400）；symlink 环 / 超深由 SYMLOOP_MAX 上限拒绝 |
| 原子写 + mode 保留 | tmp+rename 原子替换，且保留目标文件原 mode——已是 `0o600` 的 secret 文件被编辑后仍是 `0o600`；新文件默认 `0o600` 而非跟随 umask |
| content-hash 前置 | 写请求带 `expectedHash`（`sha256:<64hex>`），与磁盘现状不符则拒绝——防多 client 并发编辑的 lost update 与 TOCTOU |

---

## 六、多租户边界与生产 best practice

### 1 daemon = 1 tenant

- 租户隔离的单元是 **OS 进程**：1 daemon 绑定 1 tenant 的 1 workspace（启动 cwd + 启动用户）。systemd `MemoryMax=` / cgroup / docker `--memory` 直接就是 per-tenant 配额，daemon 内部不需要租户抽象。
- **同 daemon 内的 N 个 session 共享信任域**：同一 UID、同一文件系统视图、同一进程环境。session 之间的事件按 sessionId 隔离 fan-out，但这是产品语义，**不是安全边界**。
- 推论：**绝不可让多个租户共用一个 daemon**。多租户 = 编排层起多个 daemon 进程，每个进程注入各自租户的凭据与配额。

部署形态详见 [04 — Deployment & Client](./04-deployment-and-client.md)；编排层方向见 [06 — Roadmap](./06-roadmap.md)。

### 网络 egress 策略

- daemon host/pod 出站按 **deny-by-default + 显式 allowlist**：只放行配置的 provider 端点、MCP HTTP/SSE 端点、skills 实际调用的外部 API。
- daemon 不需要入站公网；远端 client 经反向代理或私网访问。
- 出站被挡时 `GET /workspace/preflight` / `GET /workspace/env` 返回结构化诊断（`errorKind: "blocked_egress"`），client 能渲染可操作的修复提示。

### 凭据在 daemon host

| 类型 | 位置 |
|---|---|
| Provider OAuth token | daemon host（device-flow 落盘，`0o600`） |
| API key / env | daemon 进程环境（从 secret manager / k8s secret 注入） |
| MCP server 凭据 | MCP 配置引用或 daemon host env |
| SSH agent / kubeconfig | daemon host 本地 |

**关键**：client 侧的凭据不会自动传到 daemon——所有工具执行、provider 调用都在 daemon host 上求值（runtime locality，详见 [04 — Deployment & Client](./04-deployment-and-client.md)），凭据必须 daemon host 自己持有。1 daemon = 1 tenant 时凭据天然 per-进程隔离。

### 部署 checklist

- [ ] token 经 `QWEN_SERVER_TOKEN` 注入（不要用 `--token` 上生产）
- [ ] 非 loopback 暴露时确认 bearer 已启用；考虑加 `--require-auth` 与 `--rate-limit`
- [ ] 浏览器接入才配 `--allow-origin`，列具体 origin，避免 `*`
- [ ] daemon host 装齐 MCP / skills 所需 runtime（node / uv / python / docker / cloud CLI）
- [ ] 凭据、kubeconfig 等已在 daemon host provision
- [ ] 出站网络策略只放行 provider + MCP 端点
- [ ] 用 `GET /workspace/preflight` 验证环境，错误详情含 `errorKind` 可程序化处理

---

## 七、typed-error 设计哲学

daemon 内部与 wire 上的错误是两套刻意分离的表达：

- **daemon 内部**：富类型 `Error` 子类（保留 stack / cause / metadata），如文件边界的 `FsError`、bridge 的 `BridgeTimeoutError` / `BridgeChannelClosedError`、OAuth 的 `AuthError` 族。
- **HTTP wire**：降级为**封闭枚举** `errorKind`（`SERVE_ERROR_KINDS`，源码: packages/acp-bridge/src/status.ts——`missing_binary`、`blocked_egress`、`auth_env_error`、`init_timeout`、`protocol_error`、`missing_file`、`parse_error`、`budget_exhausted` 等），加上路由级结构化 `code` 字段（`token_required`、`invalid_client_id`、`permission_forbidden`、`invalid_option_id`…）。

为什么坚持封闭枚举而不是让 client 解析错误文案：

1. **文案会变**——措辞调整、本地化都不该破坏 client 的分支逻辑；regex-match error message 是脆弱契约。
2. **枚举封闭可穷举**——SDK 能 switch 全集并对未知值有兜底；新增 kind 是显式契约变更。
3. **信息控制**——wire 上只暴露分类，不泄漏内部路径 / 堆栈；细节进审计（路径还要先哈希）。
4. **HTTP 状态码承载语义**——403 策略拒绝（你被认证了但策略不让）≠ 404 不存在（或不告诉你）≠ 501 配置超前于实现 ≠ 502 上游故障。每个分类对应明确的 client 应对动作。

HTTP API 错误模型详见 [03 — HTTP API & Protocol](./03-http-api.md)。

---

> **免责声明**：本文基于 qwen-code main 分支源码分析（截至 2026-06-12），随上游演进可能过时；行为细节（默认值、错误码、策略语义）以源码为准。

下一篇：[06 — Roadmap & Ecosystem →](./06-roadmap.md)
