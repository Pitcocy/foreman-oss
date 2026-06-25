# Foreman

Local orchestration of a tracer-bullet workflow: an Orchestrator drives a target project's plan phase-by-phase by dispatching headless agent Workers, so the Operator reviews outcomes instead of babysitting each phase. ("The Operator" is the human running Foreman.)

## Language

**Orchestrator**:
The Claude Code session that reads a target project's plan and tracker, dispatches Workers, and escalates to the Operator.
_Avoid_: manager, supervisor, main agent

**Operator**:
The human running Foreman — the judge who reviews Ready-for-Review phases, answers escalations, and owns the merge to `main`. Foreman acts as Operator-as-judge, never Operator-as-hands.

**Worker**:
A one-shot headless agent process (`claude -p` or `codex exec`) that executes exactly one Phase and cannot interact with anyone.
_Avoid_: subagent, terminal, session

**Phase**:
One vertical slice from a target project's plan in `plans/`, with acceptance criteria and "How to test as the user" steps.

**Phase Brief**:
The complete, self-contained prompt the Orchestrator compiles for a Worker — the phase spec plus pre-answered scope-lock questions.
_Avoid_: task, prompt, ticket

**Result File**:
The structured file a Worker writes on exit (`outcome: built | halted | failed`, summary, self-check against acceptance criteria); the only thing the Orchestrator reads from a Worker.
_Avoid_: report, output

**Halt**:
A Worker ending early because the plan can't be followed as written (architectural deviation, unmeetable criterion); always escalates to the Operator, never improvised around.

**Verifier**:
An independent headless Worker (never the builder, never the Orchestrator) that executes a phase's "How to test as the user" steps and writes a pass/fail Result File with evidence.
_Avoid_: tester, QA agent

**Ready-for-Review**:
The highest status Foreman can grant a phase — Verifier passed, `manual:` steps deferred to the Operator; ✅ remains the Operator's alone.

**Manual Step**:
A test step prefixed `manual:` in the plan that the Verifier never executes or judges — it is listed as deferred in the verdict for the Operator's batch review.

**Engine**:
The CLI a Worker runs on — `codex` or `claude` — chosen by the Operator per Run at kickoff (required, no silent default), with optional per-phase `worker:` override in the plan; the Verifier always runs on `claude`.

**Fail**:
A phase outcome where the plan was followable but execution fell short (builder errors, unmet criteria); auto-retried up to twice with a fresh Worker and evidence-amended Phase Brief, then escalated.
_Avoid_: conflating with Halt — a Halt means the plan itself can't be followed and always escalates with zero retries

**Failure Class**:
The Verifier's diagnosis of a fail — `code` (implementation defective), `environment` (sound code blocked by config/credentials/migrations), or `mixed` (both, or unsure — never environment when in doubt); made by the agent that inspected the tree, never the Orchestrator.

**Remediation**:
The environment-class fail path: the tree is kept (no reset), a one-shot Worker applies only the Verifier's suggested fix and produces the real artifacts; at most one per Phase, counts against the attempt budget, and a second fail of any class falls back to the blunt reset path.

**Adjudication**:
The Orchestrator's response to a builder/Verifier disagreement — it reads both Result Files and dispatches a targeted re-check to whichever side is suspect; it never self-certifies a pass without a passing Verifier verdict on disk.

**Run**:
One Orchestrator pass over a Target Project's plan, from Pre-flight to completion or stop; identified by a run id.

**Run Directory**:
`.foreman/runs/<run-id>/` inside the Target Project — manifest, Phase Briefs, Result Files, and per-Worker logs; gitignored; the Run's single source of truth, survives Orchestrator death.

**Run Branch**:
The git branch (`foreman/<plan>-<date>`) a Run works on; `main` is never touched by an autonomous Run, and Foreman never pushes.

**Phase Commit**:
The one commit per Phase, made by the Orchestrator only after the Verifier passes, authored as `Foreman <foreman@local>` so machine checkpoints are filterable from the Operator's own commits.

**Target Project**:
Any repo following the convention (`CLAUDE.md`, `plans/`, `IMPLEMENTATION.md`) that the Orchestrator is pointed at.

**Autonomous-Ready Plan**:
A plan whose phases each carry the pre-answered scope-lock decisions (`TDD: yes/no`, optional `gate: human`) so a run is deterministic from the plan file alone.

**Gate**:
A per-phase `gate: human` marker, placed at plan-writing time, where the Orchestrator must stop and wait for the Operator regardless of verification outcome.
_Avoid_: checkpoint, approval step

**Pre-flight**:
The Orchestrator's check before first dispatch that the plan is Autonomous-Ready; missing markers are resolved with the Operator in one batch (legacy-plan fallback) before any Worker fires.

## Relationships

- The **Orchestrator** dispatches exactly one **Worker** per **Phase** attempt
- A **Worker** consumes one **Phase Brief** and produces one **Result File**
- A **Halt** is re-dispatched as a fresh Worker with an amended **Phase Brief** — Workers are never resumed mid-conversation; whether the halted *tree* is kept or reset is the Operator's call at escalation (keep for spec clarifications, reset for design changes), since every Halt already has them in the loop
- The **Orchestrator** runs one **Pre-flight** per run, before the first dispatch
- A **Phase** carries at most one **Gate**; in autonomous mode "what changed since planning" is always answered *nothing* — disagreement with reality is a **Halt**
- The **Verifier** verifies *artifacts, not pipelines*: it inspects what the builder produced and only re-executes commands marked side-effect-free — it never re-runs anything that spends money or writes externally
- The **Orchestrator** is Operator-as-judge, not Operator-as-hands: it reads Result Files and decides; it never builds or executes test steps itself
- A **Fail** retries (max 2); a **Halt** never retries; a Verifier crash re-dispatches the Verifier once — *no verdict is never a pass*
- `outcome: built` means acceptance criteria are *evidenced by artifacts produced in that attempt* — code-complete-but-never-ran is a **Halt**, and in-repo environment wiring (loading existing credentials, migrations) is in the builder's scope
- An `environment` **Failure Class** triggers **Remediation** (keep the tree, fix, re-verify); `code`/`mixed`/repeat fails take the blunt reset path
- At most one **Run** per **Target Project** at a time (lockfile in `.foreman/`, with stale-lock detection); parallel Runs across *different* projects are just separate Orchestrator sessions
- Run controls are chat ("status", "stop after phase 4", "abort"); escalations, Gates, and completion notify the Operator with phase + reason + verdict summary

## Example dialogue

> **Dev:** "The phase-3 **Worker** got stuck — can I jump into its terminal and steer it?"
> **Operator:** "No. If it can't follow the plan it **Halts**, the **Orchestrator** escalates to me, I amend the plan, and a fresh **Worker** re-runs with a new **Phase Brief**."

## Flagged ambiguities

_(none yet)_
