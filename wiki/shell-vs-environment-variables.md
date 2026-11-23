---
title: "Shell 變數與環境變數"
category: concept
tags:
  - shell
  - concept
  - exam-trap
sources:
  - "03_Shell.pdf (p.14-16)"
updated: 2026-11-23
---

# Shell 變數與環境變數

> 一句話考試定位摘要：釐清僅存在於當前 Shell 的 Shell 變數（Shell variable）與可被子行程繼承的環境變數（Environment variable）差異，並牢記 `export`、賦值無空白及 `$SHELL` 含義等考點。（`03_Shell.pdf, p.14-16`）

## 核心定義與原理

| 比較維度 | Shell variable（Shell 變數） | Environment variable（環境變數） |
| :--- | :--- | :--- |
| **宣告範例** | `name=alice`<br>`echo "$name"` | `export EDITOR=vim`<br>`printenv EDITOR`（`03_Shell.pdf, p.14`） |
| **可見範圍 / 繼承** | 僅對**當前 Shell** 可見；**不會自動被子行程（child processes）繼承**（`03_Shell.pdf, p.14`） | 經過 `export` 的變數會**被此 Shell 啟動的程式（子行程）繼承**（`03_Shell.pdf, p.14`） |
| **查詢方式** | 許多 Shell 中可用 `set` 顯示 Shell 變數（`03_Shell.pdf, p.14`） | `env` / `printenv` 顯示環境變數（`03_Shell.pdf, p.14`） |

## 關鍵指令、設定與細節

### 1. 常用環境變數一覽
| 變數名稱 | 意義說明 |
| :--- | :--- |
| `HOME` | 使用者家目錄（home directory）（`03_Shell.pdf, p.15`） |
| `USER` / `LOGNAME` | 使用者名稱（可用性視系統環境而異）（`03_Shell.pdf, p.15`） |
| `SHELL` | **設定的登入 Shell（configured login shell），不一定是當前正在執行的 Shell**（`03_Shell.pdf, p.15`） |
| `PATH` | 以冒號分隔的目錄清單，Shell 會依此搜尋要執行的指令（`03_Shell.pdf, p.15`） |
| `PWD` | 目前工作目錄（current working directory）（`03_Shell.pdf, p.15`） |
| `EDITOR` / `VISUAL` | 偏好的文字編輯器（`03_Shell.pdf, p.15`） |
| `LANG` / `LC_*` | 語系（locale）與語言設定（`03_Shell.pdf, p.15`） |

檢視 `PATH` 內容範例：（`03_Shell.pdf, p.15`）

```sh
$ printf "%s\n" "$PATH"
/usr/local/bin:/usr/bin:/bin
```

### 2. POSIX 風格 Shell（`sh` / `bash`）變數賦值與使用規範
```sh
# POSIX-style shells
course=sa
month=$(date +%m)
export EDITOR=vim
printf "%s %s\n" "$course" "$month"
```
（來源：`03_Shell.pdf, p.16`）

- 在 `sh`/`bash` 的簡單賦值語法中，**`=` 兩側絕對不可加空白**（例如 `course=sa`）。（`03_Shell.pdf, p.16`）
- 展開變數時應加上雙引號（例如 `"$course"`），除非刻意要觸發欄位分割（field splitting）或路徑名稱展開（pathname expansion）。（`03_Shell.pdf, p.16`）
- 撰寫新的 `sh`/`bash` 程式碼時，指令替換應優先使用 `$(command)` 而非反引號（backticks）。（`03_Shell.pdf, p.16`）

## 考點與常見陷阱

!!! tip "考點"
    1. **`export` 的本質**：未使用 `export` 的變數（`name=alice`）只存在於目前 Shell 本身，不會傳給由該 Shell 啟動的子程式；必須 `export` 才會進入環境變數供子行程繼承。（`03_Shell.pdf, p.14`）
    2. **檢視工具區分**：`env` 與 `printenv` 顯示的是環境變數；`set` 在許多 Shell 中可列出 Shell 變數。（`03_Shell.pdf, p.14`）

!!! warning "常見陷阱"
    1. **誤以為 `$SHELL` 等於目前正在跑的 Shell**：投影片明確提醒 `SHELL` 代表的是帳號「設定好的登入 Shell（configured login shell）」，並不代表當前執行的 Shell（例如從 `bash` 內手動切進另一種 Shell 執行時，`$SHELL` 可能仍維持原本的登入 Shell）。（`03_Shell.pdf, p.15`）
    2. **在 `=` 兩旁誤加空白**：在 `sh`/`bash` 中寫成 `course = sa` 會被當作執行名為 `course` 的指令，而非變數賦值。（`03_Shell.pdf, p.16`）

## 相關頁面

- 所屬簡報總覽：[簡報總覽：Shells（命令列、管線與文字處理）](src-03-shell.md)
- 引號與展開規則：[Shell 引號規則、展開與指令替換](shell-quoting-and-expansion.md)
- 啟動設定檔載入時機：[Shell 執行模式與 Bash 啟動設定檔](shell-startup-files.md)
- 為何 `export` 必須是內建指令：[內建指令、工作控制、別名與歷史紀錄](shell-builtins-aliases-and-job-control.md)

## 資料來源

- `03_Shell.pdf`：p.14–16
