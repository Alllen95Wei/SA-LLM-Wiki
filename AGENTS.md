# AGENTS.md — System Administration LLM Wiki Schema

本文件是此知識庫的最高操作與架構規範（Schema）。所有在此專案執行的 LLM Agent 必須嚴格遵守本文件的三層架構、知識邊界、寫作格式、工作流程與 Git 版控規則。

---

## 1. 專案定位與三層架構

本專案依據 **LLM Wiki** 模式建立，目標是為使用者整理**系統管理課程（System Administration）**內容，產出以**期中／期末考複習**為核心導向的持久化知識庫，並透過 **Zensical** 編譯為一致模板與主題的靜態 HTML，部署於 **GitHub Pages**。

### 三層分工
1. **Raw Sources（`raw/`）**：使用者放置課程簡報 PDF 的原始資料層。**嚴格唯讀，由使用者自行管理，並透過 `.gitignore` 排除於 Git 版控之外。**
2. **The Wiki（`wiki/`）**：由 LLM 全權維護的扁平、細粒度原子 Markdown 知識庫，同時也是 `zensical.toml` 指定的 `docs_dir`。使用者閱讀與提問；LLM 負責撰寫、拆頁、交叉引用、繪製圖表與維護一致性。
3. **The Schema（`AGENTS.md` + `zensical.toml`）**：定義知識庫規範、頁面結構與網站建置設定。

---

## 2. 最高指導原則（不可違反的鐵則）

1. **`raw/` 嚴格唯讀且不納入版控**
   - 絕對不可建立、修改、覆寫、移動、重新命名或刪除 `raw/` 內的任何檔案。
   - 絕對不可使用 `git add -f raw/` 或任何方式將 `raw/` 納入 Git 追蹤。
2. **嚴格忠於來源（Zero External Supplementation）**
   - 所有技術事實、定義、指令、旗標（flags）、設定檔語法、預設路徑、限制條件與 Mermaid 節點，**必須嚴格源自 `raw/` 內的課程 PDF**。
   - 每個核心主張、範例或指令區塊皆須明確標註來源檔名與頁碼（格式：`(07_User_Management.pdf, p.18)`）。
   - 若投影片僅帶過名詞、未展開原理或參數細節，**嚴禁憑自身預訓練知識自行補完**；應以 `!!! note "投影片未展開"` 或明確文字標註「投影片未詳述」。
3. **聚焦於 Debian Linux（Debian 13 / trixie）**
   - 所有知識整理完全聚焦於 **Debian Linux**。
   - 略過投影片中僅供歷史或跨平台對照的 FreeBSD 專屬內容（例如 FreeBSD 的 `/etc/master.passwd`、`pw`、`wheel` 群組、UFS、jails、securelevel 等），不獨立建頁亦不收錄其操作配方。
4. **考試複習導向（期中／期末考）**
   - 內容編排以「好理解、易對比、抓得住考點」為最高準則：強調核心原理、專有名詞辨析、指令／機制的關鍵差異、操作驗證重點與常見易錯陷阱。
5. **Git 自動 Commit（但絕不 Push）**
   - 每次完成 **Ingest**，以及經使用者同意後完成的 **Query 回存** 或 **Lint 修復**，由 AI 自動執行 `git add`（僅限 `wiki/` 及相關設定檔，嚴禁納入 `raw/` 或 `site/`）並 `git commit`。
   - **絕對禁止自動執行 `git push`**；遠端推送一律由使用者自行決定與執行。

---

## 3. 目錄結構、語言與細粒度原子命名規範

### 3.1 專案目錄結構
```text
TAICA_LLM-Wiki/
├── AGENTS.md                        # 本規範文件（Schema）
├── .gitignore                       # 排除 raw/、site/、.cache/ 等
├── zensical.toml                    # Zensical 網站、主題、Mermaid 與 Tags 設定
├── .github/workflows/docs.yml       # GitHub Actions 自動建置並部署至 GitHub Pages
├── raw/                             # （嚴格唯讀、Git 排除）課程簡報 PDF
└── wiki/                            # （Zensical docs_dir）扁平原子知識庫
    ├── index.md                     # 總目錄、來源追蹤表與網站首頁
    ├── log.md                       # Append-only 演進日誌
    ├── tags.md                      # Zensical 自動標籤索引頁
    ├── src-<slug>.md                # 各份課程簡報的總覽摘要頁
    └── <slug>.md                    # 細粒度原子筆記（概念、指令、設定檔、比較）
```

### 3.2 語言與專有名詞慣例
- **內文語言**：一律使用**繁體中文**撰寫。
- **專有名詞格式**：首次出現關鍵專有名詞時，採「繁體中文（English Term, 縮寫）」格式（例：`寫入時複製（Copy-on-Write, CoW）`、`邏輯捲宗管理（Logical Volume Management, LVM）`）。
- **技術原貌保留**：Linux 指令、參數旗標、檔案與目錄路徑、設定檔欄位名稱與指令輸出範例，一律維持英文原貌並使用行內程式碼 `` `...` `` 或程式碼區塊標示。

### 3.3 扁平結構與細粒度原子命名（Slug 規範）
- **扁平組織**：不在 `wiki/` 下建立主題子資料夾，所有知識頁面平鋪於 `wiki/` 內，依賴標準相對連結、`wiki/index.md` 與 YAML frontmatter `tags` 組織關聯。
- **檔名規範**：一律使用**英文小寫 `kebab-case.md`**，避免 GitHub Pages URL 出現中文編碼。
  - 課程簡報總覽頁：`src-<簡報主題>.md`（例：`src-03-shell.md`、`src-07-user-management.md`、`src-btrfs.md`）。
  - 原子概念頁：`<概念>.md`（例：`btrfs-subvolume.md`、`nss-and-pam.md`、`linux-uid-gid.md`）。
  - 工具／指令頁：`<指令或主題>.md`（例：`useradd-vs-adduser.md`、`sudoers-policy.md`）。
  - 設定檔頁：`etc-<檔名>.md`（例：`etc-passwd.md`、`etc-shadow.md`、`etc-fstab.md`）。
  - 跨章比較／綜合頁：`<a-vs-b>.md` 或 `<主題>-synthesis.md`（例：`lvm-vs-btrfs-snapshot.md`）。
- **原子切分原則**：每頁聚焦單一明確知識點（一個機制、一個核心設定檔、一組相關指令或一個明確比較），避免將整份 30～50 頁的 PDF 塞進單一長頁；同時鼓勵頁面之間建立密集的雙向交叉引用。
- **連結語法**：一律使用標準 Markdown 相對連結 `[顯示文字](target-slug.md)`（或 `[顯示文字](target-slug.md#章節錨點)`）。**嚴禁使用 `[[Wikilink]]`**，以確保 Zensical 正常解析與超連結跳轉。

---

## 4. 頁面模板與 Zensical 語法規範

### 4.1 YAML Frontmatter（所有一般知識頁必填）
每份 `src-*.md` 與原子筆記 `<slug>.md` 開頭必須包含以下 YAML frontmatter：

```yaml
---
title: "頁面繁體中文標題"
category: concept        # 僅限: source | concept | command | config | comparison | synthesis
tags:
  - user-management      # 主題標籤（見下方建議清單）
  - concept              # 類型標籤
  - exam-trap            # 若包含高風險易錯陷阱或關鍵考點時加入
sources:
  - "07_User_Management.pdf (p.6-13)"
updated: 2026-11-23
---
```

**標準化標籤建議清單**（以維持 `tags.md` 整潔一致）：
- **主題標籤**：`shell`、`shell-programming`、`boot-shutdown`、`user-management`、`filesystem`、`lvm`、`btrfs`、`networking`、`docker`
- **類型標籤**：`source-summary`、`concept`、`command`、`config`、`comparison`、`synthesis`
- **考試輔助標籤**：`exam-trap`

### 4.2 原子筆記標準結構（`<slug>.md`）
```markdown
---
title: "<繁體中文標題>"
category: concept
tags:
  - <topic-tag>
  - concept
sources:
  - "<PDF檔名> (p.X-Y)"
updated: YYYY-MM-DD
---

# <繁體中文標題>

> 一句話考試定位摘要：說明此概念／指令／設定檔的核心作用。（`檔名.pdf, p.X`）

## 核心定義與原理

- 依投影片內容拆解核心機制、欄位意義或運作規則。（`檔名.pdf, p.X`）

## 流程與架構圖

（當內容涉及流程步驟、身分／權限分層、階層架構或決策分支時，務必繪製 Mermaid 圖表；純靜態定義頁可省略此節。）

## 關鍵指令、設定與細節

（收錄投影片中出現的 Debian 13 指令、設定範例與驗證指令，並附說明與頁碼。）

## 考點與常見陷阱

!!! tip "考點"
    濃縮本主題最核心的對比、順序或必須記住的驗證步驟。（`檔名.pdf, p.X`）

!!! warning "常見陷阱"
    標出投影片特別強調的誤用情境、安全風險或容易混淆的限制。（`檔名.pdf, p.X`）

!!! note "投影片未展開"
    僅在投影片提及名詞但未解釋細節時列出，禁止自行補充外部資料。（`檔名.pdf, p.X`）

## 相關頁面

- 所屬簡報總覽：[簡報標題](src-xxx.md)
- 相關概念／指令：[關聯頁面標題](related-slug.md)

## 資料來源

- `檔名.pdf`：p.X、p.Y–Z
```

### 4.3 簡報總覽頁模板（`src-<slug>.md`）
- `category: source`、`tags` 包含對應主題標籤與 `source-summary`。
- 包含：
  1. **來源資訊與學習目標**（對應投影片 "What You Will Learn" 等頁碼）
  2. **主題架構與知識地圖**（可視需要附整份簡報的 Mermaid 概念關聯圖）
  3. **核心考點導覽與原子頁面索引**（以清單或表格連結至本次產出的所有 `<slug>.md`）
  4. **範圍過濾與備註**（列出略過的 FreeBSD 對照頁碼與「投影片未展開」項目）

### 4.4 Mermaid 圖表撰寫規範
- 使用標準 Markdown fenced code block：` ```mermaid ` ... ` ``` `。
- 適用情境：開機／關機流程、PAM/NSS 分層流程、LVM PV/VG/LV 階層、Btrfs nested/flat subvolume 結構、指令決策樹、網路封包／容器架構等。
- **語法安全**：只要節點文字內含括號 `()`、方括號 `[]`、斜線 `/`、冒號 `:`、箭頭或空白，**務必使用雙引號包裹節點標籤**，例如：`A["/etc/passwd (7 欄位)"] --> B["getent passwd"]`。
- **忠實原則**：Mermaid 中的每個節點名稱、步驟順序與關係線，都必須能在來源 PDF 找到對應依據，不得為了美化而虛構投影片沒教的流程節點。

---

## 5. 索引（`wiki/index.md`）與日誌（`wiki/log.md`）維護規範

### 5.1 `wiki/index.md`
- 作為全站的**導覽目錄暨網站首頁**，分為：
  1. **課程來源追蹤表（Sources Status）**：追蹤每份 `raw/*.pdf` 的匯入狀態（`待匯入` / `已匯入`）、總覽頁連結與備註。
  2. **分類目錄區**：`Sources`、`Concepts`、`Commands`、`Configs`、`Comparisons & Syntheses`。
- 每個條目皆須附帶連結與「一句話考點摘要」，讓未來 Query 時能先閱讀 `index.md` 快速鎖定需要開啟的原子頁面。

### 5.2 `wiki/log.md`
- **只增不改（Append-only）** 的時間軸紀錄。
- 每筆紀錄標題必須嚴格遵循以下格式（方便以 CLI 工具檢索）：
  - `## [YYYY-MM-DD] ingest | <PDF 檔名>`
  - `## [YYYY-MM-DD] query | <回存頁面標題>`
  - `## [YYYY-MM-DD] lint | <檢查範圍或摘要>`
- 紀錄內容需簡潔列出：處理的來源、新增頁面清單、更新頁面清單，以及略過或待釐清事項。

---

## 6. 三大操作流程（Workflows）

### 6.1 Ingest（逐份互動式匯入）
每次使用者要求匯入某份 `raw/` PDF 時，必須嚴格遵守以下步驟：
1. **一次一份**：每次 Ingest 僅處理 **1 份**指定的 PDF，不擅自一次批次寫入多份尚未討論的檔案。
2. **完整閱讀與視覺檢查**：透過 `read_file` 完整檢視該 PDF 所有頁面（包含頁面的文字 OCR 與投影片截圖／終端機畫面／架構圖），確保不遺漏程式碼方塊、圖表與註腳提示框。
3. **過濾與對齊檢查**：篩除 FreeBSD 專屬對照內容；標記投影片未展開但重要之名詞；確認所有內容均可回溯至具體頁碼。
4. **提出 Ingest 提案並暫停等待確認（關鍵步驟）**：
   在寫入任何 `wiki/` 檔案之前，先向使用者輸出清晰的提案報告：
   - **本份簡報核心考點摘要**（條列主要考試重點）
   - **預計新增／更新的原子頁面清單**（列出每個頁面的 `slug.md`、繁中標題、`category`、涵蓋頁碼與設計理由）
   - **Mermaid 圖表規劃**（說明預計在哪幾頁加入何種流程／架構圖）
   - **範圍邊界說明**（列出略過了哪些 FreeBSD 頁面、哪些細節因投影片未詳述將標記為「投影片未展開」）
   等待使用者確認或微調切分粒度後，才可進入寫入階段。
5. **寫入與雙向連結**：
   - 建立 `src-<slug>.md` 與所有經確認的原子筆記 `<slug>.md`。
   - 檢查既有 `wiki/` 頁面：若有概念重疊、前置依賴或可對比之處，主動為既有頁面補上連往新頁面的交叉連結與修訂更新（同時更新其 `updated` 與 `sources`）。
   - 更新 `wiki/index.md`（更新來源追蹤表狀態為 `已匯入` 並加入各分類連結）與 `wiki/log.md`。
6. **驗證**：
   - 確認無任何 `[[Wikilink]]`、無斷連、YAML frontmatter 完整、無引用外部未載資訊。
   - 若環境可用，執行 `uvx zensical build --clean` 驗證靜態網站能無錯建置（不將 `site/` 納入版控）。
7. **自動 Git Commit**：
   - 確認 `git status` 不包含 `raw/` 或 `site/`。
   - 執行 `git add wiki/`（若有設定檔連動則一併加入）與 `git commit -m "ingest: <PDF檔名>"`。
   - **不執行 `git push`**，並向使用者回報本次新增／更新檔案與 commit 結果。

### 6.2 Query（查詢與詢問後回存）
1. **檢索定位**：收到問題時，優先讀取 `wiki/index.md` 鎖定相關頁面，再精讀目標原子頁面（必要時回查 `raw/` 對應 PDF 頁碼驗證截圖細節）。
2. **嚴格依據來源作答**：回答必須以繁體中文清楚條列（可附臨時 Mermaid 圖或對照表），並標註依據的 Wiki 頁面連結 `[標題](slug.md)` 與原始 PDF 頁碼。若課程簡報與 Wiki 均未涵蓋該細節，明確說明「目前課程簡報未涵蓋此部分」。
3. **主動詢問回存**：當本次查詢產出了具備考前複習價值的新綜合結果（例如跨章比較表、整合流程圖、指令速查整理）時，於回答結尾**主動詢問使用者**：是否要將這份整理回存為 `wiki/` 中的新頁面？
4. **確認後回存與 Commit**：若使用者同意，依原子頁規範建立新頁（`category: comparison` 或 `synthesis`）、更新相關頁面的雙向連結、更新 `wiki/index.md` 與 `wiki/log.md`、驗證建置後自動執行 `git commit -m "query: <主題摘要>"`（不執行 `git push`）。

### 6.3 Lint（知識庫健康檢查）
當使用者要求執行 Lint，或累計匯入數份來源後建議執行檢查時，應全面檢查以下項目：
1. **連結完整性**：是否存在斷連、錯誤相對路徑或誤用 `[[Wikilink]]`。
2. **孤兒與索引同步**：是否存在未被 `wiki/index.md` 收錄的頁面，或沒有任何入站連結（inbound links）的孤兒頁。
3. **跨頁一致性**：不同投影片彙整出的觀念定義、指令選項、預設值或路徑之間是否有矛盾未標註。
4. **切分與缺漏檢查**：是否存在某個在多頁被高頻提及、但尚未獨立成頁的核心概念；或過於肥大可再拆分的頁面。
5. **知識邊界與格式合規**：檢查是否每頁都有具體 `檔名.pdf, p.X` 引用、有無偷渡外部知識、有無殘留 FreeBSD 操作內容、YAML frontmatter 與 Mermaid 特殊字元雙引號是否合規。
6. **建置驗證**：執行 `uvx zensical build --clean` 確認 Zensical 建置通過。
7. **回報、修復與紀錄**：向使用者呈報發現清單；完成修復後於 `wiki/log.md` 寫入 `lint` 紀錄並自動執行 `git commit -m "lint: <摘要>"`（不執行 `git push`）。

---

## 7. Zensical 建置與 GitHub Pages 部署備忘

- **本地建置檢查**：`uvx zensical build --clean`（輸出於 `.gitignore` 排除的 `site/`）。
- **本地預覽伺服器**：`uvx zensical serve`（預覽頁面排版、Tags 索引、搜尋與 Mermaid 亮／暗色渲染）。
- **首次發布至 GitHub Pages 檢查事項**：
  1. 於 `zensical.toml` 取消註解並填入實際的 `site_url = "https://<username>.github.io/<repo>/"`（設定後可於 `[project.theme]` 取消註解 `"navigation.instant"`）。
  2. 至 GitHub Repository **Settings → Pages → Build and deployment → Source** 選擇 **GitHub Actions**。
  3. 當使用者推送 `main` 分支時，`.github/workflows/docs.yml` 會自動建置並發布網站。
