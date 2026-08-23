# Kanban research swarm artifact pattern

Use this when the user asks for multi-agent deep research / architectural analysis, especially when the work combines local repo inspection, web research, and a final synthesis.

## Trigger

- User asks to “spawn teams of agents in kanban mode”.
- The task has 3+ independent research lanes that can run in parallel.
- Results must be durable and auditable as files on disk, not only chat replies.

## Pattern

1. Discover available profiles and existing boards first:

```bash
hermes profile list
hermes kanban boards list
```

If a board/repo for the same workstream already exists, inspect it and continue it rather than spawning a duplicate swarm:

```bash
hermes kanban --board <slug> list
hermes kanban --board <slug> stats
hermes kanban --board <slug> show <task_id>
hermes kanban --board <slug> runs <task_id>
hermes kanban --board <slug> log <task_id> | tail -120
```

Use `--board <slug>` on every command when the active board is unrelated; `hermes kanban boards switch` may not affect already-running shells or expectations in compacted sessions.

2. Choose the current project board and workspace. For repo-centered research, prefer:

```text
--workspace dir:/absolute/path/to/repo
```

This makes workers use the same checkout without creating implementation worktrees. Use worktrees only when workers will edit code.

3. Create a durable artifact directory before spawning workers:

```bash
mkdir -p .hermes/research/YYYY-MM-DD-<slug>
```

Write `00-orchestration.md` with:

- topology: swarm / kanban;
- user question;
- source repo/path;
- constraints and guardrails;
- expected worker artifacts;
- final synthesis artifact.

4. Fan out independent parent tasks. Typical lanes:

- local repository/current architecture inspection;
- external concept/prior-art research;
- domain-specific research;
- configuration/autonomy/operations research.

Each worker body should name the exact artifact file it must write, for example:

```text
DELIVERABLE: Write .hermes/research/<slug>/02-state-machine-and-rule-engines.md
```

5. Create one synthesis task with all parent task ids as `--parent` dependencies. The synthesis body should list the parent artifact paths and the final decision questions.

6. If the user is in Telegram and wants updates in the current topic, subscribe each task explicitly:

```bash
hermes kanban notify-subscribe <task_id> \
  --platform telegram \
  --chat-id <chat_id> \
  --thread-id <thread_id> \
  --notifier-profile default
```

Use only known chat/thread ids from the active session context. If unknown, skip explicit subscription and report how to follow tasks.

7. Dispatch once and report the task graph:

```bash
hermes kanban dispatch
hermes kanban list
```

## Output discipline

Final response after spawning should include:

- topology;
- task ids + assignees + lane descriptions;
- artifact directory;
- which tasks are running vs gated;
- whether notifications were subscribed;
- no premature conclusions before synthesis finishes.

When synthesis completes, read the final artifact and send it as `MEDIA:/absolute/path` if the platform supports files, with a short analysis in chat.

For blueprint/repository swarms, the parent/orchestrator must do a final independent verification pass after workers complete: expected file list, Markdown local-link check, JSON fence parse check, source-URL grounding check, secret-like pattern scan, `git status`, commit, push, and remote privacy/tree verification. Do not rely only on a reviewer card saying “passed” when the user asked for a private repo deliverable.

## Pitfalls

- Do not invent profile names. The dispatcher silently ignores unknown assignees.
- Do not create the synthesis task as ready/independent. Gate it with `--parent` on every research lane.
- Do not use code-editing worktrees for pure research unless workers will change files.
- Do not let workers only “summarize in Kanban comments”; require durable artifact files.
- Do not call it an implementation RDD unless code changes are planned. This is a research swarm, not a PR workflow.
