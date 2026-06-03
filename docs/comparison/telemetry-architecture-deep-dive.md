# 遥测 (Telemetry) 架构 Deep-Dive

> AI Agent 收集什么数据、如何上报、用户如何控制？本文基于 Claude Code（v2.1.89 源码分析）和 Qwen Code（v0.16.0 开源）的源码分析，对比两者在遥测 (Telemetry) 架构、事件体系和隐私控制方面的差异。

---

## 1. 架构总览

| 维度 | Claude Code | Qwen Code |
|------|------------|-----------|
| **遥测框架** | 自定义 1P Logger + OpenTelemetry | QwenLogger（RUM）+ OpenTelemetry |
| **上报端点** | Anthropic 内部 API + Datadog | 阿里云 RUM + OTLP（可配置，支持 gRPC / HTTP 双协议） |
| **事件数量** | ~656 个 `tengu_*` 事件 | ~50 个事件类型 |
| **采样策略** | 按事件类型动态采样（GrowthBook） | 批量刷新（1,000 条 / 60 秒） |
| **调试追踪** | Perfetto（Chrome Trace，ant-only） | 层级 Span 树（interaction → llm_request → tool/hook，#3731；2026-05 加 TTFT + GenAI semconv + 跨进程 daemon trace） |
| **PII 保护** | 元数据禁止原始字符串 | 选择性 prompt 日志控制 + 敏感属性 opt-in |
| **禁用方式** | `DISABLE_TELEMETRY=true` | OTel 链 `QWEN_TELEMETRY_ENABLED=false`（默认已关）；RUM 使用统计另需关 `getUsageStatisticsEnabled()`（默认开）|

---

## 2. Claude Code：双通道遥测

### 2.1 事件日志通道（1P Event Logging）

```typescript
// 源码: services/analytics/firstPartyEventLogger.ts
// 使用 @opentelemetry/sdk-logs 的 LoggerProvider + BatchLogRecordProcessor
// 自定义 Exporter: FirstPartyEventLoggingExporter
// 端点: ${BASE_API_URL}/api/event_logging/batch
// 批量配置通过 GrowthBook 动态控制（tengu_1p_event_batch_config）
```

**事件类型**（~656 个唯一 `tengu_*` 前缀，源码 `grep -roh` 统计；含动态构造事件名时约 782 个）：

| 类别 | 示例 |
|------|------|
| Agent 生命周期 | `tengu_agent_created`, `tengu_agent_tool_selected`, `tengu_agent_tool_completed` |
| API 交互 | `tengu_api`, `tengu_api_error`, `tengu_api_cache_breakpoints` |
| 工具执行 | `tengu_tool_*`, `tengu_streaming_tool_execution_used` |
| 会话管理 | `tengu_session_*`, `tengu_compact_*` |
| Feature Flag | `tengu_amber_flint`, `tengu_amber_prism` |
| 安全 | `tengu_cancel`, `tengu_pre_stop_hooks_cancelled` |
| UI | `tengu_brief_mode_toggled`, `tengu_conversation_forked` |

**PII 保护**：

```typescript
// 源码: services/analytics/index.ts
// 元数据类型标注: AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS
// → 开发者必须显式声明元数据不含代码或文件路径
// → 类型系统阻止意外记录敏感信息
```

### 2.2 Perfetto 调试追踪（ant-only）

```typescript
// 源码: utils/telemetry/perfettoTracing.ts
// 格式: Chrome Trace Event (JSON)，在 ui.perfetto.dev 或 chrome://tracing 查看
// 启用: CLAUDE_CODE_PERFETTO_TRACE=1 或 CLAUDE_CODE_PERFETTO_TRACE=<path>
// 输出: ~/.claude/traces/trace-<session-id>.json
// 事件上限: 100K 条（超出时淘汰最老的 50%，约 30MB）
```

**Perfetto 追踪捕获**：

| 捕获内容 | 详情 |
|----------|------|
| Agent 层级 | 父子 swarm 关系 |
| API 请求 | TTFT、TTLT、prompt 长度、cache 统计 |
| 工具执行 | 名称、耗时、token 用量 |
| 用户等待 | 输入等待时间 |
| Speculation | 推测执行标记 |

### 2.3 采样 (Sampling) 与降级 (Fallback)

```typescript
// 按事件采样 (Sampling): GrowthBook tengu_event_sampling_config（每事件 0-1 概率）
// 第三方 Provider: 自动禁用分析（Bedrock/Vertex/Foundry）
// 测试环境: NODE_ENV === 'test' 时禁用
// 隐私模式: isTelemetryDisabled() 检查
```

---

## 3. Qwen Code：RUM + OTLP 双通道

### 3.1 QwenLogger（RUM 通道）

```typescript
// 源码: qwen-code/packages/core/src/telemetry/qwen-logger/qwen-logger.ts
// 类型: 单例 RUM (Real User Monitoring) Logger
// 端点: gb4w8c3ygj-default-sea.rum.aliyuncs.com（阿里云 RUM）
// 批量: 默认 1,000 条 / 60 秒刷新
// 队列: FixedDeque（溢出时丢弃最老事件）
// v0.16.0 变化: readSourceInfo() 新增支持 QWEN_HOME 自定义路径，
//   同时兼容遗留 ~/.qwen/source.json（#2953）；移除 tool_token_count 上报（#3727）
```

**事件类型**（~50 种）：

| 类别 | 事件 |
|------|------|
| 会话 | `session_start`, `session_end` |
| 用户操作 | `new_prompt`, `retry`, `slash_command`, `user_feedback` |
| 工具 | `tool_call#<name>`, `file_operation#<name>`, `tool_output_truncated` |
| API | `api_request`, `api_response`, `api_cancel`, `api_error` |
| 错误 | `invalid_chunk`, `malformed_json_response`, `loop_detected` |
| 扩展 | `extension_install`, `extension_uninstall`, `extension_update` |
| Arena | `arena_session_started`, `arena_agent_completed` |
| Hook | `hook_call#<event_name>` |
| 压缩 | `chat_compression` |

### 3.2 OpenTelemetry 通道

```typescript
// 源码: qwen-code/packages/core/src/telemetry/sdk.ts
// 3 种 Exporter（v0.16.0，移除 Console Exporter）:
// 1. OTLP gRPC   — OTLPTraceExporter + GZIP 压缩
// 2. OTLP HTTP   — OTLPTraceExporter[Http]（v0.16.0 新增 per-signal 端点路由）
// 3. File        — FileSpanExporter（本地文件）
//
// Processor: BatchSpanProcessor, BatchLogRecordProcessor, LogToSpanProcessor（新增）
// 检测: HttpInstrumentation（HTTP 请求自动追踪）
// 诊断日志: 改为写 debug log 而非 console，避免 UI 污染（#3986）
// shutdown 上限: 10 秒（#3813）
```

**遥测目标**（v0.16.0）：

| 目标 | 用途 |
|------|------|
| `TelemetryTarget.LOCAL` | 本地 OTEL Collector |
| `TelemetryTarget.GCP` | Google Cloud |
| ~~`TelemetryTarget.QWEN`~~ | **已移除**（#4061 移除 dead 代码） |

### 3.3 层级 Span 树（v0.16.0 新增）

v0.16.0 新增 `session-tracing.ts`（883 行），建立完整的层级 Trace 结构：

```
session（根）
  └─ interaction（每次用户 prompt）
       ├─ llm_request（每次 API 调用）
       └─ tool（每次工具调用）
            ├─ tool.execution（实际执行子 span）
            ├─ tool.blocked_on_user（等待用户审批）
            └─ hook（Hook 执行）
```

**关键新 Span 函数**：

| 函数 | PR | 说明 |
|------|----|------|
| `startInteractionSpan` / `endInteractionSpan` | #4071 | 每轮对话根 span |
| `startLLMRequestSpan` / `endLLMRequestSpan` | #4071 | API 调用子 span |
| `startToolSpan` / `endToolSpan` | #4071/#4126 | 工具调用子 span |
| `startToolBlockedOnUserSpan` | #3731 | 权限等待阶段子 span |
| `startHookSpan` / `endHookSpan` | #3731 | Hook 执行子 span |

**敏感属性 opt-in**（#3893/#4097）：设置 `QWEN_TELEMETRY_INCLUDE_SENSITIVE_SPAN_ATTRIBUTES=true` 后，span 记录 user prompt、system prompt、tool schema、model output、tool input/result 等敏感内容（明确 opt-in，默认关闭）。

**traceId/spanId 注入 debug log**（#3847）：debug log 行自动附加当前 span context，可与 OTEL 后端关联。

### 3.4 OTLP HTTP per-signal 端点路由（v0.16.0 新增）

```typescript
// 源码: telemetry/sdk.ts
// 支持 per-signal 端点覆盖（优先级：QWEN_ 变量 > OTEL_ 标准变量 > settings.json）:
// QWEN_TELEMETRY_OTLP_TRACES_ENDPOINT / OTEL_EXPORTER_OTLP_TRACES_ENDPOINT
// QWEN_TELEMETRY_OTLP_LOGS_ENDPOINT   / OTEL_EXPORTER_OTLP_LOGS_ENDPOINT
// QWEN_TELEMETRY_OTLP_METRICS_ENDPOINT/ OTEL_EXPORTER_OTLP_METRICS_ENDPOINT
//
// 自动追加 v1/traces、v1/logs、v1/metrics 路径（标准 OTLP HTTP 路由）
```

### 3.5 自定义 Resource Attributes 与 Metric Cardinality 控制（v0.16.0 新增）

```typescript
// 源码: telemetry/config.ts + telemetry/resource-attributes.ts（#4367）
// OTEL_RESOURCE_ATTRIBUTES 环境变量 或 settings.telemetry.resourceAttributes 自定义标签
// RESERVED 键（service.version, session.id）强制过滤 + 警告日志
// OTEL_SERVICE_NAME 覆盖 service.name
//
// Metric session ID cardinality 控制（#4367）：
// QWEN_TELEMETRY_METRICS_INCLUDE_SESSION_ID=true 才在 metric 标签中包含 session.id
// 默认关闭，防止高基数导致 TSDB 膨胀
```

### 3.6 RUM 事件协议

```typescript
// 源码: qwen-code/packages/core/src/telemetry/qwen-logger/event-types.ts
// RUM 事件层级:
// RumViewEvent     — 页面/视图导航
// RumActionEvent   — 用户交互
// RumResourceEvent — API/网络调用
// RumExceptionEvent — 错误
// 每类含 snapshot (JSON 序列化详细指标)
```

### 3.7 四条并行管道（澄清"默认开启"的歧义）

Qwen Code 的遥测**不是单一系统，是四条并行管道**，各有独立开关——理解这点才能正确判断"默认开不开"：

| # | 管道 | 去向 | 开关 | 默认 |
|---|---|---|---|---|
| ① | OTel Traces | OTLP(gRPC/HTTP) / 本地文件 / LogToSpan 桥 | `telemetry.enabled` (`QWEN_TELEMETRY_ENABLED`) | **OFF** |
| ② | OTel Metrics | OTLP / 文件（10s 周期）| 同上 | **OFF** |
| ③ | OTel Logs（events）| OTLP / 文件 | 同上 | **OFF** |
| ④ | **QwenLogger RUM** | **阿里云 `*.rum.aliyuncs.com`** | `getUsageStatisticsEnabled()` | **ON** |
| ⑤ | UI Telemetry | **纯本地内存**（驱动 TUI/IDE token·cost 面板）| 无（永不外传）| 本地 |

> **关键纠正**：[隐私与遥测对比](./privacy-telemetry.md) 表里 Qwen Code 标"默认开启"指的是**管道④ QwenLogger RUM**（使用统计，默认 on，发阿里云）；而 **OTel 主链（①②③）`telemetry.enabled` 默认 false**。两者是独立开关，不要混为一谈：用户不配 OTel 就完全不产生 OTel 数据，但 RUM 使用统计需单独用 `getUsageStatisticsEnabled()` / 配置关闭。

### 3.8 2026-05 演进：#3731 分层 tracing 全 phase + 跨进程 daemon trace

`#3731` 是一个宽泛的"加固 OpenTelemetry 配置"伞 issue，其下 **hierarchical session-tracing** 子线在 2026-05 密集推进（`session-tracing.ts` 已从 883 行增至 975 行）：

| Phase | PR | 状态 | 加了什么 |
|---|---|---|---|
| 1 | #4126 | ✅ | 统一 span 创建路径，扁平树→层级树（LLM/tool span 成 interaction 子节点）|
| 1.5 | #4302 | ✅ | parent fallback / abort-as-result / log-span 一致性 polish |
| 2 | #4321 | ✅ | `tool.blocked_on_user`（审批等待时长 + decision + source）+ `qwen-code.hook`（区分慢 hook vs 慢 tool）；tool span 生命周期上移到 `_schedule` 验证循环 |
| 3 | #4410 | ⏳ OPEN | `qwen-code.subagent` span + 并发隔离（前台 child 继承 T0；fork/background 用 linked root + OTel Link）|
| 4a | #4417 | ✅ | **TTFT 捕获**（首个用户可见 chunk，method-local 闭包防并发污染）+ **GenAI semconv 双发**（`gen_ai.*` 兼容层）|
| 4b | #4432 | ⏳ OPEN | per-attempt HTTP-status 重试可见性（`api.retry.count` + 每次重试独立 LLM span）|

**配套 PR**：
- **#4390** 客户端 HTTP span（`instrumentation-undici`，每出站 fetch 一个 client span，分离网络延迟 vs 模型处理）+ opt-in **W3C traceparent 传播** + OTLP 自反馈防回环 guard
- **#4499** 修 `startInteractionSpan` 漏传 parent ctx → 被铸随机 traceId 与 session 子树分裂；钉到 session root（基于 issue #4486 生产数据）
- **#4482** `LogToSpanProcessor` 桥（后端收 traces 不收 logs 时，如阿里云 ARMS）：导出失败给有用错误 + 可注入 `diagnosticsSink` 不污染 Ink TUI
- **#4565** skill-based RT 优化的**遥测地基**（接上死代码 `logSkillLaunchEvent` + `SkillLaunchEvent` 带 `prompt_id` 做 turn join），论点是"降 agent 轮数的杠杆在 skill 设计层不在框架层"

**TTFT 语义差异**（vs Claude Code）：Qwen 测"首个用户可见内容 chunk"（含 text/functionCall/thought/inlineData，排除 metadata-only），比 Claude 测 `message_start` 更普适。

**刻意规避 Claude 并发 bug**：`startToolBlockedOnUserSpan(toolSpan)` 显式传 span 对象，源码注释明说避开 Claude `sessionTracing` 的 `findLast`-by-type 反查（并发工具会错配 parent）。

### 3.9 daemon 侧遥测（两条独立线）

`qwen serve` daemon 的遥测分两条机制不同的线：

1. **OTel trace 跨进程传播**（#4556 → #4630 → #4682）：daemon HTTP 路由 → ACP bridge dispatch → ACP child 三层，**W3C traceparent 塞进 `_meta` 跨进程**串成一条 trace，前端协议零改动；prompt FIFO 排队时恢复捕获的 OTel context。最终层级：
   ```
   [Daemon] qwen-code.daemon.request（路由 span）
     └ qwen-code.daemon.bridge（prompt.dispatch）
          ▼ 跨进程 W3C traceparent in _meta
   [ACP Child] qwen-code.interaction（session.id, turn_status）
     ├ qwen-code.llm_request / qwen-code.tool → tool.execution
     └ conversation_finished
   ```
   `#4630` 给 daemon 路径的 llm_request/tool/tool.execution 补 `session.id`（ARMS 可按 session 查）；`#4682` 把覆盖扩到除 heartbeat 外所有写路由。

2. **SSE 事件协议 stamping**（#4360，F4 prereq）：`_meta.serverTimestamp` / `errorKind` / `provenance(builtin\|mcp\|subagent)` / `originatorClientId` 打进 SSE 事件 `_meta`——服务多客户端重连去重 + triage，进 **SSE 事件流给 SDK reducer**（非 OTLP 后端）。与上面 OTel 线机制不同：前者进 OTLP/ARMS，后者进 SSE 流。

> daemon 遥测设计详见 [Qwen Code Daemon 设计文档](./qwen-code-daemon-design/README.md)。

---

## 4. 隐私控制对比

| 维度 | Claude Code | Qwen Code |
|------|------------|-----------|
| **全局禁用** | `DISABLE_TELEMETRY=true` | `QWEN_TELEMETRY_ENABLED=false` |
| **非必要流量** | `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=true` | — |
| **Prompt 日志** | 禁止（类型系统强制） | `telemetryLogPromptsEnabled()` 控制 |
| **敏感 span 属性** | — | opt-in（`QWEN_TELEMETRY_INCLUDE_SENSITIVE_SPAN_ATTRIBUTES`，默认关闭） |
| **3P Provider** | 自动禁用分析 | — |
| **MDM 控制** | ✅ 策略可关闭遥测 | ❌ |
| **采样** | 按事件动态（GrowthBook） | 全量批刷 |
| **自定义资源标签** | — | ✅ `OTEL_RESOURCE_ATTRIBUTES` / `settings.telemetry.resourceAttributes`（v0.16.0） |
| **Metric session 基数** | — | `QWEN_TELEMETRY_METRICS_INCLUDE_SESSION_ID`，默认 false（v0.16.0） |

---

## 5. 关键源码文件

### Claude Code

| 文件 | 职责 |
|------|------|
| `services/analytics/index.ts` | `logEvent()` 中央入口 + 队列 |
| `services/analytics/firstPartyEventLogger.ts` | 1P 事件日志（OTLP LoggerProvider） |
| `services/analytics/growthbook.ts` | Feature Flag + 采样配置 |
| `services/analytics/config.ts` | 禁用条件判断 |
| `utils/telemetry/perfettoTracing.ts` | Perfetto Chrome Trace |

### Qwen Code

| 文件 | 职责 |
|------|------|
| `packages/core/src/telemetry/qwen-logger/qwen-logger.ts` | RUM Logger 单例 |
| `packages/core/src/telemetry/qwen-logger/event-types.ts` | RUM 事件协议 |
| `packages/core/src/telemetry/sdk.ts` | OTLP Exporter 配置（v0.16.0 新增 HTTP per-signal 路由、LogToSpanProcessor） |
| `packages/core/src/telemetry/config.ts` | 隐私控制 + 目标选择 + resource attributes 解析（v0.16.0 扩展） |
| `packages/core/src/telemetry/session-tracing.ts` | 层级 Span 树（#3731，975 行） |
| `packages/core/src/telemetry/session-context.ts` | 全局 session root OTel context（parent fallback / `/clear` 后 correlation） |
| `packages/core/src/telemetry/detailed-span-attributes.ts` | 敏感 span 属性（opt-in，含 hash 去重 + 60KB 截断） |
| `packages/core/src/telemetry/metrics.ts` | ~35 metric 定义（含 GenAI semconv 双发 + session.id cardinality 控制） |
| `packages/core/src/telemetry/log-to-span-processor.ts` | log→span 桥（后端只收 trace 时）+ 敏感属性黑名单脱敏 |

> **免责声明**: 初稿基于 2026 Q1，2026-05-22 对照 v0.16.0 复核，**2026-06-03 再次复核补全 2026-05 演进**。Claude Code v2.1.89；Qwen Code 源码 HEAD（含 #4321/#4417/#4390/#4499/#4482/#4565 已合，#4410/#4432 仍 OPEN）。
>
> 2026-05 实质变化：**#3731 分层 tracing 推进到 Phase 4a**（TTFT 捕获 + GenAI semconv 双发）、**客户端 HTTP span + W3C traceparent 传播**（#4390）、**interaction span 钉 session root 修 trace 分裂**（#4499）、**LogToSpan 桥可观测性修复**（#4482）、**daemon 跨进程 trace 传播**（#4556/#4630/#4682）、**skill-RT 遥测地基**（#4565）。详 §3.7–§3.9。
