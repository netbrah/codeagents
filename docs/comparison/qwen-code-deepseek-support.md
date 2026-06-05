# Qwen Code 对 DeepSeek 的支持深度分析

> Qwen Code 对 DeepSeek 是**生产级深度适配**，远超"加个 baseUrl"的程度——因为 DeepSeek **同时提供 OpenAI 兼容和 Anthropic 兼容两套 API**，且有大量 wire-format 怪癖，Qwen Code 对两套协议各写了一套专属 provider 适配。
>
> 来源：本地源码分析（`/root/git/qwen-code`，v0.17.0，HEAD `c699738f9`）。源码引用格式 `源码: path#L行号`。

## TL;DR

DeepSeek 是 Qwen Code 里**适配最深的第三方模型之一**：
- **双协议**：OpenAI-compat（`DeepSeekOpenAICompatibleProvider`）+ Anthropic-compat（anthropicContentGenerator 内 DeepSeek 分支）各一套
- **6 类 wire-format 怪癖处理**：content 扁平化 / `reasoning_effort` 翻译 / thinking 注入与剥离 / effort `max` 钳制 / `thinking:{type:'disabled'}` / cache_control
- **hostname 安全分离设计**：故意区分"只看 hostname"和"hostname OR model-name"两个检测，防把私有扩展误推给伪装端点，并用 `new URL()` 精确匹配防 `api.deepseek.com.evil.com` 子串欺骗
- **三种接入**：专属 provider preset / 多阿里网关（Bailian/ModelScope/IdeaLab）/ 自托管（sglang/vllm/ollama）
- **1M 上下文** + **KV cache 前缀意识**

## 一、接入路径（三种）

### ① 专属 provider preset

`源码: packages/core/src/providers/presets/deepseek.ts`

```ts
export const deepseekProvider: ProviderConfig = {
  id: 'deepseek',
  label: 'DeepSeek API Key',
  protocol: AuthType.USE_OPENAI,
  baseUrl: 'https://api.deepseek.com',
  envKey: 'DEEPSEEK_API_KEY',
  models: [
    { id: 'deepseek-v4-pro', contextWindowSize: 1000000,
      enableThinking: true, modalities: { image: true, video: true } },
    { id: 'deepseek-v4-flash', contextWindowSize: 1000000 },
  ],
  modelNamePrefix: 'DeepSeek',
  uiGroup: 'third-party',
};
```

CLI 的 AuthDialog 会落在匹配 provider `uiGroup` 的 tab（`源码: packages/cli/src/ui/auth/AuthDialog.tsx#L209`）。

### ② 经阿里网关（DeepSeek 是竞品，但通过阿里渠道可用）

| 网关 preset | DeepSeek 模型 ID |
|---|---|
| `alibaba-standard` | `deepseek-v4-pro` / `deepseek-v4-flash`（1M）|
| `alibaba-token-plan` | `deepseek-v4-pro` / `deepseek-v4-flash` / `deepseek-v3.2`（131K）|
| `idealab`（阿里内部 LLM）| `bailian/deepseek-v4-pro` / `bailian/deepseek-v4-flash` |
| `modelscope` | `deepseek-ai/DeepSeek-V4-Flash` |

### ③ 自托管

sglang / vllm / ollama 跑 DeepSeek 模型——靠 **model-name 匹配**识别（见下文检测分离）。

## 二、核心：双协议各有专属 provider

DeepSeek 同时支持两套 API，Qwen Code 各写一套适配：

| | OpenAI-compat | Anthropic-compat |
|---|---|---|
| provider 类 | `DeepSeekOpenAICompatibleProvider` | anthropicContentGenerator 内 DeepSeek 分支 |
| 源码 | `provider/deepseek.ts#L68` | `anthropicContentGenerator.ts#L491` |
| hostname 检测 | `isDeepSeekHostname`（#L36）| `isDeepSeekAnthropicHostname`（#L55）|
| 广义检测 | `isDeepSeekProvider`（#L60）| `isDeepSeekAnthropicProvider`（#L82）|

> DeepSeek 官方同时提供 `api.deepseek.com`（OpenAI 格式）和 Anthropic 兼容端点，Qwen Code 两边都适配，用户用哪个端点都能正常工作。

## 三、OpenAI 端的 wire-format 处理

`源码: packages/core/src/core/openaiContentGenerator/provider/deepseek.ts`

| 处理 | 说明 | 位置 |
|---|---|---|
| **content 扁平化** | DeepSeek 要求 message content 是**纯字符串非数组** → text part 拼接，非文本 part 替换成 `[Unsupported content type: <type>]` 占位（不静默丢弃，留文本面包屑）| `flattenContentParts` #L127 |
| **`reasoning_effort` 翻译** | 标准 `reasoning.effort` → DeepSeek 扁平的 `reasoning_effort`；low/medium → high，**xhigh → max**（max 是 DeepSeek 独有档）；否则 nested reasoning 被忽略、服务端默认 high | `translateReasoningEffort` #L176 |
| **thinking 关闭** | DeepSeek V4+ 默认 `thinking.type='enabled'`，关 reasoning 时显式发 `thinking:{type:'disabled'}`（仅 hostname-gated）| `pipeline.ts#L365` |
| **temperature: 0** | 默认生成配置 | `getDefaultGenerationConfig` |
| **reasoning content** | `ensureReasoningContentOnAssistantMessage` 保证 assistant 消息带 reasoning content | — |

`reasoning_effort` 翻译只在**真 DeepSeek hostname** 上做（避免把 DeepSeek 扩展推给名字含 deepseek 但跑严格 OpenAI-compat 后端的自托管部署）。

## 四、Anthropic 端的 wire-format 处理

`源码: packages/core/src/core/anthropicContentGenerator/anthropicContentGenerator.ts`

| 处理 | 说明 | 位置 |
|---|---|---|
| **thinking on → 注入** | 在缺 thinking 块的 tool_use turn 注入空 thinking 块，避免 DeepSeek 因 history 与 top-level `thinking` 不对齐而 **HTTP 400**（issue #3786）| `deepseekThinkingOn` #L504 |
| **thinking off → 剥离** | 从 assistant 历史 strip 掉残留 thinking 块（防 `includeThoughts:false` 时历史里有 `thought:true` 部分——suggestionGenerator / ArenaManager / forkedAgent 会触发）| `stripAssistantThinking` #L505 |
| **effort `max` 钳制** | `max` 是 DeepSeek 扩展，**非 DeepSeek 的 Anthropic provider 钳到 `high`**（真 Anthropic 只接受 low/medium/high，`max` 会 400）；用 `effortClampWarned` 闩只警告一次 | `resolveEffectiveEffort` #L627 |
| **cache_control** | per-request 采样 `enableCacheControl` / `useGlobalCacheScope` 转发给 converter（qwen-oauth 路径会热更新这俩，构造时缓存会 stale）| #L526 |

## 五、最值得注意：hostname 安全分离设计

Qwen Code 故意把 DeepSeek 检测**分成两个**（两套协议都如此）：

```
isDeepSeekHostname   ── 只看 hostname ──→ 决定 WIRE 形状（推 reasoning_effort / thinking 这些 DeepSeek 扩展）
                                          只对真 api.deepseek.com 推
isDeepSeekProvider   ── hostname OR ───→ 决定 FORMAT 约束（content 扁平化）
                       model-name        model-name fallback 覆盖自托管 sglang/vllm/ollama
```

**为什么分**（源码注释明确，引 [issue #3613](https://github.com/QwenLM/qwen-code/issues/3613)）：
- content 扁平化是**模型格式约束**——自托管跑 DeepSeek 模型也有同样约束，所以 model-name fallback 是对的
- 但 `reasoning_effort` / `thinking` 是**DeepSeek 自家 API 暴露的 wire 形状**——把它推给"名字含 deepseek 但跑在真 `api.anthropic.com` 上的克隆模型"会导致 400，所以这类决策**只信 hostname**

**SSRF / 子串欺骗防护**（`源码: provider/deepseek.ts#L36`）：

```ts
const hostname = new URL(baseUrl).hostname.toLowerCase();
return hostname === 'api.deepseek.com' || hostname.endsWith('.api.deepseek.com');
```

用 `new URL()` 解析 + 精确 hostname 匹配，**防 `https://api.deepseek.com.evil.com/v1` 这种子串欺骗**（测试里有 `deepseek.com.evil.com` 用例）；无效 URL 当作非 DeepSeek。

## 六、模型能力配置

| 配置 | DeepSeek 条目 | 源码 |
|---|---|---|
| **tokenLimits 输入** | `/^deepseek-v4/ → 1M` / `/^deepseek/ → 128K` | `tokenLimits.ts#L129-130` |
| **tokenLimits 输出** | `deepseek-v4 → 384K` / `reasoner·r1 → 64K` / `chat → 8K` | `tokenLimits.ts#L181-184` |
| **modality 默认** | `/^deepseek/ → {}`（纯文本）；但 `deepseek-v4-pro` preset 单独声明 `image+video` | `modalityDefaults.ts#L61` |
| **base URL 常量** | `DEFAULT_DEEPSEEK_BASE_URL = 'https://api.deepseek.com/v1'` | `constants.ts#L7` |

已知模型：`deepseek-v4-pro` / `deepseek-v4-flash` / `deepseek-chat` / `deepseek-reasoner` / `deepseek-r1`（含 `r1-0528`）/ `deepseek-v3` / `deepseek-v3.2` / `deepseek-distill`。

## 七、KV cache 前缀意识（DeepSeek 特有）

DeepSeek 靠**前缀 KV 缓存**降成本，Qwen Code 对此有专门意识：

- **settingsSchema 明确警告**（`源码: packages/cli/src/config/settingsSchema.ts#L1640`）：ToolSearch（按需加载 MCP 工具）会破坏 prompt 前缀稳定性 → 对 DeepSeek 这类**依赖 prefix-based KV caching** 的模型建议**关 ToolSearch**，保前缀稳定、最大化 cache 命中
- `buildReasoningConfig` 注释把 DeepSeek 列进"reasoning 碎片化矩阵"（`pipeline.ts#L450`）：`deepseek-reasoner — thinking 默认开且不能关`
- ⚠️ **[PR#4518](https://github.com/QwenLM/qwen-code/pull/4518)**（社区作者 Jerry2003826，`fix(core): stabilize DeepSeek tool cache prefix`，按 function name 排序 tools 稳定缓存前缀）—— **CLOSED 未合并**（属那批 2026-05-25 集中 burst PR）

## 八、DeepSeek 也是内部 review 工具链一员

DeepSeek 不只是用户可选 provider——`deepseek-v4-pro` 在 Qwen 团队内部的多模型 `/review` 审计管线里出现（与 mimo-v2.5-pro / glm-5.1 / gpt-5.5 并列做交叉 review，见 daemon PR 历史如 #4247 的 "DeepSeek-v4-pro 5 post-merge"）。即 DeepSeek 既是被支持的 provider，也是团队 review 工具链的一环。

## 九、与其它 provider 适配深度对比

| provider | 适配深度 | 特殊处理 |
|---|---|---|
| **DeepSeek** | **最深之一** | 双协议 + 6 类 wire 怪癖 + hostname 安全分离 + KV cache 意识 |
| Anthropic（Claude）| 深 | adaptive thinking / cache scope / 1h cache TTL |
| Qwen（自家）| 最深 | OAuth device-flow / Token Plan / fast-model |
| OpenAI / gpt-5.x | 中 | reasoning.effort 直通 |
| 通用 OpenAI-compat | 浅 | DefaultOpenAICompatibleProvider 默认行为 |

> DeepSeek 的适配深度甚至超过多数一线 provider——这反映 DeepSeek 在中文开发者生态里的实际使用量，以及它"双协议 + 多怪癖"带来的工程必要性。

## 一句话总结

Qwen Code 对 DeepSeek 是**生产级深度适配**：**双协议（OpenAI + Anthropic）各一套 provider + 6 类 wire-format 怪癖处理 + hostname 安全分离防欺骗 + 1M 上下文 + KV cache 前缀意识 + 多阿里网关接入**。最值得学习的工程亮点是 **hostname-only vs model-name 的故意分离**——既用 model-name fallback 覆盖自托管，又用 hostname-only 确保私有扩展只推给真端点，避免把 DeepSeek 扩展误推给伪装/克隆端点导致 400。

---

> **免责声明**：基于 `/root/git/qwen-code` v0.17.0（HEAD `c699738f9`）源码静态分析，截至 2026-06-04。源码行号为该版本快照，可能随版本变化。PR 状态取自 GitHub。"适配深度对比"为基于源码处理复杂度的主观归纳。
