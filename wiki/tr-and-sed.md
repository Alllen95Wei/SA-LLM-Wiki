---
title: "tr 與 sed 字元轉換與串流編輯"
category: command
tags:
  - shell
  - command
  - exam-trap
sources:
  - "03_Shell.pdf (p.33, 37)"
updated: 2026-11-23
---

# `tr` 與 `sed` 字元轉換與串流編輯

> 一句話考試定位摘要：區分「逐字元（character-by-character）轉換／刪除／壓縮」的 `tr` 與「逐行（line-by-line）支援正規表達式替換」的 `sed`，並留意可攜式字元類別的寫法。（`03_Shell.pdf, p.33, 37`）

## 核心定義與原理

| 工具 | 運作單位 | 能力與適用情境 |
| :--- | :--- | :--- |
| `tr` | **逐字元（character by character）**（`03_Shell.pdf, p.37`） | 字元對應替換、刪除字元、壓縮連續重複字元；**無法理解單字（words）或正規表達式擷取群組（capture groups）**（`03_Shell.pdf, p.33, 37`） |
| `sed` | **逐行（line by line）**（`03_Shell.pdf, p.37`） | 串流編輯器（stream editor），支援基於**正規表達式（regular-expression-based）**的文字替換（`03_Shell.pdf, p.33, 37`） |

## 關鍵指令、設定與細節

```sh
# 將小寫轉換為大寫（使用具 locale 感知與可攜性的字元類別）
printf "%s\n" "Hello World" | tr '[:lower:]' '[:upper:]'

# 刪除 Tab 字元
tr -d '\t' < file.txt

# 壓縮連續重複的空白字元為單一空白
tr -s ' ' < file.txt

# 替換每一行第一次出現的 old 為 new
sed 's/old/new/' file.txt
```
（來源：`03_Shell.pdf, p.37`）

## 考點與常見陷阱

!!! tip "考點"
    1. **`tr` vs `sed` 選型邊界**：只要做字元級的轉換（大小寫）、刪除（`-d`）或壓縮重複字元（`-s`），使用 `tr`；只要涉及單字邊界、正規表達式樣式比對與替換，就必須使用 `sed`。（`03_Shell.pdf, p.37`）
    2. **可攜字元類別**：在重視跨平台／語系（locale）可攜性時，優先使用如 `[:lower:]`、`[:upper:]` 等具 locale 感知的字元類別。（`03_Shell.pdf, p.37`）

!!! warning "常見陷阱"
    - **誤用 `tr` 進行單字或正規群組替換**：`tr` 只做逐字元一對一映射或刪除，並不解析單字結構或正規表達式擷取群組；若需要樣式比對替換應改用 `sed`。（`03_Shell.pdf, p.37`）

!!! note "投影片未展開"
    - 投影片僅示範 `sed 's/old/new/' file.txt`（每行替換第一次出現的比對項目），未展開 `sed` 的其他旗標、修飾符或定址語法。（`03_Shell.pdf, p.37`）

## 相關頁面

- 所屬簡報總覽：[簡報總覽：Shells（命令列、管線與文字處理）](src-03-shell.md)
- 文字工具選型地圖：[檔案目錄指令、UNIX 哲學與文字處理選型地圖](unix-philosophy-and-text-toolkit.md)

## 資料來源

- `03_Shell.pdf`：p.33、p.37
