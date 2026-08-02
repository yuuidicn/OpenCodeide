# 网页版 OpenCode IDE —— 完整开发文档

> 版本：v0.1（初稿，待评审）
> 技术主栈：Rust
> 目标：一套代码/一套后端，支撑 **Web 端 + 桌面端 + 安卓端 + 苹果端** 四端互通；统一管理后台；前台界面、排版、配色、右键菜单与 **VSCode 完全一致**；编辑、工作流、技能安装、插件安装等功能与 **OpenCode 一致**。

---

## 0. 文档说明与阅读指引

本文档面向产品、架构、前端、后端、移动端、测试、运维全体角色，作为项目立项与开发的总纲。文档结构：

1. 产品定位与范围
2. 术语表
3. 总体架构
4. 技术选型（Rust 为核心）
5. 多端方案（Web / 桌面 / 安卓 / 苹果）
6. 前台编辑器（对齐 VSCode）
7. 核心功能（对齐 OpenCode）
8. 管理后台
9. 国际化（i18n）多语言
10. 数据模型与存储
11. 后端服务与 API 设计
12. 实时协作与同步
13. 安全与权限
14. 部署与运维（DevOps）
15. 测试策略
16. 里程碑与排期
17. 风险与开放问题

---

## 1. 产品定位与范围

### 1.1 一句话定位
一个基于 Rust 构建的、跨四端互通的云端 AI 编程 IDE：前台体验完全对标 VSCode，AI/工作流/技能/插件能力完全对标 OpenCode，并提供统一的管理后台。

### 1.2 目标用户
- 个人开发者：随时随地（浏览器/手机/桌面）打开项目继续编码。
- 团队/企业：统一后台管理成员、项目、模型密钥、插件白名单、用量与计费。
- 教育/培训：多语言界面，低门槛在线编程环境。

### 1.3 产品范围（本期）
- **必须（P0）**：Web 端 IDE、后端服务、管理后台、账号与项目管理、代码编辑（语法高亮/LSP）、AI 对话与代码生成、技能安装、插件安装、i18n 多语言、桌面端。
- **重要（P1）**：安卓端、苹果端、多端实时同步、工作流编排、终端。
- **可选（P2）**：多人实时协作（CRDT）、市场（技能/插件商店）、计费。

### 1.4 非目标（本期不做）
- 不自研编译器/语言运行时（复用容器内工具链）。
- 不做本地离线大模型训练。

---

## 2. 术语表

| 术语 | 说明 |
|---|---|
| OpenCode | 开源 AI 编码 Agent 项目，本产品在功能上对齐其 Agent/工作流/工具（tool）/技能/插件体系 |
| Skill（技能） | 可安装的、封装某类任务流程的能力包（含提示词、脚本、工具声明） |
| Plugin（插件） | 扩展 IDE/Agent 能力的组件（编辑器扩展、LSP、主题、命令等） |
| Workspace（工作区） | 一个用户/项目的文件与运行环境的集合 |
| LSP | Language Server Protocol，提供补全/跳转/诊断/高亮 |
| Session（会话） | 一次 AI Agent 的交互上下文 |
| Runner/Workspace Pod | 运行用户代码、终端、LSP 的隔离容器/沙箱 |

---

## 3. 总体架构

### 3.1 架构分层

```
┌──────────────────────────────────────────────────────────────┐
│                        客户端（四端）                            │
│  Web(浏览器)   桌面(Tauri)   安卓(Tauri Mobile)  苹果(Tauri iOS) │
│  ── 共享同一套前端 UI（Web 技术）+ Rust 内核 ──                    │
└───────────────┬──────────────────────────────────────────────┘
                │  HTTPS / WebSocket (gRPC-web 可选)
┌───────────────▼──────────────────────────────────────────────┐
│                      API 网关 / BFF（Rust: axum）               │
│   鉴权、限流、路由、协议转换、i18n 资源分发                        │
└───┬───────────┬────────────┬─────────────┬───────────────────┘
    │           │            │             │
┌───▼───┐  ┌────▼────┐  ┌────▼─────┐  ┌────▼──────┐
│ 账号  │  │ 项目/文件 │  │ AI Agent │  │ 技能/插件  │
│ 服务  │  │  服务    │  │  服务    │  │  市场服务  │
└───┬───┘  └────┬────┘  └────┬─────┘  └────┬──────┘
    │           │            │             │
┌───▼───────────▼────────────▼─────────────▼───────────────────┐
│  数据层：PostgreSQL + Redis + 对象存储(S3/MinIO) + 向量库       │
└──────────────────────────────────────────────────────────────┘
                │
┌───────────────▼──────────────────────────────────────────────┐
│  运行时层：Workspace Pod（容器沙箱）：终端 / LSP / 代码执行       │
└──────────────────────────────────────────────────────────────┘
```

### 3.2 关键设计原则
- **单一后端**：四端共用同一套 REST/WebSocket API 与管理后台。
- **前端复用**：一套 Web 前端（TypeScript + Monaco），通过 Tauri 打包到桌面/移动端，最大化复用。
- **Rust 内核复用**：把可共享的核心逻辑（协议、同步、文件模型、Agent 客户端）写成 Rust crate，Web 端用 WASM，桌面/移动端用原生绑定。
- **隔离与安全**：每个用户运行环境跑在独立沙箱容器，网络与资源受限。

### 3.3 Monorepo 目录结构（建议）

```
opencode-ide/
├── crates/                     # Rust 工作区
│   ├── core/                   # 共享内核：文件模型、CRDT、协议类型
│   ├── core-wasm/              # 编译到 WASM 供 Web 使用
│   ├── api-gateway/            # axum BFF/网关
│   ├── svc-account/            # 账号服务
│   ├── svc-project/            # 项目/文件服务
│   ├── svc-agent/              # AI Agent 服务
│   ├── svc-market/             # 技能/插件市场
│   ├── runner/                 # Workspace Pod 代理（终端/LSP/执行）
│   └── admin-api/              # 管理后台 API
├── apps/
│   ├── web/                    # Web 前端（Vite + React/Svelte + Monaco）
│   ├── desktop/                # Tauri 桌面壳
│   ├── mobile-android/         # Tauri Android
│   ├── mobile-ios/             # Tauri iOS
│   └── admin/                  # 管理后台前端
├── packages/                   # 前端共享 TS 包（UI 组件、i18n、API SDK）
├── locales/                    # 多语言资源（en, zh-CN, zh-TW, ja, ko, ...）
├── deploy/                     # k8s / docker-compose / terraform
└── docs/                       # 文档
```

---

## 4. 技术选型（以 Rust 为核心）

### 4.1 后端（Rust）
| 领域 | 选型 | 说明 |
|---|---|---|
| Web 框架 | **axum**（+ tower） | 高性能、生态好、与 tokio 深度集成 |
| 异步运行时 | **tokio** | 事实标准 |
| ORM/DB | **SQLx** 或 **SeaORM** | 编译期校验 SQL / 全功能 ORM |
| WebSocket | axum + tokio-tungstenite | 实时同步、终端流 |
| gRPC（服务间） | **tonic** | 内部服务通信 |
| 序列化 | **serde** / **prost** | JSON / protobuf |
| 鉴权 | **jsonwebtoken** + argon2 | JWT + 密码哈希 |
| 缓存/队列 | Redis（`redis-rs`） | 会话、限流、pub/sub |
| 向量检索 | Qdrant / pgvector | 代码语义检索、RAG |
| 可观测 | **tracing** + OpenTelemetry | 日志/链路 |

### 4.2 前端（四端共享）
| 领域 | 选型 | 说明 |
|---|---|---|
| 编辑器 | **Monaco Editor** | VSCode 同源编辑器，天然一致的高亮/右键/命令面板 |
| UI 框架 | React 18 或 Svelte | 组件化，配合 VSCode 主题 token |
| 语法高亮 | Monaco 内置 + **TextMate 语法**（vscode-textmate + WASM oniguruma） | 与 VSCode 完全一致的高亮 |
| 图标/主题 | VSCode 官方图标主题 + Dark+/Light+ 配色 | 视觉 1:1 |
| 状态管理 | Zustand / Redux Toolkit | |
| 构建 | Vite | |
| i18n | i18next / FormatJS | 运行时切换语言 |

### 4.3 跨端打包
| 端 | 方案 |
|---|---|
| Web | 直接部署静态资源 + CDN |
| 桌面（Win/macOS/Linux） | **Tauri 2**（Rust 壳 + WebView），体积小、可调用本地能力 |
| 安卓 | **Tauri 2 Mobile（Android）** |
| 苹果（iOS/iPadOS） | **Tauri 2 Mobile（iOS）** |

> 选 Tauri 的理由：Rust 原生壳，可复用 `crates/core`；一套前端 UI 覆盖四端；相比 Electron 体积与内存更优；移动端与桌面端共享代码。

---

## 5. 多端方案

### 5.1 复用策略
- **UI 层**：一套 Web 前端代码（`apps/web`），桌面/移动端通过 Tauri 加载同一套构建产物。
- **能力桥接**：Tauri `invoke` 暴露本地能力（文件、系统菜单、通知、深链）。Web 端对应能力降级为云端实现。
- **内核**：`crates/core` 编译为 WASM（Web）与原生库（Tauri），保证同步/协议逻辑一致。

### 5.2 各端差异与适配
| 能力 | Web | 桌面 | 安卓 | 苹果 |
|---|---|---|---|---|
| 本地文件访问 | 受限（File System Access API） | 完整 | 受限（SAF） | 受限（沙盒） |
| 终端 | 云端 pod | 本地或云端 | 云端 | 云端 |
| 右键菜单 | Monaco 上下文菜单 | 原生+Monaco | 长按菜单适配 | 长按菜单适配 |
| 推送通知 | Web Push | 系统通知 | FCM | APNs |
| 触屏优化 | — | — | 需要 | 需要 |

### 5.3 多端互通（同一账号 / 同一工作区）
- 账号：统一 OAuth/JWT，任一端登录后同步会话。
- 工作区：文件与状态存云端，四端打开同一 workspace 看到相同内容。
- 实时：WebSocket 广播文件变更、光标、Agent 会话进度，跨端秒级同步（见 §12）。

### 5.4 TUI 终端客户端（第五端）
除四个图形端外，提供 **TUI 命令行客户端**（对齐 OpenCode 的 `opencode` 命令）：
- **用途**：在 SSH 服务器、Docker 容器、无桌面环境中直接在终端使用 AI 编码 Agent；键盘为主、极轻量、启动快。
- **实现**：Rust（如 `ratatui` + `crossterm`），复用同一套 **SDK/Protocol + Server**，行为与图形端一致。
- **能力**：会话对话、工具调用、技能/命令、Agent 切换（build/plan/@subagent）、diff 预览与应用、权限确认。
- **发行**：单文件二进制，`opencode` 命令，支持包管理器安装（对齐 OpenCode 安装体验）。

---

## 6. 前台编辑器（对齐 VSCode）

> **界面基准**：前台排版、布局、配色、右键菜单、快捷键均以官方 [microsoft/vscode](https://github.com/microsoft/vscode) 为准逐项复刻。

### 6.1 视觉与排版一致性
- 使用 **Monaco Editor**（VSCode 的编辑器内核），天然保证：代码实时渲染、缩进参考线、行号、minimap、括号匹配、折叠。
- 采用 VSCode 官方 **Dark+ / Light+ / High Contrast** 主题 token；布局栅格（活动栏 Activity Bar、侧边栏 Side Bar、编辑区、面板 Panel、状态栏 Status Bar）像素级复刻。
- 图标使用 VSCode Seti/官方文件图标主题。

### 6.1.1 浏览器打开本地文件夹（重点）
- Web 端支持**直接打开本地文件夹**并实时读写，无需上传到云端：
  - 基于 **File System Access API**（`showDirectoryPicker` / `FileSystemDirectoryHandle`），获取目录句柄后在浏览器内构建文件树、打开/保存文件、监听变更。
  - 句柄可持久化（IndexedDB），下次打开一键恢复授权目录；写入前请求持久化写权限。
  - 兼容性降级：不支持该 API 的浏览器回退到「拖拽文件夹 / 多选文件」+ 云端工作区模式。
  - 桌面/移动端（Tauri）走原生文件系统，能力更完整（本地终端、任意路径）。
- 与云端工作区并存：用户可在「本地文件夹」与「云端 Workspace」间切换，同一套编辑器 UI。

### 6.2 自动识别语言与高亮
- 依据文件扩展名 + shebang + 内容启发式自动识别语言。
- 高亮采用 Monaco 内置 tokenizer；对需要与 VSCode 100% 一致的场景，加载 **TextMate 语法（.tmLanguage）** + `vscode-oniguruma`（WASM），复用 VSCode 的语法定义。
- 语义高亮由 LSP 提供（Semantic Tokens）。

### 6.2.1 前台布局分区（对齐 VSCode，AI 独立侧栏）
整体沿用 VSCode 布局，**AI 回复放在右侧独立面板，编辑器工作区只显示代码**：

```
┌──────────────────────────────────────────────────────────────────────┐
│ 标题栏 Title Bar / 菜单                                                  │
├──┬───────────────┬─────────────────────────────────┬─────────────────┤
│活│ 侧边栏 Sidebar │        编辑器工作区 Editor        │  AI 面板(右侧)   │
│动│ (资源管理器/   │  ┌──────┬──────┐                 │ Secondary Side  │
│栏│  搜索/源代码/  │  │ Tab1 │ Tab2 │  代码分屏/多标签  │  ── AI 对话/    │
│Ac│  扩展/技能/    │  ├──────┴──────┴───────────────┐ │    回复流 ──     │
│t │  插件)        │  │  Monaco 代码（实时高亮）      │ │  · 会话历史      │
│iv│               │  │                             │ │  · 工具调用      │
│ty│               │  └─────────────────────────────┘ │  · 差异/应用     │
│  │               │                                  │  · Agent 切换    │
├──┴───────────────┴──────────────────────────────────┴─────────────────┤
│ 面板 Panel：终端 / 问题 / 输出 / 调试                                    │
├────────────────────────────────────────────────────────────────────────┤
│ 状态栏 Status Bar：分支 / 语言 / 光标 / 编码 / 当前 Agent                 │
└────────────────────────────────────────────────────────────────────────┘
```
- **AI 面板独立于工作区**：位于右侧 Secondary Sidebar，可显示/隐藏、可拖拽宽度；不侵占编辑器区域。
- AI 面板内容：会话流（提问/回复）、工具调用可视化、代码差异（diff）与「应用到文件」、Agent 切换（build/plan/@subagent）、会话历史。
- 编辑器工作区保持纯代码视图；AI 生成的改动以 diff 形式在编辑器内高亮，用户确认后应用。

### 6.3 与 VSCode 一致的交互
- **右键上下文菜单**：复刻 VSCode 菜单项（转到定义、查找所有引用、重命名符号、格式化文档、剪切/复制/粘贴、命令面板等）。
- **命令面板**（Ctrl/Cmd+Shift+P）、快捷键绑定与 VSCode 默认一致。
- **多标签、分屏、拖拽、面包屑导航、搜索替换（含正则）**。
- **集成终端**面板、问题面板（Problems）、输出面板；**运行/任务**面板（不做断点调试/DAP）。

### 6.4 语言智能（LSP）
- 后端在 Workspace Pod 内启动对应语言的 Language Server（rust-analyzer、tsserver、pyright、gopls 等）。
- 前端 Monaco 通过 `monaco-languageclient` + LSP over WebSocket 连接，获得补全、诊断、跳转、hover、重命名、格式化。

---

## 7. 核心功能（对齐 OpenCode）

> **功能基准**：以官方仓库 [anomalyco/opencode](https://github.com/anomalyco/opencode) 为准逐项对齐。
> OpenCode 本身是 **TypeScript/Bun monorepo**，采用「**无头服务端（server）+ 多客户端（TUI / 桌面 / Web）**」架构，通过 SDK/Protocol 通信。本产品在功能与配置格式上对齐它，并在其基础上补齐 VSCode 式图形编辑器与统一管理后台。

### 7.0 OpenCode 真实架构参考（用于对齐）
OpenCode 关键包（`packages/`）：
- `core`：Agent 内核，基于 **Vercel AI SDK** 接入大量 provider（openai / anthropic / google / bedrock / azure / groq / mistral / perplexity / alibaba / openai-compatible 等）。
- `server`：无头服务端（drizzle-orm + effect）。
- `sdk` / `protocol` / `schema`：客户端 SDK 与协议/数据结构定义。
- `tui`：终端 UI（@opentui/solid）。
- `desktop`：桌面客户端（官方用 Electron）。
- `app` / `web` / `session-ui` / `ui`：图形界面（SolidJS，代码高亮用 **Shiki/TextMate**）。
- `plugin`：插件系统（zod 定义）。
- 集成：`slack`、`github`、`enterprise`、`containers`。

> 本产品的差异化取舍：**服务端与客户端改用 Rust 实现（axum 等）**，图形前台改用 **Monaco**（更贴近 VSCode），桌面/移动端改用 **Tauri**（体积更小）。功能语义、配置文件格式、技能/插件/命令/主题的组织方式与 OpenCode 对齐，力求可迁移。

### 7.1 AI Agent（对齐 build / plan / general）
- **内置 Agent**：`build`（默认，完整读写权限）、`plan`（只读，默认禁止改文件、执行 bash 前询问，用于分析/探索）；`Tab` 键切换。
- **子 Agent（subagent）**：内置 `general`（复杂搜索/多步任务），消息中用 `@general` 调用；支持自定义子 Agent。
- **多模型/多 provider**：通过 AI SDK 接入各大厂商与 OpenAI 兼容/本地模型；密钥在管理后台配置。
- **工具调用（tools）**：读写文件、执行 bash/终端、搜索代码（grep/glob）、运行测试、抓取 URL 等，与 OpenCode 工具集对齐；工具可在配置中按需启用/禁用。
- **上下文**：仓库索引 + 语义检索（RAG）+ `references`（外部资料引用）+ `AGENTS.md` 规则文件。
- **权限系统（permission）**：对文件编辑、bash 执行等按 Agent/操作粒度配置「允许 / 询问 / 拒绝」。

### 7.2 命令与自动工作流（Commands / Workflow）
- **自定义命令（对齐 `.opencode/command/*.md`）**：Markdown + frontmatter（`description`、`model`、`subtask` 等）定义可复用命令（如 commit、changelog、translate、spellcheck）。
- **自动工作流**：多步骤编排（读需求 → 改代码 → 跑测试 → 修复 → 提交 PR）；触发方式：手动 / 定时 / 事件（push、issue、Slack）。
- **可视化编排** + 文本定义（存版本库），可复用命令与子 Agent 作为步骤。

### 7.3 技能安装（Skills，对齐 `.opencode/skills/<name>/SKILL.md`）
- **技能格式**：文件夹内 `SKILL.md`，frontmatter 含 `name`、`description`，正文为流程说明/指令；可附脚本与引用。
- 从「技能市场」浏览、搜索、一键安装 / 卸载 / 更新；支持项目级与用户级安装。
- 安装后 Agent 会话可自动/按需调用；兼容 OpenCode 技能目录结构，便于迁移。

### 7.4 插件安装（Plugins，对齐 `.opencode/plugins` + `@opencode-ai/plugin`）
- **两类扩展**：
  - **OpenCode 插件**：TS/JSON 插件（zod schema），可扩展 Agent 工具、命令、事件钩子、主题。
  - **编辑器扩展**：为贴近 VSCode 体验，另兼容 **Open VSX** 的 VSCode 扩展子集（主题、语法、部分命令；受沙箱与 API 限制）。
- **自定义工具（对齐 `.opencode/tool/*.ts`）**：项目内声明自定义工具供 Agent 调用。
- 一键安装 / 启用 / 禁用 / 卸载；权限声明 + 沙箱执行 + 后台白名单。

### 7.5 主题（Themes，对齐 `.opencode/themes/*.json`）
- 支持 OpenCode 主题 JSON 与 VSCode 主题；后台可设默认主题，用户可自定义。

### 7.6 配置文件兼容（对齐 `opencode.json` / `opencode.jsonc`）
兼容 OpenCode 配置结构，字段包括：
```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {},      // 模型 provider 与密钥/参数
  "permission": {},    // 权限（编辑/bash 等：allow|ask|deny）
  "references": {},    // 外部资料引用（repo/path + description）
  "mcp": {},           // MCP 服务器（工具扩展）
  "tools": {},         // 工具启用/禁用开关
  "agent": {},         // 自定义 agent
  "command": {},       // 自定义命令
  "theme": ""          // 主题
}
```
- **MCP 支持**：可接入 Model Context Protocol 服务器，动态扩展工具（与 OpenCode 一致）。
- 分层配置：全局（用户）→ 组织 → 项目（`.opencode/`）逐级覆盖，管理后台可下发默认值。

### 7.7 终端与执行
- 集成终端连接 Workspace Pod（PTY over WebSocket）。
- 代码运行/调试在沙箱内进行，资源与网络受限。

### 7.8 集成（对齐 Slack / GitHub / 企业版）
- **GitHub**：issue/PR 触发、自动改代码提 PR、PR 搜索/三连（triage）等（对齐 `.opencode/tool/github-pr-search.ts` 与 `github` 包）。
- **Slack**：在 Slack 中发起/接收 Agent 任务与结果。
- **企业/自托管**：对齐 `enterprise` 能力，支持私有化部署与团队治理。

### 7.9 客户端接口对齐（SDK / Server）
- 提供无头 **Server** + **SDK/Protocol**，第三方可像 OpenCode 一样以 SDK 驱动 Agent；本产品的 Web/桌面/移动/TUI 均为该 Server 的客户端，保证四端行为一致。

### 7.10 模型接口容错与自动重连（与 OpenCode 一致）
> 严格对齐 OpenCode 源码的实现（`packages/core/src/util/retry.ts`、`aisdk.ts`、`session/runner/llm.ts`），行为与参数保持一致，并在 Rust 侧等价实现。

**a) 瞬时错误自动重试（对齐 `util/retry.ts`）**
- 默认 **attempts = 3**、初始 **delay = 500ms**、**factor = 2**（指数退避）、**maxDelay = 10000ms**；均可配置。
- **仅对瞬时错误重试**，判定依据错误消息（不区分大小写）包含以下之一：
  `load failed`、`network connection was lost`、`network request failed`、`failed to fetch`、`econnreset`、`econnrefused`、`etimedout`、`socket hang up`。
- 非瞬时错误（如密钥无效、4xx 参数错误）**不重试**，直接抛出。

**b) 流式读取超时中止（对齐 `aisdk.ts`）**
- 对模型 SSE 流做 **chunk/read 超时**：单个数据块在超时时间内未到达即 `abort` 并取消读取（防止「假死」挂起），随后由 (a) 的重试逻辑决定是否重试。
- 支持整体请求 **timeout**（`AbortSignal.timeout`）与自定义 `fetch`，与 OpenCode 一致。

**c) AI SDK 自身重试**
- 复用底层 provider SDK 的 `maxRetries` 机制（与 OpenCode 一致），与 (a) 分层协作。

**d) provider 错误的处理（对齐 `session/runner/llm.ts`）**
- 流中出现 `providerError` 事件时按事件流处理；上下文溢出（context overflow）触发**自动压缩后重试**该轮（`compactAfterOverflow` → 重跑 turn）。
- 失败最终不可恢复时，标记 `failAssistant` 并结束该轮，前台显示错误，未结算的工具调用置为失败。

> 说明：OpenCode 对**进行中的单次流式响应不做「断点续接」**，而是「中止 + 按瞬时错误整轮重试」；本产品与之一致（不虚构断点续传）。前台 AI 面板在重试期间显示「重连中…」，并提供手动「重试」。

---

## 8. 管理后台

统一后台管理四端与所有 OpenCode IDE 相关设置。

### 8.1 功能模块
| 模块 | 功能 |
|---|---|
| 仪表盘 | 用量、活跃、错误率、成本概览 |
| 用户与组织 | 用户 CRUD、角色权限（RBAC）、组织/团队、SSO 配置 |
| 项目/工作区 | 工作区列表、资源配额、生命周期、快照 |
| **OpenCode IDE 设置** | 默认主题/语言、编辑器默认配置、快捷键方案、功能开关 |
| AI/模型设置 | Provider 与密钥、默认模型、温度/上限、用量配额 |
| 技能管理 | 技能审核、上下架、版本、白名单 |
| 插件管理 | 插件源（Open VSX 等）、白名单/黑名单、审核 |
| 工作流管理 | 全局工作流模板、触发器、审计 |
| 国际化 | 语言开关、默认语言、翻译资源管理 |
| 计费与配额 | 套餐、用量计费、限额（P2） |
| 审计与安全 | 操作日志、登录日志、告警 |
| 系统设置 | 存储、运行时、镜像、域名、备份 |

### 8.2 技术
- 后端 `crates/admin-api`（axum），RBAC 中间件。
- 前端 `apps/admin`（React + Ant Design / shadcn），i18n 支持。

---

## 9. 国际化（i18n）多语言

- **默认语言**：简体中文（zh-CN）、English（en）；同时规划 zh-TW、ja、ko、es、fr、de、ru、pt。
- 资源集中在 `locales/`，键值 JSON；前端 i18next 运行时切换，无需刷新。
- 后端可返回本地化的错误消息（Accept-Language / 用户偏好）。
- 编辑器 UI（菜单、命令面板）跟随语言；代码内容不翻译。
- 管理后台可开关支持语言、上传/编辑翻译、标记缺失翻译。
- 流程：抽取 → 翻译（可接机器翻译初翻 + 人工校对）→ 校验（缺 key 检测）→ 发布。

---

## 10. 数据模型与存储

### 10.1 存储选型
- **PostgreSQL**：核心业务数据（用户、项目、技能、插件、工作流、审计）。
- **Redis**：会话、限流、pub/sub、临时状态。
- **对象存储（S3/MinIO）**：文件快照、构建产物、插件/技能包。
- **向量库（pgvector/Qdrant）**：代码语义索引。

### 10.2 主要实体（简化）
```
User(id, email, name, locale, role, org_id, created_at)
Org(id, name, plan, settings_json)
Workspace(id, owner_id, org_id, name, runtime, quota, storage_ref)
File(id, workspace_id, path, blob_ref, updated_at)   # 大文件走对象存储
AgentSession(id, workspace_id, user_id, model, status, created_at)
Message(id, session_id, role, content, tool_calls_json, created_at)
Skill(id, name, version, manifest_json, publisher_id, status)
SkillInstall(id, workspace_id, skill_id, version, enabled)
Plugin(id, source, name, version, manifest_json, status)
PluginInstall(id, workspace_id, plugin_id, enabled)
Workflow(id, workspace_id, name, definition_yaml, triggers_json)
WorkflowRun(id, workflow_id, status, logs_ref, started_at)
ModelProvider(id, org_id, kind, key_ref, config_json)  # 密钥加密存储
AuditLog(id, actor_id, action, target, meta_json, created_at)
Setting(scope, scope_id, key, value_json)  # 全局/组织/用户/工作区分层配置
```

---

## 11. 后端服务与 API 设计

### 11.1 协议
- 客户端 ↔ 后端：REST（JSON）+ WebSocket（实时）。
- 服务间：gRPC（tonic）。
- 鉴权：JWT（短期 access + 刷新 token）；OAuth2/OIDC 接入第三方登录与企业 SSO。

### 11.2 REST 端点（示例）
```
POST   /api/auth/login
POST   /api/auth/refresh
GET    /api/me
GET    /api/workspaces
POST   /api/workspaces
GET    /api/workspaces/{id}/files?path=
PUT    /api/workspaces/{id}/files       # 保存文件
POST   /api/agent/sessions              # 新建 AI 会话
POST   /api/agent/sessions/{id}/messages
GET    /api/skills                      # 市场列表
POST   /api/workspaces/{id}/skills      # 安装技能
GET    /api/plugins
POST   /api/workspaces/{id}/plugins     # 安装插件
POST   /api/workspaces/{id}/workflows
POST   /api/workflows/{id}/run
# 管理后台
GET    /admin/users
GET    /admin/settings
PUT    /admin/settings/opencode-ide
```

### 11.3 WebSocket 通道
```
/ws/workspace/{id}   # 文件变更、光标、协作、Agent 进度、终端流(多路复用)
```

---

## 12. 实时协作与同步

- **文件同步**：基于 CRDT（如 `yrs` / Yjs 的 Rust 实现）实现无冲突协作编辑；`crates/core` 内实现，四端复用。
- **状态广播**：光标、选区、在线状态通过 WebSocket 广播。
- **离线与重连**：本地暂存 + 重连后增量合并。
- **一致性**：服务端为权威，落库 + 快照到对象存储。

---

## 13. 安全与权限

- **沙箱隔离**：每个 Workspace Pod 独立容器（gVisor/rootless），限制 CPU/内存/磁盘/网络。
- **RBAC**：组织/团队/项目多级角色（Owner/Admin/Member/Viewer）。
- **密钥管理**：模型密钥、第三方 token 加密存储（KMS/信封加密），后台不明文回显。
- **插件/技能安全**：安装前权限声明与审核，运行受限沙箱，白名单机制。
- **传输安全**：全站 HTTPS/WSS；CSP、CORS、速率限制。
- **审计**：关键操作全量审计日志。
- **合规**：数据分区、备份、可删除（GDPR 友好）。

---

## 14. 部署与运维（DevOps）

- **容器化**：所有 Rust 服务多阶段构建为精简镜像。
- **编排**：Kubernetes；Workspace Pod 按需拉起与回收。
- **网关**：Ingress/Nginx；CDN 分发前端与静态语言资源。
- **CI/CD**：GitHub Actions —— fmt/clippy/test/build/镜像推送/部署。
- **可观测**：Prometheus + Grafana + OpenTelemetry + 集中日志。
- **环境**：dev / staging / prod 三套。
- **备份**：数据库定时备份、对象存储版本化。

---

## 15. 测试策略

| 层级 | 内容 | 工具 |
|---|---|---|
| 单元测试 | Rust crate 逻辑 | `cargo test` |
| 集成测试 | 服务 + DB + Redis | testcontainers |
| API 测试 | 契约测试 | schemathesis / postman |
| 前端单测 | 组件/逻辑 | Vitest |
| E2E | 关键流程（登录→打开工作区→编辑→AI→安装技能） | Playwright（Web），移动端用 emulator/Appium |
| 性能 | 并发编辑/同步 | k6 |
| 安全 | 依赖审计、渗透 | `cargo audit`、SAST |

---

## 16. 里程碑与排期（建议）

| 阶段 | 里程碑 | 主要交付 | 估时 |
|---|---|---|---|
| M0 | 立项与脚手架 | Monorepo、CI、基础 axum 服务、Web 空壳 | 2 周 |
| M1 | Web IDE MVP | 账号、工作区、文件、Monaco 编辑器、语法高亮、LSP（1~2 语言） | 4~6 周 |
| M2 | AI 与后台 | Agent 会话、工具调用、管理后台基础、i18n（zh/en） | 4~6 周 |
| M3 | 技能/插件/工作流 | 市场、安装机制、工作流编排、终端 | 6~8 周 |
| M4 | 桌面端 | Tauri 桌面打包与本地能力 | 3~4 周 |
| M5 | 移动端 + TUI | 安卓 + 苹果（Tauri Mobile）、触屏适配；TUI 终端客户端（ratatui） | 6~8 周 |
| M6 | 协作与硬化 | CRDT 实时协作、安全/性能/可观测、计费 | 6~8 周 |

> 以上为单团队顺序估算，实际可并行压缩。

---

## 17. 风险与开放问题

### 17.1 风险
- **VSCode「完全一致」**：完整复刻扩展 API 生态成本极高，建议先对齐视觉/交互/编辑核心，扩展生态做子集（基于 Open VSX）。
- **OpenCode「所有功能一致」**：需锁定 OpenCode 的具体版本与功能清单逐项对照，避免范围蔓延。
- **移动端体验**：VSCode 布局在小屏需重排，右键→长按等交互需重新设计。
- **成本**：Workspace Pod + 大模型调用是主要成本，需配额与计费。
- **Rust 前端生态**：编辑器最佳方案（Monaco）为 TS，前端主体仍是 Web 技术，Rust 主要在后端/内核/壳层，需明确边界。

### 17.2 需你确认的开放问题
1. **VSCode「完全一致」的边界**：仅视觉+编辑核心一致即可，还是要支持安装 VSCode 扩展生态？
2. **OpenCode 对齐范围**：是否对齐某个具体版本？是否需要与官方 OpenCode 兼容其配置/技能/插件格式？
3. **部署形态**：纯 SaaS 云端，还是需要私有化/自托管？
4. **AI 模型**：使用哪些 provider？是否需要自带/内置模型，还是全部由后台配置密钥？
5. **优先端顺序**：是否先 Web + 桌面，移动端稍后？
6. **首批语言**：LSP 首批支持哪些编程语言？
7. **多语言范围**：首批界面语言具体是哪几种？
8. **协作**：本期是否需要多人实时协作，还是单人多端即可？

---

## 18. 对照核对：OpenCode 功能覆盖 & VSCode 界面一致性

> 依据 anomalyco/opencode 源码（`packages/core/src`、`packages/opencode/src`、`.opencode/`）逐项核对。图例：✅已在文档覆盖　⚠️需优化/补细节　❌文档缺失（需补）。

### 18.1 OpenCode 功能全量对照

**A. Agent / 会话**
| OpenCode 功能 | 状态 | 说明 |
|---|---|---|
| 内置 Agent：build / plan | ✅ | §7.1 |
| 子 Agent：general + 自定义 subagent（@ 调用） | ✅ | §7.1 |
| 自定义 Agent（`config/agent.ts`、`.opencode/agent/*.md`） | ✅ | §7.1/§7.6 |
| 权限系统 permission（allow/ask/deny） | ✅ | §7.1/§13 |
| 会话压缩 compaction（自动摘要、上下文溢出恢复） | ❌→已补 | 见 §18.3 补充项 |
| 会话快照 / 回滚（snapshot + revert，检查点撤销） | ❌→已补 | OpenCode 有 `snapshot.ts`/`session/revert.ts` |
| 会话分享 share（生成只读分享链接） | ❌→已补 | OpenCode 有 `share/` |
| todo / 任务清单工具（todowrite） | ❌→已补 | 内置工具，AI 维护任务列表 |
| 会话标题自动生成、成本/tokens 统计 | ⚠️ | 需在会话模型补字段 |

**B. 内置工具（tools）**
| 工具 | 状态 | 说明 |
|---|---|---|
| bash（执行命令） | ✅ | §7.7 |
| read / write / edit / apply-patch（读写/补丁式改文件） | ⚠️ | §7.1 提到读写，需明确 apply-patch/diff 语义 |
| glob / grep（文件与内容检索） | ✅ | §7.1 |
| read-filesystem（目录浏览） | ✅ | §7.1 |
| webfetch（抓取网页） | ⚠️ | §7.1 提到 URL，需列为独立工具 |
| **websearch（联网搜索）** | ❌→已补 | 独立工具，需支持搜索 provider |
| question（向用户提问/确认） | ⚠️ | 与权限交互相关，需明确 |
| skill（调用技能） | ✅ | §7.3 |
| 工具启用/禁用（`tools:{}`） | ✅ | §7.6 |

**C. 扩展与配置**
| 功能 | 状态 | 说明 |
|---|---|---|
| Skills（SKILL.md） | ✅ | §7.3 |
| Plugins（`@opencode-ai/plugin`，事件钩子/工具/命令） | ✅ | §7.4 |
| 自定义命令 command（`.opencode/command/*.md`） | ✅ | §7.2 |
| 自定义工具 tool（`.opencode/tool/*.ts`） | ✅ | §7.4 |
| 主题 themes（JSON） | ✅ | §7.5 |
| **MCP 服务器**（`config/mcp.ts`，本地/远程） | ✅ | §7.6 |
| references（外部资料引用） | ✅ | §7.1/§7.6 |
| instruction context（AGENTS.md 规则） | ✅ | §7.1 |
| `opencode.json` 分层配置 | ✅ | §7.6 |
| attachments（图片/文件附件入会话） | ❌→已补 | `config/attachments.ts`、`image/` |
| experimental 配置 | ⚠️ | 可选，低优先 |

**D. 代码智能与编辑器侧**
| 功能 | 状态 | 说明 |
|---|---|---|
| **LSP 集成**（`config/lsp.ts`、`opencode/src/lsp`：诊断/hover 供 Agent） | ⚠️ | §6.4 有 LSP，但需明确 Agent 也用 LSP 诊断 |
| **formatter 自动格式化**（`config/formatter.ts`，保存/编辑后格式化） | ❌→已补 | 需支持 prettier/gofmt 等 |
| **file watcher 文件监听**（`config/watcher.ts`） | ❌→已补 | 外部改动实时刷新 |
| git 集成（diff/blame/worktree） | ⚠️ | §7.8 提 GitHub，需补本地 git 能力与 worktree |

**E. Provider / 模型**
| 功能 | 状态 | 说明 |
|---|---|---|
| ~30+ provider（openai/anthropic/google/bedrock/azure/groq/mistral/xai/openrouter/vercel/cohere/perplexity/deepinfra/cerebras/together/nvidia/gitlab/opencode 等） | ⚠️ | §7.1 列了部分，需补全清单 |
| OpenAI 兼容 / 本地模型 | ✅ | §7.1 |
| **OAuth 登录类 provider**（如 GitHub Copilot、需 `oauth/credential`） | ❌→已补 | 需 OAuth 授权流与凭据存储 |
| 模型目录（models.dev 动态目录） | ⚠️ | 建议接入 models.dev |
| 自动重连/重试 | ✅ | §7.10（已对齐源码） |

**F. 客户端与集成**
| 功能 | 状态 | 说明 |
|---|---|---|
| 无头 Server + SDK/Protocol | ✅ | §7.9 |
| TUI 客户端（终端命令行版） | ✅ | 已纳入范围（见 §5.4）；Rust 实现（如 ratatui），复用 SDK/Server |
| 桌面客户端 | ✅ | Tauri（OpenCode 用 Electron） |
| **IDE 集成 / ACP**（`opencode/src/acp`、`ide`：嵌入 VSCode/Zed，Agent Client Protocol） | ❌→已补 | 需支持作为外部 IDE 的 Agent 后端 |
| GitHub 集成（issue/PR/triage） | ✅ | §7.8 |
| Slack 集成 | ✅ | §7.8 |
| 企业版 / control-plane（`control-plane`、`enterprise`） | ⚠️ | §7.8 提及，需补集中管控细节 |
| background jobs（后台任务） | ❌→已补 | 长任务异步执行 |
| sync（多端/多设备同步） | ✅ | §12（需对齐 OpenCode sync 语义） |

### 18.2 OpenCode 覆盖结论
- **已覆盖核心**：Agent（build/plan/subagent）、权限、命令、技能、插件、自定义工具、主题、MCP、references、配置格式、多 provider、自动重连、Slack/GitHub、Server/SDK。
- **文档原缺失、现补齐（§18.3）**：会话压缩、快照/回滚、会话分享、todo 工具、websearch、attachments、formatter、file watcher、OAuth 类 provider、IDE/ACP 集成、background jobs、apply-patch 语义。
- **需优化**：provider 全清单、git/worktree 本地能力、models.dev 目录、企业 control-plane 细节、会话成本/tokens 统计。（TUI 端已确认纳入，见 §5.4）

### 18.3 补充功能项（纳入范围）
以下已并入功能范围，将在后续详设：
1. **会话压缩 compaction**：超长上下文自动摘要；上下文溢出时压缩后重跑该轮（已见 §7.10）。
2. **快照与回滚**：每步操作可创建快照，支持一键回滚到任意检查点（对齐 `snapshot`/`revert`）。
3. **会话分享**：生成只读分享链接，可回放会话（对齐 `share`）。
4. **todo 任务工具**：Agent 维护任务清单，前台 AI 面板可视化。
5. **websearch / webfetch**：联网搜索 + 网页抓取工具。
6. **attachments**：图片/文件作为多模态输入进入会话。
7. **formatter**：保存/编辑后按语言自动格式化（prettier/gofmt/rustfmt 等）。
8. **file watcher**：监听工作区外部改动并实时刷新。
9. **OAuth 类 provider 与凭据管理**：支持 GitHub Copilot 等需 OAuth 的模型来源，凭据加密存储。
10. **IDE 集成 / ACP**：作为 Agent 后端接入外部 VSCode/Zed（Agent Client Protocol）。
11. **background jobs**：长耗时任务异步执行与状态跟踪。
12. **apply-patch/diff 编辑语义**：以补丁方式改文件，前台以 diff 呈现并确认应用。
13. **git 本地能力 + worktree**：本地 diff/blame/分支/worktree。

### 18.4 VSCode 界面一致性对照

**已在文档覆盖（§6）**
| 项 | 状态 |
|---|---|
| 整体布局：活动栏/侧边栏/编辑区/面板/状态栏 | ✅ |
| 主题配色：Dark+/Light+/High Contrast token | ✅ |
| Monaco 编辑器：实时高亮、行号、minimap、折叠、括号匹配 | ✅ |
| TextMate 语法（与 VSCode 一致的高亮） | ✅ |
| 语义高亮（LSP Semantic Tokens） | ✅ |
| 右键上下文菜单（转到定义/引用/重命名/格式化…） | ✅ |
| 命令面板、快捷键与 VSCode 默认一致 | ✅ |
| 多标签、分屏、拖拽、面包屑、搜索替换（正则） | ✅ |
| 文件图标主题（Seti） | ✅ |
| 浏览器打开本地文件夹 | ✅ |
| AI 独立右侧栏（不侵占编辑区） | ✅ |

**VSCode 常见能力：文档原缺失、现补齐（⚠️/❌→已补）**
| VSCode 能力 | 状态 | 说明 |
|---|---|---|
| **快速打开 Quick Open（Ctrl+P）** | ❌→已补 | 文件模糊跳转 |
| **全局搜索面板（跨文件搜索/替换、正则、include/exclude）** | ⚠️ | §6.3 提搜索，需补独立搜索视图 |
| **源代码管理（SCM）视图**：Git 变更、暂存、提交、diff、blame | ❌→已补 | 侧边栏 Source Control |
| **调试（Debug）** | ❌ 不做 | 已确认：只做「运行/终端」，不接 DAP、不做断点调试 |
| **测试资源管理器（Testing）** | ❌→已补 | 运行/查看测试 |
| **问题面板 Problems**（LSP 诊断汇总） | ⚠️ | 提及占位，需接 LSP 诊断 |
| **输出/调试控制台面板** | ⚠️ | 占位，需实现 |
| **集成终端**（多终端、拆分） | ✅ | §7.7 |
| **设置 UI + settings.json**（图形化设置 + JSON） | ❌→已补 | 需做设置界面与 JSON 双通道 |
| **键盘快捷方式编辑器 Keybindings** | ❌→已补 | 可视化改键位 |
| **代码片段 Snippets** | ❌→已补 | 用户自定义片段 |
| **IntelliSense/补全/参数提示/hover** | ✅ | §6.4（LSP） |
| **重构/快速修复 Code Actions/灯泡** | ⚠️ | LSP 支持，需在 UI 暴露 |
| **差异编辑器 Diff Editor** | ⚠️ | Monaco 支持，AI/Git 均用，需明确 |
| **多根工作区 Multi-root** | ❌→已补 | 一窗口多文件夹 |
| **Zen 模式 / 面板显隐 / 布局自定义** | ⚠️ | 布局细节 |
| **扩展视图 Extensions（安装/管理）** | ✅ | §7.4（插件，Open VSX 子集） |
| **Emmet、自动保存、格式化-保存时** | ⚠️/❌→已补 | formatter 见 §18.3 |
| **面包屑、Sticky Scroll、括号对着色** | ⚠️ | Monaco 可配，需开启 |
| **notebook（.ipynb）支持** | ❌ | 建议后期，非首期 |
| **远程开发（Remote/SSH/容器）** | ⚠️ | 本产品云端 Workspace Pod 天然覆盖多数场景 |

### 18.5 VSCode 一致性结论
- **视觉/编辑核心已一致**：布局、配色、高亮、右键、命令面板、快捷键、Monaco 能力。
- **文档原缺失、现补齐（纳入范围）**：Quick Open、SCM/Git 视图、Testing、设置 UI+settings.json、Keybindings 编辑器、Snippets、多根工作区、独立全局搜索视图。
- **需优化/明确**：Problems/Output 面板接 LSP、Diff 编辑器、Code Actions 在 UI 暴露、Sticky Scroll/括号着色等细节开关。
- **不做**：断点调试（已确认只做运行/终端，不接 DAP）。
- **建议延后**：Notebook、完整远程开发（云端 Pod 已覆盖大部分）。

### 18.6 补充里程碑（承接 §18.3 / §18.5 的新增项）
| 阶段 | 新增交付 |
|---|---|
| M2+ | LSP 诊断→Problems 面板、formatter、file watcher、Quick Open、settings.json |
| M3+ | 快照/回滚、会话分享、todo 工具、websearch/webfetch、attachments、SCM/Git 视图、设置 UI |
| M3+ | apply-patch/diff、OAuth 类 provider、MCP 完整、provider 全清单 |
| M4+ | Testing 面板、Keybindings 编辑器、Snippets、多根工作区、Diff 编辑器完善 |
| M5+ | IDE/ACP 集成、background jobs、企业 control-plane |

---

## 附：技术选型速览

- 后端：Rust + axum + tokio + SQLx/SeaORM + tonic + Redis + PostgreSQL + 对象存储 + pgvector/Qdrant
- 前端：TypeScript + React/Svelte + **Monaco** + TextMate 语法 + i18next + Vite
- 跨端：**Tauri 2**（桌面/安卓/苹果）+ WASM 复用 Rust 内核
- 协作：CRDT（yrs）
- DevOps：Docker + Kubernetes + GitHub Actions + Prometheus/Grafana/OTel
