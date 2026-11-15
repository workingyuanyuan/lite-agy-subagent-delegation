---
name: delegate
description: Before executing a plan for a design, architecture change, merge-bound code, or root-cause fix, get a second opinion from Claude Opus 5.5 Medium; delegate parallel, well-scoped exploration, implementation, check, or extraction tasks to Gemini 4 Hydrogen.
---

# Delegate and consult

User instructions override this skill. Treat every worker or advisor result as an unverified opinion.

## When

- **Advisor (`claude-opus-5-5-medium` via `agy` CLI)**: before executing a plan for a design, architecture change, merge-bound code, or root-cause fix, consult Opus for a second opinion. Skip small, low-risk changes.
- **Worker (`gemini-4-hydrogen-medium` via `swarm`)**: after you break down the problem and design interfaces, delegate parallel, well-scoped tasks when it saves context or time and checking costs less than doing the work: fast codebase exploration, leaf-module migrations, mechanical edits, boilerplate tests, high-frequency checks, and raw image/PDF/audio/video extraction. Do not delegate evidence-backed research reports.

## Run

- **Worker (`swarm`)**: call `swarm(action: "add", name: "<role>", model: "gemini-4-hydrogen-medium", workspace: ..., brief: "...")`; use `tasks: [...]` for parallel slices.
  - Read-only: `workspace: "inherit"`; forbid file edits in the brief (workers are not sandboxed).
  - Write-capable: `workspace: "share"` for parallel writers in a git repository (commit or include in the brief any uncommitted interfaces first, as worktrees branch from `HEAD`), or `"inherit"` for a single writer while you do not edit its files.
  - After adding workers, end your turn to wait for the `@coordinator` wakeup, then call `read_board`. Clarify via `post_message`. If `medium` fails acceptance or the task is clearly hard, add a new worker on `gemini-4-hydrogen-high` (optionally with `fork: "<old-worker>"`) and kill the old one; do not use `swarm(action: "model")` to raise effort.
- **Advisor (`agy` CLI from project root)**: write the brief with `write_file` to `<home>/.local/state/delegation-log/<project-dir-name>/<YYYYMMDD-HHMMSS>-opus.brief.md` and pass it by path:
  `agy --model claude-opus-5-5-medium --dangerously-skip-permissions --disable-slash-commands --print-timeout 20m -p "Read the brief at <brief path> and follow it."`
  If `agy` is not found in `PATH`, retry with `~/.local/bin/agy` on Linux/WSL. Read the review from the command output (or task log). If the output is not a valid review (e.g., `RESOURCE_EXHAUSTED` / `quota reached` or empty), tell the user the Opus opinion is unavailable and continue.

## Brief

- Worker (`brief`): objective, owned scope, designed interfaces/context, write scope, deliverable, and an acceptance check it can run itself; end with `"Do not add workers or consult other models; post your final result to @coordinator in one message."`
- Advisor (`.brief.md`): problem, constraints, relevant file paths, and your draft plan; ask for flaws, missed cases, and better alternatives, ordered by severity with reasons; state that it is read-only and must not modify files.
- Every brief: state at the top any tool failure, incomplete work, or change of method; say "unknown" rather than invent sources, numbers, or dates; do not delegate or consult other models.

## Integrate

- Do not run any `workspace: "inherit"` worker (read-only or write-capable) during an Advisor run. In a git repository, check `git status --porcelain` before the Advisor run and confirm `git status --porcelain` and `git diff` are unchanged afterwards; report any unexpected edit to the user.
- Check only the review points you adopt, then revise the plan. When the advisor disagrees with your plan on a point that matters, report the disagreement to the user instead of picking or blending.
- Run write-capable workers without asking. Verify worker output by diff (or target files outside git) and the acceptance check. For `workspace: "share"` writers, merge or apply the verified changes into your workspace before calling `swarm(action: "kill")` (which deletes the worker's worktree and branch), then kill finished workers. Discard conclusions behind a failure notice; if Hydrogen quota is exhausted, do the task yourself.
