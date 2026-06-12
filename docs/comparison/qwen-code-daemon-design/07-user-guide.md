# Qwen Code daemon (`qwen serve`) 用户使用文档

> Recipe-oriented 用户指南。daemon 功能集已合入 qwen-code main（PR#4490），随 **v0.18.0 正式版**发布（npm `@qwen-code/qwen-code@latest`，2026-06-12）。本文涵盖启动、客户端接入、常用操作、生产部署、故障排查，所有 flag / 路由 / 默认值已对照 main 分支源码（`packages/cli/src/serve/`）核实。

## TL;DR

`qwen serve` 启动一个本地 HTTP daemon，把 qwen-code 的 agent 能力暴露给**多种 client**（web UI / 桌面 app / IDE / Zed / Goose / MCP client / 自动化脚本）—— 不用每个 client 自己 spawn `qwen --acp` 子进程。

**典型场景**：

- 浏览器里跑 **web-shell**（`@qwen-code/web-shell` React 组件），让 agent 操作本地 repo
- 远端 cloud devbox / Codespaces / K8s pod 跑 daemon，本地浏览器 / IDE 连进去
- 同一 session 多 tab / 多 client 协作，shared permission 审批
- Zed / Goose 等 ACP-native client 接入（`/acp` 端点，HTTP 或 WebSocket）
- Qoder / Claude Desktop / Cursor 等 MCP client 通过 `qwen-serve-mcp` 桥接入
- CI / 自动化脚本通过 HTTP 调用 agent

**`qwen serve` ≠ 替代 `qwen` CLI**。本地单人开发不需要多 client / 不需要 web UI，**继续用 `qwen` native TUI**（直连本地 runtime，零网络 / 最低复杂度）。daemon 是给上面那些场景的，不是给 TUI 用户的。

### 按需阅读导航

| 你是 | 跳到 |
|---|---|
| 第一次配 daemon，5 分钟跑起来 | §一 快速上手 → §五 常用操作 recipes |
| 想理清术语（bearer / session / workspace / clientId 等） | §〇 核心概念 |
| 配启动参数 / 安全启动 | §二 启动选项 |
| 选客户端接入方式（web-shell / 桌面 app / SDK / ACP / MCP 桥 / curl） | §四 客户端接入 |
| 上 systemd / Docker / K8s 生产 | §八 生产部署 |
| 出问题排查 | §七 诊断 & 故障排查 → 附录 B decision tree |
| 多 client 协作选 permission strategy | §2.6 Permission strategy 选择 |

---

## 〇、核心概念

> 后续章节会反复用到的术语，先把心智模型建好，遇到生词回来查。

### 进程与状态

| 术语 | 含义 |
|---|---|
| **`qwen serve`** | 启动 daemon 的子命令；起一个 HTTP server 进程，里面再 spawn 一个 `qwen --acp` agent 子进程（`mode=http-bridge`，当前唯一实现模式） |
| **ACP child** | daemon 内嵌的 `qwen --acp` 子进程，真正执行 LLM 调用 / tool / shell / MCP / skill 的 runtime |
| **workspace** | daemon 启动时绑定的**一个**绝对路径目录（默认 `process.cwd()`，可 `--workspace` 指定）。**1 daemon = 1 workspace**，多 workspace 就起多个 daemon 各占一个端口 |
| **session** | 一段对话上下文。1 daemon = N session 复用同一个 ACP child，session 之间对话隔离但共享同一 workspace 与 OS 权限 |

### 客户端、认证与协议

| 术语 | 含义 |
|---|---|
| **client** | 调 daemon 的程序：web-shell / SDK / curl 脚本 / ACP client / MCP 桥等。一个 daemon 可被多 client 同时连接，同一 session 的事件可被多 client 订阅 |
| **bearer token** | HTTP 标准 `Authorization: Bearer <token>` 认证（RFC 6750）。来源：`--token` flag 或 `QWEN_SERVER_TOKEN` env。详 §2.3 |
| **`X-Qwen-Client-Id`** | client 的自报身份 header（≤128 字符）。`POST /session` 响应返回 daemon 分配的 `clientId`，后续请求回带；permission 投票用它判"哪个 client 在投"。**无所有权证明**——能看到 SSE 帧上 `originatorClientId` 的 client 可声明同一个 id（见 §2.6 designated 的安全边界） |
| **REST + SSE** | 主 northbound transport：HTTP request/response + `GET /session/:id/events`（`text/event-stream`）推 `session_update` / `permission_request` / `turn_complete` 等 typed event |
| **非阻塞 prompt** | `POST /session/:id/prompt` **立即返回 `202 {promptId, lastEventId}`**，不等 turn 结束（PR#4585）。turn 的产出与完成都在 SSE 流上：流式输出 → `turn_complete` 或 `turn_error`（带同一 `promptId`） |
| **`/acp`** | 第二 northbound transport（ACP Streamable HTTP，RFD #721），Zed / Goose 等 ACP-native client 直接接入；同一端点支持 WebSocket upgrade。env `QWEN_SERVE_ACP_HTTP=0` 关 |
| **capability tag** | `GET /capabilities` 的 `features` 数组；client 用 `caps.features.includes(...)` 做能力协商与优雅降级。部分 tag 仅在对应 flag 开启时公布（条件 tag）。完整列表见附录 A |
| **`Last-Event-ID`** | SSE 标准头。重连时带上次见到的最大 event id，daemon 从 ring buffer（默认 8000 条/session）回放断流期间的 missed events |

### 权限与错误模型

| 术语 | 含义 |
|---|---|
| **permission request / strategy** | agent 调有副作用的 tool 时，daemon 经 SSE 推 `permission_request` 给所有订阅 client 审批，默认 5 分钟无人应答按 cancelled 解决；多 client 的裁决方式由 4 种 strategy 决定（§2.6） |
| **mutation gate** | 写操作路由的认证闸。配置了 token 后全部路由过 bearer；**未配 token 的 loopback 默认**下，普通路由放行，但 strict 路由（文件写 / memory 写 / device-flow / reload 等）直接 401 `token_required` |
| **`errorKind`** | daemon 错误响应带的 closed enum（`missing_binary` / `blocked_egress` / `prompt_deadline_exceeded` 等 14 值），client `switch` 处理而不是 regex 匹配错误文案。速查见 §7.4 |

---

## 一、快速上手（5 分钟）

```bash
# 1) 安装（daemon 随 v0.18.0 正式版发布）
npm install -g @qwen-code/qwen-code@latest

# 2) 启动 daemon（loopback 默认免 token，仅本机可达）
cd /path/to/your/repo
qwen serve
# stdout: qwen serve listening on http://127.0.0.1:4170 (mode=http-bridge, workspace=/path/to/your/repo)

# 3) 验证健康 + 看能力清单
curl -s http://127.0.0.1:4170/health          # {"status":"ok"}
curl -s http://127.0.0.1:4170/capabilities | jq '{workspaceCwd, features: .features[:8]}'

# 4) 建 session（cwd 可省略——省略即绑定 daemon 的 workspace）
SID=$(curl -s -X POST http://127.0.0.1:4170/session \
  -H 'Content-Type: application/json' -d '{}' | jq -r .sessionId)

# 5) 先订阅 SSE（后台挂着，事件落到文件）
curl -sN -H "Accept: text/event-stream" \
  "http://127.0.0.1:4170/session/$SID/events" > /tmp/events.log &

# 6) 发 prompt —— 非阻塞：立即 202 返回 promptId
curl -s -X POST "http://127.0.0.1:4170/session/$SID/prompt" \
  -H 'Content-Type: application/json' \
  -d '{"prompt":[{"type":"text","text":"hello"}]}'
# {"promptId":"6f1c...","lastEventId":3}

# 7) 在事件流里等 turn 结束
tail -f /tmp/events.log    # agent 流式输出 → event: turn_complete（data 带同一 promptId）即完成
```

注意第 6 步的语义：**HTTP 响应不携带模型输出**。输出（`session_update` 流式 chunk）与完成信号（`turn_complete` / `turn_error`）都在第 5 步订阅的 SSE 流上。正确的调用顺序永远是 **先订阅 SSE → 再发 prompt → 在事件流里等 `turn_complete`**。

跑通这 7 步就有 daemon 了。后面是把它做对、做安全、上生产的细节。

---

## 二、`qwen serve` 启动选项

### 2.1 基本 flag

源码：`packages/cli/src/commands/serve.ts`。

| flag | 默认 | 说明 |
|---|---|---|
| `--port <N>` | `4170` | TCP 端口；`0` = OS 分配 ephemeral |
| `--hostname <addr>` | `127.0.0.1` | 绑定 interface；loopback（`127.0.0.1` / `localhost` / `::1` / `[::1]`）默认免 token，**其他值必须配 token 否则拒绝启动** |
| `--token <str>` | — | bearer token；**生产走 env 而非 flag**（`--token` 在 `/proc/<pid>/cmdline` 任何本机用户可见，启动时 stderr 会 warn） |
| `--workspace <abs path>` | `process.cwd()` | 绑死的 workspace 绝对路径；`POST /session` 携带不一致 cwd 返 `400 workspace_mismatch` |
| `--require-auth` | `false` | loopback 也强制 token（必须配 `--token` / env，否则拒绝启动）；启用后 `/health` `/demo` 也要 bearer（K8s/Compose 探针需带 token） |
| `--allow-origin <origin>` | — | 浏览器 client 的 CORS 白名单，可重复（PR#4527）。值为 canonical origin（`https://host:port`，不支持子域通配）或 `*`（`*` 必须已配 token，否则拒绝启动）；`Origin: null` 永远拒绝 |
| `--max-sessions <N>` | `20` | 并发活 session 上限，超出 spawn 返 503；attach 现有 session 不受限；`0` = 禁用 |
| `--max-connections <N>` | `256` | listener 级 TCP socket 上限，慢/僵尸 SSE client 在 accept 时被拒；`0` = 禁用 |
| `--event-ring-size <N>` | `8000` | per-session SSE 重放 ring 深度（`Last-Event-ID` 重连可回拉的 backlog） |
| `--mcp-client-budget <N>` | — | workspace 级 MCP client 数上限（N session 共享 transport pool，预算按 workspace 计而非按 session 乘） |
| `--mcp-budget-mode <off\|warn\|enforce>` | 设了 budget 则 `warn`，否则 `off` | `warn`：≥75% 预算时发警告不拒；`enforce`：超额按 `mcpServers` 声明顺序拒绝（`disabledReason: "budget"`）；`enforce` 必须配 budget |
| `--allow-private-auth-base-url` | `false` | 允许 `/workspace/auth/provider` 安装 localhost / 私网 baseUrl（仅本地开发用） |

### 2.2 超时 / 限流 / 回收 flag

| flag | 默认 | 说明 |
|---|---|---|
| `--prompt-deadline-ms <N>` | —（env `QWEN_SERVE_PROMPT_DEADLINE_MS`） | 单 turn 的 server 侧 wallclock 上限（PR#4530）；超时 turn 以 `prompt_deadline_exceeded` 失败。请求体也可带 `deadlineMs` 做 per-prompt 覆盖 |
| `--writer-idle-timeout-ms <N>` | —（env `QWEN_SERVE_WRITER_IDLE_TIMEOUT_MS`） | 单条 SSE 连接 idle 上限（PR#4530），踢掉 TCP 半死连接 |
| `--channel-idle-timeout-ms <N>` | `0`（立即） | 最后一个 session 关闭后 ACP child 保活多久再 kill |
| `--session-reap-interval-ms <N>` | `60000` | session 回收器扫描间隔；`0` = 关 |
| `--session-idle-timeout-ms <N>` | `1800000`（30 min） | 断连 session 闲置多久被回收；`0` = 关 |
| `--rate-limit` | off（env `QWEN_SERVE_RATE_LIMIT=1`） | 启用分层限流（PR#4861）：prompt 10/min、mutation 30/min、read 120/min（per-client）；`/health`、heartbeat、SSE、`/acp` 豁免；超限返 `429` + `Retry-After` |
| `--rate-limit-prompt` / `--rate-limit-mutation` / `--rate-limit-read` | `10` / `30` / `120` | 各 tier 每窗口上限，需 `--rate-limit` |
| `--rate-limit-window-ms <N>` | `60000` | 限流窗口（≥1000） |

### 2.3 Bearer token 认证基础

**什么是 bearer**：HTTP 标准认证 scheme（RFC 6750）——谁持有这串 token 谁就能调 API。请求头格式固定：

```
Authorization: Bearer <token>
```

**Token 来源**（daemon 启动时，二选一）：

| 来源 | 推荐度 | 备注 |
|---|---|---|
| `QWEN_SERVER_TOKEN` env var | 推荐 | `/proc/<pid>/environ` 仅 owner 可读 |
| `--token <str>` CLI flag | 不推荐生产 | `/proc/<pid>/cmdline` 任何本机用户可见；启动时 stderr warn |

> daemon **不会自动生成 token**。不配 token 的 loopback 部署 = 开发者默认形态（同机任意用户可访问，strict 路由除外）；任何共享主机 / 远端部署都应显式配 token。注意从文件读 token 时不要带换行：`export QWEN_SERVER_TOKEN=$(cat token.txt)` 会把尾部 `\n` 带进值里导致永远 401，用 `$(printf %s "$(cat token.txt)")` 或确保文件无换行。

**daemon 端实现要点**（源码：`packages/cli/src/serve/auth.ts`）：

- **SHA-256 hash + `crypto.timingSafeEqual` 比对**，401 对缺 header / 错 scheme / 错 token 返回完全一致——防 timing / 探测类 side-channel
- **非 loopback + 无 token**、**`--require-auth` + 无 token**、**`--allow-origin '*'` + 无 token** 三种组合都**拒绝启动**
- loopback 绑定时 enforce **Host header allowlist**（防 DNS rebinding）；浏览器跨域默认全拒，须 `--allow-origin` 显式放行

**bearer 与 `X-Qwen-Client-Id` 的分工**：bearer 解决"这个调用者允许进来吗"（传输层）；clientId 解决"这是协作中的哪个 client"（应用层身份 / 审计 / 投票归属）。`POST /session` 响应返回 daemon 分配的 `clientId`，client 后续请求回带；该 header 自报、无 proof-of-possession，**不防冒名**——挡不受信 client 靠 bearer。另注意 OAuth device-flow 是 **daemon 与上游模型 provider** 之间的事（§5.6），不是 client 与 daemon 之间的认证。

### 2.4 安全启动配方

**本机开发**——最省事：

```bash
cd /path/to/repo
qwen serve              # loopback 免 token，仅 127.0.0.1 可达
```

**共享开发机 / CI runner**——不让本机其他用户访问：

```bash
export QWEN_SERVER_TOKEN=$(openssl rand -hex 32)
qwen serve --require-auth          # loopback 也要 bearer，/health /demo 也被 gate
```

**暴露远端**（cloud devbox / K8s pod）——`0.0.0.0` + 无 token 会直接拒绝启动（防裸奔上线）：

```bash
export QWEN_SERVER_TOKEN=$(openssl rand -hex 32)
qwen serve --hostname 0.0.0.0 --port 4170
```

**浏览器跨域 client** 加 `--allow-origin http://localhost:5173`（可重复列多个 origin）。

**Production hardened**：

```bash
export QWEN_SERVER_TOKEN=$(openssl rand -hex 32)
qwen serve --require-auth \
  --hostname 0.0.0.0 \
  --max-sessions 50 \
  --max-connections 500 \
  --rate-limit \
  --prompt-deadline-ms 600000 \
  --writer-idle-timeout-ms 120000 \
  --mcp-client-budget 8 --mcp-budget-mode enforce
```

### 2.5 环境变量

| env | 说明 |
|---|---|
| `QWEN_SERVER_TOKEN` | bearer token（生产推荐用这个而非 `--token`） |
| `QWEN_SERVE_PROMPT_DEADLINE_MS` / `QWEN_SERVE_WRITER_IDLE_TIMEOUT_MS` | 对应 flag 的 env 形式 |
| `QWEN_SERVE_RATE_LIMIT=1` + `QWEN_SERVE_RATE_LIMIT_PROMPT/_MUTATION/_READ/_WINDOW_MS` | 限流的 env 形式（flag 优先） |
| `QWEN_SERVE_ACP_HTTP=0` | 关 `/acp` ACP transport（默认开） |
| `QWEN_SERVE_NO_MCP_POOL=1` | 关 MCP shared transport pool，回退 per-session client（`mcp_workspace_pool` tag 随之不公布） |
| `QWEN_SERVE_MCP_POOL_TRANSPORTS` | pool 接受的 MCP transport 类型，默认 `stdio,websocket`；显式加 `http,sse` 才让 HTTP/SSE MCP 进 shared pool |
| `QWEN_DAEMON_LOG_FILE=0` | 关 daemon 文件日志（PR#4559，详 §7.3） |
| `QWEN_CODE_SUPPRESS_YOLO_WARNING=1` | 压制 "yolo + 无 sandbox" 启动警告 |

### 2.6 Permission strategy 选择

4 个 strategy 通过 settings.json（全局 `~/.qwen/settings.json` 或 workspace `<ws>/.qwen/settings.json`）的 `policy` 段 opt-in，**修改后须重启 daemon**（boot 时读一次）：

```json
{
  "policy": {
    "permissionStrategy": "first-responder",
    "consensusQuorum": null
  }
}
```

| strategy | 行为 | 用例 |
|---|---|---|
| `first-responder`（默认） | 任意 client 抢答，后到者收 `permission_already_resolved` | 本地单用户 / 默认 |
| `designated` | 仅 prompt 发起 client 能决；其他 client 应答 403 `permission_forbidden`；匿名 prompt（无 clientId）回退 first-responder。**安全边界**：clientId 自报无所有权证明，能观察 SSE 的 client 可冒用 originator id——仅适用于 client 彼此受信、只为防误触的场景 | 多 client 但有"主控" client |
| `consensus` | N-of-M 投票，默认 N = `floor(M/2)+1`（**M=2 时即全体一致**；平票只能等超时），可 `policy.consensusQuorum` 固定 quorum；中间票推 `permission_partial_vote` 事件 | 多人 review 工作流 |
| `local-only` | 仅 loopback client 能批准/拒绝（看 kernel-stamped `req.socket.remoteAddress`，**不看 `X-Forwarded-For`**）；remote 应答 403。remote 仍可投 `{"outcome":"cancelled"}` 中止请求 | 远端 daemon 但审批只收本地 |

无论哪种 strategy，permission request 默认 **5 分钟**无人应答按 cancelled 解决（防一个没人理的审批永久阻塞后续 prompt）。

更深入的权限/认证设计见 [05-permission-auth.md](./05-permission-auth.md)。

---

## 三、2 种 deployment shape

> 核心不变式：**daemon host = workspace host = runtime host**。所有 fs / tool / MCP / skill / shell / 网络 egress / 凭据存取在 daemon 所在那台机器上 evaluate。"daemon 在远端但 workspace 在本地"是反模式——daemon 看不到本地文件，`POST /session` 携带本地 cwd 会被 `400 workspace_mismatch` 拒。

### Shape 1: Local single-machine（本地单机）

```
[ laptop:  qwen serve (workspace=/repo)  ←HTTP/SSE←  web-shell / curl / SDK ]
```

- daemon / workspace / client 都在本机；loopback 默认形态即可
- 用例：开发期跑 web-shell；多个 tab 连同一 daemon；本机工具链集成
- 提醒：native TUI（`qwen`）不走 daemon——这个 shape 是给 web client / 多 client 协作的

### Shape 2: Cloud / devbox remote-runtime（远端 runtime + 本地 client）

```
[ laptop: browser / SDK / IDE ]  ←HTTP/SSE→  [ devbox/pod: qwen serve (workspace=/workspace) ]
```

- daemon host **==** workspace host（同一台远端机器/pod），client 在本地
- 用例：Codespaces / Coder / 内部 cloud devbox
- 必须：token + `--hostname 0.0.0.0` + TLS reverse proxy（§8.4）；浏览器 client 还需 `--allow-origin`

更多部署形态推演见 [04-deployment-and-client.md](./04-deployment-and-client.md)。

---

## 四、客户端接入

daemon 对外提供**两套 northbound transport 共存**，共享同一 bridge + event bus（同一 session 可从任一 transport 访问）：**REST + SSE**（`/session/*` / `/workspace/*` / `/file*`，qwen-specific，给 web-shell / TS SDK / curl）与 **ACP**（`/acp` 单端点，RFD #721 标准，HTTP + WebSocket，给 Zed / Goose 等 ACP-native client）。在这之上还有一层 **MCP 桥**（`qwen-serve-mcp`），把 REST API 包装成 MCP server 给 MCP client 用。按"绝大多数用户最先用 → 集成开发"的顺序：

### 4.1 web-shell —— 浏览器 client（推荐 web 入口）

`@qwen-code/web-shell`（npm 包，源码 `packages/web-shell`）是 daemon-backed 的浏览器终端/会话 UI，以 **React 组件**形态发布，可嵌入任何 React 应用。能力与 CLI 对齐：session create / load / resume、流式输出与取消、model / approval mode 切换、slash command 补全、permission 审批弹窗、多 tab 共享同一 session、图片上传回显（#4922）、goal / background task 工作流与 task 审批（#4856）、内联 `/settings` 面板（鼠标可达，#4944/#4972）与 context 用量面板（#4958）。

```tsx
import { WebShellWithProviders } from '@qwen-code/web-shell';

export function QwenCodePanel() {
  return (
    <WebShellWithProviders
      baseUrl="http://127.0.0.1:4170"
      token="your-bearer-token"
      theme="dark"
      language="zh-CN"
    />
  );
}
```

多视图共享同一 session（chat + terminal）时宿主自建 `DaemonWorkspaceProvider` + `DaemonSessionProvider`（来自 `@qwen-code/webui` 的 daemon-react-sdk），`WebShell` 只消费 hooks。

部署注意：web-shell 静态资源与 daemon 是**两个服务**（Vite / nginx vs `qwen serve`），生产用反向代理把页面与 daemon API 归到同一 origin（§8.4），否则要给 daemon 配 `--allow-origin <web-shell origin>`；repo 内开发可直接 `packages/web-shell` 下 `npm run dev` 连本地 daemon。

### 4.2 官方桌面 app（packages/desktop）

qwen-code 仓库内置 Electron 桌面 app（`packages/desktop`，PR#3778）：多 session 聊天工作台 + skills / 文件预览 / automations / permission mode。它通过 **ACP 直接驱动捆绑的 qwen-code runtime**（打包时 vendor 一份 CLI），开箱即用、不要求用户预先装 `qwen` 或手动起 daemon——适合"想要图形界面但不想碰 HTTP API"的用户。要让多个外部 client 共享同一 agent 时仍然用 `qwen serve`。

### 4.3 TypeScript SDK：`@qwen-code/sdk`

daemon 的官方 HTTP client 在 TS SDK 里（`DaemonClient` + `DaemonSessionClient`）：

```ts
import { DaemonClient, DaemonSessionClient } from '@qwen-code/sdk';

const daemon = new DaemonClient({
  baseUrl: 'http://127.0.0.1:4170',
  token: process.env['QWEN_SERVER_TOKEN'],   // 远端 / require-auth 时必填
});

const caps = await daemon.capabilities();
const session = await DaemonSessionClient.createOrAttach(daemon, {
  workspaceCwd: caps.workspaceCwd,
});

// typed 事件流（async iterator，内部处理 Last-Event-ID 重连）
const ctl = new AbortController();
void (async () => {
  for await (const event of session.events({ signal: ctl.signal })) {
    console.log(event.type, event.data);
  }
})();

// wire 上是非阻塞 202 + promptId；SDK 替你在事件流里等到对应的
// turn_complete / turn_error 再 resolve —— 调用侧仍是一次 await
const result = await session.prompt({
  prompt: [{ type: 'text', text: 'Summarize this workspace.' }],
});
console.log(result.stopReason);
```

写自己的 web / TUI client 时不要 reimplement streaming merge / permission state——SDK 同包导出 reducer 层：`asKnownDaemonEvent` / `createDaemonSessionViewState` / `reduceDaemonSessionEvent`（typed event → transcript view state，framework-free）。

**Python / Java SDK 不是 daemon client**：`qwen-code-sdk`（Python）与 Java SDK 都是自己 spawn `qwen` CLI 子进程（ACP stdio）的嵌入式 SDK——用它们写自动化**不需要**起 daemon；需要"多 client 共享一个 agent / 远端 runtime"才用 daemon + TS SDK 或裸 HTTP。

### 4.4 curl 直走 REST + SSE

**建 session**（`cwd` 可省略；`sessionScope` 可选 `'single'`（默认，共享 scope）或 `'thread'`）：

```bash
curl -s -X POST http://127.0.0.1:4170/session \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $QWEN_SERVER_TOKEN" \
  -d '{}' | tee /tmp/session.json
# {"sessionId":"...","clientId":"...",...}
SID=$(jq -r .sessionId /tmp/session.json)
CID=$(jq -r .clientId /tmp/session.json)
```

**订阅 SSE**（保持挂着读流；事件帧形如 `event: session_update` + `id: 1` + `data: {...}`）：

```bash
curl -sN -H "Accept: text/event-stream" \
  -H "Authorization: Bearer $QWEN_SERVER_TOKEN" \
  "http://127.0.0.1:4170/session/$SID/events"
```

可选 query：`?maxQueued=N`（per-subscriber 队列深度，16–2048，默认 256；满 75% 推 `slow_client_warning`）、`?snapshot=1`（replay 结束后补一帧 `session_snapshot`，带当前 model / approval mode，重连 client 免一次额外往返）。

**Last-Event-ID 重连**：断流后加一个 `-H "Last-Event-ID: 42"` 头续订，daemon 从 ring 回放漏掉的事件；若这段已被 evict，daemon 推 `state_resync_required`，client 应走 `POST /session/:id/load` 重建状态再继续消费。

### 4.5 ACP-native client（Zed / Goose / RFD #721 SDK）

`/acp` 端点实现 ACP Streamable HTTP transport（RFD #721），ACP-native client 不需要懂 qwen REST 方言：

```
POST   /acp {initialize}   → 200 + capabilities + `Acp-Connection-Id` 响应头
GET    /acp（带 conn 头）   → connection-scoped SSE 流
POST   /acp {其他 method}  → 202，reply 走 SSE
DELETE /acp                → 202 teardown
```

同一路径还接受 **WebSocket upgrade**（`ws://host:4170/acp`，token 放 `Authorization` 头；同样做 Host 白名单 + Origin CSRF 检查），适合不便维护 SSE+POST 两条腿的 client。

- ACP 标准 method 直接走：`initialize` / `session/new` / `session/load` / `session/prompt` / `session/cancel` / `session/request_permission`；**model / mode 切换走标准 `session/set_config_option`**（categories `model` / `mode`）
- daemon 扩展放 `_qwen/...` namespace（`_qwen/session/heartbeat`、`_qwen/workspace/preflight`、`_qwen/file/read` 等），在 `initialize` 响应的 `_meta.qwen.methods` 里公布，client feature-detect 后再调

### 4.6 MCP 桥：`qwen-serve-mcp`

把 daemon HTTP API 封装成 MCP server（PR#4555，bin 随 `@qwen-code/sdk` 发布），任何 MCP client（Qoder / Claude Desktop / Cursor 等）都能驱动 daemon：

```jsonc
// MCP client 配置
{
  "mcpServers": {
    "qwen-serve-bridge": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "-p", "@qwen-code/sdk", "qwen-serve-mcp"],
      "env": {
        "QWEN_DAEMON_URL": "http://127.0.0.1:4170",
        "QWEN_DAEMON_TOKEN": "<your-token>"
      }
    }
  }
}
```

env：`QWEN_DAEMON_URL`（默认 `http://127.0.0.1:4170`）/ `QWEN_DAEMON_TOKEN` / `QWEN_WORKSPACE_CWD`。

### 4.7 `/demo` 调试页

`/demo` 是 daemon 内置的单文件 HTML debug UI（无构建、无外部依赖）——最薄的 POST+SSE 验证面：浏览器打开 `http://127.0.0.1:4170/demo`，填 token（如有）→ 建 session → 发 prompt → 看 typed event 流。用于联调协议 wire shape / 复现 client bug，**不是 production UI**（生产用 §4.1 web-shell）。`--require-auth` 下 `/demo` 也要 bearer。

---

## 五、常用操作 recipes

完整 HTTP API 面（50+ 路由）见 [03-http-api.md](./03-http-api.md)；运行时以 `GET /capabilities` 的 feature tag 为准。

### 5.1 Session 生命周期

| 操作 | 命令 |
|---|---|
| 建 session | `POST /session` `{cwd?, sessionScope?, modelServiceId?}` → `{sessionId, clientId, ...}` |
| 列 workspace 的 session | `GET /workspace/<url-encoded 绝对路径>/sessions`（如 `/workspace/%2Fwork%2Fa/sessions`） |
| 加载/attach session | `POST /session/:id/load` |
| 恢复历史 session | `POST /session/:id/resume` |
| 关闭 session | `DELETE /session/:id`；批量 `POST /sessions/delete` |
| client 主动解绑 | `POST /session/:id/detach` |
| client 心跳 | `POST /session/:id/heartbeat` |

### 5.2 Prompt + 控制（非阻塞语义）

| 操作 | 命令 |
|---|---|
| 发 prompt | `POST /session/:id/prompt` `{prompt: ContentBlock[], deadlineMs?}` → `202 {promptId, lastEventId}` |
| 取消进行中的 turn | `POST /session/:id/cancel`（所有订阅者收 `prompt_cancelled`） |
| 切 model | `POST /session/:id/model` `{modelId}` |
| 切 approval mode | `POST /session/:id/approval-mode` `{mode, persist?}`，mode ∈ `plan` / `default` / `auto-edit` / `auto` / `yolo`；`persist: true` 同时写入 workspace settings |
| 会话小结 | `POST /session/:id/recap`（fast model 侧查询，"上次聊到哪了"） |
| 上下文用量 | `GET /session/:id/context-usage`、`GET /session/:id/stats` |

脚本里"发一问、等结果"的完整非阻塞流：

```bash
# 1) 后台订阅 SSE 落盘
curl -sN "http://127.0.0.1:4170/session/$SID/events" \
  -H "Authorization: Bearer $QWEN_SERVER_TOKEN" > /tmp/ev.log &

# 2) 发 prompt，记下 promptId
PROMPT_ID=$(curl -s -X POST "http://127.0.0.1:4170/session/$SID/prompt" \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $QWEN_SERVER_TOKEN" \
  -H "X-Qwen-Client-Id: $CID" \
  -d '{"prompt":[{"type":"text","text":"npm test 失败原因？"}]}' | jq -r .promptId)

# 3) 在事件流里等带同一 promptId 的 turn_complete / turn_error
until grep -E "turn_(complete|error)" /tmp/ev.log | grep -q "$PROMPT_ID"; do sleep 1; done
kill %1
```

`turn_complete` 的 data 为 `{sessionId, stopReason, promptId}`（`stopReason` 缺省 `end_turn`）。配置了 `--prompt-deadline-ms`（或请求带 `deadlineMs`）时，超时 turn 以 `turn_error`（`errorKind: prompt_deadline_exceeded`）收尾。

### 5.3 Permission 投票

agent 调有副作用的 tool 时，所有订阅 client 收到 `permission_request`（带 `requestId` 与可选项 options）；应答：

```bash
# 批准（选一个 option）
curl -s -X POST "http://127.0.0.1:4170/session/$SID/permission/$REQ_ID" \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $QWEN_SERVER_TOKEN" \
  -H "X-Qwen-Client-Id: $CID" \
  -d '{"outcome":"selected","optionId":"allow_once"}'

# 或中止该请求
#   -d '{"outcome":"cancelled"}'
```

赢家产生后广播 `permission_resolved`，迟到的投票收 409 `permission_already_resolved`；`consensus` 下中间票推 `permission_partial_vote`，`designated` / `local-only` 拒绝时推 403 `permission_forbidden`。默认 5 分钟无人应答按 cancelled 解决；strategy 配置见 §2.6。

### 5.4 Workspace 状态、MCP 与热重载

| 操作 | route |
|---|---|
| 读 / 写 memory（QWEN.md 层级） | `GET /workspace/memory` / `POST /workspace/memory` |
| Subagent CRUD | `GET/POST /workspace/agents`、`GET/POST/DELETE /workspace/agents/:agentType` |
| 启/禁某 tool | `POST /workspace/tools/:name/enable`（写 settings 的 `tools.disabled`，下次 child spawn 生效） |
| workspace init（scaffold 空 QWEN.md，**不调 model**） | `POST /workspace/init` `{force?}`（已存在且无 force → 409） |
| MCP 快照（含 pool 计数 / budget 状态） | `GET /workspace/mcp`、单 server 工具列表 `GET /workspace/mcp/:server/tools` |
| MCP server 重启（budget 预检，pool-aware） | `POST /workspace/mcp/:server/restart`（多 entry 时可 `?entryIndex=N` 或 `*`） |
| 运行时增删 MCP server（免重启 daemon） | `POST /workspace/mcp/servers` / `DELETE /workspace/mcp/servers/:name` |
| **配置热重载**（PR#4965） | `POST /workspace/reload` —— 重新加载 `.env` + settings（model / permissions / tools / memory），刷新活动 session；响应含 `changedKeys` / `sessionsRefreshed` |
| 查 skills / providers / hooks / extensions | `GET /workspace/skills` 等同名 GET 路由 |

写类路由要求 bearer + `X-Qwen-Client-Id`；其中 memory 写 / agents 写 / tools toggle / init / reload / 文件写属 **strict** 路由——未配 token 的 loopback 部署下直接 401 `token_required`（配上 token 即可）。

### 5.5 文件读写（sandbox-bounded）

```bash
# 读文件（返回内容 + sha256 hash，作为后续写入的 precondition）
curl -sG "http://127.0.0.1:4170/file" \
  --data-urlencode "path=src/index.ts" \
  -H "Authorization: Bearer $QWEN_SERVER_TOKEN" \
  -H "X-Qwen-Client-Id: $CID"

# 目录/匹配/元数据：GET /list?path=...   GET /glob?pattern=...   GET /stat?path=...
# 二进制窗口读：GET /file/bytes（base64，bounded）

# 写文件（原子写 + content-hash precondition，防写丢并发修改）
curl -s -X POST "http://127.0.0.1:4170/file/write" \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $QWEN_SERVER_TOKEN" \
  -H "X-Qwen-Client-Id: $CID" \
  -d '{"path":"src/index.ts","content":"...","expectedHash":"sha256:..."}'

# 精确单点替换：POST /file/edit {"path","oldText","newText"}
#（oldText 必须恰好匹配 1 处，0 或多处都报错；header 同上）
```

所有文件操作受 workspace 边界 + symlink-escape 拒绝约束（不能经 symlink 读写出 workspace 外）。`/file/write` 与 `/file/edit` 是 strict 路由（见 §5.4 说明）。

### 5.6 OAuth device flow（远端 daemon 给上游 provider 授权）

远端 / headless daemon 没法弹浏览器登录，device flow 让授权动作发生在 client 侧浏览器、凭证落在 daemon host：

```bash
# 1) 启动 device flow
RESP=$(curl -s -X POST "http://127.0.0.1:4170/workspace/auth/device-flow" \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $QWEN_SERVER_TOKEN" \
  -H "X-Qwen-Client-Id: $CID" \
  -d '{"providerId":"qwen"}')
FLOW_ID=$(echo "$RESP" | jq -r .id)
echo "$RESP" | jq '{userCode, verificationUri}'
# 2) 用户在【client 端】浏览器打开 verificationUri 输入 userCode 登录
# 3) 轮询 GET /workspace/auth/device-flow/$FLOW_ID（或订阅 SSE 的
#    auth_device_flow_* 事件）；取消：DELETE 同路径；总览：GET /workspace/auth/status
```

完成后凭证写入 **daemon host** 的 `~/.qwen` 标准凭证缓存，daemon 内随即可用。token 值不回流到 wire / 日志（BrandedSecret redaction：`toString` / `inspect` / `JSON.stringify` 全部输出 `[REDACTED]`）。

---

## 六、配置文件 & 持久化路径

| 路径 | 内容 | 谁写 |
|---|---|---|
| `~/.qwen/settings.json` | 全局 settings（`policy.permissionStrategy` / `policy.consensusQuorum` / `tools.*` / `telemetry.*` 等） | user |
| `<workspace>/.qwen/settings.json` | workspace 级覆盖（daemon boot 与 `POST /workspace/reload` 时读取） | user |
| `<workspace>/QWEN.md` | workspace memory（`/workspace/memory` 与 `/workspace/init` 的对象） | user / daemon |
| `~/.qwen/oauth_creds.json` 等 `~/.qwen` 凭证缓存 | 上游 provider OAuth 凭证（device flow 落地，daemon host 侧） | daemon |
| `~/.qwen/debug/daemon/serve-<pid>-<wsHash>.log` | daemon 文件日志（PR#4559；`latest` symlink 指向最新一份） | daemon |

token 本身没有 daemon 维护的持久化文件——由部署层（env / systemd `EnvironmentFile` / K8s Secret）持有并注入（§2.3）。

---

## 七、诊断 & 故障排查

### 7.1 健康 / 能力检查

```bash
# 浅层健康（仅 listener；loopback 且未开 require-auth 时免认证）
curl -s http://127.0.0.1:4170/health
# {"status":"ok"}

# ?deep=1：附带 bridge 计数器（活 session / pending permission / 限流命中）。
# 这是观测计数，不是 ACP child 活性探测——child 卡死不会改变该响应
curl -s "http://127.0.0.1:4170/health?deep=1" -H "Authorization: Bearer $QWEN_SERVER_TOKEN"
# {"status":"ok","sessions":2,"pendingPermissions":0}

# 能力清单：{v, protocolVersions, qwenCodeVersion, mode: "http-bridge",
#            features: [...], workspaceCwd, policy: {permission: {...}}}
curl -s http://127.0.0.1:4170/capabilities | jq .
```

### 7.2 Preflight / Env 诊断

```bash
# readiness 整体检查：daemon 级 cell（node 版本 / cli 入口 / workspace 目录 /
# ripgrep / git / npm）始终返回；ACP 级 cell（auth / mcp_discovery / skills /
# providers / egress）需要 child 活着，idle 时返回 status: "not_started"
curl -s http://127.0.0.1:4170/workspace/preflight \
  -H "Authorization: Bearer $QWEN_SERVER_TOKEN" | jq '.cells[] | {name, status}'

# daemon host 环境信息：runtime / platform / sandbox / proxy（剥离凭证只留
# host:port）/ 白名单 secret env 的 presence-only（永不回传值）
curl -s http://127.0.0.1:4170/workspace/env \
  -H "Authorization: Bearer $QWEN_SERVER_TOKEN" | jq .
```

### 7.3 日志位置

| 日志 | 位置 |
|---|---|
| daemon stderr | 启动终端 / systemd journal / docker logs（启动横幅、警告、安全提示；ACP child stderr 也转发到这） |
| **daemon 文件日志**（PR#4559） | `~/.qwen/debug/daemon/serve-<pid>-<workspaceHash>.log`，同目录 `latest` symlink 指向最新实例；记录 boot / session spawn / prompt enqueue / turn 完成与失败 / reload。`QWEN_DAEMON_LOG_FILE=0`（或 `false`/`off`/`no`）关闭 |
| OpenTelemetry | 配置 `OTEL_EXPORTER_OTLP_ENDPOINT` 后发 OTel collector |

```bash
tail -f ~/.qwen/debug/daemon/latest
```

### 7.4 常见错误（closed `errorKind` taxonomy）

daemon 错误响应统一带 closed enum `errorKind`（源码：`packages/acp-bridge/src/status.ts` `SERVE_ERROR_KINDS`，14 值），client `switch` 处理：

| `errorKind` | 何时出现 | 应对 |
|---|---|---|
| `missing_binary` | `qwen --acp` child spawn 失败 | `which qwen`，确认安装完整 |
| `blocked_egress` | egress 探测失败（防火墙挡 provider 端点） | 看 preflight `egress` cell，查防火墙 / proxy |
| `auth_env_error` | 必要 auth env 缺失 / 无效 | 查 API key / OAuth 凭证（§5.6） |
| `init_timeout` | ACP child init 超时 | 网络慢 / provider 端点不可达 |
| `protocol_error` | ACP 帧 schema 不匹配 / 损坏 | 查 daemon 日志，提 issue |
| `missing_file` | 路径不存在 / 越 workspace 边界 | 检查 path 是否在 workspace 内 |
| `parse_error` | request body / schema 校验失败 | 检查 JSON 格式与字段名 |
| `stat_failed` | 文件元数据读取失败 | 检查权限 / 文件系统 |
| `budget_exhausted` | MCP server 在 `enforce` 模式被预算拒绝 | 调大 `--mcp-client-budget` 或裁剪 mcpServers |
| `mcp_budget_would_exceed` | MCP restart / add 会超预算，被预检拦下 | 同上；响应里有 `skipped: true` |
| `mcp_server_spawn_failed` | 运行时新增的 MCP server 起不来 | 查该 server 的命令 / env |
| `invalid_config` | MCP server 配置不合法 | 检查 `POST /workspace/mcp/servers` body |
| `prompt_deadline_exceeded` | turn 超过 `--prompt-deadline-ms` / `deadlineMs` | 调大 deadline 或拆小任务 |
| `writer_idle_timeout` | SSE 连接超过 idle 上限被服务端断开 | client 重连（带 `Last-Event-ID`） |

### 7.5 HTTP 状态码常见映射

| HTTP | 何时出现 | 应对 |
|---|---|---|
| `401` | 缺/错 bearer；strict 路由在无 token 部署下返回 `token_required` | 检查 token（注意尾部换行）；strict 路由需配置 token |
| `400 workspace_mismatch` | `POST /session` 的 cwd 与 daemon 绑定 workspace 不一致 | 对 `caps.workspaceCwd` 预检，或干脆省略 `cwd` |
| `403` | permission strategy 拒绝（designated/local-only）；CORS 未放行的 Origin | 检查 clientId / 是否 loopback / `--allow-origin` |
| `404 session_not_found` | session 不存在 | session 已关 / daemon 重启过 / 已被 idle 回收（默认 30 min） |
| `409` | 投票已被解决 / init 已存在 / restore 进行中 / device flow 数量上限 | 并发冲突，按需 retry |
| `415` / `501` | `/acp` Content-Type 不是 JSON / batch JSON-RPC | 修 client 请求 |
| `429` + `Retry-After` | `--rate-limit` 命中分层限流 | 退避重试，或调 tier 上限 |
| `502 upstream_error` | device flow 上游 IdP 故障 | 上游问题，稍后重试 |
| `503` | `--max-sessions` 满 / `/acp` 连接数到顶 / MCP enforce 拒绝 | 等资源释放或调大 cap |

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
EnvironmentFile=/etc/qwen/serve.env
# /etc/qwen/serve.env 内容: QWEN_SERVER_TOKEN=<hex>
ExecStart=/usr/local/bin/qwen serve \
  --hostname 0.0.0.0 --port 4170 --require-auth \
  --workspace /var/qwen/workspace \
  --max-sessions 50 --rate-limit \
  --prompt-deadline-ms 600000 --writer-idle-timeout-ms 120000
Restart=on-failure
RestartSec=5s
MemoryMax=4G

[Install]
WantedBy=multi-user.target
```

### 8.2 Docker

```dockerfile
FROM node:24-bookworm
RUN npm install -g @qwen-code/qwen-code@preview && useradd -ms /bin/bash qwen
USER qwen
WORKDIR /workspace
EXPOSE 4170
CMD ["qwen", "serve", "--hostname", "0.0.0.0", "--port", "4170", \
     "--require-auth", "--workspace", "/workspace", "--rate-limit"]
```

```bash
docker run -d -e QWEN_SERVER_TOKEN=$(openssl rand -hex 32) \
  -v /path/to/repo:/workspace -p 4170:4170 qwen-code-daemon
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
        image: qwen-code-daemon
        args: [serve, --hostname=0.0.0.0, --require-auth,
               --workspace=/workspace, --max-sessions=20, --rate-limit]
        env:
        - name: QWEN_SERVER_TOKEN
          valueFrom: {secretKeyRef: {name: qwen-token, key: token}}
        ports: [{containerPort: 4170}]
        volumeMounts: [{name: workspace, mountPath: /workspace}]
        resources:
          limits: {memory: 4Gi, cpu: 2}     # cgroup = per-tenant quota
        livenessProbe:
          httpGet:
            path: /health
            port: 4170
            # --require-auth 下 /health 也要 bearer，探针必须带
            httpHeaders: [{name: Authorization, value: "Bearer $TOKEN"}]
          initialDelaySeconds: 10
  volumeClaimTemplates:
  - metadata: {name: workspace}
    spec: {accessModes: [ReadWriteOnce], resources: {requests: {storage: 50Gi}}}
```

**多 tenant = 多 daemon 进程 / pod**（OS 进程级天然隔离），不要在一个 daemon 内做应用层 tenant 抽象——同 daemon 的 N 个 session 共享同一个 ACP child（同 UID、同 fs 视图），**不能让互不相信的 tenant 共享一个 daemon**。`MemoryMax=` / cgroup / `resources.limits` 直接就是 per-tenant 配额。

### 8.4 Reverse proxy / TLS（生产必备）

`qwen serve` 不做 TLS。生产在前面套 nginx / Envoy / Ingress：

```nginx
server {
  listen 443 ssl http2;
  server_name daemon.example.com;

  # web-shell 静态资源（如部署）与 daemon API 收敛到同一 origin，免 --allow-origin
  location / { root /srv/web-shell; try_files $uri /index.html; }

  location ~ ^/(session|sessions|workspace|file|list|glob|stat|capabilities|health|acp|demo) {
    proxy_pass http://127.0.0.1:4170;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header Authorization $http_authorization;
    proxy_set_header X-Qwen-Client-Id $http_x_qwen_client_id;
    proxy_set_header Upgrade $http_upgrade;          # /acp WebSocket
    proxy_set_header Connection $connection_upgrade;
    proxy_buffering off;                  # SSE 不缓冲
    proxy_read_timeout 1d;                # 长连接
  }
}
```

注意：`local-only` permission strategy 看 kernel-stamped remote IP、**不看 `X-Forwarded-For`**——反向代理后 daemon 看到的来源永远是 proxy 的 loopback 地址，**`local-only` 在反向代理后失效**（设计如此，防伪造 header 绕过）。需要"仅内网可审批"语义时在 proxy 层做 IP allowlist + daemon 用 `first-responder`。

### 8.5 egress 与凭据都在 daemon host

daemon host = runtime owner，所有出网（provider / MCP / skill / shell）从 daemon host 发起——网络策略按 **deny-by-default + 显式 allowlist**（只放行 provider 端点 / 远程 MCP / skill 依赖的外部 API），诊断走 `/workspace/preflight` 的 `egress` cell（失败映射 `blocked_egress`）。

凭据同理：provider OAuth token 在 daemon host 的 `~/.qwen` 凭证缓存（§5.6）、API key 从 K8s Secret / secret manager 注入 daemon 进程 env、MCP credentials 引用 daemon host env。**client 端的凭据（SSH agent / kubeconfig / API key）不会自动传给 daemon**——daemon host 必须自己持有；多 tenant 场景 1 daemon = 1 tenant 时凭据隔离最干净。

---

## 九、与 native TUI 的关系

**`qwen` 命令仍直连本地 runtime，不走 daemon**——这是长期保留的本地 TUI 路径（零网络 / 最低复杂度）。`qwen serve` 是给 web / remote runtime / 多 client 协作场景的，不是 TUI 替代。**不要为不需要的东西付费**：

| 场景 | 推荐 |
|---|---|
| 本机单人开发，不需要多 tab / web UI | **`qwen`**（native TUI，直连，无 daemon 开销） |
| 想要图形界面、单机使用 | 官方桌面 app（§4.2，自带 runtime，无需手动起 daemon） |
| 本机多 tab / web UI / 多 client 共享 session | `qwen serve` + web-shell / TS SDK |
| 远端 devbox / Codespaces / K8s pod | `qwen serve --hostname 0.0.0.0 --require-auth` |
| ACP-native client（Zed / Goose） | `qwen serve` + `/acp`（或让 client 自己 spawn `qwen --acp`） |
| MCP client（Qoder / Claude Desktop / Cursor） | `qwen serve` + `qwen-serve-mcp` 桥 |
| 嵌入式自动化（Python / Java / TS 单进程） | 对应 SDK 直接 spawn CLI，**不需要 daemon** |

历史设计取舍（为什么是 headless daemon 而非 TUI 同进程 `--serve`）见 [02-architectural-decisions.md](./02-architectural-decisions.md) 与 [06-roadmap.md](./06-roadmap.md)。

---

## 十、外部参考

| 资源 | 用途 |
|---|---|
| [QwenLM/qwen-code](https://github.com/QwenLM/qwen-code) | 主仓；daemon 源码 `packages/cli/src/serve/`，官方用户文档 `docs/users/qwen-serve.md` |
| PR#4490；PR#4585 / #4527 / #4530 / #4861 / #4559 / #4965 / #4555 / #3778 | daemon 合入 main 的 merge PR；非阻塞 prompt / allow-origin / 双超时 / 限流 / 文件日志 / 热重载 / MCP 桥 / 桌面 app |
| [RFC 6750](https://datatracker.ietf.org/doc/html/rfc6750) / [RFC 8628](https://datatracker.ietf.org/doc/html/rfc8628) | Bearer token 认证 / OAuth 2.0 Device Authorization Grant |
| [RFD #721](https://github.com/agentclientprotocol/agent-client-protocol/pull/721)、[Agent Client Protocol](https://github.com/agentclientprotocol/agent-client-protocol) | `/acp` 端点遵循的 ACP Streamable HTTP 草案与协议主仓 |

---

## 附录 A：capability tag 速查

`GET /capabilities` 的 `features` 数组（源码：`packages/cli/src/serve/capabilities.ts` `SERVE_CAPABILITY_REGISTRY`）。client 用 `caps.features.includes(...)` 协商；标 **（条件）** 的 tag 只在对应配置开启时公布——**tag 在场 = 行为已开启**。

```
# 基础面
health / capabilities / typed_event_schema / client_identity / client_heartbeat

# Session 生命周期与控制
session_create / session_scope_override / session_load / unstable_session_resume /
session_list / session_close / session_metadata / session_prompt / session_cancel /
non_blocking_prompt / session_events / slow_client_warning / session_set_model /
session_approval_mode_control / session_language / session_rewind / session_branch /
session_recap / session_btw / session_context / session_context_usage /
session_supported_commands / session_tasks / session_stats / session_hooks

# Permission
session_permission_vote / permission_vote /
permission_mediation   # modes: first-responder / designated / consensus / local-only

# Workspace 控制面与文件面
workspace_memory / workspace_agents / workspace_agent_generate /
workspace_tool_toggle / workspace_init / workspace_env / workspace_preflight /
workspace_skills / workspace_providers / workspace_hooks / workspace_extensions /
workspace_settings（条件）/ workspace_reload（条件）/
workspace_file_read / workspace_file_bytes / workspace_file_write

# MCP
workspace_mcp / workspace_mcp_manage / workspace_mcp_restart /
mcp_guardrails（modes: warn / enforce）/ mcp_guardrail_events /
mcp_server_runtime_mutation /
mcp_workspace_pool（条件：未设 QWEN_SERVE_NO_MCP_POOL=1）/ mcp_pool_restart（条件：同左）

# Auth / 部署姿态
auth_device_flow / auth_provider_install /
require_auth（条件：--require-auth）/ allow_origin（条件：--allow-origin）/
prompt_absolute_deadline（条件：--prompt-deadline-ms）/
writer_idle_timeout（条件：--writer-idle-timeout-ms）/ rate_limit（条件：--rate-limit）
```

`capabilities.policy.permission` 同时暴露 active strategy：

```jsonc
{"kind": "first-responder"}
{"kind": "designated", "clientId": "..."}
{"kind": "consensus", "minVotes": 3}
{"kind": "local-only"}
```

## 附录 B：典型 troubleshooting decision tree

```
症状 → 先查 → 再查
═══════════════════════════════════════════════════════════════
daemon 启动失败          → stderr 报错文案 → 非 loopback / --require-auth / --allow-origin '*'
                                            而没配 token？
401 Unauthorized         → bearer 是否带/值对（注意尾部换行）→ require-auth 下 /health /demo 也要 token
401 token_required       → 碰到 strict 路由（文件写/memory 写/device-flow/reload）
                           → 无 token 的 loopback 不放行这些，配 QWEN_SERVER_TOKEN 即可
400 workspace_mismatch   → daemon 绑定的 workspace → POST /session 干脆省略 cwd
403                      → designated: clientId 是否 originator
                         → local-only: 是否 loopback（反向代理后必 403）
                         → 浏览器跨域: --allow-origin 是否含该 origin
404 session_not_found    → daemon 是否重启过 → session 被 DELETE / idle 回收（默认 30min）？
429                      → --rate-limit 命中 → 按 Retry-After 退避（prompt 默认 10/min）
SSE 流断了               → Last-Event-ID 重连 → state_resync_required → POST .../load 重建
                         → 频繁断？查 --writer-idle-timeout-ms 与代理 buffering
prompt 202 后没下文      → 是否订阅了 SSE（202 不带输出）→ turn_error 的 errorKind？
                         → /workspace/preflight: blocked_egress / auth_env_error / init_timeout
                         → prompt_deadline_exceeded？调大 deadline
permission_request 卡住  → strategy 是否要 N 票 → 默认 5 分钟超时按 cancelled 解决
MCP server 不见          → GET /workspace/mcp → enforce 被预算拒（disabledReason: budget）？
                         → HTTP/SSE transport 没进 pool？查 QWEN_SERVE_MCP_POOL_TRANSPORTS
文件写失败               → 是否在 workspace 内 → symlink escape？expectedHash 过期（重新 GET /file）？
ACP client 接不上        → QWEN_SERVE_ACP_HTTP 被设 0？→ 是否带 Acp-Connection-Id
                         → WebSocket: Origin / Host 检查拦了（CSWSH 防御）？
改了 settings 不生效     → policy.* 需重启 daemon → 其余试 POST /workspace/reload（strict，需 token）
```

---

> **免责声明**：本文基于 qwen-code main 分支源码（v0.18.0，`packages/cli/src/serve/`）整理，截至 **2026-06-13**。HTTP API / flag / 事件 schema 可能在后续版本变化；以运行时 `GET /capabilities` 的 feature tag 协商为最准确的能力检测方式（附录 A）。
