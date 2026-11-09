---
name: delegate
description: Before executing a plan for a design, architecture change, merge-bound code, or root-cause fix, get a second opinion from Claude Opus 5.5 Medium; delegate parallel, well-scoped exploration, implementation, check, or extraction tasks to Gemini 3.8 Flash.
---

# Delegate and consult

User instructions override this skill. Treat every worker or advisor result as an unverified opinion.

## When

- **Advisor (`claude-opus-5-5-medium` via `agy` CLI)**: before executing a plan for a design, architecture change, merge-bound code, or root-cause fix, consult Opus for a second opinion. Skip small, low-risk changes.
- **Worker (`flash` via `invoke_subagent`)**: after you break down the problem and design interfaces, delegate parallel, well-scoped tasks when it saves context or time and checking costs less than doing the work: fast codebase exploration, leaf-module migrations, mechanical edits, boilerplate tests, high-frequency checks, and raw image/PDF/audio/video extraction. Do not delegate evidence-backed research reports.

## Run

- **Worker (`invoke_subagent`)**: set `Model: "flash"` (never `TypeName: "self"`, which risks recursive delegation).
  - Read-only, offline: `TypeName: "research"`, `Workspace: "inherit"`.
  - Write-capable (or needs shell/network): define a writer once per conversation with `define_subagent` (`EnableWriteTools: true`, `EnableSubagentTools: false`). Use `Workspace: "share"` for parallel writers (in a git repository) or `"inherit"` for a single writer while you do not edit its files.
- **Advisor (`agy` CLI from project root)**: write the brief with `write_file` to `<home>/.local/state/delegation-log/<project-dir-name>/<YYYYMMDD-HHMMSS>-opus.brief.md` and pass it by path:
  `agy --model claude-opus-5-5-medium --dangerously-skip-permissions --disable-slash-commands --print-timeout 20m -p "Read the brief at <brief path> and follow it."`
  Read the review from the command output (or task log). If the output is not a valid review (e.g., `RESOURCE_EXHAUSTED` / `quota reached` or empty), tell the user the Opus opinion is unavailable and continue.

## Brief

- Worker (`Prompt`): objective, designed interfaces/context, write scope, deliverable, acceptance check.
- Advisor (`.brief.md`): problem, constraints, relevant file paths, and your draft plan; ask for flaws, missed cases, and better alternatives, ordered by severity with reasons; state that it is read-only and must not modify files.
- Every brief: state at the top any tool failure, incomplete work, or change of method; say "unknown" rather than invent sources, numbers, or dates; do not delegate or consult other models.

## Integrate

- Do not run write-capable workers in the main tree during an Advisor run. In a git repository, check `git status --porcelain` before the Advisor run and confirm `git status --porcelain` and `git diff` are unchanged afterwards; report any unexpected edit to the user.
- Check only the review points you adopt, then revise the plan. When the advisor disagrees with your plan on a point that matters, report the disagreement to the user instead of picking or blending.
- Run write-capable workers without asking. Verify worker output by diff (or target files outside git) and the acceptance check. For `Workspace: "share"` writers, merge or apply the verified changes into your workspace before killing the worker, then clean up finished workers with `manage_subagents`. Discard conclusions behind a failure notice; for insufficient results, clarify via `send_message` or do it yourself.
