# Foreman commits on run branches, despite the "never commit unless asked" rule

A common project convention forbids agents committing unless asked; an autonomous overnight run needs git checkpoints for rollback, per-phase review, and blast-radius containment under `--dangerously-skip-permissions`. We decided that starting a Run *is* the ask: the Orchestrator opens a Run Branch (`foreman/<plan>-<date>`), makes exactly one Phase Commit per phase — only after the Verifier passes, authored as `Foreman <foreman@local>` so machine checkpoints are filterable from the Operator's commits — and never touches `main` or any remote. Workers and Verifiers never commit. Halt/Fail debris is stashed aside and the tree reset to the last Phase Commit before re-dispatch; the Operator reviews the branch commit-by-commit and merges to `main` themselves, which makes the merge commit their acceptance signature.

## Consequences

- Target Projects must be git repos with a clean tree at Run start; Pre-flight refuses otherwise.
- `foreman@local` is a deliberately fake identity — no account exists or is needed; it only ever appears in local commit metadata.
