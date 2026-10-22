# sdk-updates

This repo coordinates bulk mechanical updates to SDK artifacts that we publish. The managed repos are listed in `repos.yaml`.

## Working with a managed repo

Before performing any work on a repo from `repos.yaml`, or on another
Canonical repo named by a skill, follow this workflow. The user is expected to
have write access to the canonical repository, so no fork is created.

1. **Clone the canonical repo directly** into `tmp/<repo-name>/` inside this
   repo. `tmp/` is gitignored. This makes `origin` point at
   `https://github.com/canonical/<repo-name>`.
2. **Do not add an `upstream` remote.** Canonical base branches are reached
   through `origin`, for example `origin/main` and `origin/latest`.
3. **Reuse an existing clone** under `tmp/<repo-name>/` by fetching `origin`.
   If an old clone has `origin` pointing at a personal fork and `upstream`
   pointing at canonical, re-point `origin` at canonical and remove
   `upstream` only after checking for unpushed work. If unsure, re-clone.
4. **Branch from the canonical base**:

   ```
   git fetch origin --prune
   git checkout -B <branch> origin/<base>
   ```

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
