# 四家 Code Agent 2026 年 5 月战略全景

> 横向综合 **Claude Code / OpenAI Codex / OpenCode / Qwen Code** 2026 年 5 月进展，提炼跨项目的战略走向。基于四篇月度深度分析：[Claude Code](./claude-code-2026-05-progress.md) · [Codex](./codex-2026-05-progress.md) · [OpenCode](./opencode-2026-05-progress.md) · Qwen Code（散见 [daemon 设计](./qwen-code-daemon-design/README.md) / [telemetry](./telemetry-architecture-deep-dive.md) / [月活贡献者](./contributors/2026-05.md)）。

## 核心论点

**2026 年 5 月是"长时程自主 agent（long-horizon autonomous agent）"成为行业共识的月份。** 四家——一家闭源大厂（Anthropic）、一家闭源大厂（OpenAI）、一家独立开源（OpenCode/sst）、一家大厂开源（Qwen/阿里）——**在同一个月、互不协调地，都在补同一套三层能力**：

1. **本地长驻 agent 编排 / fleet**（agent 能在后台跑很久、多 session 并行）
2. **LLM 驱动的大规模编排**（让模型自己写/驱动编排，fan-out 很多 subagent）
3. **"自动放行安全动作、拦截危险动作"的护栏**（agent 能自主，但每步可治理）

这种四方趋同不是巧合，而是**产品形态的代际迁移信号**：从"一问一答的 coding 助手"→"能锁定目标、长时间自主推进、可大规模并行、且全程可治理的 agent 平台"。

## 一、趋同：四家都在补的三层

### 第 1 层：本地长驻 agent 编排 / fleet

| 项目 | 形态 | 5 月关键 |
|---|---|---|
| **Claude Code** | `claude agents` 本地 daemon fleet + dashboard | **旗舰主线**（v2.1.139 Research Preview，全月数百条打磨）；与云端 fleet（Ultraplan/Ultrareview）正交 |
| **OpenAI Codex** | app-server 控制平面（唯一状态权威，TUI 退化为客户端）| 三平面 server 架构（app/exec/mcp），daemon 生命周期 + SSH 远程 bootstrap |
| **OpenCode** | location layer（1 server 托多 workspace，进程内 Effect 服务隔离）| `LayerMap` 带 TTL 缓存的"目录→服务子图" |
| **Qwen Code** | daemon Mode B（1 daemon = 1 workspace，OS 进程隔离）| 多 client（TUI/web-shell/ACP/MCP）接同一 daemon runtime |

**设计分歧很有意思**：隔离边界四家选择不同——Claude/Codex 靠 daemon 进程 + session 管理，OpenCode 靠**语言级（Effect 服务）边界**做"1 server 托多 workspace"，Qwen 反过来靠 **OS 进程边界**做"1 daemon = 1 workspace"。**OpenCode 与 Qwen 是正面对立的两条路**（单进程多目标 vs 多进程单目标）。

### 第 2 层：LLM 驱动的大规模编排

| 项目 | 形态 | 状态 |
|---|---|---|
| **Claude Code** | **dynamic workflows**：Claude 即兴写 JS 编排脚本，隔离 runtime 后台 fan-out 几十到上百 agent | ✅ **已发**（v2.1.154，行业首发）|
| **Qwen Code** | **Workflow tool**：`node:vm` sandbox 跑 model 写的 JS + sequential `agent()` | 🚧 **port 中**（#4732 P1，subagent prompt 逐字取自 claude-code 2.1.160 binary）|
| **OpenAI Codex** | goal 自动续跑 + multi-agent assignment | ✅ 默认开 |
| **OpenCode** | 一切皆 plugin（AgentV2 registry + immer editor，7 内置 agent）| ✅ |

**关键观察**：Claude 首发 dynamic workflows，**Qwen 一周内就开始 port**（#4732，甚至逐字搬 Claude 的 subagent system prompt）——说明"LLM 写编排脚本"这个抽象**已成共识，不是 Claude 独有**。Codex/OpenCode 走的是"预定义 agent 角色 + 编排"的另一条路（不让 LLM 即兴写脚本）。

### 第 3 层：自动批安全 / 拦危险的护栏

| 项目 | 机制 | 5 月动作 |
|---|---|---|
| **Claude Code** | auto mode（分类器判断危险）| 转默认（不再 opt-in）+ `hard_deny` + **批量外泄检测** |
| **OpenAI Codex** | **Guardian**（LLM 逐动作评 risk + authorization）| 生产硬化：cache key 省 token + OTel metrics 放量可观测 |
| **OpenCode** | agent 声明式 permission ruleset（`allow/ask/deny` per pattern）| 每个 agent 带细粒度规则 |
| **Qwen Code** | AUTO classifier + PermissionMediator（4 策略）| AUTO classifier PermissionDenied hook + 多 client 投票 |

**共同命题**：自主性越强，逐动作审批越关键。四家都在用某种"判断器"把权限从"每次都问"推向"该问才问"。其中 **Codex Guardian 最重**（独立 LLM 取证式审批），**Claude auto mode 最产品化**（默认开 + 外泄检测），**Qwen 最偏多 client 协调**（投票），**OpenCode 最声明式**（规则表）。

## 二、分化：用户 / 场景定位

趋同的是能力，分化的是**为谁做、靠什么活**：

| 项目 | 战略重心 | 商业模式 | 一句话定位 |
|---|---|---|---|
| **Claude Code** | 本地 fleet + 工作流 + 自家 Opus 4.8 + 企业加固 | Anthropic 订阅（闭源护城河）| 模型 + 编排 + 平台全栈，自家模型绑定 |
| **OpenAI Codex** | 企业自托管平台 + 逐动作治理 | OpenAI 订阅 | Windows/Bedrock 合规、远程托管、可审计沙箱——**进企业 IT** |
| **OpenCode** | 全栈产品 + Effect 架构 + **商业化** | **Zen 模型网关 + stats 站 + referral** | 独立开源**求变现**：转售模型 + 增长营销 |
| **Qwen Code** | daemon 多端 runtime + telemetry 深耕 | 无（阿里背书）| 统一 runtime 接多端，无商业化压力 |

**最鲜明的对比**：
- **OpenCode 是唯一在积极变现的**（Zen 网关转售模型 + 公开 leaderboard 营销 + referral 裂变）——独立开源项目的生存逻辑
- **Codex 是唯一压倒性押企业的**（5 月最大主题是 Windows 受管部署 + Bedrock/GovCloud 合规通道 + 远程托管执行）
- **Claude 靠自家 Opus 4.8 模型**做差异化（其它三家都得接多 provider）
- **Qwen 没有商业化压力**（阿里背书），可专注 runtime/telemetry 工程

## 三、节奏与工程文化对比

| 项目 | 5 月体量 | 自动化噪声 | 集中度 | 工作流特征 |
|---|---|---|---|---|
| **Codex** | 960 PR | **~0% 机器人** | **~96% OpenAI 内部**，一人 ~20% | AI 辅助原子微 PR（`[1/8]` 切片 / `[codex]` 标记 / 7 连删）|
| **OpenCode** | 1567 commit | **~27% 机器人** | 社区 + 维护者 | 高频微 commit + 机器人 generate + 每日滚动发布 |
| **Claude Code** | 28 版本（闭源）| 不可见 | Anthropic 内部 | 近每日 patch + 海量边缘修复（打磨 `claude agents`）|
| **Qwen Code** | ~400 PR | dependabot 存在 | 混合内部团队 + 社区 | daemon 团队 4 人 + 社区长尾 |

**工程文化差异**：Codex 是"公司内部项目开源化"（~96% 内部 + ~0% 机器人 + AI 辅助原子 PR，有效人/agent 吞吐最高）；OpenCode 是"社区 + 重度自动化"（机器人 commit ≈ 头号人类作者）；Qwen 是"大厂团队 + 社区混合"；Claude 闭源不可比但近每日 patch 显示高速迭代。

## 四、对 Qwen Code 的启示

1. **Workflow tool（#4732）方向押对了**：dynamic workflows 是四家共识（Claude 已发、Qwen port 中、Codex/OpenCode 有变体），不是跟风 Claude 一家。值得继续推 P2/P3。
2. **`claude agents` 的 dashboard UX 值得 daemon web-shell 参考**：Claude 把本地 fleet 做成一个 agent dashboard（running/blocked/done 一览 + attach/detach），Qwen daemon 的 web-shell（ytahdn）可借鉴这种"fleet 总览"交互。
3. **Guardian 式逐动作审批是 Qwen permission flow 的升级方向**：Qwen 现在是规则匹配 + 多 client 投票；Codex Guardian 的"LLM 取证式判断 + cache key 省 token + OTel 可观测"是把审批从硬规则升到"会思考"的范本（详 [四方周报 §Guardian](./weekly/2026-W22-0524-0531.md)）。
4. **不必跟 OpenCode 的商业化**：Qwen 有阿里背书无变现压力，OpenCode 的 Zen/stats 是独立项目求生逻辑，不适用 Qwen。
5. **telemetry 是 Qwen 的相对优势**：Qwen 的 OTel 分层 tracing（#3731 全 phase）+ daemon 跨进程 trace 在四家里深度领先，可作为差异化继续投。

## 五、一句话总结

**2026 年 5 月，四家在"长时程自主 agent + 大规模编排 + 可治理护栏"上空前趋同——这是 coding agent 从"助手"迈向"自主 agent 平台"的代际拐点。** 但变现逻辑、部署场景、工程文化四家各异：Claude 押自家模型 + 本地 fleet，Codex 押企业自托管 + 治理，OpenCode 押商业化 + 架构，Qwen 押多端 runtime + telemetry。**能力收敛，定位分化。**

---

> **免责声明**：本文是对四篇月度分析的横向综合，证据强度**不均**——Codex/OpenCode 基于 git PR/commit 级分析，Claude Code 仅基于闭源 changelog（无源码粒度），Qwen Code 基于本仓库持续跟踪。"行业共识""代际拐点"等判断是基于四方动作模式的推断，非任何官方声明。各项目数据截至 2026-06-04。
