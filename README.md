# OpenCode IDE

网页版 OpenCode IDE：基于 Rust 构建，跨四端互通（Web / 桌面 / 安卓 / 苹果），统一管理后台；前台界面对齐 VSCode，AI/工作流/技能/插件能力对齐 OpenCode，支持多语言（i18n）。

## 文档

- 完整开发文档：[docs/DEVELOPMENT.md](docs/DEVELOPMENT.md)（中文原名：`docs/开发文档.md`）

## 技术选型速览

- 后端：Rust + axum + tokio + SQLx/SeaORM + tonic + Redis + PostgreSQL + 对象存储 + pgvector/Qdrant
- 前端：TypeScript + React/Svelte + Monaco + TextMate 语法 + i18next + Vite
- 跨端：Tauri 2（桌面 / 安卓 / 苹果）+ WASM 复用 Rust 内核
- 协作：CRDT（yrs）
- DevOps：Docker + Kubernetes + GitHub Actions + Prometheus/Grafana/OpenTelemetry
