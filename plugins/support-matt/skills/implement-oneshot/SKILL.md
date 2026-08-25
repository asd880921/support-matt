---
name: implement-oneshot
description: 在單一 session 內把一整張 ticket 做完，形狀貼近 Matt 原生的 implement，但有三處差異：開場先評估 ticket 規模，過重時建議回 to-tickets 拆小；ticket 收尾（跑受影響範圍測試、逐條核對驗收條件、寫回 ticket）完成後即結束，完全不執行也不引導 code-review；執行時套用 token 紀律。適合邊界明確、預估 commit 三個以內的較小 ticket。當使用者要一次做完一張小票、不需要逐 commit 停下審查時使用。
---

# implement-oneshot

在**單一 session** 內把一整張 ticket 做完，收尾完就結束。

本 skill 與 `implement-stepwise` 是同一套流程的兩種模式，共用實作規範，差別在 commit 的送出方式與規模上限：

| | `implement-oneshot` | `implement-stepwise` |
| --- | --- | --- |
| 執行單位 | 一整張 ticket | 一整張 ticket |
| 人工關卡 | ticket 完成後一次 | 每個 commit 送出前 |
| commit 送出方式 | 自動提交 | 確認後才提交 |
| 規模上限 | 12 條驗收 / 3 commit / 100KB | 18 條驗收 / 5 commit / 150KB |
| context | 一路到底 | 一路到底 |
| `/code-review` | 不執行、不引導 | 不執行、不引導 |

**與 Matt 原生 `implement` 的差異**：原生會在做完之後**直接執行** `/code-review`。本 skill 把那一步整段拿掉——收尾已經對本次 task 做過一輪驗收，單張票再跑一次審查是重複工；審查的位置在整條 branch 做完之後的 `to-code-review`。真的要對單一 task 跑審查時，改調用 Matt 原生的 `implement`。

想保留本 skill 的節奏、但每個 commit 送出前要看一眼的，用 `implement-stepwise`——核心與本 skill 相同，只把 commit 從自動送出改成確認後送出；因為多了這道關卡，它的規模上限也比本 skill 寬一級。票超出本 skill 的上限、但還在 stepwise 上限內時，優先改用 `implement-stepwise`，不必急著回 `to-tickets` 拆票。

## 核心行為規範（最高優先，調用時必須遵守）

- **開場必須先做規模評估**，未評估不得開始實作。
- **一整張 ticket 做完才停。** 中途不逐 commit 徵詢，但每個 commit 仍各自成形、各自 commit。
- **不執行 `/code-review`，也不詢問、不提示、不列選項。** 收尾結束就是本 skill 的終點。
- **只處理一張 ticket。** 不得跨 ticket 作業。

## 共用規範（必讀）

動手前先讀這兩份，位於 plugin 的 `references/` 目錄（相對本 skill 為 `../../references/`）：

- **`token-discipline.md`** —— 探索優先序（圖譜優先）、三條防呆、回合數成本。**本 skill 一路不清 context**：一次錯誤的讀取會跟著整張 ticket 的每一個回合重送。
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

**出現下列任一情況，停下來建議使用者回 `to-tickets` 把票拆小：**

- 驗收條件**超過 12 條**。
- 預估需要**超過 3 個 commit**。
- 預估要修改的**既有**檔案，體積合計超過約 100KB（新建檔案不計——寫新檔遠比讀既有大檔便宜）。
- 需要動到跨越三個以上模組或專案的結構。

評估結果與依據要明講，例如：

```text
規模評估：這張 ticket 有 19 條驗收條件，預估 5 個 commit，
需修改的既有檔案合計約 218KB（含 55KB 的測試檔與 49KB 的 Service）。

超過一次做完的建議上限。建議回 to-tickets 拆成 2–3 張。
要繼續一次做完嗎？
```

**使用者堅持繼續時就照做**，不要反覆勸阻——但要在回報中留下這筆評估，日後對照成本時用得上。

規模在範圍內時，簡短說明評估通過，直接往下走。

## 2. 規劃與 seam 確認

1. **探索 codebase 現況**，依 `token-discipline.md` 的優先序。
2. 列出預計的 commit 切分，以及每個 commit 要測試的 **seam**。
3. 給使用者確認一次——這同時滿足 `/tdd` 的「seam 必須事先確認」要求。
4. 確認後直接往下實作，**不再逐 commit 徵詢**。

**commit 切分**不寫回 ticket 檔案。本 skill 一路做完，不需要跨 session 的進度狀態。（收尾階段的驗收核對結果仍要寫回 ticket，見第 4 節。）

## 3. 實作

依 `implementation-rules.md` 的 TDD 規範，逐個 commit 完成：

- 每個 commit 走完整的紅綠循環，測試與實作同屬一個 commit。
- 每個 commit 完成後**直接 `git commit`**，不徵詢——這是本 skill 與 `implement-stepwise` 的核心差異。
- **定期執行 typecheck，並跑與當下變更相關的單一測試檔**；不要累積到最後才一次驗證。
- 每個 commit 完成時，依「邊做邊記」記下本次測試與涵蓋的驗收條件，收尾時併入寫回 ticket 的核對表與最終回報。

## 4. 收尾

依 `implementation-rules.md` 的「收尾」執行——跑受影響範圍的測試、逐條核對驗收條件、**把核對結果寫回 ticket**（勾選達成項 + append 帶證據的核對表）、回報。**不詢問冷眼審查**（覆寫 `implementation-rules.md` 收尾步驟 5——那一步在本 skill 不執行），**不跑完整測試套件**（那是 `to-acceptance-map` 在 branch 結束時的工作），**不判斷 scope creep 或實作對錯**，**不開 sub-agent**，**發現問題不要就地修掉再 commit**——本 skill 會自動 commit，靜默修正會產生使用者沒預期也沒看過的 commit。

**寫回 ticket 對本 skill 特別重要。** 核對結果若只留在對話裡，這張票在檔案上就完全沒有交付紀錄。因此在核對表的「依據」欄一併帶入各 commit 的測試名稱（邊做邊記的內容），讓 ticket 自己說得出這張票交付了什麼、由什麼證明。

回報時一併列出全部 commit 清單。

**回報完就停。** 不追問下一步、不提 code-review、不建議任何後續審查動作。

## 暫停與回報

發現以下任一情況，**暫停並回報**：

- ticket 與實際程式碼不一致，無法照描述實作。
- 實作過程中發現 ticket 的驗收條件本身有矛盾或無法判定。
- 需要大範圍重構才能繼續。
- 實際規模明顯超出開場的評估（例如原估 3 個 commit，做到一半發現要 6 個）——這時應主動提出：是否回 `to-tickets` 把剩餘的部分拆成獨立的票。

ticket 的驗收條件有誤時，建議使用者回到 `to-tickets` 修正，不要自行改寫 ticket。

## 下一步引導（純提示，不主動調用）

- **ticket 完成** → 提示清空 context，取下一張 blocker 已滿足的 ticket。
- **整條 branch 的 ticket 全部完成** → 提示開新 session 執行 `to-acceptance-map` 做獨立的驗收覆蓋盤點。
- **中途發現票太重** → 提示回 `to-tickets` 拆小。
