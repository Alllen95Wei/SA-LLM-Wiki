---
title: "首頁與總目錄"
---

# System Administration LLM Wiki（Debian 13）

本知識庫以**期中／期末考複習**為核心目標，嚴格依據 `raw/` 目錄內的課程簡報 PDF 建立並持續演進，並聚焦於 **Debian Linux（Debian 13 / trixie）**。

- 快速導覽：[標籤索引](tags.md) ｜ [演進日誌](log.md)
- 知識邊界：僅收錄 `raw/` 簡報原文所載內容並標註來源檔名與頁碼；不自行補充外部知識，亦不收錄 FreeBSD 歷史對照內容。

## 課程來源追蹤表（Sources Status）

| 來源檔案 | 狀態 | 總覽頁面 | 備註 |
| :--- | :--- | :--- | :--- |
| `03_Shell.pdf` | 已匯入 | [簡報總覽：Shells](src-03-shell.md) | 已拆分為 13 篇原子筆記（略過 FreeBSD/tcsh） |
| `03_Shell_Programming.pdf` | 待匯入 | — | — |
| `06_Boot_ShutDown.pdf` | 待匯入 | — | — |
| `07_User_Management.pdf` | 待匯入 | — | — |
| `10_FileSystem.pdf` | 待匯入 | — | — |
| `BTRFS.pdf` | 待匯入 | — | — |
| `Docker_Basic.pdf` | 待匯入 | — | — |
| `Linux_Networking_Commands.pdf` | 待匯入 | — | — |
| `LVM.pdf` | 待匯入 | — | — |

## 課程簡報總覽（Sources）

- [簡報總覽：Shells（命令列、管線與文字處理）](src-03-shell.md)：涵蓋 Terminal/Shell/Kernel、啟動檔、變數引號展開、工作控制、I/O 重導向與 UNIX 文字處理工具鏈（`03_Shell.pdf, p.1-49`）。

## 核心概念與原理（Concepts）

- [Terminal、Shell、Kernel 分層與指令執行流程](shell-terminal-kernel-and-execution.md)：三層職責分工、按下 Enter 後五階段流程（Read & parse → Expand → Resolve → Execute → Wait）與 Debian 預設 Shell（`03_Shell.pdf, p.4-7`）。
- [Shell 變數與環境變數](shell-vs-environment-variables.md)：`export` 子行程繼承差異、`set` vs `env`/`printenv`、`$SHELL` 含義陷阱與 POSIX 無空白賦值（`03_Shell.pdf, p.14-16`）。
- [Shell 引號規則、展開與指令替換](shell-quoting-and-expansion.md)：單／雙引號與未加引號展開差異、`$(command)` 巢狀寫法、Globbing vs Regex 及 `ShellCheck` 靜態分析（`03_Shell.pdf, p.16-19, 21, 44-45`）。
- [控制運算子與結束狀態碼（Exit Status）](shell-control-operators-and-exit-status.md)：`$?` 慣例（`0` 成功、非 `0` 失敗）與 `;`、`&&`、`||` 條件串接（`03_Shell.pdf, p.20, 45`）。
- [標準串流、I/O 重導向與管線（Pipelines）](io-redirection-and-pipelines.md)：FD `0/1/2`、`>out 2>&1` 由左至右順序考點、`|` 不含 `stderr` 及可攜寫法 `2>&1 |` vs `|&`（`03_Shell.pdf, p.27-30, 45`）。

## 指令與管理工具（Commands）

- [內建指令、工作控制、別名與歷史紀錄](shell-builtins-aliases-and-job-control.md)：Built-in vs External（`type`、`command -V`）、`Ctrl-Z`/`jobs`/`bg`/`fg` 狀態機、`&` 非 detached 陷阱、目錄堆疊、別名與 `!` 歷史展開（`03_Shell.pdf, p.21, 23-25, 45, 47-48`）。
- [`head`、`tail` 與 `grep` 檢視與搜尋](head-tail-and-grep.md)：頭尾擷取、`tail -f`、固定字串 `grep -F`／延伸正規 `grep -E`、樣式加引號保護與 `journalctl` 說明（`03_Shell.pdf, p.33-34`）。
- [`sort`、`uniq` 與 `wc` 排序、去重與計數](sort-uniq-and-wc.md)：文字 vs `-n` 數值排序、`uniq` 只偵測相鄰重複（須先 `sort`）與行／字／位元組計數（`03_Shell.pdf, p.33, 36, 41-42`）。
- [`tr` 與 `sed` 字元轉換與串流編輯](tr-and-sed.md)：`tr` 逐字元轉換／刪除／壓縮與 `sed` 逐行正規替換之能力邊界及 locale 字元類別（`03_Shell.pdf, p.33, 37`）。
- [`xargs` 參數轉換與安全檔名處理](xargs-safe-arguments.md)：將 `stdin` 轉為參數列、POSIX `-n`/`-I`/`-s` 選項，以及 `find -print0 | xargs -0` 或 `-exec ... {} +` 安全處理含空白／換行檔名（`03_Shell.pdf, p.33, 38-40, 45`）。

## 關鍵設定檔與路徑（Configs）

- [Shell 執行模式與 Bash 啟動設定檔](shell-startup-files.md)：Interactive login / Interactive non-login / Non-interactive 模式、Bash 啟動檔讀取順序、`~/.bashrc` 自動化陷阱與 `curl | sh` 風險（`03_Shell.pdf, p.9-12`）。

## 跨觀念比較與綜合整理（Comparisons & Syntheses）

- [`cut` 與 `awk` 欄位擷取比較](cut-vs-awk.md)：固定分隔欄位抽取 vs 條件篩選／排版／運算能力對比、`/etc/passwd` 實例與避免在腳本解析 `ls`（`03_Shell.pdf, p.33, 35, 41`）。
- [檔案目錄指令、UNIX 哲學與文字處理選型地圖](unix-philosophy-and-text-toolkit.md)：日常檔案指令與 `rm` 風險、8 大文字工具選型表、`/etc/passwd` 與 `access.log` 整合管線及 Shell Cheat Sheet（`03_Shell.pdf, p.32-33, 41-43, 45`）。
