# Kanban controller cheatsheet

Commands in the order you'll actually reach for them. All verified against Hermes Agent v0.20.x.

## Board lifecycle

```bash
hermes kanban init                      # create kanban.db (idempotent)
hermes kanban boards list               # every board + counts
hermes kanban --board <slug> list       # always pass the explicit slug on multi-board setups
hermes kanban watch                     # live board updates
```

## Cards

```bash
# create a controller-managed mutating card (isolated worktree)
hermes kanban create "Ship feature X" \
  --assignee <worker-profile> \
  --workspace worktree \
  --branch feat/x \
  --skill research-driven-development \
  --body "Implement per 01-spec.md..."

# pinned real-data workspace instead of a scratch worktree
hermes kanban create "..." --workspace dir:/abs/path/to/repo

# dependencies: child blocked until parent completes WITH evidence
hermes kanban create "Step 2" --parent <step1-id> --assignee <worker>

# goals with judge-evaluated completion
hermes kanban create "..." --goal --goal-max-turns 40
```

## Verification & completion (the part that matters)

```bash
hermes kanban show <id>                 # status, events, runs
hermes kanban tail <id>                 # live worker log
hermes kanban runs                      # execution history

# COMPLETE WITH EVIDENCE — commit IDs, gate output. Never "looks good".
hermes kanban complete <id> --result "commit abc123 · tests 47/47 green · lint clean"

# NOT valid (does not exist — don't let agents guess it):
#   hermes kanban update --status done        ✗

# repair a stale/dishonest completion record
hermes kanban edit --result --summary --metadata ...
```

## Notifications

```bash
hermes kanban notify-subscribe <task-id> \
  --platform telegram --chat-id <chat> --thread-id <topic>

hermes kanban notify-list
hermes kanban notify-unsubscribe <task-id> ...
```

## Recovery (see patterns 05, 06)

```bash
hermes kanban reclaim <id>              # take back an auto-spawned card
hermes kanban block <id>                # stop the dispatcher touching it
hermes kanban archive <id>
hermes kanban restore <archived-name>   # undo an archive
```

## Swarm

```bash
hermes kanban swarm "goal" \
  --worker <profile>:<title>[:skill,skill] \
  --verifier <profile> \
  --synthesizer <profile>
```

> Swarms are the newest, least battle-tested surface. Patterns 03/06 cover the failure modes we've already seen. Start with controller-managed single cards first.

## Dispatcher hygiene

| Setting | Why |
|---|---|
| `failure_limit: 5` | 2 default is too tight for flaky external CLIs |
| standing lane cards → `blocked` with a reason | otherwise the dispatcher spawns uncontrolled one-shots |
| workers stuck on approval → `kanban_block(pending)` | breaks respawn loops |
| scratch workspaces are deleted on completion | pin `dir:` for real data |
