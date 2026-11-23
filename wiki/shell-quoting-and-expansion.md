---
title: "Shell 引號規則、展開與指令替換"
category: concept
tags:
  - shell
  - concept
  - exam-trap
sources:
  - "03_Shell.pdf (p.16-19, 21, 44-45)"
updated: 2026-11-23
---

# Shell 引號規則、展開與指令替換

> 一句話考試定位摘要：Shell 中多數非預期行為來自展開規則而非指令本身；掌握單引號、雙引號、反斜線、未加引號、`$(command)` 與 Globbing 的差異，並養成 `"$var"` 加引號與使用 `ShellCheck` 檢查的習慣。（`03_Shell.pdf, p.13, 17-19, 44`）

## 核心定義與原理

### 1. 引號規則（Quoting Rules）
| 語法 | 行為意義 | 範例 |
| :--- | :--- | :--- |
| `'text'`（單引號） | 字面純文字（literal text）；**不進行**參數展開與指令替換（`03_Shell.pdf, p.17`） | `echo '$HOME'` |
| `"text"`（雙引號） | **仍會進行**參數展開與指令替換，但可抑制空白分割與路徑展開（`03_Shell.pdf, p.17`） | `echo "$HOME"` |
| `\`（反斜線） | 跳脫（escape）緊接在後的單一字元，或用於跨行接續長指令（`03_Shell.pdf, p.17, 21`） | `echo \*` |
| unquoted（未加引號） | 可能同時發生**欄位分割（field splitting）**與**路徑名稱展開（pathname expansion）** — 務必小心！（`03_Shell.pdf, p.17`） | `printf "%s\n" $files` |

- **預設習慣（Default habit）**：在變數展開外一律加上雙引號，例如 `"$var"` 或 `"${var}"`。（`03_Shell.pdf, p.17, 45`）

### 2. 指令替換（Command Substitution）
- `$(command)` 會將該運算式替換為指令的**標準輸出（standard output）**，並**移除輸出結尾的換行符號（trailing newlines）**。（`03_Shell.pdf, p.18`）
- POSIX Shell Command Language 同時支援 `$(...)` 與反引號（backquotes），但新程式碼應優先採用 **`$(...)`**，因為**巢狀（nesting）書寫簡潔且遠比反引號易讀**。（`03_Shell.pdf, p.16, 18`）

### 3. 路徑名稱展開（Pathname Expansion / Globbing）
Shell 會在執行程式前依樣式配對檔案名稱：（`03_Shell.pdf, p.5, 19`）

| 樣式（Pattern） | 配對內容 | 範例與結果 |
| :--- | :--- | :--- |
| `*` | 任意長度字串（包含空字串）（`03_Shell.pdf, p.19`） | `ls test*` → `test1 test2 test-5 testmess`（`03_Shell.pdf, p.19`） |
| `?` | 任意**單一字元**（`03_Shell.pdf, p.19`） | `ls test?` → `test1 test2`（`03_Shell.pdf, p.19`） |
| `[abc]` | 集合內的任一字元（`03_Shell.pdf, p.19`） | `ls test[12]` → `test1 test2`（`03_Shell.pdf, p.19`） |
| `[!abc]` | **不在**集合內的任一字元（`03_Shell.pdf, p.19`） | — |

## 關鍵指令、設定與細節

### 指令替換與巢狀範例
```sh
now=$(date)
count=$(wc -l < /etc/passwd)
printf "Now: %s\nUsers: %s\n" "$now" "$count"

# 巢狀指令替換
archive="$(basename "$(pwd)").tar.gz"
```
（來源：`03_Shell.pdf, p.18`）

### 反斜線跳脫與跨行續寫（Continuation）
```sh
# 跳脫字面上的 * 字元
touch test\*
ls test\*

# 跨行延續長指令
printf "%s %s\n" \
  "hello" "world"
```
（來源：`03_Shell.pdf, p.21`）

### ShellCheck 靜態分析工具
`ShellCheck` 可用於偵測**未加引號的展開**、可攜性問題、可疑條件測試、未使用變數與常見邏輯錯誤，應在 commit 腳本前執行。（`03_Shell.pdf, p.44`）

```sh
#!/bin/sh
for f in $files; do
  rm $f
done
```
執行檢查：`shellcheck backup.sh` 會回報 `SC2086: Double quote to prevent globbing and word splitting.`（提醒加上雙引號防止 globbing 與斷詞）。（`03_Shell.pdf, p.44`）

## 考點與常見陷阱

!!! tip "考點"
    1. **單引號 vs 雙引號 vs 不加引號**：單引號完全字面保留（`$HOME` 不展開）；雙引號會展開 `$HOME` 與 `$(...)`，但不做空白拆詞與檔名萬用字元展開；不加引號最危險，會同時觸發 field splitting 與 pathname expansion。（`03_Shell.pdf, p.17`）
    2. **反引號 vs `$(...)`**：`$(...)` 會去除尾端換行且容易巢狀（例如 `"$(basename "$(pwd)")"`），是現代 `sh`/`bash` 的首選。（`03_Shell.pdf, p.18`）

!!! warning "常見陷阱"
    1. **將 Globbing 與正規表達式（Regex）混淆**：Shell globbing 比正規表達式簡單得多（例如 `*` 在 globbing 代表任意字串，且否定字元集合使用 `[!abc]`）。若希望由**程式本身（如 `grep`）**而非 Shell 來解讀樣式，**務必將樣式加上引號**。（`03_Shell.pdf, p.19, 34`）
    2. **把 `ShellCheck` 通過當成取代測試**：投影片強調應將 `ShellCheck` 警告視為理解 Shell 語意的提示，但靜態分析無法取代實際測試（not a substitute for testing）。（`03_Shell.pdf, p.44`）

!!! note "投影片未展開"
    - 投影片提及 `${var}` 的安全展開寫法（p.45），並未展開大括號內的高階預設值或字串處理運算子。（`03_Shell.pdf, p.45`）
    - 投影片列出 `ShellCheck` 網址（`shellcheck.net` / `github.com/koalaman/shellcheck`）與 `SC2086` 範例，未展開安裝方式或其他警告代碼清單。（`03_Shell.pdf, p.44`）

## 相關頁面

- 所屬簡報總覽：[簡報總覽：Shells（命令列、管線與文字處理）](src-03-shell.md)
- 五階段執行順序：[Terminal、Shell、Kernel 分層與指令執行流程](shell-terminal-kernel-and-execution.md)
- 變數種類與賦值：[Shell 變數與環境變數](shell-vs-environment-variables.md)
- 搜尋與 `grep` 引號保護：[`head`、`tail` 與 `grep` 檢視與搜尋](head-tail-and-grep.md)

## 資料來源

- `03_Shell.pdf`：p.13, 16–19, 21, 44–45
