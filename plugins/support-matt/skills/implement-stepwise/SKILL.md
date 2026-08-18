---
name: implement-stepwise
description: 取代 Matt 的 implement，以「逐 commit 人工審查」的節奏實作單一張 ticket。首次調用時探索 codebase 並提出完整 commit checklist，經使用者確認後 append 進 ticket 檔案；之後每個 commit item 只設一個人工關卡：實作完停下回報，結尾固定附上二選一：commit 後停下等使用者清 context，或 commit 後自動接續下一個 item；確認後由本 skill 執行 git commit 並勾選，再照答案走。內部調用 Matt 的 tdd skill 並覆寫三條規則。適合驗收條件多、預估 commit 三個以上的較重 ticket；小票請改用 implement-oneshot。當使用者要依 to-tickets 產出的 ticket 開始實作、且希望每個 commit 都親自過目時使用。
---

# implement-stepwise

依單一張 ticket 實作，**一次一個 commit，每個 commit 送出之前停下來讓人審查**。

本 skill 與 `implement-oneshot` 是同一套流程的兩種模式，共用實作規範，差別只在停下來的頻率：

| | `implement-stepwise` | `implement-oneshot` |
| --- | --- | --- |
| 執行單位 | 一個 commit | 一整張 ticket |
| 人工關卡 | 每個 commit 之前 | ticket 完成後一次 |
| 適合的 ticket | 驗收條件多、預估 commit ≥ 3 | 邊界明確、預估 commit ≤ 3 |
| context | 每個 commit 之間，由使用者決定要不要清 | 一路到底 |

## 核心行為規範（最高優先，調用時必須遵守）

- **一次只處理一個 commit item。** 不得在同一個人工關卡內完成多個 item 的程式修改；要接著做下一個 item，必須是使用者在該關卡明確允許的。
- **實作完成後不得直接 commit。** 必須先回報變更並等待使用者確認；使用者點頭後才由本 skill 執行 `git commit`。
- **每個 commit 只設一個人工關卡，就在 commit 之前。** 回報變更時，結尾固定附上 A（commit 後停下等使用者清 context）/ B（commit 後自動接續下一個 item）二選一，格式見「執行模式」第 5 步。**commit 之後不再多停一次徵詢**，照使用者已經給的答案走。
- **那兩個選項照抄，不要自己發揮。** 不增減選項、不附上建議或傾向、不分析該選哪個——commit 後的去向由使用者決定，本 skill 不代為判斷。
- **不得自行重寫 commit checklist。** 發現 checklist 與實際程式碼不一致、或某個 commit 無法照規劃實作時，**暫停並回報**，由使用者決定是否重新規劃。
- **只處理一張 ticket。** 不得跨 ticket 作業。

## 共用規範（必讀）

動手前先讀這兩份，位於 plugin 的 `references/` 目錄（相對本 skill 為 `../../references/`）：

- **`token-discipline.md`** —— 探索優先序（圖譜優先）、三條防呆、回合數成本。
- **`implementation-rules.md`** —— TDD 三條覆寫、程式碼與註解規範、commit 格式、邊做邊記。

兩份與 `implement-oneshot` 共用，內容不在本檔重複。本 skill 只補上與「逐 commit」有關的部分。

其中 **TDD 覆寫 1（seam 只確認一次）** 在本流程的落點是：**規劃模式第 3–4 步**一次列出全部 seam 並取得確認，之後每個 commit 不再重問。checklist 未載明 seam 的項目，才需在動手前補問。

## 前置讀取

不要硬編路徑，一律從設定檔取得：

1. `.ai/docs/agents/issue-tracker.md` — ticket 檔案的位置與慣例（`setup-matt-preset` 已將 Matt 預設路徑下移至 `.ai/`）。找不到時改讀 `docs/agents/issue-tracker.md`；兩者皆無則停下來請使用者先執行 `setup-matt-preset`。
2. `.ai/docs/agents/domain.md` — `CONTEXT.md` 與 ADR 的位置。依它指出的路徑讀取；檔案不存在時**靜默略過**，不要提示使用者建立。
3. 目標 ticket 檔案本身。

Ticket 標題、測試名稱與介面命名一律使用 `CONTEXT.md` 的領域詞彙；產出若與既有 ADR 抵觸，明確指出而非默默覆蓋。

**不要主動去讀 `spec.md` / `engineering-spec.md`。** ticket 應當自足；真的缺資訊時才回頭撈，並在回報中指出 ticket 哪裡不足——那是 `to-tickets` 該修的問題。這兩份文件合計可達 50KB 以上，每輪重讀會抵銷掉清 context 的效益。

## 模式判斷

讀取 ticket 後，依其內容自動判斷本輪模式：

- 沒有 `## Commit checklist` 章節 → **規劃模式**
- 有該章節且仍有 `[ ]` 項目 → **執行模式**
- 有該章節且全部 `[x]` → **收尾**

這個判斷讓 ticket 檔案本身成為進度狀態，因此中途清 context 或換 session 都能直接接續。

## 規劃模式

1. 讀 ticket 的 `What to build`、`Acceptance criteria`、`Blocked by`。
2. **強制探索 codebase 現況**，包含 `Blocked by` 所列 ticket 已經留下的程式碼。這一步不得跳過——commit 拆分必須建立在真實的程式碼狀態上，不是憑 ticket 文字想像。探索方式依 `token-discipline.md`。
3. 拆分 commit，並為每一項宣告它要測試的 **seam**（公開介面邊界）。
4. 將完整清單列給使用者，詢問：粒度是否合適？seam 是否正確？有無需要合併或再拆的項目？
5. 使用者確認後，把 `## Commit checklist` 章節 **append 到 ticket 檔案末尾**。不得改動 `What to build`、`Acceptance criteria`、`Blocked by`、`Status` 等 Matt 既有欄位的**文字內容**（收尾階段可更新驗收條件的 checkbox 狀態，見 `implementation-rules.md` 的「寫回 ticket」；那只改勾選，不改條文）。
6. **停止。** 不在同一輪開始實作。

**規劃階段的探索要淺。** 目的是畫出 commit 的邊界，不是預先理解每個 commit 的實作細節——那些細節在清 context 後會作廢，等該 commit 真正開工時再讀才不會白費。把規劃學到的東西壓縮進 seam 宣告與變更檔案清單，那是它們的保存形式。

### Commit 拆分原則

- 每個 commit 對應一個 TDD 紅綠切片。
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

規劃階段只寫 `seam`；`tests` 與 `涵蓋` 兩行在該 commit 完成勾選時才補上。

## 執行模式

1. 找出第一個 `[ ]` 項目。
2. 覆述該 commit 的目的、seam、預期變更檔案與邊界，確認與 checklist 一致。
3. 依 `implementation-rules.md` 的 TDD 規範實作。
4. **執行 typecheck**，並跑與本次變更相關的單一測試檔。
5. 回報本次變更；列出應包含與應排除的檔案；給出建議的 commit message。這些原有資訊照舊全部保留，**最後固定以這個二選一收尾**：

   ```md
   確認後我會執行 commit。commit 之後：
   - A：停下，等你清空 context 再開新一輪
   - B：自動接續下一個 commit item
   ```

   **照這個形式寫就好。** 不要自行增減選項、不要附加建議或傾向、不要替使用者分析該選哪個——`token-discipline.md` 之外的取捨是使用者的事，本 skill 只負責把選擇權交出去。使用者若想知道判斷依據，見下方「要不要清 context」一節。

   checklist 已無其他 `[ ]` 項目時不必問這題，直接走收尾。

6. **停止，等待使用者確認。** 這是本 commit 唯一的人工關卡，兩件事在這裡一次問完：commit 本身，以及 commit 之後的去向。
7. 使用者確認後，執行 `git add` 與 `git commit`。
8. 將該項從 `[ ]` 改為 `[x]`，並依 `implementation-rules.md` 的「邊做邊記」在該項下方補記測試與涵蓋的驗收條件：

   ```md
   - [x] 1. feat: 建立 X 的資料結構與讀取路徑
         seam: `loadX()` 公開介面
         tests: TestDmsDeviceMovementService.查詢僅納入確認狀態的工單
         涵蓋: 驗收條件 #8, #9
   ```

9. 依第 5 步取得的答案決定去向：

   - **選 A（停下）** → **停止**，告知剩餘項目數，下一輪重新調用本 skill 即可接續。
   - **選 B（接續）** → 直接回到第 1 步處理下一個 item，**不再另外徵詢**。回到第 2、3 步時，**先清點 context 裡已經有的東西再決定要讀什麼**：上一個 commit 讀過的檔案、剛寫出來的程式碼都還在，不要重讀。尤其是本輪新增的程式碼——`token-discipline.md` 要求這類程式碼不查圖譜、直接讀檔，而它們此刻已經在 context 裡，這正是接續模式最省的地方。
   - **使用者沒有明確表示** → 一律當作 A 停下。

只實作當前 commit 範圍內的變更，不順手改動其他 item 或計畫外的程式碼。若實作過程中該 commit 的細節有合理調整（仍在原範圍與目的內），同步更新 checklist 該項的文字；但不得擴張或改變其整體方向。

### 要不要清 context

**這個決定屬於使用者，本 skill 不代為判斷。** 以下是留給使用者自己翻閱的權衡材料，**不是每輪要背給使用者聽的內容**——第 5 步固定只給 A / B 兩個選項，除非使用者主動問起，否則不要複述本節。

**清掉划算的情況：**

- agentic loop 的成本是「每回合的 context 大小」對回合數的累加，而 context 在一次 session 內只增不減——第 50 個回合送出的 prompt，仍帶著第 5 個回合讀進來的大型原始檔。連著做完整張 ticket，付的是這條成長曲線的完整積分。
- 長 context 還會拖累**準確度**：裡面躺著已經被改過的舊檔案內容、放棄的方案、失敗的測試輸出，越到後面的 commit，模型越容易錨定在過時的狀態上。
- 本輪如果做過大量探索、或測試反覆失敗來回除錯，累積的雜訊特別多。
- 下一個 item 動的是不同模組、不同檔案，帶著走的東西幾乎用不上。

**留著划算的情況：**

- 下一個 item 與本輪共用同一批檔案與 seam，尤其是要接上本輪剛寫出來的程式碼——那些內容**不在圖譜索引裡**（見 `token-discipline.md`），清掉只能重新整檔讀回來。
- 本輪 context 還很乾淨：沒有整檔讀過大檔、沒有除錯回圈。
- 重新進入是有成本的：清掉之後要重讀 `issue-tracker.md`、`domain.md`、ticket 全文，再重新定位一次程式碼，而且這些都是未命中快取的新 token。

進度存放在 ticket 的 checklist 裡，因此**清或不清都不會遺失狀態**——每個 commit 都能是一次獨立的短 session，也能接著同一個 context 做下去。這正是這個決定可以交給使用者的原因。

## 收尾（checklist 全部完成時）

依 `implementation-rules.md` 的「收尾」執行——跑受影響範圍的測試、逐條核對驗收條件、**把核對結果寫回 ticket**（勾選達成項 + append 帶證據的核對表）、回報、詢問冷眼審查。**不跑完整測試套件**（那是 `to-acceptance-map` 在 branch 結束時的工作），**不判斷 scope creep 或實作對錯**，**不開 sub-agent**。

本 skill 特有的兩點：

- **發現問題時，建議追加新的 commit item 處理**，維持逐 commit 審查的節奏，不要把修正混進已完成的項目。
- 詢問冷眼審查時附註：**你已逐 commit 審查過，`/code-review` 的需求通常較低**。它剩下的價值主要在跨 commit 的結構性問題（重複邏輯、命名不一致、變更散落），那是逐片審查結構上看不見的。

## 暫停與回報

發現以下任一情況，**暫停並回報，不自行重寫 checklist**：

- checklist 與實際程式碼不一致。
- 當前 commit 無法照規劃實作。
- 需要改變整體實作方向或重新拆分 commit。
- 需要超出本 commit 範圍的重構。

回報時說明問題所在與影響，並建議使用者是重新規劃 checklist、或回到 `to-tickets` 調整 ticket 本身。

## 下一步引導（純提示，不主動調用）

- **本輪 commit 完成、使用者選 A（停下）** → 告知剩餘項目數，下一輪重新調用本 skill。不必再勸清或不清——使用者選 A 時就已經做完那個決定了。
- **checklist 全部完成** → 提示使用者清空 context，取下一張 blocker 已滿足的 ticket。**整條 branch 的 ticket 全部完成後**，提示開新 session 執行 `to-acceptance-map` 做獨立的驗收覆蓋盤點。
- **需要回頭修正** → commit 級 → 重新規劃 checklist；ticket 級（範圍、驗收條件有誤）→ 回到 `to-tickets`。
