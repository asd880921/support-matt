# support-matt

套件版本：`v0.8.0`
更新時間：2026-08-06
安裝教程：[INSTALL.md](./INSTALL.md)
<!-- 版本對齊 plugins/support-matt/.claude-plugin/plugin.json，發版時一併更新此處版本與日期 -->

---

## 介紹

> [!NOTE]
> 目前 Plugin 僅支持 **Claude Code** 與 **Codex** 安裝

掛在 [Matt Pocock 的 skills](https://github.com/mattpocock/skills) 上運作的補強 skill 集：沿用 Matt 的流程骨架，補上公司實務需要、但 Matt 流程未涵蓋的環節。

本 plugin 不取代 Matt 的任何 skill，需搭配該套 skill 使用。

## 目前收錄的 skill

| Skill | 用途 |
| --- | --- |
| `setup-matt-preset` | 以個人慣例初始化 Matt skills 的 per-repo 設定：local issue tracker（GitLab 只讀不寫）、產物一律放 `.ai/`、`CLAUDE.md` 改以 `@` 匯入。取代直接調用 `setup-matt-pocock-skills`。 |
| `to-engineering-spec` | 在 `to-spec` 與 `to-tickets` 之間，建立並維護一份正式的開發規格文件 `engineering-spec.md`（系統分析 + 技術設計 + 實作約束）。 |
| `engineering-spec-deliverable` | 把工作版 `engineering-spec.md` 轉成可獨立閱讀、可直接貼上公司 GitLab Issue 的交付版。 |
| `implement-stepwise` | 取代 `implement`（**重票**）：把 ticket 拆成 commit checklist 寫回 ticket，之後一次做一個 commit，經確認後自動 commit 再停。每個 commit 之間清 context。 |
| `implement-oneshot` | 取代 `implement`（**輕票**）：單一 session 做完整張票，形狀貼近原生，但開場先評估規模、收尾**詢問**是否跑 `code-review`（附瘦身參數）。 |
| `to-acceptance-map` | branch 開發完畢後於**獨立 session** 盤點測試覆蓋，產出 `acceptance-map.md`。四級判定區分「需補測試」與「不適用測試」，回報只呈現例外。全程唯讀。 |
| `to-change-request` | 開發中途改動的**再入點**：grill 完接這一支，一次做完 `spec.md` delta、`engineering-spec.md` 修訂與追加 ticket，一個確認關卡。純實作的改動直接請你去 implement，不動文件。 |

掛載位置：

```
[setup-matt-preset]  ← 每個 repo 第一次使用前跑一次

grill-with-docs → to-spec → [to-engineering-spec] → 人工確認 → to-tickets ─┐
                                                 └─ [engineering-spec-deliverable] 隨時可跑
                                                                          ↓
                                          重票 → [implement-stepwise]  ／  輕票 → [implement-oneshot]
                                                                          ↓
                          [to-engineering-spec 定稿] ← [to-acceptance-map]（branch 完成後，新 session）

開發中途要改：grill-me / grill-with-docs → [to-change-request] → implement
```

兩個 implement 取代 Matt 的 `implement`，內部都調用其 `tdd`，都不會自動執行 `code-review`。選哪一個依 ticket 規模——`implement-oneshot` 開場會自己評估，過重時建議你換手。

實作規範由兩份共用 reference 提供（`plugins/support-matt/references/`），兩個 skill 都讀同一份，避免分岔：

| Reference | 內容 |
| --- | --- |
| `token-discipline.md` | 探索優先序（圖譜優先）、三條防呆、回合數成本 |
| `implementation-rules.md` | TDD 三條覆寫、程式碼與註解規範、commit 格式、邊做邊記 |

## 待補

後續規劃中的 skill 與流程調整尚未定案，確定後補上。
