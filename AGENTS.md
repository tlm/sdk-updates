# sdk-updates

This repo coordinates bulk mechanical updates to SDK artifacts that we publish. The managed repos are listed in `repos.yaml`.

## Working with a managed repo

Before performing any work on a repo from `repos.yaml`, follow these steps:

1. **Fork to the user's GitHub account** using `gh repo fork <upstream> --clone=false`. If the fork already exists, `gh` exits non-zero with a message indicating it already exists — treat this as success, not an error.
2. **Clone the user's fork** into `tmp/<repo-name>/` inside this repo. `tmp/` is gitignored.
3. **Add the upstream (canonical) remote** as a second remote named `upstream`, pointing at the original `canonical/<repo>` URL. The fork remains as `origin`.

After setup, `origin` is the user's fork (where branches/PRs are pushed from) and `upstream` is the canonical repo (where PRs are opened against).
