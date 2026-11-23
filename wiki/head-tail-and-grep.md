---
title: "head、tail 與 grep 檢視與搜尋"
category: command
tags:
  - shell
  - command
sources:
  - "03_Shell.pdf (p.33-34)"
updated: 2026-11-23
---

# `head`、`tail` 與 `grep` 檢視與搜尋

> 一句話考試定位摘要：`head` 與 `tail` 用於擷取檔案開頭／結尾行或即時追蹤日誌，`grep` 用於比對符合樣式的文字行（需保護樣式不被 Shell 預先展開），且在 `systemd` 系統上應留意以 `journalctl` 讀取服務日誌。（`03_Shell.pdf, p.33-34`）

## 核心定義與原理

- **`head` / `tail`**：當只需要檔案的前幾行或後幾行時使用；`tail -f` 可持續追蹤正在成長的日誌檔案。（`03_Shell.pdf, p.33-34`）
- **`grep`**：當需要篩選出符合特定樣式（pattern）的文字行時使用。（`03_Shell.pdf, p.33`）
- **`systemd` 系統實務補充**：在採用 `systemd` 的系統（如 Debian）上，許多服務的日誌讀取通常已改由 `journalctl` 取代直接讀取日誌檔。（`03_Shell.pdf, p.34`）

## 關鍵指令、設定與細節

```sh
# 讀取前 20 行 / 後 50 行
head -n 20 file.txt
tail -n 50 file.txt

# 即時追蹤成長中的日誌檔案
tail -f /var/log/messages

# 搜尋並顯示行號
grep -n "sshd" file.txt

# 僅列出包含符合項目的檔案名稱
grep -l "ERROR" *.log
```
（來源：`03_Shell.pdf, p.34`）

### `grep` 常用旗標與樣式選項
| 旗標 | 作用說明 |
| :--- | :--- |
| `-n` | 顯示符合行所在的行號（`03_Shell.pdf, p.34`） |
| `-l` | 僅列出包含符合內容的檔案名稱（`03_Shell.pdf, p.34`） |
| `-F` | 使用**固定字串（fixed strings）**比對，適用於不需要正規表達式時（`03_Shell.pdf, p.34`） |
| `-E` | 在適當需求下啟用**延伸正規表達式（extended regular expressions）**（`03_Shell.pdf, p.34`） |

## 考點與常見陷阱

!!! tip "考點"
    1. **依需求選擇 `grep -F` 或 `-E`**：僅需純文字子字串比對時用 `-F`；需要使用延伸正規表達式時才用 `-E`。（`03_Shell.pdf, p.34`）
    2. **系統日誌工具觀念**：在 `systemd` 系統中，許多服務的日誌是以 `journalctl` 查詢而非直接檢視傳統文字日誌檔。（`03_Shell.pdf, p.34`）

!!! warning "常見陷阱"
    - **未將搜尋樣式加上引號**：執行 `grep` 時必須將樣式加引號（例如 `"sshd"` 或 `'^tsaimh:'`），防止 Shell 在第 2 階段展開就先把特殊字元（如 `*`、`?`）當成檔名萬用字元展開。（`03_Shell.pdf, p.19, 34-35`）

!!! note "投影片未展開"
    - 投影片未展開 `grep -E` 的延伸正規表達式詳細語法規則，亦未展開 `journalctl` 的各項篩選旗標。（`03_Shell.pdf, p.34`）

## 相關頁面

- 所屬簡報總覽：[簡報總覽：Shells（命令列、管線與文字處理）](src-03-shell.md)
- 文字工具總覽：[檔案目錄指令、UNIX 哲學與文字處理選型地圖](unix-philosophy-and-text-toolkit.md)
- 引號與 Globbing 區別：[Shell 引號規則、展開與指令替換](shell-quoting-and-expansion.md)
- 搭配欄位過濾：[`cut` 與 `awk` 欄位擷取比較](cut-vs-awk.md)

## 資料來源

- `03_Shell.pdf`：p.19、p.33–35
