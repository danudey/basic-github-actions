# go

```sh
cp universal/workflows/*.yml .github/workflows/
cp go/workflows/*.yml        .github/workflows/
cp go/dependabot.yml         .github/dependabot.yml
cp universal/release-please-config.json universal/.release-please-manifest.json .
```

Then set `release-type: simple` in `release-please-config.json`, and enable
`language: go, build-mode: autobuild` in `codeql.yml`.

## The values to change

Every workflow has an `env:` block at the top. These are the ones that matter:

| workflow | variable | what it is |
|---|---|---|
| all | `WORKING_DIRECTORY` | the directory holding `go.mod`; `.` for a root module |
| all | `GO_VERSION` | `stable`, `oldstable`, or an exact `1.25.x` |
| `ci.yml`, `audit.yml` | `PACKAGES` | what to build and test; `./...` for everything |
| `codequality.yml` | `GOLANGCI_VERSION` | pinned, so a new release does not turn `main` red |
| `fmt.yml` | `GOFUMPT` | `"false"` to check plain `gofmt` only |
| `release.yml` | `BINARY_NAME` | the installed name, and the archive stem |
| `release.yml` | `MAIN_PACKAGE` | the package with `func main`, e.g. `./cmd/tool` |
| `release.yml` | `VERSION_VAR` | the string to stamp the version into, e.g. `main.version` |

### Code in a subdirectory

Set `WORKING_DIRECTORY` and nothing else. It feeds `defaults.run.working-directory`,
`cache-dependency-path` and the release paths.

```yaml
env:
  WORKING_DIRECTORY: "src/tool"
```

For **several modules** in one repository, run the workflow as a matrix over
them rather than setting one value:

```yaml
strategy:
  matrix:
    module: [., ./tools, ./examples]
defaults:
  run:
    working-directory: ${{ matrix.module }}
```

and add each one to `dependabot.yml` under `directories:`.

### Changing which architectures get built

Two separate lists, and they should agree:

- `ci.yml` → `cross-build.strategy.matrix.include` — proves it still compiles.
- `release.yml` → `build.strategy.matrix.include` — produces the artifacts.

`go tool dist list` prints every `GOOS/GOARCH` pair Go knows. The default set
covers Linux (amd64, arm64, armv7, riscv64, ppc64le, s390x), macOS (both),
Windows (both) and FreeBSD. Delete the rows you would not answer a bug report
for; each one is a job and an artifact.

`CGO_ENABLED=0` is set in both. That is what makes the cross-builds work at all
without a C toolchain per target, and what makes one Linux binary run on any
distribution. **If the project genuinely needs cgo**, the non-native targets
cannot be covered by this approach — reduce the matrix to the platforms you have
runners for, and set up `zig cc` or a cross toolchain per target if you need
more.

### Version stamping

`release.yml` builds with `-ldflags "-X ${VERSION_VAR}=${TAG}"`, so declare the
variable in your `main` package:

```go
package main

// Overwritten at build time by -ldflags. The default is what a `go install`
// from source reports.
var version = "dev"
```

Set `VERSION_VAR: ""` to skip the flag entirely.

## What the workflows need from the repository

### `.golangci.yml`

`codequality.yml` runs golangci-lint but deliberately does not choose linters
for you. Without a config file you get the small default set, and the workflow
is barely more than `go vet`. This is where the value is:

```yaml
version: "2"

linters:
  default: standard
  enable:
    # Correctness
    - errcheck        # an unchecked error return
    - errorlint       # %v where %w was meant, and vice versa
    - nilerr          # `return nil` after checking err != nil
    - bodyclose       # an HTTP response body never closed
    - rowserrcheck    # sql.Rows.Err() never checked
    - sqlclosecheck
    - contextcheck    # a function that should take a context and does not
    - noctx           # an HTTP request built without one
    - copyloopvar
    - exhaustive      # a switch over an enum missing a case
    - makezero
    # Security
    - gosec           # this is Bandit's Go equivalent; no separate job needed
    # Style, in decreasing order of how much anyone will thank you
    - revive
    - staticcheck
    - unconvert
    - unparam
    - misspell
    - godot
    - goconst
    - gocritic
  settings:
    errcheck:
      # Not checking the error from these is idiomatic rather than a bug.
      exclude-functions:
        - (io.Closer).Close
        - (*os.File).Close
    exhaustive:
      default-signifies-exhaustive: true
    revive:
      rules:
        - name: exported
          arguments: [checkPrivateReceivers, disableStutteringCheck]
    gosec:
      excludes:
        # Weak random is fine outside cryptography; flag it per-line instead.
        - G404

formatters:
  enable:
    - gofumpt
    - goimports

issues:
  # Report everything, not just what a pull request touched. A repository that
  # only ever lints the diff never gets clean.
  new: false
  # No cap. The defaults (50 issues, 3 per linter) silently hide the rest.
  max-issues-per-linter: 0
  max-same-issues: 0

exclusions:
  rules:
    # Test files legitimately do things production code should not.
    - path: _test\.go
      linters: [errcheck, gosec, goconst, dupl]
```

`gosec` is enabled here, which is why there is no separate gosec job: running
both duplicates every finding in the Code scanning tab.

### `go.sum`, committed

Every command uses `-mod=readonly` semantics, so a stale or missing `go.sum`
fails rather than resolving something new. That is deliberate — the alternative
is testing a dependency set nobody reviewed.

### `install(TARGETS)`-equivalent: nothing

Go needs no install rules. `release.yml` builds the binary directly.

## Things that will bite you

**`go mod tidy` produces a diff on someone else's pull request.** That is what
`fmt.yml`'s tidy check prevents: it fails on the pull request that caused the
drift rather than on the next unrelated one.

**A Dependabot bump fails only on the `oldstable` row.** A dependency raised its
own minimum Go version above yours. Either raise the `go` directive in `go.mod`
(a decision about who you support, so do it deliberately) or hold the dependency
back in `dependabot.yml`.

**govulncheck fails and no dependency bump fixes it.** It reports vulnerabilities
in the Go standard library and toolchain too. Raise `GO_VERSION`.

**govulncheck fails on a test-only dependency.** `PACKAGES: "./..."` includes
test and example code, so a vulnerability reachable only from a test fails the
build. Narrow it to `./cmd/... ./internal/...` if that is not what you want.

**The release produces a binary with no version in it.** Either `VERSION_VAR`
does not name a real package-level `var`, or the variable is a constant. It has
to be `var`, not `const`.

## Alternative: GoReleaser

`release.yml` here is hand-rolled, which makes every step visible and editable.
[GoReleaser](https://goreleaser.com) does the same job from a `.goreleaser.yaml`
and additionally handles Homebrew taps, Linux packages, Docker manifests and
announcements. It is the more capable tool for a project that ships to several
channels.

The trade-off is that its configuration is another thing to learn and its
provenance setup is less obvious than the four explicit steps here. If you
switch, keep the `release-please` gate and replace only the `build` and
`checksums` jobs:

```yaml
- uses: goreleaser/goreleaser-action@a4f02ab5355a589de60d917a659ce617ed19b3de # v7.3.0
  with:
    version: "~> v2"
    args: release --clean
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

and set `sboms:`, `signs:` and `--attestations` in `.goreleaser.yaml` so you do
not lose the provenance.
