# Single-repo production swarm recovery: research fan-out, mutating-card collision

Use this reference when a user asks for a large Kanban/swarm implementation in one Git repository and wants progress notifications.

## Durable lessons

1. **Research cards may safely share a checkout if they only write distinct RDD artifacts.** Still subscribe them and verify artifacts before planning.
2. **Mutating implementation cards must not share the canonical checkout.** Even when cards were created with `--workspace worktree`, verify the actual worker workspace after dispatch. If `hermes kanban show <task>` reports `workspace: worktree @ /path/to/canonical/repo` and `git status` in the canonical checkout starts changing, treat it as a same-checkout collision.
3. **Kanban `--parent` is a dependency edge, not merely grouping.** Children stay in `todo` until the parent is completed. For an epic/planning parent, complete it with an “epic initialized” result before dispatching child research lanes.
4. **If mutating workers collide, stop the swarm immediately.** Reclaim running cards, block ready cards to prevent auto-respawn, preserve any produced deltas, and switch to controller-verified canonical integration or true explicit worktree paths.
5. **Do not mark a collided card done just because it produced files.** Run targeted tests, write the per-card artifact under `.hermes/rdd/<slug>/tasks/<card-slug>/05-implement.md`, then complete the card with evidence.

## Safe sequence

```bash
# 1. Create/switch board and create parent epic.
hermes kanban boards switch <board>
hermes kanban create "<Epic>" --body "..." --assignee rddplanner --workspace worktree --branch <epic-branch> --json

# 2. Create deep-research cards with parent=<epic>; subscribe each to the origin chat.
hermes kanban create "R1 ... research" --parent <epic-id> --assignee rddresearcher --workspace worktree --branch <research-branch> --goal --json
hermes kanban notify-subscribe <task-id> --platform telegram --chat-id <chat> [--thread-id <thread>]

# 3. Complete the parent only after child cards exist, so dependencies unblock.
hermes kanban complete <epic-id> --result "Epic initialized; research lanes created."
hermes kanban dispatch --max <N>

# 4. Verify research artifacts before planning/implementation.
find .hermes/rdd/<slug> -maxdepth 1 -type f -name '01*.md' -printf '%f %s bytes\n'
```

## Collision recovery commands

```bash
# Stop already-running mutating cards.
for t in <running-task-ids>; do
  hermes kanban reclaim "$t" --reason "same-checkout collision; switching to canonical integration"
  hermes kanban comment "$t" "Controller reclaimed because mutating workers shared the canonical checkout. Deltas will be verified before completion."
done

# If reclaim makes cards ready, block them to prevent auto-respawn.
for t in <ready-mutating-task-ids>; do
  hermes kanban block "$t" "same-checkout collision; controller integration"
done

# Verify the partial delta before completing any card.
git status --short --branch
python -m pytest <targeted-tests> -q
python -m pytest -q
python -m compileall <packages> <tests>
git diff --check
```

## Progress notification pattern

- Subscribe parent + child tasks with `hermes kanban notify-subscribe` where the channel/thread is known.
- For long-running controller integration, a script-only cron watcher can report only on board/branch-state digest changes. Keep it quiet when unchanged.
- Do not let progress reporting replace verification: report task IDs, artifacts, branch HEAD, and observed command outputs.

## Completion criteria after recovery

A recovered card can be completed only after:

- the code delta is in the canonical branch;
- targeted tests for the card pass;
- the per-card artifact exists under `.hermes/rdd/<slug>/tasks/<card-slug>/05-implement.md` and names the recovery mode;
- the Kanban completion result includes artifact path and exact verification output.
