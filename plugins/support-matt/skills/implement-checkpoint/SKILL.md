---
name: implement-checkpoint
description: 在單一 session 內把一整張 ticket 做完，核心與 implement-oneshot 完全一致（開場規模評估、一次確認全部 seam、TDD、收尾寫回 ticket、收尾後不跑也不引導 code-review），只差一件事：每個 commit 送出之前停下來，附上完整的 commit message 與變更清單等使用者過目，使用者回「繼續」才由本 skill 執行 git commit 並接著做下一個 commit，直到下一個 commit 前再停。不預先產出 commit checklist，也不把切分寫回 ticket——使用者不需要提前知道每個 commit 要幹嘛，只需要在送出前有機會插手。適合邊界明確、預估 commit 三個以內的較小 ticket。當使用者要一次做完一張小票、但希望每個 commit 送出前都能看一眼、必要時即時調整時使用。
---

# implement-checkpoint

在**單一 session** 內把一整張 ticket 做完，但**每個 commit 送出之前停下來讓人過目**，使用者回「繼續」才提交並接續下一個 commit。

本 skill 是 `implement-oneshot` 的變體：**核心流程、規範、收尾全部相同**，唯一的差別是 commit 的送出方式從「自動」改成「先停下、確認後才送」。

三者的關係：

| | `implement-oneshot` | `implement-checkpoint` | `implement-stepwise` |
| --- | --- | --- | --- |
| 執行單位 | 一整張 ticket | 一整張 ticket | 一個 commit |
| 人工關卡 | ticket 完成後一次 | 每個 commit 送出前 | 每個 commit 之前 |
| 事前知道每個 commit 要做什麼 | 否 | 否 | 是（checklist 經確認並寫回 ticket） |
| ticket 檔案留下 commit 規劃 | 否 | 否 | 是 |
| context | 一路到底 | 一路到底 | 每個 commit 之間，由使用者決定要不要清 |
| `/code-review` | 不執行、不引導 | 不執行、不引導 | 不主動 |

**收尾之後就結束，沒有 code-review 這一步。** 收尾已經對本次 task 做過一輪驗收，單張票再跑一次審查是重複工；審查的位置在整條 branch 做完之後的 `to-code-review`。真的要對單一 task 跑審查時，改調用 Matt 原生的 `implement`。

**與 `implement-stepwise` 的差別要說清楚**：stepwise 的關卡是「事前審查規劃」加「逐 commit 審查」，使用者要先看過並確認整份 commit checklist，且每輪之間預期清 context。本 skill **不做事前規劃審查、不產出 checklist、不寫任何東西回 ticket 當進度狀態**——關卡只有一個，就在 commit 送出前的那一刻，讓使用者看一眼這次要提交什麼、commit message 寫得對不對，需要時即時插手。

## 核心行為規範（最高優先，調用時必須遵守）

- **開場必須先做規模評估**，未評估不得開始實作。
- **任何情況下都不得直接 `git commit`。** 每個 commit 都必須先停下回報並取得使用者確認。這是本 skill 與 `implement-oneshot` 的唯一差異，也是它存在的理由——違反這條，本 skill 就等同 `implement-oneshot`。
- **使用者確認後不再多問。** 執行 commit，直接接續實作下一個 commit，做到下一個 commit 送出前再停。**不在 commit 之後另外徵詢要不要繼續。**
- **不產出 commit checklist、不寫進度狀態回 ticket。** 那是 `implement-stepwise` 的機制。（收尾的驗收核對結果仍要寫回 ticket，見第 5 節。）
- **不執行 `/code-review`，也不詢問、不提示、不列選項。** 收尾結束就是本 skill 的終點；commit 關卡拿到的「繼續」更不構成執行審查的授權。
- **只處理一張 ticket。** 不得跨 ticket 作業。

## 共用規範（必讀）

動手前先讀這兩份，位於 plugin 的 `references/` 目錄（相對本 skill 為 `../../references/`）：

- **`token-discipline.md`** —— 探索優先序（圖譜優先）、三條防呆、回合數成本。**本 skill 一路不清 context**：一次錯誤的讀取會跟著整張 ticket 的每一個回合重送。
- **`implementation-rules.md`** —— TDD 三條覆寫、程式碼與註解規範、commit 格式、邊做邊記。

其中 **TDD 覆寫 1（seam 只確認一次）** 在本流程的落點是：**規模評估之後、動手之前**一次列出全部 seam 並取得確認，之後不再逐一重問。**commit 關卡不是重問 seam 的地方**——那裡只審這次要提交的東西。

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
4. 確認後直接往下實作。

**這一步的產出只留在對話裡，不寫回 ticket 檔案。** 目的是取得 seam 的確認，不是要使用者事先核准每個 commit 的內容；實際實作時切分若有合理調整（仍在原範圍與目的內），照調整後的做，並在該次的 commit 關卡說明差異即可。

## 3. 實作

依 `implementation-rules.md` 的 TDD 規範，逐個 commit 完成：

- 每個 commit 走完整的紅綠循環，測試與實作同屬一個 commit。
- **定期執行 typecheck，並跑與當下變更相關的單一測試檔**；不要累積到最後才一次驗證。
- 每個 commit 的變更就緒後，**不要 commit**，改走下一節的關卡。

## 4. commit 關卡（本 skill 的核心）

每個 commit 的變更就緒、typecheck 與相關測試通過後，**停下來回報並等待使用者確認**。回報固定包含四項：

````md
### commit N/? 待確認

**本次變更**
- `path/to/File.cs` — 一行說明
- `path/to/FileTests.cs` — 一行說明
（應排除但工作區內有的檔案，也在此點名）

**驗證**
typecheck 通過；`XxxTests` 全綠（新增 3 條）。

**涵蓋的驗收條件**
#3、#4

**commit message**
```
feat: 新增 X 的查詢路徑
```

確認後我會執行 commit 並直接接著做下一個 commit。
要調整 commit message 或變更內容，現在講。
````

**判斷這是不是最後一個 commit**，是的話把最後兩行換成：

````md
**這是這張 ticket 的最後一個 commit。** 確認後我會執行 commit，接著進入收尾：
跑受影響範圍的測試、逐條核對驗收條件並把核對結果寫回 ticket，然後結束。
要調整 commit message 或變更內容，現在講——收尾之後的修正會變成另一個 commit。
````

規則：

- **commit message 要給完整可直接使用的一整段**，格式依 `implementation-rules.md` 的 Commit 規範（type 前綴英文、描述繁中、有 issue 編號就帶）。不要只給摘要或口頭描述——使用者要能直接看出即將寫進 repo 的是哪一行字。
- **`git add` 也一起等**。關卡之前不要動暫存區，避免使用者以為只是預覽卻已經動了 git 狀態。
- **只在這裡設一個關卡。** 不要在實作中途、寫完測試、跑完 typecheck 等處另外停下來徵詢。
- **不附上「要不要繼續」的選項題。** 使用者的答案只有兩種：確認（繼續）或提出調整。
- **最後一個 commit 必須講明它是最後一個，以及接下來要進收尾。** 使用者在這個關卡的判斷跟前面幾個不同——後面沒有下一次插手機會了，`git commit` 之後就直接開始跑測試與核對驗收條件。不預告等於把這件事藏起來。不確定是不是最後一個時就不要宣稱，照一般格式走。

使用者的回應處理：

- **回「繼續」或任何等同的確認** → 執行 `git add` 與 `git commit`，**接著直接開始下一個 commit 的實作，不再徵詢**；做到下一個 commit 就緒時再回到本節停下。這是本 skill 的循環。
- **提出調整**（改 commit message、加減檔案、改實作） → 就地處理，處理完**重新走一次本節的關卡**，不要改完就自己提交。
- **要求先不提交、去做別的事** → 照做，變更留在工作區，不要自作主張 commit。
- **回應不明確** → 當作「尚未確認」，不要提交；問清楚要調整什麼。

最後一個 commit 的關卡照走，確認提交後**不再徵詢，直接進入第 5 節的收尾**——收尾本身就是既有流程的一部分，不是需要另外核准的新動作。

## 5. 收尾

依 `implementation-rules.md` 的「收尾」執行——跑受影響範圍的測試、逐條核對驗收條件、**把核對結果寫回 ticket**（勾選達成項 + append 帶證據的核對表）、回報。**不詢問冷眼審查**（覆寫 `implementation-rules.md` 收尾步驟 5——那一步在本 skill 不執行），**不跑完整測試套件**（那是 `to-acceptance-map` 在 branch 結束時的工作），**不判斷 scope creep 或實作對錯**，**不開 sub-agent**。

**寫回 ticket 對本 skill 特別重要。** 它和 `implement-oneshot` 一樣不在 ticket 留下 Commit checklist，若核對結果也只留在對話裡，這張票在檔案上就完全沒有交付紀錄。因此在核對表的「依據」欄一併帶入各 commit 的測試名稱（邊做邊記的內容），讓 ticket 自己說得出這張票交付了什麼、由什麼證明。

收尾階段測試失敗或有驗收條件未達成時，**先回報，不要自己修掉**。使用者要修的話，那份修正也是一個 commit，照樣走第 4 節的關卡；把核對表寫回 ticket 的動作，等修正提交後再做，避免留下與程式碼不符的紀錄。

回報時一併列出全部 commit 清單（含各自的 commit message）。

**回報完就停。** 不追問下一步、不提 code-review、不建議任何後續審查動作。

## 暫停與回報

發現以下任一情況，**暫停並回報**：

- ticket 與實際程式碼不一致，無法照描述實作。
- 實作過程中發現 ticket 的驗收條件本身有矛盾或無法判定。
- 需要大範圍重構才能繼續。
- 實際規模明顯超出開場的評估（例如原估 3 個 commit，做到一半發現要 6 個）——這時應主動提出：是否轉為 `implement-stepwise` 接手剩餘部分。

ticket 的驗收條件有誤時，建議使用者回到 `to-tickets` 修正，不要自行改寫 ticket。

**已提交的 commit 不回頭改。** 使用者在後面的關卡才發現前一個 commit 有問題時，建議以新的 commit 修正（同樣走第 4 節的關卡），不要 amend 或 reset——除非使用者明確要求。

## 下一步引導（純提示，不主動調用）

- **ticket 完成** → 提示清空 context，取下一張 blocker 已滿足的 ticket。
- **整條 branch 的 ticket 全部完成** → 提示開新 session 執行 `to-acceptance-map` 做獨立的驗收覆蓋盤點。
- **中途發現票太重** → 提示改用 `implement-stepwise`，或回 `to-tickets` 拆小。
- **覺得每個 commit 都停太煩** → 提示改用 `implement-oneshot`，流程完全相同，只是不停。
