# yiliang114（易良）— Qwen Code 贡献者深度档案

## 速览

| 项目 | 信息 |
|------|------|
| GitHub | [@yiliang114](https://github.com/yiliang114)（user id 11473889） |
| 真名 | 易良（GitHub `name` 字段公开填写） |
| 内部花名 | 今井 |
| 公司 | Alibaba（阿里巴巴，`company` 字段公开） |
| 地区 | Hangzhou 杭州（`location` 字段公开） |
| 个人站点 | https://yiliang.site ・ Twitter [@yiliang114](https://twitter.com/yiliang114) |
| Bio | "DataWorks/AI Coding/IDE"（公开 bio） |
| GitHub 注册 | 2015-03-14 |
| 2026-05 PR（qwen-code 累计） | **115 merged**（非任务前提的 37；前提数据已被推翻） |
| 全部状态 | 141 PR：115 merged / 17 closed / 9 open |
| 代表作合计规模 | 仅前 8 个代表 PR 即 ≈ +63k/-7k 行（VSCode 扩展首版单 PR 即 +21.7k） |
| 全 GitHub merged PR | 227（跨仓库，见下） |
| 首次 qwen-code PR | 创建 2025-11-17（#1056）；首个 merged 2025-12-10（#1059 VSCode Extension Implementation） |
| 最近 merged | 2026-05-29（#3826） |
| 角色 | **VSCode IDE Companion / WebUI / LSP 集成负责人**（CODEOWNER），兼任 CLI 核心稳定性 + 发布工程；Qwen Code 内部团队成员（阿里生态），内部花名"今井" |

> **重要更正**：本档案调查所得数据与下发任务的已知前提（"37 merged PR、方向为 standalone installer + CI/release + weixin channel"）不符。实际公开数据为 **115 merged PR**，且主线方向是 **VSCode IDE Companion 扩展、WebUI、LSP 集成**。Installer / weixin / release 工程只是其工作的一部分，非主轴。下文以实际公开数据为准。

## 在 Qwen Code 的工作

易良是 qwen-code 的 **COLLABORATOR**（PR `author_association` 字段公开显示）。PR #2312
"chore: add yiliang114 as code owner for vscode-ide-companion and webui" 公开记录他被加为
`vscode-ide-companion` 与 `webui` 两个包的 CODEOWNER，这与其 115 个 merged PR 的主题分布高度一致。

### 方向一：VSCode IDE Companion 扩展（主轴，推断为该模块主要作者）

从零搭建并长期维护 qwen-code 的 VSCode 扩展，是其贡献量最大的方向：

- [#1059 feat: VSCode Extension Implementation](https://github.com/QwenLM/qwen-code/pull/1059)（**+21,769/-93**）— VSCode 扩展首版落地，单 PR 体量最大的代表作
- [#2188 feat(vscode-ide-companion): add sidebar view and multi-position chat layout](https://github.com/QwenLM/qwen-code/pull/2188)（+1,177/-155）— 侧边栏视图 + 多位置聊天布局
- [#3398 feat(vscode): replace OAuth with Coding Plan / API Key provider setup](https://github.com/QwenLM/qwen-code/pull/3398)（+2,277/-275）— 鉴权方式从 OAuth 改为 Coding Plan / API Key
- [#1582 Support model selection through ACP in vscode ide companion](https://github.com/QwenLM/qwen-code/pull/1582)、[#2551 enable Plan Mode toggle and approval UI](https://github.com/QwenLM/qwen-code/pull/2551)、[#2548 expose /skills as slash command](https://github.com/QwenLM/qwen-code/pull/2548)、[#2593 support /insight command](https://github.com/QwenLM/qwen-code/pull/2593)、[#2592 support /export session command](https://github.com/QwenLM/qwen-code/pull/2592) — 把 CLI 侧 ACP 能力（模型选择、Plan Mode、Skills、Insight、Export）逐步搬进 IDE
- [#1978 add image paste support](https://github.com/QwenLM/qwen-code/pull/1978)、[#1460 Support Jupyter Notebook (.ipynb) File Code Selection](https://github.com/QwenLM/qwen-code/pull/1460) — 多模态/编辑器集成
- 大量 IDE 集成稳定性修复：#2959（VS Code 0.14.1 webview 白屏）、#1474（跨平台 CLI 终端执行）、#3450（split stream 消息顺序）、#1262（vscode engine 兼容 ^1.85.0）等

### 方向二：WebUI 统一架构

- [#1543 feat(webui): implement unified UI architecture with shared component library](https://github.com/QwenLM/qwen-code/pull/1543)（**+17,261/-4,012**）— WebUI 统一架构 + 共享组件库，第二大代表作
- [#2595 feat(webui): unify remaining tool display labels](https://github.com/QwenLM/qwen-code/pull/2595) — 工具显示标签统一

### 方向三：LSP / 代码智能

- [#1401 Add experimental LSP support for code intelligence](https://github.com/QwenLM/qwen-code/pull/1401)（+8,531/-136）— 实验性 LSP 支持
- [#3649 fix(lsp): expose status and startup diagnostics](https://github.com/QwenLM/qwen-code/pull/3649)、[#2547 improve C++/Java/Python language server support](https://github.com/QwenLM/qwen-code/pull/2547)、[#3615 修复 LSP 文档、isPathSafe 限制并提升 LSP 工具调用率](https://github.com/QwenLM/qwen-code/pull/3615)

### 方向四：Standalone Installer + 发布工程（任务前提涉及的方向，实为次要）

- [#3828 feat(installer): add standalone hosted install and uninstall flow](https://github.com/QwenLM/qwen-code/pull/3828)（+8,174/-303）— 独立宿主安装/卸载流程
- [#3776 feat(installer): add standalone archive installation](https://github.com/QwenLM/qwen-code/pull/3776)（+3,931/-800）、[#3853 stage hosted install assets](https://github.com/QwenLM/qwen-code/pull/3853)
- 发布流水线与 CI：多个 `chore(release): bump version`（0.14.x/0.15.x/0.16.0）、#4492（拆分 Aliyun OSS sync 为独立 post-release workflow）、#3854（issue follow-up bot workflow）、#4383/#4398/#4401（修 TS5055 release build 失败、TDZ error）、#3132（parallelize release validation）

### 方向五：CLI / Core 稳定性

- [#4286 replace structuredClone with shallow copy to prevent OOM in long sessions](https://github.com/QwenLM/qwen-code/pull/4286)、[#4186 add heap-pressure auto-compaction safety net](https://github.com/QwenLM/qwen-code/pull/4186) — 长会话内存压力治理
- 重试/限流：#3850（分类 retry errors）、#3790（stream rate-limit retry）、#1995（rate limit error code 1305）、#2126/#2125（独立重试预算、防重复 function-call yield）
- 斜杠命令体验：#3963（校验 /model 参数）、#3743/#3104（防止文件路径被当作 slash command）、#3826（追踪 model-sent slash 命令历史）

### 方向六：weixin / channel（任务前提涉及，零星）

- [#4465 fix(weixin): allow Windows image paths inside workspace](https://github.com/QwenLM/qwen-code/pull/4465)、[#4464 fix(weixin): send decryptable image payloads](https://github.com/QwenLM/qwen-code/pull/4464)、[#3294 fix(channels/dingtalk): prioritize senderStaffId](https://github.com/QwenLM/qwen-code/pull/3294)

### 专长推断

- **IDE / 编辑器扩展工程**（推断）：VSCode 扩展架构、ACP 协议对接、webview UI、LSP 集成是其最深的技术领域。
- **前端/全栈**（推断）：WebUI 共享组件库、Vue 生态（见跨仓库足迹），与 bio "IDE" 自述吻合。
- **Node/TS 构建与发布工程**（推断）：standalone installer、release CI、版本管理、依赖升级（如 express 4→5）。

## 跨仓库社区足迹

全 GitHub merged PR 227 个。除 qwen-code（141 PR）外，公开 PR 分布显示其长期活跃于编辑器/IDE 与前端开源生态：

| 仓库 | PR 数 | 说明 |
|------|------|------|
| QwenLM/qwen-code | 141 | 本档案主体 |
| microsoft/vscode | 38 | VSCode 上游贡献（与其 IDE 专长一致） |
| Tencent/omi | 13 | 腾讯 Omi 前端框架 |
| yiliang114/qwen-code、yiliang114/projj、yiliang114/config 等 | 多个 | 个人 fork / 配置仓库 |
| eclipse-theia/theia | 7 | Theia IDE |
| opensumi/core、opensumi/ide-startup-lite | 多个 | OpenSumi（阿里系 IDE 框架） |
| vuejs/core、vuejs/router、vueComponent/ant-design-vue、frejs/fre、web-infra-dev/rspack、xtermjs/xterm.js、youzan/vant 等 | 各 1 | Vue / 前端基础设施生态零星贡献 |
| Tencent/cloudbase-framework、TencentCloudBase/cloudbase-templates、Tencent/kbone | 多个 | 腾讯云开发生态 |

个人代表仓库：`vue-thanos-snap`（115★，Vue 灭霸打响指动画）、`generate-vuex`、`Blog`（记录成长）。
整体足迹印证 **IDE 框架 + Vue/前端基础设施** 的双重背景（推断），与其在 qwen-code 承担 VSCode 扩展 + WebUI 的角色一脉相承。

## 备注

- 任务下发的"已知前提"（37 PR、installer/release/weixin 为主方向）与公开数据不符，本档案已据实更正为 **115 merged PR、VSCode IDE Companion/WebUI/LSP 为主轴**。
- 公司"Alibaba"、地区"Hangzhou"、真名"易良"、bio"DataWorks/AI Coding/IDE"均为其 GitHub 资料公开字段，非私人挖掘。
- bio 中的 "DataWorks" 指阿里云数据开发平台，推测为其本职所在团队（推断，未在 qwen-code 工作中直接体现）。
- email/QQ 等联系字段虽在 API 公开返回，但属个人联系方式，本档案不予收录。

> **免责声明**：本档案截至 2026-05-31，所有数据来自 GitHub 公开 API（用户资料、PR 列表、search/issues、repos）。标注"（推断）"者为基于公开 PR 主题与资料字段的合理推测，非事实陈述；不含任何非公开个人信息。数据可能随时间变化而过时。
