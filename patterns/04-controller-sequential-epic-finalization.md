# Controller-sequential epic finalization

Use this reference when a large Kanban epic has been implemented via controller-managed isolated worktrees and must be closed cleanly with verified GitHub handoff.

## Trigger

- A controller-managed epic has child cards completed sequentially.
- Final implementation has been integrated into the canonical branch and pushed.
- The user expects the work to be PR-ready, not merely locally complete.

## Pattern

1. **Finish the final child card before touching the epic.**
   - Parse external review output; do not trust exit code.
   - If a reviewer returns `approve-with-nits`, classify nits technically.
   - Patch cheap real nits (especially test-quality or security-contract issues), rerun gates, and document whether the post-review change is test-only/controller-verified or requires a fresh review.

2. **Run canonical gates after integration, not only worktree gates.**
   - Targeted task tests.
   - Neighbor/API/MCP/job/observability tests where relevant.
   - The task/epic regression bundle.
   - Full hermetic suite.
   - `compileall`, `git diff --check`, tracked binary/Office scan, and added-line secret/content scan.

3. **Write the final review artifact before board completion.**
   Include:
   - external agent/review session ids and exact target commits;
   - worktree candidate commit;
   - reviewer verdict and blockers/nits;
   - any controller post-review nit closure;
   - canonical cherry-pick/integration commits;
   - canonical verification command outputs;
   - scan caveats (distinguish synthetic test fixtures/prose from production content paths).

4. **Push and verify the remote ref.**
   After `git push`, read back `git ls-remote` and compare local HEAD to remote branch SHA before saying it is synced.

5. **Complete the child card with compact evidence.**
   Include final commit ids, reviewer verdict, canonical test counts, and scan status. Avoid pasting raw JSONL or secret-shaped trace ids.

6. **Complete the epic only after all child cards are done.**
   Query the board for remaining `blocked|todo|running|ready|scheduled`. If none remain, complete the epic with the final branch HEAD and canonical gate evidence.

7. **Open or update the PR.**
   - Check whether a PR already exists for the branch.
   - If none exists, create one with a body summarizing scope, security controls, verification, and review evidence.
   - Read the PR back with `gh pr view` and report: number, URL, state, draft status, base/head, head SHA, mergeability, and checks.
   - GitHub may return `mergeable: UNKNOWN` immediately after PR creation. Wait briefly and poll once before reporting final mergeability.

## Pitfalls

- A board can still contain an old blocked epic even when every child card is done; close the epic explicitly after verifying no active child cards remain.
- PR creation success is not enough; read back PR state and head SHA.
- `mergeable: UNKNOWN` right after PR creation is normal async GitHub computation, not a failure. Poll once after a short wait.
- Large PRs can have no status checks configured (`statusCheckRollup: []`); report that as observed state, not as passing CI.
- Scan hits in synthetic redaction tests or anti-leak prose/regex are not production leaks, but call them out explicitly so future reviewers do not chase false positives.
