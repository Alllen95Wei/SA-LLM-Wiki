---
marp: true
theme: sa-wiki
paginate: true
size: 16:9
header: "System Administration (Debian 13) — 跨章對照教學簡報"
footer: "知識來源：wiki/storage-partition-lvm-btrfs.md（10_FileSystem / LVM / BTRFS） ｜ System Administration LLM Wiki"
---

<!-- _class: lead -->
<!-- _paginate: skip -->

# 儲存管理總對照：傳統分割區、LVM 與 Btrfs

給初學者的跨章架構釐清、擴縮容順序與 CoW 快照終極比較指南

- **這份簡報解答三個最常見困惑**：三者各自管哪一層？為什麼課程要把 Btrfs 建在 LVM 上？快照跟還原差在哪？（`10_FileSystem / LVM / BTRFS`）
- **比較定位**：嚴格整合三篇儲存主題講義，幫助你考前一次打通所有關聯（`10_FileSystem.pdf`、`LVM.pdf`、`BTRFS.pdf`）
- [閱讀完整 Wiki 總對照頁](../storage-partition-lvm-btrfs/)

---

## 學習地圖：學完這三章後，我們要釐清這 4 個問題

1. **各自負責什麼層級？為什麼會把 LVM 和 Btrfs 疊在一起？** — 區塊層儲存池 vs. 檔案系統層 Subvolume（`LVM.pdf, p.2-3, 18`；`BTRFS.pdf, p.2, 4, 13`）
2. **想要加大或拔除硬碟時，步驟順序有何差別？** — 分割區順序限制 vs. LVM 的 Bottom→Top／Top→Bottom vs. Btrfs resize（`LVM.pdf, p.2, 13-30`）
3. **都是 CoW 快照，LVM 和 Btrfs 差在哪裡？怎麼還原？** — 預切容量、多快照寫入負擔與 Rollback 天壤之別（`10_FileSystem.pdf, p.43`；`LVM.pdf, p.10`；`BTRFS.pdf, p.8-10`）
4. **`ext4` / `xfs` / `btrfs` / `ZFS` 與 `/etc/fstab` 實戰怎麼選？** — 情境決策樹與開機自動掛載寫法（`BTRFS.pdf, p.3, 7`；`10_FileSystem.pdf, p.16`）

---

<!-- _class: divider -->

# 單元一：分層定位與疊加混用

分清楚「硬碟分割區」、「LVM 儲存池」與「Btrfs 檔案系統」在 I/O 堆疊中的上下關係。

---

## 一次看懂三種儲存堆疊（含課堂混用架構）

三者在 Linux 系統裡並不是「只能三選一」——特別是 LVM 與 Btrfs 常被**上下疊加使用**：

<pre class="mermaid">
flowchart LR
    subgraph S1["1. 傳統分割區直掛 (10_FileSystem, p.14-15)"]
        D1["磁碟分割區 (/dev/sda1)"] --> F1["檔案系統 (ext4 / xfs)"] --> M1["掛載點"]
    end
    subgraph S2["2. LVM + 傳統檔案系統 (LVM, p.3)"]
        D2["實體磁碟 / RAID"] --> L2["PV -> VG -> LV"] --> F2["ext3 / xfs"] --> M2["掛載點"]
    end
    subgraph S3["3. LVM + Btrfs 混用 (LVM p.18 / BTRFS p.13)"]
        D3["多顆分割區 (sda1+sda2)"] --> L3["PV -> VG -> LV"] --> BT["mkfs.btrfs (ID 5)"] --> SV["Subvolumes (@ray)"] --> M3["mount -o subvol=..."]
    end
</pre>

---

## 傳統分割區 vs. LVM vs. Btrfs 核心定位對照表

| 比較面向 | 傳統分割區 | LVM（`LVM.pdf, p.2-7`） | Btrfs（`BTRFS.pdf, p.2, 4`） |
| :--- | :--- | :--- | :--- |
| **所在系統層級** | 磁碟分割層（直接格式化） | **Device Mapper（`/dev/mapper/`）區塊層儲存池** | **現代化 CoW 檔案系統層** |
| **最小／主要單位** | 固定分割區 + 獨立 Inode list（`10_FileSystem, p.27`） | `PV → VG → LV`（`PE/LE` 預設 `4MB`） | `Top-level (ID 5)` 與 `@` Subvolume（獨立 inode 空間） |
| **多磁碟整併能力** | 各自掛載於不同目錄（`10_FileSystem, p.15`） | 合併多顆磁碟／分割區／RAID 成一大儲存池 | 內建捲宗管理，支援軟體 **RAID 0、RAID 1** 等 |

---

## 為什麼實戰常把 LVM 和 Btrfs 疊在一起用？

在課堂實機演練（`LVM.pdf, p.18, 28-30` 與 `BTRFS.pdf, p.13-17`）中，常見將 `/dev/data-vg/data_vol` 或 `/dev/home-vg/home-vol` **先做 LVM，再格式化成 Btrfs**：

<div class="cols">
<div>

### 下層 LVM 負責「區塊彈性」
- 把多顆硬碟／分割區（如 `sda1` + `sda2`）先匯聚進同一 VG（`p.13`）
- 日後可透過 `vgextend` / `lvextend` 擴容，或用 `vgreduce` / `pvremove` 安全抽換實體磁碟（`p.17, 30`）

</div>
<div>

### 上層 Btrfs 負責「Subvolume 與高效率快照」
- 不用在 LV 裡再硬切多塊分割區，改用 **`@ray` 等 Subvolume** 分層管理目錄（`p.4, 15`）
- 享有免預定容量、多快照不降寫入效能的 CoW 快照與壓縮功能（`p.2, 9`）

</div>
</div>

---

<!-- _class: divider -->

# 單元二：空間規劃、佈局與擴縮容順序

調整空間大小時，傳統分割區最受限制，LVM 有嚴格方向性，Btrfs 則靠 Subvolume 與佈局切換彈性。

---

## 空間怎麼切？比較三者的彈性設計

<div class="cols">
<div>

### 傳統分割區 & LVM（`LVM.pdf, p.2, 7-9, 12`）
- **傳統分割區**：受限於 **partition precedence**（分割區排列順序）
- **LVM**：破除分割區順序限制，即使**單顆磁碟**也很實用；更提供 **Thin Provisioning**（`Thin Pool` = `metadata` + `data`；`Thin LV` 以 `-V` 設上限）做 VM 按需分配

</div>
<div>

### Btrfs 佈局選擇（`BTRFS.pdf, p.4-7`）
- 用 **Subvolume** 在共用池內切出獨立目錄階層：
  - **Nested layout**：隨父層自動掛載並繼承選項；**刪父前必先刪全部子 subvolume**
  - **Flat layout**：全部位於 top-level 下，須逐一掛載，但**每個掛載點能給不同掛載選項**

</div>
</div>

---

## 擴充 vs. 縮容方向總對照（考試必背順序！）

只要牽涉 LVM，**「變大由下往上、變小由上往下」** 是絕對不能顛倒的鐵則：（`LVM.pdf, p.13, 19-30`）

<pre class="mermaid">
flowchart LR
    subgraph EXT["擴充方向：LVM 由下而上 (Bottom -> Top)"]
        E1["1. fdisk (GPT 類型 44 + w)"] --> E2["2. pvcreate"] --> E3["3. vgextend"] --> E4["4. lvextend (--resizefs)"] --> E5["5. resize2fs (線上擴展)"]
    end
    subgraph SHR["縮容拆碟方向：由上而下 (Top -> Bottom)"]
        S1["1. 確認已用小於剩餘 PV"] --> S2["2. btrfs filesystem resize -2g / resize2fs"] --> S3["3. lvreduce"] --> S4["4. vgreduce"] --> S5["5. pvremove 後退碟"]
    end
</pre>

> 若 LVM 的上層就是 Btrfs（如 `/data` 案例），拆硬碟前一樣得先跑 `btrfs filesystem resize -2g /data/` 把檔案系統縮好，再接 `lvreduce` → `vgreduce` → `pvremove`！（`LVM.pdf, p.29-30`）

---

<!-- _class: divider -->

# 單元三：CoW 快照與還原（Rollback）全面對決

這是最常出成比較題的核心章節：LVM 與 Btrfs 都使用 CoW，但體驗與還原步驟截然不同！

---

## 共同基礎：什麼是 Copy-on-Write（CoW）？（`10_FileSystem.pdf, p.43`）

在討論兩家快照前，先複習 `10_FileSystem.pdf` 定義的 **CoW 原理與權衡**：

<div class="cols">
<div>

### 原理與優點（Pros，`p.43`）
- 複製但尚未修改的資料，會**參照儲存裝置上同一個實體位址**
- **減少使用空間**（Reduce space used）
- **低成本快照**（Low cost snapshots：瞬間建指標）

</div>
<div>

### 缺點與風險（Cons，`p.43`）
- **資料不一致風險**（例如 reference count 不一致）
- 看到的空間數字**並非「真實」檔案佔用空間**（Not "real" used space on file）

</div>
</div>

---

## LVM Snapshot vs. Btrfs Snapshot 六大維度對照表

| 比較維度 | LVM Snapshot（`LVM.pdf, p.10`） | Btrfs Snapshot（`BTRFS.pdf, p.8-10`） |
| :--- | :--- | :--- |
| **快照的本質** | **本身是一個 LV** | **本身是一個 Subvolume**（可 `-r` 唯讀；可寫與原 subvolume 無異） |
| **建快照需指定大小？** | **要**（常設目標 LV 的 **20–30%**） | **不要**（免預指定容量） |
| **多份快照影響** | 寫入同 LV 時會寫向所有快照，**寫速變慢且略浪費空間** | 多份快照（Reflink）**不浪費效能與空間**（`BTRFS.pdf, p.9`） |
| **遞迴行為** | 針對單一 LV | **不具遞迴性（Not recursive）**（`BTRFS.pdf, p.8`） |
| **Rollback 還原指令** | **`umount` 目標 LV** 後執行 **`lvconvert --merge`** | **沒有內建 rollback 指令**！（改複製檔案或重拍可寫 subvolume 重掛） |

---

## 還原（Rollback）流程最大差異圖解

<pre class="mermaid">
flowchart LR
    subgraph L["LVM 快照還原 (LVM.pdf, p.10)"]
        L1["目標 LV 工作中"] --> L2["1. 必先 umount 目標 LV"] --> L3["2. lvconvert --merge snapshot_LV"]
    end
    subgraph B["Btrfs 快照還原 (BTRFS.pdf, p.10)"]
        B1["1. 無內建 rollback 指令 (btrfs subvolume list -t 找快照)"] --> B2["2. 從快照拍可寫副本 @ray_new"] --> B3["3. mount -o subvol=/home/@ray_new 重新掛載"]
    end
</pre>

- 考題若問「哪一個沒有內建一鍵 `rollback` 指令？哪一個還原前必先 `umount`？」，看這張圖就能立刻秒答！（`LVM.pdf, p.10`；`BTRFS.pdf, p.10`）

---

## 初學者最常踩的 3 個快照與縮容地雷

<blockquote class="trap">
<strong>地雷 1 — 把 Snapshot 當成 Backup！</strong><br>
<code>LVM.pdf, p.10</code> 明確警告 <strong>Not backup!</strong>；做任何儲存架構調整或測試前，依然要先備份原始資料（<code>LVM.pdf, p.14</code>）。
</blockquote>

<blockquote class="trap">
<strong>地雷 2 — 以為 Btrfs Snapshot 會遞迴拍進所有子目錄</strong><br>
<code>btrfs subvolume snapshot</code> 預設 <strong>Not recursive</strong>（<code>BTRFS.pdf, p.8</code>）；遇到 Nested layout 還要記得「刪父 subvolume 前必先刪全部子層」（<code>BTRFS.pdf, p.5</code>）。
</blockquote>

<blockquote class="trap">
<strong>地雷 3 — 還原或拆碟忘記前置條件</strong><br>
LVM 快照合併前必先 <code>umount</code>（<code>LVM.pdf, p.10</code>）；LVM 移除 PV 前必先確認已用空間小於剩餘 PV，並由上而下先縮檔案系統（<code>LVM.pdf, p.27-30</code>）。
</blockquote>

---

<!-- _class: divider -->

# 單元四：檔案系統選型、`/etc/fstab` 與情境決策

把 `ext4`、`xfs`、`btrfs`、`ZFS` 與三種 `/etc/fstab` 寫法連成完整的考前決策地圖。

---

## 課程四款檔案系統定位快速比較

| 檔案系統 | 機制分類 | 課程核心評語與功能亮點（`10_FileSystem / LVM / BTRFS`） |
| :--- | :--- | :--- |
| **`ext4`**（`ext3`） | Journaling | **最常見、具備基礎功能**；縮短 `fsck` 時間；可配 LVM `resize2fs` 線上擴充（`10_FS, p.42`; `LVM, p.26`; `BTRFS, p.3`） |
| **`xfs`** | Journaling | 日誌檔案系統之一，亦常用於 LVM 上掛載資料（如 `/data` 示意）（`10_FS, p.42`; `LVM, p.3`） |
| **`btrfs`** | Journaling / CoW | **免預配空間快照**、軟體 RAID 0/1、Self-healing、壓縮（`compress=zstd`）、Reflinks、`btrfs filesystem resize`（`10_FS, p.42`; `BTRFS, p.2-3, 9`） |
| **`ZFS`** | CoW | 具備多項進階功能（如 `deduplication`），但**記憶體佔用相對較高**（`10_FS, p.43`; `BTRFS, p.3`） |

---

## `/etc/fstab` 三種寫法一次對照

| 情境類型 | 核心差異與設定重點（`10_FileSystem, p.16`；`LVM, p.18`；`BTRFS, p.7, 16`） |
| :--- | :--- |
| **通用 / Swap 基本格式** | 6 欄：`Device  Mountpoint  FStype  Options  Dump  Pass#`；swap 之 Mountpoint 填 `none` |
| **LVM LV 重開機掛載** | 建立新 LV 掛載後須**修訂 `/etc/fstab`**（對應 `/dev/<VG>/<LV>` 或 `/dev/mapper/...`），重開機才會自動掛載 |
| **Btrfs Flat Layout** | 先用 `btrfs filesystem show` 查 `UUID`，於 Options 加 **`subvol=`**，可替 `/` 設 `autodefrag`、替 `/home` 設 `noatime` |

```text
UUID=the-btrfs-fs-uuid  /      btrfs  subvol=@,defaults,autodefrag  0 0
UUID=the-btrfs-fs-uuid  /home  btrfs  subvol=@home,defaults,noatime 0 0
```

---

## 考題情境儲存選型決策樹

考試遇到情境題「該選傳統 FS、LVM、Btrfs 還是 ZFS？」，照這棵決策樹判斷：（`10_FS` / `LVM` / `BTRFS`）

<pre class="mermaid">
flowchart TD
    Q["題目儲存需求?"] -->|"穩定基本功能、無快照需求"| A1["ext4 (可建於一般分割區或 LVM LV)"]
    Q -->|"多顆磁碟整併 / 破除 partition precedence / VM Thin Pool"| A2["導入 LVM (PV -> VG -> LV / Thin Pool)"]
    Q -->|"免指定快照大小 / 多快照不降速 / 壓縮 / Self-healing"| A3{"記憶體預算與系統需求?"}
    A2 -. "上層搭配需求" .-> A3
    A3 -->|"Linux 原生且不似 ZFS 高耗記憶體"| A4["使用 Btrfs (Subvolume + CoW Snapshot)"]
    A3 -->|"注意對照點"| A5["ZFS 亦有多項功能但記憶體相對高"]
</pre>

---

## 總結：儲存系列三大技術口訣

1. **分層各司其職**：分割區切實體盤；**LVM（`PV→VG→LV`）管區塊儲存池與 Thin Pool**；**Btrfs（Subvolume `ID 5` / `@`）管檔案系統層 CoW 快照與個別掛載**——實戰還能把 Btrfs 建在 LVM 上！
2. **順序與前提別弄反**：LVM **建與擴充由下而上、縮與拆由上而下**（先縮 FS 再 `lvreduce` → `vgreduce` → `pvremove`）。
3. **快照與還原必對比**：**LVM 需指定 20–30% 容量、多快照慢、還原先 `umount` 再 `lvconvert --merge`**；**Btrfs 免指定容量、多快照不降速、無內建 rollback 須重拍重掛**。

[閱讀完整 Wiki 總對照頁](../storage-partition-lvm-btrfs/)

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
