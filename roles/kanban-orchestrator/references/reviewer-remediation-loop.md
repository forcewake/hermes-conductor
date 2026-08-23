# Reviewer Remediation Loop for Kanban Swarms

Use this when a review/final-review card blocks with concrete findings and a downstream integration/PR card is waiting.

## Pattern
1. Inspect the blocking review card, especially the latest comment body:
   - `hermes kanban show <review_task_id> --json`
2. Treat the reviewer comment as the acceptance contract for the next patch.
3. Apply fixes on the implementation branch or the relevant worker worktree.
4. Add regression tests that use the *real object shapes* named by the reviewer, not only synthetic dicts. Common issue: code accepts `source_name` dicts in tests while production objects expose `.name` / `.kind`.
5. Run focused tests for the touched contract, then the reviewer-focused suite, then the full suite before unblocking.
6. Commit the remediation.
7. Comment on the same review task with:
   - commit SHA / subject
   - exact focused suite result
   - exact full suite result
   - one-line explanation of what changed
8. Unblock the review task:
   - `hermes kanban unblock <review_task_id>`
9. Poll the review and dependent tasks until either:
   - review reaches `done` and the child integration/PR card promotes, or
   - review blocks again with new findings.
10. Repeat without frustration. Review loops can catch successively narrower contract gaps.

## Comment template

```text
Remediation applied on <branch>: <one sentence fix summary>. Added regression coverage for <real production object/edge>. Verification: <focused tests> -> <N passed>; reviewer focused suite -> <N passed>; full test suite -> <N passed>. Please re-run review.
```

## Pitfalls
- Do not unblock with only prose; include a commit and verification evidence.
- Do not treat passing synthetic-dict tests as enough when reviewer cites real dataclass/router objects.
- Do not create a new review task unless the graph requires it; usually comment + unblock the existing blocked review task so its dependents remain correctly linked.
- Do not declare final completion until the final review task and PR task are both `done`, the PR is open, and PR checks are verified.
