---
name: implement-stepwise
description: 取代 Matt 的 implement，以「逐 commit 人工審查」的節奏實作單一張 ticket。首次調用時探索 codebase 並提出完整 commit checklist，經使用者確認後 append 進 ticket 檔案；之後每次調用只做下一個未完成的 commit item，實作完停下等使用者確認，確認後由本 skill 執行 git commit 並勾選，再停。內部調用 Matt 的 tdd skill 並覆寫三條規則。不調用 code-review（改以驗收條件逐條核對收尾），大幅降低 token 消耗。當使用者要依 to-tickets 產出的 ticket 開始實作、且希望每個 commit 都親自過目時使用。
---

# implement-stepwise

依單一張 ticket 實作，**一次一個 commit，每個 commit 前後都停下來讓人審查**。

本 skill **取代** Matt 的 `implement`。兩者的差異與取捨：

| | Matt `implement` | 本 skill |
| --- | --- | --- |
| 執行單位 | 一整張 ticket 跑完 | 一個 commit |
| 人工關卡 | ticket 結束時 | 每個 commit 之前 |
| commit | 收尾時自行 commit | 每個 item 經確認後 commit |
| 收尾審查 | `/code-review`（兩個 sub-agent） | 驗收條件逐條核對（本 session，不開 sub-agent） |

**不調用 `/code-review`。** 理由是每個 commit 進去之前使用者都已親眼審查過，那是比事後掃描更前面的關卡；`/code-review` 的兩個 sub-agent 各扛一份完整 diff，在已有逐 commit 審查的前提下重複度過高。其唯一不可取代的價值——Spec 軸（交付內容是否符合 ticket 要求）——改由收尾的驗收條件核對承接。

## 核心行為規範（最高優先，調用時必須遵守）

- **一次只處理一個 commit item。** 不得連續完成多個 item 的程式修改。
- **實作完成後不得直接 commit。** 必須先回報變更並等待使用者確認；使用者點頭後才由本 skill 執行 `git commit`。
- **commit 完成後停下。** 勾選 `[x]` 後即結束本輪，等待使用者指示才處理下一個 item。
- **不得自行重寫 commit checklist。** 發現 checklist 與實際程式碼不一致、或某個 commit 無法照規劃實作時，**暫停並回報**，由使用者決定是否重新規劃。
- **只處理一張 ticket。** 不得跨 ticket 作業。

## 前置讀取

不要硬編路徑，一律從設定檔取得：

1. `.ai/docs/agents/issue-tracker.md` — ticket 檔案的位置與慣例（`setup-matt-preset` 已將 Matt 預設路徑下移至 `.ai/`）。找不到時改讀 `docs/agents/issue-tracker.md`；兩者皆無則停下來請使用者先執行 `setup-matt-preset`。
2. `.ai/docs/agents/domain.md` — `CONTEXT.md` 與 ADR 的位置。依它指出的路徑讀取；檔案不存在時**靜默略過**，不要提示使用者建立。
3. 目標 ticket 檔案本身。

Ticket 標題、測試名稱與介面命名一律使用 `CONTEXT.md` 的領域詞彙；產出若與既有 ADR 抵觸，明確指出而非默默覆蓋。

## 模式判斷

讀取 ticket 後，依其內容自動判斷本輪模式：

- 沒有 `## Commit checklist` 章節 → **規劃模式**
- 有該章節且仍有 `[ ]` 項目 → **執行模式**
- 有該章節且全部 `[x]` → **收尾**

這個判斷讓 ticket 檔案本身成為進度狀態，因此中途清 context 或換 session 都能直接接續。

## 規劃模式

1. 讀 ticket 的 `What to build`、`Acceptance criteria`、`Blocked by`。
2. **強制探索 codebase 現況**，包含 `Blocked by` 所列 ticket 已經留下的程式碼。這一步不得跳過——commit 拆分必須建立在真實的程式碼狀態上，不是憑 ticket 文字想像。探索方式依「探索優先序」。
3. 拆分 commit，並為每一項宣告它要測試的 **seam**（公開介面邊界）。
4. 將完整清單列給使用者，詢問：粒度是否合適？seam 是否正確？有無需要合併或再拆的項目？
5. 使用者確認後，把 `## Commit checklist` 章節 **append 到 ticket 檔案末尾**。不得改動 `What to build`、`Acceptance criteria`、`Blocked by`、`Status` 等 Matt 既有欄位。
6. **停止。** 不在同一輪開始實作。

### Commit 拆分原則

- 每個 commit 對應一個 TDD 紅綠切片：**測試與其實作屬於同一個 commit**，不另拆。
- 每個 commit 聚焦單一且連貫的變更，可獨立理解。
- 避免把性質相同的細微修改拆成多個無意義的 commit。
- 避免單一 commit 大到無法在一輪內安全完成。
- 涉及 migration、rollback、相容性風險或跨模組時，於該項標示風險。

### 章節格式

```md
## Commit checklist
- [ ] 1. feat: 建立 X 的資料結構與讀取路徑
      seam: `loadX()` 公開介面
- [ ] 2. feat: 接上 API 端點
      seam: `POST /x` HTTP 層
```

規劃階段只寫 `seam`；`tests` 與 `涵蓋` 兩行在該 commit 完成勾選時才補上（見「邊做邊記」）。

## 邊做邊記

勾選 `[x]` 時，在該項下方補上兩行——本次寫出的測試，以及它們涵蓋的 ticket 驗收條件編號：

```md
- [x] 1. feat: 建立 X 的資料結構與讀取路徑
      seam: `loadX()` 公開介面
      tests: TestDmsDeviceMovementService.查詢僅納入確認狀態的工單
             TestDmsDeviceMovementService.生效數量為零的明細不出現
      涵蓋: 驗收條件 #8, #9
```

規則：

- 只記**本次 commit 實際寫出**的測試，不要把既有測試算進來。
- 測試名稱寫到可直接搜尋的程度（檔名 + 測試方法名）。
- 本次沒有對應到任何驗收條件時（例如純重構、設定檔），`涵蓋` 寫「無」，不要硬湊。
- 驗收條件編號以該 ticket 內的出現順序為準。

**這份記錄是線索，不是證明。** 它由實作者自己填寫，等同自我聲明；真正的驗證由 `to-acceptance-map` 在開發完畢後於獨立 session 執行。因此不需要為了「好看」而美化——沒寫測試就照實留空，那正是驗證階段要抓的東西。

## 執行模式

1. 找出第一個 `[ ]` 項目。
2. 覆述該 commit 的目的、seam、預期變更檔案與邊界，確認與 checklist 一致。
3. 依「TDD 規範」實作。
4. **執行 typecheck**，並跑與本次變更相關的單一測試檔。
5. 回報本次變更；列出應包含與應排除的檔案；給出建議的 commit message。
6. **停止，等待使用者確認。**
7. 使用者確認後，執行 `git add` 與 `git commit`。
8. 將該項從 `[ ]` 改為 `[x]`，並在該項下方補記本次寫出的測試，以及它們涵蓋的驗收條件編號（見「邊做邊記」）。
9. **停止**，並提醒使用者：**下一個 commit 請清空 context 後重新調用本 skill**。

### 為什麼每個 commit 之間都要清 context

這是本 skill 最大的 token 優勢，比不跑 `/code-review` 省得更多。

agentic loop 的成本是「每回合的 context 大小」對回合數的累加，而 context 在一次 session 內只增不減——第 50 個回合送出的 prompt，仍帶著第 5 個回合讀進來的規格全文與大型原始檔。一張 ticket 一路做到底，成本是這條成長曲線的完整積分。

進度既然存放在 ticket 的 checklist 裡，每個 commit 就能是一次獨立的短 session，只讀該 commit 真正需要的檔案。四個 commit 拆成四段短曲線，總和遠小於一條長曲線。

因此**不要**為了「順手」而在同一個 context 內連續做多個 commit——那會把本 skill 最主要的效益丟掉。

只實作當前 commit 範圍內的變更，不順手改動其他 item 或計畫外的程式碼。若實作過程中該 commit 的細節有合理調整（仍在原範圍與目的內），同步更新 checklist 該項的文字；但不得擴張或改變其整體方向。

## 收尾（checklist 全部完成時）

1. **跑一次完整測試套件**（承接 Matt `implement` 的「full test suite once at the end」）。
2. **逐條核對 ticket 的 `Acceptance criteria`**，回報每一條是達成、未達成或部分達成，並指出對應的程式碼或測試。
3. 若完整測試套件出現失敗、或有驗收條件未達成：**不要就地修掉**。回報問題，並建議在 checklist 追加新的 commit item 處理，維持逐 commit 審查的節奏。
4. 全部通過後回報 ticket 完成。

此步驟**一律在本 session 內完成，不得開 sub-agent**——這正是本 skill 相對 `/code-review` 的成本優勢所在。

## TDD 規範

調用 Matt 的 **`/tdd`** skill，取得測試品質判準：好測試的定義、seam 概念、以及三個 anti-pattern（implementation-coupled、tautological、horizontal slicing）與 `tests.md`、`mocking.md` 的具體規則。這些內容本 skill 不重複，一律以 `/tdd` 為準。

在其之上套用三條覆寫：

**覆寫 1 — seam 已於規劃階段確認。** `/tdd` 要求「寫任何測試前先寫下 seam 並與使用者確認，未確認的 seam 不得寫測試」。本流程已在規劃模式第 3–4 步完成該確認並記錄於 checklist，**不再於每個 commit 重複詢問**。checklist 未載明 seam 的項目，才需在動手前補問。

**覆寫 2 — 先看它失敗，且必須因為對的理由失敗。** `/tdd` 只說 "Red before green"。本流程要求：執行測試，確認它**因「功能尚未實作」而失敗**，而非因語法錯誤或環境問題而失敗。**此步驟不得跳過**——沒看過它失敗，就不知道它測的是不是對的東西。

**覆寫 3 — 重構在 commit 內就地處理，不推給 review 階段。** `/tdd` 寫著「Refactoring is not part of the loop. It belongs to the review stage (see the `code-review` skill)」——但本流程不執行 `/code-review`，照抄會讓重構無處可去。改為：在**本 commit 剛寫的程式碼**內、且測試保持綠燈時做最小整理。
- **不得改動本次以外的既有程式碼**（不順手重構、不改周邊命名、不調整格式或註解）。
- **預設不整理**，可直接省略此步。
- 若判斷需要較大範圍的重構，**不要在此進行**，依「暫停與回報」提出，由使用者決定是否新增 commit item。

### 例外

純設定檔、無邏輯的樣板或產生碼、純文件、一次性拋棄式變更，可不適用測試先行，但**須在回報中說明原因**。

## 探索優先序

**若本專案已建立程式碼知識圖譜或等效的索引式導覽工具，一律優先使用它定位，再針對性讀取檔案片段。**

如何得知專案有沒有：讀 `CLAUDE.md` / `AGENTS.md` 是否宣告了可用的索引與其工具，或檢查是否有對應的 MCP 工具可用。**本 skill 不指定使用哪一套**——由你依專案當下實際可用的工具自行評估選擇。

優先序：

1. **圖譜 / 索引查詢** —— 定位符號定義與引用、找出呼叫關係、評估變更影響範圍、追執行流程。
2. **針對性讀取** —— 依前一步得到的精確位置，只讀需要的區段。
3. **grep** —— 前兩者不適用時才用（例如找字串常數、設定值）。
4. **整檔閱讀** —— 只在該檔案本身就是變更目標、且篇幅不大時。

理由是成本：在大型 codebase 裡靠 grep 加整檔閱讀去理解結構，會把大量無關內容永久留在 context；圖譜查詢回傳的是定位結果，量級小上一到兩個數量級。

**但圖譜只對既有程式碼可信。** 索引反映的是上次建立索引時的狀態，**不包含本輪剛寫出的程式碼**，也可能因為專案已變動而過期。涉及本 ticket 已完成 commit 所產生的新程式碼時，直接讀檔，不要查圖譜。索引明顯過期時，回報使用者，不要自行重建。

## 工具使用規範（token 防呆）

以下三條各自對應實測中確認過的浪費，違反任一條都會把大量無用內容永久留在 context 裡——**留下的內容之後每一個回合都會重送**。

- **改檔一律用 Edit 做定點替換，不要用 `sed -i` 之類的整檔改寫工具。** 整檔改寫會觸發 harness 把修改後的完整檔案回吐進 context；對一個 1200 行的檔案做六次機械修改，就是六次整檔回吐，而那些內容你早就知道了。
- **grep 前先確認圖譜是否更適合**（見「探索優先序」），確定要 grep 時先限定範圍，排除產生物與壓縮檔。 掃 `wwwroot/`、`dist/`、`node_modules/`、`*.min.js`、`*.min.css` 這類路徑，會把整份壓縮過的單行 JS 倒進 context，且無法再移除。先用副檔名或子目錄縮小範圍再搜。
- **為了抄版型或慣例而讀既有檔案時，只讀需要的區段。** 用 offset / limit 讀目標區域，不要為了其中 40 行而載入整份 389 行的檔案。

## 程式碼與註解規範

- **預設向既有程式碼看齊。** 動手前先看同目錄的既有檔案，沿用現行分層、命名慣例、錯誤處理方式與格式風格，不引入新風格、新抽象或新依賴。個人偏好一律讓位給專案現況。
  - 可偏離的情況僅限：既有寫法有明確缺陷（安全性、正確性、資源未釋放）、使用已棄用 API、既有風格本身不一致無單一慣例可循，或 ticket 已明訂不同做法。
  - **偏離時必須在回報中指出**「哪裡沒有沿用、依據哪一條理由、影響範圍」，讓使用者可當場否決。不得靜默偏離。
- **代碼層不寫需求文件編號**（`FR-xx`、`BR-xx`、`AC-xx` 等）。改為直接描述行為本身。ticket 與規格文件仍正常使用這些編號。
- **註解收斂到最少。** 只保留程式本身無法表達的約束或非顯而易見的推論，一個約束一行。刪除敘述性、提醒性、解釋本次改動理由的註解——那些屬於 commit message。測試斷言預設不加註解。

## 關於 `/codebase-design`

`/codebase-design` 是 model-invoked 的深模組設計語彙（module、interface、depth、seam、adapter、leverage、locality）。它會因為 seam 相關討論被自動拉進來。

**預設不主動調用。** 僅在規劃模式中遇到 seam 位置真的難以決定時才載入——那種情況它值得那些 token。單純的日常實作不需要它。

## Commit 規則

使用 conventional commit 格式。

- 允許的 type 僅 `feat`、`fix`、`refactor`、`chore`、`test`。
- Commit type prefix 保持英文，描述以繁體中文撰寫。
- 若本次工作有已知的來源 issue 編號（例如經 `gitlab-issue-fetch` 取得的 GitLab issue），每則 commit message 都必須包含 `#<issue_number>`；純本地 ticket 無此編號時省略。

## 輸出格式

實作完成、等待確認時：

```text
Commit <N>/<總數> 已完成實作，等待確認：
- Commit message: feat: 某個 commit 標題
- Seam: <本次測試的公開介面>
- Files to include:
  - path/to/file
- Files to exclude:
  - path/to/other-file
- Checks: typecheck 通過 / 相關測試 <檔名> 通過

確認後我會執行：
git add path/to/file
git commit -m "feat: 某個 commit 標題"
```

## 暫停與回報

發現以下任一情況，**暫停並回報，不自行重寫 checklist**：

- checklist 與實際程式碼不一致。
- 當前 commit 無法照規劃實作。
- 需要改變整體實作方向或重新拆分 commit。
- 需要超出本 commit 範圍的重構。

回報時說明問題所在與影響，並建議使用者是重新規劃 checklist、或回到 `to-tickets` 調整 ticket 本身。

## 下一步引導（純提示，不主動調用）

- **本輪 commit 完成、仍有未完成項目** → 告知剩餘項目數，並明確提醒**清空 context 後再調用本 skill 做下一個 commit**。
- **checklist 全部完成** → 已於收尾回報完整測試與驗收條件核對結果。提示使用者清空 context，取下一張 blocker 已滿足的 ticket 重新調用本 skill。**整條 branch 的 ticket 全部完成後**，提示使用者開新 session 執行 `to-acceptance-map`，做獨立的驗收覆蓋盤點。
- **需要回頭修正** → commit 級 → 重新規劃 checklist；ticket 級（範圍、驗收條件有誤）→ 回到 `to-tickets`。
