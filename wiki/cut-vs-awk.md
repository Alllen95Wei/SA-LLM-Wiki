---
title: "cut 與 awk 欄位擷取比較"
category: comparison
tags:
  - shell
  - comparison
  - command
  - exam-trap
sources:
  - "03_Shell.pdf (p.33, 35, 41)"
updated: 2026-11-23
---

# `cut` 與 `awk` 欄位擷取比較

> 一句話考試定位摘要：`cut` 適合簡單的固定分隔符號或字元範圍擷取，但缺乏條件判斷與運算能力；`awk` 支援欄位感知篩選、格式化與運算，兩者皆應避免在腳本中用來解析 `ls` 輸出。（`03_Shell.pdf, p.35`）

## 核心定義與原理

| 工具 | 最適合的情境（Best for） | 主要限制（Limitation） |
| :--- | :--- | :--- |
| `cut` | 簡單的固定分隔符號（fixed delimiters）或字元範圍擷取（`03_Shell.pdf, p.35`） | **不支援條件判斷或數值運算**（`03_Shell.pdf, p.35`） |
| `awk` | 欄位感知的篩選（field-aware filtering）、輸出格式化與運算（`03_Shell.pdf, p.35`） | 需要學習較多語法（`03_Shell.pdf, p.35`） |

## 關鍵指令、設定與細節

### 以 `/etc/passwd` 為例對比 `cut` 與 `awk`
`cut` 與 `awk` 皆可指定冒號 `:` 作為欄位分隔符號來擷取帳號、UID、家目錄與登入 Shell：（`03_Shell.pdf, p.35, 41`）

```sh
# 使用 grep + cut 擷取 /etc/passwd 第 1, 3, 6, 7 欄（帳號、UID、家目錄、Shell）
$ grep '^tsaimh:' /etc/passwd | cut -d: -f1,3,6,7
tsaimh:1001:/home/tsaimh:/bin/tcsh

# 使用 awk 達成相同的欄位篩選與輸出
awk -F: '$1=="tsaimh" {print $1, $3, $6, $7}' /etc/passwd

# 使用 awk 進行欄位條件篩選（列出登入 Shell 不是 nologin 結尾的帳號）
awk -F: '$7 !~ /nologin$/ {print $1, $7}' /etc/passwd
```
（來源：`03_Shell.pdf, p.35, 41`）

## 考點與常見陷阱

!!! tip "考點"
    1. **工具選型界線**：單純按固定符號抽欄位（例如 `cut -d: -f1,7`）用 `cut` 最簡潔；只要涉及「條件比對（如 `$1=="tsaimh"`、`$7 !~ /nologin$/`）、重新排版或計算」，就必須改用 `awk`。（`03_Shell.pdf, p.35, 41`）

!!! warning "常見陷阱"
    - **在腳本中解析 `ls` 輸出（Avoid parsing `ls` in scripts）**：`ls` 的輸出是為人類閱讀設計的，在腳本中可能造成歧義（ambiguous）；撰寫腳本時應改用能提供結構化檔案資訊的工具（例如 `stat` 或 `find`）。（`03_Shell.pdf, p.32, 35, 40`）

!!! note "投影片未展開"
    - 投影片提及 `cut` 支援字元範圍（character ranges）擷取，但未列出其具體參數旗標；亦未展開 `awk` 的內建變數或進階運算語法細節。（`03_Shell.pdf, p.35`）

## 相關頁面

- 所屬簡報總覽：[簡報總覽：Shells（命令列、管線與文字處理）](src-03-shell.md)
- 文字工具總覽：[檔案目錄指令、UNIX 哲學與文字處理選型地圖](unix-philosophy-and-text-toolkit.md)
- 整合搜尋與統計：[`head`、`tail` 與 `grep` 檢視與搜尋](head-tail-and-grep.md) ｜ [`sort`、`uniq` 與 `wc` 排序、去重與計數](sort-uniq-and-wc.md)

## 資料來源

- `03_Shell.pdf`：p.33、p.35、p.41
