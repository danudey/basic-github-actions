# rust

```sh
cp universal/workflows/*.yml .github/workflows/
cp rust/workflows/*.yml      .github/workflows/
cp rust/dependabot.yml       .github/dependabot.yml
cp universal/release-please-config.json universal/.release-please-manifest.json .
```

Then set `release-type: rust` in `release-please-config.json`, and enable
`language: rust, build-mode: none` in `codeql.yml`.

`pr-title.yml` exists in both `universal/` and here; they are the same file.
Copy either one.

## The values to change

| workflow | variable | what it is |
|---|---|---|
| all | `WORKING_DIRECTORY` | the directory holding the top-level `Cargo.toml` |
| all | `TOOLCHAIN` | `stable`, or an exact `1.85.0` |
| `ci.yml` | the MSRV row in `matrix.include` | **must** equal `rust-version` in `Cargo.toml` |
| `fmt.yml` | `TOOLCHAIN` | set to `nightly` if `rustfmt.toml` uses nightly-only options |
| `release.yml` | `BINARY_NAME` | the `[[bin]]` target name, and the archive stem |
| `release.yml` | `EXTRA_FILES` | files copied into every archive |

### Workspaces

Leave `WORKING_DIRECTORY` at the workspace root. Every command uses
`--workspace`, so members are picked up automatically, and `dependabot.yml`
needs one `cargo` entry, not one per member.

### Changing which targets get built

Two separate lists, and they should agree:

- `ci.yml` → `cross-build.strategy.matrix.target` — proves it still compiles.
- `release.yml` → `build.strategy.matrix.include` — produces the artifacts.

`rustc --print target-list` prints everything rustc knows.

**How the cross-compilation works.** `cargo zigbuild` uses Zig as the linker, so
one Linux runner links for every musl and glibc target with no cross toolchain
installed. Set `zig: true` on a matrix row to route it through zigbuild. The
alternative is [`cross`](https://github.com/cross-rs/cross), which uses Docker
containers per target and handles C-dependency-heavy crates better but is
noticeably slower — worth switching to if a crate in your tree builds C code
that zigbuild cannot link.

Windows rows have no `zig:` flag: both `x86_64` and `aarch64` MSVC targets
cross-build from the x64 runner with the stock toolchain.

macOS is deliberately not in the matrix. `build-macos` is its own job because
the universal binary needs both slices present in one job to run `lipo`.

### The macOS universal binary

`build-macos` produces three artifacts: `aarch64`, `x86_64`, and a
`universal-apple-darwin` built with `lipo -create`. Note the `codesign --force
--sign -` that follows: `lipo` strips the linker's ad-hoc signature, and an
unsigned binary **will not run at all** on Apple Silicon. Deleting that line
produces artifacts that fail with a SIGKILL and no useful message.

## What the workflows need from the repository

### `[lints.clippy]` in `Cargo.toml`

`codequality.yml` runs clippy but the lint selection belongs in `Cargo.toml`, so
developers see the same lints locally:

```toml
[lints.rust]
unsafe_code = "forbid"           # or "warn" if the crate needs it
missing_docs = "warn"
unreachable_pub = "warn"
unused_qualifications = "warn"

[lints.clippy]
# The `priority = -1` is required on a group so individual rules below can
# override it; without it Cargo rejects the table.
all = { level = "warn", priority = -1 }
pedantic = { level = "warn", priority = -1 }
nursery = { level = "warn", priority = -1 }

# The ones worth being strict about in a binary that other people run.
unwrap_used = "warn"
expect_used = "warn"
panic = "warn"
indexing_slicing = "warn"
integer_division = "warn"
todo = "warn"
unimplemented = "warn"
dbg_macro = "warn"

# pedantic rules that are more noise than signal in practice.
module_name_repetitions = "allow"
must_use_candidate = "allow"
missing_errors_doc = "allow"
missing_panics_doc = "allow"
```

In a workspace, put this in `[workspace.lints]` at the root and add
`[lints] workspace = true` to each member.

### `deny.toml`

`audit.yml`'s `cargo-deny` job does nothing useful without it. `cargo deny init`
writes a starting point; the two sections worth reading are:

```toml
[licenses]
version = 2
# What you are willing to ship. cargo-deny fails on anything not listed.
allow = [
  "MIT", "Apache-2.0", "Apache-2.0 WITH LLVM-exception",
  "BSD-2-Clause", "BSD-3-Clause", "ISC", "Unicode-3.0", "Zlib",
]
# A crate with no machine-readable licence, cleared by hand.
[[licenses.clarify]]
name = "ring"
expression = "MIT AND ISC AND OpenSSL"
license-files = [{ path = "LICENSE", hash = 0xbd0eed23 }]

[bans]
# Three versions of the same crate linked into one binary is a real cost:
# larger artifact, longer compile, and two copies of a type that look
# identical and are not.
multiple-versions = "warn"
wildcards = "deny"
deny = [
  # An unmaintained crate you have a replacement for.
  { name = "openssl", use-instead = "rustls" },
]
skip-tree = [
  # A dependency whose own tree duplicates half the ecosystem, accepted.
  { name = "windows-sys" },
]

[sources]
unknown-registry = "deny"
unknown-git = "deny"
# Only crates.io, unless you list an exception here.
allow-registry = ["https://github.com/rust-lang/crates.io-index"]

[advisories]
version = 2
# Advisories you have assessed and cannot act on. Say why, and when to look
# again — an unexplained entry outlives the problem it was added for.
ignore = [
  # { id = "RUSTSEC-2024-0000", reason = "not reachable; recheck 2026-06" },
]
```

The job runs `advisories`, `bans`, `licenses` and `sources` as four separate
matrix rows on purpose, so "a GPL dependency appeared" and "there are now three
versions of `rand`" are separate red checks rather than one you have to read to
find out which.

### `.cargo/audit.toml`

For `cargo audit` specifically. `--deny warnings` is set, so unmaintained and
unsound notices fail too:

```toml
[advisories]
ignore = ["RUSTSEC-2024-0000"]   # document each one
informational_warnings = ["unmaintained", "unsound"]
```

### `rustfmt.toml`

Optional. If it uses nightly-only options — `group_imports`,
`imports_granularity`, `wrap_comments` — set `TOOLCHAIN: "nightly"` in
`fmt.yml`. **Stable rustfmt silently ignores unknown options**, so the check
passes while the file is unformatted, which is worse than failing.

### `Cargo.lock`, committed

Every command uses `--locked`. A stale lock fails rather than silently resolving
something new — which is the point.

## Publishing to crates.io, with no API token

`release.yml`'s `publish-crate` job uses crates.io **Trusted Publishing**:

1. Create the GitHub environment `crates-io` (Settings → Environments).
2. On the crate's crates.io page → Settings → Trusted Publishing, add:
   repository, workflow filename `release.yml`, environment `crates-io`.

`rust-lang/crates-io-auth-action` then exchanges the workflow's OIDC token for a
short-lived registry token. If Trusted Publishing is not available, delete that
step and set `CARGO_REGISTRY_TOKEN` from a secret instead.

Delete the whole job for an application nobody will `cargo install`.

`cargo publish --no-verify` is used because the build jobs already compiled this
commit for every target; re-verifying only adds minutes.

## Things that will bite you

**A Dependabot bump fails only on the MSRV row.** A dependency raised its own
minimum Rust version above yours. Either raise `rust-version` in `Cargo.toml`
(a decision about who you support) or hold the dependency back in
`dependabot.yml`. Dependabot will not touch `rust-version` itself, correctly.

**Clippy turns `main` red with no commit behind it.** A new stable release added
lints. That is the trade for `TOOLCHAIN: "stable"`; pin an exact version if
currency matters less than a stable signal.

**`cargo test --all-targets` passes but doc tests fail.** `--all-targets`
excludes doc tests, which is why `ci.yml` runs them as a second command. Rust
doc examples are compiled and executed, so a failure there is a real failure of
documented behaviour.

**The crate builds under `--all-features` and fails for a user.** A
feature-gated import used unconditionally. `ci.yml`'s
`cargo check --no-default-features` catches it.

**`cargo doc` fails on a broken intra-doc link.** `fmt.yml` sets
`RUSTDOCFLAGS: -D warnings` deliberately: a broken link on docs.rs is not
fixable without cutting another release.

**The release binary is 40MB.** `RUSTFLAGS: "-C strip=symbols"` is already set.
For more, add to `Cargo.toml`:

```toml
[profile.release]
lto = "fat"
codegen-units = 1
panic = "abort"      # only if you do not catch panics
```

## A note on what moved

If you had the earlier single `ci.yml` in this directory, clippy and rustfmt
have moved out of it into `codequality.yml` and `fmt.yml`. Same commands, three
status checks instead of one — so a formatting nit no longer reads as "the build
is broken", and clippy findings now reach the Code scanning tab.
