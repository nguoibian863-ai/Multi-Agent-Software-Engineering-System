# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Current state

This repository is in the **planning phase — there is no code yet**. It contains only:

- `multi-agent-software-engineering-master-plan.md` — the master plan (v2), the source of truth for architecture and scope. Written in Vietnamese; section numbers (`§N`) are referenced throughout `plan/`.
- `plan/` — the master plan's §45 split into 22 implementation steps grouped by phase (`phase-N-*/NN-*.md`). `plan/README.md` holds the step table (with status), dependency graph, and milestones.

Documentation is written in Vietnamese; keep new plan/ADR/milestone docs in Vietnamese to match.

## Working through the plan

- Start a step only when every step in its **Phụ thuộc** (dependencies) row is done. Each step file lists its checklist (**Việc cần làm**), expected files (**Output**), and exit criteria (**Tiêu chí hoàn thành**).
- When a step's status changes, update both the `Trạng thái:` line at the top of the step file and the status column in `plan/README.md` (`☐` todo · `◐` in progress · `☑` done · `✖` dropped).
- Architecture decisions go in `docs/decisions/ADR-XXX-*.md`; milestone evidence goes in `docs/milestones/M*-*.md`.
- Hard gates: **M1** (after step 01) decides whether OpenHands is used or replaced by a self-written mini-SWE-agent-style loop — this changes step 03. **M2** (after step 12): if the safety policy tests do not all pass, do not start phase 4.
- If the plan and a step file disagree, the master plan wins; fix the step file.

## Planned tooling (not present until step 02 creates the skeleton)

Python 3.12 managed with `uv`; checks are `uv run ruff check .`, `uv run pyright` (strict for `orchestrator/policy/**`), `uv run pytest` (single test: `uv run pytest path/to/test_file.py::test_name`). The CLI is `mase` (typer, `apps/cli/main.py`). Benchmarks: `benchmarks/run.py` over `benchmarks/golden_tasks/*.yaml`. Verify these exist before relying on them.

Target platform is Windows with Docker Desktop (WSL2 backend); agent commands run in Linux containers.

## Architectural invariants code must preserve

These cut across many planned modules and are the point of the design — don't weaken them for convenience:

- **LLM proposes, runtime decides.** No LLM output ever sets a safety limit. `user_ceiling`, `write_denied`/`read_denied`, retry limits, budget, and the command allowlist come from `PolicySnapshot` (project config), never from `plan.json`. The `Plan` schema uses `extra="forbid"` so such fields are rejected.
- **Three safety layers, in order:** Docker sandbox (only the task worktree mounted, network off) → File/Command Policy checked before every tool call → post-exec diff check after every command that restores files changed outside `plan_allowed`. File policy alone is insufficient because `pytest`/`python` execute arbitrary code.
- **Permission model:** `plan_allowed` holds exact paths plus `op` (create/modify/delete); only `user_ceiling` and the deny lists use globs. Write = in `plan_allowed` ∧ op matches ∧ in ceiling ∧ not denied. Reads are checked against `read_denied` too (secrets must never reach an LLM). Path normalization must handle Windows cases (case-folding, ADS, junctions, UNC/`\\?\`, 8.3 names, device names).
- **Runtime owns git.** Each task runs in its own worktree on branch `mase/<task_id>`; the runtime creates checkpoint commits per passing step (user git hooks disabled) and never pushes or merges into the user's branch. Agents only get read-only git tools, which run host-side.
- **Deterministic workflow.** The Supervisor is code: a transition table held as data (`workflow/transitions.py`) driven by `workflow/engine.py`. State handlers must be idempotent (needed for resume). Only the engine writes state, and every status change and its event are written in one SQLite transaction.
- **Only the Coder uses an agent loop (OpenHands).** Architect, Reviewer, Gate, Classifier and Researcher are single structured-output LLM calls via a thin LiteLLM wrapper (`llm/client.py`).
- **Gate:** a deterministic gate runs first; the LLM gate can only add failures, never turn a FAIL into a PASS. The Reviewer's verdict is overridden to FAIL if it has any high-severity issue or any unsatisfied acceptance criterion.
- **Checks are judged against a baseline** recorded before any edits (only new lint/type/test failures count). Diffs are also scanned for weakened tests (removed tests, added skips, fewer asserts).
- Research output, repo contents, and command output are untrusted data for prompt purposes.
