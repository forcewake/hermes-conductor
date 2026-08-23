# Prototype Furnace daily survivor digest

Use this when a scheduled job asks for Pavel's Prototype Furnace survivor digest for the `zai-prototype-furnace` board / `prototype-furnace` repository.

## Durable workflow

1. Treat the digest as a reporting task, not an implementation task. Do not mutate prototype artifacts unless explicitly asked.
2. Load/consult the Red Queen quality bar before scoring claims. The bar is: one user, one painful job, one magic moment, one front-door workflow, sample data/config, demo proof, README evidence.
3. Verify repository state before reporting links or counts:
   - `git status --short --branch`
   - `git fetch origin main` then compare local `HEAD` with `origin/main`
   - verify the remote URL matches `ai-slop-company/prototype-furnace` when that repo is requested.
4. Count only real committed artifacts. Prefer `git ls-files`/tracked files over filesystem globs, because active Furnace runs may leave untracked workspaces that look substantial but are not durable evidence.
5. Read today's and yesterday's tracked artifacts under:
   - `runs/<date>-*/` Markdown files
   - `survivors/<date>-*.md`
   - `graveyard/<date>-*.md`
   - `runs/<date>-*/standalone-repo.json`
6. Distinguish these counts:
   - Runs: tracked run directories in the date window, including incomplete runs if committed research/hypothesis artifacts exist.
   - Standalone repos: unique `repo_name` values from committed `standalone-repo.json` files, not raw JSON file count. Multiple runs can republish the same idea repo.
   - Red Queen MVP-shaped: artifacts with real survivor-level evidence: front-door entrypoint, sample data/config, generated demo output, README evidence, and a runnable demo. Do not inflate this for script-only or under-extracted artifacts.
   - Script-only / under-extracted: incomplete committed runs, graveyard entries, or artifacts missing front door/sample/generated output/truthful README evidence.
7. Verify per-idea repos before claiming links are current:
   - `gh repo view <owner/repo> --json nameWithOwner,url,isPrivate,defaultBranchRef,pushedAt`
   - `gh api repos/<owner/repo>/commits/main --jq .sha`
   - Compare the remote `main` SHA with `standalone-repo.json.repo_head`.
8. If `repo_head` is stale but the repo exists and later commits are documentation/post-processing commits, report that nuance: the link is current, but the JSON records the prototype publish commit rather than current remote HEAD.
9. For top survivors, summarize only what the artifacts actually prove. Avoid calling script-only, stale-README-command, missing-output, or under-extracted artifacts strong survivors.
10. Ping `Pavel` by name only if a survivor is `>=22/25` or unusually strategic/demoable. Otherwise keep the digest compact.

## Useful extraction snippets

```bash
python - <<'PY'
from pathlib import Path
import subprocess, json
root = Path('~/projects/prototype-furnace')
tracked = set(subprocess.check_output(['git','ls-files'], cwd=root, text=True).splitlines())
dates = ('2026-06-20', '2026-06-21')  # replace for the digest window
run_dirs = sorted({p.split('/')[1] for p in tracked if p.startswith('runs/') and p.split('/')[1].startswith(dates)})
standalone = sorted([p for p in tracked if p.startswith('runs/') and p.endswith('/standalone-repo.json') and p.split('/')[1].startswith(dates)])
unique_repos = []
for p in standalone:
    repo = json.loads((root/p).read_text())['repo_name']
    if repo not in unique_repos:
        unique_repos.append(repo)
print('runs', len(run_dirs))
print('standalone_json', len(standalone), 'unique_repos', len(unique_repos))
print('\n'.join(run_dirs))
PY
```

## Common pitfalls

- Do not use raw recursive file search counts for `runs/`, `survivors/`, or `graveyard`; it will include `.git`, pycache, and untracked active workspaces.
- Do not invent missing smoke/demo results for active runs that only have research and hypothesis committed.
- Do not report a per-idea repo URL as proof of current artifact state without checking GitHub remote state. A URL can be current while `standalone-repo.json.repo_head` is stale.
- Do not equate many files with a strong survivor. The Red Queen bar is workflow/demo/evidence, not volume.