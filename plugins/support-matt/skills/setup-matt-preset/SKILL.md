---
name: setup-matt-preset
description: 以個人慣例初始化 Matt Pocock skills 的 per-repo 設定。承接 Matt 的 setup-matt-pocock-skills 流程，但預先套用三項固定慣例：一律使用 local markdown issue tracker（GitLab 只讀不寫，經 gitlab-issue-fetch）、所有根層產物改放 .ai/ 下一層、Matt 原本要寫進 CLAUDE.md 的 Agent skills 區塊改存為 .ai/docs/agents/workflow.md 並在 CLAUDE.md（及 AGENTS.md，若存在）以 @ 匯入。當使用者要在新專案初始化 Matt workflow 時使用，取代直接調用 setup-matt-pocock-skills。
---

# setup-matt-preset

初始化 Matt Pocock skills 的 per-repo 設定，並自動套用個人慣例。

本 skill **不重寫** Matt 的初始化邏輯。它在執行時讀取 Matt 的 `setup-matt-pocock-skills`，照它的流程走，只是把下列覆寫規則疊上去——這樣 Matt 那邊改版時，本 skill 不會過期。

## 為什麼需要這一層

Matt 的 `setup-matt-pocock-skills` frontmatter 有 `disable-model-invocation: true`，**無法由 model 觸發**，只能由使用者手動輸入斜線指令。因此本 skill 採取「讀取並執行」而非「呼叫」：把它的 SKILL.md 與 seed template 當成待執行的規格讀進來，就地執行。

## 執行流程

### 1. 定位 Matt 的初始化 skill

依序尋找 `setup-matt-pocock-skills` 的所在目錄：

1. `~/.claude/skills/setup-matt-pocock-skills/`
2. `~/.claude/plugins/cache/**/skills/setup-matt-pocock-skills/`
3. 專案內 `.claude/skills/setup-matt-pocock-skills/`

找不到就**停下來**告知使用者 Matt 的 skills 尚未安裝，不要自行從零產生設定檔。

找到後讀取該目錄的 `SKILL.md`，以及流程會用到的 seed template（至少 `issue-tracker-local.md` 與 `domain.md`；`triage-labels.md` 僅在 `triage` skill 已安裝時需要）。

### 2. 照 Matt 的流程執行，並套用下列覆寫

完整依照 Matt `SKILL.md` 的 Process 執行（探索 → 呈現發現並詢問 → 確認草稿 → 寫入 → 完成），但以本節覆寫取代對應段落。**未被覆寫的部分一律照 Matt 原樣**，包含步驟 3 的「先給使用者看草稿、讓他修改後才寫入」——那是人工確認關卡，不得省略。

#### 覆寫 A — Issue tracker（取代 Matt 的 Section A）

**不詢問，直接採用 local markdown**，即使 `git remote` 指向 GitHub 或 GitLab。使用 `issue-tracker-local.md` 這份 seed template。

在寫出的 `issue-tracker.md` 末尾補上一段 GitLab 唯讀規則：

```markdown
## GitLab issue（唯讀）

本 repo 的 issue tracker 是上述 local markdown。當使用者另外提供 GitLab issue 作為需求來源時：

- **只可讀取**：以 `gitlab-issue-fetch` skill 取得 issue 內容與討論串。
- **不可寫入**：不得建立、更新、留言或關閉任何 GitLab issue。所有產出一律寫回本地 markdown。
```

#### 覆寫 B — 所有根層產物下移一層到 `.ai/`

Matt 規範中放在 repo 根目錄的檔案與資料夾，一律改放到 `.ai/` 之下。**只換層級，資料夾命名、檔案命名、存放方式完全照 Matt 的規範**：

| Matt 原路徑 | 本 preset 路徑 |
| --- | --- |
| `.scratch/<feature>/` | `.ai/.scratch/<feature>/` |
| `docs/agents/issue-tracker.md` | `.ai/docs/agents/issue-tracker.md` |
| `docs/agents/domain.md` | `.ai/docs/agents/domain.md` |
| `docs/agents/triage-labels.md` | `.ai/docs/agents/triage-labels.md` |
| `docs/adr/` | `.ai/docs/adr/` |
| `CONTEXT.md` | `.ai/CONTEXT.md` |
| `CONTEXT-MAP.md` | `.ai/CONTEXT-MAP.md` |

**這條最容易漏的地方：seed template 的正文裡也寫滿了路徑。** `issue-tracker-local.md` 通篇引用 `.scratch/...`，`domain.md` 引用 `CONTEXT.md`、`docs/adr/` 並畫了目錄樹。寫出檔案時，**必須把 template 內文的每一處路徑一併改寫**，否則下游 skill 讀到的仍是舊路徑，整套設定等於沒生效。改寫後請自行複查一遍：寫出的檔案內不應再出現任何不帶 `.ai/` 前綴的 `.scratch/`、`docs/agents/`、`docs/adr/`、根層 `CONTEXT.md` 或 `CONTEXT-MAP.md`。

**例外**：multi-context 佈局中位於 `src/<context>/docs/adr/` 的 context 專屬 ADR **維持原位不動**——本規則搬移的是「落在專案根目錄」的產物，這些本來就在 `src/` 底下。若使用者希望連這些也搬，會另行說明。

#### 覆寫 C — CLAUDE.md 改為 `@` 匯入

Matt 步驟 4 是把 `## Agent skills` 區塊直接寫進 `CLAUDE.md` 或 `AGENTS.md`。改為：

1. 把該區塊的**內容**寫成獨立檔案 `.ai/docs/agents/workflow.md`（區塊內引用的三個路徑同樣要套用覆寫 B）。
2. 在專案根目錄的 `CLAUDE.md` 補上一行匯入：

   ```markdown
   @.ai/docs/agents/workflow.md
   ```

3. **若 `AGENTS.md` 也存在，同一行一併補上。** 這一點刻意偏離 Matt 步驟 4 的「二選一、不得同時寫入」規則——兩份都存在時兩份都要補。
4. 若 `CLAUDE.md` 與 `AGENTS.md` 皆不存在，建立 `CLAUDE.md` 並寫入該匯入行（Matt 原本要求詢問使用者選哪一個；本 preset 已指定 `CLAUDE.md`）。

已存在該匯入行時不要重複追加。補上匯入行以外，不改動 `CLAUDE.md` / `AGENTS.md` 的其他內容。

#### 沿用 Matt 原樣的部分

- **Section B（triage 標籤）**：照 Matt 的規則——`triage` skill 未安裝就整段跳過，已安裝則沿用五個預設標籤。
- **Section C（domain docs）**：照 Matt 的規則——預設 single-context，僅在探索到 monorepo 訊號時才詢問 multi-context。路徑套用覆寫 B。

### 3. 完成回報

除了 Matt 步驟 5 的完成訊息之外，額外列出：

- 實際寫出的檔案完整路徑清單（讓使用者一眼確認全部落在 `.ai/` 下）。
- `CLAUDE.md` / `AGENTS.md` 各自是否已補上匯入行。

## 執行限制

- 本 skill 只做 per-repo 設定初始化，**不建立任何 issue、spec 或 ticket**，也不修改任何功能程式碼。
- 找不到 Matt 的 skills 時停止，不自行從零產生設定檔——那會產出與 Matt 規範不同步的內容。
- 重複執行時，就地更新既有設定檔，不重複追加區塊。
