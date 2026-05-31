# dreamWB

> Qwen Code 贡献者档案 · 数据截至 2026-05-31

## 速览

| 项 | 值 |
|---|---|
| GitHub | [@dreamWB](https://github.com/dreamWB) |
| 真名 / 显示名 | 未公开（GitHub profile `name` 字段为空） |
| 公司 / 组织 | 未公开（`company` 字段为空；曾向 alibaba/react-intl-universal 提交 PR，可能与阿里系生态有关联——推断） |
| 地区 | 未公开（`location` 字段为空） |
| 注册时间 | 2016-09-21（账号 ID 22347282） |
| 2026-05 qwen-code merged PR | 5（约 +3.7k/-0.1k） |
| qwen-code 累计 merged PR | 9（+6,085/-517，净约 +5.6k；全部集中在本仓库） |
| 首次 qwen-code PR | 2026-04-14 / #3249（`fix(vscode): limit session tab title length to prevent tab bar overflow`，当日开当日合） |
| 角色推断 | 终端/CLI 输入交互（readline 键位、外部编辑器、终端通知）+ VS Code 扩展 UI/UX 体验贡献者（推断） |

## 在 Qwen Code 的工作

dreamWB 的 9 个 merged PR 高度聚焦在“人机交互体验层”，几乎不碰核心 agent/模型逻辑，方向可归纳为三条主线：

- **终端 / CLI 输入交互（readline / 键位 / 外部编辑器）** —— 本月主力方向：
  - #4310 `respect /editor preference in Ctrl+X external editor`（+1,279）：让 Ctrl+X 唤起外部编辑器时尊重用户 `/editor` 偏好设置，是当月体量最大的 PR。
  - #4082 `readline Ctrl+P/N for history and selection navigation`（+1,571）：为输入框补齐 Emacs 风格 Ctrl+P/N 历史与选择项导航，体量最大的单 PR。
  - #4059 `handle MinTTY Ctrl+Backspace as delete-previous-word`（+14）：修复 Windows MinTTY 终端下 Ctrl+Backspace 的删词行为，典型的跨终端兼容细节打磨。
- **终端通知 / OSC 转义序列** ——
  - #3562 `add OSC notification support for iTerm2, Kitty, and Ghostty`（+1,027/-197）：为现代终端模拟器接入 OSC 通知能力。
- **VS Code 扩展 UI/UX** ——
  - #3661 `add tab dot indicator and notification system (#3106)`（+778）：标签页小圆点指示 + 通知系统（历经 #3657/#3659/#3660 多次重开后合入）。
  - #3477 `add native context menu copy actions for webview chat`（+421）：webview 聊天的原生右键复制菜单。
  - #3948 `mark Qwen OAuth coder-model as Discontinued in model picker`（+648）：在模型选择器里把已停用的 OAuth coder-model 标注为 Discontinued。
  - #3249（首 PR）：限制会话标签标题长度，防止标签栏溢出。
- **辅助 / 其它** ——
  - #3908 `add light theme and toggle to /export HTML`（+262）：为 `/export` 生成的 HTML 增加浅色主题与切换。

代表作（按影响与体量）：

- [#4082 readline Ctrl+P/N 历史与导航](https://github.com/QwenLM/qwen-code/pull/4082) —— 输入交互体验里最系统的一笔。
- [#4310 Ctrl+X 外部编辑器尊重 /editor 偏好](https://github.com/QwenLM/qwen-code/pull/4310) —— 任务背景中点名的当月代表作。
- [#3562 多终端 OSC 通知](https://github.com/QwenLM/qwen-code/pull/3562) —— 终端通知能力从 0 到 1。
- [#3661 VS Code 标签指示 + 通知系统](https://github.com/QwenLM/qwen-code/pull/3661) —— 扩展侧体验主线。

技术专长（推断）：终端 I/O 与跨终端兼容（readline、键位映射、OSC 转义序列、MinTTY/iTerm2/Kitty/Ghostty 差异）、VS Code 扩展前端（webview、tab UI、context menu）、前端主题/HTML 模板。整体是偏“交互与前端体验”的工程师画像，而非 agent 核心/后端方向（推断）。

## 跨仓库 / 社区足迹

- 跨仓库 PR：公开 PR 主要落在 **QwenLM/qwen-code**（含所有状态约 17 个 PR，其中 9 个 merged，其余为同一功能的多次重开/关闭，如 OSC 通知 #3546/#3554/#3560/#3561、VS Code tab #3657/#3659/#3660）；另在 **alibaba/react-intl-universal** 有 2 个公开 PR，是其唯一的外部仓库足迹。
- 自有 / fork 仓库：3 个公开仓库，**全部为 fork**——`qwen-code`（fork，用于提 PR）、`react-intl-universal`（阿里开源的 React i18n 库 fork）、`DebugHack`（基于矩阵乘法的 debug 教程，Java）；均 0 star，无原创开源项目公开。
- 个人主页 / 博客 / 社交：未公开（`blog`/`twitter_username`/`email` 字段均为空）。
- 影响力指标：followers 0 / following 0，属典型“专注提交、不经营社区”的工程师画像（推断）。
- 背景推断：向 alibaba/react-intl-universal 提交过 PR，且 qwen-code 由阿里 Qwen 团队主导，dreamWB 可能处于阿里系开源生态内（推断，证据较弱，未公开其雇主信息）。

## 备注 / 不确定项

- 任务背景给出“5 merged PR / 约 +4k”：经核对，**当月（2026-05）merged 恰为 5 个**，与背景一致；但**累计 merged 为 9 个**（含 4 月的 #3249/#3477/#3562/#3661），累计净增约 +5.6k，gross 约 +6k——背景的 +4k 更接近“当月 5 PR 的增量”量级。
- 真名、公司、地区均 **未公开**，profile 对应字段为空，未做进一步身份关联。
- OSC 通知与 VS Code tab 两个功能各自经历多次 PR 重开后才合入，统计累计 merged 时未重复计入被关闭的副本。

---
> 免责声明：本档案仅基于公开 GitHub 资料与公开 OSS 活动整理，截至 2026-05-31。所有身份/背景推断已标注“（推断）”，未公开信息标注“未公开”，不含任何私人信息。
