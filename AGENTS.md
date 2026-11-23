# AGENTS.md — System Administration LLM Wiki Schema

本文件是此知識庫的最高操作與架構規範（Schema）。所有在此專案執行的 LLM Agent 必須嚴格遵守本文件的三層架構、知識邊界、主題式寫作格式、側邊欄排序規則、工作流程與 Git 版控規則。

---

## 1. 專案定位與三層架構

本專案依據 **LLM Wiki** 模式建立，目標是為使用者整理**系統管理課程（System Administration）**內容，產出以**期中／期末考複習**為核心導向的持久化知識庫，並透過 **Zensical** 編譯為一致模板與主題的靜態 HTML，部署於 **GitHub Pages**。

### 三層分工
1. **Raw Sources（`raw/`）**：使用者放置課程簡報 PDF 的原始資料層。**嚴格唯讀，由使用者自行管理，並透過 `.gitignore` 排除於 Git 版控之外。**
2. **The Wiki（`wiki/`）**：由 LLM 全權維護的**主題式（Topic-Based）** Markdown 知識庫，同時也是 `zensical.toml` 指定的 `docs_dir`。每個核心課程主題整併為一篇結構連貫、易於通篇閱讀的講義頁（透過 Zensical 右側固定目錄 TOC 快速跳轉），並將跨章對照彙整為獨立的比較／綜合頁面；LLM 負責撰寫、交叉引用、繪製圖表與維護一致性。
3. **The Schema（`AGENTS.md` + `zensical.toml`）**：定義知識庫規範、頁面結構、側邊欄分組導覽（`nav`）與網站建置設定。

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
   - 內容編排以「連貫好讀、抓得住考點、易於對比」為最高準則：強調核心原理、專有名詞辨析、指令／機制的關鍵差異、操作驗證重點與常見易錯陷阱。
5. **Git 自動 Commit（但絕不 Push）**
   - 每次完成 **Ingest**，以及經使用者同意後完成的 **Query 回存** 或 **Lint 修復**，由 AI 自動執行 `git add`（涵蓋 `wiki/` 與因 `nav` 變更而更新的 `zensical.toml` 等相關設定檔，嚴禁納入 `raw/` 或 `site/`）並 `git commit`。
   - **絕對禁止自動執行 `git push`**；遠端推送一律由使用者自行決定與執行。

---

## 3. 目錄結構、語言與主題式命名規範

### 3.1 專案目錄結構
```text
TAICA_LLM-Wiki/
├── AGENTS.md                        # 本規範文件（Schema）
├── .gitignore                       # 排除 raw/、site/、.cache/ 等
├── zensical.toml                    # Zensical 網站設定、分組 nav、Mermaid 與 Tags
├── .github/workflows/docs.yml       # GitHub Actions 自動建置並部署至 GitHub Pages
├── raw/                             # （嚴格唯讀、Git 排除）課程簡報 PDF
└── wiki/                            # （Zensical docs_dir）扁平主題式知識庫
    ├── index.md                     # 總目錄、來源追蹤表與網站首頁
    ├── tags.md                      # Zensical 自動標籤索引頁
    ├── log.md                       # Append-only 演進日誌
    ├── <topic-slug>.md              # 主題講義頁（原則上一份課程主題對應 1 篇）
    └── <a-vs-b>.md                  # 跨章比較或綜合整理頁（視需要建立）
```

### 3.2 語言與專有名詞慣例
- **內文語言**：一律使用**繁體中文**撰寫。
- **專有名詞格式**：首次出現關鍵專有名詞時，採「繁體中文（English Term, 縮寫）」格式（例：`寫入時複製（Copy-on-Write, CoW）`、`邏輯捲宗管理（Logical Volume Management, LVM）`）。
- **技術原貌保留**：Linux 指令、參數旗標、檔案與目錄路徑、設定檔欄位名稱與指令輸出範例，一律維持英文原貌並使用行內程式碼 `` `...` `` 或程式碼區塊標示。

### 3.3 扁平結構、主題式切分與連結語法
- **扁平組織**：不在 `wiki/` 下建立子資料夾，所有頁面平鋪於 `wiki/` 內，側邊欄階層由 `zensical.toml` 的 `nav` 統一控管。
- **檔名規範**：一律使用**英文小寫 `kebab-case.md`**，避免 GitHub Pages URL 出現中文編碼。
  - 系統固定頁（僅此 3 份，於 `nav` 獨立分區置放，絕不與主題筆記混排）：`index.md`、`tags.md`、`log.md`。
  - 主題講義頁（`<topic-slug>.md`）：例如 `shell.md`、`shell-programming.md`、`boot-shutdown.md`、`user-management.md`、`filesystem.md`、`lvm.md`、`btrfs.md`、`linux-networking.md`、`docker-basic.md`。
  - 跨章比較／綜合頁：`<a-vs-b>.md` 或 `<topic>-synthesis.md`（例：`lvm-vs-btrfs-snapshot.md`）。
- **主題式整併原則（Option A）**：
  - **原則上每份核心課程簡報對應 1 篇完整主題講義頁**，**不另建空泛的 `src-*.md` 總覽殼頁，也不將單一指令或小知識點過度拆碎成獨立檔案**。
  - 利用清晰的 `##` 與 `###` 標題層級組織各單元，確保透過 Zensical 右側目錄（TOC）即可流暢閱讀與跳轉複習。
  - 僅在跨多份簡報對比（如 LVM vs Btrfs 快照）或將有價值的 Query 整理回存時，才另立 `comparison` / `synthesis` 頁面。
- **連結語法**：
  - 一律使用標準 Markdown 相對連結 `[顯示文字](target.md)` 或章節錨點連結 `[顯示文字](target.md#章節標題)`。**嚴禁使用 `[[Wikilink]]`**。
  - 若在 Markdown **表格儲存格**內顯示管線符號 `|`（含行內程式碼），**必須寫作 HTML 標籤與字元實體** `<code>producer &#124; filter</code>`，避免表格切欄錯誤或渲染出多餘的反斜線 `\|`。

---

## 4. 頁面模板與 Zensical 語法規範

### 4.1 YAML Frontmatter（所有知識頁必填）
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

### 4.2 主題講義頁標準結構（`<topic-slug>.md`）
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

## 學習目標與課程原則

- 對應投影片「What You Will Learn」與「Course emphasis / platform」等核心要求。（`檔名.pdf, p.2`）

## 1. <第一單元名稱>

### 1.1 <子概念或機制>
- 條理說明核心定義、架構、流程或指令行為，每點附頁碼標註。（`檔名.pdf, p.X`）

（涉及流程、階層、分層或狀態轉換時，就近嵌入對應的 Mermaid 圖表。）

（收錄 Debian 13 實作指令、設定檔範例與驗證方式，附說明與頁碼。）

!!! tip "考點"
    濃縮本單元最核心的對比、順序或驗證重點。（`檔名.pdf, p.X`）

!!! warning "常見陷阱"
    標出本單元容易混淆的誤用情境、權限風險或限制。（`檔名.pdf, p.X`）

!!! note "投影片未展開"
    僅在投影片提及名詞但未解釋細節時列出，嚴禁補充外部資料。（`檔名.pdf, p.X`）

## 2. <第二單元名稱>
...

## 考前速查表（Cheat Sheet）

（整合該主題關鍵指令、語法或檢查清單，表格內含 `|` 時使用 `<code>...&#124;...</code>`。）

## 跨主題關聯與範圍備註

- **關聯頁面**：以相對連結指向相關主題頁或比較頁（支援 `#章節` 錨點）。
- **略過項目**：明確條列本份簡報略過的 FreeBSD／無關平台頁碼。
- **投影片未展開總覽**：彙整整份簡報中未展開細節之項目與頁碼。

## 資料來源

- `檔名.pdf`：p.1–N
```

### 4.3 跨章比較／綜合頁模板（`<a-vs-b>.md` / `<topic>-synthesis.md`）
- `category: comparison` 或 `synthesis`，`tags` 包含涉及的多個主題標籤與對應類型標籤。
- 包含：**比較定位摘要**、**核心差異對照表**、**流程／決策 Mermaid 圖**、**選型與考點陷阱（Admonitions）**、**來源主題頁錨點連結**與**資料來源**。

### 4.4 Mermaid 圖表撰寫規範
- 使用標準 Markdown fenced code block：` ```mermaid ` ... ` ``` `。
- 適用情境：指令執行生命週期、開機／關機流程、PAM/NSS 分層、LVM PV/VG/LV 階層、Btrfs nested/flat subvolume 結構、工作控制狀態機、網路／容器架構等。
- **語法安全**：只要節點文字內含括號 `()`、方括號 `[]`、斜線 `/`、冒號 `:`、`$`、`|` 或空白，**務必使用雙引號包裹節點標籤**，且避免在雙引號標籤內再使用未跳脫的雙引號 `"`，例如：`A["/etc/passwd (7 欄位)"] --> B["getent passwd"]`。
- **忠實原則**：Mermaid 中的每個節點名稱、步驟順序與關係線，都必須能在來源 PDF 找到對應依據，不得為了美化而虛構投影片沒教的流程節點。

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
  1. **課程來源追蹤表（Sources Status）**：追蹤每份 `raw/*.pdf` 的匯入狀態（`待匯入` / `已匯入`）、對應主題頁連結與備註。
  2. **課程主題筆記（Topics）**：依課程順序列出各主題頁連結、一句話考試重點與涵蓋頁碼。
  3. **跨章比較與綜合整理（Comparisons & Syntheses）**：列出所有跨章比較與 Query 回存頁。

### 5.3 `wiki/log.md`
- **只增不改（Append-only）** 的時間軸紀錄。
- 每筆紀錄標題必須嚴格遵循以下格式（方便以 CLI 工具檢索）：
  - `## [YYYY-MM-DD] init | <標題>`
  - `## [YYYY-MM-DD] ingest | <PDF 檔名>`
  - `## [YYYY-MM-DD] query | <回存頁面標題>`
  - `## [YYYY-MM-DD] lint | <檢查範圍或摘要>`
  - `## [YYYY-MM-DD] refactor | <重構範圍或摘要>`
- 紀錄內容需簡潔列出：處理的來源、新增／更新頁面清單、側邊欄調整與略過或待釐清事項。

---

## 6. 三大操作流程（Workflows）

### 6.1 Ingest（逐份互動式匯入）
每次使用者要求匯入某份 `raw/` PDF 時，必須嚴格遵守以下步驟：
1. **一次一份**：每次 Ingest 僅處理 **1 份**指定的 PDF，不擅自一次批次寫入多份尚未討論的檔案。
2. **完整閱讀與視覺檢查**：透過 `read_file` 完整檢視該 PDF 所有頁面（包含頁面的文字 OCR 與投影片截圖／終端機畫面／架構圖），確保不遺漏程式碼方塊、圖表與註腳提示框。
3. **過濾與對齊檢查**：篩除 FreeBSD 專屬對照內容；標記投影片未展開但重要之名詞；確認所有內容均可回溯至具體頁碼。
4. **提出 Ingest 提案並暫停等待確認（關鍵步驟）**：
   在寫入任何 `wiki/` 檔案之前，先向使用者輸出清晰的提案報告：
   - **預計建立／更新之主題頁**：列出目標 `<topic-slug>.md`、繁中標題與在 `zensical.toml` `nav` 中的擺放順序。
   - **本份簡報核心考點與章節大綱**：條列預計採用的 `##` / `###` 單元結構與主要考點。
   - **Mermaid 圖表規劃**：說明預計在哪些章節插入何種流程／架構圖。
   - **跨主題連結或比較頁建議**：說明是否需更新既有主題頁的交叉連結，或建議獨立建立跨章比較頁（僅於真正跨章時提案）。
   - **範圍邊界說明**：列出略過了哪些 FreeBSD 頁面、哪些細節因投影片未詳述將標記為「投影片未展開」。
   等待使用者確認或微調大綱後，才可進入寫入階段。
5. **寫入與同步維護**：
   - 建立或更新目標主題頁 `<topic-slug>.md`（以及經確認的跨章關聯更新）。
   - 同步更新 **`zensical.toml` 的 `nav`**（將新頁面放入正確分組與課程順序）、**`wiki/index.md`**（來源追蹤表狀態改為 `已匯入` 並加入目錄條目）與 **`wiki/log.md`**。
6. **驗證**：
   - 確認無任何 `[[Wikilink]]`、無斷連、無表格 `\|` 誤寫、YAML frontmatter 完整、無引用外部未載資訊。
   - 執行 `uvx zensical build --clean` 驗證靜態網站能無錯建置，並確認 `zensical.toml`、`wiki/index.md` 與 `wiki/*.md` 三方一致（不將 `site/` 納入版控）。
7. **自動 Git Commit**：
   - 確認 `git status` 不包含 `raw/` 或 `site/`。
   - 執行 `git add wiki/ zensical.toml`（若有其他設定檔連動則一併加入）與 `git commit -m "ingest: <PDF檔名>"`。
   - **不執行 `git push`**，並向使用者回報本次新增／更新檔案與 commit 結果。

### 6.2 Query（查詢與詢問後回存）
1. **檢索定位**：收到問題時，優先讀取 `wiki/index.md` 鎖定相關主題頁，再精讀目標章節（必要時回查 `raw/` 對應 PDF 頁碼驗證截圖細節）。
2. **嚴格依據來源作答**：回答必須以繁體中文清楚條列（可附臨時 Mermaid 圖或對照表），並標註依據的 Wiki 頁面連結 `[標題](slug.md#章節)` 與原始 PDF 頁碼。若課程簡報與 Wiki 均未涵蓋該細節，明確說明「目前課程簡報未涵蓋此部分」。
3. **主動詢問回存**：當本次查詢產出了具備考前複習價值的新綜合結果（例如跨章比較表、整合流程圖、指令速查整理）時，於回答結尾**主動詢問使用者**：是否要將這份整理回存為 `wiki/` 中的新頁面（或併入既有主題頁）？
4. **確認後回存與 Commit**：若使用者同意，依規範建立／更新頁面、補上雙向連結、同步更新 **`zensical.toml` `nav`**（如有新頁面）、`wiki/index.md` 與 `wiki/log.md`、驗證建置後自動執行 `git add wiki/ zensical.toml` 與 `git commit -m "query: <主題摘要>"`（不執行 `git push`）。

### 6.3 Lint（知識庫健康檢查）
當使用者要求執行 Lint，或累計匯入數份來源後建議執行檢查時，應全面檢查以下項目：
1. **三方同步檢查**：檢查 `wiki/*.md` 檔案集合是否與 `zensical.toml` 的 `nav` 清單及 `wiki/index.md` 目錄完全同步（無遺漏未上側邊欄之頁面、無失效 `nav` 路徑、固定系統頁皆位於 `知識庫維運` 分組）。
2. **連結完整性**：是否存在斷連、錯誤相對路徑或誤用 `[[Wikilink]]`。
3. **跨頁一致性**：不同主題頁彙整出的觀念定義、指令選項、預設值或路徑之間是否有矛盾未標註。
4. **章節結構與跨章整理檢查**：是否存在過於零散或宜進一步整併跨章對比（例如跨章出現的重要對照概念）的主題。
5. **知識邊界與格式合規**：檢查是否每頁都有具體 `檔名.pdf, p.X` 引用、有無偷渡外部知識、有無殘留 FreeBSD 操作內容、表格管線寫法（`<code>...&#124;...</code>`）、YAML frontmatter 與 Mermaid 特殊字元雙引號是否合規。
6. **建置驗證**：執行 `uvx zensical build --clean` 確認 Zensical 建置通過。
7. **回報、修復與紀錄**：向使用者呈報發現清單；完成修復後於 `wiki/log.md` 寫入 `lint` 紀錄並自動執行 `git commit -m "lint: <摘要>"`（不執行 `git push`）。

---

## 7. Zensical 建置與 GitHub Pages 部署備忘

- **本地建置檢查**：`uvx zensical build --clean`（輸出於 `.gitignore` 排除的 `site/`）。
- **本地預覽伺服器**：`uvx zensical serve`（預覽頁面排版、側邊欄分組、右側 TOC、Tags 索引、搜尋與 Mermaid 亮／暗色渲染）。
- **首次發布至 GitHub Pages 檢查事項**：
  1. 於 `zensical.toml` 取消註解並填入實際的 `site_url = "https://<username>.github.io/<repo>/"`（設定後可於 `[project.theme]` 取消註解 `"navigation.instant"`）。
  2. 至 GitHub Repository **Settings → Pages → Build and deployment → Source** 選擇 **GitHub Actions**。
  3. 當使用者推送 `main` 分支時，`.github/workflows/docs.yml` 會自動建置並發布網站。
