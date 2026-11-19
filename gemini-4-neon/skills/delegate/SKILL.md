---
name: delegate
description: Before executing a plan for a design, architecture change, merge-bound code, or root-cause fix, get independent second opinions from Claude Opus 5.5 Low and Gemini 4 Argon Medium; delegate parallel, well-scoped exploration, implementation, check, or extraction tasks to Gemini 4 Neon.
---

# Delegate and consult

User instructions override this skill. Treat every worker or advisor result as an unverified opinion.

## When

- **Advisors (`claude-opus-5-5-low` and `gemini-4-argon-medium` via `agy` CLI)**: before executing a plan for a design, architecture change, merge-bound code, or root-cause fix, consult both Opus and Argon simultaneously for independent second opinions. Skip small, low-risk changes.
- **Worker (`gemini-4-neon-medium` via `swarm`)**: after you break down the problem and design interfaces, delegate parallel, well-scoped tasks when it saves context or time and checking costs less than doing the work: fast codebase exploration, leaf-module migrations, mechanical edits, boilerplate tests, high-frequency checks, and raw image/PDF/audio/video extraction. Do not delegate evidence-backed research reports.

## Run

- **Worker (`swarm`)**: call `swarm(action: "add", name: "<role>", model: "gemini-4-neon-medium", workspace: ..., brief: "...")`; use `tasks: [...]` for parallel slices.
  - Read-only: `workspace: "inherit"`; forbid file edits in the brief (workers are not sandboxed).
  - Write-capable: `workspace: "share"` for parallel writers in a git repository (commit or include in the brief any uncommitted interfaces first, as worktrees branch from `HEAD`), or `"inherit"` for a single writer while you do not edit its files.
  - After adding workers, end your turn to wait for the `@coordinator` wakeup, then call `read_board`. Clarify via `post_message`. If `medium` fails acceptance or the task is clearly hard, add a new worker on `gemini-4-neon-high` (optionally with `fork: "<old-worker>"`) and kill the old one; do not use `swarm(action: "model")` to raise effort.
- **Advisors (`agy` CLI from project root, in parallel)**: write one shared brief with `write_file` to `<home>/.local/state/delegation-log/<project-dir-name>/<YYYYMMDD-HHMMSS>-advisor.brief.md` (do not mention the other advisor, and reuse it unchanged for both runs and any retry). Launch both models in the same turn with two `run_command` calls (`WaitMsBeforeAsync: 500`), passing `<advisor>` as `claude-opus-5-5-low` and `gemini-4-argon-medium`:
  `agy --model <advisor> --dangerously-skip-permissions --disable-slash-commands --print-timeout 20m -p "Read the brief at <brief path> and follow it." </dev/null`
  Close stdin (`</dev/null` in bash/WSL; `$null | agy ...` in PowerShell), or `agy -p` can hang. If `agy` is not found in `PATH`, retry with `~/.local/bin/agy` on Linux/WSL; if the sandbox hides `~/.local`, run that one command with BypassSandbox. End your turn and wait until both tasks finish before integrating; read each review from its command output or task log (`agy` can exit 0 with no review). If an advisor's output is empty or a retryable `AGY_ERROR` (e.g., `cannot rewind body after connection loss`), rerun only that advisor once after confirming the tree is clean. If an advisor hits `RESOURCE_EXHAUSTED` / `quota reached`, or its rerun also fails, tell the user which advisor is unavailable and continue with the other (or continue without advisors if both fail).

## Brief

- Worker (`brief`): objective, owned scope, designed interfaces/context, explicit write scope, deliverable, and an acceptance check it can run itself. Keep each brief narrow: split broad work into separate briefs (e.g., discovery / extraction / check / drafting). Give a time limit, require writing results to files incrementally, and forbid reading any file over ~300 KB whole (grep or read ranges instead). End with `"Do not add workers or consult other models; post your final result to @coordinator in one message."`
- Advisor (`.brief.md`): problem, constraints, relevant file paths, and your draft plan; ask for flaws, missed cases, and better alternatives, ordered by severity with reasons; state that it is read-only and must not modify files.
- Every brief: state at the top any tool failure, incomplete work, or change of method; say "unknown" rather than invent sources, numbers, or dates; do not delegate or consult other models.

## Integrate

- Do not run any `workspace: "inherit"` worker (read-only or write-capable) during an Advisor run. In a git repository, check `git status --porcelain` before starting the advisors and confirm `git status --porcelain` and `git diff` are unchanged after both finish (and before/after any retry); report any unexpected edit to the user (unattributable when both ran in parallel).
- Check only the review points you adopt, then revise the plan; verify file, symbol, count, or "unchanged" claims from either advisor before acting on them. Attribute key points to Opus vs Argon. When either advisor disagrees with your plan on a point that would change the plan — or when Opus and Argon materially disagree with each other — report the disagreement and each side's reasoning to the user instead of silently picking or blending; for minor ties between advisors, pick a side and state why.
- Run write-capable workers without asking. Verify worker output by diff (or target files outside git) and the acceptance check. For `workspace: "share"` writers, merge or apply the verified changes into your workspace before calling `swarm(action: "kill")` (which deletes the worker's worktree and branch), then kill finished workers. Check claims such as "unchanged", "new since", or "no change" against the actual data or diff, and verify every finding (including names and counts) before acting on it; Gemini has fabricated these. Discard conclusions behind a failure notice; if worker spawning hits `RESOURCE_EXHAUSTED`, do not fall back to Hydrogen or Flash (they share Neon's quota pool) — report it to the user and handle only small remaining work directly.
