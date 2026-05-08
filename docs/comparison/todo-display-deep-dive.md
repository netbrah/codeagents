# Todo / Plan 展示 Deep-Dive——4 方对比（Qwen Code / OpenCode / Codex CLI / Claude Code）

> Code Agent 的"任务列表 / Plan"展示，是 LLM 跟踪工作进展的核心 UX。本文 4 方源码层对比工具命名、数据结构、UI 展示、智能集成、resize 行为、设计哲学。
>
> **更新日期**：2026-05-07

## 零、4 方源码定位

| Agent | 工具实现 | UI 组件 | 行数 |
|---|---|---|---|
| **Claude Code** | `tools/TodoWriteTool/TodoWriteTool.ts` | （leaked 仓未见独立 component；推测 inline tool result）| 工具 ~120 行 + prompt 9527B + types |
| **Qwen Code** | `packages/core/src/tools/todoWrite.ts` | `cli/src/ui/components/TodoDisplay.tsx` + **`StickyTodoList.tsx`** | 470 + 72 + 135 = 677 行 |
| **OpenCode** | `packages/opencode/src/tool/todo.ts` + `src/session/todo.ts` | `cli/cmd/tui/component/todo-item.tsx` + **`app/src/pages/session/composer/session-todo-dock.tsx`** + `feature-plugins/sidebar/todo.tsx` | 多文件分布 |
| **Codex CLI** | `codex-rs/core/src/tools/spec_tests.rs` 注册 `update_plan` | `codex-rs/tui/src/history_cell.rs:2557 new_plan_update` + `ProposedPlanCell` | history_cell.rs 内 ~140 行 |

> **注**：Claude Code 数据来自反编译的 leaked 仓（v2.1.81 时点），Qwen Code / OpenCode / Codex CLI 来自 GitHub 公开源码。

## 一、TL;DR

```
Claude Code  → 智能化最深（VerificationNudge + Auto Mode 集成 + V2 feature gate）
Qwen Code    → 唯一 sticky 常驻列表（StickyTodoList 屏幕固定位置展示）
OpenCode     → 多端 UI 最丰富（CLI + Web dock + Sidebar + 4 状态 + 优先级）
Codex CLI    → 设计分叉（不叫 todo，叫 update_plan，含 explanation 叙事）
```

**共识**：4 方都用"工具触发 + UI 渲染"两层架构，但 4 方有 4 种不同侧重——智能化、视觉一致性、多端 UX、表达力。

## 二、工具命名 + 状态种类

| Agent | 工具名 | 状态种类（数）| 视觉规范 |
|---|---|---|---|
| **Claude Code** | `TodoWriteTool` | pending / in_progress / completed（含 `isTodoV2Enabled` gate 暗示 V2 演进）| 组件未公开（leaked 仓仅工具代码 + prompt）|
| **Qwen Code** | `todoWrite` | 3 类：pending / in_progress / completed | `○ ◐ ●` |
| **OpenCode** | `todowrite` | **4 类**：pending / in_progress / completed / **cancelled** | Checkbox + animated dot（in_progress）|
| **Codex CLI** | `update_plan`（**不叫 todo**）| 2 状态：✔ completed / □ pending（Codex 文档实测中 in_progress 也存在）| `• Updated Plan` + `└ explanation` + `✔ □` |

**独家命名**：Codex 把工具叫 **`update_plan`** 而非 `todo_write`——这暗示语义差异（plan 是"叙事 + 步骤"，todo 是"任务列表"）。

## 三、数据结构差异

### 3.1 Claude Code（leaked 仓）

```ts
// tools/TodoWriteTool/TodoWriteTool.ts
const inputSchema = lazySchema(() =>
  z.strictObject({
    todos: TodoListSchema().describe('The updated todo list'),
  }),
)

const outputSchema = lazySchema(() =>
  z.object({
    oldTodos: TodoListSchema(),
    newTodos: TodoListSchema(),
    verificationNudgeNeeded: z.boolean().optional(),  // 独家：动态计算的 nudge 信号
  }),
)
```

`TodoListSchema()` 在 `utils/todo/types.ts` 定义（leaked 仓只导出 types.ts）。

### 3.2 Qwen Code

```ts
// packages/cli/src/ui/components/TodoDisplay.tsx
export interface TodoItem {
  id: string;
  content: string;
  status: 'pending' | 'in_progress' | 'completed';
}
```

最简洁 schema：3 字段。

### 3.3 OpenCode

```ts
// packages/opencode/src/tool/todo.ts
const TodoItem = Schema.Struct({
  content: Schema.String,
  status: Schema.String,           // pending | in_progress | completed | cancelled
  priority: Schema.String,         // 独家：high | medium | low
})

export const Parameters = Schema.Struct({
  todos: Schema.mutable(Schema.Array(TodoItem)),
})
```

**独家 priority 字段**——4 方里只有 OpenCode 有任务优先级。

### 3.4 Codex CLI

```rust
// codex-rs/protocol-types/src/plan_tool.rs
struct UpdatePlanArgs {
    explanation: String,      // 独家：plan 整体叙事（独立于 items）
    plan: Vec<PlanItemArg>,
}

struct PlanItemArg {
    step: String,
    status: PlanItemStatus,   // pending | in_progress | completed
}
```

**独家 `explanation` 字段**——4 方里只有 Codex 有"plan 叙事"概念，把"为什么这个 plan"和"plan 的 steps"分开。

### 3.5 关键 schema 差异速览

| 字段 | Claude | Qwen | OpenCode | Codex |
|---|---|---|---|---|
| id | ✓ | ✓ | ✗（隐式 index）| ✗（隐式 index）|
| content / step | ✓ content | ✓ content | ✓ content | ✓ step |
| status | ✓ | ✓ | ✓ | ✓ |
| **priority** | ✗ | ✗ | **✓ high/medium/low** | ✗ |
| **explanation**（叙事）| ✗ | ✗ | ✗ | **✓** |
| activeForm（动名词）| **✓ 暗示** | ✗ | ✗ | ✗ |

## 四、UI 展示位置详解

### 4.1 Claude Code

| 维度 | 现状 |
|---|---|
| TUI 渲染位置 | leaked 仓未导出独立组件（推测 inline tool result）|
| 持久化 | `appState.todos[todoKey]`，**sessionId-keyed**（每 session 独立）|
| 多端 | CLI only（推测 web 版有但未 leaked）|
| renderToolUseMessage | `return null`（不渲染默认 tool message）|

### 4.2 Qwen Code

| 维度 | 现状 |
|---|---|
| TUI 渲染（基础）| `TodoDisplay.tsx`（72 行 · 列表内联）|
| **Sticky 常驻**（独家）| **`StickyTodoList.tsx`**（135 行 · `STICKY_TODO_MAX_VISIBLE_ITEMS` 限高 · `getOrderedStickyTodos` 排序 · `getStickyTodosRenderKey` 重渲染信号）|
| 持久化辅助 | `cli/src/ui/utils/todoSnapshot.ts`（snapshot diff 工具）|
| 多端 | CLI only |

```tsx
// StickyTodoList.tsx 核心结构（推测 layout）
<Box flexDirection="column">
  {orderedTodos.slice(0, STICKY_TODO_MAX_VISIBLE_ITEMS).map(todo => (
    <Text>{STATUS_ICONS[todo.status]} {todo.content}</Text>
  ))}
</Box>
```

**独家**：屏幕**固定位置常驻**显示 Todo——4 方里仅 Qwen 有这种"屏幕边缘任务栏"。

### 4.3 OpenCode

OpenCode 的 Todo UI 是 4 方里**最丰富的**：

| 位置 | 文件 | 形态 |
|---|---|---|
| TUI inline | `cli/cmd/tui/component/todo-item.tsx` | 列表内联 |
| **Web UI dock** | `app/src/pages/session/composer/session-todo-dock.tsx` | **dock-tray + Checkbox + AnimatedNumber + TextReveal + TextStrikethrough + motion-spring 动画** |
| Sidebar plugin | `cli/cmd/tui/feature-plugins/sidebar/todo.tsx` | 侧栏插件展示 |
| E2E 测试 | `app/e2e/todo.spec.ts` | 完整端到端测试 |
| 图标 | `ui/src/assets/icons/file-types/todo.svg` | UI 资产 |
| Storybook | `ui/src/components/todo-panel-motion.stories.tsx` | 动画 stories |

```tsx
// session-todo-dock.tsx (摘录)
function dot(status: Todo["status"]) {
  if (status !== "in_progress") return undefined
  return (
    <svg ...>
      <circle cx="6" cy="6" r="3"
        style={{ animation: "var(--animate-pulse-scale)" }}/>
    </svg>
  )
}
```

**独家**：**in_progress 状态有 pulse-scale 动画 dot**——视觉反馈最丰富。

### 4.4 Codex CLI

| 维度 | 现状 |
|---|---|
| 渲染入口 | `codex-rs/tui/src/history_cell.rs:2557 new_plan_update` |
| 持久 cell 类型 | `PlanUpdateCell { explanation, plan }`（结构化）|
| **Streaming cell** 类型 | `ProposedPlanStreamCell { lines, is_stream_continuation }`（瞬时）|
| **Source-backed cell** 类型 | `ProposedPlanCell { plan_markdown, cwd }`（finalized · 含 cwd 用于 link reflow）|
| Snapshot 渲染（实测）| `• Updated Plan / └ explanation / ✔ ✓ items / □ pending items` |

```rust
// 实测 snapshot 输出（codex_tui__history_cell__tests__plan_update_with_note_and_wrapping_snapshot.snap）
• Updated Plan
  └ I'll update Grafana call
    error handling by adding
    retries and clearer messages
    when the backend is
    unreachable.
    ✔ Investigate existing error
      paths and logging around
      HTTP timeouts
    □ Harden Grafana client
      error handling with retry/
      backoff and user-friendly
      messages
    □ Add tests for transient
      failure scenarios and
      surfacing to the UI
```

**独家**：**streaming → finalized 双 cell 结构**——streaming 期间用 `ProposedPlanStreamCell`（已渲染 lines 但不可重 reflow），完成后 consolidate 为 `ProposedPlanCell`（store raw markdown + cwd，**支持 terminal resize 时 reflow 重渲染**）。

## 五、智能集成深度对比

### 5.1 Claude Code — 智能化最深

```ts
// tools/TodoWriteTool/TodoWriteTool.ts:64-76
let verificationNudgeNeeded = false
if (
  feature('VERIFICATION_AGENT') &&
  getFeatureValue_CACHED_MAY_BE_STALE('tengu_hive_evidence', false) &&
  !context.agentId &&
  allDone &&
  todos.length >= 3 &&
  !todos.some(t => /verif/i.test(t.content))
) {
  verificationNudgeNeeded = true
}
```

**核心创新**：
- **VerificationNudge**：main agent 关闭 3+ items 且 none 是 verification step → 自动 nudge "spawn VERIFICATION_AGENT"
- 配 prompt：`"You cannot self-assign PARTIAL by listing caveats in your summary — only the verifier issues a verdict"`
- Feature gate：`VERIFICATION_AGENT` + GrowthBook `tengu_hive_evidence` 双 gate
- **`toAutoClassifierInput()`**：与 Auto Mode 权限分类器集成（v2.1.132 引入的 permission classifier）
- **`shouldDefer: true`**：不阻塞 tool flow
- **`isTodoV2Enabled()` gate**：暗示 V2 版本演进中

→ Claude 把 todo 视为**LLM 行为反馈循环的入口**——不仅是任务追踪，更是"防 LLM 跳过验证步骤"的硬约束。

### 5.2 Qwen Code — 基础展示

- 工具 + 2 个 UI 组件（TodoDisplay + StickyTodoList）
- 无 nudge / 无 classifier / 无 V2 演进
- 470 行工具实现（远比 Claude 120 行多）—— 但不在智能化方向，可能是 i18n / 验证 / sticky 集成的工程量

→ Qwen 把 todo 视为**视觉 UX**——重心在常驻显示（StickyTodoList）让用户始终看到进展。

### 5.3 OpenCode — UI 最佳，无智能化

- Effect Schema 替代 zod
- 4 状态（含 cancelled）+ 优先级
- Web UI 动画（AnimatedNumber / TextReveal / TextStrikethrough / motion-spring / pulse-scale dot）
- 无 verification nudge / 无 classifier 集成

→ OpenCode 把 todo 视为**多端共享 UX**——CLI + Web dock + Sidebar 三处展示，体验最丰富但智能化最弱。

### 5.4 Codex CLI — 表达力最强

- `explanation` 字段：plan 整体叙事独立于 items
- streaming → finalized 双 cell 结构
- 完整 resize reflow（store raw markdown + cwd）

→ Codex 把 plan 视为**LLM 输出的结构化展示**——重心在"如何展示 plan 让用户理解 LLM 思路"，而非追踪进度。这与其他 3 方的"任务列表"哲学不同。

## 六、跨终端 reflow / 持久化 / 多端

| Agent | resize 行为 | 持久化 | 多端同步 |
|---|---|---|---|
| **Claude Code** | 推测静态（leaked 仓未见 reflow 代码）| `appState.todos[todoKey]`（sessionId-keyed）| 推测无 |
| **Qwen Code** | sticky list 按宽度截断（`STICKY_TODO_MAX_VISIBLE_ITEMS`）| 内存 state | 无 |
| **OpenCode** | TUI 静态 + **Web UI 响应式 CSS** | Effect `Todo.Service` + `@opencode-ai/sdk/v2` | **✓ CLI ↔ Web SDK 同步** |
| **Codex CLI** | **完整 reflow**（store raw markdown + 重新渲染）| history cell（消息流持久化）| 无 |

**Codex 独家 reflow**：

```rust
/// Finalized proposed-plan history that can render itself again for a new width.
/// 
/// This is the source-backed counterpart to `ProposedPlanStreamCell`. It owns 
/// raw markdown and the session cwd needed for stable local-link rendering 
/// during later transcript reflow.
pub(crate) struct ProposedPlanCell {
    plan_markdown: String,
    cwd: PathBuf,
}
```

reflow 设计的**意义**：用户调整终端宽度时，plan 不会变成乱码——4 方里只有 Codex 显式处理这个问题。

## 七、设计哲学一句话

| Agent | 设计目标 |
|---|---|
| **Claude Code** | "智能跟踪 + 防 LLM 跳过验证"（VerificationNudge + Auto Mode 集成）|
| **Qwen Code** | "视觉一致性 + 持续可见"（StickyTodoList 屏幕常驻）|
| **OpenCode** | "多端最佳 UX"（Web dock-tray 动画 + 优先级 + 多 status 维度）|
| **Codex CLI** | "plan = 叙事 + checkbox 一体"（explanation 字段独立 + 完整 resize reflow）|

## 八、关键差异化矩阵

| 能力 | Claude | Qwen | OpenCode | Codex |
|---|---|---|---|---|
| 工具命名 | TodoWrite | todoWrite | todowrite | **update_plan** |
| 状态数 | 3 | 3 | **4**（+ cancelled）| 3（实测含 in_progress）|
| 优先级字段 | ✗ | ✗ | **✓** | ✗ |
| 叙事字段（explanation）| ✗ | ✗ | ✗ | **✓** |
| activeForm（动名词）| ✓ 暗示 | ✗ | ✗ | ✗ |
| **Sticky 屏幕常驻** | ✗ | **✓** | ✗（dock 在 Web）| ✗ |
| **Web UI dock** | ✗ | ✗ | **✓ + 动画** | ✗ |
| **In_progress 动画 dot** | ✗ | ✗ | **✓ pulse-scale** | ✗ |
| **VerificationNudge** | **✓ feature-gated** | ✗ | ✗ | ✗ |
| **Auto Mode classifier 集成** | **✓ toAutoClassifierInput** | ✗ | ✗ | ✗ |
| **shouldDefer**（不阻塞 tool flow）| **✓** | ? | ? | ? |
| **streaming / finalized 双 cell** | ✗ | ✗ | ✗ | **✓** |
| **Resize reflow**（raw markdown 重渲染）| ✗ | partial（截断）| Web 响应式 | **✓ raw markdown + cwd** |
| **多端 SDK 同步** | ✗（推测）| ✗ | **✓ CLI ↔ Web SDK** | ✗ |
| 状态视觉规范 | 未公开 | `○ ◐ ●` | Checkbox + dot | `✔ □` + `• `+`└` |
| Schema 引擎 | zod | zod | **Effect Schema** | Rust struct（serde）|

## 九、关键洞察

### 9.1 Codex 走分叉路线

把"todo"抽象成 **plan = explanation + checkbox steps**——工具叫 `update_plan` 不叫 todo。设计上更接近"LLM 输出的结构化叙事 + 进度",而非"用户任务列表"。

### 9.2 OpenCode UI 最丰富但智能化最弱

4 状态（多一个 cancelled）+ 优先级（high/medium/low）+ Web dock + 动画 + Sidebar + Storybook + e2e test——但**无 verification nudge / 无 LLM-in-the-loop 智能集成**。

### 9.3 Claude 智能化最深但 UI 信息最少

Leaked 仓里**没有独立 TodoDisplay 组件**——可能因为 Claude 用 inline tool result 渲染。但工具代码里有 verification nudge / Auto Mode 集成 / V2 gate / shouldDefer 等智能化层面的设计。

### 9.4 Qwen 独家 StickyTodoList

4 方里**唯一有"sticky 屏幕常驻"展示**——其他 3 方要么在消息流内（Claude/Codex）要么在 Web dock（OpenCode）。`STICKY_TODO_MAX_VISIBLE_ITEMS` 限高的设计避免占屏过多。

### 9.5 Anthropic Verification Agent 思路独家

Claude 通过 todo 的 nudge 自动启动 verification agent ——是 "LLM-in-the-loop 反馈"思路，4 方里**仅 Claude 有**。这与 [§subagent-display §六.10 LiveAgentPanel](./subagent-display-deep-dive.md#已落地-10liveagentpanel-port-claude-coordinatortaskpanel-模式pr3909) 反过来——subagent display 是 Qwen 主动学 Claude；verification nudge 是 Claude 独家创新。

### 9.6 Codex resize reflow 设计精细

`ProposedPlanStreamCell` 流期间快速渲染（已 layout 的 lines）+ `ProposedPlanCell` finalized 后存 raw markdown + cwd 支持 reflow——是**响应式终端 UI** 思路在 Code Agent 的应用范例。

## 十、推荐借鉴方向

### 10.1 Qwen Code 可借鉴

| 借鉴自 | 内容 | 优先级 |
|---|---|---|
| OpenCode | priority 字段（high/medium/low） | 中 |
| OpenCode | cancelled 状态（明确非 GOAL 终止） | 中 |
| Claude | VerificationNudge 思路（与 [§subagent-display verification agent](./subagent-display-deep-dive.md) 协同） | 高 |
| Codex | explanation 字段（plan 叙事独立） | 低 |
| Codex | resize reflow（raw markdown + cwd） | 中 |

### 10.2 Claude Code 可借鉴

| 借鉴自 | 内容 |
|---|---|
| Qwen | StickyTodoList 屏幕常驻 |
| OpenCode | Web UI dock-tray 动画 |
| Codex | resize reflow |

### 10.3 OpenCode 可借鉴

| 借鉴自 | 内容 |
|---|---|
| Claude | VerificationNudge + LLM-in-the-loop 智能集成 |
| Qwen | StickyTodoList（CLI 端常驻）|

### 10.4 Codex 可借鉴

| 借鉴自 | 内容 |
|---|---|
| OpenCode | priority + cancelled |
| Claude | nudge / classifier 集成 |
| Qwen | sticky 常驻显示 |

## 十一、关键文件速查

| 项目 | 文件 |
|---|---|
| Claude Code 工具 | `tools/TodoWriteTool/TodoWriteTool.ts`（leaked）|
| Claude Code prompt | `tools/TodoWriteTool/prompt.ts`（9527 字节）|
| Claude Code types | `utils/todo/types.ts`（leaked 仅此暴露）|
| Qwen Code 工具 | `packages/core/src/tools/todoWrite.ts`（470 行）|
| Qwen Code 内联展示 | `packages/cli/src/ui/components/TodoDisplay.tsx`（72 行）|
| Qwen Code Sticky | `packages/cli/src/ui/components/StickyTodoList.tsx`（135 行）|
| Qwen Code Snapshot | `packages/cli/src/ui/utils/todoSnapshot.ts` |
| OpenCode LLM 工具 | `packages/opencode/src/tool/todo.ts` |
| OpenCode Service | `packages/opencode/src/session/todo.ts` |
| OpenCode TUI | `packages/opencode/src/cli/cmd/tui/component/todo-item.tsx` |
| OpenCode Web Dock | `packages/app/src/pages/session/composer/session-todo-dock.tsx` |
| OpenCode Sidebar | `packages/opencode/src/cli/cmd/tui/feature-plugins/sidebar/todo.tsx` |
| OpenCode E2E | `packages/app/e2e/todo.spec.ts` |
| OpenCode Stories | `packages/ui/src/components/todo-panel-motion.stories.tsx` |
| Codex 工具注册 | `codex-rs/core/src/tools/spec_tests.rs:419+`（`update_plan`）|
| Codex Plan types | `codex-rs/protocol-types/src/plan_tool.rs`（`UpdatePlanArgs` + `PlanItemArg`）|
| Codex 渲染 | `codex-rs/tui/src/history_cell.rs:2557 new_plan_update` |
| Codex Snapshot tests | `codex-rs/tui/src/snapshots/codex_tui__history_cell__tests__plan_update_*.snap` |

## 十二、与本系列其他文档的关联

| 本文 | 相关 codeagents 文档 |
|---|---|
| Claude VerificationNudge → Verification Agent | [SubAgent 展示 Deep-Dive](./subagent-display-deep-dive.md)——LiveAgentPanel + 多 agent 设计 |
| OpenCode Web SDK 同步 | [Qwen daemon §09 OpenCode 详细对比](./qwen-code-daemon-design/09-comparison-with-opencode.md) |
| OpenCode 开源 Managed Code Agent | [Managed Code Agents 全景](./managed-agents-landscape.md) |
| 4 方功能矩阵 | [features.md](./features.md) |
| Claude Code v2.1.82 → v2.1.132 增量 | [Claude Code §23 近期更新](../tools/claude-code/23-recent-updates.md) |

## 十三、一句话总结

**Code Agent 的 Todo / Plan 展示存在 4 种设计哲学：Claude Code 走"智能化"路线（VerificationNudge + Auto Mode classifier 集成 + V2 gate 演进，把 todo 当作 LLM 行为反馈入口）；Qwen Code 走"视觉一致性"路线（独家 StickyTodoList 屏幕常驻 + 简洁 3 状态 + 内联 + sticky 双展示）；OpenCode 走"多端 UX"路线（CLI + Web dock-tray 动画 + Sidebar 三处展示 + 4 状态含 cancelled + 优先级 + Effect Schema + CLI↔Web SDK 同步）；Codex CLI 走"分叉表达力"路线（不叫 todo 叫 update_plan，含 explanation 叙事字段独立于 items + streaming/finalized 双 cell 结构 + 完整 resize reflow with raw markdown + cwd）。共识是 4 方都"工具触发 + UI 渲染"两层架构，差异化体现各自工程偏好——Claude 在反馈循环、Qwen 在屏幕占位、OpenCode 在多端动画、Codex 在结构化叙事。借鉴方向上 Qwen 可吸收 OpenCode priority/cancelled + Claude VerificationNudge + Codex resize reflow；Claude 可吸收 Qwen StickyTodoList + OpenCode dock 动画；其他类似。**

---

> **数据来源**：本文 4 方源码的最后实测日期 2026-05-07。Claude Code 数据来自反编译的 leaked 仓（v2.1.81 时点 + 2026-04 后增量）；其他 3 方来自当前 GitHub 公开源码。可能未涵盖正在研发但未合并的 PR。
