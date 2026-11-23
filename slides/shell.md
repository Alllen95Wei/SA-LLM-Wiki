---
marp: true
theme: sa-wiki
paginate: true
size: 16:9
header: "System Administration (Debian 13) — 考前複習簡報"
footer: "來源：03_Shell.pdf ｜ System Administration LLM Wiki"
---

<!-- _class: lead -->
<!-- _paginate: skip -->

# Shell：命令列、管線與文字處理

Debian 13 (trixie) 考前複習重點整理

- **課程核心強調**：先學可攜觀念（portable ideas first），特定 Shell 的便利語法擺在第二（`03_Shell.pdf, p.2`）
- **可攜腳本基準**：撰寫腳本優先以 `sh` 為目標（除非作業指定其他 Shell）（`03_Shell.pdf, p.11`）
- [返回完整文字講義](../shell/)

---

## 1. Terminal、Shell 與 Kernel 三層職責

<pre class="mermaid">
flowchart LR
    T["Terminal (顯示文字與傳送按鍵)"] --> S["Shell (解析指令並啟動程式)"]
    S --> K["Kernel (管理行程、記憶體與裝置)"]
</pre>

- **Terminal**：顯示文字並傳送按鍵輸入的**使用者介面**（`03_Shell.pdf, p.4`）
- **Shell**：執行在 Terminal 內部的程式；解析語法、展開變數與樣式，執行 built-ins 或外部程式（`03_Shell.pdf, p.4`）
- **Kernel**：管理行程、記憶體與裝置，並提供系統呼叫（`03_Shell.pdf, p.4`）

> **考點**：Terminal 只是介面，Shell 才是解譯程式，Kernel 負責底層資源與 system calls（`03_Shell.pdf, p.4`）。

---

## 2. 按下 Enter 後的 5 階段執行流程

<pre class="mermaid">
flowchart LR
    S1["1. Read & parse (引號、運算子、重導向)"] --> S2["2. Expand (變數、$()、萬用字元)"]
    S2 --> S3["3. Resolve (built-in、function、PATH)"]
    S3 --> S4["4. Execute (built-in 或 fork/exec)"]
    S4 --> S5["5. Wait / manage job (前景或背景)"]
</pre>

```sh
$ ls "$HOME"/*.txt | grep report > matches.txt 2> errors.txt
```

> **關鍵觀念（Key idea）**：Shell 會在多數外部程式看到它們的參數**之前**，就先完成引號、變數、萬用字元展開與重導向處理（`03_Shell.pdf, p.5`）。

---

## 3. 常見 Shell 與 Debian 預設設定

| Shell | 家族 / 角色 | 典型路徑 | 重點定位 |
| :--- | :--- | :--- | :--- |
| `sh` | POSIX 風格 Bourne shell | `/bin/sh` | 腳本的可攜性基準（`03_Shell.pdf, p.6`） |
| `bash` | Bourne Again Shell | `/bin/bash` | Linux 上常見的互動式 Shell（`03_Shell.pdf, p.6`） |
| `zsh` | Bourne 風格互動式 Shell | `/bin/zsh` | macOS 預設登入 Shell（`03_Shell.pdf, p.6`） |
| `tcsh` | C shell 家族 | `/bin/tcsh` | 常見於 BSD 與舊式環境（`03_Shell.pdf, p.6`） |
| PowerShell | 物件導向自動化 Shell | `pwsh` / `powershell.exe` | Windows、Linux、macOS（`03_Shell.pdf, p.6`） |

- **Debian 預設**：新增一般使用者預設用 `bash`；`/bin/sh` 是獨立的 POSIX 風格 Shell 介面（`03_Shell.pdf, p.7`）
- **查驗而非猜測**：`getent passwd "$USER"`（查帳號設定），`echo "$SHELL"` 僅作提示（`03_Shell.pdf, p.7`）

---

## 4. 三種 Shell 模式與 Bash 啟動設定檔

| 模式 | 典型情境 | Bash 讀取檔案順序 |
| :--- | :--- | :--- |
| **Interactive login** | 主控台登入、部分 `ssh` 工作階段 | 先 `/etc/profile`，再讀 `~/.bash_profile`、`~/.bash_login`、`~/.profile` 中**第一個可讀檔**（`03_Shell.pdf, p.9-10`） |
| **Interactive non-login** | 新終端機分頁、在 Shell 內啟 `bash` | `~/.bashrc`（`03_Shell.pdf, p.9-10`） |
| **Non-interactive** | 執行腳本 `./backup.sh` | **通常跳過互動 rc 檔**；若有設 `$BASH_ENV` 則讀該檔（`03_Shell.pdf, p.9-10`） |
| **Login shell exit** | 登出離開 | `~/.bash_logout`（`03_Shell.pdf, p.10`） |

<blockquote class="trap">
<strong>常見陷阱</strong>：把腳本必要變數只放在 <code>~/.bashrc</code>，會讓腳本在 <code>cron</code>、SSH 自動化或 <code>systemd</code> 下執行行為不同（<code>03_Shell.pdf, p.9</code>）。實務上常由 <code>~/.bash_profile</code> 主動 <code>source ~/.bashrc</code>（<code>p.10</code>）。
</blockquote>

---

## 5. 安全編輯啟動檔與 `curl | sh` 風險

```sh
# ~/.profile or ~/.bashrc
export EDITOR=vim
export PATH="$HOME/bin:$PATH"
alias ll="ls -lah"
```

<div class="cols">
<div>

### 安全編輯四原則（`p.12`）
1. 複製或 `source` 前先**閱讀內容**
2. 重大修改前**先備份**
3. 用新 Shell 或明確 `source` 套用變更
4. **區隔**系統全域與個人設定

</div>
<div>

<blockquote class="trap">
<strong>安全警示：Safer than <code>curl | sh</code></strong><br>
將遠端內容直接以管線送進 Shell 執行會<strong>繞過人工檢視步驟</strong>。<br>
正確做法：<strong>Download → Inspect → Verify → Execute</strong>（<code>03_Shell.pdf, p.12</code>）。
</blockquote>

</div>
</div>

---

## 6. Shell 變數 vs 環境變數

<div class="cols">
<div>

### Shell variable（Shell 變數）
```sh
name=alice
echo "$name"
```
- 僅在**當前 Shell** 可見（`p.14`）
- **不會**自動被子行程繼承（`p.14`）
- 許多 Shell 可用 `set` 檢視（`p.14`）

</div>
<div>

### Environment variable（環境變數）
```sh
export EDITOR=vim
printenv EDITOR
```
- 經 `export` 後**會被子行程繼承**（`p.14`）
- 用 `env` / `printenv` 檢視（`p.14`）
- `sh`/`bash` 簡單賦值 **`=` 兩側不可有空白**（`p.16`）

</div>
</div>

> **考點陷阱**：`$SHELL` 代表**設定好的登入 Shell**，不一定等於當前正在運行的 Shell（`03_Shell.pdf, p.15`）。其他常用環境變數：`HOME`、`USER`/`LOGNAME`、`PATH`（冒號分隔）、`PWD`、`EDITOR`/`VISUAL`、`LANG`/`LC_*`（`p.15`）。

---

## 7. 引號規則、指令替換與 Globbing vs Regex

| 語法 | 行為 | 範例 |
| :--- | :--- | :--- |
| `'text'` | 字面保留；**不展開**參數與指令（`p.17`） | `echo '$HOME'` |
| `"text"` | 允許參數與指令替換，避免拆詞與檔名展開（`p.17`） | `echo "$HOME"` |
| `\` | 跳脫單一字元（`touch test\*`）或跨行續寫（`p.17, 21`） | `echo \*` |
| unquoted | 可能觸發**欄位分割**與**路徑展開**（謹慎！）（`p.17`） | `printf "%s\n" $files` |

- **指令替換 `$(command)`**：取代為 `stdout` 並**移除尾端換行**；易於巢狀（`"$(basename "$(pwd)").tar.gz"`），優先於反引號（`03_Shell.pdf, p.16, 18`）
- **Globbing ≠ Regex**：`*`、`?`、`[abc]`、`[!abc]` 比正規表達式簡單；要把樣式交給程式解析時**須加引號**（`p.19`）
- **ShellCheck**：commit 前檢查未加引號展開（如 `SC2086`），但不可取代實際測試（`p.44`）

---

## 8. 結束狀態碼、控制運算子與內建指令

<div class="cols">
<div>

### Exit Status `$?` 與運算子（`p.20`）
- **`0` = 成功**；**非 `0` = 失敗或特殊條件**（`true` → `0`，`false` → `1`）
- `;`：無論前序結果都執行（`cd /tmp; pwd`）
- `&&`：前者成功才執行（`make && make test`）
- `||`：前者失敗才執行（`mkdir out || exit 1`）

</div>
<div>

### Built-ins vs External（`p.23`）
- **Built-ins**（`cd`、`export`、`alias`、`jobs`、`fg`、`bg`、`exit`）：在 Shell 內執行，**可改變 Shell 狀態**
- **External**（`ls`、`grep`、`awk`、`sort`、`ssh`、`python`）：由 `PATH` 尋找並以**獨立行程**執行
- 驗證指令：`type cd`、`command -V grep`

</div>
</div>

---

## 9. 工作控制（Job Control）、別名與歷史紀錄

<pre class="mermaid">
flowchart LR
    FG["foreground (前景執行)"] -->|"^Z (Ctrl-Z 送出 stop signal)"| SUS["Suspend (暫停中)"]
    SUS -->|"fg"| FG
    SUS -->|"bg"| BG["background (背景執行)"]
    BG -->|"stop"| SUS
</pre>

- **Job Control**：`command &`（背景啟動）、`Ctrl-Z`（暫停）、`jobs`（列表）、`bg %1`、`fg %1`、`kill %1` / `kill PID`（`03_Shell.pdf, p.24`）
- **Aliases 與目錄堆疊**：`alias` 隱藏破壞性指令需小心、**勿在腳本依賴**，複雜請改用函式；`pushd` / `popd` / `dirs` 維護目錄堆疊（`p.25`）

<blockquote class="trap">
<strong>常見陷阱</strong>：(1) <code>&amp;</code> <strong>不等於脫離終端機</strong>（終端訊號與輸出仍影響），長駐服務應交由 <code>systemd</code> 或 supervisor（<code>p.21</code>）；(2) 歷史展開 <code>!n</code> / <code>!!</code> / <code>!text</code> 執行前務必檢查，腳本請寫明確指令（<code>p.47-48</code>）。
</blockquote>

---

## 10. 標準串流（FD `0/1/2`）與重導向順序考點

<div class="cols">
<div>

<pre class="mermaid">
flowchart LR
    IN["stdin (0)"] --> CMD["command"]
    CMD --> OUT["stdout (1)"]
    CMD --> ERR["stderr (2)"]
</pre>

- `0 stdin`：預設鍵盤 / 呼叫端輸入（`p.27`）
- `1 stdout`：預設終端 / 呼叫端輸出（`p.27`）
- `2 stderr`：預設終端 / 呼叫端錯誤串流（`p.27`）

</div>
<div>

| 語法 | 效果（`p.28`） |
| :--- | :--- |
| `cmd < file` | 以 `file` 作為 `stdin` |
| `cmd > file` | 寫入 `stdout`（**先截斷清空**） |
| `cmd >> file` | 附加 `stdout` 至檔尾 |
| `cmd 2> file` | 將 `stderr` 寫入 `file` |
| `cmd >out 2>&1` | `stdout` 與 `stderr` 同送 `out` |

</div>
</div>

> **高頻考點（Order matters）**：`2>&1` 意為「讓 fd 2 指向 fd 1 **目前**所指的位置」。必須先 `> build.log` 再 `2>&1`，由左至右順序錯了就會改變結果（`03_Shell.pdf, p.28`）。

---

## 11. 管線（Pipelines）、`tee` 與串流合併

<pre class="mermaid">
flowchart LR
    P["producer: journalctl"] --> F["filter: grep sshd"]
    F --> T["transform: awk '{print $5}'"]
    T --> C["consumer: sort -u"]
</pre>

- **管線 `|` 行為**：將左側 `stdout` 接到右側 `stdin`；**預設不傳遞 `stderr`**（`03_Shell.pdf, p.29`）
- **`tee`**：同時將串流複製到 `stdout` 與指定檔案（`p.30`）

```sh
command >output.log 2>error.log     # 錯誤需分開處理時保持分離
command 2>&1 | tee combined.log     # 先合併再進管線（POSIX 廣泛可攜）
command |& tee combined.log         # Bash / zsh 簡寫（非 POSIX sh 語法！）
```

> **可攜性考點**：`2>&1` 具廣泛可攜性；`|&` 僅為 `bash`/`zsh` 捷徑，**不是 POSIX `sh` 語法**（`03_Shell.pdf, p.30`）。

---

## 12. UNIX 哲學與文字處理選型心智圖

**UNIX 哲學（`p.43`）**：*Small tools. Clear interfaces. Composition.*（Do one thing well / Text as glue / Automate repetition；若結構化 API 或資料庫更安全，勿硬套純文字管線）。

| 需求 | 選用工具 | 基礎檔案操作指令（`p.32`） |
| :--- | :--- | :--- |
| 取前／後幾行 | `head` / `tail`（`p.33`） | `ls`, `pwd`, `cd`, `mkdir`, `rmdir` |
| 樣式篩選行 | `grep`（`p.33`） | `cp`, `mv`, `rm`, `ln`, `stat` |
| 抽取欄位／直欄 | `cut` / `awk`（`p.33`） | <blockquote class="trap" style="margin:0;"><strong>警告</strong>：<code>rm</code> 無資源回收筒；執行遞迴或高權限刪除前務必再次檢查路徑、引號與萬用字元（<code>p.32</code>）。</blockquote> |
| 排序／群組／計數 | `sort` / `uniq` / `wc`（`p.33`） | |
| 字元轉換／串流編輯 | `tr` / `sed`（`p.33`） | |
| 輸入轉命令列參數 | `xargs`（`p.33`） | |

---

## 13. 文字處理四大高頻考點

1. **`head` / `tail` / `grep`（`p.34`）**：`tail -f` 追蹤日誌（`systemd` 系統常用 `journalctl` 取代）；`grep -F` 固定字串、`-E` 延伸正規、`-n` 行號、`-l` 檔名，**樣式須加引號**。
2. **`cut` vs `awk`（`p.35`）**：`cut` 僅處理固定分隔或字元範圍（無條件/運算）；`awk` 支援條件過濾與運算。**腳本避免解析 `ls` 輸出**（給人看且具歧義）。
3. **`sort` / `uniq` / `wc`（`p.36`）**：

<pre class="mermaid">
flowchart LR
    RAW["原始 IP: 140.113.6.2 / 140.113.1.1 / ..."] -->|"sort"| S1["相鄰集中"]
    S1 -->|"uniq -c"| U1["統計次數"]
    U1 -->|"sort -nr"| S2["依數值降冪排列"]
</pre>

<blockquote class="trap">
<strong>雙陷阱</strong>：(1) <code>uniq</code> 只偵測<strong>相鄰</strong>重複行，必須先 <code>sort</code>；(2) 未加 <code>sort -n</code> 會做字串比對而非數值比對，導致 <code>"14"</code> 排在 <code>"123"</code> 之前（<code>p.36</code>）。
</blockquote>

4. **`tr` vs `sed`（`p.37`）**：`tr` **逐字元**處理（不懂單字與正規群組，可攜建議用 `[:lower:]`）；`sed` **逐行**支援正規替換（如 `sed 's/old/new/'`）。

---

## 14. `xargs` 安全檔名處理與整合管線實戰

<pre class="mermaid">
flowchart LR
    IN["stdin"] --> X["xargs"]
    X --> OUT["utility arg1 arg2 ..."]
</pre>

- **POSIX 選項（`p.39`）**：`-n N`（每次最多 N 個參數）、`-I token`（單行替換占位符）、`-s size`（限制命令列大小）
- **安全檔名處理（`p.40`）**：檔名可含空白、Tab 與換行，一般 `xargs` 會切錯！

```sh
find . -type f -print0 | xargs -0 grep -l TODO    # 強健：以 NUL (\0) 分隔
find ... -exec command {} +                       # 有時更簡潔，直接免用 xargs
```

- **綜合管線經典範式（`p.41-42`）**：各階段維持清晰輸入輸出契約：

```sh
awk '{print $1}' access.log | sort | uniq -c | sort -nr | head -n 10
```

---

## 15. 考前速查表（Shell Cheat Sheet）

| 任務 | 語法模式（`03_Shell.pdf, p.45`） |
| :--- | :--- |
| 變數與匯出 | `name=value ; export name=value` |
| 安全展開 | `"$var" ; "${var}"` |
| 指令替換 | `$(command)` |
| 條件序列 | `cmd1 && cmd2 ; cmd1 \|\| fallback` |
| I/O 重導向 | `> >> < 2> 2>&1` |
| 管線 | `producer \| filter \| consumer` |
| 工作控制 | `jobs  bg %1  fg %1` |
| 安全 `find` + `xargs` | `find ... -print0 \| xargs -0 ...` |

> **最有價值的習慣**：變數展開一律加雙引號、漸進式測試指令行為，並在加上 `sudo` 之前仔細檢查完整命令（`03_Shell.pdf, p.45`）。

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
