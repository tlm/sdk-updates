---
name: create-sdk
description: Design and build a new Workshop SDK from scratch for one or several pieces of software. Refreshes the canonical Workshop skill and SDK template references on every run, interviews and researches upstream sources, obtains approval for a design, initialises a local git repository, implements the SDK in coherent increments with rationale-focused commits, validates it through sdkcraft and real workshops, adds behavioral Spread tests, and writes the README. Use when the user asks to create, scaffold, or build a new SDK that does not yet exist. Local source creation only; GitHub repository creation, remote operations, release automation, and publishing are out of scope.
---

# Create an SDK from scratch

Design and build a new local SDK source repository for one or more software
components. Reuse the canonical Workshop guidance and templates, but keep this
skill's local-only scope and incremental-commit requirements authoritative.

The work is complete only when the SDK's advertised behavior has been exercised
in a real workshop and the final Spread results are known. A plausible source
tree or a successful build alone is not completion.

## Inputs

- **name** (required): intended SDK name, normally ending in `-sdk`.
- **software** (required): one or more components to package. Accept names,
  upstream URLs, source checkouts, or precise descriptions. Ask when the exact
  upstream artifacts cannot be identified.
- **directory** (optional): new local repository path. Defaults to
  `tmp/<repo-name>/`, where `<repo-name>` is the SDK name without a `.git`
  suffix. The path must be absent or empty.
- **versions** (optional): requested upstream versions or compatibility
  constraints for the components.
- **bases / architectures** (optional): Ubuntu bases and target architectures.
  Derive recommendations from upstream support when omitted.
- **design_only** (optional): defaults to no. When yes, stop after the Design
  Proposal without creating or modifying the target repository.

Example invocations:

- `/create-sdk example-sdk https://github.com/example/example`
- `/create-sdk example-sdk package the Example CLI and its example-agent sidecar`
- `/create-sdk example-sdk --design-only https://github.com/example/example`

If the name, component set, or intended target is ambiguous, ask before writing
anything. Do not infer that similarly named upstream projects are equivalent.

## Scope boundaries

In scope:

- researching and designing the SDK;
- creating a new local source tree and initialising local git;
- making coherent local commits as the design is implemented;
- definitions, parts, hooks, wrappers, services, interfaces, tests, and README;
- local builds, try-SDK workshops, and behavioral validation.

Out of scope:

- creating or configuring a hosted repository;
- adding remotes, pushing, opening pull requests, or changing hosted settings;
- release-track branches, repository onboarding, and release automation;
- publishing or distributing the SDK;
- copying the template repository's GitHub workflows or Renovate configuration.

Do not improvise work in an out-of-scope area. The local SDK must not be left
with an `origin` remote.

## 1. Refresh the canonical references on every run

Every new SDK creation run must refresh both canonical references before the
requirements interview or design begins, including when familiar local clones
already exist. Do not rely on previously read content, cached knowledge, or an
old checkout:

- `https://github.com/canonical/use-workshop-skill`
- `https://github.com/canonical/template-sdk`

Follow the repository workflow and safety rules in `AGENTS.md`. Clone absent
repositories directly into `tmp/use-workshop-skill/` and `tmp/template-sdk/`.
For existing clones, verify the expected `origin` URL and check for local or
unpushed work before changing anything.

Fetch both repositories and prune stale remote refs:

```sh
git -C tmp/use-workshop-skill fetch origin --prune
git -C tmp/template-sdk fetch origin --prune
```

Then resolve the freshly fetched default branches and record their commit SHAs.
Both currently use `main`:

```sh
git -C tmp/use-workshop-skill rev-parse origin/main
git -C tmp/template-sdk rev-parse origin/main
```

Read guidance and templates from those fetched commits, for example with
`git -C tmp/use-workshop-skill show origin/main:<path>`, rather than from a
potentially stale or modified working tree. Follow cross-references within the
same fetched commit. If either repository cannot be cloned or refreshed, stop
and report the failure; do not silently continue with old reference data.

Use the reference repositories for different purposes:

- `use-workshop-skill`: SDK design doctrine, schema facts, implementation
  patterns, source templates, hook contracts, try-loop procedures, and tests.
- `template-sdk`: the consumer README structure and the initial `.gitignore`.
  Its repository automation is outside this skill's scope.

Include both resolved reference SHAs in the final report so the design's source
of truth is auditable.

## 2. Load only the relevant canonical guidance

Start with these files under
`.github/skills/design-sdk/` in `use-workshop-skill`:

- `workflows/design-sdk.md`;
- `references/design-best-practices.md`;
- `references/reference-sdk-patterns.md`;
- `references/sdkcraft-definition.md`;
- `references/anti-patterns.md`.

Before generation, also load:

- `workflows/generate-sdk.md`;
- `references/runtime-hooks.md`;
- the relevant `templates/minimal/`, `templates/binary/`, or
  `templates/service/` source templates;
- `templates/hooks/` for only the hooks required by the design.

For interface decisions, load
`.github/skills/use-workshop/references/interfaces.md`. Load `secrets.md` from
the appropriate skill when credentials are involved.

Before the try loop and tests, load:

- `workflows/iterate-and-debug.md`;
- `workflows/write-spread-tests.md`;
- `references/spread-tests.md`;
- the sibling Workshop command, asynchronous recovery, and definition-file
  references required by those workflows.

This skill's scope overrides any onward repository-onboarding or distribution
steps referenced by those documents. Do not follow those onward steps.

## 3. Preflight and requirements interview

Check the availability of the tools needed for the intended work, including
`git`, `sdkcraft`, `workshop`, and `shellcheck`. Do not install or upgrade tools
outside the workspace without permission. Check target-directory safety before
generation: it must be absent or empty, and an existing non-empty directory is
a stop condition rather than something to clean or overwrite.

State what the request already establishes, then ask only unresolved questions.
Batch the questions into one message and provide a recommendation and reason
for each. Settle the applicable points:

- the exact software components and whether they form one cohesive SDK;
- the primary component and the role of every auxiliary component;
- upstream distribution, tags or releases, download URLs, checksums, and
  licenses;
- component versions and compatibility constraints;
- required Ubuntu bases and architectures;
- executable entry points, toolchains, daemons, and expected user workflows;
- persistent caches, configuration, models, or other state;
- network services, hardware, credentials, and cross-SDK resources;
- the user-facing capabilities that tests must prove.

For several components, include a component matrix in the working design:

| Component | Role | Upstream distribution | Version or constraint | Packaging mechanism |
|---|---|---|---|---|

Verify distribution facts against upstream sources. Never guess a release URL,
architecture mapping, checksum, or compatibility claim. Mark unresolved facts
as `UNVERIFIED` and ask when they block a safe design.

When a requested hardware capability is necessary for validation, detect and
validate the corresponding resource before using it. If it is unavailable,
stop the related execution and report what is missing; never replace the
capability with a stub or claim it was tested.

## 4. Produce an approved Design Proposal

Keep the design phase read-only with respect to the target. Match the design to
named reference patterns and documented SDKcraft constructs rather than
inventing fields or flags.

The proposal must cover:

- **Components:** what is included and why it belongs in the same SDK.
- **Pattern:** the closest reference SDK patterns and the reasons they fit.
- **Platforms:** base and architecture layout.
- **Parts and sources:** each part, plugin, upstream source, integrity check,
  and staged content.
- **Licences:** the SDK's own content is declared as `Copyright <current-year>
  Canonical Ltd.` under LGPL-2.1, and every shipped upstream component is
  named with its licence, verified against the upstream repository.
  Unverified licences are marked `UNVERIFIED`.
- **Versioning:** `adopt-info` plus a bare, one-line `VERSION` for the primary
  component on release branches, the target branch for this work, and explicit
  independent pins for secondary components where needed. Do not commit
  `VERSION` on `main`; follow the branch and temporary-validation convention
  in `AGENTS.md`.
- **Artifact locking:** whether `sources.lock.json` will pin downloaded
  artifacts by component, architecture, URL, and SHA-256; the origin of the
  initial hashes; and the version-to-lock consistency checks. Recommend a lock
  for prebuilt artifacts when reproducibility is required. If such artifacts
  will not be locked, state explicitly that dynamically discovered checksums do
  not pin the artifact originally reviewed for the SDK source revision.
- **Interfaces:** each plug or slot, its purpose, endpoint or target, and exact
  connection behavior. Classify each as required or optional for the
  consumer's primary workflow, and name the consumer-side input it needs;
  the README must carry the same classification.
- **Hooks:** the minimal hook set and each hook's responsibility.
- **Services:** user-level systemd units where applicable.
- **Persistence and health:** state that must survive updates and the real
  functionality checked by `check-health`.
- **Tests:** launch plus behavioral checks for every advertised capability.
- **Implementation increments:** a provisional sequence of coherent commits,
  including the rationale each increment is expected to capture.
- **Unverified:** unresolved facts, or `none`.

Do not create the target or initialise git until the user explicitly approves
the proposal. For `design_only`, stop here.

## 5. Compose the implementation into logical increments

After approval, turn the proposal into a short task plan made of coherent units
of work. Each unit should answer a meaningful design or implementation question
and leave a reviewer able to understand what changed and why.

The order is adaptive, not a prescribed assembly line. Depending on the SDK, an
increment might cover:

- repository initialisation and the minimal SDK scaffold;
- packaging one component and establishing its trusted upstream source;
- packaging a second component with its compatibility constraints;
- the environment and runtime hooks;
- a service and its health behavior;
- persistence or another interface together with its behavioral test;
- the remaining test matrix and consumer documentation.

Tests and documentation may belong in the same commit as the behavior they
explain, or in a later coherent increment. Do not split work mechanically by
file, and do not postpone every commit until the end.

When using an artifact lock, update a component's version and its corresponding
lock entries in one coherent increment. Record the upstream release and the
source of the new hashes in the commit body so reviewers can assess their
provenance. Never resolve a mismatch by replacing the committed hash with an
unreviewed hash from the artifact that happened to download.

An intermediate commit does not need to compile, pass `sdkcraft pack`, or be
independently runnable. It may intentionally capture incomplete or temporarily
broken work when that best represents a coherent idea or design decision and
later planned increments are expected to make the final SDK work. State known
gaps or breakage in the commit body and name the intended follow-up when useful.
This permits honest intermediate states, not arbitrary unreviewed snapshots:
the final source must still go through the complete validation workflow, with
any remaining failure explicitly reported.

For each increment:

1. State its purpose before starting and how it relates to the approved design.
2. Make only the changes belonging to that unit of work.
3. Run the relevant checks available at that stage. Early scaffold commits need
   not claim that not-yet-implemented behavior works.
4. Review the complete diff for correctness, scope, secrets, generated files,
   and unintended changes.
5. Commit the increment using the protocol below before moving to the next.

A correction is another honest increment. Do not rewrite history to make the
development process look different from what happened.

## 6. Commit protocol

Initialising git and every commit require confirmation immediately before the
mutation, even when earlier mutations were approved. Never amend, rebase,
reset, force-push, or otherwise rewrite history as part of this skill.

Before each commit, when applicable:

```sh
git status --short
git diff
git log --oneline -10
```

Stage only the intended paths, inspect the staged diff, and then create one
commit for the coherent increment. Do not stage SDK archives, downloaded
artifacts, local Workshop state, credentials, or unrelated files.

Use a concise conventional-commit subject and a body that explains the reasoning
behind the change. Depending on the increment, the body should capture:

- why the component, pattern, source, hook, interface, or test is needed;
- important constraints or upstream facts that influenced the implementation;
- alternatives considered and why they were rejected, when that aids review;
- intentional limitations or work left to a later increment.

For example:

```text
feat: package the Example CLI

Use the checksum-verified binary pattern so the SDK ships the pinned
upstream CLI rather than installing it into writable user state at launch.
Separate the upstream Linux architecture names from SDKcraft's names.
```

The message must describe the actual change and reasoning; do not use generic
subjects such as `update files` or bodies that merely repeat the file list.
The sequence of commits should expose the cognitive composition of the SDK,
not a stream of unexplained checkpoints.

## 7. Initialise the local repository and scaffold

Create the approved target directory, then initialise it without adding a
remote:

```sh
git init -b main
```

Confirm this mutation immediately before running it. Do not clone
`template-sdk` into the target: its history, remote, and repository automation
are not the new SDK's source scaffold.

Build the initial tree from the freshly fetched templates. Depending on the
approved design, the repository may contain:

```text
sdkcraft.yaml
sources.lock.json
.gitignore
README.md
hooks/
services/
tests/spread.yaml
tests/main/launch/task.yaml
tests/main/launch/workshop.yaml.in
```

On a release branch, `VERSION` contains one bare upstream version string and
nothing else. On `main`, never commit `VERSION`. When local validation requires
it, create a temporary file and remove it before the final status check as
specified in `AGENTS.md`.

Create `sources.lock.json` only when artifact locking is part of the approved
design; it is not required for every SDK.

Only create directories and files that serve the design. Add wrappers or other
component-specific source directories when needed. Do not create `.github/` or
`renovate.json`.

Use `adopt-info` with a version-owning part reading the bare `VERSION`; do not
hardcode the primary version in `sdkcraft.yaml`. Keep secondary component
versions explicitly pinned and traceable to the component matrix.

When approved, use this lock structure so each part/component has an explicit
version and one source record per architecture:

```json
{
  "artifacts": {
    "example": {
      "version": "1.2.3",
      "sources": {
        "amd64": {
          "url": "https://example.com/releases/v1.2.3/example-linux-amd64.tar.gz",
          "sha256": "<expected-amd64-sha256>"
        },
        "arm64": {
          "url": "https://example.com/releases/v1.2.3/example-linux-arm64.tar.gz",
          "sha256": "<expected-arm64-sha256>"
        }
      }
    }
  }
}
```

The primary component's lock version must equal `VERSION`. An auxiliary
component's lock version must agree with its independent pin. Obtain initial
hashes from a verified upstream source and record their provenance in the
relevant commit. The lock is committed source of truth, not a generated build
artifact.

Start `.gitignore` from the canonical template and include at least:

```gitignore
# sdkcraft build artifacts
*.sdk

# spread reuse state
.spread-reuse.yaml
```

Copy YAML shapes from the selected canonical template rather than synthesising
schema from memory. Make every hook executable and ShellCheck-clean. Commit the
scaffold as the first coherent increment, including why that pattern and
platform layout were selected.

## 8. Implement the approved design incrementally

Follow the approved component and increment plan, adjusting it openly when
upstream facts or validation reveal a better decomposition.

- Put the SDK's immutable deliverables in parts; do not install the primary
  software into writable user state from hooks.
- Put distribution packages and global environment setup in `setup-base`.
- Put user/project configuration and user-service enablement in
  `setup-project`.
- Use `save-state` and `restore-state` only for state that needs them and is not
  already handled by a mount.
- Ship daemons as user-level systemd units rather than background processes
  started directly by hooks.
- Verify upstream checksums whenever available.
- Keep credentials out of source, hooks, generated files, commits, and test
  logs; use the documented secret-interface patterns when required.
- Make `check-health` probe real functionality and return actionable output.
- Walk the canonical anti-pattern reference after generation and after material
  design changes.

### Enforce the artifact lock when present

When the approved design includes `sources.lock.json`, every downloaded
artifact covered by that design must be verified against it. The lock is
authoritative, never advisory. Add `jq` to the relevant part's
`build-packages` and read the lock as `$CRAFT_PROJECT_DIR/sources.lock.json`.

For each lock-enabled artifact, the part must:

1. Parse and validate the JSON, selecting the component and
   `$CRAFT_ARCH_BUILD_FOR` explicitly.
2. Require exactly one source record containing an expected HTTPS `url` and a
   `sha256` string of 64 hexadecimal characters.
3. Verify the lock version against the bare `VERSION` for the primary component,
   or against the relevant independent pin for an auxiliary component.
4. Download from the committed URL, not from a URL newly discovered at build
   time.
5. Calculate the downloaded artifact's SHA-256 and compare it literally with
   the committed hash before staging or using the artifact.

All of the following are serious, fail-closed build errors:

- an expected-versus-actual hash mismatch;
- a missing component or architecture entry;
- a missing or malformed URL or hash;
- malformed JSON or ambiguous source records;
- a version disagreement;
- an architecture that the approved lock does not cover.

On a mismatch, print the component, architecture, source URL, expected hash,
and actual hash; remove the downloaded artifact; and make the build fail
nonzero. Missing or invalid lock data must also fail with an actionable
diagnostic.

Never fall back to a live checksum manifest, automatically update the lock,
substitute the newly calculated hash, or warn and continue. Packing must treat
the lock as read-only. A live manifest may be inspected separately by a
developer proposing a reviewed lock change, but it is not an alternative source
of truth during `sdkcraft pack`, `sdkcraft try`, or `sdkcraft test`.

Commit each coherent increment with its rationale. A failed experiment that is
reverted before review need not become a commit; a meaningful implementation
correction should be a new commit rather than an amendment.

## 9. Validate in a real workshop

Follow the canonical try-refresh workflow using the freshly loaded command
references. When validating source intended for `main`, first create the
temporary `VERSION` required by the build as specified in `AGENTS.md`; never
commit it, and remove it before the final status check.

At minimum:

1. Build into the try area with `sdkcraft clean && sdkcraft try` and confirm the
   try artifact was produced.
2. Consume it as `try-<name>` in a test workshop on a supported base.
3. Launch with verbose failure handling, then inspect the Workshop change, its
   tasks, and the final status.
4. Exercise every component's primary workflow and every advertised capability
   with real commands through `workshop exec`.
5. Verify service, interface, persistence, and state behavior where applicable.
6. After a `setup-base` change, recreate the workshop rather than assuming a
   refresh reran the hook.
7. Turn successful manual checks into focused Spread jobs. Always include
   launch, and test behavior rather than mere file existence.
8. Run `sdkcraft pack` for the final source tree on each declared architecture,
   confirming that every lock-enabled artifact matches its committed hash.
9. When a lock is present, prove the fail-closed path where practical: in a
   scratch copy or test fixture, alter an expected hash or the artifact and
   confirm that packing fails with the intended diagnostic and does not stage
   the unverified artifact. Never leave the real repository's lock modified.
10. Run `sdkcraft test` and report every job and base variant as pass, fail, or
    genuinely blocked.

Do not weaken a behavioral assertion merely to obtain a green result. Fix the
SDK, or obtain the user's agreement that the advertised capability itself must
change. Put each meaningful fix and its rationale into a coherent commit.

If tooling, hardware, upstream availability, or another environmental issue
prevents validation, stop the affected work and report the exact limitation.
Never describe an untried SDK as complete.

## 10. Write the consumer README

Use `README.md` from the freshly fetched `canonical/template-sdk` default
branch as the structural reference. Remove template commentary and all
placeholders, omit sections that do not apply, and keep the result concise and
consumer-facing.

Ensure that:

- the overview agrees with `sdkcraft.yaml`;
- examples use the actual SDK and command names and have been tested;
- plugs, slots, required connections, and persistence match the implementation;
- host commands are distinguishable from commands inside the workshop;
- upstream documentation and license information are included where relevant;
- the README is the consumer's quickest path to a working SDK, and every
  manual step they must perform is documented as an explicit procedure;
- each interface is documented with what it represents, whether it is
  required or optional, how it connects, and what the user must provide;
- the README does not claim hosted availability or describe out-of-scope work.

### Quick-start orientation

The README's primary job is to get a consumer from nothing to using the SDK's
primary workflow as fast as possible. Structure it so the fastest working
path comes first:

- lead with the overview and a reference workshop that works as-is;
- list the consumer's real prerequisites before any workflow steps:
  accounts, credentials, host requirements such as a keyring or desktop
  session, and any other SDKs the workflow depends on;
- give one working path to the primary workflow; link upstream documentation
  for alternatives and detail rather than enumerating every option;
- state what the SDK does to the consumer's environment on launch and
  refresh, and what persists across workshop updates;
- every command shown must have been exercised during validation; a command
  that could not be exercised must not be presented as tested.

### Manual user workflows

Document every action the consumer must perform that Workshop cannot do
automatically, as a numbered procedure. For each step, state:

- where it runs, on the host or inside the workshop;
- when it is needed: once per host or tailnet, at each launch, after each
  refresh, or on first use;
- what the consumer must provide, and where the value comes from, such as
  an upstream account, a host keyring entry, or an interactive prompt;
- what success looks like, and what the SDK does once the step completes;
- what stays broken or degraded when the step is skipped.

Typical manual steps include connecting manual-only interfaces, one-time
upstream setup such as registering an OAuth application, supplying
credentials, and interactive authorisation flows. Never present a
manual-only interface as automatic: give the exact `workshop connect`
invocation, and note that a `connections:` entry in the workshop
definition does not connect interfaces whose policy forbids it.

### Interface documentation

Document every plug and slot from `sdkcraft.yaml`, using the same names.
For each:

- what it represents, in plain language, and its endpoint or target;
- whether it is required or optional: required means the SDK's primary
  advertised workflow cannot work without it; optional means the SDK
  remains usable with it unconnected;
- how it connects: automatically at launch and refresh, or the exact manual
  `workshop connect` command;
- what the consumer must provide on their side, for example a `system` SDK
  slot with lookup `attributes` for a `secret` plug, a system-side tunnel
  plug in the workshop definition, or a host keyring entry.

If the SDK defines no plugs or slots, say so explicitly rather than
dropping the section.

### Licence and copyright section

Reference SDKs carry no top-level `LICENSE` file; the licence declaration
lives in the README's "License and copyright" section, following the
canonical template. Reproduce the established convention:

- The SDK's own scaffolding content is declared as:

  Copyright <current-year> Canonical Ltd.

  This program is free software: you can redistribute it and/or modify it
  under the terms of the
  [GNU Lesser General Public License version 2.1 (LGPLv2.1)](https://www.gnu.org/licenses/old-licenses/lgpl-2.1.html)
  as published by the Free Software Foundation.

  This program is distributed in the hope that it will be useful, but
  WITHOUT ANY WARRANTY; without even the implied warranty of
  MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU
  Lesser General Public License for more details.

- Then name every shipped upstream component with its licence and a link
  to the licence text, as verified against the upstream repository during
  research. Never infer a licence from a project's name or ecosystem.
  Example for a shipped binary:

  [Tailscale](https://github.com/tailscale/tailscale) is licensed under
  the [BSD 3-Clause License](https://github.com/tailscale/tailscale/blob/main/LICENSE).

- Do not add a top-level `LICENSE` file; current reference SDKs keep the
  declaration in the README only.

Commit documentation separately only when it is a coherent increment; otherwise
include it with the behavior and tests it documents.

## 11. Final review and report

Review the finished repository, not just the final diff:

- `git status --short` shows no unintended files;
- `git log --oneline --decorate` presents a coherent sequence of increments;
- commit messages and bodies explain the important decisions and their why;
- `git remote -v` is empty;
- hooks are executable and relevant checks have passed;
- on a release branch, `VERSION` contains only the bare primary version; on
  `main`, no `VERSION` is committed and no temporary copy remains;
- any approved `sources.lock.json` is valid, consistent with its component
  pins, and has no dynamic-checksum fallback in the parts that consume it;
- the README's "License and copyright" section names the SDK's own
  content under LGPL-2.1 and every shipped component's verified upstream
  licence, with no top-level `LICENSE` file committed;
- no generated archives, credentials, hosted automation, or template
  placeholders were committed.

Report:

- the SDK name and local path;
- the two canonical reference SHAs used;
- the components and versions selected;
- whether artifact locking was used, the provenance of the committed hashes,
  and the positive and fail-closed validation results;
- the increments and commits created;
- the build, workshop, and per-job Spread validation results;
- any unverified facts, blocked checks, or remaining limitations.

## Success criteria

- [ ] Both canonical references were freshly fetched for this run, and their
      resolved SHAs were used and reported.
- [ ] Upstream facts were verified and the user approved the Design Proposal
      before target files or git history were created.
- [ ] A new local git repository was initialised with no remote.
- [ ] The implementation was decomposed into coherent increments, each
      committed separately with an accurate rationale-focused message.
- [ ] The commit sequence makes the composition and important decisions easy
      for a reviewer to follow without rewriting history.
- [ ] The source matches the approved design and canonical SDKcraft patterns.
- [ ] On a release branch, `VERSION` remains a bare, one-line primary version;
      on `main`, no `VERSION` is committed or left behind after validation.
- [ ] Every lock-enabled artifact was checked against its committed URL and
      SHA-256 on each declared architecture; missing or mismatched lock data
      fails closed with no dynamic-checksum fallback.
- [ ] Every advertised capability was exercised in a real workshop, or any
      blocker was explicitly reported.
- [ ] Final Spread jobs cover launch and real behavior, with per-job results.
- [ ] The README matches the implemented SDK.
- [ ] The README leads the consumer quickly to a working primary workflow,
      documents every manual step with where, when, and what to provide,
      and covers each interface with its required or optional status and
      connection guidance.
- [ ] The README licence section follows the reference-SDK convention
      (LGPL-2.1 for SDK content, verified upstream licences for shipped
      components) and no top-level `LICENSE` file was committed.
- [ ] No hosted repository, remote, release automation, or distribution work
      was performed.
