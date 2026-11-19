# AGY Subagent Delegation

讓 Antigravity（AGY）主代理透過全域 `GEMINI.md` 的一句提示讀取個人全域 skill，在 AGY 內部提供兩項功能：

1. **委派（Worker）**：透過 AGY 內建 `swarm`（`gemini-4-neon-medium`，必要時升 `-high`）把任務交給 Gemini 4 Neon。只在能省下主代理 context 或時間、且驗證成本低於自己做時才委派。
2. **雙重第二意見（Advisors）**：執行設計、架構變更、要 merge 的程式碼或根因修正的計畫前，透過 `agy` CLI 同時平行詢問 `claude-opus-5-5-low` 與 `gemini-4-argon-medium`，取得兩份獨立審查意見。

`gemini-4-neon` 是為 Gemini 4 Neon 主代理設計的最新版本；舊版 `gemini-4-argon`（Gemini 4 Argon 主代理、Gemini 4 Hydrogen worker、單一 Opus 5.5 advisor）亦保留於 repo 中。Claude Code 版見 [lite-claude-subagent-delegation](https://github.com/workingyuanyuan/lite-claude-subagent-delegation)，Codex 版見 [lite-codex-subagent-delegation](https://github.com/workingyuanyuan/lite-codex-subagent-delegation)。

## 檔案

```text
gemini-4-neon/                 # 最新版（Gemini 4 Neon 主代理）
  GEMINI.md
  MODEL-SELECTION.md
  skills/
    delegate/
      SKILL.md

gemini-4-argon/                # 舊版（Gemini 4 Argon 主代理）
  GEMINI.md
  MODEL-SELECTION.md
  skills/
    delegate/
      SKILL.md
```

- [GEMINI.md](gemini-4-neon/GEMINI.md)：加入個人全域規則的英文入口句。
- [SKILL.md](gemini-4-neon/skills/delegate/SKILL.md)：英文委派與雙 advisor 策略、呼叫方式。
- [MODEL-SELECTION.md](gemini-4-neon/MODEL-SELECTION.md)：選模型的依據與實測紀錄。只在升級 skill 時讀，不安裝。

## 前置需求

- [Antigravity CLI](https://antigravity.google/docs/cli/reference/) `agy`（`gemini-4-neon` 已驗證 `1.5.12`；`gemini-4-argon` 已驗證 `1.5.3`），已登入。
- 支援 WSL / Linux（bash）與 Windows（PowerShell 5.1 / 7）雙環境。Worker 直接使用 AGY 內建 `swarm` 工具；Advisors 的共用 brief 由 `write_file` 寫出並以檔案路徑傳給 `agy -p`（WSL / Linux 若 `PATH` 未含 `~/.local/bin` 則使用 `~/.local/bin/agy`，沙盒隱藏時以 `BypassSandbox` 執行），輸出直接由 `run_command` / task log 擷取，不需檔案重導向。關閉 stdin 的寫法依 shell 而異：bash / WSL 於指令結尾加 `</dev/null`，PowerShell 改用 `$null | agy ...`（Windows 端尚未實測驗證）。

## 手動安裝

在每個要使用的環境（Windows、WSL）各自安裝一次。以下以最新版 `gemini-4-neon` 為例（若主代理使用 Gemini 4 Argon，請改複製 `gemini-4-argon/skills/delegate`）：

### 1. 複製 skill 資料夾

將 `gemini-4-neon/skills/delegate` 整個資料夾複製到 AGY 的全域技能目錄：

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

## 策略（`gemini-4-neon`）

| 角色 | 模型 / 呼叫方式 | 用途 |
| --- | --- | --- |
| Worker | `gemini-4-neon-medium` / `-high`（AGY 內建 `swarm`） | 主代理拆解問題並設計好介面後，平行委派範圍明確的任務：快速程式碼探索、獨立 leaf module 遷移、機械性編輯、樣板測試、高頻檢查，以及圖片／PDF／影音原始資訊抽取 |
| Advisor 1 | `claude-opus-5-5-low`（`agy` CLI，平行執行） | 執行設計、架構變更、要 merge 的程式碼或根因修正計畫前的跨模型獨立第二意見 |
| Advisor 2 | `gemini-4-argon-medium`（`agy` CLI，平行執行） | 與 Opus 5.5 同步讀取同一份 brief，提供獨立的高階 Gemini 審查第二意見 |

- **Worker 隔離與清理**：唯讀任務使用 `workspace: "inherit"` 並在 brief 中禁止改檔；平行寫檔任務使用 `workspace: "share"`（自動建立獨立 git worktree，完成驗收與合併後以 `swarm(action: "kill")` 清理）。Worker 完成時透過 `@coordinator` 喚醒主代理並以 `read_board` 收取結果。因 Neon、Hydrogen 與 Flash 共用配額池，遇 `RESOURCE_EXHAUSTED` 不退回 Hydrogen / Flash。
- **雙 Advisor 獨立盲審與唯讀保護**：兩位 advisor 共用同一份存於 `~/.local/state/delegation-log/<專案>/` 的 `-advisor.brief.md`（不提及另一位 advisor），於同回合以兩個背景 `run_command` 平行啟動並等待兩者皆完成後才整合。因 `agy` 沒有可靠的唯讀模式，在 brief 中寫明唯讀，並於 git 專案執行前後（含單邊重試）比對 `git status --porcelain` 與 `git diff`。任一邊遇額度耗盡（`RESOURCE_EXHAUSTED`）或重試失敗時回報該 advisor 不可用並以另一邊繼續。
- **驗收與整合**：所有結果一律視為未驗證，主代理負責最終驗收與整合並標註各點來源（Opus vs Argon）；任一 advisor 與主計畫在影響計畫走向的關鍵處衝突，或兩位 advisor 在關鍵處嚴重分歧時，回報雙方理由給使用者。

## 授權

[MIT](LICENSE)
