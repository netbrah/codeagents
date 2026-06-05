# 在 Qwen Code 中使用 DeepSeek / Use DeepSeek with Qwen Code

> 本文为「面向 DeepSeek 官方文档站的集成介绍稿」初稿，供 DeepSeek 团队评审收录（如 api-docs.deepseek.com 的「集成 / Integrations」或「生态 / Ecosystem」栏目）。中英双语，配置示例均对照 Qwen Code 实际源码（v0.17.0）核对。
>
> 维护方备注：本文是外部投稿稿，可由 DeepSeek 文档团队按其文档风格改写。Qwen Code 对 DeepSeek 的工程实现细节见 [Qwen Code 对 DeepSeek 的支持深度分析](./qwen-code-deepseek-support.md)。

---

# 中文版

## Qwen Code 简介

[Qwen Code](https://github.com/QwenLM/qwen-code) 是阿里云开源的命令行 AI 编程代理（Apache-2.0，21k+ stars，TypeScript），从 Gemini CLI 分叉发展而来，支持多模型、多提供商，提供工具调用、子代理、上下文压缩、MCP 集成等完整能力。Qwen Code **原生深度支持 DeepSeek 模型**——不只是通用 OpenAI 兼容接入，而是针对 DeepSeek API 的特性做了专门适配。

## 为什么用 Qwen Code + DeepSeek

- **原生深度适配，不是泛泛兼容**：Qwen Code 同时支持 DeepSeek 的 **OpenAI 兼容**和 **Anthropic 兼容**两套 API，并针对 DeepSeek 的请求格式特性做了专门处理（推理强度 `reasoning_effort`、思维链开关、内容格式、KV 缓存前缀稳定性等）。
- **支持 DeepSeek 的全部推理档位**，包括 DeepSeek 独有的 `max` 强推理档。
- **1M 超长上下文**：Qwen Code 为 DeepSeek V4 系列配置了 1M token 的上下文窗口。
- **工具免费开源**：Qwen Code 本身 Apache-2.0 免费，DeepSeek 用户只需带上自己的 DeepSeek API Key。

## 快速开始

### 1. 获取 DeepSeek API Key

在 [DeepSeek 开放平台](https://platform.deepseek.com/) 创建 API Key，并设置为环境变量：

```bash
export DEEPSEEK_API_KEY="sk-..."
```

### 2. 在 Qwen Code 中配置 DeepSeek

在 `~/.qwen/settings.json` 的 `modelProviders` 下添加 DeepSeek（OpenAI 兼容协议）：

```jsonc
{
  "modelProviders": {
    "openai": [
      {
        "id": "deepseek-v4-pro",
        "name": "DeepSeek V4 Pro",
        "baseUrl": "https://api.deepseek.com/v1",
        "envKey": "DEEPSEEK_API_KEY",
        "generationConfig": {
          // 推理强度四档：low/medium（服务端归一为 high）| high（默认）| max（DeepSeek 强推理档）
          // 或设为 false 完全关闭推理
          "reasoning": { "effort": "max" }
        }
      },
      {
        "id": "deepseek-v4-flash",
        "name": "DeepSeek V4 Flash",
        "baseUrl": "https://api.deepseek.com/v1",
        "envKey": "DEEPSEEK_API_KEY"
      }
    ]
  }
}
```

> Qwen Code 也内置了「DeepSeek API Key」快速配置项（设置向导中选择即可，自动填好 `https://api.deepseek.com` 与 `DEEPSEEK_API_KEY`）。手动 `settings.json` 配置适合需要精细控制模型列表与推理参数的场景。

### 3. 切换到 DeepSeek 并开始使用

```bash
qwen
```

在会话中用 `/model` 命令切换到 DeepSeek 模型即可开始编码。

## DeepSeek 特性的专门适配

Qwen Code 对 DeepSeek 做了以下专门处理，用户无需手动干预：

| 特性 | Qwen Code 的处理 |
|---|---|
| **推理强度 `reasoning_effort`** | 自动把标准 `reasoning.effort` 翻译为 DeepSeek 的扁平 `reasoning_effort` 参数；`low`/`medium` → `high`，`xhigh` → `max`，与 DeepSeek 服务端的[向后兼容映射](https://api-docs.deepseek.com/zh-cn/api/create-chat-completion)一致 |
| **关闭思维链** | DeepSeek V4+ 默认 `thinking.type='enabled'`；当用户关闭推理时，Qwen Code 在 `api.deepseek.com` 上显式发送 `thinking: { type: 'disabled' }`，避免白付思维链开销 |
| **思维链历史对齐** | Anthropic 兼容端在工具调用轮自动对齐 thinking 块，避免历史不一致导致的 HTTP 400 |
| **超长上下文** | DeepSeek V4 系列配置 1M token 输入上下文 |
| **KV 缓存前缀稳定性** | Qwen Code 识别 DeepSeek 依赖前缀 KV 缓存的特性，对 DeepSeek 建议保持 prompt 前缀稳定以最大化缓存命中、降低成本 |

## 使用 DeepSeek 的 Anthropic 兼容端点（可选）

DeepSeek 也提供 Anthropic 兼容端点，Qwen Code 同样支持。把 `authType` 设为 `anthropic`、`baseUrl` 指向 DeepSeek 的 Anthropic 端点即可。此端点下 DeepSeek 独有的 `max` 推理档可原样透传。

## 支持的 DeepSeek 模型

`deepseek-v4-pro`、`deepseek-v4-flash`、`deepseek-chat`、`deepseek-reasoner`，以及通过阿里云百炼 / ModelScope 等网关提供的 DeepSeek 模型。

---

# English Version

## What is Qwen Code

[Qwen Code](https://github.com/QwenLM/qwen-code) is Alibaba Cloud's open-source command-line AI coding agent (Apache-2.0, 21k+ stars, TypeScript), forked from Gemini CLI. It supports multiple models and providers with full tool-calling, sub-agents, context compaction, and MCP integration. Qwen Code provides **first-class, deep support for DeepSeek models** — not just generic OpenAI-compatibility, but dedicated handling tuned to DeepSeek's API.

## Why Qwen Code + DeepSeek

- **Deep, dedicated integration** — Qwen Code supports both DeepSeek's **OpenAI-compatible** and **Anthropic-compatible** APIs, with handling tuned to DeepSeek's request format (`reasoning_effort`, thinking toggle, content shape, KV-cache prefix stability).
- **Full reasoning tiers**, including DeepSeek's exclusive `max` high-reasoning tier.
- **1M context** configured for the DeepSeek V4 family.
- **Free, open-source tool** — Qwen Code is Apache-2.0; DeepSeek users simply bring their own DeepSeek API key.

## Quick Start

### 1. Get a DeepSeek API key

Create a key at the [DeepSeek Platform](https://platform.deepseek.com/) and export it:

```bash
export DEEPSEEK_API_KEY="sk-..."
```

### 2. Configure DeepSeek in Qwen Code

Add DeepSeek under `modelProviders` in `~/.qwen/settings.json` (OpenAI-compatible protocol):

```jsonc
{
  "modelProviders": {
    "openai": [
      {
        "id": "deepseek-v4-pro",
        "name": "DeepSeek V4 Pro",
        "baseUrl": "https://api.deepseek.com/v1",
        "envKey": "DEEPSEEK_API_KEY",
        "generationConfig": {
          // Effort tiers: low/medium (server-mapped to high) | high (default) | max (DeepSeek's strong tier)
          // Or set false to disable reasoning entirely.
          "reasoning": { "effort": "max" }
        }
      },
      {
        "id": "deepseek-v4-flash",
        "name": "DeepSeek V4 Flash",
        "baseUrl": "https://api.deepseek.com/v1",
        "envKey": "DEEPSEEK_API_KEY"
      }
    ]
  }
}
```

> Qwen Code also ships a built-in "DeepSeek API Key" quick-setup option (select it in the setup wizard and `https://api.deepseek.com` + `DEEPSEEK_API_KEY` are filled in automatically). Manual `settings.json` is for fine-grained control of the model list and reasoning parameters.

### 3. Switch to DeepSeek and start coding

```bash
qwen
```

Use the `/model` command in-session to switch to a DeepSeek model.

## DeepSeek-specific handling (automatic)

| Feature | What Qwen Code does |
|---|---|
| **`reasoning_effort`** | Translates the standard `reasoning.effort` into DeepSeek's flat `reasoning_effort`; `low`/`medium` → `high`, `xhigh` → `max`, mirroring DeepSeek's [server-side back-compat](https://api-docs.deepseek.com/api/create-chat-completion) |
| **Disable thinking** | DeepSeek V4+ defaults to `thinking.type='enabled'`; when reasoning is off, Qwen Code explicitly sends `thinking: { type: 'disabled' }` on `api.deepseek.com` to avoid paying thinking cost |
| **Thinking-history alignment** | On the Anthropic-compatible path, thinking blocks are aligned on tool-use turns to avoid HTTP 400 from history mismatch |
| **Long context** | 1M-token input context for the DeepSeek V4 family |
| **KV-cache prefix stability** | Qwen Code is aware that DeepSeek relies on prefix-based KV caching and keeps the prompt prefix stable to maximize cache hits and reduce cost |

## Using DeepSeek's Anthropic-compatible endpoint (optional)

DeepSeek also offers an Anthropic-compatible endpoint, which Qwen Code supports. Set `authType` to `anthropic` and point `baseUrl` at DeepSeek's Anthropic endpoint; the DeepSeek-exclusive `max` reasoning tier passes through unchanged there.

## Supported DeepSeek models

`deepseek-v4-pro`, `deepseek-v4-flash`, `deepseek-chat`, `deepseek-reasoner`, plus DeepSeek models served via gateways such as Alibaba Cloud Bailian and ModelScope.

---

> **撰稿说明 / Drafting notes**：本文配置示例对照 Qwen Code v0.17.0 源码（`packages/core/src/providers/presets/deepseek.ts`、`docs/users/configuration/model-providers.md`）核对。模型名称、端点、参数以 Qwen Code 与 DeepSeek 官方文档为准；DeepSeek 文档团队收录时请按最新模型名与定价更新。截至 2026-06-04。
