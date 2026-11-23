---
marp: true
theme: sa-wiki
paginate: true
size: 16:9
header: "System Administration (Debian 13) — 教學型簡報"
footer: "知識來源：wiki/lvm.md（LVM.pdf） ｜ System Administration LLM Wiki"
---

<!-- _class: lead -->
<!-- _paginate: skip -->

# LVM 入門教學：邏輯捲宗、精簡配置、快照與磁碟擴縮

給初學者的彈性儲存池（Storage Pool）管理指南

- **核心定位**：LVM 建立在 Linux Device Mapper（`/dev/mapper/`）之上，位於實體磁碟與檔案系統之間（`LVM.pdf, p.2`）
- **主要優勢**：把多顆硬碟整合成一個儲存池，輕鬆調整空間大小，不再受限於傳統分割區順序（`LVM.pdf, p.2, 12`）
- [閱讀完整 Wiki 講義](../lvm/)

---

## 學習地圖：這份簡報會帶你學會什麼？

1. **為什麼傳統分割區不夠用？LVM 分了哪幾層？** — 從硬碟到 `PV → VG → LV` 與最小分配單位 `PE / LE`（`p.2-7, 12`）
2. **什麼是 Thin Provisioning 與 CoW 快照？** — 空間「按需分配」的魔法，以及不複製資料卻能瞬間留影的快照與還原（`p.8-11`）
3. **實際要新增、擴充或拔除硬碟時該怎麼做？** — 掌握 **由下而上** 與 **由上而下** 兩條黃金順序與三套實機範例（`p.13-30`）

---

<!-- _class: divider -->

# 單元一：為什麼需要 LVM？認識 PV、VG、LV 與 PE / LE

先看懂傳統硬碟的煩惱，再把多塊實體磁碟組裝成彈性虛擬分割區。

---

## 傳統分割區的煩惱 vs. LVM 儲存池

<div class="cols">
<div>

### 為什麼要用 LVM？（`p.2, 12`）
- 想同時合併多顆磁碟成同一塊大空間
- 目前只有一顆磁碟，但未來計畫再加磁碟擴充
- 不確定 `/home`、`/data` 等掛載點以後要多大
- **不想管理分割區相鄰順序（partition precedence）**——即使只有一顆磁碟也很實用！

</div>
<div>

### LVM 如何運作？（`p.2`）
- 透過 Linux 核心 **Device Mapper 框架（`/dev/mapper/`）**
- 在實體硬碟／分割區之上做出**儲存池（storage pool）**
- 檔案系統建立在「虛擬分割區」上，需要擴容或移碟時更有彈性

</div>
</div>

---

## LVM 六層架構全景圖（由下到上）

一張圖看懂底層硬碟、分割區如何一步步變成 `ext3` / `xfs` 掛載點：（`p.3`）

<pre class="mermaid">
flowchart LR
    subgraph HW["實體層"]
        D["Physical Drives (/dev/sdb, /dev/sdc) -> Partitions (sdb1, sdb2, sdc1, sdc2)"]
    end
    subgraph LVM["LVM 儲存池三層架構"]
        PV["Physical Volumes (PV)"] --> VG["Volume Group (VG: primary_vg)"]
        VG --> LV1["LV: /dev/primary_vg/home_lv"]
        VG --> LV2["LV: /dev/primary_vg/data_lv"]
    end
    subgraph FS["檔案系統層"]
        F1["/home (ext3)"]
        F2["/data (xfs)"]
    end
    D --> PV
    LV1 --> F1
    LV2 --> F2
</pre>

---

## 底層積木與儲存池：PV 與 VG

<div class="cols">
<div>

### 1. Physical Volume（PV，`p.4`）
- LVM 最底層的儲存積木：可以是整顆硬碟、分割區或 Linux software RAID（`/dev/mdXXX`）
- 建立時會在裝置寫入 **LVM header**
```sh
pvcreate /dev/sdb1 /dev/sdb2 /dev/sdc1 /dev/sdc2
```
- 查看工具：`pvs`、`pvdisplay`（`p.7, 24`）

</div>
<div>

### 2. Volume Group（VG，`p.5`）
- 把一個或多個 PV 綁成同一座 **儲存池（storage pool）**
```sh
vgcreate primary_vg /dev/sdb1 /dev/sdb2 /dev/sdc1 /dev/sdc2
```
- 之後容量不夠時，可直接把新 PV 併入：
```sh
vgextend <VG name> <PV name>
```
- 查看工具：`vgs`、`vgdisplay`（`p.7, 24`）

</div>
</div>

---

## 切出虛擬分割區：LV 與 `-l` vs. `-L` 語法

有了 VG 儲存池後，就能切出一塊塊可格式化掛載的 **Logical Volume（LV）**：（`p.6`）

- **指定 extents 數量或百分比（小寫 `-l`）**：
  ```sh
  lvcreate -l 50%FREE primary_vg -n home_lv     # 亦常用 -l 100%FREE
  ```
- **指定明確容量大小（大寫 `-L`）**：
  ```sh
  lvcreate -L 50G primary_vg -n data_lv
  ```
- **建立後的兩條等效裝置路徑（`p.6, 18`）**：
  - `/dev/<VG name>/<LV name>`
  - `/dev/mapper/<VG name>-<LV name>`（若名稱中原本含 `-`，在 `mapper` 下會顯示成 `--`）

---

## 最小分配單位：PE（預設 `4MB`）與 LE

LVM 並不是以「1 Byte」在切空間，而是靠 **Extent** 運作：（`p.7`）

<div class="cols">
<div>

### Physical Extent（PE）
- PV 中能配給 LV 的**最小連續單位**
- **預設大小為 `4MB`**
- LV 需要的空間是從 VG 分配一個個 PE 來拼出
- **PV 的容量必須是 PE 的整數倍**
- 用 `pvdisplay`、`vgdisplay` 查看

</div>
<div>

### Logical Extent（LE）
- 在 LV 層級與 PE 相對應的邏輯單位
- **大小與 PE 完全一樣**（一對一映射概念）
- 可用 `lvdisplay` 查看總共擁有幾個 LE

</div>
</div>

---

<!-- _class: divider -->

# 單元二：精簡配置與 CoW 快照

學會 LVM 最受虛擬機主機歡迎的兩項功能：**按需分配（Thin Provisioning）** 與 **Copy-on-Write 快照**。

---

## Thick vs. Thin Provisioning：為什麼能省空間？

<pre class="mermaid">
flowchart LR
    subgraph THICK["Thick Provisioning (預佔鎖定)"]
        TA["Volume A: DATA + 預佔但未用空間"]
        TB["Volume B: DATA + 預佔但未用空間"]
    end
    subgraph THIN["Thin Provisioning (按需分配，p.8-9)"]
        TP["Thin Pool (metadata + data 兩隱藏 LV)"] --> AV["共享 Available Storage"]
        VA["Thin LV A (-V 容量上限)"] -->|"用多少才拿多少"| AV
        VB["Thin LV B (-V 容量上限)"] -->|"用多少才拿多少"| AV
    end
</pre>

- **Thin Provisioning** 採用**按需分配（On-demand allocation）**，不用一開始就把未用到的空間鎖死，特別適合虛擬機器（VM）主機（`p.8-9`）

---

## 建立 Thin Pool 與 Thin LV

<div class="cols">
<div>

### Step 1：建立 Thin Pool（`p.8`）
Thin Pool 內部由 **2 個隱藏元件 LV（`metadata` 與 `data`）** 組成：
```sh
lvcreate --type thin-pool -n <pool name> -L <Size> <VG name>
```
先把 VG 的一部分實際容量配置成精簡儲存池。

</div>
<div>

### Step 2：建立 Thin LV（`p.8`）
Thin LV 住在 Thin Pool 裡，**`-V` 指定的是虛擬容量上限（Virtual size）**：
```sh
lvcreate --type thin -n <LV name> -V <VirtualSize> --thinpool <pool name> <VG name>
```
只有實際寫入資料時才會真正消耗池內空間。

</div>
</div>

---

## 什麼是 LVM CoW 快照？三階段變化圖解

**Copy-on-Write（CoW）** 讓快照在建立當下**不複製整份資料（僅複製指標）**，所以速度極快又省空間：（`p.10-11`）

<pre class="mermaid">
flowchart LR
    subgraph A["a. 建立快照前"]
        AF["Current FS"] --> AA["A"] & AB["B"] & AC["C"] & AD["D"]
    end
    subgraph B["b. 建立快照當下 (僅指到原塊)"]
        BS["Snapshot"] -.-> BA["A"] & BB["B"] & BC["C"] & BD["D"]
        BF["Current FS"] --> BA & BB & BC & BD
    end
    subgraph C["c. 資料修改後 (B刪/D改D+/新增E1,E2)"]
        CS["Snapshot (凍結原貌)"] -.-> CA["A"] & CB["B"] & CC["C"] & CD["D"]
        CF["Current FS"] --> CA & CC & CDP["D+"] & CE1["E1"] & CE2["E2"]
    end
</pre>

---

## 建立 LVM 快照與還原（Rollback）步驟

1. **建立快照**：**快照本身也是一個 LV**，空間大小通常設為目標 LV 的 **20–30%**（`p.10`）：
   ```sh
   lvcreate -L <SIZE> --snapshot --name <snapshot LV name> <target LV name>
   ```
2. **還原回快照時間點**：先 **`umount`** 目標 LV，再執行合併（`p.10`）：
   ```sh
   umount <target LV mountpoint>
   lvconvert --merge <snapshot LV name>
   ```

<blockquote class="trap">
<strong>絕對必記：Snapshot is Not backup!（<code>p.10</code>）</strong><br>
快照只保留建立後被改寫前的資料指標差異，若整顆底層磁碟或 VG 損壞，快照也會一起毀損，不能取代正式備份！
</blockquote>

---

<!-- _class: divider -->

# 單元三：三大實機演練——新增、擴充與移除磁碟

掌握考試最常考的順序差異：**新增與擴充「由下而上」**，**縮容與拆碟「由上而下」**！

---

## 操作前鐵則與兩條相反的順序方向

**任何異動前鐵則**：為了確保資料一致性，**先備份（backup）原始資料**！（`p.14`）

<pre class="mermaid">
flowchart LR
    subgraph UP["建立 / 擴充: 由下而上 (Bottom -> Top)"]
        U1["1. 磁碟/分割區"] --> U2["2. PV (pvcreate)"] --> U3["3. VG (vgcreate/vgextend)"] --> U4["4. LV (lvcreate/lvextend)"] --> U5["5. FS (mkfs/resize2fs) & mount"]
    end
    subgraph DN["縮容 / 移除: 由上而下 (Top -> Bottom)"]
        D1["1. 確認已用小於剩餘 PV"] --> D2["2. 縮 FS (btrfs resize/resize2fs)"] --> D3["3. 縮 LV (lvreduce)"] --> D4["4. 移 PV (vgreduce)"] --> D5["5. pvremove 後退碟"]
    end
</pre>

---

## 實戰一：加入新硬碟建立 `/data` 儲存空間（前半）

情境：機器新加了一顆 `4G` 硬碟 `/dev/sdb`，希望切出獨立於 rootfs 的 VG 掛載給 `/data`：（`p.15-17`）

1. 先用 **`lsblk`** 確認新硬碟名稱（看到 `sdb 4G disk`）（`p.16`）
2. 依 **Bottom → Top** 順序建立 PV → VG → LV：（`p.17`）
```sh
pvcreate /dev/sdb
vgcreate data-vg /dev/sdb
lvcreate -l 100%FREE -n data_vol data-vg
```

> 若未來還要替 `/data` 增加更多硬碟，只需對新硬碟 `pvcreate` 後再執行 **`vgextend`** 與 **`lvextend`** 即可（`p.17`）。

---

## 實戰一（續）：格式化、掛載與設定 `/etc/fstab`

有了 LV 裝置 `/dev/data-vg/data_vol` 後，就能像一般分割區一樣格式化與掛載：（`p.18`）

```sh
mkfs.btrfs /dev/data-vg/data_vol
mkdir /data
mount /dev/data-vg/data_vol /data
df -h
```

- 執行 `df -h` 可看到 `/dev/mapper/data--vg-data_vol`（`4.0G`）已成功掛載到 `/data`（`p.18`）
- **重要收尾**：記得修改 **`/etc/fstab`**，確保下次重開機後 `/data` 依然會自動掛載（`p.18`）

---

## 實戰二：線上擴充 `rootfs`（`/`）前半——掃描與 `fdisk` 分割

情境：調整 VM 硬碟容量（`48G`）後，替原本只用到 `24G` 的 `/dev/sda` 擴大 `/`：（`p.19-23`）

1. **通知核心重讀磁碟大小**（若新容量尚未顯示）：（`p.19`）
   ```sh
   echo 1 > /sys/block/<disk>/device/rescan
   ```
2. **進入 `fdisk /dev/sda` 操作步驟（`p.21-23`）**：
   - **`p`**：列印目前 GPT 分割表
   - **`n`**：將未配置空間建為新分割區 `/dev/sda4`（`22.1 GiB`）
   - **`t`**：將 `/dev/sda4` 類型改為 GPT 代碼 **`44`（`Linux LVM`）**
   - **`p`**：寫入前最後核對類型已為 `Linux LVM`
   - **`w`**：**寫入分割表並同步磁碟（Most important!）**

---

## 實戰二（續）：擴大 PV → VG → LV 與檔案系統（`rootfs` 變大！）

完成 `/dev/sda4` 分割區後，由下而上依序放大每一層：（`p.24-26`）

```sh
pvcreate /dev/sda4                     # 建立 PV (pvs 確認)
vgextend trixws-vg /dev/sda4           # 擴大 VG (24.00g -> 46.09g)
lvextend /dev/trixws-vg/root /dev/sda4 # 擴大 LV (22.77g -> 44.86g)
resize2fs /dev/trixws-vg/root          # 線上擴展已掛載的 / 檔案系統
df -h                                  # 驗證 / 從 23G 變為 44G
```

<blockquote class="trap">
<strong>實務小技巧（<code>p.25</code>）：</strong>直接執行 <code>lvextend --resizefs /dev/trixws-vg/root /dev/sda4</code> 就能自動一併擴大檔案系統，省下後面單獨下 <code>resize2fs</code> 的步驟！
</blockquote>

---

## 實戰三：縮減並移除磁碟（前提檢查與縮小檔案系統）

情境：`/data`（Btrfs）建立在橫跨兩個 PV（`sda1` `2G` + `sda2` `2G`）的 LV 上，要把 **`sda2`** 拔除：（`p.27-29`）

1. **絕對先決檢查（`p.28`）**：用 `df -h` 確認**目前已用空間小於留下來的 PV 容量**（本例僅用 `5.8M` ＜ `2G`）
2. **由上而下第一步：先縮小檔案系統！（`p.27, 29`）**
   ```sh
   btrfs filesystem resize -2g /data/
   df -h    # 確認 /data 已從 4.0G 縮成 2.0G
   ```
   （課程總覽步驟中，若目標為對應檔案系統亦會提及使用 `resize2fs` 縮容，`p.27`）

---

## 實戰三（續）：由上而下拆除 LV → VG → PV

檔案系統縮好後，才能逐層往下收回空間：（`p.27, 30`）

```sh
lvreduce --size -2g /dev/data-vg/data-vol   # 1. 縮小 LV (1022 -> 510 extents)
vgreduce data-vg /dev/sda2                  # 2. 把 sda2 從 VG 移出
pvremove /dev/sda2                          # 3. 清除 sda2 上的 LVM 標籤
```
（最後再安全從機器退出該硬碟，`p.27`）

- 因前一步已完成 `btrfs filesystem resize -2g`，`lvreduce` 會偵測 `File system reduce is not needed, skipping.` 並安心完成縮小（`p.30`）

---

## 總結：LVM 核心指令與方向順序速查表

| 層級 | 由下而上：建立 / 擴充 | 由上而下：縮容 / 移除 | 重要考點提醒 |
| :--- | :--- | :--- | :--- |
| 分割區 | `fdisk` (`n` → `t 44` → `p` → `w`) | 最後才退出磁碟 | 調整 VM 磁碟後可寫入 `.../device/rescan`（`p.19, 23`） |
| **PV** | `pvcreate` / `pvs` / `pvdisplay` | `pvremove` | PE 預設 `4MB`；PV 大小為 PE 整數倍（`p.4, 7, 30`） |
| **VG** | `vgcreate` / `vgextend` / `vgs` | `vgreduce` | 彙整一到多顆 PV 成儲存池（`p.5, 24, 30`） |
| **LV** | `lvcreate -l/-L` / `lvextend --resizefs` | `lvreduce` / `lvresize` | 兩條路徑 `/dev/<VG>/<LV>` 與 `/dev/mapper/...`（`p.6, 25, 30`） |
| **進階** | Thin Pool (`metadata`+`data`) / Snapshot | `umount` → `lvconvert --merge` | 快照常設 **20–30%**，且 **Not backup!**（`p.8-11`） |

[閱讀完整 Wiki 講義](../lvm/)

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
