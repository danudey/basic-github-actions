# cmake (C and C++)

```sh
cp universal/workflows/*.yml .github/workflows/
cp cmake/workflows/*.yml     .github/workflows/
cp cmake/dependabot.yml      .github/dependabot.yml
cp universal/release-please-config.json universal/.release-please-manifest.json .
```

Then in `release-please-config.json` set `release-type: simple` and point
`extra-files` at wherever the version lives, and enable
`language: c-cpp, build-mode: autobuild` in `codeql.yml`.

## The values to change

| workflow | variable | what it is |
|---|---|---|
| all | `SOURCE_DIR` | the directory holding `CMakeLists.txt` |
| `ci.yml`, `codequality.yml`, `release.yml` | `CMAKE_ARGS` | the project's own options — **keep consistent across all three** |
| `ci.yml` | `CTEST_ARGS` | `--output-on-failure` is essential |
| `codequality.yml`, `fmt.yml` | `LINT_PATHS`, `CHECK_PATH` | **your own source only** |
| `fmt.yml` | `CLANG_FORMAT_VERSION` | pin it; output changes between majors |
| `release.yml` | `BINARY_NAME` | the `add_executable` target name |

`LINT_PATHS` and `CHECK_PATH` matter more than they look. Pointing either at the
repository root when third-party code is vendored produces thousands of findings
you cannot fix, and a check nobody can pass is a check people disable.

### Version stamping

For `release-type: simple`, mark the version line so release-please can rewrite
it:

```cmake
project(tool VERSION 1.2.3 LANGUAGES CXX)  # x-release-please-version
```

and list the file in `release-please-config.json`:

```json
"extra-files": ["CMakeLists.txt"]
```

## Why the sanitizer job is the most valuable thing here

A C++ test suite that passes proves the tests did not crash. It does not prove
there was no use-after-free, no read past the end of a buffer, no signed
overflow, no data race — all of which are undefined behaviour that happened to
work today, on this compiler, at this optimisation level.

`ci.yml`'s `sanitizers` job turns those into test failures. Two rows, because
they cannot be combined:

| row | finds |
|---|---|
| `address,undefined` | use-after-free, buffer overflow, leaks, signed overflow, misaligned loads, invalid enum values |
| `thread` | data races. Only worth a row if the project uses threads |

`-O1` rather than `-O0`, because the sanitizers need some optimisation for
usable stack traces, and `-fno-omit-frame-pointer` is what makes those traces
readable. The `ASAN_OPTIONS`/`UBSAN_OPTIONS` in the test step set
`halt_on_error`: without it the run reports the first problem and carries on
with a corrupted heap, producing a cascade of nonsense after the one finding
that mattered.

If the sanitizer job is failing on a pre-existing bug and you need to ship,
`continue-on-error: true` on the job is a better temporary answer than deleting
it — the finding stays visible.

## Why the compiler matrix is wide

In C and C++ the compiler is part of the platform. Code that builds under GCC
fails under Clang and again under MSVC, and each finds bugs the others miss.
Single-compiler CI is the usual reason a project "does not build on Windows".

The default matrix is six rows: GCC Release and Debug, Clang Release, GCC on
arm64, Apple Clang, and MSVC. The **arm64 row** is the one people leave out and
should not: different alignment rules, a signed `char` on some ABIs, and no x86
intrinsics.

## What the workflows need from the repository

### `.clang-tidy`

`codequality.yml`'s clang-tidy job does almost nothing without it. A starting
point that is strict but not unreasonable:

```yaml
# .clang-tidy
Checks: >
  clang-analyzer-*,
  bugprone-*,
  cert-*,
  concurrency-*,
  cppcoreguidelines-*,
  misc-*,
  modernize-*,
  performance-*,
  portability-*,
  readability-*,
  -modernize-use-trailing-return-type,
  -readability-identifier-length,
  -readability-magic-numbers,
  -cppcoreguidelines-avoid-magic-numbers,
  -misc-non-private-member-variables-in-classes,
  -cppcoreguidelines-pro-bounds-array-to-pointer-decay

WarningsAsErrors: >
  bugprone-use-after-move,
  bugprone-dangling-handle,
  bugprone-string-constructor,
  cppcoreguidelines-slicing,
  clang-analyzer-core.*

# Only our own headers. Without this, every finding is about a system or
# vendored header you cannot change.
HeaderFilterRegex: '^(src|include)/.*\.(h|hpp)$'

CheckOptions:
  readability-identifier-naming.ClassCase: CamelCase
  readability-identifier-naming.FunctionCase: lower_case
  readability-identifier-naming.VariableCase: lower_case
  readability-identifier-naming.PrivateMemberSuffix: _
  performance-unnecessary-value-param.AllowedTypes: 'std::shared_ptr;std::string_view'
```

`HeaderFilterRegex` is the one to get right first. Without it clang-tidy reports
on everything it parses, including the standard library.

### `.clang-format`

Generate a starting point from a style you like, then commit it:

```sh
clang-format --style=llvm --dump-config > .clang-format
```

Pin `CLANG_FORMAT_VERSION` in `fmt.yml` to whatever that was generated with.
clang-format's output changes between major versions, so a floating version
means CI and the developer's editor disagree about the same file, and the check
becomes noise.

### `install(TARGETS ...)` in `CMakeLists.txt`

`release.yml` packages whatever `cmake --install` puts in the staging directory.
A project with no install rules produces an empty archive — the workflow checks
for that and fails with a message saying so, but it is better to have the rules:

```cmake
include(GNUInstallDirs)

install(TARGETS tool
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})

install(DIRECTORY include/ DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
```

### `enable_testing()` and some `add_test()`

Otherwise ctest has nothing to run and the test step passes vacuously.

## Why the release builds in an old container

Unlike Go, Rust or Zig, C and C++ get no free cross-compilation: macOS needs a
macOS runner, MSVC needs Windows, and another Linux architecture needs a native
runner or a hand-installed cross toolchain. So `release.yml`'s matrix is one row
per runner, and the set of platforms you can ship is the set of runners you are
willing to use.

The Linux rows build inside `quay.io/pypa/manylinux_2_28_*` — a RHEL 8 image
with a modern GCC backported. Linking against its glibc 2.28 means the binary
runs on anything from about 2019 onwards. **A binary linked against the runner's
own glibc refuses to start on anything older**, and that is the most common
complaint about a hand-rolled C++ release.

The macOS rows set `CMAKE_OSX_DEPLOYMENT_TARGET=11.0` for the same reason.

## Dependency auditing, and what is genuinely missing

There is no `cmake` Dependabot ecosystem, no `vcpkg` one, and no `conan` one.
What is available:

| | supported | how |
|---|---|---|
| git submodules | yes | `gitsubmodule` in `dependabot.yml` |
| Dockerfile base | yes | `docker` |
| Actions | yes | `github-actions` |
| vcpkg | no | pin `builtin-baseline`, raise with `vcpkg x-update-baseline` |
| Conan | no | `conan graph outdated` |
| `FetchContent` | no | pin `GIT_TAG` to a SHA, raise by hand — or use Renovate, which handles this with a regex manager |

`audit.yml` covers the gap three ways:

- **osv-scanner** reads `vcpkg.json` and `conan.lock` where they exist.
- **Trivy `fs`** fingerprints bundled libraries from their own headers —
  `zlib.h`'s `ZLIB_VERSION`, `sqlite3.h`'s `SQLITE_VERSION` — so it finds a
  vendored zlib in a repository with no manifest at all. This is the one that
  works on a typical C++ project.
- **the `submodules` job** reports how many commits behind each submodule is,
  into the job summary. Not a vulnerability check, but a submodule pinned to a
  2019 commit is where the vulnerabilities are, and nothing else will tell you.

Note what `gitsubmodule` actually does: it bumps to the tip of the upstream
**default branch**, not to the latest tag. For a well-run upstream that is fine;
for one that commits directly to `main`, it is a stream of pull requests to
review rather than merge unread. That is why the interval is monthly and the
limit is 3.

## Things that will bite you

**The configure step fails on `cmake_minimum_required`.** The CMake preinstalled
on the GitHub images lags. `lukka/get-cmake` is in every job for this reason;
keep it.

**clang-tidy reports on system headers.** `HeaderFilterRegex` is missing or too
broad. See above.

**clang-tidy cannot find a generated header.** The `Generate headers` step
builds the project before linting, because clang-tidy reads the source as
written and a `#include "generated/config.h"` that does not exist yet produces a
finding on every file. If your generator has a specific target, narrow
`--target all` to it.

**cppcheck takes a minute to install.** It is built from source, because the
Ubuntu package lags and SARIF output is recent. If that minute matters more than
the SARIF, use `apt-get install cppcheck` and `--template=gcc` for annotations
instead.

**ccache never hits.** The key includes the compiler and build type. It hits on
repeat pushes to the same branch and on `main`; a fresh pull request branch
starts cold, because GitHub's Actions cache is scoped per branch.

**The release archive is empty.** No `install(TARGETS)` rules. See above.

**MSVC fails on something GCC accepted.** Usually a missing `#include` that GCC
pulled in transitively, or a template that MSVC's stricter two-phase lookup
rejects. That is the row doing its job.
