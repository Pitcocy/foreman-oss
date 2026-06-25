# CLAUDE.md

Guidance for Claude Code when working **on the Foreman codebase itself** (not on a target project Foreman is orchestrating).

## What this repo is

Foreman is a Claude Code **plugin**: a set of skills + Phase Brief templates + a small bash dispatch helper. There is no app runtime, no package manager, no build step. Changes are edits to Markdown skills and one shell script.

- The orchestration skill lives in `skills/orchestrate/` (SKILL.md, `bin/dispatch.sh`, `templates/`).
- The authoring skills (`prd-to-plan`, `write-a-prd`, `execute-phase`, `grill-with-docs`, `prototype`, `tdd`) produce the `CLAUDE.md` / `plans/` / `IMPLEMENTATION.md` convention that `orchestrate` then runs.
- Domain language is defined in `CONTEXT.md`; design rationale in `docs/adr/`. Read ADR-0001 and ADR-0002 before changing the run model.

## House rules

- **Keep it bash + Markdown.** No new runtimes (Python/Node apps), no daemon, queue, or background service. A Run lives and dies with its Orchestrator session, restartable from `.foreman/` files.
- **Surgical changes.** Match the existing prose style of the skills. The skills are prompts — wording is load-bearing; don't paraphrase carelessly.
- **No personal identity in shipped files.** The human is always "the Operator", never a name. Machine-specific values (escalation channels) live in `foreman.config.json`, which is gitignored.
- **Attribution.** `grill-with-docs`, `prototype`, and `tdd` are adapted from Matt Pocock's MIT-licensed skills — keep the attribution comment at the top of each and `THIRD_PARTY_LICENSES/` intact.
- Never push or touch `main` of a target project — that constraint is the heart of ADR-0002 and applies to Foreman's own behavior.

## Do NOT introduce

- No dashboard/web UI — monitoring is the Orchestrator chat + `tail -f` on worker logs + optional pings.
- No interactive/resumable Workers — one-shot headless only (ADR-0001).
- No second lockfile / package manager.
