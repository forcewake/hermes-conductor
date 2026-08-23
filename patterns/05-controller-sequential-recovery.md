# Controller-sequential recovery for Kanban same-checkout collisions

Use this when a Kanban swarm is supposed to mutate a single repository via isolated worktrees, but workers are observed claiming the canonical checkout path.

## Trigger signals

- Worker task shows `workspace: worktree @ /path/to/repo` but the actual path is the canonical checkout.
- Multiple mutating cards enter `running` in the same checkout.
- New dependency completions auto-promote downstream cards into the same checkout.

## Safe recovery pattern

1. **Stop parallel mutation immediately.** Reclaim running mutating cards with a reason like `same-checkout collision; controller sequential integration`.
2. **Block auto-respawned cards** after reclaiming them. Kanban may promote children as dependencies complete; check the board after each `complete`.
3. **Do not discard useful partial deltas blindly.** Inspect the working tree and integrate only coherent code/tests into the canonical branch.
4. **Switch to controller-sequential batches** until real worktree isolation is verified. Pick dependency-critical slices, implement one batch at a time, and keep blocked collision cards blocked until their controller integration is ready.
5. **For each card/batch:**
   - implement tests + code;
   - run the card target gate;
   - run focused regressions;
   - run full/static/secret gates before commit when the batch touches shared infrastructure;
   - write the card artifact under its planned `.hermes/rdd/.../tasks/<card>/05-implement.md` path;
   - commit and push a durable checkpoint;
   - `complete` the Kanban card with commit hash, artifact path, and exact verification evidence.
6. **After completing any dependency card, immediately re-check board state** and reclaim/block newly auto-started mutating children if they landed in the canonical checkout.
7. **Progress reporting:** report concise status with branch HEAD, done/todo/blocked/running counts, last verified gate, and the explicit reason parallel workers remain paused.

## Verification checklist before claiming a card done

- `git status --short --branch` reviewed.
- Target test command from the card passed.
- Focused regressions for touched surfaces passed.
- Full suite/static/secret scan passed for cross-cutting/shared infra changes.
- Artifact written and staged if RDD artifacts are tracked.
- Commit pushed to remote.
- Kanban card completed with evidence.

## Pitfalls

- Completing a parent/gate can auto-promote children immediately; do not assume the board remains idle after `complete`.
- `--workspace worktree` in a task display is not proof of isolation. Verify actual worker cwd/path before allowing mutation.
- Avoid telling the user the swarm is safely parallel if implementation cards are actually controller-integrated sequentially; say so plainly.
