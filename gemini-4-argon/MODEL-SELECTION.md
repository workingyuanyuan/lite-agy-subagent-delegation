# delegate skill：選模型與呼叫方式的依據

升級 `SKILL.md` 時才讀。skill 本體不引用這份檔案。

## 適用範圍
- harness：Antigravity（AGY）；主代理是 Gemini 4 Argon。
- 一個 harness 一個 repo（Claude Code 版：`lite-claude-subagent-delegation`；Codex 版：`codex-subagent-delegation` / `lite-codex-subagent-delegation`）。repo 內以主代理模型命名版本資料夾，主代理換模型時另開資料夾。
- 這份檔案放在版本資料夾根目錄，不在 `skills/delegate/` 裡，安裝時不會被複製進 skill。

## 名冊與架構決定（2026-11-14 更新，與使用者確認並經 Opus 5.5 審查）
- **範圍完全收斂在 AGY 內**：不依賴 Codex CLI 或 Claude Code CLI，只使用 AGY 內建工具與 `agy` CLI。
- **Worker 委派給 Gemini 4 Hydrogen（透過 AGY 內建 `swarm`，預設 `gemini-4-hydrogen-medium`，必要時升 `gemini-4-hydrogen-high`）**：
  - 依據 [Gemini 4 Argon Developer Guide](https://aistudio.google.com/learn/gemini-4-argon-developer-guide)（*4. Use Argon as an orchestrator*），官方建議以 Gemini 4 Argon 作為主導協調者（負責問題拆解、介面設計、重度架構推理與最終審查），搭配高速輕量級子代理承接平行且範圍明確的任務：快速程式碼探索（fast codebase explorations）、獨立葉節點模組遷移（migrating individual leaf modules）、樣板測試撰寫（writing boilerplate tests）、高頻檢查（running high-frequency checks），以及耗費 context 的大檔案／多模態原始資訊抽取。Gemini 4 Hydrogen 為接替 Gemini 3.8 Flash 相同定位的下代模型。
  - **為何從 `invoke_subagent(Model="flash")` 改用 `swarm`**：
    - `invoke_subagent` 的 `Model` 參數僅支援 `inherit | flash_lite | flash | pro`（`run_workflow` 的 `agent()` 亦同，`define_subagent` 無模型參數），且實測 `Model="flash"` 底層仍對應舊代 `gemini-3.8-flash-low`（2026-11-08 實測為 `gemini-3.8-flash-tiered`），無法指定 `gemini-4-hydrogen-*`。
    - 比較 `swarm(action="add", model="gemini-4-hydrogen-medium")` 與 `agy --model gemini-4-hydrogen-medium` CLI：
      1. `swarm` 原生支援 `workspace: "share"`（自動於 `~/.gemini/antigravity/worktrees/<conv-id>/swarm-<name>-<hash>` 建立獨立 git worktree 與分支，並在 `kill` 時自動清理）與 `tasks: [...]` 單次批次平行派發，免除在 `SKILL.md` 手動管理 `git worktree` 的複雜度。
      2. `swarm` 為 AGY 原生工具，無跨平台 shell / `PATH` 差異，且支援以 `post_message` 在保留熱 context 下追問，或以 `fork: "<old-worker>"` 升級至 `gemini-4-hydrogen-high`（依 `swarm` 工具規範，不使用 `action: "model"` 作非額度原因的模型切換）。
      3. 單次小型唯讀任務實測（n=1）：`swarm` 耗時 7 秒（`workspace: "share"` 寫檔 12 秒），`agy` CLI 因冷啟動耗時 27 秒。
  - **`swarm` Worker 的安全與喚醒規範**：
    - `swarm` worker 無唯讀沙盒，因此唯讀任務必須在 brief 明文禁止修改檔案，且 Advisor 執行期間禁止同時跑任何 `workspace: "inherit"` 的 worker。
    - `workspace: "share"` 的 worktree 從 `HEAD` 切出，若主代理有尚未 commit 的介面設計，需先 commit 或寫入 brief。
    - `swarm` 靠 `@coordinator` 觸發主代理喚醒，因此 brief 結尾固定要求：`Do not add workers or consult other models; post your final result to @coordinator in one message.`，主代理被喚醒後以 `read_board` 讀取結果，並在驗收／合併後以 `swarm(action: "kill")` 清除 worker。
    - 實測 `swarm(action: "models")` 顯示 `gemini-4-hydrogen-*` 與 `gemini-3.8-flash-*` 共用同一配額池（剩餘比例與重置時間相同），因此 Hydrogen 額度耗盡時不退回 Flash，直接由主代理自行處理。
- **Advisor 為 AGY 內的 Claude Opus 5.5 Medium（`agy --model claude-opus-5-5-medium`）**：
  - Advisor 屬低頻、單次、唯讀的第二意見審查，維持透過 `agy` CLI 呼叫可在單輪直接由 `run_command` 收回完整輸出，不在對話中留下 Swarm 留言板狀態。
  - **Linux / WSL `PATH` fallback**：AGY 2.0 桌面端在 Linux / WSL 的 `run_command` 啟動非 login `/bin/bash`，`PATH` 可能不含 `~/.local/bin`；若直接執行 `agy` 找不到指令，先重試 `~/.local/bin/agy`，避免誤判為「Opus 意見不可用」。
  - **Advisor 觸發點是固定檢查點**：執行設計、架構變更、要 merge 的程式碼或根因修正的計畫之前詢問 Opus；小而低風險的改動跳過。
  - 若 AGY 的 Opus 額度耗盡（`RESOURCE_EXHAUSTED` / `quota reached`）或失敗，不再切換至 Claude Code CLI，直接向使用者回報「Opus 意見不可用」並繼續。
- 從 Claude Code 版沿用：不委派需要證據支撐的研究報告（前例：Gemini 曾捏造模型、分數和日期）；委派結果一律視為未驗證；意見不一致時回報差異，不挑選也不混寫。

## 實測紀錄（2026-11-08，AGY 2.0 / agy 1.5.3）
- **為什麼完全省去 Windows PowerShell 5.1 的編碼與重導向設定**：
  - 雖然 Windows 本機裝有 PowerShell 7（`pwsh.exe` 7.6.9），但 AGY 的 `run_command` 工具在 Windows 下固定啟動 `powershell.exe`（5.1）作為外層 shell。
  - PowerShell 5.1 有兩個陷阱：(1) 原生執行檔參數中的內嵌雙引號會被吃掉；(2) 使用 `>` / `*>` 重導向寫檔會輸出 `UTF-16LE`（導致 AGY 的 `read_file` 報錯 `charset=utf-16le` 拒絕讀取）。
  - 當 Worker 改用內建 `invoke_subagent`、且 Advisor 改為「用 `write_file` 寫出 UTF-8 brief、以路徑傳給 `agy -p`、不加 `>` 重導向而直接由 `run_command` 擷取 stdout/stderr（或讀取系統自動生成的 UTF-8 task log）」時，實測在 Windows PowerShell 5.1 下約 70 秒的長篇含中文審查輸出可完整無亂碼收回，且指令語法在 Windows PowerShell（5.1 / 7）與 WSL / Linux（bash）完全一致。
- **agy 加上 `--disable-slash-commands`**：避免在 `-p` print mode 下展開 brief 或 prompt 內的 slash commands 與 skills，同時在 brief 中規定不得再委派或諮詢其他模型。
- **agy 沒有可靠的唯讀模式**：
  - `--mode plan` 遇列目錄或執行指令時會在第一句後中斷且 exit code 仍為 0；`--sandbox` 搭配 `--dangerously-skip-permissions` 仍可建檔。
  - → Opus advisor 用預設模式加 `--dangerously-skip-permissions --disable-slash-commands`，在 brief 中寫明唯讀，並在 git 專案中於執行前後比對 `git status --porcelain` 與 `git diff`（期間不在主工作樹同時跑寫檔 worker）。
- **`Workspace: "share"` 子代理的清理順序**：`manage_subagents` 終止子代理時會一併刪除其分支工作區，因此必須先驗收並把變更合併回主工作區，最後才用 `manage_subagents` 清除已完成但仍處於 `SUBAGENT_STATE_ALIVE` 的子代理。

## 待觀察
- 固定檢查點是否讓 Opus advisor 呼叫太頻繁、太花額度。
- 內建 `swarm`（`gemini-4-hydrogen-medium` / `-high`）在平行機械性修改與模組遷移上的驗收通過率，以及 `@coordinator` 喚醒與 `read_board` 流程是否穩定。
