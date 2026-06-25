# Workers are one-shot headless processes, never interactive sessions

Foreman could have spawned interactive agent sessions in terminals (tmux-style, steerable mid-phase). We decided instead that a Worker is a one-shot headless process (`claude -p` / `codex exec`): input is a Phase Brief, output is a Result File plus a streamed log, and a Worker that can't follow the plan Halts loudly — it can never ask anyone anything. The Orchestrator is Operator-as-judge, not Operator-as-hands: it dispatches Workers (including the Verifier) and reads Result Files, but never builds or executes test steps itself, keeping its context lean enough to survive a long multi-phase run and making every run restartable from `.foreman/` files alone.

## Considered Options

- **Interactive worker sessions (tmux/Claude Squad style):** rejected — mid-phase steering is the babysitting Foreman exists to eliminate, resumable conversations drag state out of files back into context, and monitoring would require terminal multiplexing (tmux wasn't even installed).
- **Orchestrator verifies phases itself:** rejected — independence holds (it didn't write the code), but walking test steps for many phases floods the Orchestrator's context with transcripts and the evidence dies with the session; a dispatched Verifier leaves its verdict on disk.

## Consequences

- A halted/failed phase is always re-run from scratch with an amended Phase Brief and a fresh Worker — never resumed.
- Live monitoring is `tail -f` on a Worker's log file; no terminal multiplexer or UI is required for v1.
