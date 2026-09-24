# springtree-ci-workflows

Shared GitHub Actions workflows for SpringTree repositories.

A repository does not copy these definitions; it calls them. One change here
reaches every caller, which is the whole point — a control that has to be
re-applied by hand to each repository is a control that drifts.

The workflows here are the *enforcing* copy of SpringTree's supply-chain
controls. A developer's own setup protects that developer; what CI refuses to
merge is what the control actually rests on.

---

## Calling the supply-chain gates

Add `.github/workflows/compliance.yml` to the repository that should be gated:

```yaml
name: compliance

on:
  pull_request:
  workflow_dispatch:
    inputs:
      full-history:
        description: Scan all history rather than the pull request's range
        type: boolean
        default: false
  schedule:
    - cron: "0 5 * * 1"

jobs:
  supply-chain:
    uses: SpringTree/springtree-ci-workflows/.github/workflows/supply-chain.yml@main
    with:
      # `github.event.inputs`, not `inputs`: the latter does not exist on a
      # pull_request event, and referencing it fails the workflow before any
      # job starts.
      full-history: ${{ github.event_name == 'schedule' || github.event.inputs.full-history == 'true' }}
```

Nothing else is required: no secrets, no tokens, no repository variables.

### Prerequisites

- This repository's **Settings → Actions → General → Access** must allow
  repositories in the SpringTree organisation to use its workflows. Without it
  a caller fails with a "workflow was not found" error that says nothing about
  permissions.
- The caller needs no write permissions. The reusable workflow declares
  `contents: read` and nothing in it writes.

---

## The gates

Four jobs, run in parallel, with no dependency between them — one failure does
not prevent the other three from reporting. A pull request therefore shows every
problem it has in one run rather than one problem per push.

| Job          | What it refuses                                                                         | Tool                    | Sends anything out |
| ------------ | --------------------------------------------------------------------------------------- | ----------------------- | ------------------ |
| `secrets`    | A credential committed anywhere in the scanned range                                      | gitleaks and TruffleHog | No                 |
| `licences`   | A dependency whose licence is not on the allow-list                                       | `@lizenz/checker`       | No                 |
| `code`       | A code pattern the ruleset classes as a security defect, introduced by the change         | Semgrep CE              | No                 |
| `quarantine` | A repository whose committed configuration does not enforce the package-cooldown window   | built in                | No                 |

On the last column: TruffleHog runs with verification disabled, so no candidate
secret is ever sent to a provider to see whether it authenticates. Semgrep runs
with metrics off. Both are deliberate and both are commented at the call site —
read those comments before changing either.

### `secrets`

Two tools rather than one, for the union of their detector sets. Both run in git
mode over a range: a pull request is scanned for what it introduces, and the
scheduled run re-scans all history, which catches a rewritten history a range
scan cannot see.

Filesystem mode is deliberately not used. It has no `.gitignore` support, so it
flags a developer's local `.env` on every run — invisible in CI, where a fresh
checkout has none, and noisy everywhere else. A gate that cries wolf locally
while showing green in CI stops being read.

### `licences`

Reads the installed dependency tree, not GitHub's dependency graph. The graph
does not fully resolve Bun lockfiles in either format, so on a Bun repository it
sees a fraction of what is installed and a licence check built on it would pass
without having looked at most of the dependencies.

Every directory with its own lockfile is installed and checked, not just the
repository root — that is what a lockfile means, and a subdirectory resolving
its own tree is a tree nothing else reads. A workspace package has no lockfile
of its own, so each tree is still installed exactly once.

The allow-list fails closed: a licence nobody has classified is refused until
someone looks at it, and a malformed list refuses everything rather than
permitting everything. A package whose metadata names a licence that is not a
valid SPDX identifier is refused for the same reason — the string cannot be
matched against anything, so it is a question for a person rather than a pass.

A repository with manifests but no lockfile warns rather than fails: nothing can
be installed with `--frozen-lockfile`, so nothing can be read.

#### Clarifications

Some packages name their licence in a form no tool can match — `url-template`
declares `"license": "BSD"`, which is not an SPDX identifier, though its text is
BSD-3-Clause verbatim. A clarification states what the licence actually is:

```json
{
  "url-template@2.0.8": {
    "licenses": "BSD-3-Clause",
    "licenseFile": "LICENSE",
    "checksum": "70723b90e3f26aa2808616e846cd8cd349fe33864fd328a91a5bb9d33e58e9d9"
  }
}
```

This is not a waiver. The entry names one version and carries the SHA-256 of the
licence text it was read from, so if upstream ever changes that text the checksum
stops matching and the gate fails until someone reads it again — a clarification
cannot outlive its evidence. `licenseFile` resolves inside the package's own
directory, so it is `LICENSE`, never `node_modules/…/LICENSE`.

The workflow carries an org-wide set for packages every repository meets, so the
same reading is not re-done in each one. `licence-clarifications` points at a
repository-local file, merged over the shared set with the local file winning.

### `code`

Semgrep with a baseline. A pull request is answerable for what it introduces,
not for everything that was already there — without a baseline, every
pre-existing finding blocks every change, which is how a gate gets switched off
rather than satisfied. The scheduled run has no baseline and sees the lot.

### `quarantine`

A freshly published malicious package version is caught by waiting rather than
by scanning, so both package managers are configured to refuse a version younger
than the quarantine window. This gate checks that the repository carries that
configuration:

| Check                                                            | Verdict on failure |
| ---------------------------------------------------------------- | ------------------ |
| `.npmrc` sets `min-release-age` at or above the window            | fail               |
| `bunfig.toml` sets `minimumReleaseAge` under `[install]`          | fail               |
| `.npmrc` carries no literal credential (an `${ENV}` ref is fine)  | fail               |
| the Bun lockfile is the text format, not `bun.lockb`              | warn               |

**Both files are required beside every `package.json`, at any depth**, not only
at the repository root. That is not belt-and-braces; it is what the two package
managers actually do:

| Install run in | npm reads the root `.npmrc` | Bun reads the root `bunfig.toml` |
| -------------- | --------------------------- | -------------------------------- |
| the repository root | yes | yes |
| a package inside a declared workspace | **no** | yes |
| a subdirectory with its own manifest, no root manifest | **no** | **no** |

npm resolves project configuration from the nearest ancestor holding a
`package.json` and stops there; Bun walks up to the workspace root. So a root
config file is read by nothing at all when someone runs an install inside a
subdirectory that carries its own manifest — and checking only the root would
report green for a monorepo that enforces the window nowhere.

Two things keep that from becoming noise. A manifest that declares no
dependencies and no `workspaces` installs nothing, so it is skipped — which is
most test fixtures. Anything left that is genuinely not an install location goes
in `quarantine-exclude` on the caller, where someone can be asked why.

A repository with no `package.json` anywhere installs nothing from a package
registry, so the gate reports that and passes.

No attempt is made to detect which package manager a directory uses. Detection
would have to read a lockfile that may not be committed or a `packageManager`
field that is usually absent, and every case it got wrong would be an install
location silently exempted. A config file for a manager that is never used
costs nothing.

---

## Inputs

| Input                  | Default                                      | Effect                                                           |
| ---------------------- | -------------------------------------------- | ---------------------------------------------------------------- |
| `full-history`         | `false`                                      | Scan all history instead of the pull request's range              |
| `code-scan-rules`      | `p/default`                                  | Semgrep ruleset                                                   |
| `allowed-licences`     | `MIT;ISC;Apache-2.0;BSD-3-Clause;BlueOak-1.0.0;Python-2.0` | Semicolon-separated SPDX identifiers permitted in the tree |
| `licence-clarifications` | none                                       | Path to a repository-local clarifications file                    |
| `min-release-age-days` | `7`                                          | The package quarantine window the `quarantine` gate enforces      |
| `quarantine-exclude`   | none                                         | Semicolon-separated globs of `package.json` paths the gate ignores |

`min-release-age-days` is the single place the window is stated; the seconds
value Bun wants is derived from it, so the two settings cannot come to disagree.
The window itself is a management decision recorded in the secure development
standard — change it there first, and here to match.

---

## Required status checks

A job's name becomes the status check, reported as `<caller job> / <job>`. With
the caller above, the four checks are:

```
supply-chain / secrets
supply-chain / licences
supply-chain / code
supply-chain / quarantine
```

Renaming a job here silently turns a required check on a caller into one that
never reports — and a check that never reports is indistinguishable from one
that has not finished, so the pull request simply waits forever. Job names and
the branch ruleset change together.

---

## Pinning

Callers may pin to a commit SHA instead of `@main`. That guarantees a caller's
gates never change without an explicit bump, and costs an update in every
repository whenever a gate is improved or a tool is upgraded.

The `@main` form is what the starter repositories ship, on the reasoning that a
security gate improving without being asked is the behaviour we want, and that
`main` here is protected and code-owner reviewed. Pin a repository that needs a
reproducible pipeline more than it needs the newest gate.

Tool *versions* are pinned regardless of which form the caller uses: scanner
binaries by version and SHA-256, container images by digest, third-party actions
by commit SHA. A mutable action reference is exactly what the quarantine gate
exists to prevent, so this repository does not use one.

A tool version is adopted only once its release has cleared the same quarantine
window every other dependency has to clear.

---

## Relation to the developer tool

`stc.sh`, the SpringTree compliance tool, checks a **developer's machine**: that
the required tools are installed and that the global npm and Bun configuration
carries the quarantine window. It lives in the ISMS repository, where the rules
it enforces are decided.

These workflows check a **repository**: the files it has committed. The two do
not share an implementation and are not meant to — different subjects, and
sharing one would mean handing every caller a way to reach a script in a private
repository, which needs a cross-repository token the per-repository
`GITHUB_TOKEN` is not.

A developer whose machine is set up correctly can still push a repository that
fails `quarantine`, and vice versa. Both are needed.

---

## Changing a gate

1. Open a pull request. `main` is protected; the self-check workflow runs these
   gates against this repository on every pull request.
2. A gate that becomes stricter breaks callers that were passing. Say so in the
   pull request, and check whether the change wants a grace period as a warning
   before it becomes a failure.
3. If a job is renamed, update the branch rulesets that require it in the same
   change.
