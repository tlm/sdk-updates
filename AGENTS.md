# sdk-updates

This repo coordinates bulk mechanical updates to SDK artifacts that we publish. The managed repos are listed in `repos.yaml`.

## Workshop references

All skills should consult these canonical references when a task depends on
Workshop or SDKcraft commands, SDK definitions, configuration, architecture,
or runtime behaviour:

- **Workshop documentation:** <https://ubuntu.com/workshop/docs/> — the source
  of truth for documented user-facing behaviour, including CLI commands,
  workshop and SDK definitions, SDKcraft usage, SDK development, interfaces,
  and troubleshooting.
- **Workshop source repository:** <https://github.com/canonical/workshop> — use
  the implementation, tests, and documentation source under `docs/` when the
  public documentation does not answer a question or implementation details
  matter.
- **SDKcraft source repository:** <https://github.com/canonical/sdkcraft/> — use
  the implementation, tests, and project schema when SDKcraft behaviour or
  definition details require source-level confirmation.

Documentation for SDK definitions and SDKcraft can be found through the
Workshop documentation, including its SDK development guides and SDKcraft
command reference. Prefer current documentation over assumptions or remembered
command syntax. Check the source repositories when observed behaviour and
documentation disagree, and report any unresolved discrepancy rather than
guessing. Clone a source repository only when local source inspection is
necessary, following the workflow below.

## Working with a managed repo

Before performing any work on a repo from `repos.yaml`, or on another
Canonical repo named by a skill, follow this workflow. The user is expected to
have write access to the canonical repository, so no fork is created. The
same location and naming rules apply to supporting or upstream repositories
cloned for research. Skills should reference this section rather than repeat
its rules.

1. **Clone the repository directly** into `tmp/<repo-name>/` at the root of
   this repository. `<repo-name>` is exactly the GitHub repository name,
   without a `.git` suffix or extra suffixes such as `-upstream`, `-fork`, or
   `-qa`. For example, `canonical/opencode-sdk` goes in `tmp/opencode-sdk/`
   and `anomalyco/opencode` goes in `tmp/opencode/`. Never clone into `/tmp`
   or any other location. `tmp/` is gitignored. For managed repos this makes
   `origin` point at `https://github.com/canonical/<repo-name>`.
2. **Do not add an `upstream` remote.** Canonical base branches are reached
   through `origin`, for example `origin/main` and `origin/latest`.
3. **Reuse a clone only when it matches the repository.** Check that the
   existing `tmp/<repo-name>/` has the expected `origin` URL, then fetch
   `origin`. If an old clone has `origin` pointing at a personal fork and
   `upstream` pointing at canonical, re-point `origin` at canonical and
   remove `upstream` only after checking for unpushed work. If unsure,
   re-clone.
4. **Branch from the canonical base**:

   ```
   git fetch origin --prune
   git checkout -B <branch> origin/<base>
   ```

Non-repository scratch directories may use a suffix only when a skill names
one explicitly, for example `tmp/<repo-name>-qa/`. Never clone a repository
into a suffixed directory.

### Branch naming

Name branches after the task, not the repository or the user. When a skill
provides a branch name, use it. Otherwise use `<type>/<short-description>`,
where `<type>` matches the conventional commit type and the description is
lower-case kebab-case identifying the change and its scope. Include a version,
channel, or dependency name when that distinguishes the task.

Examples: `add-ubuntu-26.04-base`, `chore/update-craft-providers`,
`fix/vscode-remote-health-check`. Avoid vague names such as `update`, `fixes`,
or `work`.

### Pushing and pull requests

- Push only the task branch: `git push -u origin <branch>`.
- **Never push directly to `main`, a channel branch such as `latest`, or any
  release/version branch.** These branches change only through pull requests.
  The only exception is when the user explicitly instructs a direct push and
  names the target branch.
- Open a PR between branches in the canonical repository:

  ```
  gh pr create --repo canonical/<repo> --base <base> --head <branch> ...
  ```

  Omit the `<user>:` prefix from `--head`; the branch lives in the canonical
  repository.
- If pushing or opening the PR fails because of permissions, stop and report
  the failure. Do not create a fork or push to a personal remote as a
  workaround unless the user asks.

## SDK source conventions

### Version files

- `VERSION` belongs on release/version branches. Never commit it to `main`.
- When local validation on `main` requires a `VERSION`, fetch it temporarily
  from the applicable release branch. If no release branch exists yet, create
  it temporarily using the version agreed in the design. Never stage or commit
  this temporary file on `main`, and remove it before the final status check.

### Interface names

- For new SDKs, and for new plugs or slots added to an existing SDK, prefix
  each interface name with the SDK's runtime name. This is normally the
  repository name without the `-sdk` suffix. For example, `hindsight-sdk`
  uses `hindsight-data`, `hindsight-config`, and `hindsight-api`.
- Do not prefix singleton interfaces whose names are fixed by their contract,
  including `gpu`, `desktop`, `camera`, and `ssh-agent`.
- Do not rename existing committed plugs or slots merely to follow this
  convention; preserve their names for compatibility unless the user
  explicitly asks for a rename. The prefix rule applies only to new SDKs and
  newly introduced interfaces.

## SDK testing practices

- `workshop exec` already runs commands through a login shell as the
  workshop user. Invoke commands directly, such as
  `workshop exec -- command`; do not add `bash -lc` merely to load profile
  configuration.
- Only invoke an explicit shell when the test requires its specific syntax
  or behaviour.
- Keep tests version-independent when the behaviour is shared across SDK
  versions. Prefer asserting effective behaviour over implementation
  details.
- Let `workshop remove` in the test's `restore` phase tear down workshop
  services. Only stop services explicitly when their lifecycle is under
  test or shutdown ordering matters.
