---
title: "內建指令、工作控制、別名與歷史紀錄"
category: command
tags:
  - shell
  - command
  - exam-trap
sources:
  - "03_Shell.pdf (p.21, 23-25, 45, 47-48)"
updated: 2026-11-23
---

# 內建指令、工作控制、別名與歷史紀錄

> 一句話考試定位摘要：理解為何只有 Shell 內建指令（built-ins）能直接改變當前 Shell 狀態，並掌握前景／暫停／背景工作控制狀態機、目錄堆疊、別名隱藏風險與 `!` 歷史展開的注意事項。（`03_Shell.pdf, p.21-25, 47`）

## 核心定義與原理

### 1. Built-in（內建指令）vs External Commands（外部指令）
由 Shell 本身實作的指令才能改變 Shell 自身的狀態（例如工作目錄、環境變數與工作表）；外部指令則會在獨立行程中執行。（`03_Shell.pdf, p.22-23`）

| 比較項目 | Built-ins（內建指令） | External commands（外部指令） |
| :--- | :--- | :--- |
| **常見範例** | `cd`、`export`、`alias`、`jobs`、`fg`、`bg`、`exit`（`03_Shell.pdf, p.23`） | `ls`、`grep`、`awk`、`sort`、`ssh`、`python`（`03_Shell.pdf, p.23`） |
| **定位與執行方式** | **在 Shell 行程內部直接執行**（`03_Shell.pdf, p.23`） | 透過 `PATH` 或明確路徑尋找，並作為**獨立的行程（separate processes）**執行（`03_Shell.pdf, p.23`） |
| **狀態影響** | **可以修改 Shell 本身的狀態**（`03_Shell.pdf, p.23`） | 無法直接修改呼叫端 Shell 的工作目錄或內部變數（`03_Shell.pdf, p.22-23`） |

使用 `type` 與 `command -V` 查驗指令屬於內建或外部程式：（`03_Shell.pdf, p.23`）

```sh
$ type cd
cd is a shell builtin
$ command -V grep
grep is /usr/bin/grep
```

### 2. 工作控制（Job Control）與 Signal 模型
- **訊號模型（Signal model）**：按下 `Ctrl-Z` 會送出停止訊號（stop signal）；`fg` 與 `bg` 則改變 Shell 管理該已停止或執行中工作的方式。（`03_Shell.pdf, p.24`）
- **背景不等於脫離終端機（Background does not mean detached）**：使用 `&` 在互動式 Shell 啟動背景工作時，終端機訊號與輸出仍然會造成影響。（`03_Shell.pdf, p.21`）

## 流程與架構圖

```mermaid
flowchart LR
    FG["foreground (前景執行)"] -->|"^Z (Ctrl-Z 送出 stop signal)"| SUS["Suspend (暫停中)"]
    SUS -->|"fg"| FG
    SUS -->|"bg"| BG["background (背景執行)"]
    BG -->|"stop"| SUS
```
（來源：`03_Shell.pdf, p.24`）

## 關鍵指令、設定與細節

### 1. Job Control 操作對照表與實例
| 動作 | 指令 / 按鍵 |
| :--- | :--- |
| 在背景啟動工作 | `command &`（`03_Shell.pdf, p.24`） |
| 暫停前景工作 | `Ctrl-Z`（`03_Shell.pdf, p.24`） |
| 列出工作清單 | `jobs`（`03_Shell.pdf, p.24`） |
| 在背景恢復執行 | `bg %1`（`03_Shell.pdf, p.24`） |
| 切回前景執行 | `fg %1`（`03_Shell.pdf, p.24`） |
| 終止工作 | `kill %1` 或 `kill PID`（`03_Shell.pdf, p.24`） |

實際操作範例：（`03_Shell.pdf, p.21, 24`）

```text
# 直接在背景啟動
$ sleep 30 &

# 前景工作暫停後切換至背景再切回前景
$ sleep 300
^Z
[1]+  Stopped                 sleep 300
$ bg %1
[1]+ sleep 300 &
$ jobs
[1]+  Running                 sleep 300 &
$ fg %1
```

### 2. 別名（Aliases）與目錄堆疊（Directory Stack）
- **別名（Aliases）**：在互動式 Shell 減少輸入負擔，複雜邏輯通常應改用函式（functions），且**在腳本中應避免依賴別名**。（`03_Shell.pdf, p.25`）

```sh
alias ll="ls -lah"
alias ..="cd .."
```

- **目錄堆疊（Directory Stack）**：`pushd`、`popd` 與 `dirs` 可在 `bash` 與 `zsh`（投影片亦註記 `tcsh`）中維護目錄堆疊。（`03_Shell.pdf, p.25`）

```text
[~] % dirs
~
[~] % pushd /var/log
/var/log ~
[/var/log] % pushd /etc
/etc /var/log ~
[/etc] % dirs
/etc /var/log ~
[/etc] % popd
/var/log ~
[/var/log] % popd
~
[~] %
```
（來源：`03_Shell.pdf, p.25`）

### 3. 指令歷史紀錄（Command History）
日常互動回呼指令的方式：（`03_Shell.pdf, p.47`）

| 動作 | 指令 / 按鍵（`bash` / `zsh`） |
| :--- | :--- |
| 顯示歷史紀錄 | `history`（`03_Shell.pdf, p.47`） |
| 叫出上一筆指令 | `Up arrow`（上方向鍵）（`03_Shell.pdf, p.47`） |
| 反向搜尋歷史紀錄 | `Ctrl-R`（`03_Shell.pdf, p.47`） |
| 依編號重新執行 | `!n`（`03_Shell.pdf, p.47`） |
| 重新執行上一筆指令 | `!!`（`03_Shell.pdf, p.47`） |
| 重新執行最近以特定文字開頭的指令 | `!text`（`03_Shell.pdf, p.47`） |

## 考點與常見陷阱

!!! tip "考點"
    1. **判別指令類型**：使用 `type <cmd>` 或 `command -V <cmd>` 判斷是 Shell 內建指令還是 `PATH` 下的外部可執行檔。（`03_Shell.pdf, p.23`）
    2. **工作控制循環**：`Ctrl-Z` 是將前景工作「暫停（Stopped）」而非中止；之後可透過 `bg %1` 轉為背景繼續執行，或 `fg %1` 帶回前景。（`03_Shell.pdf, p.24`）

!!! warning "常見陷阱"
    1. **以 `&` 取代系統服務管理器**：背景執行（`&`）並不代表脫離終端機，終端機訊號與輸出依然會影響行程；長駐服務應交給 `systemd` 或專用的行程監控工具（supervisor），不要依賴 `&`。（`03_Shell.pdf, p.21`）
    2. **別名遮蔽破壞性指令**：別名可能會隱藏具破壞性的指令或改變預期參數選項，進行系統管理操作前務必確認別名實際展開的內容。（`03_Shell.pdf, p.25`）
    3. **誤用 `!` 歷史展開與腳本依賴**：執行包含 `!` 的歷史替換前務必檢查實際將執行的指令，在具破壞性或特權指令中特別容易發生意外；而腳本中則應使用明確的指令與變數，不可依賴互動式歷史快捷鍵。（`03_Shell.pdf, p.47-48`）

!!! note "投影片未展開"
    - 投影片提及 `Ctrl-Z` 送出「stop signal」以及 `kill %1` / `kill PID`，但未展開具體的 POSIX signal 名稱或數字編號，亦未詳述 `nohup`、`disown` 等進階脫離終端機指令。（`03_Shell.pdf, p.21, 24`）

## 相關頁面

- 所屬簡報總覽：[簡報總覽：Shells（命令列、管線與文字處理）](src-03-shell.md)
- 指令解析與執行：[Terminal、Shell、Kernel 分層與指令執行流程](shell-terminal-kernel-and-execution.md)
- 啟動檔中的別名：[Shell 執行模式與 Bash 啟動設定檔](shell-startup-files.md)
- 變數與 `export`：[Shell 變數與環境變數](shell-vs-environment-variables.md)

## 資料來源

- `03_Shell.pdf`：p.21–25、p.45、p.47–48
