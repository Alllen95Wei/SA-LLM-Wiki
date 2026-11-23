---
title: "檔案目錄指令、UNIX 哲學與文字處理選型地圖"
category: synthesis
tags:
  - shell
  - synthesis
  - command
  - exam-trap
sources:
  - "03_Shell.pdf (p.32-33, 41-43, 45)"
updated: 2026-11-23
---

# 檔案目錄指令、UNIX 哲學與文字處理選型地圖

> 一句話考試定位摘要：掌握日常檔案／目錄管理指令與 `rm` 風險、8 種需求對應的 UNIX 文字工具選型地圖、分階段管線設計法，以及「先引用變數、漸進測試、加 `sudo` 前先檢查」的核心實務習慣。（`03_Shell.pdf, p.32-33, 41-43, 45`）

## 核心定義與原理

### 1. UNIX 哲學（The UNIX Philosophy）
**Small tools. Clear interfaces. Composition.**（小工具、清晰介面、組合運用）（`03_Shell.pdf, p.43`）

1. **Do one thing well**：偏好功能專注、行為可預期的工具。（`03_Shell.pdf, p.43`）
2. **Text as glue**：透過管線輕鬆串接彼此獨立的程式。（`03_Shell.pdf, p.43`）
3. **Automate repetition**：將驗證成功的指令序列寫成腳本。（`03_Shell.pdf, p.43`）

- **現代解讀（Modern interpretation）**：在能提升清晰度時善用管線組合；但若問題適合使用結構化 API、資料庫或真正的程式語言，不應強行套用純文字管線。（`03_Shell.pdf, p.43`）

### 2. 分階段管線設計（Think in stages）
建構多階段管線時，每個階段都應具備清晰的輸入與輸出契約（input and output contract）：（`03_Shell.pdf, p.41`）
- Choose fields（挑選欄位） → Filter records（篩選紀錄） → Transform output（轉換輸出） → Sort / aggregate（排序／彙總） → Format for humans or another program（格式化給人或下一個程式使用）。（`03_Shell.pdf, p.41`）

## 關鍵指令、設定與細節

### 1. 常用檔案與目錄操作指令
| 指令 | 用途 |
| :--- | :--- |
| `ls` | 列出目錄內容（`03_Shell.pdf, p.32`） |
| `pwd` | 印出目前工作目錄（`03_Shell.pdf, p.32`） |
| `cd` | 切換工作目錄（`03_Shell.pdf, p.32`） |
| `mkdir` | 建立目錄（`03_Shell.pdf, p.32`） |
| `rmdir` | 移除空目錄（`03_Shell.pdf, p.32`） |
| `cp` | 複製檔案或目錄（`03_Shell.pdf, p.32`） |
| `mv` | 移動或重新命名（`03_Shell.pdf, p.32`） |
| `rm` | 移除檔案或目錄（`03_Shell.pdf, p.32`） |
| `ln` | 建立硬連結（hard link）或符號連結（symbolic link）（`03_Shell.pdf, p.32`） |
| `stat` | 顯示檔案中繼資料（metadata）（`03_Shell.pdf, p.32`） |

實作操作範例：（`03_Shell.pdf, p.32`）

```sh
$ pwd
/home/tsaimh
$ mkdir -p lab/logs
$ cp config.example lab/config
$ stat lab/config
```

### 2. 文字處理選型心智圖（Text Processing: A Mental Map）
| 需求情境 | 選用工具 | 深入頁面 |
| :--- | :--- | :--- |
| 需要檔案最前或最後幾行？ | `head` / `tail` | [`head`、`tail` 與 `grep` 檢視與搜尋](head-tail-and-grep.md) |
| 需要符合特定樣式的文字行？ | `grep` | [`head`、`tail` 與 `grep` 檢視與搜尋](head-tail-and-grep.md) |
| 需要抽取特定欄位或直欄？ | `cut` / `awk` | [`cut` 與 `awk` 欄位擷取比較](cut-vs-awk.md) |
| 需要排序或群組化？ | `sort` / `uniq` | [`sort`、`uniq` 與 `wc` 排序、去重與計數](sort-uniq-and-wc.md) |
| 需要計數（行數、重複次數等）？ | `wc` / `uniq -c` | [`sort`、`uniq` 與 `wc` 排序、去重與計數](sort-uniq-and-wc.md) |
| 需要字元替換或刪除？ | `tr` | [`tr` 與 `sed` 字元轉換與串流編輯](tr-and-sed.md) |
| 需要串流文字編輯？ | `sed` | [`tr` 與 `sed` 字元轉換與串流編輯](tr-and-sed.md) |
| 需要將輸入轉成指令列參數？ | `xargs` | [`xargs` 參數轉換與安全檔名處理](xargs-safe-arguments.md) |
（來源：`03_Shell.pdf, p.33`）

### 3. 整合管線範例
#### 範例 A：探索 `/etc/passwd`（`03_Shell.pdf, p.41`）
```sh
# 1) 顯示登入名稱與登入 Shell
cut -d: -f1,7 /etc/passwd | head

# 2) 統計每種登入 Shell 各有多少帳號使用
cut -d: -f7 /etc/passwd | sort | uniq -c | sort -nr

# 3) 列出登入 Shell 不是 nologin 的帳號
awk -F: '$7 !~ /nologin$/ {print $1, $7}' /etc/passwd
```

#### 範例 B：彙總日誌的前 10 名用戶端 IP（`03_Shell.pdf, p.42`）
```sh
# 假設 access.log 的第 1 欄為用戶端 IP
awk '{print $1}' access.log \
  | sort \
  | uniq -c \
  | sort -nr \
  | head -n 10
```
各階段分工：`awk` 擷取欄位 → `sort` 將相同值排在一起 → `uniq -c` 計算相鄰重複次數 → `sort -nr` 依數量由大到小數值反序排列 → `head` 保留前 10 名。（`03_Shell.pdf, p.42`）

### 4. Shell 核心語法速查表（Shell Cheat Sheet）
| 任務 | 模式寫法 |
| :--- | :--- |
| 變數與匯出 | `name=value ; export name=value`（`03_Shell.pdf, p.45`） |
| 安全展開 | `"$var" ; "${var}"`（`03_Shell.pdf, p.45`） |
| 指令替換 | `$(command)`（`03_Shell.pdf, p.45`） |
| 條件序列 | `cmd1 && cmd2 ; cmd1 || fallback`（`03_Shell.pdf, p.45`） |
| I/O 重導向 | `> >> < 2> 2>&1`（`03_Shell.pdf, p.45`） |
| 管線 | <code>producer &#124; filter &#124; consumer</code>（`03_Shell.pdf, p.45`） |
| 工作控制 | `jobs  bg %1  fg %1`（`03_Shell.pdf, p.45`） |
| 安全搭配 `find` + `xargs` | <code>find ... -print0 &#124; xargs -0 ...</code>（`03_Shell.pdf, p.45`） |

## 考點與常見陷阱

!!! tip "考點"
    1. **最高價值習慣（Most valuable habit）**：變數展開一律加雙引號、漸進式測試指令效果，並在加上 `sudo` 之前先仔細檢查完整指令。（`03_Shell.pdf, p.45`）
    2. **「最常見 Top N」管線固定範式**：`擷取欄位 | sort | uniq -c | sort -nr | head -n N`。（`03_Shell.pdf, p.41-42`）

!!! warning "常見陷阱"
    1. **破壞性刪除指令 `rm`**：`rm` 沒有通用的資源回收筒；在執行遞迴或高權限刪除前，務必再次確認路徑、引號與萬用字元。（`03_Shell.pdf, p.32`）

## 相關頁面

- 所屬簡報總覽：[簡報總覽：Shells（命令列、管線與文字處理）](src-03-shell.md)
- 串流與管線基礎：[標準串流、I/O 重導向與管線（Pipelines）](io-redirection-and-pipelines.md)
- 引號保護與靜態檢查：[Shell 引號規則、展開與指令替換](shell-quoting-and-expansion.md)

## 資料來源

- `03_Shell.pdf`：p.32–33、p.41–43、p.45
