# Kanban single-repo MCP swarm lessons

Use this reference when a user asks to use Hermes Kanban to implement a multi-slice feature inside one repository, especially MCP/server integrations.

## Preflight

1. Create or switch to a project-specific board with a default workdir.
2. Verify available assignees and profile-local skills before creating cards.
3. Run one cheap delegate/worker smoke before fan-out.
4. Keep a canonical integration branch name in the plan.

## Card creation pitfalls

- Unknown `--skill` names can crash workers before useful work starts. If a skill is important, verify it is resolvable in the worker profile or omit the `--skill` and embed the required instructions in the card body.
- `--workspace worktree` is not enough by itself if the board/default workspace resolves to the same checkout. After the first dispatch, inspect `hermes kanban show <task>` and `git branch --show-current` to verify where the worker actually ran.
- Avoid parallel mutation of the same checkout. If workspace isolation is not real, sequence dependent cards or reclaim/pause cards before integrating prerequisites.

## Review-required loop

Workers may correctly block with `review-required` after a local commit or uncommitted slice. The orchestrator/controller should:

1. Inspect `hermes kanban show <task>` and `hermes kanban runs <task>`.
2. Verify the commit/artifacts locally with the task's targeted commands.
3. Run a small cumulative lane for nearby slices when shared files exist.
4. If a worker's optional read-only reviewer fails for provider/model routing or setup reasons, do not keep retrying the same broken reviewer. Record the deviation in the task completion note, perform a degraded controller review with stronger local verification, and proceed only if the evidence is clean.
5. Mark the card `complete` with exact evidence.
6. Dispatch dependents only after completion.

## Integration pattern

For one cohesive feature branch:

- Let early cards produce local commits, but integrate them into a canonical branch before server/docs/final cards.
- If a completed card lives on a side branch, cherry-pick or rebase it into the current integration lineage before dispatching cards that depend on it.
- Verify file-level equivalence for cherry-picked side-branch work when the final commit SHA differs from the original worker commit.
- If the canonical checkout already has important uncommitted baseline work and the user has not approved a commit, do **not** send workers to clean `HEAD` worktrees blindly. Capture a baseline patch from the canonical dirty state, create explicit per-card worktrees, apply the baseline patch there, and copy only the reviewed slice deltas back to the canonical checkout. Exclude secrets and `.env` values from all printed output.
- When comparing worker output against a dirty canonical checkout, compare the worker worktree to the canonical checkout by file/path to isolate the slice delta; raw `git diff --stat` in the worker includes the whole baseline patch and is misleading.
- Final synthesis should record the final branch, commit ancestry, verification commands, and any divergence from worker branch SHAs.

## Verification checklist

Minimum final checks for a Python MCP integration:

```bash
python -m pytest -q
python -m ruff check .
python -m compileall -q src/<mcp_package>
docker compose -f docker-compose.<name>.yml config
python -m py_compile openwebui-tools/chat_context.py
```

Also add source greps for unsafe patterns relevant to the feature, for example:

```bash
grep -RIn --exclude-dir=.git --exclude-dir=.venv 'response\._content' src/<mcp_package> || true
grep -RIn --exclude-dir=.git --exclude-dir=.venv -E 'testsentinel|AZSENTINEL|DLSENTINEL|Bearer testsentinel' src/<mcp_package> || true
```

Capture results in `07-synthesis.md` and do not push/open PR unless the user explicitly approves.