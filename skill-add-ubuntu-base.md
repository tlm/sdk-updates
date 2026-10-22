---
name: add-ubuntu-base
description: Add support for a new Ubuntu LTS build base across one, several, or all SDK repos in repos.yaml. Detects which sdkcraft.yaml shape each repo uses (single `build-base:` / `base:` field vs. multi-base `platforms:` list), makes the appropriate edit, also updates `.github/workflows/upload.yml` for multi-base repos, builds with `sdkcraft pack`, runs `sdkcraft test` when a `tests/` directory exists, runs an end-to-end `workshop launch` probe on the new base when the host arch matches, commits, pushes the task branch to canonical, and opens a PR against `main`. Use when the user asks to "add 26.04", "bump build-base", "add a new Ubuntu LTS", or similar.
---

# Add Ubuntu base

Update each selected SDK repo so it builds against a new Ubuntu LTS base, then open a PR per repo against `main`.

## Inputs

- **version** (required): the Ubuntu LTS version, e.g. `26.04`. If not provided in the user's message, ask before proceeding.
- **repos** (optional): subset of repo names (or URLs) from `repos.yaml` to process. Defaults to **all** repos in the inventory. Accept any of: the short name (`flutter-sdk`), the `owner/name` form (`canonical/flutter-sdk`), or the full URL. If the user names a repo that isn't in `repos.yaml`, stop and ask before proceeding — do not silently process it.

### Example invocations

- `/add-ubuntu-base 26.04` — add `ubuntu@26.04` support to every repo in `repos.yaml`.
- `/add-ubuntu-base 26.04 flutter-sdk` — only `flutter-sdk`.
- `/add-ubuntu-base 26.04 flutter-sdk uv-sdk` — those two repos only.

## Per-repo workflow

For each target repo (`canonical/<repo>`):

### 1. Setup

Follow the "Working with a managed repo" workflow in `AGENTS.md`. Clone
`canonical/<repo>` into `tmp/<repo>/` if it is not already present; otherwise
fetch `origin`. Do not create a fork or add an `upstream` remote.

### 2. Branch off canonical main
```
git fetch origin --prune
git checkout -B add-ubuntu-<version>-base origin/main
```

### 3. Detect the sdkcraft.yaml pattern

Read `sdkcraft.yaml` at the repo root and classify it:

- **Pattern A — `build-base` bump**: contains a top-level `build-base: ubuntu@<X.Y>` line.
- **Pattern B — `base` bump**: contains a top-level `base: ubuntu@<X.Y>` line and **no** `build-base:` line.
- **Pattern C — additive platforms**: `platforms:` keys look like `ubuntu@<X.Y>:<arch>:` (multi-base shorthand). No `build-base:` field.
- **Already on target**: the file already references `ubuntu@<version>` in the relevant place (Pattern A/B value equals target; Pattern C has at least one `ubuntu@<version>:` row). Skip this repo and note it in the final summary.
- **Unknown**: none of the above match. Stop processing this repo and surface it for manual review — do not guess.

### 4. Apply the edit

**Pattern A** — replace the existing `build-base: ubuntu@<X.Y>` value with `ubuntu@<version>`. Record the previous value for the commit/PR body.

**Pattern B** — replace the existing `base: ubuntu@<X.Y>` value with `ubuntu@<version>`. Record the previous value.

**Pattern C** — additive:
- Identify the arch set to mirror. Look at the **most recent** existing version's arch entries (e.g. if both `ubuntu@22.04:amd64:` and `ubuntu@24.04:amd64:` exist, the most recent is 24.04 with `[amd64]`; if `ubuntu@24.04:amd64:` and `ubuntu@24.04:arm64:` both exist, the most recent is 24.04 with `[amd64, arm64]`). Mirror exactly that set into the new version.
- Insert the new `ubuntu@<version>:<arch>:` rows at the **top** of the `platforms:` block (newest release first), preserving the file's indentation and key style (some entries are empty mappings — keep that). The convention across the inventory is newest-to-oldest in `sdkcraft.yaml`.
- Then update `.github/workflows/upload.yml`: for **every** `platforms: '[...]'` JSON array that contains strings of the form `ubuntu@<X.Y>:<arch>`, append `ubuntu@<version>:<arch>` for each arch that already appears in that specific array. The JSON array convention is oldest-to-newest (append at the end), which is the opposite of `sdkcraft.yaml` — match each file's existing order rather than forcing one direction. Some repos (e.g. `rust-sdk`) split arches across multiple jobs — handle each array independently. If a `platforms:` array uses bare arches (`["amd64"]` with no version prefix), leave it alone — that array is base-agnostic.

After editing, sanity-check with `git diff` to make sure the change matches the pattern's expected shape (one line for A/B, additive rows for C plus optional workflow change).

### 5. Verify the build

**VERSION caveat (most SDKs).** Many SDKs in the inventory keep `VERSION` only on the `latest` branch — `main` is deliberately VERSION-less. If `sdkcraft.yaml`'s `override-pull` reads `$CRAFT_PROJECT_DIR/VERSION`, pack will fail on the bump branch unless you drop a local copy first:

```
git show origin/latest:VERSION > VERSION
```

This is **local-only**: do **not** `git add` it, and ensure it's gone before the commit in step 8 (`rm VERSION`). If the repo's `main` already carries a `VERSION` file, skip this step.

Run `sdkcraft pack` in the repo root. If it fails, record the failure and stop processing this repo (leave the working tree in place so the user can inspect — do not revert the edit).

### 6. Run tests if present
If a `tests/` directory exists at the repo root:

1. **Extend the spread test matrix if applicable.** If `tests/spread.yaml` defines a `BASE/<codename>: ubuntu@<X.Y>` axis under `suites.<suite>.environment:` (the convention across the inventory), append a matching entry for the new version using the Ubuntu release codename (e.g. `BASE/jammy: ubuntu@22.04`, `BASE/noble: ubuntu@24.04`, `BASE/resolute: ubuntu@26.04`). Without this, `sdkcraft test` silently skips the new base — there's no spread environment to match it against. If `tests/spread.yaml` doesn't use a `BASE/*` axis, skip this sub-step. This change becomes a second small commit in step 8 (`test:` type) so the platform bump and the test-coverage change are reviewable independently.
2. **Prerequisite — fetch the Workshop snap into `tests/`.** The test suite requires a local copy of the Workshop snap (it's a private snap, so the test runner cannot fetch it itself):
   ```
   ( cd tests && snap download workshop )
   ```
   If the snap is already present, `snap download` will overwrite it harmlessly.
3. **Run the test filtered to the new base.** `sdkcraft test` with no filter packs every host-arch platform and runs every spread variant — slow for a bump validation. Filter to just the new variant:
   ```
   sdkcraft test main/<task>:<codename>
   ```
   e.g. `sdkcraft test main/launch:resolute`. Sdkcraft still packs every host-arch platform (that's a sdkcraft-side behaviour, not a spread filter), but only the matching spread variant runs. If it fails, handle as in step 5.

### 7. Workshop launch QA probe (host-arch only)

Try the freshly packed SDK in a real workshop on the new base. This catches runtime problems that `sdkcraft pack` alone won't.

**Most repos already have a `tests/` directory with a `BASE/*` matrix that exercises the new base on host arch in step 6 — when that's the case, step 7 is automatically satisfied and should be skipped.** Step 7 exists for repos where step 6 didn't cover the new base end-to-end (no `tests/` directory, no `BASE/*` axis in `tests/spread.yaml`, or the test run skipped this variant).

**Skip conditions** (note the reason in the final summary, then go to step 8):
- Step 6 already exercised the new base on host arch (a `BASE/<codename>` variant was added in 6.1 and `sdkcraft test main/<task>:<codename>` passed) — this is the common case.
- `workshop` is not on `PATH`.
- The host arch (`dpkg --print-architecture`) is not in the arch set added in step 4. The probe needs to launch a workshop on the host's native arch.

Otherwise:

1. Locate the packed SDK file for `<host-arch>` and the new version. Pattern C produces `<name>_<host-arch>_ubuntu@<version>.sdk`; Pattern A/B produces `<name>_<host-arch>.sdk`. The file is at the repo root after step 5.
2. `sdkcraft try <path-to-sdk>` — copies it into the workshop try area as `try-<name>`. The output line confirms the SDK name.
3. Create a sibling QA workspace `tmp/<repo>-qa/` (sibling to `tmp/<repo>/`). Write a minimal `workshop.yaml` modelled on the SDK's `README.md` example, but pinned to the new base and using the try-SDK:
   ```yaml
   name: <name>-try
   base: ubuntu@<version>
   sdks:
     - name: try-<name>
   ```
   Carry over a couple of `actions:` from the README example if present (they document intent; not strictly needed for the probe).
4. From the QA workspace, run `workshop launch`. Then `workshop info` to confirm `status: ready`. If it doesn't reach ready, record the failure and stop — leave the workshop in place so the user can inspect.
5. `workshop exec <name>-try -- bash -lc 'which <primary-binary> && <primary-binary> --version && grep PRETTY_NAME /etc/os-release'`. Pick `<primary-binary>` from the SDK's main CLI (e.g. `cargo` for rust-sdk, `claude` for claude-code-sdk, `opencode` for opencode-sdk, `codex` for codex-sdk). Confirm:
   - the binary resolves under `/var/lib/workshop/sdk/<name>/...`,
   - it reports a version,
   - `PRETTY_NAME` matches the target Ubuntu version.
6. `workshop remove <name>-try` to tear it down.

Capture the evidence (binary path, version line, `PRETTY_NAME`) — it goes into the PR body's QA section in step 10. If any sub-step fails, record it and stop processing this repo.

### 8. Commit

Use [conventional commits](https://www.conventionalcommits.org/) format. If the test matrix was extended in step 6.1, produce **two** commits so the platform change and the test-coverage change can be reviewed independently.

**Commit 1 — the platform bump**, keyed to the pattern:

**Pattern A**:
```
feat: bump build-base to ubuntu@<version>

Update sdkcraft.yaml build-base from ubuntu@<old> to ubuntu@<version> so
the SDK is built against the new Ubuntu LTS release.
```

**Pattern B** — same as Pattern A but substitute `base` for `build-base` throughout.

**Pattern C**:
```
feat: add ubuntu@<version>:<archs> build target(s)

Add ubuntu@<version>:<arch> entries to sdkcraft.yaml platforms and to the
platforms array in .github/workflows/upload.yml so the SDK is built and
published for the new Ubuntu LTS release alongside the existing targets.
```
Where `<archs>` is a comma-joined list (e.g. `amd64,arm64`). If only one arch was added the subject uses the singular form `feat: add ubuntu@<version>:<arch> build target`.

**Commit 2 — test matrix extension (only if step 6.1 added a BASE variant):**
```
test: cover ubuntu@<version> in the <suite> suite

Add `BASE/<codename>: ubuntu@<version>` so the existing <suite-summary>
runs against the new Ubuntu LTS release alongside the others. Without
this, `sdkcraft test` would skip the <version> variant (no spread
environment to match against), leaving the new base untested.
```

Before either commit, **remove any local-only files** dropped during validation:
- `rm VERSION` if you created one in step 5.
- Confirm `git status` shows nothing unexpected staged.

Keep messages simple — no extra prose beyond the templates.

### 9. Push the task branch
```
git push -u origin add-ubuntu-<version>-base
```
Never push directly to `main`, a channel branch, or a release branch.

### 10. Open the PR against canonical
```
gh pr create \
  --repo canonical/<repo> \
  --base main \
  --head add-ubuntu-<version>-base \
  --title "<title>" \
  --body "<body>"
```

Title matches commit 1's subject from step 8 (the platform bump, not the test commit).

PR body template — **Pattern A / B**:
```
## Summary
- Update `sdkcraft.yaml` `build-base` from `ubuntu@<old>` to `ubuntu@<version>` so the SDK builds against the new Ubuntu LTS release.

## Why
Track the latest Ubuntu LTS for the <SDK display name> build environment, in line with the wider effort to migrate Canonical SDKs onto <version>.

<QA section — see below>
```
(Substitute `base` for `build-base` if Pattern B.)

PR body template — **Pattern C**:
```
## Summary
- Add `ubuntu@<version>:<arch>` to the `platforms` list in `sdkcraft.yaml`.
- Add `ubuntu@<version>:<arch>` to the `platforms` array in `.github/workflows/upload.yml` so it is built and published by CI alongside the existing targets.

## Why
Track the latest Ubuntu LTS for the <SDK display name> build environment, in line with the wider effort to migrate Canonical SDKs onto <version>. This SDK uses the multi-base platforms shorthand (each platform key encodes its own base + arch), so adding <version> support is an additive change — existing consumers are unaffected.

<QA section — see below>
```

If multiple arches were added in Pattern C, repeat the bullet per arch (or list them comma-separated within the same bullet — match the style of the existing repo's PR history if there is one).

**QA section** — choose based on what actually ran in steps 5–7:

- **If the workshop probe ran (step 7) — use the captured evidence**:
  ```
  ## QA steps performed
  Verified end-to-end on an <host-arch> host.

  1. `sdkcraft pack --platform ubuntu@<version>:<host-arch>` — produced `<sdk-filename>`.
  2. `sdkcraft try <sdk-filename>` — copied into the workshop try area as `try-<name>`.
  3. Wrote a `workshop.yaml` based on the README example, pinning `base: ubuntu@<version>` and `sdks: [{ name: try-<name> }]`.
  4. `workshop launch` → workshop `<name>-try` reached `Ready`.
  5. `workshop exec <name>-try -- bash -lc 'which <primary-binary> && <primary-binary> --version && grep PRETTY_NAME /etc/os-release'`:
     - `<binary-path>`
     - `<version-line>`
     - `PRETTY_NAME="Ubuntu <version> LTS"`
  6. `workshop remove <name>-try` — clean teardown.

  Other-arch platform(s) not validated locally; CI will exercise them.
  ```
- **If only `sdkcraft test` ran (step 6) and the probe was skipped**:
  ```
  ## QA
  - `sdkcraft pack` produced `<sdk-filename>`.
  - `sdkcraft test` passed against the new base.
  ```
- **If neither ran** (probe and tests both skipped):
  ```
  ## QA
  Not performed.
  ```

**Deriving the SDK display name**: prefer `sdkcraft.yaml`'s `title:` field; if absent, title-case the `name:` field and append " SDK" (e.g. `name: go` → "Go SDK", `name: github-runner` → "GitHub Runner SDK"). Don't fabricate marketing copy beyond that.

**Do not fabricate QA evidence.** Only include numbered QA steps that actually ran end-to-end. If the workshop probe was skipped, do not assemble a fake numbered list — fall back to the shorter template.

## Final summary

After processing all repos, report:
- Repos with PRs opened (include PR URLs; note whether the workshop probe ran or was skipped)
- Repos already on `<version>` (skipped)
- Repos where the pattern was unrecognized (need manual review)
- Repos where `sdkcraft pack`, `sdkcraft test`, or the workshop probe failed (include the failing step and the branch name so the user can pick up the working tree under `tmp/<repo>/`)
