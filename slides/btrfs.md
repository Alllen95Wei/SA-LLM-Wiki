---
marp: true
theme: sa-wiki
paginate: true
size: 16:9
header: "System Administration (Debian 13) — 教學型簡報"
footer: "知識來源：wiki/btrfs.md（BTRFS.pdf） ｜ System Administration LLM Wiki"
---

<!-- _class: lead -->
<!-- _paginate: skip -->

# Btrfs 入門教學：Subvolume、佈局、快照與遷移實戰

給初學者的現代化 CoW（Copy-on-Write）檔案系統指南

- **核心定位**：針對 Linux 設計的現代化 CoW 檔案系統，聚焦進階功能、容錯、修復與易管理性（`BTRFS.pdf, p.2`）
- **最大亮點**：不用預切固定空間的子捲宗（Subvolume）與近乎零負擔的快照機制（`BTRFS.pdf, p.4, 8-9`）
- [閱讀完整 Wiki 講義](../btrfs/)

---

## 學習地圖：這份簡報會帶你學會什麼？

1. **為什麼大家改用 Btrfs？** — 對照 `ext4` 與 `ZFS`，認識 Btrfs 五大核心特性（`p.2-3`）
2. **什麼是 Subvolume？Nested 和 Flat 佈局差在哪？** — 理解 ID `5` Top-level、`@` 命名與掛載選項繼承差異（`p.4-7`）
3. **Btrfs 快照跟 LVM 快照有何不同？怎麼還原？** — 非遞迴快照特性、零預配空間與「無內建 rollback」應對方法（`p.8-10`）
4. **怎麼把現有 `/home` 從 `ext4` 遷移到 Btrfs 並拍快照？** — Debian 13 實機 7 步驟拆解（`p.11-17`）

---

<!-- _class: divider -->

# 單元一：認識 Btrfs 與五大現代特性

從熟悉的 `ext4` 出發，看看現代 CoW 檔案系統多出了哪些關鍵能力。

---

## 為什麼選擇 Btrfs？比較 `ext4`、`Btrfs` 與 `ZFS`

選型時常見的三種檔案系統定位：（`p.2-3`）

| 檔案系統 | 角色與特性取捨（`BTRFS.pdf, p.2-3`） |
| :--- | :--- |
| **`ext4`** | 最常見的傳統檔案系統，提供基礎檔案系統功能 |
| **`Btrfs`** | 現代 CoW 檔案系統，具備**快照（snapshots）**等進階功能，同時兼顧容錯、修復與易管理 |
| **`ZFS`** | 同樣擁有豐富進階功能，但**記憶體佔用相對較高（high memory）** |

---

## Btrfs 五大核心特性與實用指令

1. **Snapshots**：拍快照**不會對檔案做完整複製**（`p.2`）
2. **Built-in volume management**：內建捲宗管理，支援軟體 **RAID 0、RAID 1** 等（`p.2`）
3. **Self-healing**：針對資料與中繼資料紀錄 checksum，自動偵測靜默資料毀損（`p.2`）
4. **Transparent data compression**：掛載時直接開啟透明壓縮：
   ```sh
   mount -o compress=zstd /dev/sdx /mnt
   ```
5. **Reflinks**：快速高效率的檔案複製：
   ```sh
   cp --reflink=always source target
   ```

---

<!-- _class: divider -->

# 單元二：Subvolume 與 Nested / Flat 佈局

Subvolume 不是硬性切死的磁碟分割區，而是擁有獨立目錄樹與 inode 空間的彈性單位。

---

## 什麼是 Subvolume？認識 ID `5` 與 `@` 命名慣例

- **Subvolume 定義**：Btrfs 檔案系統的一部分，但擁有**自己獨立的檔案／目錄階層與 inode 編號命名空間**（`p.4`）
- **Top-level Subvolume（ID `5`）**：（`p.4`）
  - 每個 Btrfs 檔案系統都有一個 **ID 為 `5`** 的頂層子捲宗，路徑是檔案系統內的 `/`
  - 其他 subvolume 都位於 top-level 之下
  - 沒有特別指定時，掛載 Btrfs **預設掛載的就是 top-level**
- **常見命名慣例**：名稱前面加上 **`@`**，例如 `@` 代表 `/`、`@www` 代表 `/var/www`（`p.4`）

---

## Subvolume 兩大佈局：Nested vs. Flat 結構圖

同樣幾個 subvolume，擺放位置不同就會決定掛載與刪除規則：（`p.5-6`）

<pre class="mermaid">
flowchart TB
    subgraph NESTED["Nested Layout (巢狀佈局，p.5)"]
        NT["toplevel (未直接掛載)"] --> NA["@ (/)"] --> NH["@home"] & NV["@var"]
        NV --> NW["@www"]
        NV --> NL["lib (目錄)"] --> NP["@postgresql"]
    end
    subgraph FLAT["Flat Layout (扁平佈局，p.6)"]
        FT["toplevel (預設不掛載)"] --> FA["@ -> /"] & FH["@home -> /home"] & FVAR["var (目錄)"]
        FVAR --> FWW["@www -> /var/www"] & FPG["@postgres -> /var/lib/postgresql"]
    end
</pre>

---

## Nested Layout vs. Flat Layout 實務差異比較

| 比較項目 | Nested Layout（`p.5`） | Flat Layout（`p.6`） |
| :--- | :--- | :--- |
| **建立位置** | 建立在**另一個 subvolume 裡面** | **全部直接是 top-level 的子項目** |
| **需要逐一手動掛載嗎？** | **不需要**（掛載父層時自動一併掛載） | **全部都需要手動（或 fstab）逐一掛載** |
| **掛載選項差異** | 子層自動**繼承父 subvolume 的掛載選項** | **每個掛載點可個別套用不同掛載選項** |
| **刪除時要注意什麼？** | **刪父 subvolume 前，必須先刪掉所有子 subvolume** | 彼此無巢狀父子 subvolume 束縛 |

---

## Flat Layout 在 `/etc/fstab` 中的個別選項實例

正因為 Flat layout 可以分開掛載，就能針對不同用途套用不同選項（例如 `autodefrag` vs. `noatime`）：（`p.7`）

```text
UUID=the-btrfs-fs-uuid  /                    btrfs  subvol=@,defaults,autodefrag        0 0
UUID=the-btrfs-fs-uuid  /home                btrfs  subvol=@home,defaults,noatime       0 0
UUID=the-btrfs-fs-uuid  /var/www             btrfs  subvol=/var/@www,noatime            0 0
UUID=the-btrfs-fs-uuid  /var/lib/postgresql  btrfs  subvol=/var/@postgres,noatime       0 0
```

- 關鍵參數就是 **`subvol=...`**，用來明確告訴系統這一行掛載點要掛入哪一個 subvolume（`p.7`）

---

<!-- _class: divider -->

# 單元三：CoW 快照、LVM 對比與還原技巧

Btrfs 快照其實也是一種 Subvolume——理解這一點，就會看懂它的建立與還原邏輯。

---

## Btrfs Snapshot 三大特性與建立指令

**Snapshot 本質上就是一個 subvolume**，透過 CoW 與來源 subvolume 共用資料與中繼資料：（`p.8`）

```sh
btrfs subvolume snapshot [-r] <source subvolume> <destination subvolume>
```

1. **不具遞迴性（Not recursive）**：來源裡面若還有其他子 subvolume，拍快照**不會**連它們一起拍
2. **`-r` 參數**：建立**唯讀（read-only）**快照
3. **可寫快照與原 subvolume 無差別**：不加 `-r` 時，快照本身就等同一個可讀寫的一般 subvolume

---

## LVM Snapshot vs. Btrfs Snapshot 關鍵對比

同樣都是 **Copy-on-Write（CoW）**，但實作層級與表現不同：（`p.9`）

<pre class="mermaid">
flowchart LR
    C["共同點: Copy-on-Write (CoW)"] --> LV["LVM Snapshot (p.9)"]
    C --> BT["Btrfs Snapshot (p.9)"]
    LV --> LV1["建立時須預先指定 LV 空間"] --> LV2["同 LV 多快照影響寫入速度並略浪費空間"]
    BT --> BT1["不需預先指定快照空間"] --> BT2["多份快照 (Reflink) 不會浪費效能與空間"]
</pre>

---

## 沒有內建 Rollback 指令怎麼還原？

**重要事實**：**Btrfs 沒有內建的 `rollback` 指令**（官方文件也未規定單一固定流程），常見兩種做法：（`p.10`）

<pre class="mermaid">
flowchart LR
    A["1. btrfs subvolume list -t /mnt 確認快照路徑"] --> B["2. 從快照再拍一份可寫新 subvolume (@ray_new)"]
    B --> C["3. mount -o subvol=/home/@ray_new /dev/sdb /home/ray"]
</pre>

- **做法 A**：掛載快照 subvolume，直接把舊檔複製回目標資料夾
- **做法 B（上圖流程）**：`btrfs subvolume snapshot /mnt/snapshots/@homeray-1762502638 /mnt/home/@ray_new`，再用 `-o subvol=` 重新掛載切換過去（`p.10`）

---

<!-- _class: divider -->

# 單元四：實機演練——將 `/home` 遷移至 Btrfs 並拍快照

把 LVM 與 Btrfs 串起來：在 Debian 13 的 LV 上格式化 Btrfs、建立 `@ray` 並拍下第一張快照。

---

## `/home` 遷移 7 步驟全景與佈局設計

將原本 `ext4` 的家目錄搬到 Btrfs，規劃的架構會把 `@ray` 與 `@homeUSER` 快照都放在 top-level 之下：（`p.12`）

<pre class="mermaid">
flowchart LR
    S1["1. apt install btrfs-progs"] --> S2["2. 準備 LV (lvs/lsblk)"]
    S2 --> S3["3. mkfs.btrfs 與 filesystem show"]
    S3 --> S4["4. 暫掛 /mnt 並 subvolume create"]
    S4 --> S5["5. cp -a 複製原資料"]
    S5 --> S6["6. -o subvol= 掛載並寫 fstab"]
    S6 --> S7["7. snapshot 並 list -t"]
</pre>

> 佈局提示：`toplevel` → `home/`（目錄）→ `@USER`（subvolume）；`snapshots/`（目錄）→ `@homeUSER`（快照 subvolume）（`p.12`）

---

## 步驟 1–3：安裝工具、確認 LV 與格式化 Btrfs

<div class="cols">
<div>

### 1. 安裝與確認裝置（`p.12-13`）
```sh
apt install btrfs-progs
lvs
lsblk
```
本例使用跨兩顆分割區的 LV `/dev/home-vg/home-vol`（`3.99g`）

</div>
<div>

### 2. 格式化並取得 UUID（`p.14`）
```sh
mkfs.btrfs /dev/home-vg/home-vol
btrfs filesystem show
```
`btrfs filesystem show` 會列出整體容量與 **`uuid`**（接下來寫入 `/etc/fstab` 必用）

</div>
</div>

---

## 步驟 4–5：暫掛 `/mnt`、建立 `@ray` 並複製家目錄

先透過 top-level 暫時掛載 `/mnt`，建立子捲宗並完整複製原本 `/home/ray` 的資料與權限：（`p.15`）

```sh
mount /dev/home-vg/home-vol /mnt
mkdir -p /mnt/home
btrfs subvolume create /mnt/home/@ray
cp -a /home/ray/. /mnt/home/@ray/
```

<blockquote class="trap">
<strong>細節提醒（<code>p.15</code>）：</strong><code>mkdir -p /mnt/home</code> 建的是一般目錄；真正的 Btrfs subvolume 是用 <code>btrfs subvolume create /mnt/home/@ray</code> 建出來的 <code>@ray</code>；複製家目錄時要用 <code>cp -a</code> 與尾端的 <code>/.</code>，才能把所有權限與隱藏設定檔一起完整帶過去！
</blockquote>

---

## 步驟 6–7：掛載 Subvolume、寫入 `/etc/fstab` 與拍快照

<div class="cols">
<div>

### 6. 切換掛載並寫入開機設定（`p.16`）
```sh
mount -o subvol=/home/@ray /dev/home-vg/home-vol /home/ray
df -h
echo "UUID=0ed693ae-28ac-44ce-8674-9059f685d8f1 /home/ray btrfs subvol=/home/@ray 0 0" >> /etc/fstab
```

</div>
<div>

### 7. 建立時間戳快照與驗證（`p.17`）
```sh
mkdir -p /mnt/snapshots
btrfs subvolume snapshot /mnt/home/@ray /mnt/snapshots/@homeray-$(date +%s)
btrfs subvolume list -t /mnt
```
用 `list -t` 可看到 `@ray`（ID `261`）與快照（ID `267`）`top level` 皆為 **`5`**

</div>
</div>

---

## 總結：Btrfs 核心觀念與指令速查表

| 主題 | 核心觀念或指令 | 必記重點 |
| :--- | :--- | :--- |
| 核心特色 | `mount -o compress=zstd` / `cp --reflink=always` | 支援快照、RAID 0/1 等、Self-healing、壓縮、Reflink（`p.2`） |
| Subvolume | ID `5` Top-level / `@` 前綴命名 | 擁有獨立目錄樹與 **inode 命名空間**；預設掛載 top-level（`p.4`） |
| Nested vs. Flat | 父子自動掛載 vs. `subvol=` 個別掛載 | Nested 繼承選項且必先刪子層；Flat 可分別給掛載選項（`p.5-7`） |
| Snapshot | `btrfs subvolume snapshot [-r] src dst` | **Not recursive**；**不需指定容量**；多快照不降寫入效能（`p.8-9`） |
| Rollback | **無內建指令**（拍可寫新 subvolume 重掛） | 與 LVM `lvconvert --merge` 不同（`p.10`） |

[閱讀完整 Wiki 講義](../btrfs/)

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
