# python

```sh
cp universal/workflows/*.yml .github/workflows/
cp python/workflows/ci.yml python/workflows/codequality.yml \
   python/workflows/fmt.yml python/workflows/audit.yml \
   python/workflows/release.yml .github/workflows/
cp python/dependabot.yml     .github/dependabot.yml
cp universal/release-please-config.json universal/.release-please-manifest.json .
```

Then set `release-type: python` in `release-please-config.json`, and enable
`language: python, build-mode: none` in `codeql.yml`.

**Pick one release workflow, not both.** `release.yml` for pure Python;
`release-binary-wheels.yml` for a compiled extension module. Two workflows
publishing the same version to PyPI will race and the loser fails.

## uv, Poetry or pip

Every workflow here has one knob:

```yaml
env:
  PACKAGE_MANAGER: "uv"   # uv | poetry | pip
```

The pattern is a `case` over the manager that writes a `$RUN` prefix into
`$GITHUB_ENV`; later steps use `$RUN pytest` rather than hard-coding anything.
So switching managers is one line per workflow, not a rewrite.

| | needs | install command used | run prefix |
|---|---|---|---|
| `uv` | `uv.lock` + `[project]` in `pyproject.toml` | `uv sync --frozen --all-extras --dev` | `uv run --frozen` |
| `poetry` | `poetry.lock` + `[tool.poetry]` | `poetry install --with dev --all-extras` | `poetry run` |
| `pip` | `pyproject.toml` with a `dev` extra | `pip install -e .[dev]` | *(none)* |

**uv is the recommended one** and the default: it manages its own interpreters
(so `actions/setup-python` is not needed), installs an order of magnitude
faster, and `--frozen` gives a genuinely reproducible install. Poetry is a
close second. Plain pip works, but note what you give up: with no lock file the
CI run tests whatever resolved this morning, which is a much weaker check than
it looks. If you are on pip and want to keep it, adopt `pip-tools` or
`uv lock` for the lock file and leave `pip` as the installer.

`release.yml` and the build job in `ci.yml` always use `uv build` regardless of
`PACKAGE_MANAGER`, because it drives whichever build backend
`pyproject.toml` declares — Hatch, Poetry, setuptools, maturin. That is why the
release path does not need the knob.

## The values to change

| workflow | variable | what it is |
|---|---|---|
| all | `PACKAGE_MANAGER` | `uv`, `poetry` or `pip` |
| all | `WORKING_DIRECTORY` | the directory holding `pyproject.toml` |
| `ci.yml` | `TEST_COMMAND`, `TYPECHECK_COMMAND` | if not pytest and mypy |
| `ci.yml` | `PIP_INSTALL` | pip only; an extra or `-r requirements-dev.txt` |
| `codequality.yml`, `fmt.yml` | `RUFF_VERSION` | **keep these two identical** |
| `audit.yml` | `IGNORE_VULNS` | advisory IDs, space-separated, each documented |
| `release-binary-wheels.yml` | `CIBW_BUILD`, `CIBW_SKIP` | which wheels to build |

### The Python version matrix

`ci.yml`'s `matrix.python` should match `requires-python` in `pyproject.toml`.
Nothing checks that agreement, and a version claimed there but not tested here
is the usual source of "it does not work on 3.9".

The matrix runs all versions on Linux and only the newest elsewhere, on the
grounds that what the other platforms test is the filesystem rather than the
interpreter. Delete the `exclude:` block to test everything everywhere, at
roughly four times the runner minutes.

### Architectures

For **pure Python** the OS matrix in `ci.yml` is the whole story — there is one
`py3-none-any` wheel and it runs everywhere.

For a **compiled extension module**, the architecture matrix is the wheel
matrix, and it lives in `release-binary-wheels.yml` under cibuildwheel. The
runner determines which architectures are reachable:

| runner | wheels |
|---|---|
| `ubuntu-latest` | `manylinux`/`musllinux` x86_64, i686 |
| `ubuntu-24.04-arm` | `manylinux`/`musllinux` aarch64 — native, much faster than QEMU |
| `macos-15-intel` | macOS x86_64 (Intel runner) |
| `macos-latest` | macOS arm64, and `universal2` if asked |
| `windows-latest` | `win_amd64`, `win32` |
| `windows-11-arm` | `win_arm64` |

`cibuildwheel --print-build-identifiers` lists every identifier available on a
given runner.

Do not try to hand-roll this. A wheel that installs and then fails to import on
someone else's distribution is the normal outcome of building `manylinux` wheels
without cibuildwheel: the tag has to be honest about the glibc it needs, and the
shared libraries have to be bundled and the tag repaired.

## What the workflows need from the repository

### `[tool.ruff]` in `pyproject.toml`

`codequality.yml` runs ruff but without configuration you get its small default
rule set. This is where the value is:

```toml
[tool.ruff]
target-version = "py311"          # the oldest version you support
line-length = 100
extend-exclude = ["migrations", "_vendor"]

[tool.ruff.lint]
select = [
  "E", "W",      # pycodestyle
  "F",           # pyflakes: undefined names, unused imports
  "I",           # isort: import order (fmt.yml checks this one separately)
  "N",           # pep8-naming
  "UP",          # pyupgrade: syntax newer than target-version allows
  "B",           # bugbear: mutable default args, assert in production, ...
  "A",           # shadowing a builtin
  "C4",          # comprehension simplifications
  "DTZ",         # a naive datetime where an aware one was meant
  "T20",         # a stray print()
  "PT",          # pytest style
  "RET",         # unnecessary else after return
  "SIM",         # simplifiable code
  "PTH",         # os.path where pathlib would do
  "ERA",         # commented-out code
  "PL",          # pylint's useful subset
  "TRY",         # except: pass, raising a bare Exception
  "RUF",         # ruff's own rules
  "S",           # flake8-bandit: THE SECURITY RULES. See note below.
]
ignore = [
  "E501",        # line length; the formatter owns this
  "PLR0913",     # too many arguments — often a legitimate API
  "TRY003",      # long messages inside raise; fine in practice
]

[tool.ruff.lint.per-file-ignores]
# Tests assert, use magic numbers, and hard-code fixtures. All fine there.
"tests/**" = ["S101", "PLR2004", "S105", "S106"]
"__init__.py" = ["F401"]          # re-exports

[tool.ruff.lint.pydocstyle]
convention = "google"
```

**On `"S"` and Bandit.** The `S` rules *are* Bandit, reimplemented in ruff. With
`S` selected you do not need a separate Bandit job — running both duplicates
every finding in the Code scanning tab. `codequality.yml` has a commented Bandit
job for the case where you need Bandit's own configuration format; if you enable
it, remove `"S"` from `select`.

### `[tool.mypy]`

`ci.yml` runs `mypy .` as a test rather than a lint, because its answer depends
on the Python version being checked.

```toml
[tool.mypy]
python_version = "3.11"
strict = true
warn_unreachable = true
# Third-party packages without type hints, so `strict` does not fail on them.
[[tool.mypy.overrides]]
module = ["some_untyped_lib.*"]
ignore_missing_imports = true
```

Swap in `pyright` or `ty` by changing `TYPECHECK_COMMAND`.

### A lock file, committed

`uv.lock` or `poetry.lock`. `fmt.yml` checks it is current, which stops it
producing a diff on somebody else's unrelated pull request.

## PyPI publishing, with no API token

Both release workflows use **trusted publishing**: you tell PyPI once that this
repository's workflow may publish this project, and the workflow proves who it
is with an OIDC token. Nothing to store, nothing to rotate, and a stolen token
is useless because there is none.

Setup, once per project:

1. Create the GitHub environments `pypi` and `testpypi`
   (Settings → Environments). Add a required reviewer to `pypi` if you want a
   manual gate on every release.
2. On <https://pypi.org/manage/account/publishing/>, add a publisher:
   - Owner / repository: yours
   - Workflow filename: `release.yml` (or `release-binary-wheels.yml`)
   - Environment: `pypi`
3. Repeat at <https://test.pypi.org/manage/account/publishing/> with environment
   `testpypi`, or delete the `testpypi` job.

The publish action generates **PEP 740 attestations** from the same OIDC
identity, which `pip` verifies. That is separate from, and in addition to, the
GitHub SLSA attestation and cosign bundle the `github-release` job produces.

### Why the TestPyPI rehearsal is worth keeping

A version number on PyPI can be deleted but never reused. A metadata field PyPI
rejects, a name collision, or a misconfigured publisher all fail *after* the
version is burnt. The rehearsal catches all three for the cost of one extra job.

## Things that will bite you

**The built package has version `0.1.dev1`.** `setuptools-scm` and `hatch-vcs`
derive the version from the git history, and a shallow clone gives them nothing.
Both release workflows use `fetch-depth: 0` for exactly this; if you add another
job that builds, it needs the same.

**`uv sync --frozen` fails with "lock file out of date".** `pyproject.toml`
changed without `uv lock` being run. That is the check working: run `uv lock`
and commit it.

**ruff and the format check disagree about the same file.** `RUFF_VERSION` is
not identical in `codequality.yml` and `fmt.yml`. Make them match.

**pip-audit reports a vulnerability in a dev dependency you cannot fix.** Add it
to `IGNORE_VULNS` with a comment saying why. An undocumented ignore entry
outlives the problem it was added for.

**cibuildwheel builds forever.** `CIBW_SKIP` is doing a lot of work in the
default configuration. musllinux and 32-bit targets roughly double the build
time; remove a pattern from the skip list only when somebody asks for it.

**The wheel installs and then fails to import.** The `pack`-equivalent check is
in `ci.yml`'s `build` job: it lists the wheel's contents, because a wheel
containing nothing but a `.dist-info` directory installs perfectly cleanly.
