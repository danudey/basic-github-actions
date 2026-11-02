# node

```sh
cp universal/workflows/*.yml .github/workflows/
cp node/workflows/*.yml      .github/workflows/
cp node/dependabot.yml       .github/dependabot.yml
cp universal/release-please-config.json universal/.release-please-manifest.json .
```

Then set `release-type: node` in `release-please-config.json`, and enable
`language: javascript-typescript, build-mode: none` in `codeql.yml`.

## npm, pnpm or Yarn

Every workflow here has one knob:

```yaml
env:
  PACKAGE_MANAGER: "pnpm"   # npm | pnpm | yarn
```

It has to match the lock file in the repository. If `package.json` has a
`packageManager` field, match that instead and Corepack will enforce it.

| | lock file | install command used |
|---|---|---|
| `npm` | `package-lock.json` | `npm ci` |
| `pnpm` | `pnpm-lock.yaml` | `pnpm install --frozen-lockfile` |
| `yarn` | `yarn.lock` | `yarn install --immutable` (Yarn 1: `--frozen-lockfile`) |

All three are the frozen-lockfile mode, so a lock file that no longer matches
`package.json` fails rather than resolving something new.

The publish in `release.yml` always uses `npm` regardless, because provenance is
an npm registry feature. The manager only affects how dependencies are installed
and the project is built.

## The values to change

| workflow | variable | what it is |
|---|---|---|
| all | `PACKAGE_MANAGER` | `npm`, `pnpm` or `yarn` |
| all | `WORKING_DIRECTORY` | the directory holding `package.json` |
| all | `PNPM_VERSION` | pnpm only |
| `codequality.yml` | `LINT_PATHS` | what ESLint lints; `.` with a flat config |
| `fmt.yml` | `FORMATTER` | `prettier` or `biome` — **not both** |
| `release.yml` | `REGISTRY_URL` | for a private or scoped registry |
| `release.yml` | `NPM_TAG` | `latest`, or `next` for a prerelease line |

### The Node version matrix

`ci.yml`'s `matrix.node` should match `engines.node` in `package.json`. Nothing
checks that agreement, and a version in `engines` but not here is the usual
source of "it does not work on Node 20".

Even-numbered Node releases are the LTS ones. Testing an odd release is only
worth it if you intend to support it.

### Script names

`ci.yml` runs `typecheck`, `build` and `test`. The first two use `--if-present`
so a project without them still passes; `test` does not, because a repository
with no test script should fail loudly rather than quietly.

If your scripts are named differently, change the step, not the script — the
names in `package.json` are what contributors type locally.

### Architectures

For pure JavaScript and TypeScript the OS matrix is the whole story, and what it
tests is the filesystem: path separators, case sensitivity, line endings, file
locking on Windows. Those genuinely break tests.

If the package has a native addon (`node-gyp`, `napi-rs`, `prebuildify`), the
architecture matrix that matters is the prebuild matrix, and it belongs in
`release.yml`. `napi-rs` has its own generator for that; the pattern is the same
as `python/workflows/release-binary-wheels.yml` — one job per runner, artifacts
collected by a publish job.

## What the workflows need from the repository

### `eslint.config.js`

`codequality.yml` uses the project's own ESLint, not a pinned one, so the CI
result matches what a developer sees locally. It does not install ESLint — that
has to be a devDependency.

A starting flat config for TypeScript:

```js
// eslint.config.js
import js from "@eslint/js";
import tseslint from "typescript-eslint";

export default tseslint.config(
  js.configs.recommended,
  // recommendedTypeChecked is the single biggest upgrade available to a
  // TypeScript project: it uses the type checker, so it finds a floating
  // promise, an `any` leaking through a boundary, an await on a non-promise.
  // It needs projectService below to know about your tsconfig.
  ...tseslint.configs.recommendedTypeChecked,
  ...tseslint.configs.stylisticTypeChecked,
  {
    languageOptions: {
      parserOptions: {
        projectService: true,
        tsconfigRootDir: import.meta.dirname,
      },
    },
    rules: {
      "@typescript-eslint/no-unused-vars": [
        "error",
        { argsIgnorePattern: "^_" },
      ],
      "@typescript-eslint/consistent-type-imports": "error",
      "no-console": "warn",
      eqeqeq: ["error", "always"],
    },
  },
  {
    // Tests do things production code should not.
    files: ["**/*.test.ts", "**/*.spec.ts"],
    rules: {
      "@typescript-eslint/no-non-null-assertion": "off",
      "@typescript-eslint/no-explicit-any": "off",
    },
  },
  {
    // Nothing generated, and nothing that is not ours.
    ignores: ["dist/**", "coverage/**", "**/*.d.ts"],
  },
);
```

The SARIF formatter (`@microsoft/eslint-formatter-sarif`) is installed by the
workflow with `--no-save`, so it does not need to be in `package.json` and the
lock file stays untouched.

### A formatter, and only one

`prettier` or `biome`, never both — they disagree about whitespace and you get
an unwinnable check. Whichever you pick must be a devDependency, so the CI
version matches the local one; a formatter one minor version ahead reformats
different files and the check becomes noise.

```json
{
  "prettier": {
    "printWidth": 100,
    "singleQuote": false,
    "trailingComma": "all"
  }
}
```

Add a `.prettierignore` for generated output. A formatter that reformats
generated files produces a diff on every regeneration.

### `"files"` in `package.json`

`ci.yml`'s `pack` job exists because of this field. It packs the tarball npm
would publish, lists its contents, and fails if there is almost nothing in it —
which is what happens when `files` excludes the built output, or the build
artefact was never generated. That package installs perfectly cleanly and then
fails on import, and finding out at release time is the bad way.

```json
{
  "files": ["dist", "README.md", "LICENSE"],
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    }
  }
}
```

## Publishing to npm, with no token

`release.yml` uses npm **trusted publishing**: you tell npmjs.com once that this
repository's workflow may publish this package, and the workflow proves who it
is with an OIDC token.

1. Create the GitHub environment `npm` (Settings → Environments). Add a required
   reviewer for a manual gate on every release.
2. On npmjs.com → the package → Settings → Trusted publisher, add: repository,
   workflow filename `release.yml`, environment `npm`.

If the registry does not support it, fall back to a granular automation token in
`secrets.NPM_TOKEN` and uncomment the `NODE_AUTH_TOKEN` line on the publish
step.

**`--provenance` works either way.** It records which repository, workflow and
commit produced the tarball, verifies it against Sigstore, and shows a "Built
and signed on GitHub Actions" badge on the package page. It needs
`id-token: write`, a public repository, and a public package.

For a **scoped package**, add this or the publish fails as private:

```json
{ "publishConfig": { "access": "public" } }
```

## Things that will bite you

**The publish fails with a 404 on a scoped package.** Missing
`publishConfig.access`. See above.

**Provenance is silently absent.** The npm bundled with Node 20 is old enough to
lack it on some releases, which is why the publish job runs
`npm install --global npm@<pinned version>` first. The version is pinned, not
`@latest`, so a compromised npm release cannot reach the publish job unreviewed.

**`npm audit` fails permanently.** The gate in `audit.yml` deliberately audits
production dependencies only (`--omit=dev`). A vulnerable dev dependency is a
real risk to the build machine but is not shipped, and the npm dev tree is large
enough that auditing all of it fails forever and teaches everyone to ignore the
check. The dev tree is still covered by the osv-scanner job, where it produces
an alert rather than a red build. If your build is itself security-sensitive,
remove the flag.

**Dependabot opens twenty pull requests a week.** `dependabot.yml` groups them
into three: dev tooling (merge largely unread), runtime minor/patch, and runtime
major. A major bump is a migration and belongs in its own reviewed pull request,
which is why it is not batched with the patches.

**`yarn install --immutable` fails on Yarn 1.** Yarn 1 spells it
`--frozen-lockfile`. The workflows try the modern flag first and fall back, so
both work, but upgrading to a modern Yarn is the better answer.

**The lock file changed during install.** `fmt.yml` catches this and says what to
do. It means the lock file did not match `package.json` — run your manager's
install locally and commit the result.
