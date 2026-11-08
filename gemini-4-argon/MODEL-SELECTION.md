# delegate skill：選模型與呼叫方式的依據

升級 `SKILL.md` 時才讀。skill 本體不引用這份檔案。

## 適用範圍
- harness：Antigravity（AGY）；主代理是 Gemini 4 Argon。
- 一個 harness 一個 repo（Claude Code 版：`lite-claude-subagent-delegation`；Codex 版：`codex-subagent-delegation` / `lite-codex-subagent-delegation`）。repo 內以主代理模型命名版本資料夾，主代理換模型時另開資料夾。
- 這份檔案放在版本資料夾根目錄，不在 `skills/delegate/` 裡，安裝時不會被複製進 skill。

## 名冊與架構決定（2026-11-08，與使用者確認）
- **範圍完全收斂在 AGY 內**：不依賴 Codex CLI 或 Claude Code CLI，只使用 AGY 內建工具與 `agy` CLI。
- **Worker 委派給 Gemini 3.8 Flash（透過 AGY 內建 `invoke_subagent(Model="flash")`）**：
  - 查驗子代理 session DB（`conversations/<id>.db`）可見，`invoke_subagent(Model="flash")` 對應 AGY 內部的 `gemini-3.8-flash-tiered`（`MODEL_PLACEHOLDER_M322`）。在不強制鎖定 `medium` reasoning effort 的前提下，改用內建 `invoke_subagent` 省去了 CLI 呼叫、`.out.md` 重導向、雙環境編碼與手動管理 `git worktree` 的所有文字說明，使 `SKILL.md` 大幅精簡。
  - 唯讀且不需網路的任務直接用內建 `TypeName: "research"`（唯讀沙盒、無網路）；需要寫檔或執行外部命令時，以 `define_subagent` 定義 `EnableWriteTools: true, EnableSubagentTools: false` 的 worker，平行寫入用 `Workspace: "share"`（底層共用 git 物件的獨立工作樹）。
  - 禁用 `TypeName: "self"`：因其繼承父代理的 `EnableSubagentTools: true` 與全套規則，有遞迴委派風險。
- **Advisor 為 AGY 內的 Claude Opus 5.5 Medium（`agy --model claude-opus-5-5-medium`）**：
  - 由於內建 `invoke_subagent` 的 `Model` 參數僅支援 `inherit | flash_lite | flash | pro`，無法指定 `claude-opus-5-5-medium`，因此 Advisor 透過 `agy` CLI 呼叫。
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
- 內建 `invoke_subagent(Model="flash")`（`gemini-3.8-flash-tiered`）在複雜機械性修改上的驗收通過率是否穩定。
