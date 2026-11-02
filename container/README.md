# container

```sh
cp universal/workflows/*.yml  .github/workflows/
cp container/workflows/*.yml  .github/workflows/
cp container/dependabot.yml   .github/dependabot.yml
cp universal/release-please-config.json universal/.release-please-manifest.json .
```

There is no `fmt.yml` here: `hadolint` in `codequality.yml` is the Dockerfile
equivalent, and a Dockerfile has no formatter with a settled opinion.

**This directory is additive.** A repository that builds an image also has an
application inside it, so use a language directory's workflows too and append
that language's `updates:` entry to `dependabot.yml`. Rename these files
(`container-ci.yml`, `container-release.yml`) if they would collide.

## The values to change

| workflow | variable | what it is |
|---|---|---|
| all | `DOCKERFILE`, `BUILD_CONTEXT` | if not `./Dockerfile` and `.` |
| `ci.yml` | `BUILD_ARGS` | newline-separated `KEY=value`; **never a secret** |
| `audit.yml` | `IMAGE`, `TAG_TO_SCAN` | defaults to `ghcr.io/<owner>/<repo>:latest` |
| `audit.yml` | `SEVERITY` | the floor that fails the build |
| `release.yml` | `REGISTRY`, `IMAGE_NAME` | defaults to GHCR at `<owner>/<repo>` |

`BUILD_ARGS` is recorded in the image history and readable by anyone who pulls
the image. For a secret, use `secrets:` on the build step and a
`RUN --mount=type=secret` in the Dockerfile.

## How the multi-architecture build works, and why

Building `--platform linux/amd64,linux/arm64` in one job means emulating one of
them through QEMU. It is correct and very slow — an emulated compile can be ten
times the native time, so a release that should take four minutes takes forty.

Both `ci.yml` and `release.yml` do it the other way round: **one job per
platform on a native runner.**

| platform | runner |
|---|---|
| `linux/amd64` | `ubuntu-latest` |
| `linux/arm64` | `ubuntu-24.04-arm` (native, free on public repos) |
| `linux/arm/v7`, `ppc64le`, `s390x` | `ubuntu-latest` under QEMU — slow |

`release.yml` then has a second job that assembles the per-platform digests into
one multi-arch manifest:

```
build (linux/amd64)  ─┐
                      ├─► manifest ─► tags: 1.2.3, 1.2, 1, latest, sha-…
build (linux/arm64)  ─┘              ─► attest, cosign sign
```

The build jobs push **by digest with no tag** (`push-by-digest=true`). That
matters: if each platform job claimed the tag, the second one to finish would
overwrite the first and you would ship a single-architecture image under a
multi-arch tag.

QEMU is still set up on every job, because a Dockerfile with its own
`--platform` cross-building stage needs it even on a native runner.

To add a platform, add the row to **both** `ci.yml` and `release.yml`. If they
diverge, CI stops being a rehearsal for the release.

## What the image comes with

| | where it lives | who can check it |
|---|---|---|
| BuildKit provenance (`mode=max`) | registry, beside the image | `docker buildx imagetools inspect` |
| BuildKit SBOM | registry | same |
| GitHub SLSA attestation | attestation store **and** registry | `gh attestation verify` |
| cosign signature | registry | anyone with cosign, no GitHub account |

`mode=max` records the whole build graph — every base image digest, every build
argument, every intermediate stage — rather than just the final layer. It is the
difference between provenance that proves *who* built it and provenance that
also shows *what from*.

Everything is signed over the **manifest digest**, not a tag. Signing
`image:latest` would produce a signature that stops matching the moment the tag
moves.

### Verifying

```sh
gh attestation verify oci://ghcr.io/OWNER/REPO:1.2.3 --repo OWNER/REPO

cosign verify ghcr.io/OWNER/REPO:1.2.3 \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity-regexp '^https://github.com/OWNER/REPO/'

docker buildx imagetools inspect ghcr.io/OWNER/REPO:1.2.3 \
  --format '{{ json .Provenance }}'
```

### Enforcing it at deploy time

A signature nobody checks is decoration. In Kubernetes, use
[policy-controller](https://docs.sigstore.dev/policy-controller/overview/) or
Kyverno to require it:

```yaml
# Kyverno
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-signed-images
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-signature
      match:
        any:
          - resources:
              kinds: [Pod]
      verifyImages:
        - imageReferences: ["ghcr.io/OWNER/REPO*"]
          attestors:
            - entries:
                - keyless:
                    subject: "https://github.com/OWNER/REPO/.github/workflows/release.yml@refs/heads/main"
                    issuer: "https://token.actions.githubusercontent.com"
```

## Publishing somewhere other than GHCR

GHCR needs no secrets: `secrets.GITHUB_TOKEN` with `packages: write` is enough
and is scoped to this repository. For **Docker Hub**, which has no OIDC
equivalent:

```yaml
env:
  REGISTRY: docker.io
  IMAGE_NAME: youruser/yourrepo

# and in the login step:
- uses: docker/login-action@81c6f93d4bc991e510b342a00b419b3bb4efc5c6 # v4.7.0
  with:
    registry: docker.io
    username: ${{ vars.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}
```

Use an access token from Docker Hub → Account Settings → Personal access tokens,
scoped to read/write, not your account password.

For **ECR**, **GAR** or **ACR**, use that cloud's OIDC federation rather than a
static key — all three support it, and it is the same trade as everything else
here: nothing stored, nothing to rotate.

## Dependabot for base images

The `docker` entry in `dependabot.yml` is the single most valuable one for a
container repository, and it has a prerequisite most people miss.

Most of an image's CVEs come from its base layer, and a base image tag does not
move under you: `FROM debian:12.5-slim` stays on 12.5 forever, quietly
accumulating advisories, until somebody edits the line. Dependabot is what edits
the line.

But it needs a version it can compare. These give it nothing and it will
silently do nothing:

```dockerfile
FROM alpine:latest          # no
FROM debian:stable-slim     # no
```

These work:

```dockerfile
FROM alpine:3.21                                      # yes
FROM alpine:3.21@sha256:56fa17d2a7e7f168a043a2712e63…  # better
```

With a digest, Dependabot updates the tag and the digest together, and the
digest is what makes the base immutable between builds.

The entry runs `daily` rather than weekly, and uses `prefix: fix` rather than
`build`, because a base image bump usually does carry a security fix through to
users and should cut a patch release.

**It will also offer major bumps** — `debian:12` to `debian:13`. That is a
distribution upgrade, not a dependency update. Hold them back:

```yaml
ignore:
  - dependency-name: "debian"
    update-types: ["version-update:semver-major"]
```

## Why `audit.yml` scans the published image

It scans `IMAGE:latest` — what users actually pull — rather than a freshly built
one, and it runs **daily**. That combination is the point:

An image built two months ago has a two-month-old base layer. Nothing in the
repository changes when an openssl advisory lands, so a scan of a fresh build
would miss exactly the drift this workflow exists to catch. On a pull request it
scans the pull request's own image instead, so a base image bump can be seen to
have fixed something.

`ignore-unfixed: true` is set: an advisory with no upstream patch is not
actionable, and a list full of them is a list nobody reads. Set it to `false`
before a compliance audit, where the unfixable ones have to be enumerated and
accepted.

The first run will fail if `release.yml` has never published. That is expected.

## Things that will bite you

**The push fails with "name unknown" or a 403.** GHCR requires a lower-case
path, and owner and repository names may contain capitals. Both workflows
normalise it with `tr '[:upper:]' '[:lower:]'`; if you replace that step, keep
the normalisation.

**The first push creates a private package even on a public repository.** Go to
the package → Package settings → Change visibility. It only has to be done once.

**Trivy fails with a registry rate limit.** Its databases are pulled from a
container registry that rate-limits anonymous pulls. Every Trivy step here sets
`TRIVY_DB_REPOSITORY` and `TRIVY_JAVA_DB_REPOSITORY` to the GHCR mirrors; keep
those if you edit the steps.

**hadolint fails on `DL3008`.** "Pin versions in apt-get install." Good advice,
impractical on Debian where the versions vanish from the archive. Add a
`.hadolint.yaml`:

```yaml
ignored:
  - DL3008   # apt version pinning; Debian archives do not keep old versions
failure-threshold: warning
```

**The smoke test fails on an image with no CLI.** `ci.yml` runs the image with
`--version`, then `--help`, then fails. For a server image, replace it with
something that actually proves it started — a `docker run -d`, a `curl` against
a health endpoint, and a `docker logs` on failure.

**The `gha` cache is empty on every run.** GitHub's Actions cache is scoped per
branch, and a pull request branch cannot read another branch's cache. The cache
hits on pushes to `main` and on repeat pushes to the same pull request branch,
which is where it matters.
