---
marp: true
theme: sa-wiki
paginate: true
size: 16:9
header: "System Administration (Debian 13) — 教學型簡報"
footer: "知識來源：wiki/shell-programming.md（03_Shell_Programming.pdf） ｜ System Administration LLM Wiki"
---

<!-- _class: lead -->
<!-- _paginate: skip -->

# Shell 腳本入門教學：流程控制、健壯性與 `sed` / `awk`

給初學者的 Debian Linux 可攜腳本設計指南

- **核心強調**：寫出可攜、易讀且行為可預測的 Shell 腳本，勝過難懂的炫技單行指令（`03_Shell_Programming.pdf, p.2`）
- **實作方針**：當工作主要由現有系統指令完成時，Shell 最能發揮威力（`03_Shell_Programming.pdf, p.5`）
- [閱讀完整 Wiki 講義](../shell-programming/)

---

## 學習地圖：這份簡報會帶你學會什麼？

1. **怎麼寫出並執行第一支腳本？** — 直譯器、`#!/bin/sh` Shebang 與兩種執行方式
2. **如何安全地傳入參數與設定預設值？** — 參數展開防呆、字串裁切與 `"$@"` vs `"$*"`
3. **腳本如何判斷條件與重複執行？** — `[ ... ]` 測試規則、算術、分支與迴圈防雷
4. **程式變長了怎麼模組化與收選項？** — 函式作用域、Subshell 陷阱與 `getopts`
5. **如何避免腳本寫到一半崩潰或留垃圾？** — 錯誤導流、Signal、`trap` 清理、`sh -x` 與 `ShellCheck`
6. **如何用 Regex、`sed`、`awk` 處理文字？** — 兩套樣式語言與資料流處理入門
7. **如何重構一份脆弱的實戰腳本？** — 主機健康檢查重構對比與 7 點檢查清單

---

<!-- _class: divider -->

# 單元一：認識 Shell 腳本與 Shebang

從把重複指令寫成檔案開始，了解為什麼系統腳本愛用 `#!/bin/sh`。

---

## 為什麼要寫 Shell 腳本？為什麼選 `#!/bin/sh`？

<div class="cols">
<div>

### Shell 腳本扮演的角色（`p.5`）
- 自動化重複的命令列任務，不必為此開發完整應用程式
- 用管線、重導向與結束狀態碼（exit status）把現有工具串接在一起
- 非常適合**系統管理、部署、監控與批次任務**
- Shell 是**直譯器（interpreter）**，會依序把腳本內容當成指令執行

</div>
<div>

### 本課程的語言選擇（`p.6`）
- **使用 POSIX 風格 `sh`（`#!/bin/sh`）**：可在 Debian 與眾多類 Unix 系統間可攜
- **避開特定 Shell 專屬功能**（例如 `bash` 的擴充與陣列、`zsh` 擴充語法），除非明確需要
- `/bin/sh` 是一個**介面規範**，底層實作會因作業系統而異

</div>
</div>

---

## 寫出第一支腳本與兩種執行方式

```sh
#!/bin/sh
for i in 1 2 3; do
    printf 'Hello world %s\n' "$i"
done
```

<pre class="mermaid">
flowchart TD
    S["腳本檔案 hello.sh"] --> A["chmod +x hello.sh 後執行 ./hello.sh"]
    S --> B["直接執行 sh hello.sh"]
    A --> A1["讀取第 1 行 Shebang (#!/bin/sh) 啟動直譯器"]
    B --> B1["明確指定 sh 直譯器 (不需檔案具備可執行權限)"]
</pre>

> **初學細節（`p.7-8`）**：格式化輸出優先用 `printf`（比 `echo` 更可預測）。系統腳本常用絕對路徑 `#!/bin/sh`；若寫 `#!/usr/bin/env bash` 會受 `PATH` 影響，重視可重現性時應固定絕對路徑。

---

<!-- _class: divider -->

# 單元二：變數、防呆展開與腳本參數

不靠外部指令就能設定預設值、裁切路徑，並正確轉傳每一筆參數。

---

## 變數、引號與指令替換快速回顧

- **賦值與作用域（`p.10`）**：`name=value`（目前 Shell 可見）、`export name=value`（子行程可繼承）；**`=` 兩側絕不可加空白**。
- **引號守則（`p.11`）**：`'text'` 完全保留字面；`"text"` 展開變數與指令但不依空白切詞；**最佳實踐是一律引用 `"$var"`、`"$(command)"` 與 `"$@"`**。
- **指令替換（`p.12`）**：`now=$(date)` 會把指令的 `stdout` 放進來並**移除尾端換行**，比舊式反引號易讀又容易巢狀書寫。

```sh
export PAGER=/usr/bin/less
count=$(wc -l < /etc/passwd)
printf 'Users: %s\n' "$count"
```

---

## 參數展開（上）：設定預設值與必填防呆

當變數**未設定（unset）或為空字串**時，直接在 `${...}` 裡設定保護機制（`03_Shell_Programming.pdf, p.13`）：

| 語法 | 當變數未設定或為空時的行為 | 實用範例 |
| :--- | :--- | :--- |
| `${var:-default}` | 回傳 `default`，**不改動**原變數 | `config=${CONFIG_FILE:-/etc/myapp.conf}` |
| `${var:=default}` | **賦值** `default` 給 `var` 後回傳 | 設定缺少時的初始值 |
| `${var:?message}` | 印出 `message` 到 `stderr` 並**中止展開** | `: "${HOME:?HOME must be set}"` |
| `${var:+alternate}` | 僅當變數**有值且非空**才改用 `alternate` | 有開啟旗標時才附帶額外參數 |

---

## 參數展開（下）：不呼叫外部工具就裁切字串

在 Shell 內建語法就能算長度、去前綴或去後綴（`03_Shell_Programming.pdf, p.14`）：

| 語法 | 效果 | 小口訣 |
| :--- | :--- | :--- |
| `${#var}` | 字串長度 | `#` 放前面代表算數量 |
| `${var#pattern}` / `${var##pattern}` | 移除開頭**最短** / **最長**前綴 | `#` 在 `$` 左邊 → 去左邊 |
| `${var%pattern}` / `${var%%pattern}` | 移除結尾**最短** / **最長**後綴 | `%` 在右邊 → 去右邊 |

```sh
path=/var/log/system.log
base=${path##*/}    # 變成 system.log（從左移除最長 */）
stem=${base%.log}   # 變成 system（從右移除最短 .log）
```

---

## 位置參數與經典陷阱：`"$@"` vs `"$*"`

<div class="cols">
<div>

### 認識位置與特殊參數（`p.15`）
- `$0`：腳本名稱（例如 `./arg.sh`）
- **`$1, $2, ...`**：傳入的參數（**從 `$1` 開始！**）
- `$#`：位置參數總數
- `$?`：上一個指令的結束狀態碼
- `$$`：目前 Shell 的 PID
- `$!`：最近背景管線的 PID

</div>
<div>

### 收到 `one`、`"two words"`、`three` 時（`p.16`）
```sh
for x in "$@"; do printf '<%s>\n' "$x"; done
# <one>
# <two words>
# <three>

printf '<%s>\n' "$*"
# <one two words three>
```
> **轉傳參數必用 `"$@"`**，才能完整保留每個參數原本的空白與邊界！

</div>
</div>

---

<!-- _class: divider -->

# 單元三：條件判斷、算術與迴圈防雷

Shell 沒有傳統布林型別——所有判斷都圍繞「指令結束狀態碼」與「空白／引號規則」。

---

## `test` / `[ ... ]` 與檔案、字串、整數測試

在 Shell 裡，**`0` 代表成功（真）、非 `0` 代表失敗（假）**；`[` 本質上是一道指令，而結尾的 `]` 是它需要的最後一個參數，所以 **`[`、運算子與 `]` 周圍必須有空白**，且變數一定要加雙引號（`p.18-19`）。

| 類別 | 常用寫法（`p.20-22`） | 初學者最容易踩的坑 |
| :--- | :--- | :--- |
| **檔案測試** | `-e` 存在、`-f` 一般檔、`-d` 目錄、`-L` 軟連結、`-r/-w/-x`、`-s` 非空、`-nt` 較新 | 忘記加引號 `[ -f $file ]` 遇到空變數會報語法錯 |
| **字串測試** | `-z "$s"` 空、`-n "$s"` 非空、`"$a" = "$b"`、`"$a" != "$b"` | POSIX `sh` 請用 **`=`** 不要寫 `==`；勿用 `<` / `>` |
| **整數比較** | `-eq`、`-ne`、`-gt`、`-ge`、`-lt`、`-le` | 誤拿 `=` 比數字或誤拿 `-eq` 比文字 |

---

## 組合條件、算術 `$((...))` 與 `if` / `case` 分支

<div class="cols">
<div>

### 組合測試與整數算術（`p.23-24`）
- **組合多條件**：用 Shell 的 **`&&` 與 `||`**，避開歷史有歧義的 `-a` / `-o`：
  ```sh
  if [ -r "$file" ] && [ -s "$file" ]; then
      process "$file"
  fi
  ```
- **整數算術 `$(( ... ))`**：比 `expr` 更清楚，括號內變數名通常不必寫 `$`：
  ```sh
  count=$((count + 9))
  ```

</div>
<div>

### `if / elif / else` 與 `case`（`p.25-26`）
```sh
case "$answer" in
    [Yy]|[Yy][Ee][Ss]) printf 'yes\n' ;;
    [Nn]|[Nn][Oo])     printf 'no\n' ;;
    *) printf 'invalid\n' >&2 ;;
esac
```
> **重要區別**：`case` 使用的是 **Shell Glob 樣式**（不是正規表達式），且可攜 `sh` 每個分支以 `;;` 結束（`p.26`）。

</div>
</div>

---

## `for` 迴圈與 `while IFS= read -r` 防雷指南

<div class="cols">
<div>

### 1. `for` 迴圈不要解析 `ls`（`p.27`）
- 列舉集合：`for x in red green blue; do ...`
- 安全掃描檔案：`for f in *.log; do ...`

<blockquote class="trap">
<strong>絕對避開 <code>for f in $(ls)</code></strong>：檔名一旦含空白、Tab、換行或萬用字元就會直接拆壞（<code>p.27</code>）。
</blockquote>

</div>
<div>

### 2. 逐行讀取與管線歸零陷阱（`p.28`）
```sh
while IFS= read -r line; do
    printf 'line: %s\n' "$line"
done < input.txt
```
- `IFS=` 保護首尾空白；`-r` 不直譯反斜線
- **陷阱**：`cat data.txt | while ...` 會開 Subshell，**迴圈裡改的變數結束後全歸零**！應改用 `< input.txt` 重導向。

</div>
</div>

---

## 安全互動輸入與暫存目錄自動清理

<div class="cols">
<div>

### 安全讀取使用者輸入（`p.29`）
```sh
printf "Delete temp files? [y/N] "
IFS= read -r answer
case "$answer" in
    [Yy]|[Yy][Ee][Ss]) rm -f -- "$tmpfile" ;;
    *) printf 'Cancelled\n' ;;
esac
```
- 用 `printf` 取代 `echo -n`
- 非預期輸入預設採較安全動作
- 支援 `--` 的指令在檔名前加 `--`

</div>
<div>

### 安全使用暫存檔與 `trap`（`p.30`）
```sh
tmpdir=$(mktemp -d) || exit 1
trap 'rm -rf "$tmpdir"' EXIT HUP INT TERM

tmpfile=$tmpdir/output.txt
printf 'program output\n' > "$tmpfile"
```
- 讓 `mktemp` 選唯一名稱，放入私有暫存目錄
- **建立後立刻註冊 `trap` 清理**

</div>
</div>

---

<!-- _class: divider -->

# 單元四：函式、Subshell 與 `getopts`

把重複邏輯抽成函式、看懂變數作用域，並標準化解析 `-v`、`-f` 命令列選項。

---

## 函式基本寫法與 Same shell vs Subshell 作用域

<pre class="mermaid">
flowchart TD
    P["父 Shell (Parent Shell)"] --> F["Same shell (例如函式呼叫 / 重導向迴圈 &lt; file)"]
    P --> SUB["Subshell / child process (例如管線 | 右側)"]
    F --> F1["直接共用並修改變數 -> 結束後變更留存"]
    SUB --> S1["僅接收環境變數副本 -> 內部修改不會回傳給父 Shell"]
</pre>

- **函式規則（`p.32-33`）**：必須在首次呼叫前定義；有自己獨立的 `$1, $2, ...`；預設直接與呼叫端共用 Shell 變數，但**不要依賴管線（Subshell）內的變數修改在管線後還看得到**。

---

## 函式回傳值觀念：`return`（狀態碼）vs `stdout`（文字）

初學者最常誤解 `return`：在 Shell 裡，**`return` 只能回傳 `0–255` 的結束狀態碼，不是一般函數回傳值**！要回傳文字資料請印到 **`stdout`**（`03_Shell_Programming.pdf, p.34`）。

```sh
copy_if_readable() {
    [ "$#" -eq 2 ] || return 2     # 先驗證參數數量
    [ -r "$1" ] || return 3        # 再驗證輸入檔可讀
    cp -- "$1" "$2"
}

copy_if_readable source.txt backup.txt
status=$?
```

> 另一常見實踐：在函式開頭就先檢查 `$#`，避免存取到空的 `$1`、`$2`（`03_Shell_Programming.pdf, p.34`）。

---

## 用 `getopts` 優雅解析命令列選項

<pre class="mermaid">
flowchart LR
    G["while getopts 'vf:' opt"] -->|"依序取出選項"| C["case '$opt' in"]
    C -->|"v"| V1["verbose=1"]
    C -->|"f (冒號代表需要參數)"| F1["file=$OPTARG"]
    C -->|"?"| ERR["exit 2 (無效選項)"]
    G -->|"迴圈結束後"| SH["shift $((OPTIND - 1)) 移掉已解析選項"]
</pre>

```sh
verbose=0; file=
while getopts 'vf:' opt; do
    case "$opt" in
        v) verbose=1 ;;
        f) file=$OPTARG ;;
        ?) exit 2 ;;
    esac
done
shift $((OPTIND - 1))
```
（來源：`03_Shell_Programming.pdf, p.35`）

---

<!-- _class: divider -->

# 單元五：錯誤處理、Signal 與除錯工具

讓腳本在失敗時看得見原因、不殘留垃圾，也不會誤把密碼印進紀錄。

---

## 錯誤處理與 Shell 選項（`set -u` / `set -x`）

- **錯誤處理起點（`p.37`）**：檢查關鍵步驟的失敗，將診斷輸出送到 **`stderr`（`printf ... >&2`）**，回傳有意義的非零結束碼，**別用長管線或未檢查的指令替換掩蓋失敗**。

```sh
if ! cp -- "$src" "$dst"; then
    printf 'copy failed: %s -> %s\n' "$src" "$dst" >&2
    exit 1
fi
```

| 選項 | 效果（`p.38`） | 注意事項 |
| :--- | :--- | :--- |
| `set -u` | 展開未設定變數直接報錯 | 選填變數需搭配預設值（如 `${PORT:-8080}`） |
| `set -x` | 在展開之後、執行前追蹤印出指令 | **可能把密碼或 Token 印進日誌** |

> **課程建議**：對重要操作做明確檢查；刻意使用 Shell 選項，不要當成萬能魔法開關（`p.38`）。

---

## 認識常見 Signal 與 `trap` 清理機制

用 `trap cleanup EXIT HUP INT TERM` 註冊清理函式（使用訊號名稱比數字易讀，`p.39`）：

| Signal | 常見意義（`p.40`） | 能否被 `trap` 捕捉（Catchable）？ |
| :--- | :--- | :--- |
| `SIGHUP` | 終端機／工作階段結束，或 reload 慣例 | Yes |
| `SIGINT` | 來自 `Ctrl+C` 的中斷 | Yes |
| `SIGQUIT` | 來自終端機的退出 | Yes |
| `SIGTERM` | 禮貌性的終止請求 | Yes |
| `SIGKILL` | **立即強制終止** | **No（不可捕捉或忽略！）** |
| `SIGSTOP` | **暫停行程** | **No（不可捕捉或忽略！）** |

---

## 執行追蹤（`sh -x`）與 `ShellCheck` 開發流程

<pre class="mermaid">
flowchart LR
    E["Edit (編輯腳本)"] --> SC["shellcheck (靜態分析檢查)"]
    SC --> T["test with representative inputs (代表性輸入測試)"]
    T --> R["review exit status and logs (檢視結束碼與日誌)"]
</pre>

<div class="cols">
<div>

### Tracing 追蹤除錯（`p.41`）
```sh
$ sh -x script.sh
+ config=/etc/app.conf
+ '[' -r /etc/app.conf ']'
```
- 腳本內也可用 `set -x` 開、`set +x` 關
- **小心印出密碼、Token 等機密！**

</div>
<div>

### `ShellCheck` 靜態分析（`p.42`）
```sh
$ shellcheck script.sh
```
- 抓出引號漏寫、可疑展開、可攜問題與無效結構
- **把警告當成需要調查的線索**，而非盲目照改

</div>
</div>

---

<!-- _class: divider -->

# 單元六：正規表達式（Regex）、`sed` 與 `awk`

分清楚檔名萬用字元與文字匹配規則，學會逐行編輯與欄位彙總的兩大經典工具。

---

## 正規表達式核心：Globbing vs Regex 與工具方言

- **兩套不同的樣式語言（`p.44`）**：**Shell globbing（如 `*.log`）用來挑檔名**；**Regular expressions 用來匹配資料內部的文字**（由串接、選擇、重複三概念組成）。
- **常用 ERE 運算子（`p.45`）**：`.`（任一字元）、`[abc]` / `[^abc]`（字元集合與排除）、`^` / `$`（行首行尾）、`*`（0+）、`+`（1+）、`?`（0/1）、`{m,n}`（次數範圍）；**由左至右閱讀樣式**（`p.46`）。

| 工具 | 預設正規表達式方言（`p.47`） | 延伸正規寫法 |
| :--- | :--- | :--- |
| `grep` | BRE | `grep -E` |
| `sed` | BRE | `sed -E`（現代 GNU 等系統常見） |
| `awk` | **ERE（內建即是 ERE）** | 內建支援 |

> 教學或除錯樣式時，永遠先說明正在用哪個工具與哪種 Regex 方言（`p.47`）。

---

## `sed` 串流編輯入門：位址、印出／刪除與替換

`sed` 一次讀一行進入 pattern space 處理，**不會改動原始檔案**（除非顯式啟用原地編輯）；加 `-n` 可抑制自動輸出（`p.49, 51`）。

<div class="cols">
<div>

### 位址選取、印出與刪除（`p.50-51`）
- 行號 `10`、範圍 `10,20`、Regex `/error/`、混合範圍 `10,/END/`
- 只印選中行：`sed -n '/^root:/p' /etc/passwd`
- 刪除註解行：`sed '/^#/d' file.txt`
- 多指令串接：`sed -e '1,10d' -e 's/yellow/black/g' input.txt`

</div>
<div>

### 替換旗標與增刪行（`p.52-53`）
```sh
sed 's/old/new/' file.txt         # 每行第 1 個
sed 's/old/new/g' file.txt        # g：該行全部
sed -n 's/error/ERROR/p' log.txt  # p：印出有改的行
```
- 旗標 `N`：只換第 N 次出現
- `i`（前插）、`a`（後附）、`c`（整行替換）；複雜邏輯放獨立腳本檔

</div>
</div>

---

## `awk` 欄位處理入門：`pattern { action }` 與內建變數

<pre class="mermaid">
flowchart LR
    B["BEGIN (讀取輸入前)"] --> R["逐筆讀紀錄並拆欄位 ($1, $2, ...)"]
    R --> P{"pattern 符合? (/regex/ 或運算式)"}
    P -- "符合" --> A["執行 { action }"]
    P -- "不符合" --> N["下一筆紀錄"]
    A --> N
    N -->|"處理完畢"| E["END (彙總輸出)"]
</pre>

- 沒寫 `pattern` 代表「每一筆紀錄都執行」；沒寫 `{ action }` 代表「印出該紀錄」（`p.54`）。
- 常用內建變數（`p.56`）：`$0`（整筆紀錄）、`$1..`（各欄位）、`NF`（欄位數）、`NR`（紀錄編號）、`FILENAME`、`FS`（輸入分隔符）、`OFS`（輸出分隔符）。

```sh
awk -F: '$1 == "root" { print "uid=" $3, "home=" $6, "shell=" $7 }' /etc/passwd
```
（當篩選與欄位邏輯相關時，一個 `awk` 常比一長串 `grep + cut` 更清楚，`p.57`。）

---

<!-- _class: divider -->

# 單元七：實戰案例重構與最佳實踐

把容易失敗、看語系臉色的脆弱腳本，改寫成乾淨又容易測試的小工具。

---

## 實戰重構：主機健康檢查腳本

<div class="cols">
<div>

### 脆弱寫法 vs 較佳寫法（`p.59`）
- **脆弱**：解析給人看的 `ping` 統計文字、寫可預測暫存檔、變數不加引號、寫死單一實作郵件邏輯
- **較佳**：**直接用 `ping` 的結束狀態碼**、驗證恰有 1 個參數、變數全加引號、回傳狀態碼交給排程器決定通知

</div>
<div>

```sh
#!/bin/sh
usage() { printf 'Usage: %s HOST\n' "$0" >&2; }
[ "$#" -eq 1 ] || { usage; exit 2; }
host=$1
if ping -c 3 "$host" >/dev/null 2>&1; then
    printf '%s is reachable\n' "$host"; exit 0
else
    printf '%s is unreachable\n' "$host" >&2; exit 1
fi
```

</div>
</div>

> **為什麼更好（`p.60`）**：免用暫存檔、不依賴受語系（localization）影響的文字解析，把通知邏輯拆出去後更容易單元測試。

---

## 總結：初學者 7 大腳本檢查清單

1. 選定目標直譯器（如 `#!/bin/sh`）並嚴格遵守其語言規範（`03_Shell_Programming.pdf, p.61`）
2. **變數展開預設加雙引號**；轉傳參數用 `"$@"`（`p.61`）
3. **檢查關鍵指令的結束狀態碼**；錯誤訊息送至 `stderr`（`p.61`）
4. 用安全迴圈與輸入處理，**不解析 `ls` 輸出**或依賴 `echo` 怪癖（`p.61`）
5. 用 **`trap`** 清理暫存資源（`p.61`）
6. 用 **`ShellCheck`** 靜態分析後，再用真實邊界情況測試（`p.61`）
7. 在能讓邏輯**更清晰**時才用 `sed` 與 `awk`，而不只是為了寫得更短（`p.61`）

[閱讀完整 Wiki 講義](../shell-programming/)

<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs';

  mermaid.initialize({
    startOnLoad: false,
    theme: 'default',
    securityLevel: 'loose',
  });

  async function renderVisibleMermaid() {
    const candidates = Array.from(
      document.querySelectorAll('.mermaid:not([data-processed="true"])')
    ).filter((el) => {
      const rect = el.getBoundingClientRect();
      return rect.width > 0 && rect.height > 0;
    });

    if (candidates.length === 0) return;

    for (const node of candidates) {
      node.dataset.rendering = 'true';
    }
    try {
      await mermaid.run({ nodes: candidates });
    } catch (err) {
      console.error('Mermaid render error:', err);
    }
  }

  let scheduled = false;
  function scheduleRender() {
    if (scheduled) return;
    scheduled = true;
    requestAnimationFrame(() => {
      scheduled = false;
      renderVisibleMermaid();
    });
  }

  window.addEventListener('load', scheduleRender);
  window.addEventListener('hashchange', () => setTimeout(scheduleRender, 50));
  window.addEventListener('keydown', () => setTimeout(scheduleRender, 50));
  window.addEventListener('click', () => setTimeout(scheduleRender, 50));

  const observer = new MutationObserver(scheduleRender);
  if (document.body) {
    observer.observe(document.body, {
      attributes: true,
      subtree: true,
      attributeFilter: ['class', 'style'],
    });
  }
</script>
