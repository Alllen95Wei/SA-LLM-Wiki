---
title: "首頁與總目錄"
---

# System Administration LLM Wiki（Debian 13）

本知識庫以**期中／期末考複習**為核心目標，嚴格依據 `raw/` 目錄內的課程簡報 PDF 建立並持續演進，並聚焦於 **Debian Linux（Debian 13 / trixie）**。

- 快速導覽：[標籤索引](tags.md) ｜ [演進日誌](log.md)
- 組織模式：採**主題式（Topic-Based）**編排，每份核心主題整合為一篇結構連貫的講義，透過頁面右側目錄（TOC）快速定位單元；側邊欄依 `zensical.toml` 分組秩序呈現。
- 知識邊界：僅收錄 `raw/` 簡報原文所載內容並標註來源檔名與頁碼；不自行補充外部知識，亦不收錄 FreeBSD 歷史對照內容。

## 課程來源追蹤表（Sources Status）

| 來源檔案 | 狀態 | 主題講義頁面 | 教學簡報 | 備註 |
| :--- | :--- | :--- | :--- | :--- |
| `03_Shell.pdf` | 已匯入 | [Shell：命令列、管線與文字處理](shell.md) | [開啟簡報](slides/shell.html){ target="_blank" } | 主題式完整講義（略過 FreeBSD/tcsh） |
| `03_Shell_Programming.pdf` | 已匯入 | [Shell 腳本設計：流程控制、健壯性與 sed / awk](shell-programming.md) | [開啟簡報](slides/shell-programming.html){ target="_blank" } | 主題式完整講義（POSIX sh 與 Debian 13） |
| `Docker_Basic.pdf` | 已匯入 | [Docker 基礎：容器概念、安裝與常用指令](docker-basic.md) | [開啟簡報](slides/docker-basic.html){ target="_blank" } | 主題式完整講義（無 FreeBSD 內容） |
| `Linux_Networking_Commands.pdf` | 待匯入 | — | — | — |
| `06_Boot_ShutDown.pdf` | 待匯入 | — | — | — |
| `07_User_Management.pdf` | 待匯入 | — | — | — |
| `10_FileSystem.pdf` | 已匯入 | [檔案系統：架構、Inode、連結與權限管理](filesystem.md) | [開啟簡報](slides/filesystem.html){ target="_blank" } | 主題式完整講義（已略過 FreeBSD 專屬內容與 bonus flags） |
| `LVM.pdf` | 已匯入 | [邏輯捲宗管理（LVM）：架構、精簡配置、快照與磁碟擴縮實戰](lvm.md) | [開啟簡報](slides/lvm.html){ target="_blank" } | 主題式完整講義（Debian 13 實戰，無 FreeBSD 內容） |
| `BTRFS.pdf` | 已匯入 | [Btrfs（B-Tree Filesystem）：Subvolume、佈局、快照與遷移實戰](btrfs.md) | [開啟簡報](slides/btrfs.html){ target="_blank" } | 主題式完整講義（Debian 13 實戰，無 FreeBSD 內容） |

## 課程主題筆記（Topics）

- [Shell：命令列、管線與文字處理](shell.md)（[教學簡報](slides/shell.html){ target="_blank" }）：完整涵蓋 Terminal/Shell/Kernel 與五階段執行、Bash 啟動檔、變數／引號／展開與控制運算子、Built-in 與工作控制、FD `0/1/2` 重導向與管線、UNIX 八大文字處理工具鏈、整合實戰與考前速查表（`03_Shell.pdf, p.1-49`）。
- [Shell 腳本設計：流程控制、健壯性與 sed / awk](shell-programming.md)（[教學簡報](slides/shell-programming.html){ target="_blank" }）：涵蓋 POSIX `#!/bin/sh` 與 Shebang、參數預設值與字串展開、`"$@"` vs `"$*"`、`test`/`[ ... ]` 與 `while IFS= read -r` 管線 Subshell 陷阱、`getopts`、`trap`/Signals、`ShellCheck`、BRE/ERE 正規表達式、`sed`/`awk` 串流處理及健康檢查重構實例（`03_Shell_Programming.pdf, p.1-62`）。
- [Docker 基礎：容器概念、安裝與常用指令](docker-basic.md)（[教學簡報](slides/docker-basic.html){ target="_blank" }）：涵蓋容器／映像檔與 Linux namespaces、Container vs. VM 架構與四點對比、Docker / `containerd` / OCI / `shim` / `runc` 與 `Podman` 堆疊分工、Linux 官方安裝／`systemd` 啟用與 `docker` 群組設定、Docker Workflow、`docker pull`（Tag 與多層結構）、`docker run`（含 Named Volumes 與 Bind Mount）、`docker exec`、`docker logs` 與 `ps`/`stop`/`restart`/`rm` 管理（`Docker_Basic.pdf, p.1-22`）。
- [檔案系統：架構、Inode、連結與權限管理](filesystem.md)（[教學簡報](slides/filesystem.html){ target="_blank" }）：涵蓋 `ls -l` 八欄、VFS 與系統呼叫分層、Pathname 與 `hier(7)` 目錄配置、`mount` / `/etc/fstab` / `umount` + `lsof`、七大檔案類型與 Socket / Pipe（`mkfifo`）、Inode 與分割區四大區塊、Hard Link vs. Symbolic Link 三階段實驗、`chmod` 與特殊權限 `4000/2000/1000`、`umask` 位元遮罩運算、File Protection 最低權限表、`chown`/`chgrp`，以及 Journaling / CoW / 擴充屬性（`10_FileSystem.pdf, p.1-44`）。
- [邏輯捲宗管理（LVM）：架構、精簡配置、快照與磁碟擴縮實戰](lvm.md)（[教學簡報](slides/lvm.html){ target="_blank" }）：涵蓋 Device Mapper 儲存池、`PV → VG → LV` 三層架構與 `PE / LE`（預設 `4MB`）、Thin Provisioning（`Thin Pool` 與 `Thin LV`）、CoW 快照（`Not backup!`、`umount` + `lvconvert --merge`），以及新增／擴充（Bottom → Top）對比縮容／拆碟（Top → Bottom）完整實戰（`LVM.pdf, p.1-31`）。
- [Btrfs（B-Tree Filesystem）：Subvolume、佈局、快照與遷移實戰](btrfs.md)（[教學簡報](slides/btrfs.html){ target="_blank" }）：涵蓋五大核心特性、`ext4`/`Btrfs`/`ZFS` 選型、Subvolume（ID `5` 與 `@` 命名慣例）及 Nested vs. Flat 佈局對比與 `/etc/fstab`、Btrfs 快照三大特性與 LVM 快照對比、無內建 rollback 指令之兩種還原方式，以及 `/home` 從 `ext4` 遷移至 Btrfs 並拍時間戳快照實戰（`BTRFS.pdf, p.1-18`）。

## 跨章比較與綜合整理（Comparisons & Syntheses）

*尚未建立。後續匯入相關主題或回存高價值跨章整理後將陸續新增。*
