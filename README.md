# support-matt

套件版本：`v0.2.0`
更新時間：2026-08-05
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

掛載位置：

```
[setup-matt-preset]  ← 每個 repo 第一次使用前跑一次

grill-with-docs → to-spec → [to-engineering-spec] → 人工確認 → to-tickets → implement
                                                 └─ [engineering-spec-deliverable] 隨時可跑
```

## 待補

後續規劃中的 skill 與流程調整尚未定案，確定後補上。
