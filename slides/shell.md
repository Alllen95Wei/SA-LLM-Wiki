---
marp: true
theme: sa-wiki
paginate: true
size: 16:9
header: "System Administration (Debian 13) — 教學型簡報"
footer: "知識來源：wiki/shell.md（03_Shell.pdf） ｜ System Administration LLM Wiki"
---

<!-- _class: lead -->
<!-- _paginate: skip -->

# Shell 入門教學：命令列、管線與文字處理

給初學者的 Debian Linux 命令列基礎指南

- **課程強調**：先建立可攜的通用觀念（portable ideas first），再學特定 Shell 的便利功能（`03_Shell.pdf, p.2`）
- **可攜腳本基準**：撰寫腳本優先以 `sh` 為目標，互動環境則使用系統提供的 Shell（`03_Shell.pdf, p.11`）
- [閱讀完整 Wiki 講義](../shell/)

---

## 學習地圖：這份簡報會帶你懂什麼？

1. **命令是怎麼跑起來的？** — 分辨 Terminal、Shell 與 Kernel，以及按下 Enter 後的五個步驟
2. **為何有時改設定沒生效？** — 搞懂三種執行模式與 Bash 啟動設定檔的讀取順序
3. **為何指令行為常出乎意料？** — 理解變數作用域、單雙引號、展開順序與結束狀態碼
4. **如何與工作／歷史互動？** — 內建指令與外部指令差異、工作控制、別名與目錄堆疊
5. **如何像水管一樣串接程式？** — 標準串流 `0/1/2`、I/O 重導向順序與管線（Pipelines）
6. **如何用 UNIX 小工具處理文字？** — `grep`、`cut`/`awk`、`sort`/`uniq`/`wc`、`tr`/`sed` 與安全 `xargs`

---

<!-- _class: divider -->

# 單元一：認識 Shell 與指令執行

當你在黑底白字的視窗輸入指令並按下 Enter，電腦背後究竟發生了什麼事？

---

## Terminal、Shell 與 Kernel 的分工

許多初學者會把終端機視窗與 Shell 混為一談，其實系統分成三個明確層級：

<pre class="mermaid">
flowchart LR
    T["Terminal (顯示文字與傳送按鍵)"] --> S["Shell (解析指令並啟動程式)"]
    S --> K["Kernel (管理行程、記憶體與裝置)"]
</pre>

<div class="cols">
<div>

- **Terminal（終端機）**：負責顯示文字與傳送按鍵的使用者介面（`03_Shell.pdf, p.4`）
- **Shell**：執行在 Terminal 內部的程式，負責解析語法、展開變數與樣式，並啟動內建或外部程式（`p.4`）
- **Kernel（核心）**：真正管理行程、記憶體與裝置，並提供系統呼叫（`p.4`）

</div>
<div>

```sh
$ pwd
/home/tsaimh
$ whoami
tsaimh
$ date
Sun Sep  7 20:00:00 CST 2026
```

> 先從這三個最基礎的指令感受與系統互動：看目前位置、看身分、看時間（`p.4`）。

</div>
</div>

---

## 按下 Enter 後的 5 個步驟

<pre class="mermaid">
flowchart LR
    S1["1. Read & parse (引號、運算子、重導向)"] --> S2["2. Expand (變數、$()、萬用字元)"]
    S2 --> S3["3. Resolve (built-in、function、PATH)"]
    S3 --> S4["4. Execute (built-in 或 fork/exec)"]
    S4 --> S5["5. Wait / manage job (前景或背景)"]
</pre>

以這行指令為例：

```sh
$ ls "$HOME"/*.txt | grep report > matches.txt 2> errors.txt
```

- 在 `ls` 真正執行之前，Shell 就已經先替它把 `"$HOME"` 與 `*.txt` 展開成實際檔案清單，並設定好管線與檔案重導向（`03_Shell.pdf, p.5`）。

> **核心觀念**：Shell 會在多數外部程式看到參數**之前**，先處理完語法與展開（`p.5`）。

---

## 常見 Shell 與 Debian 預設環境

不同平台與用途會有不同 Shell，腳本必須明確宣告自己針對哪一種語法撰寫（`p.6`）：

| Shell | 家族 / 角色 | 典型路徑 | 初學者要記住的定位 |
| :--- | :--- | :--- | :--- |
| `sh` | POSIX 風格 Bourne shell | `/bin/sh` | 撰寫可攜腳本的基準（`03_Shell.pdf, p.6`） |
| `bash` | Bourne Again Shell | `/bin/bash` | Linux 常見互動 Shell；**Debian 一般新使用者預設**（`p.6-7`） |
| `zsh` | Bourne 風格互動 Shell | `/bin/zsh` | macOS 預設登入 Shell（`p.6`） |
| `tcsh` | C shell 家族 | `/bin/tcsh` | 常見於 BSD 與舊式環境（`p.6`） |
| PowerShell | 物件導向自動化 Shell | `pwsh` / `powershell.exe` | 跨 Windows、Linux 與 macOS（`p.6`） |

- 在 **Debian** 上，`/bin/sh` 是獨立的 POSIX 風格 Shell 介面；想確認自己帳號的設定別靠猜，請執行 `getent passwd "$USER"`（`echo "$SHELL"` 僅作輔助提示）（`p.7`）。

---

<!-- _class: divider -->

# 單元二：執行模式與 Bash 啟動設定檔

為什麼有時把環境設定寫進設定檔，開新終端機有效，交給排程卻失效？

---

## 三種執行模式與 Bash 讀檔順序

Shell 會依據「怎麼被啟動」來決定要讀取哪些設定檔（`03_Shell.pdf, p.9-10`）：

| 執行模式 | 什麼時候會遇到？ | Bash 會讀取什麼檔案？ |
| :--- | :--- | :--- |
| **Interactive login** | 主控台登入、部分設定下的 SSH 連線 | `/etc/profile`，再讀 `~/.bash_profile`、`~/.bash_login`、`~/.profile` 中**第一個可讀檔**（`p.9-10`） |
| **Interactive non-login** | 開啟新終端機分頁、在 Shell 中再啟 `bash` | `~/.bashrc`（`p.9-10`） |
| **Non-interactive** | 執行腳本（如 `./backup.sh`） | **通常跳過互動 rc 檔**；若有設 `$BASH_ENV` 則讀該檔（`p.9-10`） |
| **Login shell exit** | 結束登入工作階段 | `~/.bash_logout`（`p.10`） |

> **實務常見做法**：`~/.bash_profile` 常在裡面 `source ~/.bashrc`，讓登入與非登入互動工作階段都能共用互動設定（`03_Shell.pdf, p.10`）。

---

## 初學常見盲點與安全編輯習慣

```sh
# ~/.profile 或 ~/.bashrc 常見設定範例
export EDITOR=vim
export PATH="$HOME/bin:$PATH"
alias ll="ls -lah"
```

<div class="cols">
<div>

### 安全編輯四原則（`p.12`）
1. 複製或 `source` 他人設定前**先看懂內容**
2. 重大修改前**先備份原檔**
3. 用**新開 Shell** 或明確 `source` 驗證變更
4. 將**全系統**與**個人**設定分開維護

</div>
<div>

<blockquote class="trap">
<strong>盲點 1（自動化失效）</strong>：非互動腳本通常跳過 <code>~/.bashrc</code>。把腳本必備變數只放 <code>~/.bashrc</code>，會讓腳本在 <code>cron</code>、SSH 自動化或 <code>systemd</code> 下跑出不同結果（<code>p.9</code>）。
</blockquote>

<blockquote class="trap">
<strong>盲點 2（勿直接 <code>curl | sh</code>）</strong>：管線直接執行遠端內容會跳過檢視；應先 <strong>Download → Inspect → Verify → Execute</strong>（<code>p.12</code>）。
</blockquote>

</div>
</div>

---

<!-- _class: divider -->

# 單元三：變數、引號、展開與狀態碼

看懂為什麼「空白」和「引號」是寫對 Shell 指令最重要的基本功。

---

## Shell 變數 vs 環境變數：子程式看得到嗎？

<div class="cols">
<div>

### Shell 變數（局部於當前 Shell）
```sh
name=alice
echo "$name"
```
- 只有**目前這個 Shell** 看得到（`03_Shell.pdf, p.14`）
- **不會**自動傳給由它啟動的子程式（`p.14`）
- 在許多 Shell 中可用 `set` 查看（`p.14`）

</div>
<div>

### 環境變數（會被子行程繼承）
```sh
export EDITOR=vim
printenv EDITOR
```
- 加上 `export` 後，啟動的**子程式都會繼承**（`03_Shell.pdf, p.14`）
- 可用 `env` 或 `printenv` 查看（`p.14`）
- **`sh`/`bash` 賦值 `=` 兩側絕不可加空白**（`p.16`）

</div>
</div>

> **常用環境變數（`p.15`）**：`HOME`（家目錄）、`USER`/`LOGNAME`、`PATH`（冒號分隔的指令搜尋路徑）、`PWD`（目前目錄）、`EDITOR`/`VISUAL`、`LANG`/`LC_*`；特別注意 **`SHELL` 只代表設定好的登入 Shell，不一定是目前正在跑的 Shell**。

---

## 引號怎麼選？單引號、雙引號與不加引號

在變數外加上雙引號 `"$var"` 是最安全的預設習慣（`03_Shell.pdf, p.17`）：

| 寫法 | Shell 會不會展開變數與指令？ | 會不會依空白切詞或展開檔名？ | 範例 |
| :--- | :--- | :--- | :--- |
| `'text'`（單引號） | **不會**（完全保留字面文字） | 不會 | `echo '$HOME'` |
| `"text"`（雙引號） | **會**（展開參數與指令替換） | **不會**（安全保護空白） | `echo "$HOME"` |
| `\`（反斜線） | 跳脫緊鄰的下一個字元，或把長指令延續到下一行（`p.17, 21`） | 保護該字元 | `touch test\*` |
| unquoted（不加引號） | **會** | **會！（可能出錯，務必小心）** | `printf "%s\n" $files` |

```sh
# 用反斜線將長指令拆成多行更好讀（p.21）
printf "%s %s\n" \
  "hello" "world"
```

---

## 指令替換 `$(...)` 與檔名展開（Globbing）

<div class="cols">
<div>

### 1. 指令替換 `$(command)`（`p.18`）
把指令的標準輸出直接填入該位置，且會**自動去掉結尾換行**：
```sh
now=$(date)
count=$(wc -l < /etc/passwd)
archive="$(basename "$(pwd)").tar.gz"
```
- 新寫法優先用 **`$(...)`**，因為**巢狀書寫**比反引號清晰得多（`p.16, 18`）。

</div>
<div>

### 2. Globbing vs 正規表達式（`p.19`）
- `*` 任意字串（含空字串）、`?` 單一字元、`[abc]` 字元集合、`[!abc]` 排除集合
- **Globbing ≠ Regex**：想交給程式（如 `grep`）解讀樣式時**必須加引號**

> **靜態檢查好幫手 `ShellCheck`（`p.44`）**：commit 腳本前執行可抓出如 `SC2086`（未加雙引號）等風險，但警告是用來幫助理解語意，**不能取代實際測試**。

</div>
</div>

---

## 結束狀態碼 `$?` 與條件串接 `;` / `&&` / `||`

每個指令結束後都會回傳一個數字給 `$?`，讓後續指令決定要不要執行（`03_Shell.pdf, p.20`）：

- **通用約定**：**`0` 代表成功**，**非 `0` 代表失敗或特殊條件**（`true` 回傳 `0`、`false` 回傳 `1`）。

| 運算子 | 執行邏輯 | 範例 | 初學者理解 |
| :--- | :--- | :--- | :--- |
| `;` | **不管成功失敗**，一律接著跑下一個 | `cd /tmp; pwd` | 單純把兩個指令寫在同一行（`p.20`） |
| `&&` | 只有**前一個成功（`0`）**才跑下一個 | `make && make test` | 編譯成功了才執行測試（`p.20`） |
| `||` | 只有**前一個失敗（非 `0`）**才跑下一個 | `mkdir out || exit 1` | 建立資料夾失敗就立刻中止（`p.20`） |

---

<!-- _class: divider -->

# 單元四：內建指令、工作控制與日常效率

為什麼 `cd` 不能是一個獨立程式？如何在單一終端機裡暫停或切換工作？

---

## 內建指令（Built-ins）vs 外部指令（External）

當外部程式啟動時，它會跑在一個獨立的子行程裡——**子行程無法回頭改變父 Shell 的目前目錄或環境變數**，因此改變 Shell 本身狀態的功能必須內建在 Shell 裡（`03_Shell.pdf, p.22-23`）。

<div class="cols">
<div>

### Built-ins（內建指令）
- 範例：`cd`、`export`、`alias`、`jobs`、`fg`、`bg`、`exit`
- 直接在 **Shell 行程內部**執行
- **可以修改 Shell 本身狀態**（`p.23`）

</div>
<div>

### External commands（外部指令）
- 範例：`ls`、`grep`、`awk`、`sort`、`ssh`、`python`
- 透過 `PATH` 或明確路徑找到並跑成**獨立行程**（`p.23`）

</div>
</div>

```sh
$ type cd
cd is a shell builtin
$ command -V grep
grep is /usr/bin/grep
```

---

## 工作控制（Job Control）實境演練

<pre class="mermaid">
flowchart LR
    FG["foreground (前景執行)"] -->|"^Z (Ctrl-Z 送出 stop signal)"| SUS["Suspend (暫停中)"]
    SUS -->|"fg"| FG
    SUS -->|"bg"| BG["background (背景執行)"]
    BG -->|"stop"| SUS
</pre>

```text
$ sleep 300               # 在前景執行；若要一開始就放背景，改寫 sleep 300 &
^Z                        # 按 Ctrl-Z 送出 stop signal 暫停目前工作
[1]+  Stopped                 sleep 300
$ bg %1                   # 讓工作 %1 在背景繼續跑
$ jobs                    # 查看目前工作清單（會顯示 Running）
$ fg %1                   # 帶回前景；或用 kill %1 / kill PID 終止（p.24）
```

<blockquote class="trap">
<strong>重要提醒</strong>：用 <code>&amp;</code> 放到背景<strong>不等於脫離終端機</strong>（終端訊號與輸出依然會影響它）。需要長期穩定運作的系統服務，請交給 <code>systemd</code> 或專用行程監控工具，不要靠 <code>&amp;</code>（<code>03_Shell.pdf, p.21</code>）。
</blockquote>

---

## 別名、目錄堆疊與歷史紀錄的安全用法

<div class="cols">
<div>

### 互動便利工具
- **別名（Aliases，`p.25`）**：幫互動輸入省字（如 `alias ll="ls -lah"`）；當邏輯變複雜時請改寫成函式（functions）
- **目錄堆疊（`p.25`）**：在 `bash`/`zsh` 可用 `pushd /var/log` 記住路徑並切換、`dirs` 查看堆疊、`popd` 彈回前一個目錄
- **歷史紀錄（`p.47`）**：`history`、`Up arrow`、`Ctrl-R` 反向搜尋、以及 `!n` / `!!` / `!text`

</div>
<div>

<blockquote class="trap">
<strong>初學必防的三大風險</strong><br>
1. <strong>別名會隱藏真正執行的指令</strong>：管理系統前先確認別名展開後是什麼（<code>p.25</code>）。<br>
2. <strong><code>!</code> 歷史展開容易誤觸</strong>：在具破壞性或特權操作前務必檢查實際將執行的指令（<code>p.47</code>）。<br>
3. <strong>寫腳本不可依賴別名或歷史快捷鍵</strong>：腳本中應使用明確的指令與變數（<code>p.25, 48</code>）。
</blockquote>

</div>
</div>

---

<!-- _class: divider -->

# 單元五：標準串流、I/O 重導向與管線

如何把一個程式的輸出存成檔案，或直接接成下一個程式的輸入？

---

## 標準串流（FD `0/1/2`）與 I/O 重導向

每個執行中的程式都有三個預設開啟的資料通道（檔案描述子，FD）（`03_Shell.pdf, p.27`）：

<div class="cols">
<div>

<pre class="mermaid">
flowchart LR
    IN["stdin (0)"] --> CMD["command"]
    CMD --> OUT["stdout (1)"]
    CMD --> ERR["stderr (2)"]
</pre>

- `0 stdin`：標準輸入（預設鍵盤）
- `1 stdout`：標準輸出（預設終端機）
- `2 stderr`：標準錯誤（預設終端機）

</div>
<div>

| 語法 | 效果（`p.28`） |
| :--- | :--- |
| `cmd < file` | 用 `file` 當作 `stdin` |
| `cmd > file` | 寫入 `stdout`（**先清空舊檔**） |
| `cmd >> file` | 附加 `stdout` 到檔尾 |
| `cmd 2> file` | 將 `stderr` 寫入 `file` |
| `cmd >out 2>&1` | 把 `stdout` 與 `stderr` 寫進同一檔 |

</div>
</div>

> **一定要搞懂的左右順序**：`2>&1` 的意思是「讓 fd 2 指向 fd 1 **現在**所指的位置」。寫 `make > build.log 2>&1` 時，先讓 fd 1 指向 `build.log`，接著 fd 2 才會一起指向 `build.log`（`03_Shell.pdf, p.28`）。

---

## 管線（Pipelines）、`tee` 與合併錯誤串流

<pre class="mermaid">
flowchart LR
    P["producer: journalctl"] --> F["filter: grep sshd"]
    F --> T["transform: awk '{print $5}'"]
    T --> C["consumer: sort -u"]
</pre>

- **管線 `|`**：把左側程式的 **`stdout`** 接到右側程式的 **`stdin`**；**`stderr` 預設不會被 `|` 傳過去**（`03_Shell.pdf, p.29`）。
- **`tee` 指令**：想同時「在螢幕看輸出」又「存進檔案」時使用（`p.30`）。

```sh
command >output.log 2>error.log     # 錯誤需分開處理時，分開存檔
command 2>&1 | tee combined.log     # 先把 stderr 併進 stdout 再進管線（POSIX 可攜寫法）
command |& tee combined.log         # Bash/zsh 便利簡寫（注意：不是 POSIX sh 語法！）
```

---

<!-- _class: divider -->

# 單元六：UNIX 哲學與文字處理工具

不必每次遇到小需求就從頭寫程式——把專精單一任務的小工具串起來。

---

## 基礎檔案指令與 UNIX 小工具哲學

<div class="cols">
<div>

### 基礎檔案與目錄指令（`p.32`）
- 瀏覽與切換：`ls`、`pwd`、`cd`
- 建立與搬移：`mkdir`、`cp`、`mv`、`ln`
- 刪除與檢查：`rmdir`（空目錄）、`rm`、`stat`（看中繼資料）

<blockquote class="trap">
<strong>安全警示</strong>：<code>rm</code> 沒有資源回收筒；執行遞迴或高權限刪除前，務必再次檢查路徑、引號與萬用字元（<code>p.32</code>）。
</blockquote>

</div>
<div>

### UNIX 哲學三要點（`p.43`）
**Small tools. Clear interfaces. Composition.**
1. **Do one thing well**：每個工具做好一件可預期的小事
2. **Text as glue**：用文字串流與管線把工具接起來
3. **Automate repetition**：把成功驗證的序列寫成腳本

> **現代視角**：能提升清晰度時善用管線；但若適合結構化 API、資料庫或正規程式語言，別硬套純文字管線（`p.43`）。

</div>
</div>

---

## 文字處理入門（上）：選型、`head`/`tail`/`grep` 與 `cut`/`awk`

先看清需求再挑工具（`03_Shell.pdf, p.33-35`）：

- **看頭尾 / 追日誌**：`head -n 20 file.txt`、`tail -n 50 file.txt`、`tail -f /var/log/messages`（提醒：在 `systemd` 系統如 Debian，許多服務改用 `journalctl` 讀取日誌，`p.34`）。
- **依樣式過濾整行**：`grep -n "sshd"`（印行號）、`grep -l "ERROR" *.log`（僅列檔名）；純固定字串用 **`-F`**，延伸正規用 **`-E`**，樣式務必加引號（`p.34`）。
- **挑出特定欄位**：

| 工具 | 什麼時候用？ | 限制與提醒（`p.35`） |
| :--- | :--- | :--- |
| `cut` | 固定分隔符號或字元範圍（如 `cut -d: -f1,3,6,7`） | **無法做條件篩選或數值計算** |
| `awk` | 依欄位做條件過濾、排版與計算（如 `awk -F: '$7 !~ /nologin$/ {print $1, $7}'`） | 需學習較多語法 |

> **腳本鐵則**：不要在腳本中解析 `ls` 輸出——`ls` 是設計給人類看的，格式容易造成歧義（`03_Shell.pdf, p.35`）。

---

## 文字處理入門（中）：`sort`/`uniq`/`wc` 與 `tr`/`sed`

### 1. 排序、去重與計數（`p.36`）

<pre class="mermaid">
flowchart LR
    RAW["原始 IP: 140.113.6.2 / 140.113.1.1 / ..."] -->|"sort"| S1["相同項相鄰"]
    S1 -->|"uniq -c"| U1["統計重複次數"]
    U1 -->|"sort -nr"| S2["依數值由大到小"]
</pre>

- `wc -l` / `-w` / `-c` 分別計算行數、字數、位元組數；`sort` 常用 `-n`（數值）、`-r`（反序）、`-u`（唯一）、`-k`（欄位）、`-t`（分隔符）。
- **初學雙陷阱**：(1) `uniq` **只會合併相鄰的重複行**，必須先 `sort`；(2) 沒有 `-n` 時是依文字比較，會導致 `"14"` 排在 `"123"` 前面（`p.36`）。

### 2. 字元轉換 `tr` vs 串流編輯 `sed`（`p.37`）
- **`tr`** 是**逐字元**轉換／刪除／壓縮（例：`tr '[:lower:]' '[:upper:]'`、`tr -d '\t'`、`tr -s ' '`），不懂單字也不懂正規群組。
- **`sed`** 是**逐行**串流編輯器，支援正規替換（例：`sed 's/old/new/' file.txt` 替換每行首次出現的目標）。

---

## 文字處理入門（下）：`xargs` 與安全處理檔名

有些程式只能透過**命令列參數**接收輸入，而不會讀 `stdin`，這時就需要 `xargs` 幫忙轉換（`03_Shell.pdf, p.38`）：

<pre class="mermaid">
flowchart LR
    IN["stdin"] --> X["xargs"]
    X --> OUT["utility arg1 arg2 ..."]
</pre>

- **常用 POSIX 選項（`p.39`）**：`-n N`（每次最多 N 個參數）、`-I token`（單行替換占位符，如 `ls | xargs -I % echo "[%]"`）、`-s size`（限制命令列大小）。

<blockquote class="trap">
<strong>安全處理含空白與換行的檔名（<code>p.40</code>）</strong><br>
一般 <code>xargs</code> 依空白與換行切分，但合法檔名可包含空白、Tab 與換行！<br>
• 不安全：<code>find . -type f | xargs grep -l TODO</code><br>
• 安全（NUL 分隔）：<code>find . -type f -print0 | xargs -0 grep -l TODO</code><br>
• 或直接用 <code>find</code> 內建機制免除 <code>xargs</code>：<code>find ... -exec command {} +</code>
</blockquote>

---

## 綜合實戰與初學者總結

### 分階段拆解管線（Think in stages，`p.41-42`）
想從 `access.log`（第 1 欄是 IP）找出前 10 名常客？拆成五個小步驟串起來：

```sh
awk '{print $1}' access.log \
  | sort \
  | uniq -c \
  | sort -nr \
  | head -n 10
```
1. `awk` 抽第 1 欄 → 2. `sort` 讓相同 IP 靠在一起 → 3. `uniq -c` 算次數 → 4. `sort -nr` 數值降冪排序 → 5. `head -n 10` 取前 10 名。

> **帶走這三個好習慣（`p.45, 49`）**：展開變數一律加雙引號 `"$var"`、漸進式測試每段管線、在加上 `sudo` 前先檢查完整命令；忘記參數時優先查閱本機 `man` 手冊。

[閱讀完整 Wiki 講義](../shell/)

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
