# Controller-managed external-agent worktree lanes

Use this when a Kanban board is the durable state machine, but implementation is launched manually via external coding agents (MiMo, OpenCode, Claude Code, agy, Codex, or another explicitly available CLI) rather than the Hermes dispatcher.

## Controller contract

1. Keep the Kanban card blocked/controller-owned until the worktree exists and the exact CLI launch command is verified.
2. Create a fresh worktree from the verified canonical integration branch.
3. Write a prompt file with task scope, forbidden scope, tests, evidence artifact path, and `do not push/merge`.
4. Launch the CLI as a tracked background process where possible (`terminal(background=true, notify_on_complete=true)`).
5. Treat the agent's report as untrusted until Hermes verifies git state and gates.

## CLI launch examples

```bash
# MiMo implementation or review lane
PROMPT=$(cat /tmp/<task>.prompt.md)
mimo run -m zai-coding-plan/glm-5.2 --variant max --format json --pure -- "$PROMPT" \
  > /tmp/<task>.mimo.jsonl

# Other CLIs: first run `<cli> --help` in this environment, then launch from the isolated worktree.
# Record exact command, model label, worktree path, prompt path, and log path in the card comment.
```

## Verification gates

After the lane exits, the controller must run:

```bash
git -C "$WT" status --short --branch
git -C "$WT" log --oneline --decorate -8
git -C "$WT" diff --stat "$BASE"...HEAD
# then the task's targeted tests and at least one canonical integration gate
```

Only complete the card after:

- expected files/artifacts exist;
- targeted gates pass in the lane worktree;
- the patch is integrated/cherry-picked into the canonical branch;
- canonical gates pass after integration;
- the card result records commit IDs and command output.

## Stale-base and auto-promotion recovery

If a dependency completion auto-promotes downstream cards before their worktrees are recreated from current HEAD:

1. Immediately block/reclaim the downstream card.
2. Inspect `git worktree list`, each worktree `HEAD`, and `git status`.
3. Preserve old worktrees only as untrusted references.
4. Relaunch from a fresh worktree based on the verified canonical branch.

## Artifact hygiene

Exclude scratch artifacts unless independently justified:

- agent logs / raw JSONL unless required as evidence;
- local venv links;
- temporary prompt files;
- accidental lockfile churn unrelated to the task.

For Python suites, rerun hermetically if inherited env vars contaminate the run (SQLite/hash/json/local defaults before judging failures).