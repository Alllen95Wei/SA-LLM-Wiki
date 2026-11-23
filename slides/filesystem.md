---
marp: true
theme: sa-wiki
paginate: true
size: 16:9
header: "System Administration (Debian 13) — 教學型簡報"
footer: "知識來源：wiki/filesystem.md（10_FileSystem.pdf） ｜ System Administration LLM Wiki"
---

<!-- _class: lead -->
<!-- _paginate: skip -->

# 檔案系統入門教學：架構、Inode、連結與權限位元遮罩

從分層架構、Inode 與軟硬連結，到徹底算懂 `chmod` 與 `umask` 的每一個位元

- **學習目標**：看懂 Linux 檔案樹與掛載機制、區分 Inode 與 Hard / Symbolic Link，並能徒手用二進位 Bit Mask 推算任意 `chmod` 與 `umask` 權限（`10_FileSystem.pdf, p.1-44`）
- **知識範圍**：嚴格依據課程講義並聚焦 Debian Linux（略過 FreeBSD 專屬內容）（`10_FileSystem.pdf, p.1-44`）
- [閱讀完整 Wiki 講義](../filesystem/)

---

## 學習地圖：這份簡報會帶你學會什麼？

1. **檔案系統如何把不同磁碟整合成一棵樹？** — VFS 分層、目錄用途、`mount`、`/etc/fstab` 與 `umount` 排障（`p.5-18`）
2. **Linux 中有哪些檔案類型？** — 七種 `ls -l` 類型符號、Socket 與 Pipe / `mkfifo`（`p.19-24`）
3. **檔名背後是誰？Inode 與連結有何差異？** — 分割區四大區塊與 Hard / Symbolic Link 實驗（`p.25-31`）
4. **權限數字與 `umask` 位元遮罩怎麼算？（重點展開）** — 從 `4/2/1` Bit Mask、`4000/2000/1000` 特殊位元到 `full & (!umask)` 逐位演算與最低權限（`p.32-38`）
5. **現代檔案系統進階特性** — Journaling、CoW（預告 Btrfs）與擴充屬性（`p.42-44`）

---

<!-- _class: divider -->

# 單元一：檔案系統架構、路徑與掛載

理解應用程式如何透過核心存取磁碟，以及多顆分割區如何掛載成單一 `/` 目錄樹。

---

## 先看懂 `ls -l` 的八個欄位

日常檢視檔案資訊時，`$ ls -l` 的每一欄都對應檔案系統的重要屬性：（`p.3`）

```text
drwx--x--x  7  tsaimh  dcs  1024  Sep 22 17:25  public_html
```

| 欄位順序 | 顯示內容 | 意義說明（`10_FileSystem.pdf, p.3`） |
| :---: | :--- | :--- |
| 1 | `d` | File Type（檔案類型；例如 `d` 為目錄） |
| 2 | `rwx--x--x` | File Access Mode（檔案存取權限模式） |
| 3 | `7` | inodes（投影片於該表標註欄位名） |
| 4 / 5 | `tsaimh` / `dcs` | File User Owner（擁有者） / File Group Owner（群組擁有者） |
| 6 / 7 / 8 | `1024` / `Sep 22 17:25` / `public_html` | File Size / File Last Modify Time / File Name |

---

## 檔案系統如何工作？VFS 與系統呼叫分層圖

**Application ↔ Kernel ↔ Hardware**：程式呼叫 system calls，核心透過 **VFS** 與驅動程式存取儲存裝置：（`p.5, 8`）

<pre class="mermaid">
flowchart LR
    subgraph UM["User mode (使用者模式)"]
        P["User Application (例如 open / read / write)"]
    end
    SCI["System Call Interface (核心系統呼叫介面)"]
    subgraph KM["Kernel mode (核心模式: File system)"]
        VFS["Virtual file system (VFS)"]
        FS["ext2 / msdos / minix / proc"]
        BC["Buffer cache"]
        DD["Device drivers"]
        VFS --- FS --- BC --- DD
    end
    HW["儲存裝置 (Disks)"]
    P --> SCI --> VFS
    DD --> HW
</pre>

---

## 檔案系統的四大元件與五類物件

<div class="cols">
<div>

### 四大核心元件（`p.6`）
1. **Namespace**：命名並排列階層（如 directory）
2. **API**：巡覽與操作節點的 system calls（例如 `cp` 透過 `open → read/write loop → close` 完成複製，`p.7`）
3. **Security model**：保護、隱藏與分享（如 permissions）
4. **Implementation**：將邏輯模型繫結至實體磁碟

</div>
<div>

### 檔案系統內可找到的五類物件（`p.9`）
- Files and directories
- Hardware device files
- Processes information
- Interprocess communication（IPC）channel
- Shared memory（SHM）segments

> 全數可用共通介面操作：`open`、`read`、`write`、`close`、`seek`、`ioctl`、`fcntl`（`p.9`）

</div>
</div>

---

## 路徑規則（Pathname）與常見系統目錄

<div class="cols">
<div>

### 路徑種類與長度上限（`p.10`）
- **絕對路徑**：從 `/` 開始，如 `/net/dcs/.../haha.c`
- **相對路徑**：從目前目錄開始，如 `test/haha.c`
- **單一組件**：**`≦ 255` 字元**
- **單一完整絕對路徑**：**`≦ 1023` 字元**

</div>
<div>

### 關鍵標準目錄速覽（`hier(7)`, `p.12-13`）
- `/`：根目錄；`/bin` & `/sbin`：核心基礎工具
- `/etc`：系統組態檔與腳本；`/dev`：裝置項目
- `/proc`：運行中行程映像資訊
- `/mnt`：暫時掛載點；`/tmp`：暫存檔（重開機不保證留存）
- `/var`（含 `/var/log`、`/var/mail`、`/var/spool`）：日誌與變動資料

</div>
</div>

---

## 什麼是掛載（`mount`）？多磁碟掛載樹圖解

`mount(8)` 會把既有目錄樹的**掛載點（mount point）**對應至新檔案系統的根目錄：（`p.14-15`）

<pre class="mermaid">
flowchart TB
    D1["Disk 1 (Partition 0)"] -->|"掛載於 /"| ROOT["/"]
    ROOT --> SUB1["/bin, /etc, /usr, /var"]
    ROOT --> D2P["/d2 (Mount point)"]
    ROOT --> D3P["/d3 (Mount point)"]
    D2["Disk 2 (Partition 7)"] -->|"掛載於 /d2"| D2P --> P12["/proj1, /proj2"]
    D3["Disk 3 (Partition 7)"] -->|"掛載於 /d3"| D3P --> P34["/proj3, /proj4"]
</pre>

<blockquote class="trap">
<strong>重要現象（<code>p.14</code>）：</strong>掛載完成後，該掛載點目錄<strong>原本存放在裡面的舊內容會暫時無法存取</strong>（直到卸載為止）。
</blockquote>

---

## 開機自動掛載 `/etc/fstab` 與卸載排障 `lsof`

<div class="cols">
<div>

### 開機自動掛載 `/etc/fstab`（`p.16`）
列於 `/etc/fstab` 的檔案系統會在**開機時自動檢查與掛載**，紀錄含 6 欄：
- `Device`、`Mountpoint`、`FStype`、`Options`（如 `rw`、`sw`）、`Dump`、`Pass#`
- Swap 分割區：`Mountpoint` 寫 `none`、`FStype` 寫 `swap`

</div>
<div>

### 卸載與 Busy 排障（`p.17-18`）
```sh
umount /home    # 或 umount <device>
umount -f ...   # 忙碌時強制卸載
```
- **Busy 兩原因**：有人 `cd` 停在裡面（`cwd`），或有檔案被開啟
- 用 **`lsof`**（list open files）查出佔用的 `PID` / `COMMAND` / `FD`（`cwd`, `txt`）

</div>
</div>

---

<!-- _class: divider -->

# 單元二：七大檔案類型與行程通訊物件

從 `ls -l` 第一個字元看懂 Linux 裡的檔案、裝置、連結、Socket 與管線。

---

## 七大檔案類型符號與 `file` 指令

在 `ls -l` 最左側的第一個字元，直接告訴你這個檔案物件屬於哪一類：（`p.19-20`）

| 符號 | 檔案類型 | 說明（`p.19-25`） |
| :---: | :--- | :--- |
| `-` | Regular file（一般檔案） | 可用 `file` 指令分辨內容（如 ASCII text、ELF executable） |
| `b` / `c` | Block / Character device file | 區塊裝置檔 / 字元裝置檔 |
| `d` | Directory（目錄） | 含 `.` 與 `..`；以 `mkdir` / `rmdir` 管理 |
| `l` | Symbolic link（符號連結） | 指向另一個路徑名稱（`ln -s ori-file soft-file`） |
| `s` | UNIX domain socket | 透過檔案系統物件而非網路 Port 做本機通訊 |
| `p` | Named pipe（具名管線） | 以 `mkfifo` 建立，讓不同行程進行 FIFO 通訊 |

---

## 目錄、UNIX Domain Socket 與 Pipes / `mkfifo`

<div class="cols">
<div>

### Socket 與匿名 Pipe（`p.22-23`）
- **UNIX domain socket**：由 `socket()` 建立、限**單一主機本機**通訊，`Process A` 寫入核心 `Socket buffer` 供 `Process B` 讀取
- **Pipes（FIFO 通訊）**：核心維護讀取端 `fd[0]` 與寫入端 `fd[1]`，經 **`Fork`** 後父行程寫 `fd[1]`、子行程讀 `fd[0]`

</div>
<div>

### Named Pipe（具名管線，`p.24`）
不同獨立行程可直接透過檔案系統上的管線檔名稱交換資料：
```sh
$ mkfifo pipe
$ du >> pipe
# 在另一個行程讀取：
$ sort -n pipe
```
語法：`mkfifo [-m mode] fifo_name ...`

</div>
</div>

---

<!-- _class: divider -->

# 單元三：檔名背後的真相——Inode 與硬／軟連結

理解 UNIX「人認檔名、系統認 Inode」的設計，徹底分清楚 Hard Link 與 Symbolic Link。

---

## UNIX 磁碟分割區四大組成與 Inode 結構

<div class="cols">
<div>

### 分割區四大區塊（`p.27`）
1. **Boot block**：啟動 OS 的 bootstrap 程式
2. **Super block**：分割區總大小、block size、**根目錄的 inode 編號**
3. **Inode list（i-list）**：線性 inode 陣列
4. **Data blocks**：檔案實際資料與目錄項目

</div>
<div>

### `inode` 裡記錄什麼？（`p.26`）
用 **`ls -i`** 可查看檔案的 inode 編號；inode 記錄：
- `mode`、`owners(2)`、`timestamps(3)`、`size block`、`count`
- 指向 `data` 區塊的四級指標：`direct blocks`、`single indirect`、`double indirect`、`triple indirect`

</div>
</div>

---

## 目錄如何對應 Inode？Directory Block 與 `.` / `..`

**使用者用「檔名」想檔案，UNIX 核心用「inode」認檔案**——目錄塊存放的正是 `(inode number, filename)` 對照表：（`p.27-29`）

<pre class="mermaid">
flowchart LR
    PDIR["父目錄 Directory block: (1267, '.') / (2549, 'testdir')"] -->|"查找 testdir"| IN2["i-list 中 inode 2549"]
    IN2 -->|"指向目錄資料塊"| CDIR["testdir Directory block: (2549, '.') / (1267, '..')"]
    CDIR -->|"'.' 指自己"| IN2
    CDIR -->|"'..' 指回父目錄"| IN1["i-list 中 inode 1267"]
</pre>

- 當我們存取 `testdir` 時，系統是從目錄塊查出 `testdir` 對應 `2549`，再去 `i-list` 找出 `inode 2549` 及其指向的資料塊（`p.28-29`）

---

## Hard Link vs. Symbolic Link：本質差別與指令

<div class="cols">
<div>

### Hard Link（硬連結，`p.30`）
- 把兩個以上的檔名**關聯到同一個 inode**
- **限制**：**必須在同一個分割區內**（因為不同分割區各有獨立 `i-list`）
- 指令：
```sh
$ ln ori-file hard-file
```

</div>
<div>

### Soft / Symbolic Link（軟連結，`p.25, 30`）
- 是一個獨立檔案（有自己的 inode），內容**指向另一個路徑名稱（pathname）**，像 Windows 捷徑
- 可跨分割區指向其他路徑
- 指令：
```sh
$ ln -s ori-file soft-file
```

</div>
</div>

---

## 經典三階段實驗：建立 → `rm` 刪除 → `touch` 重建

看懂下面三步，就永遠不會弄錯 Hard Link 與 Symbolic Link：（`p.31`）

<pre class="mermaid">
flowchart LR
    subgraph A["(A) ln myfile hardlink / ln -s myfile symlink"]
        AM["myfile"] --> AN1["N1 / B1"]
        AH["hardlink"] --> AN1
        AS["symlink"] --> AN2["N2 / B2"] -. "路徑: myfile" .-> AM
    end
    subgraph B["(B) rm myfile"]
        BH["hardlink"] --> BN1["N1 / B1 (仍有效)"]
        BS["symlink"] --> BN2["N2 / B2"] -. "指向不存在" .-> X["now points nowhere"]
    end
    subgraph C["(C) touch myfile (重建同名檔)"]
        CM["新 myfile"] --> CN3["N3 / B3"]
        CH["hardlink"] --> CN1["N1 / B1 (仍是舊資料)"]
        CS["symlink"] --> CN2["N2 / B2"] -. "自動改指新檔" .-> CM
    end
</pre>

---

<!-- _class: divider -->

# 單元四：權限管理與 Bit Mask 徹底拆解

用二進位位元（Bit Mask）的視角，一次搞懂 `rwx` 數字權限、特殊位元 `4000/2000/1000`、`umask` 與最低權限表！

---

## 第一步：認識 9 個基本權限位元與 3 群組分工

在 `ls -l` 中，檔型符號後面的 **9 個字元**其實就是 **9 個獨立的二進位開關（bit）**，平均分成 3 組：（`p.32`）

```text
檔案型別   User (擁有者 u)   Group (群組 g)   Other (其他人 o)
   -          r  w  x          r  -  x          r  -  x
位元開關：     1  1  1          1  0  1          1  0  1
```

- **User (`u`)**：檔案擁有者自己（前 3 個位元）
- **Group (`g`)**：與檔案同群組的使用者（中間 3 個位元）
- **Other (`o`)**：既非擁有者也不在該群組的其他所有人（最後 3 個位元）
- **每組內三個位置順序永遠固定**：第 1 格 `r`（讀）、第 2 格 `w`（寫）、第 3 格 `x`（執行）

---

## 第二步：`rwx` 的二進位 Bit Mask 與權重換算（`4 / 2 / 1`）

每一組只有 3 個位元，從左到右的二進位權重剛好是 **$2^2 = 4$、$2^1 = 2$、$2^0 = 1$**，把為 `1` 的位元加總就得到 `0`–`7` 數字：（`p.32, 35`）

<div class="cols">
<div>

| `r` (`4`) | `w` (`2`) | `x` (`1`) | 加總數字 | 顯示字串 |
| :---: | :---: | :---: | :---: | :--- |
| `1` | `0` | `0` | **`4`** | `r--` |
| `0` | `1` | `0` | **`2`** | `-w-` |
| `0` | `0` | `1` | **`1`** | `--x` |
| `1` | `0` | `1` | **`5`** (`4+1`) | `r-x` |
| `1` | `1` | `0` | **`6`** (`4+2`) | `rw-` |
| `1` | `1` | `1` | **`7`** (`4+2+1`) | `rwx` |

</div>
<div>

### 實例拆解：`755` 與 `644`
- **`755`** = `111` · `101` · `101`
  - `u=7 (rwx)`、`g=5 (r-x)`、`o=5 (r-x)`
  - 合起來就是 **`rwx r-x r-x`**（`p.32, 35`）
- **`644`** = `110` · `100` · `100`
  - `u=6 (rw-)`、`g=4 (r--)`、`o=4 (r--)`
  - 合起來就是 **`rw- r-- r--`**（`p.35`）

</div>
</div>

---

## 第三步：`chmod` 符號式修改 vs. 數字位元遮罩

`chmod(1)` 支援兩種修改權限的語法（詳見 `chmod(1)` `"MODES"` 章節）：（`p.32, 35`）

<div class="cols">
<div>

### 1. 符號式（`chmod access-string file`）
組合規則：**`[對象 u/g/o/a] + [符號 +/-/=] + [權限 r/w/x]`**
- `+` 加入指定權限、`-` 移除權限、`=` 精確設定（`p.32`）
```sh
$ chmod u+x test.sh
$ chmod go-w myfile
$ chmod u+w,g-w hehe haha
```

</div>
<div>

### 2. 數字 Bit Mask 式（一次重設全部位元）
直接給定完整位元權重數字；加上 **`-R`** 可遞迴修改目錄及底下內容：
```sh
$ chmod -R 755 public_html/
```
- 符號式適合「微調單一對象」（如只幫擁有者加 `+x`）
- 數字式適合「一口氣覆寫成目標權限遮罩」（`p.32, 35`）

</div>
</div>

---

## 第四步：高位 3 個特殊權限位元（`4000 / 2000 / 1000`）

在 9 個基本權限位元**更左邊**，還有 **3 個高位特殊位元（千位數字）**，一樣對應 `4 / 2 / 1` 二進位遮罩：（`p.35`）

| 高位數值 | 高位 3-bit (`s u t` 對應遮罩) | 名稱 | 落在哪個位置顯示？ | 實例拆解（`p.35`） |
| :---: | :---: | :--- | :--- | :--- |
| **`4000`** | `1 0 0` | **`setuid`** | User 的 `x` 位置顯示 **`s`** | **`4755`** → `- rws r-x r-x`（`4555` → `- r-s r-x r-x`） |
| **`2000`** | `0 1 0` | **`setgid`** | Group 的 `x` 位置顯示 **`s`** | **`2755`** → `- rwx r-s r-x`（`2775` → `d rwx rws r-x`） |
| **`1000`** | `0 0 1` | **`sticky bit`** | Other 的 `x` 位置顯示 **`t`** | **`1777`** → `d rwx rwx rwt` |

> **速記法**：特殊位元同樣遵循 `4 (setuid)`、`2 (setgid)`、`1 (sticky)`；放在三組權限數字最前面形成 4 位數字（例：`4000 + 755 = 4755`）。（`p.35`）

---

## 第五步：特殊權限在「檔案」與「目錄」上的實際效果

<div class="cols">
<div>

### 套用在「檔案」上（`p.33`）
- **`setuid`（`4000`）**：任何人執行該程式時，產生的行程之**有效 UID** 會變成**該檔案擁有者的 UID**
  - 常見實例：`passwd`、`chsh`、`crontab`
- **`setgid`（`2000`）**：產生行程之**有效 GID** 會變成**該檔案的群組 GID**

</div>
<div>

### 套用在「目錄」上（`p.34`）
- **`setgid`（`2000`）**：在這個目錄裡**新建的所有檔案，都會自動繼承該目錄的群組**
- **`sticky`（`1000`）**（例：`/tmp` 的 `1777`）：禁止刪除或重新命名目錄內檔案，**除非你是**：
  1. 該檔案的擁有者（owner of the file）
  2. 該目錄的擁有者（owner of the directory）
  3. `root`

</div>
</div>

---

## 第六步：常見權限數字模式速讀對照表

將 Bit Mask 訓練成反射動作——看到數字立刻讀出物件用途與 `rwx` 字串：（`p.35`）

| Mode | Attribute | 典型身分 | Mode | Attribute | 典型身分 |
| :---: | :--- | :--- | :---: | :--- | :--- |
| `644` | `- rw- r-- r--` | 一般檔案（本人寫、他人讀） | `755` | `d rwx r-x r-x` | 一般目錄（本人改、他人可看可進） |
| `600` | `- rw- --- ---` | 私密檔案（僅本人讀寫） | `750` | `d rwx r-x ---` | 群組目錄（阻擋 Other 存取） |
| `444` | `- r-- r-- r--` | 全體唯讀檔案 | `700` | `d rwx --- ---` | 私密目錄（僅本人可進出） |
| `755` | `- rwx r-x r-x` | 一般可執行檔 | `711` | `d rwx --x --x` | 可穿越但不可列出他人清單的目錄 |
| `4755` | `- rws r-x r-x` | `setuid` 執行檔 | `2775` | `d rwx rws r-x` | 同群組協作且繼承群組之目錄 |
| `4555` | `- r-s r-x r-x` | 唯讀 + `setuid` 執行檔 | `1777` | `d rwx rwx rwt` | `/tmp` 共用暫存目錄（Sticky 保護） |

---

## 第七步：預設權限從哪來？認識 `umask` 與位元遮罩公式

剛建立新檔案或新目錄時，系統怎麼決定預設權限？靠的是 Shell 內建指令 **`umask`**：（`p.36`）

- **兩個固定起跑點（Full permission bit）**：
  - **新建一般檔案（File）**：起跑點為 **`666`**（二進位 **`110 110 110`**，即 `rw- rw- rw-`，預設不給 `x`）
  - **新建目錄（Dir）**：起跑點為 **`777`**（二進位 **`111 111 111`**，即 `rwx rwx rwx`）
- **核心 Bit Mask 遮罩公式（`p.36`）**：新物件權限 = **`full permission bit & (!umask)`**
  - **`umask` 的每個 bit 只要是 `1`，就代表「要遮掉／禁止的權限」**！
  - 先對 `umask` 做逐位反相（`!umask`，原本 `1` 變 `0`、原本 `0` 變 `1`），再與起跑點做逐位 **`AND (&)`**，就能精準把要遮掉的 bit 清成 `0`。

---

## 第八步：Bit Mask 逐位演算（一）`umask 022` 與 `066`

用逐位反相 + AND 拆解兩種最實用的 `umask`：（`p.36`）

<div class="cols">
<div>

### 例 1：`umask = 022`（`000 010 010`）
1. **反相 `!022`**：`111 101 101`（遮掉 `g` 與 `o` 的 `w`）
2. **新檔案（起跑 `666 = 110 110 110`）**：
   `110 110 110 & 111 101 101 = 110 100 100`
   → **`644`（`- rw- r-- r--`）**
3. **新目錄（起跑 `777 = 111 111 111`）**：
   `111 111 111 & 111 101 101 = 111 101 101`
   → **`755`（`d rwx r-x r-x`）**

</div>
<div>

### 例 2：`umask = 066`（`000 110 110`）
1. **反相 `!066`**：`111 001 001`（遮掉 `g` 與 `o` 的 `r, w`）
2. **新檔案（起跑 `666 = 110 110 110`）**：
   `110 110 110 & 111 001 001 = 110 000 000`
   → **`600`（`- rw- --- ---`）**
3. **新目錄（起跑 `777 = 111 111 111`）**：
   `111 111 111 & 111 001 001 = 111 001 001`
   → **`711`（`d rwx --x --x`）**

</div>
</div>

---

## 第九步：Bit Mask 逐位演算（二）必考陷阱 `umask 033` 與 `277`

為什麼 **`umask 033` 的新檔案不是 `633`**？看二進位 `AND` 一次瞬間明白：（`p.36`）

<div class="cols">
<div>

### 破解 `umask = 033`（`000 011 011`）
- 反相 **`!033`** = `111 100 100`（要遮掉 `g/o` 的 **`w` 與 `x`**）
- File `666` = `110 110 110`（`g/o` 本來就**沒有 `x`**）
- 逐位 AND：`110 110 110 & 111 100 100 = 110 100 100`
- 結果：**File = `644`（`- rw- r-- r--`）**，**Dir = `744`（`d rwx r-- r--`）**

</div>
<div>

### 完整 `umask` 對照表（`p.36`）
| `umask` | New File | New Dir |
| :---: | :--- | :--- |
| `022` | `- rw- r-- r--` (`644`) | `d rwx r-x r-x` (`755`) |
| `033` | `- rw- r-- r--` (`644`) | `d rwx r-- r--` (`744`) |
| `066` | `- rw- --- ---` (`600`) | `d rwx --x --x` (`711`) |
| `000` | `- rw- rw- rw-` (`666`) | `d rwx rwx rwx` (`777`) |
| `277` | `- r-- --- ---` (`400`) | `d r-x --- ---` (`500`) |
| `777` | `- --- --- ---` (`000`) | `d --- --- ---` (`000`) |

</div>
</div>

<blockquote class="trap">
<strong>考試禁忌：絕對不可用「減法」算 <code>umask</code>！</strong> 若用 <code>666 - 033 = 633</code> 會荒謬地生出原本沒有的 <code>x</code> 位元（<code>-wx</code>）；只有 <code>full &amp; (!umask)</code> 永遠正確！
</blockquote>

---

## 第十步：File Protection 最低權限表（檔 vs. 目錄）

操作檔案時，「檔案本身」與「所在目錄」各自需要什麼最低權限？（`p.37`）

| 指令操作 | On file itself（檔案本身） | On directory（所在目錄） | 關鍵觀念 |
| :--- | :---: | :---: | :--- |
| `cd /home/test` | — | **`x`** | 切換進目錄只需要穿越（`x`） |
| `ls /home/test` | — | **`r`** | 單純列出檔名只需目錄 `r` |
| `ls -s /home/test/*.c` | — | **`rx`** | 讀取目錄內各檔案大小需目錄 `rx` |
| `cat runme` / `cat >> runme` | **`r`** / **`w`** | **`x`** | 經目錄（`x`）進入後讀（`r`）或寫（`w`） |
| `run-binary` vs. **`run-script`** | **`x`** vs. **`rx`** | **`x`** | 腳本須讓直譯器讀取內容，故檔案需 **`rx`** |
| **`rm rumme`** | **不需任何權限** | **`wx`** | 刪檔改的是**目錄項目**，看的是目錄 `wx`！ |

---

## 第十一步：變更擁有者與群組（`chown` 與 `chgrp`）

當需要調整檔案或整個目錄樹的擁有者（User Owner）或群組（Group Owner）時：（`p.38`）

- **`chown(8)`**：變更檔案擁有者（亦可同時或單獨變更群組）
- **`chgrp(1)`**：變更群組擁有者（加 **`-R`** 代表遞迴套用至子目錄與檔案）

```sh
$ chown -R tsaimh /home/tsaimh        # 僅變更 user owner
$ chown -R tsaimh:dcs /home/tsaimh    # 同時變更 user owner 與 group owner
$ chown -R :dcs /home/tsaimh          # 只改 group owner（冒號前省略）
$ chgrp -R dcs /home/tsaimh           # 使用 chgrp 變更 group owner
```

---

<!-- _class: divider -->

# 單元五：現代檔案系統進階特性

認識 Journaling、CoW（寫入時複製）與擴充屬性，銜接後續 LVM 與 Btrfs 儲存主題。

---

## Journaling 檔案系統：先寫日誌與崩潰恢復流程

先將操作記錄寫入 **journal** 再**非同步提交（commit asynchronously）**，可大幅縮短 `fsck` 時間並降低不一致（例：`ext3`、`ext4`、`xfs`、`btrfs`）：（`p.42`）

<pre class="mermaid">
flowchart LR
    W["寫入操作日誌到 Journal"] --> C["非同步 Commit 至檔案系統"]
    W -. "若系統崩潰 (Crash): 檢查 Journal" .-> CHK{"紀錄狀態?"}
    CHK -- "fully committed" --> S["skip (略過)"]
    CHK -- "partial committed" --> RC["rollback or commit (回滾或提交)"]
    CHK -- "non-committed" --> IC["ignore or commit (忽略或提交)"]
</pre>

---

## CoW（寫入時複製）與 File Attribute Extension

<div class="cols">
<div>

### CoW File System（`p.43`）
複製資料若**尚未修改**，兩者會**指向同一實體儲存位址**（例：`ZFS deduplication`）：
- **優點**：節省使用空間、**低成本快照（Low cost snapshots）**
- **缺點**：資料不一致風險（如 reference count 不一致）、看到的並非真實檔案佔用空間

</div>
<div>

### File Attribute Extension（`p.44`）
在 **`inode`** 裡以 **Key-value pairs** 儲存「不被檔案系統主動解譯」的額外中繼資料：
- `mime_type`
- `md5/sha1 checksum`
- `security attributes`

</div>
</div>

---

## 總結：檔案系統與權限 Bit Mask 核心小抄

| 必背主題 | 一句話心法與公式 | 來源頁碼 |
| :--- | :--- | :--- |
| 掛載與卸載排障 | `mount` 會暫時遮蔽原目錄內容；`umount` 遇到 busy 用 `lsof` 查 `cwd` 或開檔行程 | `p.14, 17-18` |
| Hard vs. Soft Link | 硬連結共用同一個 inode（限同分割區），`rm` 原檔仍可讀；軟連結看路徑名 | `p.25, 30-31` |
| 特殊權限高位遮罩 | `4000` (`setuid`)、`2000` (`setgid`，目錄繼承群組)、`1000` (`sticky`，保護 `/tmp`) | `p.33-35` |
| `umask` Bit Mask | **`full & (!umask)`**（File `666` / Dir `777`），禁止用減法（`033` → File `644`） | `p.36` |
| 最低保護權限 | `rm` 不需檔案權限只要目錄 **`wx`**；執行腳本要檔案 **`rx`** + 目錄 **`x`** | `p.37` |

[閱讀完整 Wiki 講義](../filesystem/)

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
