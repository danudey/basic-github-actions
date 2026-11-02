# universal

Workflows that do not care what language the repository is written in. Copy all
of these into every repository, then add one language directory on top.

```sh
cp universal/workflows/*.yml            .github/workflows/
cp universal/release-please-config.json .
cp universal/.release-please-manifest.json .
# Use the language directory's dependabot.yml instead of this one; it already
# contains the github-actions entry.
cp universal/dependabot.yml             .github/dependabot.yml
```

## What each file does

| file | what it answers | needs |
|---|---|---|
| `codeql.yml` | are there security defects in the code | public repo or GHAS |
| `scorecard.yml` | is the repository's own security posture sound | public repo |
| `workflow-lint.yml` | are the workflows correct and safe | public repo or GHAS for the SARIF half |
| `dependency-review.yml` | does this pull request add a vulnerable or badly-licensed dependency | public repo or GHAS |
| `pr-title.yml` | is the pull request title a conventional commit | nothing |
| `release-please.yml` | keep a release pull request open | nothing |

`release-please.yml` is standalone: use it for a repository that has no build
artifacts to publish. Every language directory's `release.yml` already contains
the same job as its first gate, so **do not use both** — two release-please runs
on the same repository will fight over the release pull request.

## First-time repository setup

Three settings, none of which a workflow can set for you.

**1. Squash merge with the pull request title as the subject.** release-please
reads commit subjects on `main` and nothing else.

> Settings → General → Pull Requests
> - Allow squash merging ✓
> - Default commit message → **Pull request title**
> - Allow merge commits ✗ (a merge commit's subject is `Merge pull request #12
>   from…`, which is not a conventional commit and produces no release)

**2. Environments for the publish jobs.** Only needed once you use a language
`release.yml`. Settings → Environments, then create whichever applies: `pypi`,
`testpypi`, `npm`, `crates-io`, `maven-central`. Add a required reviewer to each
to make every publish a manual approval.

**3. Branch protection.** Require the checks you care about. The useful set,
once the workflows have run at least once so the names exist:

```
fmt
codequality / <linter>
test (ubuntu-latest, …)
CodeQL (<language>)
dependency-review
pr-title / conventional-commit
```

Do not require `scorecard` or `audit` — both can go red because of something
published upstream overnight, and a branch protection rule that blocks every
merge until an unrelated advisory is fixed is a rule people will route around.

## Notes on the individual workflows

### codeql.yml

The matrix is commented out except for `actions`. Uncomment the languages the
repository actually has; a language with no matching source files fails the run.

`build-mode` matters:

- `none` — `actions`, `javascript-typescript`, `python`, `ruby`
- `autobuild` — `go`, `java-kotlin`, `csharp`, `swift`, usually `c-cpp`
- `manual` — a compiled language whose build needs specific commands. There is
  a worked example at the bottom of the file.

CodeQL does not support Zig. Keep the workflow anyway for the `actions`
language, which scans the workflow files themselves.

The `queries: security-and-quality` setting is the noisiest of the three suites.
If the alert list becomes something people stop reading, drop to
`security-extended`, or remove the line for the default suite.

### scorecard.yml

Grades the repository rather than the code: are actions pinned, are branches
protected, does the release pipeline sign anything, is a maintainer still
responding. It is the only check here that will tell you your CI configuration
is the weak point.

Set `publish_results: false` on a private repository — the run fails otherwise.
The Branch-Protection check needs a PAT with read access to `administration` even
on a public repository; without one, that check reports inconclusive rather than
failing, which is usually fine.

With `publish_results: true` you get a badge:

```markdown
[![OpenSSF Scorecard](https://api.securityscorecards.dev/projects/github.com/OWNER/REPO/badge)](https://scorecard.dev/viewer/?uri=github.com/OWNER/REPO)
```

### workflow-lint.yml

Two tools that do not overlap:

- **actionlint** — correctness. A typo in `runs-on`, an expression referencing
  something that does not exist, an invalid `if`, a matrix key no job reads, and
  shellcheck over every `run:` block. Anything it reports is a real defect,
  which is why `fail_level: any`.
- **zizmor** — security. Script injection through `${{ github.event.* }}`,
  unpinned actions, over-broad `permissions`, `pull_request_target` with a
  checkout of the fork's head, cache poisoning in a release workflow.

Silence a deliberate zizmor finding at the line rather than globally:

```yaml
- run: echo "${{ github.event.pull_request.title }}"  # zizmor: ignore[template-injection]
```

Everything in this library passes both at these settings.

### dependency-review.yml

Only ever looks at what a pull request *adds*, which makes it different from the
language `audit.yml` workflows — those ask whether what you already depend on is
vulnerable today, and run on a schedule.

The `deny-licenses` list here is the copyleft family, which most projects cannot
absorb into a permissively-licensed binary. Pick either `deny-licenses` or
`allow-licenses`, never both. An allow-list is stricter and will block the first
transitive dependency with an unusual SPDX identifier, which is often a false
alarm about a dual-licensed package.

Needs a committed lock file to be worth much. Without one the action can only
see the ranges in the manifest, not what actually resolves.

### pr-title.yml

The `task_types` list must stay in step with `changelog-sections` in
`release-please-config.json`. A type that passes this check but is not in the
config produces a commit release-please silently ignores, which reads as "my
feature did not make the changelog" and is very hard to debug after the fact.

`add_label: 'true'` writes a type label onto the pull request. A pull request
from a fork gets a read-only token, so the labelling fails there. If the
repository takes fork pull requests, set it to `'false'` rather than switching
to `pull_request_target` — that trigger runs with a write token against
untrusted code, and is the single most common way an Actions setup gets
compromised.

### release-please-config.json

`release-type` is the only field you must change. It decides which files get the
version written into them:

| language | `release-type` |
|---|---|
| Go | `simple` (release-please will not edit `go.mod`) |
| Python | `python` |
| Rust | `rust` |
| Node | `node` |
| Java | `maven` |
| C/C++, Zig | `simple`, plus `extra-files` |

For `simple`, point `extra-files` at whatever holds the version and mark the
line:

```cmake
project(tool VERSION 1.2.3) # x-release-please-version
```

`bump-minor-pre-major: true` is set: below 1.0.0 a breaking change bumps 0.3.x
to 0.4.0 rather than jumping to 1.0.0. Remove it when you reach 1.0.0.

Go v2 and later: Go requires the module path to gain a `/v2` suffix.
release-please tags `v2.0.0` but does not edit `go.mod` or the import paths, so
do that by hand in the same pull request.

JSON has no comments, and the config schema rejects unknown top-level keys, so
keep explanations here rather than as `"//"` keys at the top of the file.

Set the starting version in `.release-please-manifest.json`. For an existing
repository, put the version you have already released there — release-please
uses it as the floor, and getting it wrong means the first release pull request
proposes a version that already exists.
