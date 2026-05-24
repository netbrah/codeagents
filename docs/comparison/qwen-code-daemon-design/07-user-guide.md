# 07 — User Guide：怎么用 `qwen serve` daemon

> [← 上一篇：Roadmap & Ecosystem](./06-roadmap.md) · [回 README](./README.md)
>
> 这是一篇 **recipe-oriented 使用文档**，面向"已经装了 v0.16.0，想把 daemon 跑起来"的工程师。设计动机看 [§01 Overview](./01-overview.md)、决策依据看 [§02](./02-architectural-decisions.md)、协议细节看 [§03](./03-http-api.md)、客户端架构看 [§04](./04-deployment-and-client.md)、安全模型看 [§05](./05-permission-auth.md)、roadmap 看 [§06](./06-roadmap.md)。

## TL;DR

`qwen serve` 是 **qwen-code v0.16.0** 起作为 production-capable feature 的 headless HTTP daemon —— 把 `qwen --acp` agent 封进一个 1-workspace-per-process 的进程，对外暴露 HTTP/SSE，让 SDK / web client / IDE 不再各自 spawn `qwen --acp` child。

**v0.16.0 main 上你拿到了什么**：daemon Wave 1-4 全部 + Wave 5 acp-bridge package 抽取（PR 22a/22b/1/22b/2）：
- 协议：~40 HTTP routes、~28 capability tag、SSE typed event v1、closed `errorKind` 7-值 taxonomy、Last-Event-ID 重连 + heartbeat + slow_client_warning
- session 生命周期：create / load / resume / cancel / close-delete + bounded scope
- 权限：first-responder vote / session-scoped permission / bearer token + clientId stamping + `--require-auth` mutation gate
- workspace state CRUD：memory / agents / tools / approval-mode / init / MCP restart + budget guardrail
- 文件：sandbox-bounded read / write / edit（atomic temp+rename）
- OAuth：device-flow / BrandedSecret 4-way redaction / 0o600

**v0.16.0 main 上没有**：
- F1-F4 prereq（acp-bridge package 完整 self-sufficiency / MCP shared transport pool / PermissionMediator 4 strategies / daemon protocol completion）—— 在 `daemon_mode_b_main` 集成分支等周期 merge
- chiga0 SDK 共享 daemon UI 层（#4328/#4353）、ACP Streamable HTTP transport（#4472 DRAFT）—— 同样在集成分支
- ytahdn web-shell 真实 web client（#4380）—— 同样在集成分支

**本地 native TUI 不变**：`qwen` 直连本地 runtime 链路长期保留。`qwen serve` 是给 **web chat / web terminal / remote runtime / CI** 场景的，不是 native TUI 的替代。详 [§02 §7](./02-architectural-decisions.md#7-部署模式--mode-b-mainline--mode-a-parking-lot)。

---

## 一、快速上手（5 分钟）

```bash
# 1) 装 v0.16.0
npm install -g @qwen-code/qwen-code@0.16.0

# 2) 启动 daemon（loopback 默认免 token，仅本机可用）
cd /path/to/your/repo
qwen serve

# stderr 输出：
#   qwen serve listening on http://127.0.0.1:4170 (workspace=/path/to/your/repo)

# 3) 验证健康
curl -s http://127.0.0.1:4170/health
# {"status":"ok"}

# 4) 看能力清单
curl -s http://127.0.0.1:4170/capabilities | jq .

# 5) 建一个 session + 发 prompt + 流式读
curl -s -X POST http://127.0.0.1:4170/session -H 'Content-Type: application/json' \
  -d '{"sessionScope":"session","workspaceCwd":"'"$PWD"'"}' \
  | jq -r .sessionId
# → sess_abc123

curl -N -H "Accept: text/event-stream" \
  "http://127.0.0.1:4170/session/sess_abc123/events" &
SSE_PID=$!

curl -s -X POST "http://127.0.0.1:4170/session/sess_abc123/prompt" \
  -H 'Content-Type: application/json' \
  -d '{"prompt":[{"type":"text","text":"hello"}]}'
# SSE 流里会看到 session_update 事件

kill $SSE_PID
```

跑通这 5 步就有 daemon 了。后面是把它做对、做安全、上生产的细节。

---

## 二、`qwen serve` 启动选项

所有选项默认值来自 v0.16.0 `packages/cli/src/commands/serve.ts`。

### 2.1 基本 flag

| flag | 默认 | 说明 |
|---|---|---|
| `--port <N>` | `4170` | TCP 端口；`0` = OS 分配 ephemeral |
| `--hostname <addr>` | `127.0.0.1` | 绑定 interface；loopback (`127.0.0.1` / `localhost` / `::1` / `[::1]` / `host.docker.internal`) 默认免 token，其他必须配 token |
| `--token <hex>` | — | bearer token；**生产强烈建议走 env 而非 flag**（`--token` 在 `/proc/<pid>/cmdline` 任何本机用户可见，env 则只在 `/proc/<pid>/environ` owner-only）|
| `--workspace <abs path>` | `process.cwd()` | 绑死的 1 workspace 绝对路径，`POST /session` 携带不一致 cwd 返 `400 workspace_mismatch` |
| `--require-auth` | `false` | 即使 loopback 也强制 token；防共享 dev host / CI runner / 多租户工作站任意本地用户打 `127.0.0.1`；启用后 `/health` 也要 bearer |
| `--max-sessions <N>` | `20` | 并发活 session 上限，超出 spawn 返 503；attach 现有 session 仍可；`0` = 禁用 |
| `--max-connections <N>` | `256` | listener 级 TCP socket 上限（`server.maxConnections`），慢/僵尸 SSE client 在 accept time 被拒；`0` = 禁用 |
| `--event-ring-size <N>` | `8000` | per-session SSE 重放 ring 深度，`Last-Event-ID` 重连可回拉的 backlog 大小 |
| `--http-bridge` | `true` | Stage 1 模式（daemon 内嵌 `qwen --acp` child）；`--no-http-bridge` 是预留 Stage 2 native in-process，v0.16.0 未实现，传了会 fallback 并 stderr warning |
| `--mcp-client-budget <N>` | — | MCP client 数 cap（per ACP child for bound workspace）；正整数 |
| `--mcp-budget-mode <off\|warn\|enforce>` | `warn` if budget set else `off` | budget 怎么 enforce：`off` 纯观测 / `warn` 75% 阈值发警告不拒 / `enforce` 超 cap 拒绝（按 `mcpServers` 声明顺序确定）；`enforce` 必须配 budget |

### 2.2 安全启动配方

**本机开发** —— 最省事：
```bash
cd /path/to/repo
qwen serve              # loopback 免 token，仅 127.0.0.1 可达
```

**本机但工作站多用户 / CI runner** —— 不让本机其他用户访问：
```bash
export QWEN_SERVER_TOKEN=$(openssl rand -hex 32)
qwen serve --require-auth          # 强制 token，loopback 也要 bearer
echo "$QWEN_SERVER_TOKEN" > ~/.qwen/serve/token
chmod 600 ~/.qwen/serve/token       # SDK 默认会从这读
```

**暴露远端**（远端 cloud devbox / k8s pod）：
```bash
export QWEN_SERVER_TOKEN=$(openssl rand -hex 32)
qwen serve --hostname 0.0.0.0 --port 4170
# ⚠️ 0.0.0.0 + 无 token = boot 拒绝启动（防开发时不开 token 直接走勿勿上线）
```

**production hardened**（mutation routes 全 gate）：
```bash
export QWEN_SERVER_TOKEN=$(openssl rand -hex 32)
qwen serve --require-auth \
  --hostname 0.0.0.0 \
  --max-sessions 50 \
  --max-connections 500 \
  --mcp-client-budget 8 --mcp-budget-mode enforce
```

### 2.3 环境变量

| env | 说明 |
|---|---|
| `QWEN_SERVER_TOKEN` | bearer token（优先于 `--token`，推荐生产用这个）|
| `QWEN_SERVE_MCP_POOL_TRANSPORTS` | （仅 `daemon_mode_b_main` 集成分支 F2 #4336）默认 `stdio,websocket`；显式开 `http,sse` 才让 HTTP/SSE MCP transport 走 shared pool。**v0.16.0 main 上无此功能**。|
| `QWEN_TELEMETRY_METRICS_INCLUDE_SESSION_ID` | （#4367 main 未合）开 `true` 让 metric 重新带 `session.id` —— 默认不带防 Prometheus/ARMS unbounded cardinality fan-out |

---

## 三、3 种 deployment shape

来自 chiga0 [#3803 deployment forms](https://github.com/QwenLM/qwen-code/issues/3803) 设计，详 [§01 §三·一](./01-overview.md)。

### Shape 1: Local single-machine（本地单机）✅ 推荐

```
┌─────────────────────────────────────────┐
│ your laptop                             │
│   ┌──────────────┐    ┌──────────────┐  │
│   │ qwen serve   │←───│ web-shell /  │  │
│   │ workspace=   │    │  curl / SDK  │  │
│   │ /repo        │    └──────────────┘  │
│   └──────┬───────┘                      │
│   /repo  ↓ fs/tool/MCP/skill            │
└─────────────────────────────────────────┘
```

- daemon host = workspace host = client host = 都本机
- 用例：开发期跑 web-shell；多个 tab 同时连同一 daemon
- 提醒：**native TUI 不走 daemon** —— 这个 shape 是给 web client / 多 tab 协作，不是替代 `qwen` CLI

### Shape 2: Cloud / devbox remote-runtime（远端 runtime + 本地 client）✅ 推荐

```
┌──────────────┐         HTTP/SSE         ┌──────────────────────────┐
│ your laptop  │   ───────────────────→   │ remote devbox / k8s pod  │
│              │                          │   ┌──────────────┐       │
│ browser /    │   ←───────────────────   │   │ qwen serve   │       │
│ SDK / IDE    │                          │   │ workspace=   │       │
└──────────────┘                          │   │ /workspace   │       │
                                          │   └──────┬───────┘       │
                                          │  /workspace ↓ fs/tool... │
                                          └──────────────────────────┘
```

- daemon host **==** workspace host（同一台远端机器/pod）
- client 在本地浏览器/IDE，通过 HTTP+SSE 与远端 daemon 对话
- 用例：Codespaces / Coder / 内部 cloud devbox
- 必须配 token + hostname 0.0.0.0 + TLS reverse proxy

### Shape 3: Local workspace + remote daemon ❌ 不推荐

daemon 在远端但 workspace 在本地 —— daemon 看不到 local files / tools / MCP / skills，除非显式 sync / mount / orchestrator。`POST /session` 携带本地 cwd 会被 `400 workspace_mismatch` 拒。

**daemon host = runtime host 是核心不变式**。详 [§04 §五 Runtime locality](./04-deployment-and-client.md)。

---

## 四、客户端接入

### 4.1 TypeScript SDK：`DaemonSessionClient`

```ts
import { DaemonSessionClient } from '@qwen-code/sdk-typescript/daemon';

const client = new DaemonSessionClient({
  baseUrl: 'http://127.0.0.1:4170',
  bearerToken: process.env.QWEN_SERVER_TOKEN,    // 远端 / require-auth 时必填
});

// 1) 建 session
const { sessionId } = await client.createSession({
  workspaceCwd: process.cwd(),
  sessionScope: 'session',
});

// 2) 订阅 SSE 事件（typed discriminated union, PR#4217）
const unsubscribe = client.subscribe(sessionId, (event) => {
  switch (event.type) {
    case 'session_update':         /* LLM stream / tool call / result */  break;
    case 'permission_request':     /* 弹审批 */                            break;
    case 'permission_resolved':    /* first-responder 已应答 */            break;
    case 'slow_client_warning':    /* ring 80% 满 */                       break;
    case 'session_died':           /* child 死了 */                        break;
    case 'unknown':                /* 向前兼容新 daemon */                  break;
  }
});

// 3) 发 prompt（HTTP request 挂起到 prompt 完成）
await client.prompt(sessionId, [{ type: 'text', text: 'hello' }]);

// 4) 关 session
await client.closeSession(sessionId);
unsubscribe();
```

详 [§03 §五 Capability negotiation](./03-http-api.md)。

### 4.2 curl 直走 HTTP/SSE

**建 session**：
```bash
curl -s -X POST http://127.0.0.1:4170/session \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $QWEN_SERVER_TOKEN" \
  -d "$(jq -nc --arg ws "$PWD" '{sessionScope:"session", workspaceCwd:$ws}')" \
  | tee /tmp/session.json
# {"sessionId":"sess_abc","clientId":"client_xyz","capabilities":{...}}
```

**订阅 SSE**（保持开着读流）：
```bash
SID=$(jq -r .sessionId /tmp/session.json)
curl -N -H "Accept: text/event-stream" \
  -H "Authorization: Bearer $QWEN_SERVER_TOKEN" \
  "http://127.0.0.1:4170/session/$SID/events"
# event: session_update
# id: 1
# data: {"id":1,"v":1,"type":"session_update","data":{"sessionUpdate":"agent_message_chunk","content":{"type":"text","text":"..."}}}
# event: session_update
# id: 2
# ...
```

**发 prompt**（另开一个 shell）：
```bash
curl -s -X POST "http://127.0.0.1:4170/session/$SID/prompt" \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $QWEN_SERVER_TOKEN" \
  -d '{"prompt":[{"type":"text","text":"write a fizzbuzz in TypeScript"}]}'
```

**Last-Event-ID 重连**：SSE 断了之后续：
```bash
curl -N -H "Accept: text/event-stream" \
  -H "Authorization: Bearer $QWEN_SERVER_TOKEN" \
  -H "Last-Event-ID: 42" \
  "http://127.0.0.1:4170/session/$SID/events"
# daemon 从 ring buffer 回拉 43..N 的 event
# 如果 ring 已 evict 过这段，daemon 推 client_evicted event 让 client 知道要重建状态
```

### 4.3 `/demo` 调试页

v0.16.0 含 `/demo`（PR#4132）—— 单 HTML browser-based debug UI，最薄 POST+SSE 验证面：

```bash
qwen serve
# 浏览器打开 http://127.0.0.1:4170/demo
# 输入 token（如果 --require-auth）→ 发 prompt → 看 typed event 流
```

主要用于：联调 daemon 协议时验证 wire shape / 看 typed event payload / 复现 client bug。**不是 production UI**。

### 4.4 ACP-native client（仅集成分支）

`daemon_mode_b_main` 上的 #4472 加了 `/acp` 端点的 official ACP Streamable HTTP transport（RFD #721），可让 **Zed / Goose / future ACP-native SDK 直接驱动 daemon**。**v0.16.0 main 不含**。等周期 merge。

---

## 五、常用操作 recipes

下表 route 都在 v0.16.0。完整 route 表见 [§03 §一](./03-http-api.md#一路由总览)。

### 5.1 Session 生命周期

| 操作 | 命令 |
|---|---|
| 建 session | `POST /session` `{workspaceCwd, sessionScope?}` |
| 列出 session | `GET /workspace/sessions` |
| 加载 session（attach） | `POST /session/:id/load` |
| 恢复 paused session | `POST /session/:id/resume` |
| 关闭 session（tombstone） | `DELETE /session/:id` |
| client 心跳 | `POST /session/:id/heartbeat` |

### 5.2 prompt + 控制

| 操作 | 命令 |
|---|---|
| 发 prompt | `POST /session/:id/prompt` `{prompt: ContentBlock[]}` |
| 取消 prompt | `POST /session/:id/cancel` |
| 切 model | `POST /session/:id/model` `{model}` |
| 切 mode | `POST /session/:id/mode` `{mode}` |
| 切 approval mode | `POST /session/:id/approval-mode` `{mode: "plan"\|"default"\|"auto-edit"\|"yolo", persist?: true}` |

### 5.3 Permission vote（first-responder）

```bash
# 客户端订阅 SSE 时收到：
# event: permission_request
# data: {"requestId":"req_abc", "tool":"Bash", "args":{"cmd":"npm test"}, ...}

# 任意 client 抢答：
curl -X POST "http://127.0.0.1:4170/session/$SID/permission/req_abc" \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $QWEN_SERVER_TOKEN" \
  -H "X-Qwen-Client-Id: $CLIENT_ID" \
  -d '{"outcome":"allow_with_scope", "scope":"session"}'

# winner 收到 permission_resolved 事件；loser 收到 permission_already_resolved
```

详 [§05 §五 Permission Vote 流](./05-permission-auth.md#五permission-vote-流pr3889--pr4232--完整)。

### 5.4 Workspace state CRUD（mutation gate 后）

启用 `--require-auth` 后这些都要 bearer + `X-Qwen-Client-Id` header。

| 操作 | route |
|---|---|
| 读 memory | `GET /workspace/memory` |
| 写 memory | `POST /workspace/memory` |
| 列 agents | `GET /workspace/agents` |
| 加/删 agents | `POST /workspace/agents` |
| 启/禁 tool | `POST /workspace/tools/:name/enable` |
| workspace init（scaffold QWEN.md，**不调 model**）| `POST /workspace/init` |
| MCP server 状态 | `GET /workspace/mcp` |
| MCP budget 状态 + push | `GET /workspace/mcp/budget` |
| MCP server restart | `POST /workspace/mcp/:server/restart` |

### 5.5 文件读写（sandbox-bounded）

```bash
# 读文件（PR#4269）
curl -G "http://127.0.0.1:4170/workspace/file" \
  --data-urlencode "path=src/index.ts" \
  -H "Authorization: Bearer $QWEN_SERVER_TOKEN" \
  -H "X-Qwen-Client-Id: $CLIENT_ID"

# 写文件（PR#4280，原子 temp+rename + content-hash precondition）
curl -X POST "http://127.0.0.1:4170/workspace/file" \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $QWEN_SERVER_TOKEN" \
  -H "X-Qwen-Client-Id: $CLIENT_ID" \
  -d '{"path":"src/index.ts", "content":"...", "expectedHash":"sha256-..."}'

# edit（single-match policy: oldText 必须 exactly 1 处匹配，0 或多处都报错）
curl -X POST "http://127.0.0.1:4170/workspace/file/edit" \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $QWEN_SERVER_TOKEN" \
  -H "X-Qwen-Client-Id: $CLIENT_ID" \
  -d '{"path":"src/index.ts", "oldText":"foo()", "newText":"bar()"}'
```

文件操作受 sandbox root + symlink_escape rejection 约束（PR#4282 / PR#4250）。

### 5.6 OAuth device-flow（远端 daemon auth provider）

```bash
# 1) 启动 device flow
RESP=$(curl -s -X POST "http://127.0.0.1:4170/workspace/auth/device-flow" \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $QWEN_SERVER_TOKEN" \
  -d '{"provider":"qwen"}')
echo "$RESP" | jq .
# {"userCode":"ABCD-EFGH", "verificationUri":"https://...", "interval":5, "expiresIn":900}

# 2) client UI 显示 userCode + verificationUri 让用户在浏览器登录
#    ⚠️ 浏览器开在 client 端，不是 daemon 端 —— daemon 经常在 headless container 无 GUI

# 3) 轮询完成
while true; do
  POLL=$(curl -s -X POST "http://127.0.0.1:4170/workspace/auth/device-flow/poll" \
    -H 'Content-Type: application/json' \
    -H "Authorization: Bearer $QWEN_SERVER_TOKEN" \
    -d '{"provider":"qwen"}')
  STATUS=$(echo "$POLL" | jq -r .status)
  case "$STATUS" in
    authorized) echo "✓ done"; break ;;
    pending)    sleep 5 ;;
    expired|denied) echo "✗ $STATUS"; break ;;
  esac
done
# token 落地到 ~/.qwen/auth/qwen.json (0o600)
```

详 [§05 §八 OAuth device-flow + BrandedSecret](./05-permission-auth.md)。

---

## 六、配置文件 & 持久化路径

| 路径 | 权限 | 内容 | 谁写 |
|---|---|---|---|
| `~/.qwen/serve/token` | `0o600` | bearer token（未设 `--token` / `QWEN_SERVER_TOKEN` 时 daemon 启动随机生成 + 写这） | daemon |
| `~/.qwen/auth/<provider>.json` | `0o600` | OAuth 凭证（device-flow 落地）| daemon |
| `~/.qwen/permissions.json` | — | global scope permission decisions | daemon |
| `~/.qwen/workspaces/<wsId>/permissions.json` | — | workspace scope decisions | daemon |
| `~/.qwen/serve/instances/<host>-<port>-<wsHash>/` | — | per-instance state（Wave 6 PR 29 计划，v0.16.0 未必完整）| daemon |

**SDK 默认 token 查找顺序**：
1. 显式传 `bearerToken` 参数
2. `QWEN_SERVER_TOKEN` env var
3. `~/.qwen/serve/token`（如果存在）

---

## 七、诊断 & 故障排查

### 7.1 健康 / 能力检查

```bash
# 浅层健康（仅 listener，无 auth）
curl -s http://127.0.0.1:4170/health
# {"status":"ok"}

# 深度健康（Stage 2a 候选，v0.16.0 不一定全开）
curl -s http://127.0.0.1:4170/health?deep=1 \
  -H "Authorization: Bearer $QWEN_SERVER_TOKEN"

# 能力清单
curl -s http://127.0.0.1:4170/capabilities | jq .
# {
#   "v": 1,
#   "mode": "http-bridge",
#   "workspaceCwd": "/path/to/repo",
#   "features": ["session_lifecycle", "session_prompt", ...],
#   "protocol_versions": {"acp": "0.14.x", "daemon_envelope": 1},
#   "auth": {"required": true, "type": "bearer"}
# }
```

### 7.2 Preflight 诊断（PR#4247，Wave 3 PR 13）

```bash
# daemon 启动 + 配置 readiness 整体检查
curl -s http://127.0.0.1:4170/workspace/preflight \
  -H "Authorization: Bearer $QWEN_SERVER_TOKEN" | jq .
# {
#   "cells": [
#     {"name":"node_version", "status":"ok"},
#     {"name":"cli_entry", "status":"ok"},
#     {"name":"workspace_dir", "status":"ok"},
#     {"name":"ripgrep", "status":"ok"},
#     {"name":"git", "status":"ok"},
#     {"name":"auth", "status":"not_started"},      # ACP 还没起
#     {"name":"mcp_discovery", "status":"not_started"},
#     ...
#   ]
# }
```

### 7.3 Env 诊断

```bash
# daemon host 关键环境信息（masked secrets）
curl -s http://127.0.0.1:4170/workspace/env \
  -H "Authorization: Bearer $QWEN_SERVER_TOKEN" | jq .
# {
#   "cells": [
#     {"name":"runtime", "status":"ok", "value":"node v24.12.0"},
#     {"name":"platform", "status":"ok", "value":"darwin arm64"},
#     {"name":"sandbox", "status":"ok"},
#     {"name":"proxy_https", ...}, {"name":"proxy_http", ...},
#     {"name":"env_var:OPENAI_API_KEY", "status":"present"},   # presence-only, 不暴露 value
#     ...
#   ]
# }
```

### 7.4 常见错误（`errorKind` closed taxonomy, PR#4251）

daemon 错误响应统一带 7-值 closed `errorKind`，client `switch` 处理：

| `errorKind` | 何时出现 | 应对 |
|---|---|---|
| `missing_binary` | `qwen` CLI 找不到 / `qwen --acp` child spawn 失败 | 检查 `which qwen`，确认 v0.16.0 装好 |
| `blocked_egress` | egress probe 失败（防火墙阻断 provider endpoint）| 看 `/workspace/preflight` 的 `egress` cell，检查防火墙 / proxy |
| `auth_env_error` | 必要 auth env 缺失 / 无效 | 检查 `OPENAI_API_KEY` / OAuth 凭证 |
| `init_timeout` | ACP child init 超时 | 网络慢 / provider endpoint 不可达 |
| `protocol_error` | ACP NDJSON schema 不匹配 / 帧损坏 | bug，看 daemon stderr，提 issue |
| `missing_file` | 文件读路径不存在 / 越 sandbox 边界 | 检查 path，是否在 workspace 内 |
| `parse_error` | request body parse / zod schema 失败 | 检查 request JSON 格式 |

### 7.5 HTTP 状态码常见映射

| HTTP | 何时出现 | 应对 |
|---|---|---|
| `401 Unauthorized` | 缺 bearer / 错误 token / 缺 `X-Qwen-Client-Id` 头 | 检查 token + clientId |
| `400 workspace_mismatch` | `POST /session` 携带 `workspaceCwd` 与 daemon 启动绑定的 workspace 不一致 | client 拿错 daemon URL，或 daemon 启动 workspace 不对 |
| `404 session_not_found` | URL 里的 `:sessionId` 在 daemon 内不存在 | session 已被 close / daemon 重启过 |
| `409` | `permission_already_resolved` / `WorkspaceInitConflictError` / `RestoreInProgressError` | concurrent state 冲突，retry |
| `503` | `--max-sessions` 已满 / MCP budget enforce 模式拒 | 等现有 session 关闭或调大 cap |

### 7.6 日志位置

| 日志 | 路径 |
|---|---|
| daemon stderr | 启动 terminal 或 systemd journal |
| daemon audit log | 由 `originatorClientId` 索引（每 mutation 一条；SHA-256-hashed paths）—— 具体路径取决于部署 |
| ACP child stderr | daemon 转发到自己 stderr |
| OpenTelemetry logs/spans/metrics | 如配 `OTEL_EXPORTER_OTLP_ENDPOINT` 等，发到 OTel collector（详 `docs/comparison/telemetry-architecture-deep-dive.md`）|

---

## 八、生产部署

### 8.1 systemd

```ini
# /etc/systemd/system/qwen-serve.service
[Unit]
Description=Qwen Code daemon
After=network.target

[Service]
Type=simple
User=qwen
Environment="QWEN_SERVER_TOKEN_FILE=/etc/qwen/serve.token"
ExecStartPre=/bin/sh -c 'export QWEN_SERVER_TOKEN=$(cat ${QWEN_SERVER_TOKEN_FILE})'
ExecStart=/usr/local/bin/qwen serve \
  --hostname 0.0.0.0 \
  --port 4170 \
  --require-auth \
  --workspace /var/qwen/workspace \
  --max-sessions 50

Restart=on-failure
RestartSec=5s
MemoryMax=4G

[Install]
WantedBy=multi-user.target
```

### 8.2 Docker

```dockerfile
FROM node:24-bookworm
RUN npm install -g @qwen-code/qwen-code@0.16.0
RUN useradd -ms /bin/bash qwen
USER qwen
WORKDIR /workspace
EXPOSE 4170
CMD ["qwen", "serve", "--hostname", "0.0.0.0", "--port", "4170", "--require-auth", "--workspace", "/workspace"]
```

```bash
docker run -d \
  -e QWEN_SERVER_TOKEN=$(openssl rand -hex 32) \
  -v /path/to/repo:/workspace \
  -p 4170:4170 \
  qwen-code-daemon
```

### 8.3 Kubernetes（1 pod = 1 daemon = 1 workspace 天然契合）

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata: {name: qwen-daemon}
spec:
  serviceName: qwen-daemon
  replicas: 1                          # 1 pod = 1 workspace
  selector: {matchLabels: {app: qwen}}
  template:
    metadata: {labels: {app: qwen}}
    spec:
      containers:
      - name: qwen
        image: qwen-code-daemon:0.16.0
        args:
          - serve
          - --hostname=0.0.0.0
          - --require-auth
          - --workspace=/workspace
          - --max-sessions=20
        env:
        - name: QWEN_SERVER_TOKEN
          valueFrom: {secretKeyRef: {name: qwen-token, key: token}}
        ports: [{containerPort: 4170}]
        volumeMounts:
        - name: workspace
          mountPath: /workspace
        resources:
          limits: {memory: 4Gi, cpu: 2}     # cgroup = per-tenant quota
        livenessProbe:
          httpGet: {path: /health, port: 4170, httpHeaders: [{name: Authorization, value: "Bearer $TOKEN"}]}
          initialDelaySeconds: 10
  volumeClaimTemplates:
  - metadata: {name: workspace}
    spec: {accessModes: [ReadWriteOnce], resources: {requests: {storage: 50Gi}}}
```

**多 tenant 部署 = 多 daemon process / pod**（OS 进程级 / namespace 级隔离）。1 daemon = 1 workspace 不变，不要在 daemon 内做应用层 tenant 抽象。详 [§05 §六 Multi-Tenant 约束](./05-permission-auth.md#六multi-tenant-关键约束)。

### 8.4 网络 egress 策略（生产 best practice）

来自 chiga0 [Issue #3803](https://github.com/QwenLM/qwen-code/issues/3803)：daemon host = runtime owner，所有 fs / tool / MCP / skill / shell / LSP / provider auth 在 daemon host 上 evaluate。

- **默认 deny-by-default + explicit allowlist** —— daemon host/pod 只允许 configured provider / MCP / skill 实际所需的 network surface
- **不需要 daemon 开公网** —— allowlist 列入 provider 端点 / MCP HTTP-SSE / skill 调用的外部 API
- **诊断** —— `/workspace/preflight` 的 `egress` cell 给 client 渲染 actionable error

### 8.5 凭据 / Secrets 都在 daemon host

| 类型 | 位置 |
|---|---|
| Provider OAuth tokens | daemon host `~/.qwen/auth/*.json` (0o600) |
| API keys / env vars | daemon process env（从 k8s secret / secret manager 注入）|
| MCP server credentials | MCP config 引用 daemon host env |
| SSH agent / kubeconfig | daemon host 本地（client 端的不自动传过来）|

**client credentials 不会自动可用** —— daemon host 必须自己持有。

---

## 九、当前限制 & 预览特性

### 9.1 v0.16.0 main 上已发布的能力

✅ daemon Wave 1-4 全 + Wave 5 PR 22-series（acp-bridge package 抽取）：
- 协议层：HTTP routes、SSE typed events、Last-Event-ID 重连、heartbeat、slow_client_warning、capability registry plug-in
- session：create/load/resume/cancel/close-delete + sessionScope override + bounded
- permission：first-responder vote + session-scoped permission + bounded record + parsePermissionOutcome shared helper
- auth：bearer token + clientId stamping + `--require-auth` mutation gate + OAuth device-flow with BrandedSecret 4-way redaction
- workspace：memory/agents/tools/approval-mode/init/MCP-restart CRUD + MCP budget warn/enforce
- 文件：sandbox-bounded read / write/edit (atomic temp+rename + content-hash precondition + edit single-match policy) + symlink_escape rejection
- 诊断：`/workspace/preflight` + `/workspace/env` 多 cell + closed `errorKind` 7-value taxonomy

### 9.2 v0.16.0 上**没有**（在 `daemon_mode_b_main` 集成分支等周期 merge）

- **F1 acp-bridge package 完整 self-sufficiency**（#4319 + #4334 + #4445）—— `httpAcpBridge.ts` 4682 LOC → 97 LOC shim
- **F2 shared MCP transport pool**（#4336）—— N session 共享 `(name + fingerprint)` 一个 transport（解 "double MCP discovery in ACP child" 浪费）+ workspace-scope budget
- **F3 PermissionMediator 4 strategies**（#4335）—— `designated` / `consensus` / `local-only` + audit ring 512-entry + `permission_partial_vote` / `permission_forbidden` typed events（v0.16.0 main 上 permission 仅 first-responder）
- **F4 prereq daemon protocol completion**（#4360）—— `_meta.serverTimestamp` / `errorKind` on `stream_error` / tool `provenance` + `serverId` / `state_resync_required` SSE gap detection
- **chiga0 SDK 共享 daemon UI 层**（#4328 / #4353）—— `@qwen-code/sdk-typescript/daemon-ui` 共享 reducer + render contract + 28+ typed event coverage
- **ACP Streamable HTTP transport**（#4472 DRAFT）—— `/acp` 端点接 ACP-native client（Zed / Goose 等）
- **ytahdn web-shell**（#4380）—— `packages/web-shell` daemon-backed React web client（**第一个真实 web client 消费者**）

### 9.3 与 native `qwen` TUI 的关系

**`qwen` 命令仍直连本地 runtime 不走 daemon** —— 这是长期保留的本地 TUI 路径（零网络 / 最低复杂度）。`qwen serve` 是给 web / remote runtime / 多 client 协作场景的，不是 TUI 替代。Stroustrup 零成本抽象原则 —— **不要为不需要的东西付费**。

详 [§02 §7 部署模式](./02-architectural-decisions.md#7-部署模式--mode-b-mainline--mode-a-parking-lot)。

### 9.4 Mode A (`qwen --serve`) parking lot

`qwen --serve` 模式（CLI + HTTP server 同进程，本地 TUI super-client）2026-05-15 起 parking lot，不进近期主线。Mode B (`qwen serve` headless) 是唯一推荐生产模式。

---

## 十、延伸阅读

| 章节 | 内容 |
|---|---|
| [§01 Overview](./01-overview.md) | 架构图 + decision matrix + Wave 进度 + 资源经济性 |
| [§02 Architectural Decisions](./02-architectural-decisions.md) | 8 决策：1 daemon = 1 workspace × N session / MCP 生命周期 / FileReadCache / Permission flow / 多 client 并发 / Mode B mainline / server-client-runtime boundary |
| [§03 HTTP API & Protocol](./03-http-api.md) | 完整路由表 + SSE 事件 catalog + capability tags + `errorKind` taxonomy + 三套来源 lockstep |
| [§04 Deployment & Client](./04-deployment-and-client.md) | Deployment shape matrix + client convergence + Runtime locality + Reverse RPC 5 类 |
| [§05 Security & Permission](./05-permission-auth.md) | 三层权限模型 + bearer + clientId stamping + mutation gate + permission vote + OAuth device-flow + typed-error 设计哲学 + audit hooks |
| [§06 Roadmap & Ecosystem](./06-roadmap.md) | Timeline + 31-PR Wave breakdown + chiga0 #3803 6 contracts + F-series 重组 + vs OpenCode + vs Anthropic Managed Agents |
| 上游 issue tracker | [QwenLM/qwen-code#4175](https://github.com/QwenLM/qwen-code/issues/4175) doudouOUC Mode B v0.16 31-PR rollout plan |

---

## 附录 A：常见 capability tag 速查

通过 `GET /capabilities` 返的 `features` 数组协商能力。v0.16.0 ~28 个 tag，client `if (caps.features.includes(...))` 决定是否启用对应 UI：

```
# Stage 1 baseline (9):
session_lifecycle / session_prompt / session_cancel /
session_model_switch / session_mode_switch / permission_vote /
sse_events / last_event_id_replay / capabilities_endpoint

# Wave 1:
capability_registry_v1 / typed_session_event_v1 / typed_control_event_v1 /
daemon_session_client_v1 / lockstep_invariant_v1

# Wave 2:
session_scope_override / session_load / session_resume /
client_id_stamped / session_scoped_permission / permission_bounded_record

# Wave 2.5:
session_heartbeat / slow_client_warning / session_close_delete

# Wave 3:
workspace_status_readonly / preflight_diagnostics / env_diagnostics /
mcp_restart_guarded / mcp_budget_push

# Wave 4:
mutation_gate / workspace_memory_crud / workspace_agents_crud /
workspace_tools_crud / workspace_init / session_approval_mode /
fs_boundary_enforced / workspace_file_read / workspace_file_write /
auth_device_flow / branded_secret
```

仅集成分支（不在 v0.16.0）：`mcp_workspace_pool` / `mcp_pool_restart`（F2 #4336）+ `permission_mediation` modes（F3 #4335）+ 新 typed event `permission_partial_vote` / `permission_forbidden` / `state_resync_required` 等。

## 附录 B：典型 troubleshooting decision tree

```
症状 → 先查 → 再查
═══════════════════════════════════════════════════════════════
daemon 启动失败          → stderr 看 errorKind   → 0.0.0.0 + 无 token? 改 loopback 或 加 token
401 Unauthorized         → bearer 是否带          → clientId 是否带
                                                  → require-auth 模式下 /health 也要 token
400 workspace_mismatch   → daemon 启动 workspace  → curl 时 workspaceCwd 是否同一绝对路径
404 session_not_found    → daemon 是否重启过       → session 是否被 DELETE
SSE 流断了               → Last-Event-ID 重连     → 看 client_evicted 事件，超 ring 要 loadSession 重建
permission_request 卡住  → 是否有 client 在 vote  → 60s 默认 deny；cancel 解锁
prompt 不响应             → /workspace/preflight   → ACP child 是否存活（session_died 事件）
                                                  → blocked_egress?
                                                  → auth_env_error?
MCP server 不见            → /workspace/mcp 状态    → budget mode=enforce 时被拒？
                                                  → mcp_discovery preflight cell
文件写返回 missing_file    → 是否在 workspace 内    → 是否 symlink_escape
                                                  → 是否 prior-read 守卫触发
```

---

> 用户文档基于 qwen-code **v0.16.0**（2026-05-21 release）source 编写。后续 daemon Mode B 功能完备宣告（daemon Wave plan PR 31）= 周期 merge `daemon_mode_b_main` 回 main 后下一个 semver 版本。
