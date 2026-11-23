# AGENTS.md — System Administration LLM Wiki Schema

本文件是此知識庫的最高操作與架構規範（Schema）。所有在此專案執行的 LLM Agent 必須嚴格遵守本文件的三層架構、知識邊界、主題式寫作與 Marp 簡報格式、側邊欄排序規則、工作流程與 Git 版控規則。

---

## 1. 專案定位與三層架構

本專案依據 **LLM Wiki** 模式建立，目標是為使用者整理**系統管理課程（System Administration）**內容，產出以**期中／期末考複習**為核心導向的持久化知識庫，並以該知識庫為來源產出適合初學者自學的**教學型 HTML 簡報**，最終透過 **Marp CLI + Zensical** 編譯為一致模板與主題的靜態 HTML，部署於 **GitHub Pages**。

### 三層分工
1. **Raw Sources（`raw/`）**：使用者放置課程簡報 PDF 的原始資料層。**嚴格唯讀，由使用者自行管理，並透過 `.gitignore` 排除於 Git 版控之外。**
2. **The Wiki & Slides（`wiki/` + `slides/`）**：由 LLM 全權維護的**主題式（Topic-Based）** Markdown 知識庫與對應的**教學型 Marp 簡報**。每個核心課程主題對應 1 篇結構連貫、考試導向的講義頁（`wiki/<topic-slug>.md`），並以該 Wiki 頁面作為知識來源製作 1 份讓不熟悉該主題者也能理解基本概念的教學簡報（`slides/<topic-slug>.md`，編譯輸出至被 Git 排除的 `wiki/slides/` 後由 Zensical 一併發佈）；LLM 負責撰寫、交叉引用、繪製圖表與維護一致性。
3. **The Schema（`AGENTS.md` + `zensical.toml` + `themes/sa-wiki.css`）**：定義知識庫規範、頁面與簡報模板、側邊欄分組導覽（`nav`）與網站／簡報建置設定。

---

## 2. 最高指導原則（不可違反的鐵則）

1. **`raw/` 嚴格唯讀且不納入版控**
   - 絕對不可建立、修改、覆寫、移動、重新命名或刪除 `raw/` 內的任何檔案。
   - 絕對不可使用 `git add -f raw/` 或任何方式將 `raw/` 納入 Git 追蹤。
2. **嚴格忠於來源（Zero External Supplementation）**
   - 所有技術事實、定義、指令、旗標（flags）、設定檔語法、預設路徑、限制條件與 Mermaid 節點，**必須嚴格源自 `raw/` 內的課程 PDF**（簡報 `slides/*.md` 則以對應之 `wiki/*.md` 為知識來源，同樣嚴禁引入外部未載知識）。
   - 每個核心主張、範例或指令區塊皆須明確標註來源檔名與頁碼（格式：`(07_User_Management.pdf, p.18)`）。
   - 若投影片僅帶過名詞、未展開原理或參數細節，**嚴禁憑自身預訓練知識自行補完**；應以 `!!! note "投影片未展開"` 或明確文字標註「投影片未詳述」。
3. **聚焦於 Debian Linux（Debian 13 / trixie）**
   - 所有知識整理與簡報完全聚焦於 **Debian Linux**。
   - 略過投影片中僅供歷史或跨平台對照的 FreeBSD 專屬內容（例如 FreeBSD 的 `/etc/master.passwd`、`pw`、`wheel` 群組、UFS、jails、securelevel 等），不獨立建頁亦不收錄其操作配方。
4. **考試複習導向（期中／期末考）**
   - 內容編排以「連貫好讀、抓得住考點、易於對比」為最高準則：強調核心原理、專有名詞辨析、指令／機制的關鍵差異、操作驗證重點與常見易錯陷阱。
5. **Git 自動 Commit（但絕不 Push）**
   - 每次完成 **Ingest**，以及經使用者同意後完成的 **Query 回存** 或 **Lint 修復**，由 AI 自動執行 `git add`（涵蓋 `wiki/`、`slides/`，以及因 `nav` 或主題異動而更新的 `zensical.toml`、`themes/`；嚴禁納入 `raw/`、`site/` 或編譯產生的 `wiki/slides/`）並 `git commit`。
   - **絕對禁止自動執行 `git push`**；遠端推送一律由使用者自行決定與執行。

---

## 3. 目錄結構、語言與主題式命名規範

### 3.1 專案目錄結構
```text
TAICA_LLM-Wiki/
├── AGENTS.md                        # 本規範文件（Schema）
├── .gitignore                       # 排除 raw/、site/、wiki/slides/、.cache/ 等
├── zensical.toml                    # Zensical 網站設定、分組 nav、Mermaid 與 Tags
├── .github/workflows/docs.yml       # GitHub Actions 依序編譯 Marp 與 Zensical 並部署
├── raw/                             # （嚴格唯讀、Git 排除）課程簡報 PDF
├── themes/
│   └── sa-wiki.css                  # Marp 全站統一簡報主題樣式
├── slides/                          # 教學型 Marp Markdown 簡報（以對應 Wiki 頁為來源）
│   └── <topic-slug>.md
└── wiki/                            # （Zensical docs_dir）扁平主題式知識庫
    ├── index.md                     # 總目錄、來源追蹤表與網站首頁
    ├── tags.md                      # Zensical 自動標籤索引頁
    ├── log.md                       # Append-only 演進日誌
    ├── slides/                      # （Git 排除）Marp CLI 編譯出的 HTML 簡報暫存目錄
    ├── <topic-slug>.md              # 主題講義頁（原則上一份課程主題對應 1 篇）
    └── <a-vs-b>.md                  # 跨章比較或綜合整理頁（視需要建立）
```

### 3.2 語言與專有名詞慣例
- **內文語言**：一律使用**繁體中文**撰寫。
- **專有名詞格式**：首次出現關鍵專有名詞時，採「繁體中文（English Term, 縮寫）」格式（例：`寫入時複製（Copy-on-Write, CoW）`、`邏輯捲宗管理（Logical Volume Management, LVM）`）。
- **技術原貌保留**：Linux 指令、參數旗標、檔案與目錄路徑、設定檔欄位名稱與指令輸出範例，一律維持英文原貌並使用行內程式碼 `` `...` `` 或程式碼區塊標示。

### 3.3 扁平結構、主題式切分與連結語法
- **扁平組織**：不在 `wiki/` 下建立主題子資料夾，所有頁面平鋪於 `wiki/` 內（僅保留編譯用且不進 Git 的 `wiki/slides/`），側邊欄階層由 `zensical.toml` 的 `nav` 統一控管。
- **檔名規範**：一律使用**英文小寫 `kebab-case.md`**，避免 GitHub Pages URL 出現中文編碼。
  - 系統固定頁（僅此 3 份，於 `nav` 獨立分區置放，絕不與主題筆記混排）：`index.md`、`tags.md`、`log.md`。
  - 主題講義頁（`wiki/<topic-slug>.md`）與對應簡報（`slides/<topic-slug>.md`）：例如 `shell.md`、`shell-programming.md`、`boot-shutdown.md`、`user-management.md`、`filesystem.md`、`lvm.md`、`btrfs.md`、`linux-networking.md`、`docker-basic.md`。
  - 跨章比較／綜合頁：`<a-vs-b>.md` 或 `<topic>-synthesis.md`（例：`storage-partition-lvm-btrfs.md`、`lvm-vs-btrfs-snapshot.md`）；可視需求選配對應教學簡報 `slides/<slug>.md`。
- **主題式整併原則（Option A）**：
  - **原則上每份核心課程簡報對應 1 篇完整主題講義頁與 1 份 Marp 簡報**，**不另建空泛的 `src-*.md` 總覽殼頁，也不將單一指令或小知識點過度拆碎成獨立檔案**。
  - 利用清晰的 `##` 與 `###` 標題層級組織各單元，確保透過 Zensical 右側目錄（TOC）即可流暢閱讀與跳轉複習。
  - 僅在跨多份簡報對比（如 LVM vs Btrfs 快照）或將有價值的 Query 整理回存時，才另立 `comparison` / `synthesis` 頁面。
- **連結語法**：
  - 一律使用標準 Markdown 相對連結 `[顯示文字](target.md)` 或章節錨點連結 `[顯示文字](target.md#章節標題)`。**嚴禁使用 `[[Wikilink]]`**。
  - 若在 Markdown **表格儲存格**內顯示管線符號 `|`（含行內程式碼），**必須寫作 HTML 標籤與字元實體** `<code>producer &#124; filter</code>`，避免表格切欄錯誤或渲染出多餘的反斜線 `\|`。

---

## 4. 頁面與簡報模板規範

### 4.1 YAML Frontmatter（所有 Wiki 知識頁必填）
每份主題頁或比較／綜合頁開頭必須包含以下 YAML frontmatter：

```yaml
---
title: "Shell：命令列、管線與文字處理"
category: topic          # 僅限: topic | comparison | synthesis
tags:
  - shell                # 主題標籤（見下方建議清單）
  - topic                # 類型標籤: topic | comparison | synthesis
  - exam-trap            # 若包含關鍵考點或高風險易錯陷阱時加入
sources:
  - "03_Shell.pdf (p.1-49)"
updated: 2026-11-23
---
```

**標準化標籤建議清單**（以維持 `tags.md` 整潔一致）：
- **主題標籤**：`shell`、`shell-programming`、`boot-shutdown`、`user-management`、`filesystem`、`lvm`、`btrfs`、`networking`、`docker`
- **類型標籤**：`topic`、`comparison`、`synthesis`
- **考試輔助標籤**：`exam-trap`

### 4.2 主題講義頁標準結構（`wiki/<topic-slug>.md`）
```markdown
---
title: "<主題繁體中文標題>"
category: topic
tags:
  - <topic-tag>
  - topic
  - exam-trap
sources:
  - "<PDF檔名> (p.1-N)"
updated: YYYY-MM-DD
---

# <主題繁體中文標題>

> 一句話考試定位摘要：總結本主題在系統管理中的核心任務與必考脈絡。（`檔名.pdf, p.1-2`）

!!! info "教學型 HTML 簡報"
    想從零開始透過翻頁簡報循序理解本單元基本概念，請點此：[開啟教學型 HTML 簡報（新分頁）](slides/<topic-slug>.html){ target="_blank" }。

## 學習目標與課程原則
...
```

### 4.3 跨章比較／綜合頁模板（`<a-vs-b>.md` / `<topic>-synthesis.md`）
- `category: comparison` 或 `synthesis`，`tags` 包含涉及的多個主題標籤與對應類型標籤。
- 包含：**比較定位摘要**、**核心差異對照表**、**流程／決策 Mermaid 圖**、**選型與考點陷阱（Admonitions）**、**來源主題頁錨點連結**與**資料來源**。

### 4.4 Mermaid 圖表撰寫規範
- **Wiki 頁面**：使用標準 Markdown fenced code block ` ```mermaid ` ... ` ``` `。
- **Marp 簡報**：使用 `<pre class="mermaid">...</pre>` 搭配文末延遲渲染腳本（見 §4.5）。
- **語法安全**：只要節點文字內含括號 `()`、方括號 `[]`、斜線 `/`、冒號 `:`、`$`、`|` 或空白，**務必使用雙引號包裹節點標籤**，且避免在雙引號標籤內再使用未跳脫的雙引號 `"`，例如：`A["/etc/passwd (7 欄位)"] --> B["getent passwd"]`。
- **忠實原則**：Mermaid 中的每個節點名稱、步驟順序與關係線，都必須能在來源 PDF 找到對應依據，不得為了美化而虛構投影片沒教的流程節點。

### 4.5 Marp HTML 簡報規範（`slides/<slug>.md`）
- **定位與知識來源**：以對應的 **`wiki/<slug>.md` 作為直接知識來源**，為**原本不熟悉該主題的初學者**製作**教學型 HTML 簡報**（與考試複習導向的 Wiki 主題頁互補）。每篇 **`category: topic` 主題講義必配** 1 份對應簡報；**`category: comparison` / `synthesis` 頁亦可視需求選配簡報**，全站皆採用統一主題 `themes/sa-wiki.css`。
- **Frontmatter 標準設定**：
  ```yaml
  ---
  marp: true
  theme: sa-wiki
  paginate: true
  size: 16:9
  header: "System Administration (Debian 13) — 教學型簡報"
  footer: "知識來源：wiki/<topic-slug>.md（<PDF檔名>） ｜ System Administration LLM Wiki"
  ---
  ```
- **內容編排原則（初學者教學導向）**：
  - **學習地圖與章節過場**：首頁封面（`<!-- _class: lead -->`）點明學習目標與 `[閱讀完整 Wiki 講義](../<topic-slug>/)`；第 2 頁提供「學習地圖」（用初學者問題串起全單元）；各主要學習階段以 `<!-- _class: divider -->` 章節過場頁區隔。
  - **由淺入深、一頁一觀念**：先說明「為什麼需要／解決什麼問題」，再帶出語法與實例，避免把多個工具塞進同一張高密度投影片而造成閱讀壓力或版面溢位。
  - **嚴格忠於來源**：雖為教學口吻與循序展開，所有技術事實、指令、旗標、範例與 Mermaid 節點仍須**嚴格源自對應 `wiki/<topic-slug>.md`（並標註 PDF 頁碼）**，不自行補充外部知識或 FreeBSD 專屬配方。
  - **視覺輔助元件**：使用 `<blockquote class="trap">...</blockquote>` 提醒初學者最常踩的盲點，使用 `<div class="cols">` 輔助雙欄對照（例如觀念說明配指令範例）。
- **Mermaid 渲染機制**：在投影片中以 `<pre class="mermaid">...</pre>` 嵌入圖表，並於簡報檔最底部附上 `mermaid@11` ESM 延遲渲染腳本（設定 `startOnLoad: false`，僅對目前可視投影片內未處理的 `.mermaid:not([data-processed="true"])` 節點執行 `mermaid.run`），確保 Bespoke 隱藏投影片不會出現尺寸異常。
- **雙向連結**：對應的 `wiki/<slug>.md` 頁首提示框與 `wiki/index.md` 皆需提供 `[開啟教學型 HTML 簡報（新分頁）](slides/<slug>.html){ target="_blank" }` 連結（比較／綜合頁亦同）。

---

## 5. 側邊欄導覽（`zensical.toml` `nav`）、索引（`wiki/index.md`）與日誌（`wiki/log.md`）維護規範

### 5.1 `zensical.toml` `nav` 分組排序規範
- **禁止依賴未定義 `nav` 的檔名字母自動排序**，以避免 `index.md`、`log.md`、`tags.md` 與知識主題頁混排。
- `[project]` 下的 `nav` 必須永遠維持以下固定分組順序，且 `[project.theme]` 啟用 `"navigation.sections"`：
  1. `首頁與總目錄`：`index.md`
  2. `課程主題筆記`：依**課程學習脈絡與主題順序**排列各 `<topic-slug>.md`（而非英文字母順序）。
  3. `跨章比較與綜合整理`：於實際建立比較或綜合頁後加入此分組（尚未有頁面時維持註解）。
  4. `知識庫維運`：固定置於側邊欄最底部，依序為 `tags.md`（標籤索引）與 `log.md`（演進日誌）。

### 5.2 `wiki/index.md`
- 作為全站的**導覽目錄暨網站首頁**，分為：
  1. **課程來源追蹤表（Sources Status）**：追蹤每份 `raw/*.pdf` 的匯入狀態（`待匯入` / `已匯入`）、對應主題頁連結、`HTML 簡報` 連結與備註。
  2. **課程主題筆記（Topics）**：依課程順序列出各主題頁連結、HTML 簡報連結、一句話考試重點與涵蓋頁碼。
  3. **跨章比較與綜合整理（Comparisons & Syntheses）**：列出所有跨章比較與 Query 回存頁。

### 5.3 `wiki/log.md`
- **只增不改（Append-only）** 的時間軸紀錄。
- 每筆紀錄標題必須嚴格遵循以下格式（方便以 CLI 工具檢索）：
  - `## [YYYY-MM-DD] init | <標題>`
  - `## [YYYY-MM-DD] ingest | <PDF 檔名>`
  - `## [YYYY-MM-DD] query | <回存頁面標題>`
  - `## [YYYY-MM-DD] lint | <檢查範圍或摘要>`
  - `## [YYYY-MM-DD] refactor | <重構範圍或摘要>`

---

## 6. 三大操作流程（Workflows）

### 6.1 Ingest（逐份互動式匯入）
每次使用者要求匯入某份 `raw/` PDF 時，必須嚴格遵守以下步驟：
1. **一次一份**：每次 Ingest 僅處理 **1 份**指定的 PDF，不擅自一次批次寫入多份尚未討論的檔案。
2. **完整閱讀與視覺檢查**：透過 `read_file` 完整檢視該 PDF 所有頁面（包含頁面的文字 OCR 與投影片截圖／終端機畫面／架構圖），確保不遺漏程式碼方塊、圖表與註腳提示框。
3. **過濾與對齊檢查**：篩除 FreeBSD 專屬對照內容；標記投影片未展開但重要之名詞；確認所有內容均可回溯至具體頁碼。
4. **提出 Ingest 提案並暫停等待確認（關鍵步驟）**：
   在寫入任何 `wiki/` 或 `slides/` 檔案之前，先向使用者輸出清晰的提案報告：
   - **預計建立／更新之主題頁與簡報頁**：列出目標 `wiki/<topic-slug>.md`、`slides/<topic-slug>.md`、繁中標題與在 `zensical.toml` `nav` 中的擺放順序。
   - **本份簡報核心考點與章節／教學簡報大綱**：條列 `##` / `###` 單元結構、主要考點，以及以該 Wiki 為來源規劃之教學簡報學習路線、章節過場與預估張數。
   - **Mermaid 圖表規劃**：說明預計在哪些章節與投影片插入何種流程／架構圖。
   - **跨主題連結或比較頁建議**：說明是否需更新既有主題頁的交叉連結，或建議獨立建立跨章比較頁（僅於真正跨章時提案）。
   - **範圍邊界說明**：列出略過了哪些 FreeBSD 頁面、哪些細節因投影片未詳述將標記為「投影片未展開」。
   等待使用者確認或微調大綱後，才可進入寫入階段。
5. **寫入與同步維護**：
   - 建立或更新目標主題頁 `wiki/<topic-slug>.md`（含頁首 `教學型 HTML 簡報` 連結），再以該主題頁為知識來源製作對應的教學型 Marp 簡報 `slides/<topic-slug>.md`（含學習地圖、單元過場與返回完整講義連結）。
   - 同步更新 **`zensical.toml` 的 `nav`**、**`wiki/index.md`**（來源追蹤表改為 `已匯入`、補上主題頁與 HTML 簡報連結）與 **`wiki/log.md`**。
6. **雙重編譯與驗證**：
   - 先編譯 Marp 簡報：`npx --yes @marp-team/marp-cli --input-dir slides --output wiki/slides --theme-set themes --html`。
   - 再編譯 Zensical 網站：`uvx zensical build --clean`。
   - 確認無 `[[Wikilink]]`、無斷連、無表格 `\|` 誤寫、YAML frontmatter 完整、無外部知識，且 `site/slides/<topic-slug>.html` 確實產出、連結正確。
7. **自動 Git Commit**：
   - 確認 `git status` 不包含 `raw/`、`site/` 或 `wiki/slides/`。
   - 執行 `git add wiki/ slides/ zensical.toml`（若有調整 `themes/` 一併加入）與 `git commit -m "ingest: <PDF檔名>"`。
   - **不執行 `git push`**，並向使用者回報本次新增／更新檔案與 commit 結果。

### 6.2 Query（查詢與詢問後回存）
1. **檢索定位**：收到問題時，優先讀取 `wiki/index.md` 鎖定相關主題頁，再精讀目標章節（必要時回查 `raw/` 對應 PDF 頁碼驗證截圖細節）。
2. **嚴格依據來源作答**：回答必須以繁體中文清楚條列（可附臨時 Mermaid 圖或對照表），並標註依據的 Wiki 頁面連結 `[標題](slug.md#章節)` 與原始 PDF 頁碼。若課程簡報與 Wiki 均未涵蓋該細節，明確說明「目前課程簡報未涵蓋此部分」。
3. **主動詢問回存**：當本次查詢產出了具備考前複習價值的新綜合結果（例如跨章比較表、整合流程圖、指令速查整理）時，於回答結尾**主動詢問使用者**：是否要將這份整理回存為 `wiki/` 中的新頁面（或併入既有主題頁）？
4. **確認後回存與 Commit**：若使用者同意，依規範建立／更新頁面、補上雙向連結、同步更新 **`zensical.toml` `nav`**（如有新頁面）、`wiki/index.md` 與 `wiki/log.md`、驗證建置後自動執行 `git add wiki/ zensical.toml` 與 `git commit -m "query: <主題摘要>"`（不執行 `git push`）。

### 6.3 Lint（知識庫健康檢查）
當使用者要求執行 Lint，或累計匯入數份來源後建議執行檢查時，應全面檢查以下項目：
1. **多方同步檢查**：檢查 `wiki/*.md` 檔案集合是否與 `zensical.toml` 的 `nav` 清單及 `wiki/index.md` 目錄完全同步；並檢查所有 `category: topic` 主題頁是否皆有對應的 `slides/<topic-slug>.md`，且任何既有的 `slides/<slug>.md`（含比較／綜合頁簡報）皆對應有效的 `wiki/<slug>.md` 與雙向連結。
2. **連結完整性**：是否存在斷連、錯誤相對路徑、失效簡報連結或誤用 `[[Wikilink]]`。
3. **跨頁一致性**：不同主題頁彙整出的觀念定義、指令選項、預設值或路徑之間是否有矛盾未標註。
4. **章節結構與跨章整理檢查**：是否存在宜進一步整併跨章對比（例如跨章出現的重要對照概念）的主題。
5. **知識邊界與格式合規**：檢查是否每頁與每份簡報都有具體 `檔名.pdf, p.X` 引用、有無偷渡外部知識、有無殘留 FreeBSD 操作內容、表格管線寫法（`<code>...&#124;...</code>`）、YAML frontmatter 與 Mermaid 特殊字元雙引號是否合規。
6. **雙重編譯驗證**：依序執行 Marp CLI 編譯與 `uvx zensical build --clean` 確認建置通過。
7. **回報、修復與紀錄**：向使用者呈報發現清單；完成修復後於 `wiki/log.md` 寫入 `lint` 紀錄並自動執行 `git commit -m "lint: <摘要>"`（不執行 `git push`）。

---

## 7. 本地預覽建置與 GitHub Pages 部署備忘

- **本地完整建置檢查**（需先編譯 Marp 簡報至 `wiki/slides/`，再執行 Zensical 建置至 `site/`）：
  ```powershell
  npx --yes @marp-team/marp-cli --input-dir slides --output wiki/slides --theme-set themes --html
  uvx zensical build --clean
  ```
- **本地預覽伺服器**：完成上方 Marp 編譯後執行 `uvx zensical serve`，即可於本地預覽完整 Wiki、分組側邊欄、右側 TOC、Mermaid 圖表，並直接點擊各主題頁的 HTML 簡報連結。
- **首次發布至 GitHub Pages 檢查事項**：
  1. 於 `zensical.toml` 取消註解並填入實際的 `site_url = "https://<username>.github.io/<repo>/"`（設定後可於 `[project.theme]` 取消註解 `"navigation.instant"`）。
  2. 至 GitHub Repository **Settings → Pages → Build and deployment → Source** 選擇 **GitHub Actions**。
  3. 當使用者推送 `main` 分支時，`.github/workflows/docs.yml` 會自動依序編譯 Marp 簡報與 Zensical 網站並發布至 GitHub Pages。
