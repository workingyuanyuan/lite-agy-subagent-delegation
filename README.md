# AGY Subagent Delegation

讓 Antigravity（AGY）主代理透過全域 `GEMINI.md` 的一句提示讀取個人全域 skill，在 AGY 內部提供兩項功能：

1. **委派（Worker）**：透過 AGY 內建 `invoke_subagent`（`Model: "flash"`）把任務交給 Gemini 3.8 Flash。只在能省下主代理 context 或提升品質、且驗證成本低於自己做時才委派。
2. **第二意見（Advisor）**：執行設計、架構變更、要 merge 的程式碼或根因修正的計畫前，透過 `agy` CLI 詢問 `claude-opus-5-5-medium`。

`gemini-4-argon` 是為 Gemini 4 Argon 主代理設計的版本。Claude Code 版見 [lite-claude-subagent-delegation](https://github.com/workingyuanyuan/lite-claude-subagent-delegation)，Codex 版見 [lite-codex-subagent-delegation](https://github.com/workingyuanyuan/lite-codex-subagent-delegation)。

## 檔案

```text
gemini-4-argon/
  GEMINI.md
  MODEL-SELECTION.md
  skills/
    delegate/
      SKILL.md
```

- [GEMINI.md](gemini-4-argon/GEMINI.md)：加入個人全域規則的英文入口句。
- [SKILL.md](gemini-4-argon/skills/delegate/SKILL.md)：英文委派與 advisor 策略、呼叫方式。
- [MODEL-SELECTION.md](gemini-4-argon/MODEL-SELECTION.md)：選模型的依據與實測紀錄。只在升級 skill 時讀，不安裝。

## 前置需求

- [Antigravity CLI](https://antigravity.google/docs/cli/reference/) `agy`（已驗證 1.5.3），已登入。
- 支援 Windows（PowerShell 5.1 / 7）與 WSL / Linux（bash）雙環境。Worker 直接使用 AGY 內建 `invoke_subagent` 工具；Advisor 的 brief 由 `write_file` 寫出並以檔案路徑傳給 `agy -p`，輸出直接由 `run_command` 擷取，指令在 Windows PowerShell 與 WSL bash 下完全相同。

## 手動安裝

在每個要使用的環境（Windows、WSL）各自安裝一次。

### 1. 複製 skill 資料夾

將 `gemini-4-argon/skills/delegate` 整個資料夾複製到 AGY 的全域技能目錄：

| 系統 | 全域技能目錄 |
| --- | --- |
| Windows | `%USERPROFILE%\.gemini\config\skills` |
| WSL / Linux | `~/.gemini/config/skills` |

完成後應為 `~/.gemini/config/skills/delegate/SKILL.md`。

### 2. 加入全域入口句

在 `~/.gemini/GEMINI.md`（Windows：`%USERPROFILE%\.gemini\GEMINI.md`）加入：

```text
Before executing a plan for a design, architecture change, merge-bound code, or root-cause fix, and whenever delegating would save your context or improve quality, read and follow the delegate skill.
```

若已有其他委派 skill 的入口句，改成上句。

### 3. 確認載入

開啟新對話，確認 Skills & Customizations 中出現 `delegate`，再以明確要求檢查：

```text
請讀取 delegate skill，說明你讀取的 SKILL.md 路徑、委派對象，以及何時詢問 advisor。
```

## 策略

| 角色 | 模型 / 呼叫方式 | 用途 |
| --- | --- | --- |
| Worker | `flash`（AGY 內建 `invoke_subagent`） | 主代理拆解問題並設計好介面後，平行委派範圍明確的任務：快速程式碼探索、獨立 leaf module 遷移、機械性編輯、樣板測試、高頻檢查，以及圖片／PDF／影音原始資訊抽取 |
| Advisor | `claude-opus-5-5-medium`（`agy` CLI） | 執行設計、架構變更、要 merge 的程式碼或根因修正計畫前的第二意見 |

- **Worker 隔離與清理**：唯讀離線調查用內建 `research`；寫檔任務以 `define_subagent` 定義無子代理權限的 writer，平行寫檔用 `Workspace: "share"`（完成驗收與合併後，以 `manage_subagents` 清理）。
- **Advisor 唯讀保護**：`agy` 沒有可靠的唯讀模式，因此在 brief（存於 `~/.local/state/delegation-log/<專案>/`）中寫明唯讀，並在 git 專案中於執行前後比對 `git status --porcelain` 與 `git diff`。若遇額度耗盡（`RESOURCE_EXHAUSTED`）則回報 Opus 意見不可用並繼續。
- **驗收與整合**：所有結果一律視為未驗證，主代理負責最終驗收與整合；advisor 與主代理計畫在關鍵處意見衝突時，回報給使用者。

## 授權

[MIT](LICENSE)
