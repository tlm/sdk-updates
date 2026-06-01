---
name: forward-merge-channel
description: Forward merge `main` into a channel branch (e.g. `latest`) across one, several, or all SDK repos in repos.yaml. Builds with `sdkcraft pack`, runs `sdkcraft test` when a `tests/` directory exists, commits with a conventional commit message, pushes to the user's fork, and opens a PR against the canonical channel branch. Use when the user asks to "forward merge", "update the latest channel", "sync channel branches", or similar.
---

# Forward merge channel

Forward merge `main` into a channel branch across the selected repos in `repos.yaml`, then open a PR per repo against the canonical channel branch.

## Inputs

- **channel** (required): the channel branch name (e.g. `latest`). If not provided in the user's message, ask before proceeding.
- **repos** (optional): subset of repo names (or URLs) from `repos.yaml` to process. Defaults to **all** repos in the inventory. Accept any of: the short name (`claude-code-sdk`), the `owner/name` form (`canonical/claude-code-sdk`), or the full URL. If the user names a repo that isn't in `repos.yaml`, stop and ask before proceeding — do not silently process it.

### Example invocations

- `/forward-merge-channel latest` — forward merge `main` into `latest` on every repo in `repos.yaml`.
- `/forward-merge-channel latest claude-code-sdk` — only `claude-code-sdk`.
- `/forward-merge-channel latest claude-code-sdk codex-sdk` — those two repos only.

## Per-repo workflow

For each target repo (`canonical/<repo>`):

### 1. Setup
Follow `AGENTS.md` — ensure the repo is forked, cloned into `tmp/<repo>/`, and that `upstream` points to `canonical/<repo>`. If the local clone already exists, fetch both remotes instead of re-cloning.

**Resume detection.** Before doing anything else, check whether a prior run left an in-progress merge on a `forward-merge/...` branch:

- `git rev-parse -q --verify MERGE_HEAD` succeeds, **and**
- the current branch matches `forward-merge/main-into-<channel>-*`

If both hold, the previous run stopped on conflicts and the user has been resolving them. Skip steps 2–4 and jump to **Resuming after conflict resolution** below.

### 2. Determine commits to merge
```
git fetch upstream
git log --oneline upstream/<channel>..upstream/main
```
If the output is empty, the channel is already up to date — skip this repo and note it in the summary.

Capture the upstream main short sha:
```
SHA=$(git rev-parse --short upstream/main)
```

### 3. Create the forward-merge branch from the upstream channel tip
```
git checkout -B forward-merge/main-into-<channel>-<SHA> upstream/<channel>
```

### 4. Merge main, deferring the commit
```
git merge upstream/main --no-ff --no-commit
```
If conflicts arise:
- **Do not abort the merge.** Leave the conflicted working tree in place so the user can resolve it manually.
- **Capture the conflicting paths to disk** so the next run (after resolution) can list them in the commit and PR body:
  ```
  git diff --name-only --diff-filter=U > .git/forward-merge-conflicts.txt
  ```
  Keep this inside `.git/` so it's automatically ignored by git.
- Record the repo, the conflicting paths, and the branch name (`forward-merge/main-into-<channel>-<SHA>`) for the final summary of this run.
- Do not commit, push, or open a PR for this repo.
- Continue with the next repo. Conflicts in one repo do not block others (each repo has its own `tmp/<repo>/` directory).

### Resuming after conflict resolution
Entered from step 1 when an in-progress merge is detected on a `forward-merge/...` branch.

- Verify the user has resolved every conflict: `git diff --name-only --diff-filter=U` must be empty. If not, list the still-unresolved files, do not commit, and stop processing this repo.
- Load the captured conflict list from `.git/forward-merge-conflicts.txt`. These are the **resolved-conflict paths** to be referenced in the commit body and PR body in steps 7 and 9.
- Proceed to step 5.

### 5. Verify the build
Run `sdkcraft pack` in the repo root. If it fails, record the failure and stop processing this repo (leave the working tree in place so the user can inspect — do not abort the merge).

### 6. Run tests if present
If a `tests/` directory exists at the repo root:

1. **Prerequisite — fetch the Workshop snap into `tests/`.** The test suite requires a local copy of the Workshop snap (it's a private snap, so the test runner cannot fetch it itself):
   ```
   ( cd tests && snap download workshop )
   ```
   Run this once per repo before `sdkcraft test`. If the snap is already present, `snap download` will overwrite it harmlessly.
2. Run `sdkcraft test` from the repo root. If it fails, handle as in step 5.

### 7. Commit
Use a conventional commit message. The body is two dot-point sections:

```
chore: forward merge main into <channel>

Included commits:
- <sha> <subject>
- <sha> <subject>
...

Resolved conflicts:
- <path>
- <path>
...
```

- **Included commits** is the list captured in step 2 (short sha + subject).
- **Resolved conflicts** is included **only if** this run is a resume from a previous conflict — use the paths loaded from `.git/forward-merge-conflicts.txt`. Omit the section entirely on a clean merge.

Keep it simple — no extra prose. After committing, remove the conflict file: `rm -f .git/forward-merge-conflicts.txt`.

### 8. Push to the user's fork
```
git push origin forward-merge/main-into-<channel>-<SHA>
```
Do **not** push to `upstream`.

### 9. Open the PR against canonical
```
gh pr create \
  --repo canonical/<repo> \
  --base <channel> \
  --head <user>:forward-merge/main-into-<channel>-<SHA> \
  --title "chore: forward merge main into <channel>" \
  --body "<body>"
```
PR body:
```
Forward merge of `main` into `<channel>`.

Included commits:
- <sha> <subject>
- <sha> <subject>
...

Resolved conflicts:
- <path>
- <path>
...
```

Include the **Resolved conflicts** section only when this run resumed from a conflict (same condition as the commit body). Omit it for clean merges.

## Final summary

After processing all repos, report:
- Repos with PRs opened (include PR URLs)
- Repos already up to date (skipped)
- Repos with merge conflicts (list conflicting paths)
- Repos where `sdkcraft pack` or `sdkcraft test` failed (include the failing step)
