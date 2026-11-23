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

## [2026-11-23] refactor | 調整簡報定位為教學型 HTML 簡報並重寫 Shell 簡報

- **簡報定位調整**：依作業需求明確區分雙層產物——`wiki/` 持續聚焦期中／期末考複習講義，而 `slides/` 則以對應 Wiki 頁面為知識來源，改為製作適合初學者從零理解概念的**教學型 HTML 簡報**（同步更新 `AGENTS.md`、`wiki/shell.md`、`wiki/index.md`）。
- **樣式與內容重構**：於 `themes/sa-wiki.css` 新增 `.divider` 單元過場樣式並調整 Mermaid 高度上限；依 `wiki/shell.md`（`03_Shell.pdf`）重寫 `slides/shell.md`（擴展為 27 張教學投影片，包含學習地圖、6 個單元過場頁、循序漸進的白話說明與一頁一重點設計）。

## [2026-11-23] ingest | 03_Shell_Programming.pdf

- **來源檔案**：`03_Shell_Programming.pdf`（p.1–62）
- **新增頁面**：
  - 主題講義：`wiki/shell-programming.md`（涵蓋 POSIX `#!/bin/sh` 與 Shebang、參數展開、`"$@"` vs `"$*"`、條件測試與流程控制、管線 Subshell 陷阱、函式與 `getopts`、錯誤處理／`trap`／Signals／`ShellCheck`、BRE/ERE 正規表達式、`sed`/`awk` 及主機健康檢查實戰重構）
  - 教學型簡報：`slides/shell-programming.md`（以 `wiki/shell-programming.md` 為知識來源製作，含學習地圖、7 個單元過場頁與 5 張 Mermaid 圖表）
- **更新檔案**：`zensical.toml`（於 `課程主題筆記` 加入 `shell-programming.md`）、`wiki/index.md`（狀態更新為 `已匯入` 並加入講義與教學簡報連結）、`wiki/shell.md`（於跨主題關聯處補上前往 `shell-programming.md` 之雙向連結）。
- **範圍與略過說明**：依聚焦 Debian Linux 與 POSIX `sh` 原則，略過 FreeBSD/`tcsh` 專屬歷史與手冊參照（p.6、47、62）；投影片未展開項目（如 `/bin/sh` 底層套件實作、no-match 具體代碼、非 POSIX `local` 語法、其他 `set` 旗標、BRE 跳脫規則及 `sed` 原地編輯／外部腳本旗標）已於講義明確標註。

## [2026-11-23] ingest | Docker_Basic.pdf

- **來源檔案**：`Docker_Basic.pdf`（p.1–22）
- **新增頁面**：
  - 主題講義：`wiki/docker-basic.md`（涵蓋 Container／Image 與 Linux namespaces、Container vs. VM 架構與四點對比、Docker / `containerd` / OCI / `shim` / `runc` 與 `Podman` 堆疊分工、官方安裝腳本／`systemd` 啟用與 `docker` 群組設定、Docker Workflow、`docker pull` 與 Tag／多層結構、`docker run` 常用選項與 Named Volumes／Bind Mount、`docker exec` / `logs` / `ps` / `stop` / `restart` / `rm` 生命週期與考前速查表）
  - 教學型簡報：`slides/docker-basic.md`（以 `wiki/docker-basic.md` 為知識來源製作，含學習地圖、3 個單元過場頁與 4 張 Mermaid 圖表）
- **更新檔案**：`zensical.toml`（於 `課程主題筆記` 排入 `docker-basic.md` 第 3 順位）、`wiki/index.md`（狀態更新為 `已匯入` 並加入講義與教學簡報連結）、`wiki/shell.md`（於跨主題關聯處補上前往 `docker-basic.md` 之雙向連結）。
- **範圍與略過說明**：本份來源全數聚焦 Linux 與 Docker 操作，無 FreeBSD 專屬內容需略過；投影片未展開項目（如除 Linux namespaces 外的資源配額機制、`OCI`/`shim`/`crun`/`Singularity` 細節、`apt` 安裝、`docker` 群組權限風險與重新登入生效細節、`Dockerfile`/`docker build` 語法及其他未提及之子指令）已於講義明確標註。

## [2026-11-23] lint | 前三份主題全面健康檢查與修正

- **檢查範圍**：全部已匯入主題（`wiki/shell.md`、`wiki/shell-programming.md`、`wiki/docker-basic.md`）、對應教學型簡報（`slides/*.md`）、`zensical.toml`、`wiki/index.md`、`wiki/tags.md`、`.gitignore` 與編譯輸出 `site/`。
- **通過項目**：`zensical.toml` `nav` / `wiki/*.md` / `wiki/index.md` 三方同步、所有 `category: topic` 與 `slides/*.md` 一對一同步、零雙括號維基連結（Wikilink）、零斷連、無表格反斜線管線字元誤寫、YAML／Marp frontmatter 與標籤合規、Mermaid 雙引號安全、無殘留 FreeBSD 操作配方。
- **完成修復**：
  - 修正 `wiki/docker-basic.md` 交叉引用 `03_Shell.pdf` 的頁碼（補上安全下載原則 `p.12` 與執行流程 `p.5`）。
  - 移除 `wiki/shell-programming.md` 「投影片未展開」提示框中夾帶之外部範例代碼與未載術語（涵蓋第 3、6、7 節提示框修訂），確保嚴格忠於來源。
  - 於 `wiki/shell.md` 三處「投影片未展開」提示框補上前往 `wiki/shell-programming.md` 的跨頁連結。
  - 修正 `slides/shell-programming.md` 與 `slides/docker-basic.md` 中 `<pre class="mermaid">` 內原始 `<` 字元，確保 HTML 輸出零歧義。

## [2026-11-23] ingest | 10_FileSystem.pdf

- **來源檔案**：`10_FileSystem.pdf`（p.1–44）
- **新增頁面**：
  - 主題講義：`wiki/filesystem.md`（涵蓋 `ls -l` 欄位拆解、VFS 與系統呼叫分層、Pathname 與 `hier(7)` 目錄用途、`mount` / `/etc/fstab` / `umount` + `lsof`、七大檔案類型與 UNIX domain socket / Pipes / `mkfifo`、Inode 與分割區四大區塊、Hard Link vs. Symbolic Link 三階段實驗、`chmod` 與特殊權限 `4000/2000/1000`、`umask` 位元遮罩逐位演算、File Protection 最低權限表、`chown`/`chgrp`，以及 Journaling / CoW / File Attribute Extension）
  - 教學型簡報：`slides/filesystem.md`（以 `wiki/filesystem.md` 為知識來源製作，依使用者需求擴增為 33 張投影片、其中單元四用 11 張專頁細講 `rwx` Bit Mask、特殊位元與 `umask` 逐位演算，含學習地圖、5 個單元過場頁與 5 張 Mermaid 圖表）
- **更新檔案**：`zensical.toml`（於 `課程主題筆記` 排入 `filesystem.md` 第 4 順位）、`wiki/index.md`（狀態更新為 `已匯入` 並加入講義與教學簡報連結）、`wiki/shell.md` 與 `wiki/shell-programming.md`（補上權限與管線跨主題雙向連結）。
- **範圍與略過說明**：依聚焦 Debian Linux 原則，略過 FreeBSD Handbook 網址（p.2）、FreeBSD `hier(7)` 專屬目錄 `/usr/src`/`/usr/obj`/`/var/db/ports`（p.13）、FreeBSD 裝置路徑 `/dev/ad*` 與 `ufs` `/etc/fstab` 實例（p.14, 16–17）、`fstat` 與 `/usr/ports` 範例輸出（p.17–18）、`(FreeBSD)` ELF 標籤與 `.tcshrc`（p.20, 32），以及整段 `Appendix: FreeBSD bonus flags`（`chflags`、`ls -ol`、`wheel` 等，p.39–41）；投影片未展開細節（`ls -l` 第三欄連結數關聯、`/etc/fstab` `Dump`/`Pass#` 與詳細選項、匿名 Pipe 程式碼、`inode` `owners(2)`/`timestamps(3)` 子欄位名、`setgid` 執行檔範例與無 `x` 時的顯示符號、`fsck`/`xattr` 操作指令）已明確標註。
