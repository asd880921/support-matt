# support-matt

套件版本：`v0.10.0`
更新時間：2026-08-17
安裝教程：[INSTALL.md](./INSTALL.md)
<!-- 版本對齊 plugins/support-matt/.claude-plugin/plugin.json，發版時一併更新此處版本與日期 -->

---

## 介紹

> [!NOTE]
> 目前 Plugin 僅支持 **Claude Code** 與 **Codex** 安裝

強化 [Matt Pocock 的 skills](https://github.com/mattpocock/skills)，提供更完整的落地開發工作流與規格文件生成。

本 plugin 不取代 Matt 的任何 skill，需搭配該套 skill 使用。

## 目前收錄的 skill

| Skill | 用途 |
| --- | --- |
| `setup-matt-preset` | 以個人慣例初始化 Matt skills 的 per-repo 設定：local issue tracker（GitLab 只讀不寫）、產物一律放 `.ai/`、`CLAUDE.md` 改以 `@` 匯入。取代直接調用 `setup-matt-pocock-skills`。 |
| `to-engineering-spec` | 在 `to-spec` 與 `to-tickets` 之間，建立並維護一份正式的開發規格文件 `engineering-spec.md`（系統分析 + 技術設計 + 實作約束）。 |
| `engineering-spec-deliverable` | 把工作版 `engineering-spec.md` 轉成可獨立閱讀、可直接貼上公司 GitLab Issue 的交付版。 |
| `implement-stepwise` | 取代 `implement`（**重票**）：把 ticket 拆成 commit checklist 寫回 ticket，之後一次做一個 commit，經確認後自動 commit 再停。每個 commit 之間清 context。 |
| `implement-oneshot` | 取代 `implement`（**輕票**）：單一 session 做完整張票，形狀貼近原生，但開場先評估規模、收尾**詢問**是否跑 `code-review`（附瘦身參數）。 |
| `implement-checkpoint` | `implement-oneshot` 的變體（**輕票 + 即時插手**）：核心完全相同，只差每個 commit 送出前停下，附完整 commit message 與變更清單等你過目；回「繼續」才提交並接著做下一個 commit。不預先產 commit checklist。 |
| `to-acceptance-map` | branch 開發完畢後於**獨立 session** 盤點測試覆蓋，產出 `acceptance-map.md`。四級判定區分「需補測試」與「不適用測試」，回報只呈現例外。全程唯讀。 |
| `to-change-request` | 開發中途改動的**再入點**：grill 完接這一支，一次做完 `spec.md` delta、`engineering-spec.md` 修訂與追加 ticket，一個確認關卡。純實作的改動直接請你去 implement，不動文件。 |
| `to-code-review` | Matt `code-review` 的**上層入口**：自家 branch 只需給目標分支，規格由 feature 目錄自動取得、`REVIEW.md` 寫回該目錄；代審他人 MR 則另外要背景說明，寫到 `.ai/code-review/`。兩軸結果一律過證據門檻後才輸出 P0–P3 findings。 |

掛載位置：

```
[setup-matt-preset]  ← 每個 repo 第一次使用前跑一次

grill-with-docs → to-spec → [to-engineering-spec] → 人工確認 → to-tickets ─┐
                                                 └─ [engineering-spec-deliverable] 隨時可跑
                                                                          ↓
   單 ticket 拆 Commit (逐筆檢核) → [implement-stepwise]  /  單張 ticket 一次做完 → [implement-oneshot] 或 [implement-checkpoint]（每個 commit 送出前停）
                                                                          ↓
                          [to-engineering-spec 定稿] ← [to-acceptance-map]（branch 完成後，新 session）
                                                                          ↓
                                                              [to-code-review]（發 MR 前最後一關，自行 Code Review 驗證）

開發中途要改：grill-me / grill-with-docs → [to-change-request] → implement (stepwise / oneshot / checkpoint)
代審他人 MR：[to-code-review]（模式 B），與上面的 pipeline 無關
```

三個 implement 取代 Matt 的 `implement`，內部都調用其 `tdd`，都不會自動執行 `code-review`。先依 ticket 規模選重票／輕票（`implement-oneshot` 與 `implement-checkpoint` 開場都會自己評估，過重時建議你換手），輕票再依你想不想在 commit 送出前插手：

| | 事前知道每個 commit 要幹嘛 | commit 送出前停 | ticket 檔案留下 commit 規劃 |
| --- | --- | --- | --- |
| `implement-oneshot` | 否 | 否 | 否 |
| `implement-checkpoint` | 否 | **是** | 否 |
| `implement-stepwise` | 是（checklist 先過目） | 是 | 是 |

實作規範由兩份共用 reference 提供（`plugins/support-matt/references/`），三個 skill 都讀同一份，避免分岔：

| Reference | 內容 |
| --- | --- |
| `token-discipline.md` | 探索優先序（圖譜優先）、三條防呆、回合數成本 |
| `implementation-rules.md` | TDD 三條覆寫、程式碼與註解規範、commit 格式、邊做邊記 |

## 待補

後續規劃中的 skill 與流程調整尚未定案，確定後補上。
