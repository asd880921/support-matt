---
name: implement-oneshot
description: 在單一 session 內把一整張 ticket 做完，形狀貼近 Matt 原生的 implement，但有三處差異：開場先評估 ticket 規模，過重時建議改用 implement-stepwise 或回 to-tickets 拆小；ticket 完成後詢問使用者是否執行 code-review，不強制綁定；執行時套用 token 紀律與瘦身版的 code-review 參數。適合邊界明確、預估 commit 三個以內的較小 ticket。當使用者要一次做完一張小票、不需要逐 commit 停下審查時使用。
---

# implement-oneshot

在**單一 session** 內把一整張 ticket 做完，收尾時由使用者決定要不要跑 `/code-review`。

本 skill 與 `implement-stepwise` 是同一套流程的兩種模式，共用實作規範，差別只在停下來的頻率：

| | `implement-oneshot` | `implement-stepwise` |
| --- | --- | --- |
| 執行單位 | 一整張 ticket | 一個 commit |
| 人工關卡 | ticket 完成後一次 | 每個 commit 之前 |
| 適合的 ticket | 邊界明確、預估 commit ≤ 3 | 驗收條件多、預估 commit ≥ 3 |
| context | 一路到底 | 每個 commit 之間清空 |
| `/code-review` | 完成後**詢問**使用者 | 不主動，由使用者自行決定 |

**與 Matt 原生 `implement` 的差異**：原生會在做完之後**直接執行** `/code-review`，沒有詢問。本 skill 把那一步改成選擇題，並在使用者要跑時套用瘦身參數（見下）。

## 核心行為規範（最高優先，調用時必須遵守）

- **開場必須先做規模評估**，未評估不得開始實作。
- **一整張 ticket 做完才停。** 中途不逐 commit 徵詢，但每個 commit 仍各自成形、各自 commit。
- **`/code-review` 一律用問的，不得自行執行。**
- **只處理一張 ticket。** 不得跨 ticket 作業。

## 共用規範（必讀）

動手前先讀這兩份，位於 plugin 的 `references/` 目錄（相對本 skill 為 `../../references/`）：

- **`token-discipline.md`** —— 探索優先序（圖譜優先）、三條防呆、回合數成本。**本 skill 一路不清 context，這份的重要性比在 `implement-stepwise` 更高**：一次錯誤的讀取會跟著整張 ticket 的每一個回合重送。
- **`implementation-rules.md`** —— TDD 三條覆寫、程式碼與註解規範、commit 格式、邊做邊記。

其中 **TDD 覆寫 1（seam 只確認一次）** 在本流程的落點是：**規模評估之後、動手之前**一次列出全部 seam 並取得確認，之後不再逐一重問。

## 前置讀取

不要硬編路徑，一律從設定檔取得：

1. `.ai/docs/agents/issue-tracker.md` — ticket 檔案的位置與慣例。找不到時改讀 `docs/agents/issue-tracker.md`；兩者皆無則停下來請使用者先執行 `setup-matt-preset`。
2. `.ai/docs/agents/domain.md` — `CONTEXT.md` 與 ADR 的位置。依它指出的路徑讀取；檔案不存在時**靜默略過**。
3. 目標 ticket 檔案本身。

**不要主動去讀 `spec.md` / `engineering-spec.md`。** ticket 應當自足；真的缺資訊時才回頭撈，並在回報中指出 ticket 哪裡不足。這兩份合計可達 50KB 以上，而本 skill 一路不清 context——讀進來就會跟著每一個回合重送到結束。

## 1. 規模評估（不得跳過）

讀完 ticket 後，先評估它適不適合一次做完，再決定是否繼續。

**出現下列任一情況，停下來建議使用者改走 `implement-stepwise`，或回 `to-tickets` 把票拆小：**

- 驗收條件**超過 12 條**。
- 預估需要**超過 3 個 commit**。
- 預估要修改的**既有**檔案，體積合計超過約 100KB（新建檔案不計——寫新檔遠比讀既有大檔便宜）。
- 需要動到跨越三個以上模組或專案的結構。

評估結果與依據要明講，例如：

```text
規模評估：這張 ticket 有 19 條驗收條件，預估 5 個 commit，
需修改的既有檔案合計約 218KB（含 55KB 的測試檔與 49KB 的 Service）。

超過一次做完的建議上限。建議改用 implement-stepwise，或回 to-tickets 拆成 2–3 張。
要繼續一次做完嗎？
```

**使用者堅持繼續時就照做**，不要反覆勸阻——但要在回報中留下這筆評估，日後對照成本時用得上。

規模在範圍內時，簡短說明評估通過，直接往下走。

## 2. 規劃與 seam 確認

1. **探索 codebase 現況**，依 `token-discipline.md` 的優先序。
2. 列出預計的 commit 切分，以及每個 commit 要測試的 **seam**。
3. 給使用者確認一次——這同時滿足 `/tdd` 的「seam 必須事先確認」要求。
4. 確認後直接往下實作，**不再逐 commit 徵詢**。

這份切分**不寫回 ticket 檔案**。本 skill 一路做完，不需要跨 session 的進度狀態；ticket 的 `## Commit checklist` 章節是 `implement-stepwise` 的機制，兩者不混用。

## 3. 實作

依 `implementation-rules.md` 的 TDD 規範，逐個 commit 完成：

- 每個 commit 走完整的紅綠循環，測試與實作同屬一個 commit。
- 每個 commit 完成後**直接 `git commit`**，不徵詢——這是本 skill 與 `implement-stepwise` 的核心差異。
- **定期執行 typecheck，並跑與當下變更相關的單一測試檔**；不要累積到最後才一次驗證。
- 每個 commit 完成時，依「邊做邊記」記下本次測試與涵蓋的驗收條件。因為不寫回 ticket，**改為在最終回報中一併列出**。

## 4. 收尾

依 `implementation-rules.md` 的「收尾」執行——跑受影響範圍的測試、逐條核對驗收條件、回報、詢問冷眼審查。**不跑完整測試套件**（那是 `to-acceptance-map` 在 branch 結束時的工作），**不判斷 scope creep 或實作對錯**，**不開 sub-agent**，**發現問題不要就地修掉再 commit**——本 skill 會自動 commit，靜默修正會產生使用者沒預期也沒看過的 commit。

回報時一併列出全部 commit 清單，以及每個 commit 的測試與涵蓋的驗收條件（邊做邊記的內容）——本 skill 不寫回 ticket，這份回報是它唯一的留存處。

接著進入下一節的 code-review 詢問。

## 5. code-review 詢問

**問，不要自己跑。** 明確告訴使用者三個選項與各自的取捨：

```text
Ticket 已完成，N 個 commit，受影響範圍測試通過，驗收條件 M/M 達成。
（完整測試套件未跑——整條 branch 結束後由 to-acceptance-map 執行）

要執行 code-review 嗎？
  1. 不跑 —— 你自己讀一次 `git diff <起點>...HEAD`。零 token，但要自己看完 X 行。
  2. 只跑 Standards 軸 —— 抓重複邏輯、命名不一致、變更散落等跨 commit 的結構問題。
  3. 兩軸都跑 —— 另加 Spec 軸，核對交付內容與 ticket 要求是否相符。

  驗收條件已於上一步逐條核對過，Spec 軸的邊際價值因此偏低；
  跨 commit 的結構問題則是逐條核對看不到的，那是 Standards 軸的守備範圍。
```

### 使用者選擇執行時的瘦身參數

原生 `/code-review` 在實測中花掉 **174,646 token**（Standards agent 75,060 + Spec agent 99,586），其中很大一塊是可省的重複工：兩個 sub-agent 各自冷啟動，各自把 `spec.md`(19KB) + `engineering-spec.md`(33KB) + 完整 diff + 一批原始碼重讀一遍——同一份 52KB 的中文規格，那一輪被讀了三次。

調用 `/code-review` 時，在給它的指示中明確加上：

- **Spec 軸的來源用 ticket 檔案，不要用 `spec.md` / `engineering-spec.md`。** ticket 本來就是從規格拆出來的驗收基準，這正是它存在的意義。sub-agent 真的需要回溯原始規格時再去讀。
- **把 `token-discipline.md` 的三條防呆一併傳給 sub-agent**，避免它們重蹈 grep 壓縮檔、`sed -i` 整檔回吐、為 40 行讀 389 行的覆轍。
- **使用者只選 Standards 軸時，明確要求不要啟動 Spec sub-agent。**
- 固定點用本次 ticket 開始前的 commit，不要用整條 branch 的起點——diff 越小，兩個 agent 各付一次的那份就越小。

## 暫停與回報

發現以下任一情況，**暫停並回報**：

- ticket 與實際程式碼不一致，無法照描述實作。
- 實作過程中發現 ticket 的驗收條件本身有矛盾或無法判定。
- 需要大範圍重構才能繼續。
- 實際規模明顯超出開場的評估（例如原估 3 個 commit，做到一半發現要 6 個）——這時應主動提出：是否轉為 `implement-stepwise` 接手剩餘部分。

ticket 的驗收條件有誤時，建議使用者回到 `to-tickets` 修正，不要自行改寫 ticket。

## 下一步引導（純提示，不主動調用）

- **ticket 完成** → 提示清空 context，取下一張 blocker 已滿足的 ticket。
- **整條 branch 的 ticket 全部完成** → 提示開新 session 執行 `to-acceptance-map` 做獨立的驗收覆蓋盤點。
- **中途發現票太重** → 提示改用 `implement-stepwise`，或回 `to-tickets` 拆小。
