# Qoder CLI 文档

> QoderAI（阿里巴巴通义灵码系）闭源终端 AI 编程代理
>
> ⚠️ **2026-06 重大变化**：Qoder CLI **v1.0 已从 Go 二进制完全重写为 JS bundle，基座是 Gemini CLI**（与 Qwen Code 是 Gemini CLI 的兄弟 fork，但**不是** Qwen Code 的 fork——bundle 保留 Gemini 原版 `geminiIgnoreFilter`/`GEMINI.md`，Qwen 专属标记全 0）。v0.x 时代的 Go 二进制架构（02-architecture / EVIDENCE 旧内容）已不再适用。详见 [Qwen Code vs Qoder CLI 对比](../../comparison/qwen-code-vs-qoder-cli.md)。

| 文档 | 内容 |
|------|------|
| [01-概述](./01-overview.md) | 核心功能、CLI 命令、斜杠命令、模型、定价 |
| [02-architecture](./02-architecture.md) | ⚠️ v0.x Go 二进制架构（v1.0 已重写为 Gemini CLI fork，待刷新）|
| [EVIDENCE.md](./EVIDENCE.md) | 二进制分析证据（含 v1.0.18 三方三角验证复核）|

**npm 包：** `@qoder-ai/qodercli` **v1.0.18**（2026-06-11，最新）
**血脉：** v1.0 = **Gemini CLI fork**（与 Qwen Code 是兄弟 fork，非 Qwen Code fork）；v0.x 曾是独立 Go 二进制
**官网：** [qoder.com/cli](https://qoder.com/cli)（国内 [qoder.com.cn](https://qoder.com.cn)）
**文档：** [docs.qoder.com](https://docs.qoder.com)
