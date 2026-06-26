# Foreman

**Foreman is a local orchestrator that runs a tracer-bullet plan autonomously.** You point it at a project that follows a simple convention (`CLAUDE.md`, `plans/`, `IMPLEMENTATION.md`), and it works the plan phase by phase: it dispatches a one-shot headless Worker to build each phase, dispatches an independent Verifier to walk the "how to test as the user" steps, commits a checkpoint to a run branch when a phase passes, and only interrupts you on exceptions — a Halt, exhausted retries, or a phase explicitly gated for a human.

You stay the judge. Foreman never marks a phase done (✅), never merges, and never touches `main`. Its ceiling is **Ready-for-Review**: a passing verdict on disk and a checkpoint commit you review and merge yourself.

> ⚠️ **Foreman runs agents with permissions bypassed** (`claude -p --dangerously-skip-permissions` / `codex exec --dangerously-bypass-approvals-and-sandbox`) so Workers can build unattended. Only run it on a project you trust, in a git repo with a clean tree, and read [Safety](#safety) before your first run.

---

## How it works

```
Orchestrator (you, running /foreman)
   │  pre-flights the plan, then per phase:
   ├─► Builder Worker      one-shot `claude -p` or `codex exec`  → writes a Result File
   ├─► Verifier Worker     independent `claude -p`               → writes a pass/fail Verdict
   └─► on pass: commit checkpoint to  foreman/<plan>-<date>  branch  (never main)
       on Halt / exhausted retries / gate:human → escalate to you, wait
```

- **Workers are one-shot and headless.** They get a self-contained Phase Brief, build, and write a structured Result File. They can't ask questions — if the plan can't be followed, they *Halt* loudly.
- **Verification is independent.** A separate agent that didn't write the code walks the test steps and leaves a verdict on disk. No verdict is never a pass.
- **Every checkpoint is a real git commit** authored as `Foreman <foreman@local>`, on a run branch, so you can review commit-by-commit and merge yourself.
- **Run state lives in the target project** under `.foreman/runs/<run-id>/` (gitignored). A Run survives the Orchestrator dying — a new session resumes from the manifest.

## What's in the box

| Skill | Role |
|-------|------|
| `foreman` | **Foreman core** — runs a plan autonomously (the loop above). |
| `prd-to-plan` | Turn a PRD into a phased tracer-bullet plan in `plans/`. |
| `write-a-prd` | Produce a PRD through structured discovery. |
| `execute-phase` | Run a single phase interactively (the non-autonomous path). |
| `grill-with-docs` | Stress-test a plan against your domain model and update docs. |
| `prototype` | Build a throwaway prototype to settle a design question. |
| `tdd` | Red-green-refactor loop for a phase. |

The authoring skills produce the convention that `foreman` consumes — together they're the whole loop from idea → plan → autonomous build.

## Requirements

- **[Claude Code](https://claude.com/claude-code)** ≥ 2.1, logged into your own subscription.
- **[`codex`](https://github.com/openai/codex) CLI** ≥ 0.138, logged in — only if you want Codex as a build engine. The Verifier always runs on `claude`, so Claude Code is required regardless.
- **`git`**.
- Optionally, a Discord channel wired into your Claude Code, if you want escalations pinged to your phone (see [Configuration](#configuration)).

## Install

Foreman is a Claude Code plugin distributed via this git repo as a marketplace.

```
/plugin marketplace add <your-org>/foreman
/plugin install foreman
```

(Replace `<your-org>/foreman` with wherever you host this repo.) The skills become available as `/foreman`, `/prd-to-plan`, etc.

## The target-project convention

Foreman orchestrates *your* project, which must carry three things:

- **`CLAUDE.md`** (and/or `AGENTS.md` for Codex Workers) — the project's agent instructions.
- **`plans/`** — one or more tracer-bullet plan files. Each remaining phase needs *Acceptance criteria*, *How to test as the user* steps, a `TDD: yes/no` marker, and an explicit `gate: human` where a human must sign off. The `prd-to-plan` skill writes plans in exactly this shape.
- **`IMPLEMENTATION.md`** — a status tracker whose phase numbers and titles agree with the plan.

If a plan is missing those markers, Foreman's pre-flight stops and resolves them with you in one batch before any Worker fires — it never invents answers.

## Usage

From inside (or pointing at) your target project:

```
/foreman
```

Then tell it the target path and the build **Engine** (`claude` or `codex` — required, no default). Foreman pre-flights, then runs. While it works:

- `status` — where the run is, answered from the manifest.
- `stop after 4` — finish phase 4, then pause.
- `abort` — kill the current Worker, preserve debris, reset to the last checkpoint, release the lock.

Live-watch any Worker with the `tail -f` command Foreman prints when it dispatches.

On completion you get a batch-review handoff: per-phase one-liners, the deferred `manual:` steps to walk yourself, and `git log --author=Foreman --oneline` to review the checkpoints. **Marking ✅ and merging to `main` are yours** — the merge commit is your acceptance signature.


## Credits & license

Foreman is MIT licensed — see [`LICENSE`](LICENSE).

The `grill-with-docs`, `prototype`, `write-a-prd`, `prd-to-plan` and `tdd` skills are **adapted from [Matt Pocock's skills collection](https://github.com/mattpocock/skills)** (MIT), modified to fit Foreman's tracer-bullet convention. His license is preserved in [`THIRD_PARTY_LICENSES/`](THIRD_PARTY_LICENSES/). Thanks to Matt for publishing them freely.
