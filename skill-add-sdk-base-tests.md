---
name: add-sdk-base-tests
description: Add a strong baseline Spread test suite to one or more SDK repos from repos.yaml. Forks/clones each canonical SDK using the standard tmp/ workflow, inspects sdkcraft.yaml and SDK hooks to design repo-specific workshop tests, adds tests/spread.yaml plus focused tests/main/* tasks for launch, PATH, version/toolchain, cache/configuration, and setup-project idempotency where applicable, validates with sdkcraft pack and sdkcraft test, commits, pushes to the user's fork, and opens a PR against canonical main. Use when the user asks to add base tests, improve SDK test coverage, add Spread/workshop tests, or create initial tests for an SDK repo.
---

# Add SDK base tests

Add a useful baseline Spread test suite to selected SDK repos, then open a PR per repo against `main`.

The goal is not just to prove that `sdkcraft pack` succeeds. The tests should launch a real workshop with the freshly packed SDK and verify the core user-facing behaviours that would catch regressions in SDK installation, shell setup, project setup hooks, cache placement, version pinning, and basic toolchain/runtime usage.

Use the existing test PRs as examples of the expected quality and shape:

- `canonical/rust-sdk#10`
- `canonical/go-sdk#13`
- `canonical/copilot-sdk#17`
- `canonical/opencode-sdk#18`
- `canonical/codex-sdk#14`

## Inputs

- **repos** (optional): subset of repo names or URLs from `repos.yaml` to process. Defaults to **all** repos in the inventory. Accept any of: short name (`rust-sdk`), `owner/name` (`canonical/rust-sdk`), or full URL. If the user names a repo that is not in `repos.yaml`, stop and ask before proceeding — do not silently process it.
- **bases** (optional): explicit Ubuntu bases to include in `tests/spread.yaml`, e.g. `ubuntu@22.04 ubuntu@24.04 ubuntu@26.04`. If not supplied, derive the base list from `sdkcraft.yaml` and any existing `.github/workflows/upload.yml` platform list. Prefer testing all supported bases for host architecture when practical.
- **open_pr** (optional): defaults to yes. If the user only wants local changes, skip push and PR creation but still validate and summarise.

### Example invocations

- `/add-sdk-base-tests rust-sdk` — add baseline tests to `rust-sdk` only.
- `/add-sdk-base-tests go-sdk codex-sdk` — process those two repos.
- `/add-sdk-base-tests` — process every repo in `repos.yaml` that does not already have a suitable test suite.
- `/add-sdk-base-tests --bases ubuntu@24.04 ubuntu@26.04 opencode-sdk` — limit the test matrix to those bases.

## Per-repo workflow

For each target repo (`canonical/<repo>`):

### 1. Setup

Follow `AGENTS.md` for the fork/clone/remotes workflow. Do not call `gh repo fork canonical/<repo>` blindly:

1. Determine the GitHub user:
   ```sh
   gh api user --jq .login
   ```
2. Check whether `<user>/<repo>` already exists:
   ```sh
   gh api repos/<user>/<repo>
   ```
3. Interpret the result:
   - **404 (not found)**: run `gh repo fork canonical/<repo> --clone=false`.
   - **200, `fork: true`, `parent.full_name == "canonical/<repo>"`**: reuse the existing fork.
   - **200, `fork: true`, but `parent.full_name` differs**: stop and surface the conflict.
   - **200, `fork: false`**: a same-named non-fork exists. Stop and ask the user how to proceed. Do not call `gh repo fork`, because GitHub may create `<repo>-1` silently.
4. Clone `<user>/<repo>` into `tmp/<repo>/` if it does not already exist. If it exists, fetch both remotes.
5. Ensure `origin` points to the user's fork and `upstream` points to `https://github.com/canonical/<repo>`.

### 2. Branch off upstream main

```sh
git fetch upstream
git checkout -B test/add-base-spread-suite upstream/main
```

If the branch already exists locally, reset it to `upstream/main` only after checking that it does not contain uncommitted user work.

### 3. Inspect the SDK before writing tests

Read and understand the repo before creating files:

- `sdkcraft.yaml`
- `README.md`
- `.gitignore`
- `.github/workflows/upload.yml`, if present
- `hooks/` or any setup/check-health files used by the SDK
- existing `tests/`, if present
- `VERSION`, if present on `main`; otherwise check whether `upstream/latest:VERSION` exists

Identify:

- SDK `name:` and `title:` from `sdkcraft.yaml`.
- The primary command(s) the SDK exposes, from hooks and README examples.
- How the SDK is added to `PATH` (`/etc/profile.d/<name>.sh`, shell init files, symlinks, or another mechanism).
- Whether the SDK has a `setup-project` hook and what files it writes under `/home/workshop`.
- Whether the SDK has a `check-health` hook and what it checks.
- Any cache directories or environment variables configured by `setup-base`, such as `GOMODCACHE`, `GOTOOLCHAIN`, Cargo registry/cache paths, package manager caches, model caches, or config directories.
- Whether the SDK requires a version from `VERSION`, `sdkcraft.yaml`, an upstream release, or a downloaded binary.

Do not create generic tests that only assert files exist if a stronger user-facing behaviour can be tested.

### 4. Derive the Spread base matrix

Create or update `tests/spread.yaml` with a `BASE/*` matrix that matches the SDK's supported bases.

Use this shape unless the repo already has a better project-specific structure:

```yaml
project: <sdk-name>

path: /home/ubuntu/project

backends:
  lxd:
    systems:
      - ubuntu-24.04:
          username: ubuntu
          vm: true

suites:
  main/:
    summary: Test the <sdk-name> SDK inside workshops with different bases
    environment:
      BASE/jammy: ubuntu@22.04
      BASE/noble: ubuntu@24.04
      BASE/resolute: ubuntu@26.04
```

Rules:

- Include only bases the SDK actually supports, unless the user explicitly requested otherwise.
- Use Ubuntu codenames in the `BASE/<codename>` axis. Known mappings include:
  - `BASE/focal: ubuntu@20.04`
  - `BASE/jammy: ubuntu@22.04`
  - `BASE/noble: ubuntu@24.04`
  - `BASE/resolute: ubuntu@26.04`
- For SDKs that read `VERSION` during tests but do not keep `VERSION` on `main`, add:
  ```yaml
  EXPECTED_VERSION: '$(HOST: cat "`git rev-parse --show-toplevel`/VERSION")'
  ```
  Then fetch `VERSION` locally from `upstream/latest` before validation. Do not commit the local `VERSION` unless the repo already tracks it on `main`.
- Keep the LXD backend at `ubuntu-24.04` unless the repo has a documented reason to use another host system.

### 5. Add common test scaffolding

Ensure `.gitignore` ignores SDK build artefacts and Spread reuse state:

```gitignore
# sdkcraft build artifacts
*.sdk

# spread reuse state (created when a previous run leaves instances around)
.spread-reuse.yaml
```

Preserve existing `.gitignore` entries and comments. Do not reorder unrelated entries.

Each task should normally live at:

```text
tests/main/<task-name>/task.yaml
tests/main/<task-name>/workshop.yaml.in
```

Use this `workshop.yaml.in` shape for most tasks:

```yaml
name: test-<sdk-name>
base: ${BASE}
sdks:
  - name: try-<sdk-name>
```

The SDK name must match the packed SDK's `sdkcraft try` name. If the repo's actual try name differs, use the real name.

Most tasks should use this lifecycle:

```yaml
prepare: |
  envsubst '${BASE}' <workshop.yaml.in >workshop.yaml
  workshop launch

restore: |
  workshop remove || true
```

Add a `debug:` section that prints the most useful state for the behaviour being tested. Keep debug commands safe and non-destructive.

### 6. Add the baseline tasks

Add a focused set of tasks that together prove the SDK works in a real workshop. Prefer several small tasks over one large opaque task.

#### Required: launch smoke test

Add `tests/main/launch/task.yaml`:

- `prepare` launches the workshop.
- `execute` runs `workshop exec -- true` or another minimal interaction.
- The summary should say it launches a workshop with the SDK.
- The debug section should include `workshop tasks`.

This catches basic pack/try/launch/check-health failures.

#### Required when the SDK exposes a CLI: command-on-path test

Add `<command>-on-path` or `<sdk-name>-on-path`:

- Run `workshop exec -- bash -lc 'command -v <command>'`.
- Match a path ending in `/<command>`.
- If setup-base installs `/etc/profile.d/<name>.sh`, debug should print `PATH` and list that file.

This verifies login-shell setup from the perspective users actually get in `workshop exec`.

#### Required when the SDK has a versioned CLI: version test

Add `<command>-version`:

- Run the CLI's documented version command.
- Match the expected version when it is stable and available through `VERSION` or another reliable source.
- Escape dots before passing versions to `MATCH`:
  ```sh
  EXPECTED_REGEX="${EXPECTED_VERSION//./\\.}"
  workshop exec -- bash -lc '<command> --version' | MATCH "$EXPECTED_REGEX"
  ```
- If the upstream CLI prints a semver with prefixes, suffixes, or build metadata, match a precise stable fragment rather than the entire line.

Do not hard-code a version that will go stale unless that is already the repo's convention.

#### Required for language/toolchain SDKs: realistic compile/build task

For runtimes and languages, add a task such as `compile-program` that writes a small realistic project inside the workshop and exercises the toolchain end to end.

The task should usually cover:

- creating a project with multiple source files or packages/modules;
- resolving at least one external dependency into the SDK's configured cache;
- a unit test command (`go test`, `cargo test`, `npm test`, `dotnet test`, etc.) when practical;
- a build command;
- running the built program and checking its output.

Good examples:

- Go: multi-package module using `github.com/google/uuid`, `go mod tidy`, cache check under `/home/workshop/go/pkg/mod`, `go test ./...`, `go build`, run the binary.
- Rust: crate using `uuid`, `cargo fetch`, registry cache check under `/home/workshop/.cargo/registry/cache`, `cargo test`, `cargo build`, run the binary.

Keep the program small but non-trivial enough to exercise the runtime, linker, dependency resolver, and cache paths.

#### Required for SDK-specific environment/configuration: setup-base assertions

Add focused tasks for important configuration installed by `setup-base`, such as:

- toolchain state (`rustup show active-toolchain` starts with `stable-`);
- environment variables (`go env GOMODCACHE`, `go env GOTOOLCHAIN`);
- shell completions or profile snippets;
- package manager cache directories;
- expected config files created for the workshop user.

Name these by behaviour, e.g. `go-env-vars`, `rust-default-toolchain`, `rustup-completions`.

#### Required when setup-project writes prompt/rules files: setup-project tests

If the SDK writes agent prompt/rules files during `setup-project`, add two tests:

1. A fresh-write test, such as `agents-md`, `copilot-instructions`, or another product-specific name:
   - Verify the file exists in the expected workshop-user path.
   - Match the expected prompt fingerprint, usually `Workshop Container Environment Instructions`.
2. A preservation/idempotency test, such as `agents-md-preserves-user-content`:
   - Launch the workshop.
   - Pre-seed or overwrite the file with unrelated user content.
   - Run `workshop restore` to re-run setup hooks.
   - Assert that user content is preserved or appended according to the SDK's intended behaviour.
   - Run `workshop restore` again and compare hashes to prove idempotency.

Choose the expected preservation semantics from the actual hook implementation:

- If the hook appends its prompt to user content, assert the original first line is still present and the prompt fingerprint is present.
- If the hook must leave existing files untouched, assert the file remains byte-for-byte user content.

Do not copy another SDK's semantics blindly.

### 7. Keep tests deterministic and maintainable

- Avoid real external services unless the SDK's normal package/dependency resolver necessarily uses public package indexes. Do not use authenticated APIs.
- Do not require user secrets or tokens.
- Prefer commands that work on both `amd64` and `arm64` unless the SDK is explicitly single-arch.
- Match stable output fragments rather than full verbose output.
- Use comments only for non-obvious testing intent, lifecycle quirks, or spread/workshop behaviours.
- Keep task names descriptive and lowercase with hyphens.
- Use `MATCH` patterns carefully; escape regex metacharacters in versions.
- Include useful `debug:` commands for every task.

### 8. Validate locally

Before committing:

1. If `sdkcraft.yaml` or hooks read `VERSION` and `VERSION` is absent on `main`, fetch a local copy:
   ```sh
   git show upstream/latest:VERSION > VERSION
   ```
   This is local-only. Do not commit it unless the repo already tracks `VERSION` on `main`.
2. Run:
   ```sh
   sdkcraft pack
   ```
3. Fetch the private Workshop snap into `tests/` before running tests:
   ```sh
   ( cd tests && snap download workshop )
   ```
4. Run the most useful test first:
   ```sh
   sdkcraft test main/launch:<codename>
   ```
5. Then run the full suite if practical:
   ```sh
   sdkcraft test
   ```
   If the full suite is too slow or the user asked for a narrow validation, run at least one representative base variant and say exactly what was not run.

If validation fails, fix issues caused by the new tests when the root cause is clear. If the failure exposes a real SDK bug, keep the test if it is valid, record the failure, and ask the user whether to include a fix in the same PR or split it.

Before committing, remove local-only artefacts:

```sh
rm -f VERSION
rm -f *.sdk
rm -f tests/workshop_*.snap tests/workshop_*.assert
rm -f .spread-reuse.yaml
```

Only remove `VERSION` if you created it locally and it is not tracked. Confirm with `git status` before staging.

### 9. Commit

Use a conventional commit message:

```text
test: add baseline workshop spread tests

Add a baseline Spread suite that launches the SDK in real workshops and
covers core user-facing behaviour:
- workshop launch with the try SDK
- CLI availability on PATH
- version/toolchain/configuration checks
- setup-project idempotency where applicable
- realistic build/compile workflow where applicable
```

Adjust the bullet list to match what was actually added. Keep the commit focused on tests and ignore-file changes. If you also fix SDK behaviour uncovered by the tests, prefer a separate commit before the test commit with an appropriate `fix:` subject.

### 10. Push to the user's fork

```sh
git push origin test/add-base-spread-suite
```

Do not push to `upstream`.

### 11. Open the PR against canonical

```sh
gh pr create \
  --repo canonical/<repo> \
  --base main \
  --head <user>:test/add-base-spread-suite \
  --title "test: add baseline workshop spread tests" \
  --body "<body>"
```

PR body template:

```md
## Summary
- Add a baseline Spread suite under `tests/` for the <SDK display name>.
- Launch a real workshop with the locally packed try SDK across supported bases.
- Cover <specific behaviours: PATH setup, version checks, toolchain config, cache placement, setup-project idempotency, realistic compile/build workflow>.

## Why
These tests provide base signal for future SDK maintenance by exercising the SDK the way workshop users consume it, rather than only checking that the SDK packs successfully.

## QA
- `sdkcraft pack` passed and produced `<sdk-filename>`.
- `( cd tests && snap download workshop )` completed.
- `sdkcraft test <filter>` passed.
- `sdkcraft test` passed. <!-- Include only if the full suite actually ran. -->
```

Only include QA lines that actually ran and passed. If a valid new test currently fails because it uncovered an SDK bug, say that explicitly instead of claiming success.

## Final summary

After processing all repos, report:

- Repos with PRs opened, including PR URLs.
- Repos skipped because they already had comparable baseline tests.
- Repos where test design needs manual input, with the unclear behaviour.
- Repos where `sdkcraft pack`, Workshop snap download, or `sdkcraft test` failed, including the failing command and branch name.
- For each PR, list the main behaviours covered by the new tests and the validation that passed.
