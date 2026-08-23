# Single-repo Kanban workspace isolation and recovery

Use this when orchestrating a Kanban swarm inside one Git repository.

## Durable lesson

Do not create parallel implementation cards against the same `dir:<repo>` checkout. Even if cards look independent, the dispatcher can start multiple workers that mutate the same working tree concurrently. That creates hidden test/file races and invalidates per-card evidence.

## Safe creation patterns

### Preferred: isolated worktree per card

Use Kanban `worktree` workspace mode when cards may run in parallel. `--branch` is only valid with `--workspace worktree`; it is rejected with `--workspace dir:<path>`.

```bash
hermes kanban --board <board> create \
  "T1: schema" \
  --body "..." \
  --workspace worktree \
  --branch feat/<slice>-schema \
  --assignee <profile> \
  --idempotency-key <stable-key>
```

### Safe fallback: single canonical checkout

If worktree workers are unavailable or the final result must be integrated as one branch, do not dispatch parallel mutating cards. Use the board as tracking only, then integrate sequentially in one canonical checkout with explicit artifacts and one final verification pass.

## Collision recovery

If cards are already `running` in the same checkout:

1. Stop the collision before further mutation:
   ```bash
   hermes kanban --board <board> reclaim <task-id> --reason "same checkout collision; switching to canonical integration"
   ```
   `reclaim` accepts one task id at a time.

2. Block or complete cards one at a time with evidence. `block` positional reason words can be parsed as task ids if not quoted; prefer a short quoted reason or use comments for longer notes.
   ```bash
   hermes kanban --board <board> block <task-id> "same checkout collision; controller integrating sequentially"
   ```

3. Integrate in the canonical checkout, write per-task artifacts under the parent RDD task directory, run full verification, then complete each card one at a time:
   ```bash
   hermes kanban --board <board> complete <task-id> \
     --result "Integrated in canonical branch; verification: <command> => <observed output>."
   ```
   `complete --summary/--metadata` is per-task; do not use those flags with multiple ids.

## Evidence requirements

When falling back from swarm to canonical integration, record:

- why parallel isolation was unsafe;
- which task ids were reclaimed/blocked/completed;
- the artifact paths for each logical slice;
- the exact final verification command and output;
- the final git status and remote state when pushed.

This is a workflow recovery pattern, not a reason to avoid Kanban. The durable rule is: parallel mutating Kanban work in one repo requires real workspace isolation.