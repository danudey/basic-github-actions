# basic-github-actions

A library of GitHub Actions workflows worth having in almost any repository,
one directory per language, plus a `universal/` set that is language-agnostic.

They are meant to be copied and edited, not referenced. Each file starts with a
comment block saying what to change, what it needs from you, and what it
produces. Read that block; it is the documentation.

## Install

```sh
# The language-agnostic set, in every repository.
cp universal/workflows/*.yml   .github/workflows/
cp universal/dependabot.yml    .github/dependabot.yml
cp universal/release-please-config.json .
cp universal/.release-please-manifest.json .

# Then one language directory on top.
cp go/workflows/*.yml  .github/workflows/
cp go/dependabot.yml   .github/dependabot.yml   # replaces the universal one
```

Then open each file and work through its `WHAT TO CHANGE` block. Most of them
need one or two values at the top; `release.yml` needs three or four.

A repository has exactly **one** `.github/dependabot.yml`. Each language
directory's copy already contains the `github-actions` entry, so use that one
and delete `universal/dependabot.yml`. For a polyglot repository, append the
second `updates:` entry from each language you use into the one file.

## What is here

| | code quality | build & test | release | audit | format |
|---|---|---|---|---|---|
| `go/` | golangci-lint → SARIF | 4 runners, 12 cross-targets | 12 platforms, attested, signed | govulncheck + osv-scanner | gofmt, gofumpt, tidy, generate |
| `python/` | ruff → SARIF | 4 OS × 3 pythons | PyPI trusted publishing + cibuildwheel | pip-audit + osv-scanner | ruff format, lock check |
| `rust/` | clippy → SARIF | 4 runners, 6 cross-targets, MSRV | 11 targets + universal macOS, crates.io | cargo audit + deny + osv | rustfmt, taplo, rustdoc |
| `node/` | eslint → SARIF | 4 OS × 3 nodes | npm provenance, attested | npm/pnpm/yarn audit + osv | prettier or biome |
| `java/` | SpotBugs → SARIF, PMD | 4 OS × 3 JDKs | Maven Central, attested | trivy + osv-scanner | spotless, sortpom |
| `cmake/` | clang-tidy + cppcheck → SARIF | 6 compilers, ASan/UBSan/TSan | 5 platforms, glibc-compatible | trivy fs + osv + submodules | clang-format, cmake-format |
| `zig/` | compiler diagnostics, zlint | 4 runners × 4 opt modes, 10 targets | 10 targets from one runner | dependency hygiene | zig fmt |
| `container/` | hadolint + trivy config → SARIF | native multi-arch build | multi-arch GHCR, cosign + SBOM | trivy image, daily | (hadolint) |
| `universal/` | CodeQL, scorecard, zizmor | — | release-please | dependency-review | actionlint |

Each language directory also has a `README.md` with the tool configuration the
workflows expect — `.golangci.yml`, `[tool.ruff]`, `.clang-tidy` and so on.
Those configurations are where most of the value is; the workflows only run
them.

## Five things that apply everywhere

### 1. Every action is pinned to a commit SHA

```yaml
- uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
```

A tag is a mutable pointer. `@v7` can be repointed at any commit by anyone with
write access to that repository, and a compromised action runs with your
workflow's token — which is how the `tj-actions/changed-files` supply-chain
attack worked in March 2025. A commit SHA cannot be repointed.

The cost is that a SHA never updates itself, and nobody notices a stale pin.
That is what the `github-actions` Dependabot entry is for: it rewrites the SHA
and the version comment together, so the pin stays current as well as
immutable. **Pinning without Dependabot is worse than not pinning.** Copy the
`dependabot.yml`.

To re-resolve every pin in this library, or to check one:

```sh
gh api repos/actions/checkout/commits/v7.0.1 --jq .sha
```

### 2. SARIF, not log output

Every code-quality and audit workflow uploads its findings to the repository's
**Security → Code scanning** tab rather than only printing them. That turns a
finding into something with a location, a rule, a history, and a dismissal
reason: it appears when introduced, closes when fixed, and can be silenced
individually with an explanation. A linter whose output is a wall of text in a
log is a linter people learn to skip.

This needs a **public repository, or GitHub Advanced Security on a private
one**. Without either, the upload step fails with a 403. Each affected workflow
says so in its header and tells you what to do instead.

### 3. Signing and provenance need no keys

The release workflows produce, for every artifact:

- a **SLSA provenance attestation** in GitHub's attestation store —
  `gh attestation verify ./artifact --repo OWNER/REPO`
- a **`.sigstore` bundle** beside the artifact — checkable with `cosign` by
  anyone, with no GitHub account
- an **SBOM**, attested separately
- registry-native provenance where the registry supports it: PyPI's PEP 740
  attestations, npm's `--provenance`, BuildKit's `mode=max` for containers

All of it is keyless. The signing identity is the workflow's own OIDC token, so
what a verifier checks is *"this artifact was built by this workflow, in this
repository, from this commit"* — not *"somebody who held a key said so"*. There
is no private key to leak, rotate, or hand over when you stop maintaining the
project.

The one exception is `java/workflows/release.yml`: Maven Central requires a GPG
signature and a portal token, and has no OIDC equivalent. Its header says
exactly which four secrets that needs and why.

### 4. Publishing usually needs no token either

PyPI, npm, crates.io and GHCR all accept a workflow OIDC token in place of a
stored credential. Each release workflow's header has the one-time setup: you
tell the registry, once, that this repository's `release.yml` may publish this
package. A stolen token is then useless, because there is no token.

Each publish job also runs in a named GitHub **environment** (`pypi`, `npm`,
`crates-io`, `maven-central`). That is not decoration — the registry's trusted
publisher configuration names the environment, so a run outside it cannot
publish. Adding a required reviewer to that environment turns every release
into a manual approval.

### 5. Conventional commits drive the releases

Every `release.yml` is gated on `release-please`, which keeps a release pull
request open against `main` holding the next version and the changelog earned by
the commits since the last release. Merging that pull request is what cuts the
release.

It reads **commit subjects on `main`**, so it only works if pull requests are
squash-merged with the pull request title as the subject:

> Settings → General → Pull Requests → Default commit message → **Pull request
> title**

`universal/workflows/pr-title.yml` checks those titles on the way in, so a
non-conventional title fails a check rather than silently costing you a
release.

## Cross-cutting things worth knowing

**arm64 runners are free on public repositories.** `ubuntu-24.04-arm` and
`windows-11-arm` are native, and roughly ten times faster than emulating arm64
under QEMU. Every matrix here uses them rather than emulation.

**The scheduled runs are the point, not an afterthought.** An advisory is
published without your repository changing; a linter grows rules without your
repository changing; a base image accumulates CVEs while nobody commits. Every
`audit.yml` and `codequality.yml` runs weekly (daily for containers) on top of
its pull-request trigger. GitHub emails the repository owner when a scheduled
workflow fails on the default branch, which is the notification that matters.

**Every cron is offset from the top of the hour.** GitHub asks for this to
spread scheduler load, and a job scheduled at `0 * * * *` is genuinely more
likely to be delayed.

**Permissions are set per job, not per workflow.** The workflow-level default is
`contents: read`, and each job adds only what it needs. A `codequality` job gets
`security-events: write`; a release job gets `id-token: write`. This is what
`zizmor` checks for, and getting it wrong is the most common way a workflow ends
up more privileged than it looks.

**`concurrency` cancels superseded runs.** Pushing three times to a branch
should not run CI three times. Release workflows deliberately do *not* cancel
in progress — an interrupted publish is worse than a duplicated one.

## Verifying the action pins

```sh
# Every pin in the library, checked against the GitHub API.
grep -rhoE 'uses: [^@]+@[0-9a-f]{40} # \S+' --include='*.yml' . \
  | sed 's/^uses: //' | sort -u \
  | while read -r ref _ tag; do
      repo=$(echo "${ref%@*}" | awk -F/ '{print $1"/"$2}')
      actual=$(gh api "repos/$repo/commits/$tag" --jq .sha </dev/null 2>/dev/null)
      [ "$actual" = "${ref##*@}" ] && echo "ok       $repo@$tag" \
                                   || echo "MISMATCH $repo@$tag"
    done
```

## Linting the workflows themselves

`universal/workflows/workflow-lint.yml` runs `actionlint` (correctness,
including shellcheck over every `run:` block) and `zizmor` (security: expression
injection, over-broad permissions, unpinned actions). Everything in this library
passes both at their strictest settings. To check locally:

```sh
actionlint .github/workflows/*.yml
uvx zizmor --collect all .
```

## Maintaining this repository

This repository's own CI lives in `.github/` and does two jobs.

**`validate.yml`** runs on every pull request and every push to `main`. It
checks every example workflow, every example `dependabot.yml`, the
release-please files, and `.github/` itself:

| check | what fails it |
|---|---|
| actionlint | invalid workflow syntax or expressions; shellcheck findings in `run:` blocks |
| zizmor | security findings (injection, permissions, credential persistence, …) |
| pinact | a `uses:` not pinned to a full SHA, or a `# vX.Y.Z` comment that does not match its SHA (README snippets included) |
| check-jsonschema | a workflow, `dependabot.yml`, or release-please file that does not match its schema |
| yamllint | YAML syntax or style, per `.yamllint.yml` |

**`update-example-pins.yml`** runs every Monday and opens one pull request that
moves every pin in `<language>/workflows/` and the README snippets to the
latest release at least 7 days old. Dependabot cannot do this: it only reads
`.github/workflows`. Dependabot handles `.github/` and the tool versions in
`.github/tools/` instead.

One-time setup for the updater:

1. Settings → Actions → General → turn on **Allow GitHub Actions to create and
   approve pull requests**.
2. Optional: create a GitHub App with Contents and Pull requests read/write,
   install it on this repository, and set the `PIN_UPDATER_APP_ID` variable
   and `PIN_UPDATER_APP_KEY` secret. Without it, the pull request is opened
   with `GITHUB_TOKEN`, and `validate` does not run on it until you push to
   the branch or close and reopen the pull request.

To run the checks locally (needs Go and uv; the tool versions come from
`.github/tools/`):

```sh
go build -C .github/tools -o "$PWD/.bin/" \
  github.com/rhysd/actionlint/cmd/actionlint \
  github.com/suzuki-shunsuke/pinact/v5/cmd/pinact
export GITHUB_TOKEN=$(gh auth token) GH_TOKEN=$(gh auth token)

git ls-files -z '*/workflows/*.yml' | xargs -0 .bin/actionlint
git ls-files -z '*/workflows/*.yml' 'dependabot.yml' '*/dependabot.yml' \
  | xargs -0 uv run --project .github/tools --frozen zizmor
git ls-files -z '*/workflows/*.yml' '*.md' \
  | xargs -0 .bin/pinact run --check --verify-comment
uv run --project .github/tools --frozen yamllint --strict .
```
