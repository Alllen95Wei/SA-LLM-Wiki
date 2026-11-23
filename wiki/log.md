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

## [2026-11-23] ingest | 03_Shell.pdf

- **來源檔案**：`03_Shell.pdf`（p.1–49）
- **新增頁面（共 14 頁）**：
  - 總覽頁：`wiki/src-03-shell.md`
  - 原子頁：`wiki/shell-terminal-kernel-and-execution.md`、`wiki/shell-startup-files.md`、`wiki/shell-vs-environment-variables.md`、`wiki/shell-quoting-and-expansion.md`、`wiki/shell-control-operators-and-exit-status.md`、`wiki/shell-builtins-aliases-and-job-control.md`、`wiki/io-redirection-and-pipelines.md`、`wiki/unix-philosophy-and-text-toolkit.md`、`wiki/head-tail-and-grep.md`、`wiki/cut-vs-awk.md`、`wiki/sort-uniq-and-wc.md`、`wiki/tr-and-sed.md`、`wiki/xargs-safe-arguments.md`
- **更新頁面**：`wiki/index.md`（更新來源追蹤表狀態為 `已匯入` 並加入全部分類連結）。
- **範圍與略過說明**：依聚焦 Debian Linux 原則，略過 FreeBSD 14+ 預設 shell 對照（p.7）、FreeBSD `sh` / `tcsh` 啟動設定檔（p.11）、`tcsh` 變數賦值語法（p.16）、FreeBSD `rc.d`（p.21）、FreeBSD 延伸旗標 `xargs -J`（p.39）與 `tcsh` 歷史替換修飾符（p.48）；未展開的實作細節（如 Debian `/bin/sh` 底層實作、`Ctrl-Z` 具體 signal 名稱、`grep -E`/`sed` 進階語法、`ShellCheck` 安裝方式）皆標註為「投影片未展開」。
