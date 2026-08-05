---
name: to-engineering-spec
description: 在 Matt 的 to-spec 與 to-tickets 之間，建立並維護一份公司正式的開發規格文件 engineering-spec.md（系統分析 + 重要技術設計 + 實作約束），與 Matt Spec 分工而不競爭：Matt Spec 管需求、範圍、行為與驗收基準，本文件管系統分析、設計決策與實作約束。分析段收斂角色權限、狀態流程、畫面欄位與業務規則後停下來讓人確認，設計段才補模組責任、介面契約、資料表、交易與併發，並把技術內容明確分成「設計決策（必須遵守）／實作方向（可提出調整）／實作示意（僅供閱讀）」三種約束程度。支援建立、修訂、定稿三種模式，一律就地修訂同一份文件。本 skill 不拆 ticket、不規劃 commit、不寫任何遠端 Issue。當使用者在 Matt workflow 中需要一份可審查、可貼回公司 GitLab Issue 的正式開發規格，或需要修訂／定稿既有 engineering-spec.md 時使用。
---

# to-engineering-spec

從 Matt 最新版 Spec、程式碼現況、`CONTEXT.md` 與 ADR 出發，建立或維護**一份** `engineering-spec.md`——公司正式的開發規格文件。文件以繁體中文寫成，同時涵蓋系統分析與系統設計，但不切成「SA」「SD」兩大章。

這份文件要同時滿足三種讀者：

- **審查的人**——可直接人工審查，交付版可貼到公司 GitLab Issue、完成後留存。
- **實作的工程師**——細節完整到不必回頭問人。
- **下游 Matt Agent**——`to-tickets` 依它切 vertical slices，`implement` 依它落地，`code-review` 依它判斷實作是否偏離。

**三者之中以人為準。** 這份文件會被貼到公司 GitLab Issue，由工程師從頭讀到尾、照著它開工。Agent 吞得下的重複與贅語，人讀不下去；反過來，一份工程師願意讀完的文件，Agent 一定也用得了。

## 工作流位置

```text
grill-with-docs → to-spec → [to-engineering-spec] → 人工確認 → to-tickets → implement → code-review → [to-engineering-spec 定稿]
```

本 skill **不取代** Matt 的任何一支 skill，也不是把 Matt Spec 改寫成比較好看的 SA / SD。它補的是 Matt Spec 刻意不寫、但公司需要的那一層：系統分析、重要技術設計與實作約束。

**不是每個需求都需要它。** 小改動（不動角色權限、不跨模組、不動既有架構假設、無 schema 變更）直接走 Matt 原生流程 `to-spec → to-tickets → implement`，不要為了走完整流程而產生一份沒人會讀的文件。判斷準則見下方「適用門檻」。

需要細節時，讀本 skill 的 references：

- **開始任何寫入前，先讀 `references/workflow.md`**（三種模式的判定、確認關卡、各模式流程）。
- **起草或修訂內容前，先讀 `references/writing-rules.md`**（單一事實來源、反模式、精簡準則）——這是文件品質的核心，不可略過。
- **寫任何技術內容前，先讀 `references/constraint-levels.md`**（設計決策／實作方向／實作示意的分界與標記法）——這是本 skill 與 Matt 自由度之間的介面，不可略過。
- 起草或修訂文件時，讀 `references/document-template.md`。
- 判斷權威順序、檔案位置、與 Matt skill 的銜接方式時，讀 `references/matt-workflow-integration.md`。
- 對既有專案查找程式碼前，讀 `references/codebase-discovery.md`。
- 出現任何疑問或不明確之處時，讀 `references/clarification-and-open-questions.md`。
- 需求影響 persisted data、schema、SQL、storage、reporting 或資料生命週期時，讀 `references/data-design-checklist.md`。

## 核心行為規範（最高優先，調用時必須遵守）

**遇到模糊需求必須暫停並反問，不得擅自假設或繼續設計。**

出現以下任一情況時，**立即停下來**向使用者提問，取得明確回覆後才繼續：

- 需求不明確、語意有多種解讀。
- 邊界不清晰：不確定某職責、組件、資料或流程是否屬於本次範圍。
- 角色、權限、業務規則、狀態流轉或預期行為無法從 Matt Spec、ADR 或程式碼確認。
- 任何會實質改變範圍、架構、功能需求或驗收基準的判斷。
- 資料來源、資料流或資料寫入責任不明。
- 錯誤處理、重試、冪等或一致性策略無法判斷。
- 設計段發現分析段已確認的內容有問題。
- **Matt Spec 與本文件（或與 ADR、程式碼）互相衝突**——見下方「衝突處理」，一律指出並要求處理，不得自行選一方當答案。

禁止以「先假設一個方向繼續做」「這部分等之後再說」的方式略過。使用者選擇暫緩或尚未達成共識的問題，必須完整彙整至「待釐清事項」章節。

**但也不要重問已經問過的事。** grilling 與 Matt Spec 已經確認的需求、範圍、行為與驗收基準，直接引用，不重新訪談；程式碼能回答的問題自己查程式碼。反問只留給「缺少、且會影響需求或重大設計」的項目。

## 衝突處理（不得自行裁決）

發現 Matt Spec 與本文件、ADR 或程式碼現況不一致時：

1. **停下來**，明確指出衝突的雙方各說了什麼、衝突點在哪、各自的影響是什麼。
2. 說明可能的處理方向（改 Matt Spec／改本文件／補一則 ADR／記為待釐清），但**不自行選定**。
3. 使用者裁決後，只改那個事實的權威位置：屬於需求／範圍／行為／驗收基準的，回 Matt Spec 改；屬於系統分析／設計決策／實作約束的，改本文件；屬於重大且難以回復的取捨，補 ADR。
4. 使用者選擇暫緩時，列入「待釐清事項」，並在該處註明衝突的另一方位置。

**特別提醒：不得因為程式碼已經這樣實作，就自動把偏差合理化成正式設計。** 那是定稿模式的偏差項，不是設計決策。

## 單邊遺漏（比衝突更難發現）

「衝突處理」是**對稱偵測**：兩份文件都對同一件事有話講、內容不一致，才會觸發。它擋不住另一種——**Matt Spec 有一條設計事實，本文件完全沒有它**。

Matt 的 Spec 模板允許放 API contracts 與 technical clarifications from the developer。這些區塊裡經常混著真正的設計事實（網址形狀、端點分派方式、反查責任歸屬）。一旦本文件沒有接過來，那條事實就只活在 Matt Spec 裡：

- **沒有任何東西會報錯。** 本文件內部完全沒有徵兆——那條事實只是不在這裡而已。三種膨脹的檢查、歸屬測試、推導測試，全部查不到它。
- 下游 `to-tickets` 與 `implement` 讀本文件找設計約束，讀不到就自己決定；`code-review` 也不會判它偏離，因為本文件沒說過話。
- 本文件重寫或重構時（例如章節結構調整），這種事實最容易整條蒸發，因為它從來沒有一個明確的家。

**每次送出前主動掃一次 Matt Spec 的 Implementation Decisions / technical clarifications**（見 `references/workflow.md` 的「送出前自檢」第 7 項），逐條判斷後三選一：

| 判斷 | 處置 |
| --- | --- |
| 這是**設計事實**（形狀、落點、責任歸屬、契約） | 搬進本文件的對應章節並補上理由與被排除的替代方案；**Matt Spec 那處收斂成需求層級**（描述要達成什麼，不描述怎麼做） |
| 這是**需求層級事實**（要不要做、做到什麼程度） | 留在 Matt Spec，本文件不碰，必要時帶摘要引用 |
| 判斷不了 | 停下來問使用者，或列入「待釐清事項」 |

**不允許兩邊各留一份完整敘述。** 那是最省事的解法，但它把「單邊遺漏」換成了下一次的「互相牴觸」——兩份會分岔的敘述，而且沒人知道哪份權威。

## 適用門檻

以下任一為真時，這個需求值得一份 `engineering-spec.md`：

1. 新增或調整**角色與權限**。
2. 跨越單一模組的職責邊界（牽動其他模組）。
3. 需要新建或調整**既有架構假設**。
4. 涉及 schema 變更、交易邊界或併發控制。
5. 公司流程要求留下可審查、可貼回 GitLab Issue 的正式規格。

三者以上皆否時，主動告知使用者這個需求走 Matt 原生流程即可，並停下來讓使用者決定；使用者仍要求產出時照做，不爭論。

本 skill **沒有輕量模式**：輕量需求的最佳解是 Matt 原生流程，不是一份縮水的正式規格。

## 三種模式

由文件狀態自動判定，使用者也可用參數明確指定（`建立` / `修訂` / `定稿`，或 `create` / `revise` / `finalize`）。判定規則與各模式完整流程見 `references/workflow.md`。

| 模式 | 何時 | 產出 |
| --- | --- | --- |
| **建立** | `.scratch/<feature-slug>/engineering-spec.md` 不存在 | 新建文件，分析段 → 確認關卡 → 設計段 → `可拆 Ticket` |
| **修訂** | 文件已存在，且開發尚未結束 | 就地修訂同一份文件，列出受影響的 Matt Spec、ADR 與 Tickets |
| **定稿** | 開發與 code review 已完成 | 核對預期／設計／實作三者，就地定稿為實際交付結果 |

**三種模式共用同一份文件。** 任何情況下都不建立第二份取代文件、不產生 `-v2`、不另存副本。

## 約束程度（本 skill 的核心機制）

技術內容必須讓讀者與下游 Agent 辨認出它有多硬。三種程度，完整定義與標記法見 `references/constraint-levels.md`：

| 程度 | 意義 | 標記 |
| --- | --- | --- |
| **設計決策** | 已確認，後續必須遵守。偏離要回本 skill 修訂 | `> **設計決策**：…` |
| **實作方向** | 原則上遵循；Agent 依 codebase 發現更合適做法可提出調整，但不得靜默偏離 | `> **實作方向**：…` |
| **實作示意** | 僅協助閱讀，不構成固定簽章 | 程式碼區塊前一行 `*實作示意（非固定簽章）*` |

**不要求每一段都套上這三個標題。** 大多數正文（範圍、角色、狀態、規則、資料表欄位）本身就是設計決策，靠章節性質即可辨認；標記只用在「同一章節內程度不同」的地方。判準：讀者看到這段時，能不能答出「我可以改嗎」。

## 單一事實來源（跨文件）

**每個事實只有一個文件、一個章節能定義它。** 權威順序完整說明見 `references/matt-workflow-integration.md`：

| 事實類型 | 權威 |
| --- | --- |
| 需求、範圍、行為、驗收基準、user stories | **Matt Spec**（`.scratch/<feature-slug>/spec.md`） |
| 系統分析、設計決策、實作約束 | **本文件**（`engineering-spec.md`） |
| 少量、重要、難以回復且有真實取捨的決策理由 | **ADR**（`docs/adr/`） |
| 專案術語 | **`CONTEXT.md`** |
| 工作切片 | **Tickets**（`.scratch/<feature-slug>/issues/`）——不是需求或架構的權威 |
| 實際落地證據 | **程式碼與測試**——不自動凌駕已確認規格 |
| 正式的人類紀錄 | **公司 GitLab Issue**——可貼入最終規格，不作為 Agent 的工作佇列 |

**不在 Matt Spec 與本文件重複同一段內容。** 能引用就引用；Local markdown 路徑穩定，使用相對連結（例如 `見 [spec.md](./spec.md) 的 User Story 12`）。文件內部同樣受單一事實來源約束——完整分工表見 `references/writing-rules.md`。

## Input

至少需要下列其一：

- Matt Spec 的路徑（通常是 `.scratch/<feature-slug>/spec.md`）或內容。
- 當前對話（剛跑完 `grill-with-docs` / `to-spec` 的脈絡）。
- 待修訂或待定稿的現有 `engineering-spec.md`。

會主動讀取（存在時）：

- 現有程式碼（既有專案）。
- `CONTEXT.md` / `CONTEXT-MAP.md`、`docs/adr/`。
- `.scratch/<feature-slug>/issues/`（修訂與定稿模式需要知道受影響的 Tickets）。

選填：

- 公司 PRD 或 GitLab Issue 內容（可直接貼上，或提供 `project_id` 與 `issue_number` 透過 `gitlab-issue-fetch` 取得）。**只讀不寫**。
- 輸出路徑、語言覆蓋設定。

預設輸出語言：Skill 指令與內部推理可用英文；最終 Markdown 文件與釐清提問一律繁體中文，用台灣軟體開發常用語，避免生硬 AI 用詞。

## 檔案位置

Matt 的 issue tracker 設定為 Local markdown 時，同一功能的產物集中在一個目錄：

```text
.scratch/<feature-slug>/
├── spec.md              # Matt to-spec 產出（需求權威）
├── engineering-spec.md  # 本 skill 產出（系統分析與設計權威）
└── issues/              # Matt to-tickets 產出
    └── NN-<slug>.md
```

`<feature-slug>` **沿用 Matt Spec 已經建立的那個目錄名，不自己另取**。開工前先確認 `docs/agents/issue-tracker.md` 的設定：

- **Local markdown**（預設情境）→ 用上面的路徑。
- **其他 tracker**（GitHub / GitLab / Linear）→ 本地沒有 `.scratch/<feature-slug>/` 可用，改與使用者確認落點（建議 `docs/specs/<feature-slug>/engineering-spec.md`），並在文件資訊記錄 Matt Spec 的 Issue 連結取代相對路徑。

**本 skill 預設不修改任何遠端 Issue。** 不呼叫 `gitlab-issue-write`、不呼叫 `gh issue`、不建立、不留言、不關閉。文件寫進本地檔案，由使用者自行決定何時產出交付版並貼回公司 GitLab Issue（見下方「對外交付」）。

寫入本地檔案後，**在 Matt Spec 頂部加一行指標**（若尚未存在）：

```markdown
> Engineering spec: [engineering-spec.md](./engineering-spec.md)
```

這是唯一允許對 Matt Spec 做的改動——一行指標，不是內容。它讓 `to-tickets`、`implement` 與 `code-review` 在只拿到 `spec.md` 時仍找得到本文件。

## Workflow

完整流程見 `references/workflow.md`。摘要：

**建立模式**

1. 讀 Matt Spec 全文、當前對話、`CONTEXT.md`、相關 ADR；確認 `feature-slug` 與 issue tracker 設定。
2. 依「適用門檻」判斷是否值得產出；三者以上皆否時停下來告知使用者。
3. 既有專案：依 `references/codebase-discovery.md` 查找現況，查找與撰寫同時進行。
4. 讀 `references/writing-rules.md`、`references/constraint-levels.md`、`references/document-template.md`。
5. 寫**分析段**：文件資訊、一 背景與目的、二 範圍與非範圍、三 角色與權限、四 狀態與流程、五 功能介面、六 業務規則（`實作約束` 區塊留空）、待釐清事項。`文件狀態` 填 `分析中`。
6. **停止**。回報路徑、摘要、待釐清事項，並告知使用者確認後回來執行設計段。不自動接續。
7. 使用者確認後：寫**設計段**——回填各 BR 的 `實作約束`、新增 模組責任、介面契約、資料與 Schema、交易與併發、相容性與風險、設計檢核點、交給 to-tickets。`文件狀態` 更新為 `設計中` → `可拆 Ticket`。
8. 停止。輸出 handoff（見下方）。

**修訂模式**

1. 讀現有 `engineering-spec.md` 全文、Matt Spec、相關 ADR、`.scratch/<feature-slug>/issues/` 現況。
2. 找出該事實的**唯一權威位置**，就地修改那一處。若覺得別處也要跟著改，代表那裡違規重述了——把它改成引用。
3. 在「修訂影響」回報：受影響的 Matt Spec 段落、ADR、已發布 Tickets。**不直接修改已發布 Ticket**，除非使用者明確要求。
4. 只是局部實作細節、且不影響已確認的設計決策時，明說「不需修訂正式規格」並停手。
5. 更新 `文件狀態` 與修訂日期。停止並回報。

**定稿模式**

1. 讀最新 Matt Spec、現有 `engineering-spec.md`、ADR、最終 git diff、最終程式碼、測試與驗收結果、Tickets 完成狀態。
2. 逐項比對「預期需求」「確認設計」「實際實作」三者。
3. 有正式修訂依據的變更 → 同步回文件。
4. 實作與規格不一致、且沒有修訂紀錄 → 列為**偏差**，在「偏差與待處理」列出，**不得直接覆寫原設計**。使用者確認接受該實作結果後，才修訂對應章節並移除該偏差列。
5. 移除已失效的「預計做法」，讓文件描述**實際交付結果**；保留必要的設計決策理由，不塞開發過程日誌。
6. `文件狀態` 更新為 `定稿`。停止並告知使用者可用 `engineering-spec-deliverable` 產出交付版貼回公司 GitLab Issue。

## Codebase Discovery（既有專案）

除非使用者明確表示為全新專案或無需查找，否則一律查找相關程式碼，遵循 `references/codebase-discovery.md`。查找方式自行決定。

- 把程式碼觀察轉化為**設計影響**，而非列出檔案。
- 不臆造資料庫欄位、API 合約、權限或業務規則；所有陳述須有 Matt Spec、對話或程式碼依據。
- 不要僅憑命名相似就認定為同一事物；多個候選時檢查足夠上下文，或把模糊點記入待釐清事項。
- 程式碼與 Matt Spec 不一致時，依「衝突處理」停下來，不自行判定以哪一方為準。
- 查找結果**融入它該去的章節**，不另闢「程式碼查找結果」章節。

## 標記（只有三種）

本 skill **不使用** `PRD-confirmed` / `Code-confirmed` / `SA-confirmed` / `Inferred` 等來源標記欄位，也不產生「來源標記說明」章節——一整欄的雜訊換來的資訊，還需要一整節解釋自己。

只保留三種對讀者決策有影響的標記：

| 標記 | 用途 |
| --- | --- |
| `> **設計決策**：` | 已確認、後續必須遵守的判斷（含理由與被排除的替代方案） |
| `> **實作方向**：` | 原則上遵循、可由下游提出調整的建議 |
| 「待釐清事項」的一列 | 尚未確認、需要他人回答的問題 |

`*實作示意（非固定簽章）*` 不是標記而是免責註記，貼在程式碼區塊正上方。

能自行合理判斷的，做出決策並標為「設計決策」；確實缺少必要資訊、無法從 Matt Spec／程式碼／使用者推導的，才進「待釐清事項」。後者不得作為逃避設計責任的區塊。

## 待釐清事項

遵循 `references/clarification-and-open-questions.md`。此章節必須保留；修訂時把已回答的問題寫入對應章節的**權威位置**（只寫一處），再從本章節移除該列；無未解問題時填入一列 `目前無待釐清事項`。

## 設計檢核點（不與 Matt Spec 的驗收基準競爭）

**驗收基準的權威是 Matt Spec。** 本文件不重寫 user stories，也不產生第二套 AC。

本文件只補「設計正確性檢核」——Matt Spec 的 user story 表達不出、但設計上必須成立的事：交易邊界確實成立、併發衝突回正確訊息、schema 與索引確實落地、權限判斷式在所有入口都生效、相容性沒被破壞。

- 一條一句可判斷真偽的結果陳述，加上「依據」欄指回本文件的 BR / 設計決策編號，或 Matt Spec 的 user story 編號。
- 不重述規則細節；需要三行才講得清楚，代表在重述 BR，收斂它。
- 與 Matt Spec 的驗收基準重疊者刪除，只留 Matt Spec 表達不出的。
- 夠用即止，不膨脹成測試案例清單。

## 交給 to-tickets（handoff）

設計段或定稿完成後，本文件最後一章與回報訊息都要輸出 handoff。**本 skill 不自己拆 Tickets、不自行調用 `to-tickets`。**

回報時給使用者可直接貼上的一行：

```text
$to-tickets 依 .scratch/<feature-slug>/spec.md 與 .scratch/<feature-slug>/engineering-spec.md 拆票，並參照 docs/adr/ 與 CONTEXT.md
```

文件最後一章寫入固定的 handoff 段落（格式見 `references/document-template.md`），內容必須讓 `to-tickets` 知道：

- **Matt Spec** 決定需求、範圍與驗收基準；本文件不覆蓋它。
- 本文件的「**設計決策**」是實作約束，Ticket 不得違反。
- 「**實作方向**」可以合理調整，但要在 Ticket 或實作中記錄原因，不得靜默偏離。
- 「**實作示意**」不可被誤判成固定實作或固定簽章。
- Ticket 採 **tracer-bullet vertical slices**，每片切穿 schema、API、UI、測試；**不退回按 Controller / Service / DB 分層拆工**。
- **Ticket 不等於單一 commit。**

若 `to-tickets` 或 `implement` 在此專案中沒有穩定讀到本文件，採最小侵入的整合方式（repo-level `AGENTS.md` / `CLAUDE.md` 的 `## Agent skills` 區塊、`docs/agents/` companion reference、Matt Spec 頂部的一行指標），**不修改 Matt 的上游 Skill**。做法見 `references/matt-workflow-integration.md`。

## Output Quality Bar

文件必須符合：

- 單一 `.md` 檔案，繁體中文，章節編號連續，**不切成「SA」「SD」兩大章**。
- 每個事實只在其權威章節出現一次；跨文件的需求事實引用 Matt Spec，不複製。
- 三種膨脹都查過：重複（同一事實出現多次）、越界（內容在錯的章節）、可推導（讀者自己得出得來的話）。見 `references/writing-rules.md` 第〇章與第四章。
- 沒有空章節：不存在只有標題、內文寫「無」「N/A」「不適用」的章節或區塊（見 `references/document-template.md` 的「章節去留」）。
- 同一個概念全文用同一個詞；反覆出現的限定語已具名或上收成前提。
- 技術內容的約束程度可辨認：讀者看得出哪些必須遵守、哪些可調整、哪些只是示意。
- 細節完整：錯誤訊息原文、狀態轉移（含自環）、併發與交易約束、權限與依賴條件（含例外）、畫面欄位的必填與預設值、邊界值與格式限制、設計決策理由——一項都不能因收斂而消失。
- 業務規則以小節承載，只寫業務效果；程式落點回模組責任、畫面行為回功能介面、安全實作要求回設計檢核點。**不存在「實作注意事項」章節**。
- 每條 BR 都通過 `writing-rules.md` 的三問（業務決定的、後果不只是畫面提示、換入口仍成立）；欄位規格、畫面配置與範圍聲明不在 BR 章。
- 表格單格不超過約 80 字；超過者改用小節。
- 狀態機只有一份描述：一張完整 mermaid 狀態圖 + 一張動作對照表。**無狀態機的功能不硬套**——動作對照表降級為「動作與權限對照」移進三、角色與權限，第四章省略，不留一整欄「無」。
- 必定保留「待釐清事項」；無未解問題時填 `目前無待釐清事項`。
- 最後一章是 handoff 給 `to-tickets`。
- 不出現「由下游設計／供後續參考」式的轉嫁語句。
- 沒有 Agent 對話痕跡、沒有開發過程日誌、沒有內部工作註記；`$to-tickets ...` 指令行只出現在回報訊息，不寫進文件。

## 對外交付（本 skill 不做）

`engineering-spec.md` 是**工作版**。它刻意帶著三樣東西，在管線裡都是必要的，貼上公司 GitLab Issue 後卻全部失效：

| 工作版有的 | 貼上 Issue 後 |
| --- | --- |
| `[spec.md](./spec.md)` 等本地相對連結 | 死連結，讀者點不開也找不到 |
| ADR 編號引用（理由不複製） | 讀者只看到一個編號，看不到理由 |
| 「交給 to-tickets」章節 | 內部工作註記，對人類審查者沒有意義 |

**不要為了「隨時可貼」而在工作版裡拿掉它們**——那會讓下游 Agent 失去它需要的引用與 handoff。

正確做法是產出一份**交付版**：`engineering-spec-deliverable` 讀本文件，把引用內嵌成可獨立閱讀的內容、把 handoff 改寫成給人看的「約束程度解讀規則」，寫成 `engineering-spec_deliverable.md`，**原檔完全不動**。使用者可以繼續在工作版上修訂，再重新產一次交付版。

本 skill 不執行這個轉換、不自行調用該 skill；設計段或定稿結束時，在回報訊息裡提一句下一步即可。

## Execution Constraint

本 skill **不負責**下列事項，出現時停下來導引使用者用對應的 skill：

- 規劃 atomic commits、產生 `IMPLEMENTATION_PLAN.md`、逐 commit 執行 → 不屬於這條管線，由 Matt `implement` 負責。
- 拆 Tickets → `to-tickets`。
- 寫程式 → `implement`。
- Review 程式碼 → `code-review`。
- 建立或修改遠端 Issue（GitLab / GitHub）→ 使用者自行決定，本 skill 只產出本地文件。

寫入 Markdown 檔案後：

- 停止。
- 回報檔案路徑、內容摘要、待釐清事項狀態、修訂／偏差清單（若適用），以及下一步（分析段結束 → 回來執行設計段；設計段結束 → handoff 給 `to-tickets`；定稿結束 → 可產出交付版貼回 GitLab Issue）。
- **不自行調用下一支 skill。**
