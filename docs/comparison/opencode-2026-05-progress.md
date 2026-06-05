# OpenCode 2026 年 5 月进展深度分析

> 对 OpenCode（sst/opencode）2026 年 5 月（`origin/dev` 分支，1567 commit）的逐主题分析。基于本地源码 + commit diff，6 个调查 agent 分面深挖后综合。
>
> 与本仓库其它进展跟踪并列：[四方周报](./weekly/2026-W22-0524-0531.md) · [Qwen Code 月活贡献者](./contributors/2026-05.md)。穿插与 **Qwen Code** 的设计对比（本仓库主对象）。

## TL;DR

OpenCode 5 月**三线并行全速推进，无单一焦点**：① **Effect-TS 全栈架构重写**（最深，但低调）② **产品全栈铺开**（ACP 完整重写 + Desktop v2 + 双前端 + "一切皆 plugin"）③ **商业化基建**（Zen 模型网关 + stats 排行榜营销站 + referral 裂变）。节奏靠**高频微 commit + 机器人自动生成 + 每日滚动 patch 发布**（v1.14.31 → v1.15.13，33 次发版）。**"月产 1567 commit"约 27% 是自动化水分，真实 agent 核心产出 ~430 commit。**

最值得关注的两个战略选择：**Effect-TS 语言级架构押注**，以及与 Qwen 相反的 **"1 server 托多 workspace"** 路线。

## 一、数据校正：1567 commit 的"水分"

| 类别 | commit | 占比 | 性质 |
|---|---:|---:|---|
| `chore: generate`（bot 重新生成代码/migration/snapshot）| 323 | 20.6% | 机器人 |
| `chore: update nix node_modules` | 63 | 4.0% | 机器人 |
| `sync release versions` | 33 | 2.1% | 发版号同步 |
| **机器/版本噪声小计** | **419** | **26.7%** | —— |
| 商业化运营（stats + zen + referral）| ~86 | ~5.5% | 非 agent 核心 |
| `feat:` + `fix:` 真实产品 | 461 | 29.4% | 含 stats，去后 agent 核心 **~430** |

> `opencode-agent[bot]` 一个机器人提了 **388 commit ≈ 头号人类作者（Kit Langton 435）的量**。剔噪后实际人工核心产出 ~430。走**高频微 commit**（大量单行 `fix(stats): tighten 2m chart gaps`）+ **每日滚动 patch 发布**（5 月 33 次发版、21 天有发版、05-15 切 v1.15.0，无 major）。

## 二、🏛️ 最深的一条线：Effect-TS 架构迁移

5 月**架构第一主线**，被前端/商业化的热闹盖过，但深度最高：

- **规模**：`packages/core/src` **208/281 文件（~74%）已 `import "effect"`**，核心域服务（session/provider/config/agent/lsp/workspace/git…）全是 Effect `Service`/`Layer`
- **有正式治理**：`packages/opencode/specs/effect/todo.md` 是 Kit Langton 维护的 **P0–P6 路线图**（typed errors / CLI·HTTP·tool 边界渲染 / 删 HTTP 中间件按 error name 猜状态码 / RuntimeFlags / 删 legacy 兜底）
- **5 月性质从"迁移"→"边界收口"**：裸 `spawn` 迁 `AppProcess.run`（#27178）、`AppFileSystem.Service`（#27152/#27155）；`NamedError`→`Schema.TaggedErrorClass`；**删除 legacy ALS/Instance 兜底**（`retire WithInstance adapter` #27782 / `remove legacy runtime fallback` #27757）；32 个 `test(server)` 迁 Effect 原生 fixture（`it.instance`/`testEffect`）
- **新基建**：`packages/effect-drizzle-sqlite`（drizzle 包进 Effect SQL 层 `@effect/sql-sqlite-bun`，#28547）+ `@effect/cli` scaffold（连命令解析层也要 Effect 化）
- **可量化长尾**：仍有 26 文件用 `NamedError`、44 文件 import `flag/flag`（P2 目标删 `flag.ts`）→ 主体完成、长尾未清

> **对比 Qwen Code / Claude Code**：两者都无此类全栈框架迁移。OpenCode 把整个代码库重写到一个函数式效果系统上，是它最独特的工程选择，也是其高 `test`/`refactor` commit 占比（17.4%）的根源。

## 三、🔌 最干净的工程线：ACP 完整重写并硬切

`acp-next` **2026-05-25 → 05-30** 一周完成，Shoubhit Dash 主导，**#29929 promote 转正**：

| 阶段 | 关键 PR | 内容 |
|---|---|---|
| 基线 | #29222 | 用旧实现固化协议行为作 parity 标尺 |
| 骨架 | #29226 | `acp-next/` 空壳 + `OPENCODE_ACP_NEXT` flag 灰度 |
| 纯函数切片 | #29231/#29232/#29233 | content/tool 转换 + typed error → RequestError 映射 |
| service 切片 | #29240/#29241/#29249/#29250 | session state / directory snapshot / usage / session slice |
| 生命周期 | #29320/#29327/#29333 | close/fork/resume/list + event routing + tool 流式 |
| 权限+prompt | #29656/#29664 | permission events + prompt（斜杠命令分流）|
| 优化 | #29709/#29713 | 首会话启动（纯 config 解析）+ 热切换（snapshot 缓存）|
| **晋升** | **#29929** | `git mv` acp-next→acp，删旧单体，**净 -3718 行，硬切不灰度** |

- **旧** 1947 行单体 `Agent` class（事件/权限/用量/会话全耦合，几乎无法单测）→ **新** `service.ts`(1062) + 7 个职责 service（session/directory/usage/event/permission/content/tool），全 Effect 依赖注入，每个可单测
- **定位**：OpenCode ACP 是**被编辑器连的 agent/server 侧**，走 **stdio JSON-RPC 子进程**（Zed/JetBrains fork `opencode acp`，无网络）

> **对比 Qwen Code**：两者同期都在 ACP 上重投入（再次印证 ACP 正成多方共识），但 **OpenCode 走 stdio 子进程（本地无网络），Qwen 走 HTTP transport（远端/多 client）**。OpenCode 的 stdio 模型契合"本地集成不走网络"；Qwen 的 HTTP 面向 daemon 多 client。

## 四、🗂️ 战略级架构概念：location layer（多 workspace 服务隔离）

Dax 主导，5 月底落地：
- **`LocationServiceMap`**（`packages/core/src/location-layer.ts`）= 基于 Effect `LayerMap` 的 **"目录 → 服务子图"带 TTL 缓存/对象池**。给一个 `{directory, workspaceID}` lookup 出该目录专属 Layer（Catalog+Config+AgentV2），空闲回收；`Layer.fresh()` 保证每 location 全新隔离实例（#29625/#0269d6f5）
- **location-scoped config**（#29625，碰 ~50 文件）：不同目录读各自 `.opencode` 配置而非单一全局
- 配套 **server SDK 全局化 + 多 server 连接去重**（#29285/#29313）→ 整体朝 **"一 app 连多 server / 一 server 托多目录"** 拓扑走

> **与 Qwen Code 的设计正面对立（很有意思）**：Qwen 核心决策是 **"1 daemon = 1 workspace"（OS 进程级隔离，blast radius 最小）**；OpenCode 反过来做 **"1 server 进程托管多 workspace"（进程内 Effect 服务隔离）**。两条路——Qwen 靠 OS 进程边界，OpenCode 靠语言级效果系统边界。各有取舍：Qwen 隔离更硬但要多进程编排；OpenCode 单进程更省但隔离靠框架正确性。

## 五、🖥️ 前端：全栈 SolidJS + Desktop v2 + 双前端并重

- **三端共用一套 SolidJS UI**：Desktop（Electron renderer）与 Web 共享 `@opencode-ai/app`；**连 TUI 都是 `@opentui/solid`**（SolidJS-on-terminal，非 Ink）。技术栈高度统一
- **"Desktop v2" 实质是 `app` 包的新设计语言**（home/titlebar/tabs/session controls），不是 Electron 外壳重写；05-20 起一次铺开整套 `ui/src/v2/` 组件（带 Storybook），`newLayoutDesigns()` flag 灰度，v1/v2 共存未默认全量（关键 commit `38b406fb3` tabs 起点 / `b207e3224` v2 home+titlebar）
- **Electron 外壳走工程加固**：server 迁 `utilityProcess`（`54a78c922`）、main 进程 Effect-TS 化（`21ae91b4f`）、`desktop-electron` 包合并、全平台安装器/证书/代理/更新
- **TUI 富交互补齐**：diff viewer（`17d66ee4f`，05-22 默认开）、session switcher 带预览面板（`8f8b161ca`）、workspace dialog（`28a06e52f`）、thinking 折叠、subagent 改按需 picker（`ed839846d`）

## 六、🧩 "一切皆 plugin" 的 v2 架构

- **默认 agent 改 plugin 注入**（`f6062bd5b`）：7 个内置 agent（build/plan/general/explore + compaction/title/summary 三 hidden）通过 `PluginV2` + immer editor 写入 `AgentV2` registry；**用户自定义 agent 走同一管线**（`config-agent` plugin）
- 每 agent 带**声明式细粒度 permission ruleset**（`{permission, pattern, allow|ask|deny}`，如 `read:{"*.env":"ask"}` / `external_directory:{"*":"ask"}` / `doom_loop:"ask"`）
- **新扩展点**：plugin **dispose hook**（`519d34447`，scope 关闭时清理）/ session **metadata**（#23068）/ **location-scoped config**
- **flags 大重构**（~24 commit）：编译期全局 `Flag.*` 常量 → 运行时注入的 `RuntimeFlags` 服务（可 per-instance 传入）；细粒度实验 flag 可覆盖 `OPENCODE_EXPERIMENTAL` 总开关
- **TUI 也插件化**：session switcher 是 `internal:session-switcher` feature-plugin（与 Notifications/WhichKey/DiffViewer 并列）

## 七、💰 商业化三件套（清晰的变现信号）

- **Zen** = OpenCode 自营 **AI 网关/模型转售**（`opencode.ai/zen/v1`，config 用 `opencode/<model-id>`）：挑 benchmark 过的 coding 模型，pay-as-you-go（Opus 4.7 输入 $5/输出 $25），余额 <$5 自动 reload $20，team workspace + 成员限额。5 月密集上新（opus 4.8/gemini 3.5 flash/grok/deepseek）+ 计费/限流（redis/upstash rate limit）
- **Go** = 低价订阅档（首月 $5/后 $10/月，开源模型，US/EU/SG 多区，源码里 "Lite"）
- **stats.opencode.ai** = 公开**模型用量排行榜/市场份额**营销站（#28666 从零搭建）：自建 **S3 Tables(Iceberg) + AWS Athena datalake**，按 All Users/Zen/Go/Enterprise 分产品 + 按厂商市场份额 + 地理分布
- **referral** = $5 邀请返利，首次 Go checkout 归因（防刷）

> **对比 Qwen Code**：OpenCode 积极**把开源工具变现**（网关 + 营销站 + 裂变三件套）；Qwen Code 阿里背书、无商业化动作。开源独立项目 vs 大厂项目的根本路线差异。

## 八、其它要点

- **Provider/LLM**：Opus 4.7+ adaptive reasoning 铺到 vertex/gateway/sap-ai-core 四通道；OpenAI **Responses WebSocket transport**（#29477 + 5 加固）；**native LLM runtime**（实验，把 LLM 调用从外部 SDK 迁进程内 `OPENCODE_EXPERIMENTAL_NATIVE_LLM` — 与"减少外部依赖"一致）；新接 xAI Grok OAuth / DigitalOcean / NVIDIA
- **Worktree**：managed **CoW 完整克隆** Rust crate（macOS `clonefile` / Linux `FICLONE` reflink，含 ignored 文件，DB 维护 parent 树），但 **05-31 同一天又移出主仓库**走外部二进制路线
- **http-recorder**：VCR/cassette 式 HTTP+WS 录制回放测试库（专为 Effect `HttpClient`），写盘前强 secret 扫描（命中拒写 + `UnsafeCassetteError`）

## 九、与 Qwen Code 的设计对比小结

| 维度 | OpenCode | Qwen Code |
|---|---|---|
| 架构底座 | **Effect-TS 全栈重写**（~74% core）| 无框架迁移，常规 TS |
| 多 workspace | **1 server 托多 workspace**（location layer 进程内隔离）| **1 daemon = 1 workspace**（OS 进程隔离）|
| ACP | server 侧，**stdio 子进程**（本地无网络）| server 侧，**HTTP transport**（远端多 client）|
| 前端 | 全栈 SolidJS（Desktop+Web+TUI 共用）| Ink TUI + React web-shell（daemon-backed）|
| 商业化 | **Zen 网关 + stats 站 + referral** | 无（阿里背书）|
| 发布节奏 | 每日滚动 patch（月走完一 minor）| 阶段性版本（v0.16.x）|
| agent 扩展 | 一切皆 plugin（AgentV2 registry + immer editor）| skill + subagent + auto-dream |

## 源码路径参考

| 主题 | 路径 |
|---|---|
| Effect 路线图 | `packages/opencode/specs/effect/todo.md` |
| location layer | `packages/core/src/location-layer.ts` / `location.ts` |
| ACP（promote 后）| `packages/opencode/src/acp/{service,event,permission,session,directory,usage}.ts` |
| 默认 agent plugin | `packages/core/src/plugin/agent.ts` |
| Zen / Go 文档 | `packages/web/src/content/docs/{zen,go}.mdx` |
| stats 站 + datalake | `packages/stats/{app,core,function}/` + `infra/{stats,lake}.ts` |
| worktree（已移出）| `packages/worktree`（5-31 删，走外部二进制）|
| http-recorder | `packages/http-recorder/` |

---

> **免责声明**：基于 `origin/dev` 2026-05-01 ~ 06-01 区间 `git log`（1567 commit）+ 关键 commit diff 分析，截至 fetch 时点。PR/commit 号取自 commit 标题。噪声分类按 conventional-commit 前缀 + 关键词匹配，存在少量跨类重叠。版本号未打 git tag，发版次数以 `sync release versions` commit 计。与 Qwen Code 的对比基于本仓库已有 daemon/telemetry 分析。
