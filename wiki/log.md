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

## [2026-11-23] refactor | 改採主題式架構與明確側邊欄排序（nav）

- **架構規範更新**：修訂 `AGENTS.md`，將細粒度單點拆頁與 `src-*.md` 改為**主題式（Option A）**單篇完整講義架構，並新增 `zensical.toml` `nav` 分組排序規範與表格管線字元規範。
- **側邊欄排序調整**：更新 `zensical.toml`，啟用 `navigation.sections` 並設定明確 `nav` 分組（`首頁與總目錄` → `課程主題筆記` → `知識庫維運`），將 `tags.md`、`log.md` 固定分離於底部，不再受 slug 字母順序影響。
- **內容無損整併**：將原 `03_Shell.pdf` 拆分之 14 份檔案（`wiki/src-03-shell.md`、`wiki/shell-terminal-kernel-and-execution.md`、`wiki/shell-startup-files.md`、`wiki/shell-vs-environment-variables.md`、`wiki/shell-quoting-and-expansion.md`、`wiki/shell-control-operators-and-exit-status.md`、`wiki/shell-builtins-aliases-and-job-control.md`、`wiki/io-redirection-and-pipelines.md`、`wiki/unix-philosophy-and-text-toolkit.md`、`wiki/head-tail-and-grep.md`、`wiki/cut-vs-awk.md`、`wiki/sort-uniq-and-wc.md`、`wiki/tr-and-sed.md`、`wiki/xargs-safe-arguments.md`）完整整併為單一主題講義 `wiki/shell.md` 並移除舊檔，同時更新 `wiki/index.md`。

## [2026-11-23] refactor | 新增 Marp HTML 簡報流程與 Shell 簡報

- **簡報系統建立**：新增共用 Marp 主題 `themes/sa-wiki.css`（與 Zensical 一致之 Indigo/Blue 配色、支援 `mermaid@11` 延遲渲染與雙欄／陷阱樣式），並於 `.gitignore` 排除編譯暫存目錄 `wiki/slides/`。
- **首份簡報產出**：依 `03_Shell.pdf`（p.1–49）建立 `slides/shell.md`（共 16 張考前快刷投影片，包含 6 張 Mermaid 圖表、雙欄比較與考前速查表），並於 `wiki/shell.md` 與 `wiki/index.md` 建立雙向連結。
- **部署與規範同步**：更新 `.github/workflows/docs.yml`（整合 Node.js / `@marp-team/marp-cli` 編譯步驟至 Zensical 建置之前）及 `AGENTS.md`（納入 Marp 簡報規範與雙重編譯驗證流程）。
