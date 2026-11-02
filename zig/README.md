# zig

```sh
cp universal/workflows/*.yml .github/workflows/
cp zig/workflows/*.yml       .github/workflows/
cp zig/dependabot.yml        .github/dependabot.yml
cp universal/release-please-config.json universal/.release-please-manifest.json .
```

Then in `release-please-config.json` set `release-type: simple` and point
`extra-files` at `build.zig.zon`.

**CodeQL does not support Zig.** Keep `codeql.yml` anyway with only the
`actions` language enabled — it scans the workflow files themselves, which is
worth having.

## The values to change

| workflow | variable | what it is |
|---|---|---|
| all | `WORKING_DIRECTORY` | the directory holding `build.zig` |
| all | `ZIG_VERSION` | **keep identical across all five files** |
| `fmt.yml` | `EXCLUDE` | directories `zig fmt` should not touch |
| `codequality.yml` | `ZLINT` | `"true"` to enable the third-party linter |
| `release.yml` | `BINARY_NAME` | the `addExecutable` name |
| `release.yml` | `OPTIMIZE` | `ReleaseSafe`, `ReleaseFast` or `ReleaseSmall` |

### Zig versions

Zig is pre-1.0 and the language and standard library still change between
releases. `ZIG_VERSION` is pinned to an exact release everywhere rather than
`master`, because a floating version turns a green `main` red with no commit
behind it — and the failure is usually a renamed standard library function, not
a defect in your code.

Keep it equal to `minimum_zig_version` in `build.zig.zon` if you declare one.
**Keep it identical in all five workflows**: `zig fmt`'s output changes between
releases, so a mismatch between `fmt.yml` and the rest means the format check
and the compiler disagree about the same file.

`ci.yml` has a `master` row that is `continue-on-error: true`. That is the row
that gives warning of a breaking change weeks before the release carrying it —
upstream breaking is upstream's news, not a reason to block a pull request.

### Changing which targets get built

Two separate lists, and they should agree:

- `ci.yml` → `cross-build.strategy.matrix.target`
- `release.yml` → `build.strategy.matrix.target`

`zig targets` prints every target the compiler knows; the list is long.

**Zig's cross-compilation is the best of any language here.** The compiler ships
every target's libc and linker, so one Linux runner produces working macOS arm64
and Windows x86-64 binaries with nothing to install. `release.yml` therefore has
no per-runner matrix at all, no QEMU, no `lipo` dance — one `ubuntu-latest` job
builds all ten targets. That is a reason to make the matrix wide rather than
narrow.

### `ReleaseSafe` vs `ReleaseFast`

`OPTIMIZE: "ReleaseSafe"` is the default here, and it is the right one for
anything that parses input it did not write.

Zig's safety checks — bounds, integer overflow, null unwrap, invalid enum tag —
are present in `Debug` and `ReleaseSafe` and **compiled out** in `ReleaseFast`
and `ReleaseSmall`. A bug that the first two catch as a clean panic is undefined
behaviour in the last two. The cost of keeping them is usually a few per cent.

This is also why `ci.yml` has an `optimize` job that builds and tests in all
four modes: testing only `Debug` tells you nothing about what you ship.

## What the workflows need from the repository

### `build.zig` with a `test` step

`zig build test` is the convention:

```zig
const test_step = b.step("test", "Run unit tests");
const unit_tests = b.addTest(.{
    .root_module = b.createModule(.{
        .root_source_file = b.path("src/root.zig"),
        .target = target,
        .optimize = optimize,
    }),
});
test_step.dependOn(&b.addRunArtifact(unit_tests).step);
```

### `-Dno-bin`, optionally

`codequality.yml` tries `zig build -Dno-bin` first, which runs semantic analysis
without codegen — much faster, and it still reports every diagnostic. It is a
convention rather than a compiler flag, so add it to `build.zig` if you want the
speedup:

```zig
const no_bin = b.option(bool, "no-bin", "Skip emitting binaries") orelse false;
if (no_bin) {
    b.getInstallStep().dependOn(&exe.step);   // analyse only
} else {
    b.installArtifact(exe);
}
```

The workflow falls back to a full build if the flag is rejected, so this is
optional.

### Hashed dependencies in `build.zig.zon`

`audit.yml` fails if any dependency has a URL and no hash. Always add
dependencies with:

```sh
zig fetch --save https://github.com/owner/repo/archive/refs/tags/v1.0.0.tar.gz
```

which writes the URL and the hash together. A URL without a hash is fetched and
trusted blind: the tarball at that URL can change and nothing notices.

## Code quality: what is and is not available

This is the thinnest area for Zig, and worth being honest about rather than
shipping a workflow that looks reassuring and checks nothing.

**Not available:** CodeQL support. No clippy or golangci-lint equivalent with a
settled rule set. No SARIF-emitting linter, so nothing in `codequality.yml`
reaches the Code scanning tab — findings arrive as job annotations instead.

**What replaces it:** the Zig compiler is much stricter than a C compiler and
reports as *errors* most of what a linter would report elsewhere — unused
variables, unreachable code, implicit narrowing, a `switch` missing an arm. So
`codequality.yml` is mostly "build in all four optimisation modes and treat the
compiler as the linter".

**zlint** is a third-party Zig linter, off by default (`ZLINT: "false"`) and
`continue-on-error: true` when enabled. It is young and its rules move between
releases, so a new rule should not turn `main` red on its own. Turn it on and
see if you like it.

**The honest summary:** for a Zig project, `ci.yml`'s four-optimisation-mode
matrix and a good test suite do the work a linter does elsewhere. That is where
to spend the effort.

## Dependency auditing: what is genuinely missing

There is no advisory database for Zig packages. No `zig audit`, no RustSec
equivalent, no OSV ecosystem — osv-scanner does not parse `build.zig.zon`. And
Dependabot has no Zig ecosystem, so nothing will open a pull request when a
dependency gets a new release.

`audit.yml` checks what can be checked:

1. **Every dependency is hashed.** The most valuable check here, because it is
   what makes the manifest worth anything as a supply-chain control.
2. **Where the dependencies come from**, printed into the log. Not pass/fail —
   there is no list of blessed hosts to compare against — but it is how anyone
   notices a dependency now points at a personal fork.
3. **Dependencies still resolve**, against an empty cache. A deleted or retagged
   tarball breaks every fresh clone, and this is better than finding out from a
   new contributor.
4. **Vendored C source**, against Trivy's filesystem scan. This part does work:
   Trivy fingerprints bundled libraries from their own headers, so it finds a
   vendored zlib with no manifest at all. If the project vendors C, this is the
   job that matters most.

**If keeping Zig dependencies current matters**, use
[Renovate](https://docs.renovatebot.com) rather than Dependabot — it handles
`build.zig.zon` with a regex manager.

## No SBOM in the release

Unlike the other languages here, `release.yml` has no SBOM step. Syft has no Zig
analyser, so it would produce an SBOM listing nothing — which is worse than
none, because an empty SBOM reads as "no dependencies" rather than "not
measured".

If the project vendors C libraries, add a step over the source tree where Syft
does find them:

```yaml
- uses: anchore/sbom-action@66cbf4bc1f1c0d2edc94016e65bc221b6bb0ad6c # v0.24.3
  with:
    path: ${{ env.WORKING_DIRECTORY }}
    format: spdx-json
    output-file: ${{ steps.package.outputs.dir }}/sbom.spdx.json
    upload-artifact: false
    upload-release-assets: false
```

## Things that will bite you

**Everything breaks on a Zig upgrade.** Expected, pre-1.0. The `master` row in
`ci.yml` is what gives you notice; read its failure before the release lands.

**`zig fmt --check` passes locally and fails in CI.** Different Zig versions.
Make `ZIG_VERSION` identical everywhere.

**`zig fmt` reformats a vendored dependency.** `zig fmt` has no ignore file, so
`EXCLUDE` in `fmt.yml` passes the exclusions as paths. Add your vendored
directories there.

**A dependency fetch fails on a clean runner but works locally.** Your global
cache has a copy of something that is no longer at its URL. That is exactly what
`audit.yml`'s fetch check is for.

**`-Dno-bin` is rejected.** `build.zig` does not define the option. Harmless —
the workflow falls back to a full build and says so.
