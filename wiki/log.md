---
title: "演進日誌"
---

# 演進日誌（Wiki Log）

此頁面以時間順序（append-only）記錄每次 **Ingest**、**Query 回存** 與 **Lint** 的維運紀錄。每筆紀錄遵循格式：`## [YYYY-MM-DD] <init|ingest|query|lint> | <標題>`。

## [2026-11-23] init | 初始化 System Administration LLM Wiki 骨架

- 建立專案設定：`zensical.toml`（設定 `docs_dir = "wiki"`、繁體中文介面、Mermaid 與 Tags 支援）、`.gitignore`（排除 `raw/`、`site/`、`.cache/`）與 `.github/workflows/docs.yml`（GitHub Pages 自動建置部署流程）。
- 建立操作規範：`AGENTS.md`（定義三層架構、嚴格忠於來源、聚焦 Debian 13、扁平原子筆記、Mermaid 規範與互動式 Ingest／Query／Lint 流程）。
- 建立初始系統頁面：`wiki/index.md`、`wiki/tags.md`、`wiki/log.md`。
- 待匯入 `raw/` 來源（共 9 份）：`03_Shell.pdf`、`03_Shell_Programming.pdf`、`06_Boot_ShutDown.pdf`、`07_User_Management.pdf`、`10_FileSystem.pdf`、`BTRFS.pdf`、`Docker_Basic.pdf`、`Linux_Networking_Commands.pdf`、`LVM.pdf`。
