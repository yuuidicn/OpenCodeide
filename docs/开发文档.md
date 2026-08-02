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

---

## 6. 前台编辑器（对齐 VSCode）

### 6.1 视觉与排版一致性
- 使用 **Monaco Editor**（VSCode 的编辑器内核），天然保证：代码实时渲染、缩进参考线、行号、minimap、括号匹配、折叠。
- 采用 VSCode 官方 **Dark+ / Light+ / High Contrast** 主题 token；布局栅格（活动栏 Activity Bar、侧边栏 Side Bar、编辑区、面板 Panel、状态栏 Status Bar）像素级复刻。
- 图标使用 VSCode Seti/官方文件图标主题。

### 6.2 自动识别语言与高亮
- 依据文件扩展名 + shebang + 内容启发式自动识别语言。
- 高亮采用 Monaco 内置 tokenizer；对需要与 VSCode 100% 一致的场景，加载 **TextMate 语法（.tmLanguage）** + `vscode-oniguruma`（WASM），复用 VSCode 的语法定义。
- 语义高亮由 LSP 提供（Semantic Tokens）。

### 6.3 与 VSCode 一致的交互
- **右键上下文菜单**：复刻 VSCode 菜单项（转到定义、查找所有引用、重命名符号、格式化文档、剪切/复制/粘贴、命令面板等）。
- **命令面板**（Ctrl/Cmd+Shift+P）、快捷键绑定与 VSCode 默认一致。
- **多标签、分屏、拖拽、面包屑导航、搜索替换（含正则）**。
- **集成终端**面板、问题面板、输出面板、调试面板占位。

### 6.4 语言智能（LSP）
- 后端在 Workspace Pod 内启动对应语言的 Language Server（rust-analyzer、tsserver、pyright、gopls 等）。
- 前端 Monaco 通过 `monaco-languageclient` + LSP over WebSocket 连接，获得补全、诊断、跳转、hover、重命名、格式化。

---

## 7. 核心功能（对齐 OpenCode）

> 目标：Agent、工具（tools）、工作流、技能、插件的能力与 OpenCode 一致。

### 7.1 AI Agent
- 会话式编码助手：读代码、写代码、运行命令、修 bug、跑测试。
- 多模型支持：可配置 OpenAI/Anthropic/本地模型等 provider（密钥在管理后台配置）。
- 工具调用（tool use）：读写文件、执行终端、搜索代码、运行测试、打开 URL 等，与 OpenCode 工具体系对齐。
- 上下文管理：仓库索引 + 向量检索（RAG）提供相关代码上下文。

### 7.2 自动工作流（Workflow）
- 可编排的多步骤流程：如「读取需求 → 生成代码 → 运行测试 → 修复 → 提交 PR」。
- 触发方式：手动、定时、事件（如 push/issue）。
- 可视化编排界面 + YAML 定义（存版本库）。

### 7.3 技能安装（Skills）
- 技能 = 提示词模板 + 工具声明 + 可选脚本，打包为可安装单元。
- 从「技能市场」浏览、搜索、一键安装/卸载/更新，支持版本与依赖。
- 安装后在 Agent 会话中可被调用。

### 7.4 插件安装（Plugins）
- 插件扩展 IDE/Agent：编辑器扩展、LSP、主题、命令、Agent 工具。
- 兼容策略：优先兼容 **Open VSX** 生态的 VSCode 扩展（受运行环境限制的做能力子集）。
- 一键安装/启用/禁用/卸载，权限声明与沙箱执行。

### 7.5 终端与执行
- 集成终端连接 Workspace Pod（PTY over WebSocket）。
- 代码运行/调试在沙箱内进行，资源与网络受限。

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
| M5 | 移动端 | 安卓 + 苹果（Tauri Mobile）、触屏适配 | 6~8 周 |
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

## 附：技术选型速览

- 后端：Rust + axum + tokio + SQLx/SeaORM + tonic + Redis + PostgreSQL + 对象存储 + pgvector/Qdrant
- 前端：TypeScript + React/Svelte + **Monaco** + TextMate 语法 + i18next + Vite
- 跨端：**Tauri 2**（桌面/安卓/苹果）+ WASM 复用 Rust 内核
- 协作：CRDT（yrs）
- DevOps：Docker + Kubernetes + GitHub Actions + Prometheus/Grafana/OTel
