---
name: implement-stepwise
description: 取代 Matt 的 implement，以「逐 commit 人工審查」的節奏實作單一張 ticket。首次調用時探索 codebase 並提出完整 commit checklist，經使用者確認後 append 進 ticket 檔案；之後每次調用只做下一個未完成的 commit item，實作完停下等使用者確認，確認後由本 skill 執行 git commit 並勾選，再停。內部調用 Matt 的 tdd skill 並覆寫三條規則。適合驗收條件多、預估 commit 三個以上的較重 ticket；小票請改用 implement-oneshot。當使用者要依 to-tickets 產出的 ticket 開始實作、且希望每個 commit 都親自過目時使用。
---

# implement-stepwise

依單一張 ticket 實作，**一次一個 commit，每個 commit 前後都停下來讓人審查**。

本 skill 與 `implement-oneshot` 是同一套流程的兩種模式，共用實作規範，差別只在停下來的頻率：

| | `implement-stepwise` | `implement-oneshot` |
| --- | --- | --- |
| 執行單位 | 一個 commit | 一整張 ticket |
| 人工關卡 | 每個 commit 之前 | ticket 完成後一次 |
| 適合的 ticket | 驗收條件多、預估 commit ≥ 3 | 邊界明確、預估 commit ≤ 3 |
| context | 每個 commit 之間清空 | 一路到底 |

## 核心行為規範（最高優先，調用時必須遵守）

- **一次只處理一個 commit item。** 不得連續完成多個 item 的程式修改。
- **實作完成後不得直接 commit。** 必須先回報變更並等待使用者確認；使用者點頭後才由本 skill 執行 `git commit`。
- **commit 完成後停下。** 勾選 `[x]` 後即結束本輪，等待使用者指示才處理下一個 item。
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
5. 使用者確認後，把 `## Commit checklist` 章節 **append 到 ticket 檔案末尾**。不得改動 `What to build`、`Acceptance criteria`、`Blocked by`、`Status` 等 Matt 既有欄位。
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
5. 回報本次變更；列出應包含與應排除的檔案；給出建議的 commit message。
6. **停止，等待使用者確認。**
7. 使用者確認後，執行 `git add` 與 `git commit`。
8. 將該項從 `[ ]` 改為 `[x]`，並依 `implementation-rules.md` 的「邊做邊記」在該項下方補記測試與涵蓋的驗收條件：

   ```md
   - [x] 1. feat: 建立 X 的資料結構與讀取路徑
         seam: `loadX()` 公開介面
         tests: TestDmsDeviceMovementService.查詢僅納入確認狀態的工單
         涵蓋: 驗收條件 #8, #9
   ```

9. **停止**，並提醒使用者：**下一個 commit 請清空 context 後重新調用本 skill**。

只實作當前 commit 範圍內的變更，不順手改動其他 item 或計畫外的程式碼。若實作過程中該 commit 的細節有合理調整（仍在原範圍與目的內），同步更新 checklist 該項的文字；但不得擴張或改變其整體方向。

### 為什麼每個 commit 之間都要清 context

這是本 skill 相對 `implement-oneshot` 的主要優勢來源，**不清就等於沒有**。

agentic loop 的成本是「每回合的 context 大小」對回合數的累加，而 context 在一次 session 內只增不減——第 50 個回合送出的 prompt，仍帶著第 5 個回合讀進來的大型原始檔。一路做到底，成本是這條成長曲線的完整積分。

進度既然存放在 ticket 的 checklist 裡，每個 commit 就能是一次獨立的短 session，只讀該 commit 真正需要的檔案。

而且逐 commit 審查**本身會增加回合數**（回報 → 確認 → commit → 勾選）。在乾淨的小 context 裡這些回合很便宜；在累積的大 context 裡，每一個都要重付一次完整 context。**同一個設計，清與不清，一個省錢一個燒錢。**

因此不要為了「順手」而在同一個 context 內連續做多個 commit。

## 收尾（checklist 全部完成時）

1. **跑一次完整測試套件**。
2. **逐條核對 ticket 的 `Acceptance criteria`**，回報每一條是達成、未達成或部分達成，並指出對應的程式碼或測試。
3. 若完整測試套件出現失敗、或有驗收條件未達成：**不要就地修掉**。回報問題，並建議在 checklist 追加新的 commit item 處理，維持逐 commit 審查的節奏。
4. 全部通過後回報 ticket 完成。

此步驟**一律在本 session 內完成，不得開 sub-agent**。

**是否執行 `/code-review` 由使用者決定，本 skill 不主動調用也不主動詢問。** 逐 commit 審查已經涵蓋了單點的正確性與慣例問題；`/code-review` 的價值在跨 commit 的結構性問題（重複邏輯、命名不一致、變更散落），那是逐片審查結構上看不見的。需要時使用者可自行調用，或參考 `implement-oneshot` 的「code-review 詢問」一節所述的瘦身做法。

## 暫停與回報

發現以下任一情況，**暫停並回報，不自行重寫 checklist**：

- checklist 與實際程式碼不一致。
- 當前 commit 無法照規劃實作。
- 需要改變整體實作方向或重新拆分 commit。
- 需要超出本 commit 範圍的重構。

回報時說明問題所在與影響，並建議使用者是重新規劃 checklist、或回到 `to-tickets` 調整 ticket 本身。

## 下一步引導（純提示，不主動調用）

- **本輪 commit 完成、仍有未完成項目** → 告知剩餘項目數，並明確提醒**清空 context 後再調用本 skill 做下一個 commit**。
- **checklist 全部完成** → 提示使用者清空 context，取下一張 blocker 已滿足的 ticket。**整條 branch 的 ticket 全部完成後**，提示開新 session 執行 `to-acceptance-map` 做獨立的驗收覆蓋盤點。
- **需要回頭修正** → commit 級 → 重新規劃 checklist；ticket 級（範圍、驗收條件有誤）→ 回到 `to-tickets`。
