# delegate skill：選模型與呼叫方式的依據

升級 `SKILL.md` 時才讀。skill 本體不引用這份檔案。

## 適用範圍
- harness：Antigravity（AGY）；主代理是 Gemini 4 Neon。
- 一個 harness 一個 repo（Claude Code 版：`lite-claude-subagent-delegation`；Codex 版：`codex-subagent-delegation` / `lite-codex-subagent-delegation`）。repo 內以主代理模型命名版本資料夾，主代理換模型時另開資料夾（前版為 `gemini-4-argon/`）。
- 這份檔案放在版本資料夾根目錄，不在 `skills/delegate/` 裡，安裝時不會被複製進 skill。

## 名冊與架構決定（2026-11-19 建立，與使用者確認並經 Opus 5.5 審查）
- **範圍完全收斂在 AGY 內**：不依賴 Codex CLI 或 Claude Code CLI，只使用 AGY 內建 `swarm` 與 `agy` CLI。
- **Worker 改為 Gemini 4 Neon（透過 AGY 內建 `swarm`，預設 `gemini-4-neon-medium`，困難或驗收失敗時升 `gemini-4-neon-high`），不再委派 Gemini 4 Hydrogen**：
  - 2026-11-19 實測 `swarm(action: "models", all: true)` 確認：`gemini-4-neon-*`、`gemini-4-hydrogen-*` 與 `gemini-3.8-flash-*` 共用同一快速重置配額池（剩餘比例與重置時間相同）。
  - 既然配額池相同，改用能力較強的 `gemini-4-neon-medium`／`-high` 作為 worker，不再委派 `gemini-4-hydrogen-*`。
  - 維持使用 AGY 內建 `swarm` 而非 `invoke_subagent`：保留 `model: "gemini-4-neon-medium"`（允許 worker 使用比主代理低的推理等級以兼顧速度與成本）、`tasks: [...]` 批次平行分片、`workspace: "share"` 自動建立／清理 git worktree，以及 `post_message` 熱 context 追問與 `fork` 升級 `-high`（依 `swarm` 規範，不使用 `action: "model"` 作非額度原因的等級切換）。
  - **共用配額池的降級規則**：主代理、Neon worker、Hydrogen 與 Flash 全共用同一配額池；升級 `gemini-4-neon-high` 亦消耗主代理自己的配額。若 worker 觸發 `RESOURCE_EXHAUSTED`，不得退回 Hydrogen 或 Flash，改為回報使用者並僅由主代理直接處理剩餘小型工作。
  - **`swarm` Worker 的安全與喚醒規範**：
    - `swarm` worker 無唯讀沙盒，唯讀任務必須在 brief 明文禁止修改檔案，且 Advisor 執行期間禁止同時跑任何 `workspace: "inherit"` 的 worker。
    - `workspace: "share"` 的 worktree 從 `HEAD` 切出，若主代理有尚未 commit 的介面設計，需先 commit 或寫入 brief；驗收並合併變更回主工作區後才呼叫 `swarm(action: "kill")`。
    - brief 結尾固定要求：`Do not add workers or consult other models; post your final result to @coordinator in one message.`，喚醒後以 `read_board` 收取結果。
- **雙 Advisor：`claude-opus-5-5-low` 與 `gemini-4-argon-medium` 同時平行呼叫（皆經 `agy` CLI）**：
  - 新增 `gemini-4-argon-medium` 為第二個 advisor，與 `claude-opus-5-5-low` 同時針對同一份計畫提供獨立第二意見，兼顧跨家族（Claude vs Gemini）與更高階推論模型（Argon）的互補視角。
  - 2026-11-19 實測配額池：`claude-opus-5-5-*` 與 `gemini-4-argon-*` 各自獨立於每週配額池，且皆與 Neon 快速重置配額池分離；其中一邊額度耗盡不影響另一邊。
  - **為何兩位 Advisor 皆走 `agy` CLI 而不把 Argon 放進 `swarm`**：
    1. 兩位 advisor 在同回合以兩個 `run_command`（`WaitMsBeforeAsync: 500`）平行啟動，讀取同一份未更動的 `<YYYYMMDD-HHMMSS>-advisor.brief.md`（內容不提及另一位 advisor），確保完全獨立的盲審，不互相影響，也不在對話中留下 Swarm 留言板狀態。
    2. 統一路徑、逾時（`--print-timeout 20m`）、重試與讀寫檢查規則。
  - **平行回收與失敗處理**：
    - 兩筆 `run_command` 轉入背景後必須結束當前回合並等待**兩筆完成通知都抵達**才整合，從各自指令輸出或 task log 讀取審查結果（避免只收到先完成的那一份就提前整合）。
    - 重試與額度耗盡採個別處理：任一邊輸出為空或遇可重試 `AGY_ERROR` 時，確認工作樹乾淨後僅重跑該 advisor 一次；任一邊遇 `RESOURCE_EXHAUSTED` / `quota reached` 或重跑失敗，向使用者回報該 advisor 不可用並以另一邊結果繼續（兩邊皆失敗則直接繼續）。
  - **衝突整合規則**：
    - 明確標註各觀點來源（Opus vs Argon）；任一 advisor 與主計畫在會影響計畫走向的關鍵處意見衝突，或 Opus 與 Argon 彼此在關鍵處嚴重衝突時，完整回報雙方理由給使用者，不默默挑選或混寫；兩位 advisor 在次要細節上相左時，由主代理擇一並簡述理由。
  - **唯讀保護**：`agy` 沒有可靠的唯讀模式（沿用 2026-11-08 實測），因此於兩位 advisor 啟動前檢查 `git status --porcelain`，兩者皆結束後（及單邊重跑前後）再確認 `git status --porcelain` 與 `git diff` 未變；若平行期間發生意外改動，回報使用者並註明無法歸因於單一 advisor。
- **Linux / WSL `PATH` 與沙盒 fallback**：`run_command` 在 Linux / WSL 啟動非 login `/bin/bash`，`PATH` 可能不含 `~/.local/bin`；找不到 `agy` 時改用 `~/.local/bin/agy`，若沙盒隱藏 `~/.local`，僅對該指令使用 `BypassSandbox`。
- 從前版沿用：不委派需要證據支撐的研究報告（Gemini 曾捏造模型、分數、名稱與數量）；所有 worker / advisor 結果一律視為未驗證。

## 實測紀錄（2026-11-08 與 2026-11-19，AGY 2.0 / agy 1.5.12）
- **模型與配額確認（2026-11-19，agy 1.5.12）**：
  - `agy models` 與 `swarm(action: "models", all: true)` 均已驗證支援 `gemini-4-neon-{low,medium,high,xhigh}`、`gemini-4-argon-{low,medium,high}`、`claude-opus-5-5-{low,medium,high}`。
- **Windows PowerShell 與 bash/WSL 的 stdin 關閉差異**：
  - `agy -p` 若未關閉 stdin 可能卡住。bash / WSL 使用 `</dev/null`。
  - **注意（2026-11-19 修正前版文件落差）**：PowerShell 5.1 / 7 不支援 `<` 輸入重導向（會報錯 `The '<' operator is reserved for future use`），因此 `SKILL.md` 改為寫明 bash / WSL 用 `</dev/null`、PowerShell 用 `$null | agy ...`。`$null | agy ...` 在 Windows 上是否確實關閉 `agy -p` stdin **尚未於 Windows 實測驗證（unverified）**，後續於 Windows 環境使用時需實測確認。
  - 其餘部分（以 `write_file` 寫 UTF-8 brief、不加 `>` 重導向避免 PowerShell 5.1 產生 UTF-16LE、直接由 `run_command` 與 UTF-8 task log 回收輸出）沿用 2026-11-08 實測結果。
- **`--disable-slash-commands` 與唯讀限制（2026-11-08 實測）**：
  - 加上 `--disable-slash-commands` 防止 `-p` 模式展開 brief 中的 slash commands / skills。
  - `--mode plan` 會在工具呼叫時提前 exit 0 中斷，`--sandbox` 搭 `--dangerously-skip-permissions` 仍可建檔，故維持預設模式加 `--dangerously-skip-permissions --disable-slash-commands` 並以 brief 規定唯讀＋git 前後檢查。

## 待觀察
- 每次固定檢查點同時呼叫 Opus 5.5 與 Argon 兩位 advisor 時，Claude Opus 5.5 每週配額消耗是否過快（若過快，可評估改為預設只問 Argon、僅架構／根因修正加問 Opus）。
- Windows PowerShell 下 `$null | agy ...` 關閉 stdin 的實測結果。
- `gemini-4-neon-medium` / `-high` worker 與主代理共用配額池的實際耗用速度與驗收通過率。
