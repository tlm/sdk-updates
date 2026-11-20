---
name: feature-work
description: Implement a focused feature in one SDK repo from repos.yaml. Clones the canonical repo directly using the standard tmp/ workflow, inspects the SDK and its conventions, agrees the scope with the user, implements the change with tests, validates with sdkcraft pack and sdkcraft test where available, commits, pushes the task branch to canonical, and opens a PR against main. Use when the user asks to do feature work on a specific SDK.
---

# SDK feature work

Implement a focused feature in one SDK repo, then open a PR against `main`.

## Inputs

- **repo** (required): the SDK repo to change, normally one listed in
  `repos.yaml`. Accept the short name (`opencode-sdk`), the `owner/name` form,
  or the full URL.
- **feature** (required): what to build and why. The user may provide this in
  the invocation or in linked notes and issues.
- **base** (optional): the branch to target. Defaults to `main`.
- **open_pr** (optional): defaults to yes. If the user only wants local
  changes, still implement and validate but skip push and PR creation.

If the repo or the feature requirements are unclear, ask before changing
anything. Do not infer product decisions or expand the scope.

### Example invocations

- `/feature-work opencode-sdk add a --json output option`
- `/feature-work rust-sdk add support for a custom cache directory`
- `/feature-work github-runner-sdk --base latest add a health-check hook`

## Workflow

### 1. Setup

Follow the "Working with a managed repo" workflow in `AGENTS.md`.

If the requested repo is not in `repos.yaml`, confirm with the user that it is
the intended Canonical repository before proceeding.

### 2. Understand the feature and the SDK

Inspect the repo before editing:

- `AGENTS.md`, `README.md`, and `CONTRIBUTING.md`, if present;
- `sdkcraft.yaml` and any hooks the SDK uses;
- `tests/` and any existing test conventions;
- the primary language tooling and lint/typecheck commands;
- recent `git log` history around the area being changed.

Then state a short plan: the intended behaviour, files likely to change,
tests to add or update, and any risks or open questions. Ask the user to
resolve genuine ambiguities before writing code.

### 3. Create the task branch

```sh
git fetch origin --prune
git checkout -B feat/<short-description> origin/<base>
```

Use a branch name that describes the feature, following the branch naming
guidance in `AGENTS.md`.

### 4. Implement the feature

- Follow the repo's existing patterns, libraries, and style.
- Keep the change focused on the requested feature.
- Add or update tests that exercise the new behaviour.
- Update user-facing documentation when the feature changes usage.
- Avoid unrelated refactors and dependency churn.

Validate incrementally while working rather than waiting until the end.

### 5. Validate

Run the repo's own checks. At minimum:

```sh
sdkcraft pack
```

If a `tests/` directory exists, also run:

```sh
sdkcraft test
```

Run any lint, typecheck, or unit-test commands documented by the repo. If a
failure is caused by the change, fix it. If a failure is pre-existing or
environmental, record the details and ask the user whether to proceed.

### 6. Commit

Use conventional commit messages and keep logical changes in separate commits
where that improves review. Do not commit generated artifacts, local `VERSION`
files, downloaded snaps, or unrelated files.

### 7. Push and open the PR

```sh
git push -u origin feat/<short-description>
```

Never push directly to `main`, a channel branch, or a release branch.

```sh
gh pr create \
  --repo canonical/<repo> \
  --base <base> \
  --head feat/<short-description> \
  --title "<type>: <short feature summary>" \
  --body "<body>"
```

PR body template:

```md
## Summary
- <what the feature does>
- <any notable implementation choice>

## Why
<the user-facing or maintenance reason for the change>

## Testing
- `sdkcraft pack` passed.
- `sdkcraft test` passed. (or why it was not run)
- <lint/typecheck/unit test command> passed.

## Follow-ups
- <known limitations or deferred work, if any>
```

Include issue links or design notes when the user provided them. Omit the
follow-ups section when there is nothing to list.

### 8. Report

After finishing, report:

- the repo, branch, and commits created;
- the validation performed and its result;
- the PR URL, or why push and PR creation were skipped;
- any remaining risks, follow-ups, or questions.
