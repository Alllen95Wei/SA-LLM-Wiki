---
title: "控制運算子與結束狀態碼（Exit Status）"
category: concept
tags:
  - shell
  - concept
sources:
  - "03_Shell.pdf (p.20, 45)"
updated: 2026-11-23
---

# 控制運算子與結束狀態碼（Exit Status）

> 一句話考試定位摘要：Shell 以結束狀態碼（exit status，`$?`）的 `0` 代表成功、非 `0` 代表失敗或特殊條件，並透過 `;`、`&&` 與 `||` 控制指令串接的執行條件。（`03_Shell.pdf, p.20`）

## 核心定義與原理

### 1. 結束狀態碼（Exit Status）慣例
- 每一個指令執行結束後都會回傳一個結束狀態碼，可透過特殊參數 `$?` 讀取最近一次執行指令的結果。（`03_Shell.pdf, p.20`）
- **通用慣例**：結束狀態碼 **`0` 代表成功（success）**；**非零值（non-zero）代表失敗或某種特殊條件（failure or special condition）**。（`03_Shell.pdf, p.20`）

```sh
$ false
$ echo $?
1
$ true
$ echo $?
0
```
（來源：`03_Shell.pdf, p.20`）

### 2. 控制運算子（Control Operators）
| 運算子 | 行為意義 | 範例 |
| :--- | :--- | :--- |
| `;` | 無論前一個指令的結束狀態為何，一律接著執行下一個指令（`03_Shell.pdf, p.20`） | `cd /tmp; pwd` |
| `&&` | 只有在第一個指令**成功**（結束狀態為 `0`）時，才執行下一個指令（`03_Shell.pdf, p.20`） | `make && make test` |
| `||` | 只有在第一個指令**失敗**（結束狀態為非 `0`）時，才執行下一個指令（`03_Shell.pdf, p.20`） | `mkdir out || exit 1` |

## 關鍵指令、設定與細節

常見條件序列整理（來自 Shell Cheat Sheet）：（`03_Shell.pdf, p.45`）

```sh
# 第一步成功才執行第二步
cmd1 && cmd2

# 第一步失敗才執行備援或錯誤處理
cmd1 || fallback
```

## 考點與常見陷阱

!!! tip "考點"
    1. **Linux / Shell 的成功碼是 `0`**：不要與許多程式語言的布林真值 `1` 混淆——在 Shell 結束狀態碼中，`true` 回傳 `0`、`false` 回傳 `1`。（`03_Shell.pdf, p.20`）
    2. **`;` vs `&&`**：使用 `;` 時即使前方失敗（例如 `cd` 失敗）仍會繼續執行後續指令；若後續步驟依賴前一步成功，必須用 `&&`。（`03_Shell.pdf, p.20`）

## 相關頁面

- 所屬簡報總覽：[簡報總覽：Shells（命令列、管線與文字處理）](src-03-shell.md)
- 指令執行順序：[Terminal、Shell、Kernel 分層與指令執行流程](shell-terminal-kernel-and-execution.md)
- 整合速查表：[檔案目錄指令、UNIX 哲學與文字處理選型地圖](unix-philosophy-and-text-toolkit.md)

## 資料來源

- `03_Shell.pdf`：p.20、p.45
