---
marp: true
theme: sa-wiki
paginate: true
size: 16:9
header: "System Administration (Debian 13) — 教學型簡報"
footer: "知識來源：wiki/docker-basic.md（Docker_Basic.pdf） ｜ System Administration LLM Wiki"
---

<!-- _class: lead -->
<!-- _paginate: skip -->

# Docker 基礎入門：容器概念、安裝與常用指令

從零理解容器隔離、底層架構、Linux 安裝與容器生命週期操作

- **學習目標**：搞懂「容器和虛擬機哪裡不同」、「Docker 背後如何運作」，並能自己安裝及跑起第一個容器（`Docker_Basic.pdf, p.2-22`）
- **知識範圍**：完全依據課程講義整理，專注於 Linux 環境與常用核心操作（`Docker_Basic.pdf, p.1-22`）
- [閱讀完整 Wiki 講義](../docker-basic/)

---

## 學習地圖：這份簡報會帶你學會什麼？

1. **為什麼大家都在用容器？** — 從「容器與映像檔」出發，比較 Container 與傳統 VM 的差異，再揭開 Docker / `containerd` / `runc` 的底層分工（`p.3-9`）
2. **怎麼在 Linux 主機裝好 Docker？** — 使用官方安裝腳本、`systemd` 啟用服務，以及設定一般使用者免 `sudo` 操作（`p.10-12`）
3. **如何從零下載映像檔並操作容器？** — 看懂 Docker Workflow、`pull` / `run` / Volume 持久化儲存、`exec` / `logs` 除錯與 `ps` / `stop` / `restart` / `rm` 管理（`p.13-22`）

---

<!-- _class: divider -->

# 單元一：什麼是 Container 與 Docker？

先搞懂「為什麼需要容器」，再認識 Docker 與虛擬機器（VM）的本質差異。

---

## 先分清楚：Container、Image 與 Linux Namespaces

<div class="cols">
<div>

### 兩個最常被混用的名詞（`p.4`）
- **Container（容器）**：一個擁有**隔離資源（isolated resources）**的子系統
- **Container image（容器映像檔）**：一個**輕量、獨立的套件**，把執行軟體所需的一切都打包在裡面

> 簡單比喻：Image 像是靜態打包好的「軟體套件包」，啟動之後在隔離資源中跑起來的才是 Container。

</div>
<div>

### Linux 是如何做到「隔離」的？（`p.4`）
在 Linux 上，容器透過 **Linux namespaces** 實現資源隔離，涵蓋：
- **行程（processes）**
- **網路堆疊（network stacks）**
- **檔案系統（file systems）**

> *註：投影片於此處未展開其他資源配額機制，理解重點在於這三類資源隔離。*

</div>
</div>

---

## Docker 是什麼？除了 Docker 還有其他選擇嗎？

### Docker 的核心角色（`p.5`）
- 一個讓開發者把**應用程式及其相依套件（dependencies）**打包成輕量、可攜（portable）容器的平台
- 目標是讓軟體能夠在**不同環境中一致地執行（run consistently across different environments）**
- 是目前**最廣泛使用的容器引擎（container engine）**

### 其他容器執行時期（Container Runtimes）（`p.5`）
- **`Podman`**
- **`Singularity`**（以及生態系中其他 runtime）

---

## 圖解對照：Container vs. VM（架構差異）

為什麼常說容器比虛擬機器輕巧？關鍵在於底層是否各自帶一整套 Guest OS：（`p.6`）

<pre class="mermaid">
flowchart TB
    subgraph VM["Machine Virtualization (VM)"]
        direction BT
        V_INF["Infrastructure"] --> V_HYP["Hypervisor"]
        V_HYP --> G1["Guest OS"] --> B1["Bins/Lib"] --> A1["App 1"]
        V_HYP --> G2["Guest OS"] --> B2["Bins/Lib"] --> A2["App 2"]
        V_HYP --> G3["Guest OS"] --> B3["Bins/Lib"] --> A3["App 3"]
    end
    subgraph CT["Containers"]
        direction BT
        C_INF["Infrastructure"] --> C_OS["Operating System (共用核心)"] --> C_ENG["Container Engine"]
        C_ENG --> CB1["Bins/Lib"] --> CA1["App 1"]
        C_ENG --> CB2["Bins/Lib"] --> CA2["App 2"]
        C_ENG --> CB3["Bins/Lib"] --> CA3["App 3"]
    end
</pre>

---

## Container vs. VM：實務取捨怎麼看？

| 比較重點 | 虛擬機器（VM） | 容器（Container） |
| :--- | :--- | :--- |
| **環境獨立性（`p.7`）** | 具備**完全獨立的環境**（含完整 OS） | 與主機共用作業系統核心（kernel） |
| **速度與輕量度（`p.7`）** | 較重，每個 App 都要背負一份 Guest OS | **更快、更輕量**（因為共用核心） |
| **安全性（`p.7`）** | **較安全（safer）** | 隔離強度相對不及獨立 OS 之 VM |
| **可重用性（`p.7`）** | 相對較低 | **相對更容易重用（relatively reusable）** |

<blockquote class="trap">
<strong>初學常見誤解：</strong>容器並不是「比較迷你的 VM」——它<strong>沒有自己的獨立 OS 核心</strong>，而是各自帶上所需的 <code>Bins/Lib</code>，在同一個 <code>Operating System</code> 之上透過容器引擎與 Linux namespaces 隔離運作（<code>p.4, 6-7</code>）。
</blockquote>

---

## 揭開黑盒子：Docker、containerd 與 runc 如何分工？

其實 **Docker 本身是一個 CLI 工具**，真正從命令列到底層執行容器分為四層合作：（`p.8-9`）

1. **Docker CLI（透過 `dockerd`）**：接收使用者指令，去驅動名為 `containerd` 的執行時期（`p.8-9`）
2. **`containerd`**：管理容器的**生命週期（lifecycle）**（`p.8`）
3. **OCI 與 `shim`**：`containerd` 透過 **OCI（Open Container Initiative，容器開放規範）** 呼叫 `runc`，並用 **`shim`** 追蹤 `runc` 容器狀態（`p.8`）
4. **`runc`**：**低階容器執行時期（low-level container runtime）**，能夠**直接利用系統核心（kernel）**執行容器（`p.8`）

<blockquote class="trap">
<strong>觀念釐清：</strong>直接與 Linux kernel 互動跑起容器的是低階 runtime（例如 <code>runc</code>），不是 <code>docker</code> CLI 本身（<code>p.8</code>）。
</blockquote>

---

## 多工具路線圖：Docker 與 Podman 等有何不同？

不只 `docker`，其他命令列工具（如 `nerdctl`、`ctr`、`podman`）也能運作於不同架構上：（`p.9`）

<pre class="mermaid">
flowchart TB
    subgraph CLIs["CLIs (命令列)"]
        N["$ nerdctl run"]
        D["$ docker run"]
        C["$ ctr run"]
        P["$ podman run"]
    end
    subgraph Daemons["Daemons (背景服務)"]
        DD["dockerd"]
        CD["containerd"]
        ND["No daemon(s)!"]
    end
    subgraph Containers["Containers (低階 runtime)"]
        R1["runc: nginx / app1 / mysql"]
        CR["crun: app 2"]
    end
    D --> DD --- CD
    N --> CD
    C --> CD
    CD -->|"OCI"| R1
    P --> ND --> CR
</pre>

---

<!-- _class: divider -->

# 單元二：在 Linux 上安裝並啟動 Docker

只需要三個步驟：下載執行官方腳本、啟動 `systemd` 服務，並設定使用者群組權限。

---

## 安裝 Docker 並透過 `systemd` 啟動服務

Docker 官方提供適用多種 Linux 發行版的安裝腳本，完成後再由 `systemd` 啟用服務：（`p.11`）

<div class="cols">
<div>

### 1. 下載並執行官方安裝腳本
```sh
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh ./get-docker.sh
```
先存成 `get-docker.sh` 再以 `sudo sh` 執行

</div>
<div>

### 2. 設定開機啟動與立即啟動
```sh
sudo systemctl enable docker
sudo systemctl start docker
```
確保重開機後自動啟動，並立刻讓服務就緒

</div>
</div>

> **與先前的 Shell 知識相通（`wiki/shell.md`）**：安裝指令將遠端腳本先下載為本地檔案再執行，避免了直接把遠端內容透過管線送進 shell 的風險。

---

## 讓一般使用者免 `sudo` 操作 Docker

建立 `docker` 群組並將目前使用者 `$USER` 加入後，即可不必每次輸入 `sudo` 執行指令：（`p.12`）

```sh
sudo groupadd docker
sudo usermod -aG docker $USER
docker -v
```

- `usermod -aG docker $USER`：將目前使用者加入 `docker` 次要群組（`-aG`）
- `docker -v`：印出版本資訊，驗證指令可執行

<blockquote class="trap">
<strong>知識邊界提醒：</strong>本單元投影片僅列出以上三行設定指令，未展開「加入 <code>docker</code> 群組後何時生效」及「<code>docker</code> 群組的安全權限影響」，後續學到帳號與權限章節時務必留意系統安全原則（<code>p.12</code>）。
</blockquote>

---

<!-- _class: divider -->

# 單元三：從下載映像檔到操作第一個容器

認識 Docker 整體運作流程後，依序練習下載、啟動、儲存資料、進入容器與排查問題。

---

## 操作前先看懂全景：Docker Workflow

每當你在終端機輸入 `docker build`、`docker pull` 或 `docker run` 時，背後互動如下：（`p.14`）

<pre class="mermaid">
flowchart LR
    subgraph Client["Client (使用者端)"]
        B["docker build"]
        PL["docker pull"]
        RN["docker run"]
    end
    subgraph Host["DOCKER_HOST"]
        DM["Docker daemon"]
        IM["Images (本地映像檔)"]
        CT["Containers (運行容器)"]
        DM -->|"管理儲存"| IM
        IM -->|"建立實例"| CT
        DM -->|"控制運作"| CT
    end
    subgraph Reg["Registry (遠端倉庫)"]
        RI["Ubuntu / CentOS / Redis / NGINX"]
    end
    B --> DM
    PL --> DM
    RN --> DM
    DM <-->|"下載映像檔"| RI
</pre>

- `Client` 下指令給 `DOCKER_HOST` 上的 `Docker daemon`，由它向遠端 `Registry` 拉取 `Images` 並啟動管理 `Containers`（`p.14`）

---

## 第一步：用 `docker pull` 取得映像檔

啟動容器前必須先有映像檔；`docker pull` 會把遠端倉庫的映像檔下載到本地儲存空間：（`p.15`）

```sh
docker pull <image>:[<tag>]
```

- **從 Docker Hub 下載（公開映像檔倉庫）**：
  ```sh
  docker pull debian:trixie
  ```
- **從其他 Image Registry 下載（例如 AWS ECR）**：
  在映像檔名稱前加上前綴 `<registry_address>/<repository_name>`（`p.15`）

> *註：投影片雖在 Workflow 圖提及 `docker build`，但實際教學指令由 `docker pull` 下載現成映像檔開始（`p.14-15`）。*

---

## 看懂 Image Tag 與「多層結構」

<div class="cols">
<div>

### Tag 代表「版本」與「屬性」（`p.16`）
| 映像檔 / Tag | 意義說明 |
| :--- | :--- |
| `debian:trixie` | Debian Trixie 映像檔 |
| `debian:trixie-slim` | Debian Trixie，但**更輕量** |
| `python:3.13-bookworm` | Python 3.13（基底為 Debian Bookworm） |
| `python:3.13.5-alpine3.22` | Python 3.13.5（基底為 Alpine 3.22） |

</div>
<div>

### 映像檔是由「多層」堆疊而成（`p.17`）
- 一個 Container image 由**多個 layers（multiple layers）**組成
- 可在 **Image Layer Details** 頁面檢視每一層的設定與大小
- 例如 `python:3.13-bookworm` 截圖中可見第 4–11 層對應的 `ENV`（`0 B`）、`RUN ... apt-get ...`（`6.54 MB`）、`RUN ... wget ...`（`26.48 MB`）與 `CMD ["python3"]`

</div>
</div>

---

## 第二步：用 `docker run` 啟動容器與常用選項

取得映像檔後，執行 `docker run <image>` 就能啟動一個新容器：（`p.18`）

| 常用選項 | 功能說明 | 常見使用情境 |
| :--- | :--- | :--- |
| `-it` | Interactive + 開啟 TTY | 需要互動操作終端機時常搭配使用 |
| `-d` | Detach to background | 讓容器在背景持續運作 |
| `--name <name>` | 指定容器名稱 | 方便後續用名稱對它下達 `exec` / `logs` / `stop` |
| `--rm` | 容器停止時自動刪除 | 一次性測試跑完就清掉容器實例 |
| `-v <host>:<container>` | 掛載儲存捲（Volume mount） | 把資料持久化或與主機共享目錄 |
| `-p <host>:<container>` | 連接埠轉發（Port forwarding） | 將主機連接埠轉發到容器內部 |

<blockquote class="trap">
<strong>記憶口訣：</strong><code>-v</code> 與 <code>-p</code> 的冒號左右方向一致，永遠是 <strong><code>主機端（左）：容器端（右）</code></strong>（<code>p.18</code>）。
</blockquote>

---

## 容器刪掉資料會不見嗎？認識 Docker Volumes

**Docker volume** 讓容器能夠**獨立於自身生命週期**儲存與分享資料——即使容器重啟（restarts）或被刪除（deletions），資料依然保留（persistent）：（`p.19`）

<div class="cols">
<div>

### 1. Named Volumes（具名儲存捲）
- 幫 Volume 取一個名稱，由 Docker 集中存放在 **`/var/lib/docker/volumes`**
```sh
docker run -v my_vol:/data debian:trixie
```

</div>
<div>

### 2. Bind Mount（綁定掛載）
- 把**主機上的特定路徑**直接綁定進容器內部路徑
```sh
docker run -v ./data:/data debian:trixie
```

</div>
</div>

<blockquote class="trap">
<strong>仔細看 <code>-v</code> 冒號左邊！</strong>寫成純名稱 <code>my_vol</code> 會建立交由 Docker 管理的 Named Volume；寫成路徑格式（如 <code>./data</code>）則會把宿主機的實際目錄綁進容器（<code>p.19</code>）。
</blockquote>

---

## 進入運行中容器與排查問題：`exec` 和 `logs`

容器在運行中時，最常使用的兩個除錯與互動指令：（`p.20-21`）

<div class="cols">
<div>

### `docker exec`：進去下指令（`p.20`）
在**已經啟動並運行中的容器**內執行指定命令（最常見是 `bash` 或 `sh`）：
```sh
docker exec -it <container name> bash
```
- `-it`：開啟互動式 TTY，啟動 Shell 進入容器時必搭選項

</div>
<div>

### `docker logs`：看容器輸出日誌（`p.21`）
當容器運作發生問題時查看標準輸出與日誌紀錄：
```sh
docker logs <container name>
docker logs -f <container name>
```
- `-f`：顯示既有日誌並持續等待新日誌（follow）

</div>
</div>

---

## 日常生命週期管理與操作全流程

從下載映像檔、啟動除錯，到停止與清理容器的完整生命週期指令：（`p.15-22`）

<pre class="mermaid">
flowchart LR
    PL["1. docker pull image:tag"] --> RN["2. docker run image"]
    RN --> OPS["運行中: ps / exec -it / logs -f"]
    OPS --> ST["3. docker stop name"]
    ST --> PSA["docker ps -a (含已停止)"]
    ST --> RS["docker restart name (重啟)"]
    ST --> RM["docker rm name (移除)"]
</pre>

- `docker ps`：列出執行中的容器；加上 **`-a`** 可列出所有容器（包含**已停止**的容器）（`p.22`）
- `docker stop <name>` / `docker restart <name>` / `docker rm <name>`：分別用來停止、重啟與移除容器（`p.22`）

---

## 總結：Docker 初學者核心指令速查表

| 你想做的事 | 使用的指令 | 重要備忘 |
| :--- | :--- | :--- |
| 下載映像檔 | `docker pull debian:trixie` | `Tag` 決定版本與基底（如 `-slim`、`-alpine3.22`）（`p.15-16`） |
| 啟動容器 | `docker run [選項] <image>` | `-it`、`-d`、`--name`、`--rm`、`-v`、`-p`（`p.18`） |
| 保存資料 | `-v my_vol:/data` 或 `-v ./data:/data` | Named Volume 存於 `/var/lib/docker/volumes`；Bind Mount 綁宿主機路徑（`p.19`） |
| 進入容器 Shell | `docker exec -it <name> bash` | 對象必須是**運行中**容器；可改用 `sh`（`p.20`） |
| 追蹤問題日誌 | `docker logs -f <name>` | `-f` 持續等待新日誌（`p.21`） |
| 查看與清理容器 | `docker ps -a` / `stop` / `restart` / `rm` | `ps` 不加 `-a` 不會顯示已停止的容器（`p.22`） |

[閱讀完整 Wiki 講義](../docker-basic/)

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
