# OpenAI Codex 2026 年 5 月进展深度分析

> 对 OpenAI Codex（openai/codex，Rust `codex-rs` + TS）2026 年 5 月（**960 merged PR**）的逐主题分析。基于公开 GitHub PR 数据，6 个调查 agent 分面深挖后综合。
>
> 与本仓库其它进展跟踪并列：[OpenCode 5 月进展](./opencode-2026-05-progress.md) · [四方周报](./weekly/2026-W22-0524-0531.md)。穿插与 **Qwen Code** 的设计对比。

## TL;DR

Codex 5 月的投入**压倒性指向"企业级、可自托管、可治理的自主 agent 平台"**，三条主线：① **平台化三层**（extension 内部架构 / plugin 企业生态 + marketplace 治理 / skill 内容单元）② **三平面 server 架构**（app-server 控制 / exec-server 执行 / mcp 互操作，为远程/云编排设计）③ **"长时程自主 + 每步可治理"**（goal 自动续跑 + Guardian 逐动作审批 + hook 全栈挂点）。外加 **Windows 企业受管部署是第一平台优先级**、**Bedrock/GovCloud 多云 provider**、**Python SDK 独立 beta**。

节奏上：**960 PR ≈ 96% OpenAI 内部、~0% 机器人、头号作者 jif-oai 一人占 ~20%**——这是"在开源仓库里开发的公司内部项目"，与 OpenCode（社区 + 27% 机器人）、Qwen（混合团队）形态都不同。

## 一、数据校正：960 PR 的真实成色

| 指标 | 值 | 说明 |
|---|---|---|
| 5 月 merged PR | **960** | 是 Qwen（~400）的 2.4×、OpenCode commit（1567）的 ~60% |
| 机器人 PR | **~0%** | 几乎无 bot/dependabot（vs OpenCode 27% 机器人 commit）|
| OpenAI 内部占比 | **~96%**（`-oai`/`-openai` 账号 85% + 其它内部）| 社区外部贡献 ≤~4% |
| 头号作者 | jif-oai **189 PR（~20%）** | Top 5 占 ~50%，Top 15 占 ~76% |
| 微 PR 工作流 | `[1/8]…[8/8]` 拆分系列 + `[codex]` 前缀 136 PR（~14%）+ debug-client **7 连删纯 `+0/-X`** | AI 辅助、原子化、易 review/revert/bisect |

> **解读**：960 PR 这个量级 + 极细粒度（一文件一删、`[1/8]` 切片、14% `[codex]` 标记）= **高速 AI 辅助 trunk 工作流 + 强制小 diff 卫生**。很可能 Codex 自己起草了大量机械 PR。**有效人/agent 主导吞吐三者最高**（~0% 机器人水分），但也是**最集中**（一人 1/5、一组织 96%）。

**发版**：Rust CLI 快车——5 月 **7 个 stable `rust-vX`**（v0.130→v0.135，~3-4 天一个 minor）+ 近每日 alpha；**Python SDK 被刻意拉出主车，独立版本线**。

## 二、🧩 平台化三层：extension / plugin / skill

这三个词在 Codex 里**不是同义词，是三个抽象层**：

```
Plugin（用户安装/分享/治理的分发包，跑在 marketplace + 远程同步 + 企业治理体系）
  ├ 包含 → Skill（文件系统提示/能力片段，turn 内注入）
  └ 包含 → Hooks（默认开）
Extension（Codex 内部第一方 typed contributor 骨架，进程内）
  └ host 侧"承接平面"：plugin 工具 / 托管能力本地化都经 extension 的 ToolBundle 接入 core
```

- **Extension = 内部架构层**（给 OpenAI 自己工程师）：5/11 落地 typed extension API（`ext/extension-api/`，contributor 模型 + host-owned state），5 月把 git 署名 / guardian / memory / goal / **standalone websearch（#23823）** / **standalone image gen（#24723）** 一批核心能力**重构成独立 extension crate**，并把"OpenAI 后端托管的工具"迁成"客户端进程内可独立执行"（eligible 时替换 hosted，否则回退）
- **Plugin = 企业生态层**（5 月最成熟，50 PR）：marketplace 升级 / CLI / **企业级 share 治理**（role-aware / 版本 / checkout / admin-disabled / `plugin_sharing` gate）/ **模型驱动安装**（`list_available_plugins_to_install` + `request_plugin_install`，按已装 app connector 过滤 #24996）/ 远程 plugin runtime enablement（#24707）/ plugin hooks 默认开
- **Skill = 内容单元层**（9 PR，偏收敛）：搬到 app-server 管理；运行时非持久 extra roots（#24977，服务 IDE/远程动态注入）
- **标准对齐**：`ui.visibility` meta（#24700）直接采用 **MCP ext-apps 规范**

> **战略意图**：Codex 在**双轨平台化**——对外建带 marketplace + 企业分发治理 + 模型驱动安装的**插件生态**（把 Codex 推成企业工作流入口，绑定 SaaS connector）；对内用第一方 extension 把 core 拆干净 + 托管能力本地化。这是从"一个 CLI agent"→"agent 平台"的**形态转变**。

## 三、🖧 三平面 server 架构

Codex 5 月固化为**三类独立 server 进程/crate**，每个专属 transport + auth：

| Server | crate | 定位 | transport |
|---|---|---|---|
| **app-server** | `codex-app-server` | **控制平面 = 唯一状态权威**（thread/settings/skills/memory 全收归），TUI 退化为它的客户端 | stdio:// / unix:// 本地；ws+auth 远程 |
| **exec-server** | `codex-exec-server` | **执行平面**（沙箱命令 + 文件系统 RPC），供 **Codex Cloud（CaaS）远程编排器**驱动 | ws(HTTP upgrade) + `/readyz` |
| **mcp** | `codex-mcp-server` | **互操作平面**：既是 client 消费外部 MCP，又是 server 暴露自身 | per-server environment 隔离 |

- **本地优先 + 远程强认证**：本地一律 stdio/unix（零网络零认证）；任何离开 loopback（app-server 非 loopback ws / remote_control / exec-server cloud）**强制认证**；两个本地 ws server 统一**拒绝带 Origin 头的浏览器请求（403）**
- **认证去用户化**：远程从"用户 ChatGPT token"迁到**短期 server token（内存不落盘）/ Agent Identity / capability-token**——为长驻 daemon + 容器 agent 设计
- **daemon 生命周期**：`codex-app-server-daemon`（start/restart/stop/bootstrap/hourly-updater，SSH 远程保活，graceful SIGHUP drain）
- **thread vs session 解耦**：thread = 用户会话（SQLite + rollout JSONL 双真相）；session = 运行时身份，**一个 session 可跨多 thread**（subagent spawn 各自 thread 但共享 root session，hook 能跨关联）

> **与 Qwen daemon 对比**：两者都走"长驻 server + 外部前端驱动 + thread idle lifecycle + 本地 UX 最高优先级"（Codex 本地 stdio/unix 零网络 ≈ Qwen 本地 TUI in-process 不走网络）。**最大差异是分层粒度**：Codex 把执行/控制/互操作拆成**三个独立 server crate**，各有传输和认证，并为 SSH 远程/容器 agent 设计了完整 daemon 生命周期 + server-token 体系；Qwen daemon 更偏**单进程统一承载 + ACP/REST parity 双协议**。Codex 远程认证成熟度 + 分离度更高。

## 四、🤖 长时程自主 + 每步可治理：goal + Guardian + hook

5 月把"会自己长时间干活的 agent"和"管得住它每一步的护栏"**同步推进、统一收敛到扩展化架构**：

### goal（自主性轴，33 PR，改动最深）
用户 `/goal <objective>` 设目标绑定持久 thread，Codex **自动合成续跑 turn 不断推进直到完成**：
- `#23732` 标 Stable **默认开启**；独立 `goals_1.sqlite` DB + 迁到 `ext/goal/`
- **删"过早停"启发式**（#20523）：删掉"觉得卡住就等输入"+"续跑没调工具就停"——goal 真的跑到完成
- **steering 统一**（#24918）：旧 `<goal_context>` → 通用 `InternalModelContextFragment`（`<codex_internal_context source="goal">`，不进可见 history）；活跃 turn 走 `inject_if_running`（#24924）
- **绑 thread 生命周期**：新增 `on_thread_idle` 钩子（#24744）作续跑锚点；`#24925` 按 thread eligibility gate goal 工具（ephemeral/review subagent 隐藏）
- **优雅退出**：resumable `blocked`/`usageLimited` 终态（#23094），防限流时空烧 token

### Guardian（治理轴，13 PR，进入生产硬化）
逐动作 LLM 审批守门员（详见 [四方周报对 Guardian 的澄清](./weekly/2026-W22-0524-0531.md)）。5 月**审批语义已冻结、做运维硬化**：
- **prompt cache key 稳定化省 token**（#24803 + 7 拆分 PR）：fork 间复用缓存
- **OTel review metrics**（#24897）：count/duration/TTFT/token + 全维度 tag，**为灰度放量服务**
- 多 backend（Bedrock reviewer override）+ 只读沙箱 + MCP elicitation 纳入审批

### hook（可观测/可拦截轴，36 PR）
挂点从根 agent 铺到 **subagent + 每工具 + 每网络请求**：
- schema 收紧（#24962，每 event 锁字面量 `hookEventName`）+ 多行渲染（#24965）
- SubagentStart/Stop hook；**内部系统 subagent（Review/Compact/Memory）不跑 hook**
- 工具 hook 通用化（#23757 默认契约）；**网络 MITM hook**（hooked HTTPS host CONNECT 后评估/改 header/block）

> **合起来 = "长时程自主 agent，但每一步可治理"**：自主性越强（goal），逐动作审批越关键（Guardian），全生命周期挂点越必要（hook）。三者都在往 `ext/*` 迁、收敛成可复用原语——既要更强自主，也要同等强的可观测可控。

## 五、🪟 Windows 企业部署 + Bedrock + 沙箱安全

**Windows 是 5 月第一平台优先级（42 PR）**——重点不是"能跑"，而是**让企业 IT 在无本地管理员权限、有 UAC/防火墙/ACL 的环境批量安全托管部署**：
- `#24831` `codex sandbox setup --elevated`（IT 预配置，无需用户提权）+ `#23766` managed requirements 强制 elevated + `#20708` readiness RPC（避免无解释 UAC 弹窗）
- 双后端（elevated / unelevated restricted-token）；**unelevated 对 deny-read fail-closed**（#18202 诚实安全决策）；权限从 `SandboxPolicy` 迁到 resolved permission profiles

**Bedrock（9 PR）= 进客户已有云合规边界**：SigV4/Mantle 客户端，`#24701` GPT-5.5 上 Bedrock 设默认；移除 OSS 模型 + 锁 default service tier（企业 catalog 治理）；GovCloud `#23860` 加了又 `#24690` revert（尝试中）

**沙箱安全朝 fail-closed/不可绕过/词汇可审计**：deny-read 不被 safe command 绕过（#23943）；Unix socket 权限 `none`→`deny` 统一（#24970）；Origin 403（#24947）

**多模态（image）= 能力补全非重心**：standalone image gen extension（#24723，默认关、固定 `gpt-image-2`、Bedrock 上禁用）

## 六、🐍 Python SDK 独立 beta + TUI

**Python SDK 产品化（一个月内完成）**：`[1/8]…[8/8]` 地基系列（05-11）→ packaged runtime + login（#23786/#23093）→ **rename `AppServerConfig`→`CodexConfig` + Codex names 全表面去内部品牌化**（#24800/#24804）→ **独立 beta 发布**（#24828，`openai-codex` 包，从 tag staged，脱离 monorepo 发版车）。是 Codex 第一个独立 ship 的客户端表面。

**TUI（Rust，96 PR）**：唯一**原生编译**TUI（**ratatui 0.29 即时模式 + crossterm + syntect 语法高亮 + pulldown-cmark**），5 月重点：响应式 Markdown 表格 + OSC 8 超链接、**Vim 模态编辑**（`ciw`/`daw` 文本对象）、可配置中断键、`/diff` 走 app-server、history cell 模块化重构、session picker 重设计。

## 七、三方对比小结（Qwen / OpenCode / Codex）

| 维度 | OpenAI Codex | OpenCode | Qwen Code |
|---|---|---|---|
| 5 月体量 | **960 PR**（~0% 机器人，~96% 内部）| 1567 commit（~27% 机器人，社区+维护者）| ~400 PR（混合团队）|
| 战略重心 | **企业自托管 agent 平台**（Windows/Bedrock/远程/治理）| 全栈产品 + Effect 重写 + **商业化**（Zen/stats）| **daemon Mode B** + telemetry + web-shell |
| server 架构 | **三平面分离**（app/exec/mcp 独立 crate）| 1 server 托多 workspace（location layer）| 1 daemon = 1 workspace（OS 隔离）|
| TUI 栈 | **Rust/ratatui**（原生编译）| Bun/SolidJS（OpenTUI）| TS/Ink+React |
| 自主性 | **goal 自动续跑 + Guardian 审批 + hook 全栈** | plan/build agent + plugin | skill + subagent + auto-dream + **Workflow tool（#4732 port Claude）**|
| 平台化 | **extension/plugin/skill 三层 + marketplace 企业治理** | 一切皆 plugin（AgentV2 registry）| skill + MCP bridge |
| 商业化 | OpenAI 订阅（无独立动作）| **Zen 网关 + stats 站 + referral** | 无（阿里背书）|
| 节奏 | ~3-4 天一 minor + Python SDK 独立线 | 每日滚动 patch | 阶段性版本 |

**三家的"性格"**：Codex = **企业级、内部高度集中、原子 PR、治理优先**；OpenCode = **社区 + 自动化重 + 商业化激进 + Effect 架构押注**；Qwen = **大厂支撑、daemon 多 client、telemetry 深**。

## 八、PR 参考

extension/plugin：#21736/#21738/#22216/#23823/#24723/#24700/#24996/#24977 · server：#24940/#21843/#22404/#19575/#24947/#25116/#24698 · thread：#19474/#22476/#24744/#24928/#22268/#25021 · goal：#23732/#20523/#24918/#24924/#24925/#23094 · Guardian：#24803/#24897/#22258 · hook：#24962/#24965/#22782/#23757 · Windows/sandbox：#24831/#23766/#20708/#18202/#23943 · Bedrock：#24701/#24960/#23860↔#24690 · SDK：#21891–#22021/#24800/#24828

---

> **免责声明**：基于 openai/codex 公开 merged PR（2026-05-01 ~ 05-31，960 PR）的标题/正文 + 选择性 diff 分析，无本地仓库（纯 gh CLI），截至 fetch 时点。PR 号取自标题。agent 校正：goal 33 / hook 36（非背景假设的 36/19）；5 月无 "PermissionDenied hooks" PR；GovCloud 加后即 revert。"企业战略意图"是基于动作模式的推断，非官方声明。与 Qwen/OpenCode 对比基于本仓库已有分析。
