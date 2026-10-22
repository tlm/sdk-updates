---
name: update-sdkcraft-craft-deps
description: Update sdkcraft's craft-* dependency pins to released minor versions. Clones canonical/sdkcraft directly using the standard tmp/ workflow, evaluates released craft-* dependency updates, reviews release notes before changing pins, updates pyproject.toml and uv.lock only for safe minor-version bumps unless a major bump is explicitly requested, validates the update, commits, pushes the task branch to canonical/sdkcraft, and opens a PR against `main`. Use when the user asks to update sdkcraft's craft dependencies, bring in craft-providers/craft-application/craft-platforms fixes, or refresh sdkcraft's craft-* pins.
---

# Update SDKcraft craft-* dependencies

Update `canonical/sdkcraft` so its craft-related Python dependencies track safe released minor versions, then open a PR against `main`.

This skill is for dependency maintenance in the `sdkcraft` repository itself, not for updating SDK repos listed in `repos.yaml`.

## Inputs

- **dependencies** (optional): specific craft-related dependencies to evaluate. Defaults to all craft-related runtime dependencies in `sdkcraft`.
- **target versions** (optional): explicit target versions. If provided, verify they are released and are not major-version bumps unless the user explicitly requested a major bump.

### Craft-related dependencies

Treat these as in scope when they appear in `sdkcraft`'s dependency files:

- `craft-application`
- `craft-archives`
- `craft-cli`
- `craft-grammar`
- `craft-parts`
- `craft-platforms`
- `craft-providers`
- `craft-store`

Other `craft-*` dependencies may be included if present in `pyproject.toml` or `uv.lock`. Do not update unrelated dependencies unless they are required by the selected craft dependency updates and the resolver changes them.

### Example invocations

- `/update-sdkcraft-craft-deps` — evaluate and update all safe released craft-* minor bumps.
- `/update-sdkcraft-craft-deps craft-providers` — evaluate and update only `craft-providers` and resolver-required transitive changes.
- `/update-sdkcraft-craft-deps craft-providers 3.6.0` — update `craft-providers` to `3.6.0` if released and compatible.

## Workflow

### 1. Setup `canonical/sdkcraft`

Use the "Working with a managed repo" workflow in `AGENTS.md`, even though
`sdkcraft` is not listed in `repos.yaml`. Clone `canonical/sdkcraft` into
`tmp/sdkcraft/` if it is not already present; otherwise fetch `origin`. Do not
create a fork or add an `upstream` remote.

Important machine quirk: if plain `gh` is unavailable, run GitHub CLI commands via:

```sh
nix-shell -p gh --run 'gh ...'
```

### 2. Branch off canonical main

```sh
git fetch origin --prune
git checkout -B chore/update-craft-dependencies origin/main
```

If the user requested a single dependency, use a narrower branch name, for example:

```sh
git checkout -B chore/update-craft-providers origin/main
```

### 3. Identify current dependency state

Inspect:

- `pyproject.toml`
- `uv.lock`

Record current versions for all in-scope dependencies from `uv.lock` and any version constraints from `pyproject.toml`.

Use the lock file as the source of the currently installed/resolved version. Use `pyproject.toml` as the source of declared lower bounds and compatibility constraints.

### 4. Discover released target versions

Only update to versions that have been released by the dependency project.

Acceptable evidence of a released version includes:

- a Git tag in the upstream project matching the version, such as `3.6.0`;
- a published package version available from the package index used by the project;
- the dependency's own release metadata or release notes.

Do not update to an unreleased commit from `main` unless the user explicitly asks for an unreleased/git dependency. In normal maintenance, wait for a release.

### 5. Enforce version-bump policy

By default, apply only patch or minor updates within the same major version:

- `3.3.0` → `3.6.0`: allowed.
- `6.0.1` → `6.4.0`: allowed.
- `0.10.0` → `0.11.1`: allowed as a `0.x` minor bump, but review carefully because pre-1.0 projects may include breaking changes in minor releases.
- `3.x` → `4.x`: not allowed unless the user explicitly requested a major bump.

If the latest release is a major bump, skip it and use the newest released version within the current major series, or stop and ask if no safe target is available.

### 6. Review release notes before editing

Before changing a dependency, review release notes or changelogs for every version crossed between the current and target versions.

Look for:

- breaking changes or migration notes;
- base/image mapping changes, especially Ubuntu release/daily buildd image changes;
- provider behaviour changes for LXD or Multipass;
- lifecycle/build-plan changes;
- changes to Python version requirements;
- changes to dependency constraints;
- known regressions or deprecations.

If release notes are missing, inspect the git log between tags and call this out in the PR body. Do not pretend release notes were reviewed when only commits were reviewed.

For Canonical craft dependencies, useful upstream repositories include:

- `https://github.com/canonical/craft-application`
- `https://github.com/canonical/craft-archives`
- `https://github.com/canonical/craft-cli`
- `https://github.com/canonical/craft-grammar`
- `https://github.com/canonical/craft-parts`
- `https://github.com/canonical/craft-platforms`
- `https://github.com/canonical/craft-providers`
- `https://github.com/canonical/craft-store`

### 7. Decide the update set

Prefer the smallest update set that brings in the desired released fixes.

Examples:

- If the goal is to bring in a `craft-providers` LXD image mapping fix, prefer bumping `craft-providers` only, plus resolver-required lock changes.
- Do not broadly update `craft-application`, `craft-parts`, or `craft-platforms` just because newer versions exist unless their release notes show relevant fixes or the resolver requires them.
- If updating one dependency requires raising a lower bound in `pyproject.toml`, make the lower-bound change explicit and justify it in the PR body.

### 8. Apply dependency updates

Use the project's existing dependency tooling. `sdkcraft` uses `uv.lock`, so prefer `uv` commands when available.

For a focused dependency update, use the narrowest update command supported by the installed `uv` version, for example:

```sh
uv lock --upgrade-package craft-providers
```

If a specific version is required, constrain it in `pyproject.toml` first or use the supported `uv` syntax for an exact target, then regenerate `uv.lock`.

If `uv` is unavailable, install or run it in a way consistent with the project. Do not hand-edit large lock-file sections unless there is no practical alternative.

After updating, inspect the diff:

```sh
git diff -- pyproject.toml uv.lock
```

Confirm the diff only contains:

- intended craft dependency version changes;
- resolver-required transitive dependency changes;
- intended lower-bound changes in `pyproject.toml`.

If unrelated dependencies changed unexpectedly, try a narrower update. If they are unavoidable, explain why in the PR body.

### 9. Validate

Run the most targeted useful validation first, then broader validation using a locally built snap.

#### Unit and source-level checks

Recommended checks:

```sh
uv run python -m pytest tests/unit
```

If the repo's documented test command differs, use the documented command instead.

For provider/image mapping changes, also perform a lightweight behavioural check when practical:

- confirm the relevant mapping in the installed dependency points to the expected remote/image;
- or run a small Python snippet importing the dependency from the updated lock environment;
- or launch/query the relevant LXD remote image if the change is specifically about LXD image selection.

For example, when validating a `craft-providers` fix that changes `ubuntu@26.04` from buildd daily to buildd release, verify that the updated source maps `BuilddBaseAlias.RESOLUTE` to `craft-com.ubuntu.cloud-buildd` rather than `craft-com.ubuntu.cloud-buildd-daily`.

#### Local snap validation

Build the `sdkcraft` snap locally so the dependency update is tested the way users receive it:

```sh
snapcraft pack
```

If the project documents a different snap build command, use the documented command instead. Capture the produced `.snap` filename.

Install the locally built snap for validation:

```sh
sudo snap install --dangerous --classic <sdkcraft-snap>
```

If `sdkcraft` is already installed, record the currently installed revision/channel first so it can be restored after validation:

```sh
snap list sdkcraft
```

Then install the local snap with `--dangerous --classic`. If snap installation is unsafe in the current environment, stop and ask the user before proceeding.

#### Pack representative SDKs with the local snap

Use the locally installed snap to pack several SDKs from `repos.yaml`. Choose a representative subset rather than every repo unless the user asks for exhaustive validation.

Recommended subset:

- one simple SDK with fast packaging, e.g. `direnv-sdk` or `vscode-remote-sdk`;
- one multi-base SDK that exercises `platforms:` entries, e.g. `docker-sdk`, `jupyter-sdk`, `rust-sdk`, `uv-sdk`, or `node-sdk`;
- one SDK from a different category when practical, e.g. an AI agent SDK or `github-runner-sdk`.

For each selected SDK:

1. Use the "Working with a managed repo" workflow in this repo's `AGENTS.md` if the repo is not already present under `tmp/<repo>/`. If the local clone exists, fetch `origin`.
2. Check out a clean branch from `origin/main` or use a detached worktree state that does not modify the repo.
3. If the SDK needs a local `VERSION` file from a release branch, use the convention from `skill-add-ubuntu-base.md`:
   ```sh
   git show origin/latest:VERSION > VERSION
   ```
   Treat this as local-only and remove it after packing.
4. Run `sdkcraft pack` using the locally installed snap.
5. Record the produced `.sdk` filename and the platform/base it exercised.
6. Remove local-only files such as `VERSION` and generated `.sdk` artifacts unless the user wants to inspect them.

If a selected SDK is known to be slow or resource-heavy, either choose a lighter representative SDK or explicitly note why it was skipped. Do not let one slow SDK block all validation when another repo can exercise the same dependency path.

After snap validation, restore the user's previous `sdkcraft` snap if it was replaced and restoration is practical. If restoration requires a specific channel or revision that cannot be inferred safely, ask the user.

Do not claim tests or snap validation passed unless they were run and passed.

### 10. Commit

Use a conventional commit message.

For one dependency:

```text
chore(deps): bump <dependency> to <version>

Update <dependency> from <old-version> to <version>.

Reviewed release notes/changes:
- <version>: <short relevant note>
- <version>: <short relevant note>
```

For multiple dependencies:

```text
chore(deps): update craft dependencies

Update released craft-* dependency pins:
- <dependency>: <old-version> -> <version>
- <dependency>: <old-version> -> <version>

Reviewed release notes/changes:
- <dependency> <version>: <short relevant note>
- <dependency> <version>: <short relevant note>
```

If release notes were unavailable and git history was reviewed instead, say that plainly:

```text
Release notes were unavailable for <dependency>; reviewed git history between <old-tag> and <new-tag> instead.
```

### 11. Push

Push the task branch to canonical:

```sh
git push -u origin <branch>
```

Never push directly to `main`, a channel branch, or a release branch.

### 12. Open the PR

Open against `canonical/sdkcraft:main`:

```sh
gh pr create \
  --repo canonical/sdkcraft \
  --base main \
  --head <branch> \
  --title "chore(deps): <title>" \
  --body "<body>"
```

PR body template:

```md
## Summary
- Update released craft-* dependency pins:
  - `<dependency>`: `<old-version>` -> `<version>`

## Why
- <Short reason, e.g. bring in released `craft-providers` support for using the Ubuntu 26.04 buildd release image instead of the daily image.>

## Release notes / change review
- `<dependency>` `<version>`: <relevant notes or inspected commits>.

## QA
- `<unit/source check command>` passed.
- Built local snap: `<sdkcraft-snap>`.
- Installed local snap with `sudo snap install --dangerous --classic <sdkcraft-snap>`.
- Verified representative SDK packs with the local snap:
  - `<repo>`: `sdkcraft pack` produced `<sdk-filename>` (`<platform/base>`).
  - `<repo>`: `sdkcraft pack` produced `<sdk-filename>` (`<platform/base>`).
- <Any additional behavioural check performed, such as provider image mapping verification.>
```

If some newer releases were intentionally skipped, include:

```md
## Not included
- `<dependency>` `<version>` was skipped because it is a major-version bump and this PR only performs minor updates.
- `<dependency>` `<version>` was skipped because it has not been released yet.
```

## Final summary

After opening the PR, report:

- PR URL;
- dependency versions changed;
- release notes or git history reviewed;
- validation commands run and their results;
- locally built snap filename and whether it was installed/restored;
- representative SDK repos packed with the local snap and the `.sdk` files produced;
- any newer versions intentionally skipped and why.
