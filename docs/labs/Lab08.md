# Lab 8: Publishing artifacts: registries, versioning and security

Use the results of **Lab 7** as the starting point for this lab.

In [Lab 7](Lab07.md), you built a pipeline that builds the application, tests it on four levels, and pushes a tested Docker image. That last step was only a quick `docker push`. In real projects, **publishing** is a stage of its own: it decides **where** the build results are stored, **what they are called** (their version), **who may read and write them**, and **how a build moves from "tested" to "released"**. In this lab, you will extend the pipeline with a proper `publish` stage.

You will cover:

1. **artifact registries**: what they are and which kinds exist (public, platform-bundled, self-hosted, cloud),
2. **versioning**: how to name artifacts, and how to publish **test** and **production** versions,
3. publishing **Docker images** and **native packages** (`.nupkg` with a `.dll`, `.jar`, `npm` packages),
4. working with two kinds of registries: **open** (GitHub Container Registry, Docker Hub) and **self-hosted**.

> **Prerequisites:** the `testing-demo` repository and its workflows from Lab 7, the Docker build from Lab 6, git tags from Lab 4, and the required checks from Labs 6 and 7. Accounts and tools: GitHub, **Docker Hub** (a free account), **Node.js (LTS)** with npm, the **.NET SDK 10**, a **JDK 21** with Maven and Docker.

---

## Concepts

### What is an artifact, and what is a registry?

An **artifact** is a file (or a set of files) that the build produces and that is later deployed or consumed: a Docker image, a `.jar`, a `.nupkg` (a NuGet package containing `.dll` files), an npm tarball, a Helm chart, a ZIP with a website.

An **artifact registry** (also *repository manager*, *package registry*) is a server that **stores artifacts, names them by version, and controls who may upload and download them**. It is the handover point between the build and everything that comes after it: other developers, test environments, production servers, customers.

Why not just copy the files to a server or keep them as workflow artifacts (`actions/upload-artifact`)?

| | Workflow artifact (Lab 6/7) | Artifact registry |
|---|---|---|
| Purpose | pass files between jobs, keep evidence of a run | store and distribute **releases** |
| Lifetime | days to weeks, then deleted | as long as you decide |
| Naming | run-specific | **versioned**, addressable by name + version |
| Consumed by | people who open the run page | tools: `docker pull`, `dotnet add package`, `mvn`, `npm install` |
| Access control | whoever can see the repository | fine-grained permissions, tokens, audit |

### Formats and registries

Each ecosystem has its own package format and registry protocol, and most registries support several of them:

| Artifact | Format / contents | Version is set in | Typical commands |
|----------|-------------------|-------------------|------------------|
| Docker image | OCI image (layers + manifest) | the **tag** (and an immutable digest) | `docker push`, `docker pull` |
| .NET library | `.nupkg` (ZIP with `.dll` files and metadata) | `<Version>` in the `.csproj` | `dotnet pack`, `dotnet nuget push` |
| Java library | `.jar` + `pom.xml` (+ sources, javadoc) | `<version>` in `pom.xml` | `mvn deploy` |
| JavaScript package | tarball with `package.json` | `version` in `package.json` | `npm publish` |
| Python package | wheel / sdist | `version` in `pyproject.toml` | `twine upload` |
| Anything else | a file | the file name | GitHub Releases, generic repositories |

The kinds of registries you will meet:

| Kind | Examples | Good for | Watch out for |
|------|----------|----------|---------------|
| **Public ecosystem registries** | Docker Hub, Maven Central, nuget.org, npmjs.com | distributing open-source releases to everyone | **permanent**: published packages usually cannot be deleted; never publish tests, secrets or unfinished builds |
| **Platform registries** | GitHub Packages and GitHub Container Registry (GHCR), GitLab Package Registry | artifacts next to the code, with the same permissions and the built-in `GITHUB_TOKEN` | lock-in to the platform; the Maven/NuGet/npm registries of GitHub require authentication even for reading |
| **Self-hosted** | Docker `registry` (Distribution), Harbor, Sonatype Nexus, JFrog Artifactory, Verdaccio (npm) | full control, data on your own servers, caching of public packages | **you** run, secure, update and back it up |
| **Cloud registries (not covered in this lab)** | AWS ECR and CodeArtifact, Azure Container Registry and Artifacts, Google Artifact Registry | production deployments in that cloud, integration with cloud identity (IAM) | cost, vendor-specific setup, cloud IAM complexity |

### The flow of an artifact: build once, promote many times

```text
 commit ─► build ─► tests ─► IMAGE built and tested ──► "dev"     every commit on main     (test channel)
                              │                    ──► "rc"      tag v1.3.0-rc.1           (test/staging channel)
                              │                    ──► "release" tag v1.3.0                (production channel)
                              ▼
                      the same bytes (the same digest) all the way:
                      promotion = giving it another name, NOT rebuilding it
```

A **promotion** does not rebuild. The thing that goes to production is exactly the thing that was tested. If you rebuild for production, you are shipping something that nobody has tested (a different base image may have been pulled, a dependency may have changed).

---

## Versioning

### Semantic Versioning (SemVer)

`MAJOR.MINOR.PATCH`, optionally with a **pre-release** and **build metadata**:

```text
1.4.2                      a release
1.4.2-rc.1                 a pre-release: sorts BEFORE 1.4.2
1.4.2-dev.57               a pre-release (a development build)
1.4.2+sha.3f2c1ab          build metadata: ignored when ordering versions
```

| Change | Increase |
|--------|----------|
| incompatible API change | **MAJOR** |
| new functionality, backwards compatible | **MINOR** |
| bug fix, backwards compatible | **PATCH** |

Order: `1.4.0-dev.2` < `1.4.0-dev.10` < `1.4.0-rc.1` < `1.4.0` < `1.4.1`. (Compare the numeric parts as numbers. `dev.10` is *newer* than `dev.2`.)

Alternatives: **CalVer** (`2026.11.3`: year, month, counter), good for applications and services where "what breaks" is not the relevant question; **build numbers** (`build-1432`) for internal use only.

### Mutable and immutable names

| Name | Mutable? | Example | Use |
|------|----------|---------|-----|
| Exact version | **should be immutable** | `1.4.2`, `1.4.2-rc.1` | the thing you deploy and reproduce |
| Floating tag | moves | `1.4`, `1`, `latest`, `dev` | convenience ("latest 1.4 patch") |
| Docker **digest** | **always immutable** | `lendit@sha256:ab12...` | exact reproducibility, deployment manifests |
| Maven `-SNAPSHOT` | mutable by design | `1.4.0-SNAPSHOT` | "the latest development build" |
| npm **dist-tag** | moves | `latest`, `next`, `dev` | the channel; points at a version |

Rule: **deploy by an exact version (or digest), never by `latest`.** `latest` tells you nothing about what is running, and it changes under your feet.

### Test and production versions: the channels

| Channel | Created when | Version example | Who uses it | Retention |
|---------|--------------|-----------------|-------------|-----------|
| **dev** | every commit on `main` that passes the tests | `1.4.0-dev.57` | developers, the test environment | short (clean up) |
| **rc** (release candidate) | a git tag `v1.3.0-rc.1` | `1.3.0-rc.1` | staging, acceptance testing | until the release |
| **release** | a git tag `v1.3.0` | `1.3.0` | production, customers | **forever, immutable** |

How the channels are separated technically, depending on the ecosystem:

| Ecosystem | Separation of test and production |
|-----------|-----------------------------------|
| Docker | different **tags** (`1.4.0-dev.57` vs `1.3.0`), and/or different **repositories** (`lendit/test`, `lendit/prod`) with different permissions and an immutability setting |
| NuGet | **pre-release versions** (`1.4.0-dev.57`): consumers receive them only if they ask (`--prerelease`) |
| Maven | `-SNAPSHOT` / pre-release versions vs release versions (some repository managers have separate *snapshot* and *release* repositories) |
| npm | **dist-tags**: `dev`, `next` and `latest`; `npm install pkg` installs only `latest` |

### Connection to the branching strategy (Lab 5) and to git tags (Lab 4)

| Git event | Channel | What happens |
|-----------|---------|--------------|
| pull request | none | build + test only. **Never publish from a pull request** |
| push to `main` | dev | publish `…-dev.N` |
| tag `vX.Y.Z-rc.N` | rc | promote the image built for this commit; publish the packages |
| tag `vX.Y.Z` | release | promote; publish; create a GitHub Release |

In **Git Flow**, `develop` plays the role of the dev channel, `release/*` the rc channel, and tags on `main` the release channel.

---

## Security concepts for publishing

Publishing is where secrets and trust meet, so it is a favourite target for attackers. The topics you will meet in this lab:

| Topic | Question | Practice |
|-------|----------|----------|
| **Authentication** | *who* is calling? | passwords/tokens for registries; for cloud, **IAM identities** instead of passwords |
| **Authorization** | *what* may they do? | **least privilege**: CI may push, servers may only pull, nobody gets admin |
| **Secrets** | where are credentials stored? | never in the repository, the image or the logs; use GitHub **secrets** and **environment secrets** |
| **Short-lived credentials** | how long is a stolen token useful? | prefer the built-in `GITHUB_TOKEN` and **OIDC** (a token valid for minutes) over static keys |
| **Transport** | can someone listen in? | TLS for every registry |
| **Gating** | can a bad build be published? | publish only after tests pass; **environments** with required approvals for production |
| **Immutability** | can a release be changed later? | immutable tags/versions; deploy by digest |
| **Content** | what is inside the artifact? | vulnerability **scanning**; minimal images; no secrets in layers (Lab 6) |
| **Provenance** | where does it come from? | signatures, SBOM, build attestations (introduced in Task 2) |
| **Supply chain** | what do we pull in? | pin versions and third-party actions; protect against **dependency confusion** (Task 4) |

Credential types you will use, from worst to best:

| Credential | Lifetime | Scope | Verdict |
|------------|----------|-------|---------|
| Account password | until changed | everything | **never** use it in automation |
| Personal access token (PAT) | up to forever | the scopes you tick | acceptable on your laptop. Use the minimum scopes and an expiry date |
| Deploy/service account key (static) | until rotated | what you assign | works, but must be rotated and stored securely |
| `GITHUB_TOKEN` | **one workflow run** | the repository, as granted by `permissions:` | the best choice for GitHub-hosted registries |
| **OIDC federation** (GitHub -> AWS) | **minutes** | one IAM role | the best choice for cloud registries; **no stored secret at all** |

---

## Task 0: Preparation

You will use two repositories:

| Repository | Content | Used in |
|------------|---------|---------|
| `testing-demo` (from Lab 7) | the LendIt application (C# and/or Java), its Dockerfile and test pipeline | Docker image tasks (Tasks 1, 2, 4) |
| `publish-demo` (**new**, public) | three small **libraries**: a .NET class library, a Java library, an npm package | native package tasks (Tasks 1, 3, 4) |

In `testing-demo`, check that the pipeline from Lab 7 is green, and that `main` has the required checks. Create `publish-demo` with this layout:

```text
publish-demo/
├── .github/
│   ├── scripts/version.sh        # Task 1
│   └── workflows/publish-packages.yml   # Task 3
├── dotnet-lib/                   # LendIt.Pricing  → NuGet
├── java-lib/                     # lendit-pricing  → Maven
└── js-lib/                       # @<owner>/lendit-pricing → npm
```

The libraries all contain the same tiny piece of logic, the **late fee** from Lab 7 (0.50 per day after 14 days, at most 10.00). That is enough, because this lab is about **packaging and publishing**, not about the code.

**`dotnet-lib/src/LendIt.Pricing/LateFee.cs`**

```csharp
namespace LendIt.Pricing;

public static class LateFee
{
    public static decimal Calculate(int daysKept)
    {
        var daysLate = daysKept - 14;
        return daysLate <= 0 ? 0m : Math.Min(daysLate * 0.50m, 10.00m);
    }
}
```

```bash
cd publish-demo
dotnet new classlib -n LendIt.Pricing -o dotnet-lib/src/LendIt.Pricing
dotnet new xunit    -n LendIt.Pricing.Tests -o dotnet-lib/tests/LendIt.Pricing.Tests
dotnet add dotnet-lib/tests/LendIt.Pricing.Tests reference dotnet-lib/src/LendIt.Pricing
rm dotnet-lib/src/LendIt.Pricing/Class1.cs
# add LateFee.cs, a README.md next to the .csproj, and 3 tests in the test project
```

**`java-lib/src/main/java/hr/lendit/pricing/LateFee.java`**

```java
package hr.lendit.pricing;

import java.math.BigDecimal;

public final class LateFee {
    private LateFee() {}

    public static BigDecimal calculate(int daysKept) {
        int daysLate = daysKept - 14;
        if (daysLate <= 0) {
            return BigDecimal.ZERO;
        }
        return new BigDecimal("0.50").multiply(BigDecimal.valueOf(daysLate)).min(new BigDecimal("10.00"));
    }
}
```

**`js-lib/index.js`**

```javascript
'use strict';

function lateFee(daysKept) {
  const daysLate = daysKept - 14;
  return daysLate <= 0 ? 0 : Math.min(daysLate * 0.5, 10);
}

module.exports = { lateFee };
```

Write at least three tests in each library, as in Lab 7 (`node --test` is enough for the npm package: create `js-lib/index.test.js` with `node:test` and `node:assert`). Copy the Maven wrapper (`mvnw`, `mvnw.cmd`, `.mvn/`) from your Lab 7 `java-app` into `java-lib/`.

### Expected result of Task 0

- `publish-demo` with the three libraries and their tests, all passing locally (`dotnet test`, `./mvnw test`, `npm test`).

---

## Task 1: A versioning strategy and an automatic version

### 1.1 Decide the rules

Create `docs/versioning.md` in `publish-demo` (and later in your project) and fill in:

1. **The scheme** (SemVer is expected) and what counts as MAJOR, MINOR and PATCH for your artifacts.
2. **The channels** (`dev`, `rc`, `release`), the tag format that triggers each, and where each channel is published.
3. **Which names are mutable** (floating tags) and which are immutable.
4. **Who may create a release** (who may push a `v*` tag).

### 1.2 Compute the version automatically

Never type a version number into a workflow. **Derive it** from the git context, so that a human decision (creating a tag) is the only thing that makes a release. Create `.github/scripts/version.sh`:

```bash
#!/usr/bin/env bash
# Prints VERSION and CHANNEL (dev | rc | release) for the current git context.
# Usage in a workflow:  .github/scripts/version.sh >> "$GITHUB_OUTPUT"
set -euo pipefail

if [[ "${GITHUB_REF_TYPE:-}" == "tag" ]]; then
  VERSION="${GITHUB_REF_NAME#v}"
  if [[ ! "$VERSION" =~ ^[0-9]+\.[0-9]+\.[0-9]+(-rc\.[0-9]+)?$ ]]; then
    echo "Tag '${GITHUB_REF_NAME}' is not vMAJOR.MINOR.PATCH or vMAJOR.MINOR.PATCH-rc.N" >&2
    exit 1
  fi
  if [[ "$VERSION" == *-rc.* ]]; then CHANNEL=rc; else CHANNEL=release; fi
else
  # development build: the NEXT minor version, as a pre-release, numbered by the workflow run
  LAST_TAG="$(git describe --tags --abbrev=0 --match 'v[0-9]*' 2>/dev/null || echo v0.0.0)"
  LAST="${LAST_TAG#v}"
  LAST="${LAST%%-*}"                                  # strip a possible -rc.N
  IFS=. read -r MAJOR MINOR _ <<< "$LAST"
  VERSION="${MAJOR}.$((MINOR + 1)).0-dev.${GITHUB_RUN_NUMBER:-0}"
  CHANNEL=dev
fi

echo "VERSION=${VERSION}"
echo "CHANNEL=${CHANNEL}"
```

```bash
chmod +x .github/scripts/version.sh
git update-index --chmod=+x .github/scripts/version.sh      # keep the executable bit in git
```

Try it locally, simulating the workflow environment:

```bash
git tag -a v1.0.0 -m "First release"
GITHUB_REF_TYPE=branch GITHUB_REF_NAME=main GITHUB_RUN_NUMBER=57 .github/scripts/version.sh
# VERSION=1.1.0-dev.57   CHANNEL=dev
GITHUB_REF_TYPE=tag GITHUB_REF_NAME=v1.1.0-rc.1 .github/scripts/version.sh
# VERSION=1.1.0-rc.1     CHANNEL=rc
GITHUB_REF_TYPE=tag GITHUB_REF_NAME=v1.1.0 .github/scripts/version.sh
# VERSION=1.1.0          CHANNEL=release
GITHUB_REF_TYPE=tag GITHUB_REF_NAME=v1.1 .github/scripts/version.sh
# error: the tag is rejected
```

A workflow using it needs the tags: use `actions/checkout` with `fetch-depth: 0` (a full clone). You will see the complete use in Tasks 2 and 3.

> This script is intentionally simple. Real projects often use a tool for the same job: **MinVer** (.NET, versions from git tags), **GitVersion**, **semantic-release** or **release-please** (versions derived from *conventional commit* messages such as `feat:` and `fix:`), `versions-maven-plugin` (Maven). What does the script get wrong? (Hint: what is the "next" version after a `v1.3.0-rc.1` tag, before `v1.3.0` is released?)

### 1.3 The tags per ecosystem

| Ecosystem | Dev | RC | Release |
|-----------|-----|----|---------|
| Docker tags | `1.1.0-dev.57`, `dev` | `1.1.0-rc.1`, `rc` | `1.1.0`, `1.1`, `1`, `latest` |
| NuGet version | `1.1.0-dev.57` | `1.1.0-rc.1` | `1.1.0` |
| Maven version | `1.1.0-dev.57` | `1.1.0-rc.1` | `1.1.0` |
| npm version + dist-tag | `1.1.0-dev.57` + `dev` | `1.1.0-rc.1` + `next` | `1.1.0` + `latest` |

(Maven teams often use `1.1.0-SNAPSHOT` for the dev channel. A `-SNAPSHOT` version is *overwritten* by every build, so consumers always get the newest, but two machines can have different bytes for the same version. Unique `-dev.N` versions avoid that, at the price of more versions to clean up.)

### Questions for your report

1. Why must a **release** version be immutable, and what could go wrong if someone could re-publish `1.1.0` with different content?
2. Why is the `latest` tag a bad choice for a deployment manifest? Give a concrete failure scenario.
3. Why does `1.1.0-rc.1` sort *before* `1.1.0`, and why does that matter for NuGet and npm consumers?
4. What is the weakness of the script from the hint in 1.2? How would `MinVer` or `semantic-release` handle it?

### Expected result of Task 1

- `docs/versioning.md` and a working `version.sh` with the four outputs from 1.2.
- Answers to the questions.

---

## Task 2: Publishing Docker images: test and production

This task continues in `testing-demo`. The image of the LendIt app from Lab 7 will travel through three channels (`dev`, `rc`, `release`) and be stored in two registries: **GHCR** (all channels) and **Docker Hub** (releases).

### 2.1 GHCR: the platform registry

You pushed an image to GHCR manually in Lab 6. Recap and look at the security settings:

- **Visibility:** a package is **private** by default. A *public* package can be pulled by anyone without a login. Make your **dev/test** images private, and release images of an open-source project public. (Package settings → *Change package visibility*.)
- **Link to the repository:** the label `org.opencontainers.image.source=https://github.com/<owner>/<repo>` in the Dockerfile (Lab 6) connects the package to the repository, so that the repository's permissions apply and the package appears on the repository page.
- **Who may push:** in the package settings → **Manage Actions access**, the repository whose workflows publish the package must have the *Write* role. A package that a workflow created from a linked repository is already connected.
- **Credentials:** a workflow uses the built-in `GITHUB_TOKEN` and the permission `packages: write`. Nothing is stored. A person pushing from a laptop uses a PAT with the `write:packages` scope (and an expiry date).

```bash
echo "$GHCR_PAT" | docker login ghcr.io -u <user> --password-stdin
cat ~/.docker/config.json
```

Look at the output of the last command. **Where is your credential stored, and is it encrypted?** (On Linux without a *credential helper*, it is only base64-encoded. Docker Desktop uses the operating system's credential store.) Then `docker logout ghcr.io`.

### 2.2 GitHub Environments: separating test from production

An **environment** is a named deployment target in GitHub (Settings → Environments) that can have its own **secrets, variables and protection rules**. A job that declares `environment: production` can read that environment's secrets, and waits for the protection rules.

Create two environments in `testing-demo`:

| Environment | Protection rules | Used for |
|-------------|------------------|----------|
| `test` | none | dev and rc publishing |
| `production` | **Required reviewers** (yourself or a teammate); **Deployment branches and tags**: only tags matching `v*` | release publishing |

(Required reviewers for **private** repositories need a paid plan. Public repositories have this feature on every plan.)

This is a security control, not a formality: even if somebody pushes a malicious workflow change on a branch, it cannot read the production secrets, because the branch is not allowed to deploy to `production`.

### 2.3 The pipeline: build once, publish to `dev` automatically

Change the pipeline from Lab 7 (`.github/workflows/dotnet-ci.yml`, or the Java one).

**(a)** Remove `tags: ["v*"]` from the `on: push:` trigger. Tags will be handled by a separate release workflow (2.4). Otherwise a tag would rebuild the image, which would violate "build once".

**(b)** In the last step of the `e2e` job, push **only the tested image identified by the commit hash**. Replace the content of the step `Push the tested image` with:

```yaml
      - name: Push the tested image (identified by the commit)
        if: github.event_name != 'pull_request'
        run: docker push "$IMAGE:$GITHUB_SHA"
```

That `:<commit-sha>` image is the **build identity**: it exists only if the commit passed all tests, and everything else is a name added to it.

**(c)** Add a job that gives the tested image its **dev** names:

```yaml
  publish-test:
    name: publish dev image
    needs: e2e
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: test
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0                       # the version script needs the tags

      - name: Compute version
        id: v
        working-directory: .                   # this workflow's default is dotnet-app/
        run: .github/scripts/version.sh >> "$GITHUB_OUTPUT"

      - name: Compute the (lowercase) image name
        run: echo "IMAGE=ghcr.io/${GITHUB_REPOSITORY,,}/lendit-dotnet" >> "$GITHUB_ENV"

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Add the dev names to the tested image (no rebuild)
        run: |
          docker buildx imagetools create \
            --tag "$IMAGE:${{ steps.v.outputs.VERSION }}" \
            --tag "$IMAGE:dev" \
            "$IMAGE:$GITHUB_SHA"
          docker buildx imagetools inspect "$IMAGE:dev"
```

Copy `version.sh` into `testing-demo/.github/scripts/` as well. `docker buildx imagetools create` writes a **new tag that points at the same manifest** directly in the registry: nothing is downloaded and nothing is rebuilt.

Merge a change, and watch the pipeline. In the package page, you should find three tags with **the same digest**: `<sha>`, `1.1.0-dev.N` and `dev`. Verify:

```bash
docker buildx imagetools inspect ghcr.io/<owner>/<repo>/lendit-dotnet:dev | grep -i '^Digest'
docker buildx imagetools inspect ghcr.io/<owner>/<repo>/lendit-dotnet:<full-commit-sha> | grep -i '^Digest'
```

### 2.4 Promotion to `rc` and `release`: `release.yml`

A release is a **decision**, and the decision is a git tag. The release workflow does **not build** anything. It checks that the tagged commit has a tested image, adds the names, and publishes to the production registries.

Create `.github/workflows/release.yml`:

```yaml
name: release

on:
  push:
    tags: ["v*"]

concurrency:
  group: release-${{ github.ref }}
  cancel-in-progress: false

permissions:
  contents: read

env:
  IMAGE_NAME: lendit-dotnet

jobs:
  version:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.v.outputs.VERSION }}
      channel: ${{ steps.v.outputs.CHANNEL }}
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0
      - id: v
        run: .github/scripts/version.sh >> "$GITHUB_OUTPUT"

  # ---------------------------------------------------------------- GHCR: add the names
  promote-ghcr:
    name: promote in GHCR
    needs: version
    runs-on: ubuntu-latest
    environment: ${{ needs.version.outputs.channel == 'release' && 'production' || 'test' }}
    permissions:
      contents: read
      packages: write
    env:
      VERSION: ${{ needs.version.outputs.version }}
      CHANNEL: ${{ needs.version.outputs.channel }}
    steps:
      - name: Compute the (lowercase) image name
        run: echo "IMAGE=ghcr.io/${GITHUB_REPOSITORY,,}/${IMAGE_NAME}" >> "$GITHUB_ENV"

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: The tagged commit must already have a tested image
        run: |
          if ! docker buildx imagetools inspect "$IMAGE:$GITHUB_SHA" > /dev/null 2>&1; then
            echo "::error::No tested image for commit $GITHUB_SHA. Tag a commit that has passed the pipeline on main."
            exit 1
          fi

      - name: Promote (retag, no rebuild)
        run: |
          TAGS=("$VERSION")
          if [ "$CHANNEL" = "rc" ]; then
            TAGS+=("rc")
          elif [ "$CHANNEL" = "release" ]; then
            MAJOR="${VERSION%%.*}"
            MINOR="${VERSION#*.}"; MINOR="${MINOR%%.*}"
            TAGS+=("$MAJOR.$MINOR" "$MAJOR" "latest")
          fi
          for TAG in "${TAGS[@]}"; do
            docker buildx imagetools create --tag "$IMAGE:$TAG" "$IMAGE:$GITHUB_SHA"
          done
          docker buildx imagetools inspect "$IMAGE:$VERSION" | grep -i '^Digest'

  # ---------------------------------------------------------------- Docker Hub: releases only
  publish-dockerhub:
    name: publish to Docker Hub
    needs: [version, promote-ghcr]
    if: needs.version.outputs.channel == 'release'
    runs-on: ubuntu-latest
    environment: production                      # DOCKERHUB_TOKEN exists only in this environment
    permissions:
      contents: read
      packages: read
    env:
      VERSION: ${{ needs.version.outputs.version }}
    steps:
      - name: Compute the image names
        run: |
          echo "SOURCE=ghcr.io/${GITHUB_REPOSITORY,,}/${IMAGE_NAME}" >> "$GITHUB_ENV"
          echo "TARGET=docker.io/${{ vars.DOCKERHUB_USERNAME }}/lendit" >> "$GITHUB_ENV"

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/login-action@v3
        with:
          username: ${{ vars.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Copy the exact image from GHCR to Docker Hub
        run: |
          docker pull "$SOURCE:$VERSION"
          MAJOR="${VERSION%%.*}"
          MINOR="${VERSION#*.}"; MINOR="${MINOR%%.*}"
          for TAG in "$VERSION" "$MAJOR.$MINOR" "$MAJOR" latest; do
            docker tag "$SOURCE:$VERSION" "$TARGET:$TAG"
            docker push "$TARGET:$TAG"
          done

  # ---------------------------------------------------------------- GitHub Release
  github-release:
    name: GitHub Release
    needs: [version, promote-ghcr]
    runs-on: ubuntu-latest
    permissions:
      contents: write                            # the only job that may write to the repository
    env:
      GH_TOKEN: ${{ github.token }}
      CHANNEL: ${{ needs.version.outputs.channel }}
    steps:
      - uses: actions/checkout@v6
      - name: Create the release with generated notes
        run: |
          FLAGS=(--generate-notes)
          if [ "$CHANNEL" = "rc" ]; then FLAGS+=(--prerelease); fi
          gh release create "$GITHUB_REF_NAME" "${FLAGS[@]}"
```

Things to notice:

- The `production` environment's required reviewers must **approve** before `promote-ghcr` for a release runs. Until then, nothing is published.
- `DOCKERHUB_TOKEN` is an **environment secret** of `production` (Settings → Environments → production → *Environment secrets*), `DOCKERHUB_USERNAME` is an environment *variable*. Jobs of the `test` environment, or a workflow on a feature branch, cannot read it.
- `contents: write` is granted only to the job that needs it, and the others use the minimal `permissions`.
- The image on Docker Hub is **copied** from GHCR (`pull`, `tag`, `push`), and `imagetools create` can only create tags inside one registry. The bytes are still the same, so the digest is the same.

### 2.5 Docker Hub: the open registry

1. Create a free account at <https://hub.docker.com>, and a repository `lendit` (public).
2. Create an **access token**: Account settings → Personal access tokens, with the permission *Read & Write* (not *Delete*, and not your password!). Add it to the `production` environment as the secret `DOCKERHUB_TOKEN`, and your user name as the variable `DOCKERHUB_USERNAME`.
3. Notice the difference from GHCR: the image name is `docker.io/<user>/<repo>` (or just `<user>/<repo>`), it has a public **web page**, and anonymous **pull rate limits** apply (see the Docker documentation for the current numbers, as they depend on the plan and on whether you are logged in). A deployment that pulls many times from Docker Hub anonymously from a shared IP address can be throttled. That is a good reason to run your own mirror, or to use the cloud registry of the environment where the images run.
4. Official images and *verified publisher* badges are Docker Hub's trust signals. Your own image has none, so users must trust you. Consider signing (2.7).

### 2.6 Scan before you publish

An image can pass all functional tests and still contain a library with a known vulnerability. Scan **after the tests, before the release**:

```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy:<pinned-version> image --severity HIGH,CRITICAL --ignore-unfixed \
  ghcr.io/<owner>/<repo>/lendit-dotnet:dev
```

(Replace `<pinned-version>` with a concrete version from the Trivy releases. In a real pipeline you never use `latest` for a tool that decides whether your release is safe.) Read the report: which package, which CVE, which severity, and is there a fixed version? Try the same on the naive image from Lab 6 and on the multi-stage one, and compare the numbers.

Add the scan as a step in the `e2e` job, after the tests and **before** the push, and make it fail the build on `HIGH`/`CRITICAL` findings with a fix available:

```yaml
      - name: Scan the image (fail on fixable HIGH/CRITICAL vulnerabilities)
        run: >
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock
          aquasec/trivy:<pinned-version> image
          --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1
          "$IMAGE:$GITHUB_SHA"
```

### 2.7 Optional: provenance, SBOM and signatures

- **SBOM** (software bill of materials): the list of components inside the image. `docker buildx build --sbom=true --provenance=mode=max --push ...` attaches an SBOM and a **provenance attestation** (which workflow, which commit, which builder built it) to the image in the registry. (Attestations cannot be attached when the image is only `--load`ed into the local daemon, as in the Lab 7 workflow. Think about how you would restructure the pipeline to get both.)
- **Signing** with **cosign** (Sigstore): `cosign sign` stores a signature next to the image. With *keyless* signing in GitHub Actions (the job needs `id-token: write`), the signature proves *"this image was produced by this workflow in this repository"*, and consumers can verify it with `cosign verify`. Try it on one release.

### 2.8 Do it

1. Merge a change into `main`. Verify the `dev` tags (2.3).
2. Create the tag `v1.1.0-rc.1` on a commit that has passed the main pipeline, and push it. Verify the `rc` tags, and the pre-release in the GitHub Releases.
3. Create the tag `v1.1.0` on the same commit. Approve the `production` environment. Verify the tags `1.1.0`, `1.1`, `1`, `latest` on GHCR and Docker Hub, and that **the digest is identical** on both, and identical to the `dev` image that was tested.
4. Tag a commit that never ran on `main` (for example from a branch). The release must **fail** with the error message. Screenshot it.
5. Try to pull the `dev` image from a computer that is *not* logged in (or in a private browser window): you should be denied, while the public Docker Hub image can be pulled.

### Questions for your report

1. Why does `release.yml` not build the image? What guarantee does this give you, and what does it cost? (Think about a version number that is *inside* the application, for example in a label or in the binary: in this scheme, the image carries the version of the *dev* build. How would you handle it?)
2. Why is `docker buildx imagetools create` better than `docker pull` + `docker tag` + `docker push` inside the same registry?
3. Which secrets exist in your pipeline, where is each stored, and which jobs can read it?
4. What would an attacker with write access to a *feature branch* be able to do in this setup, and what would they not be able to do? Which setting prevents it?
5. What is the risk of floating tags (`1`, `1.1`, `latest`), and why are exact versions still available?

### Expected result of Task 2

- Working `publish-test` and `release.yml` for the Docker image, with approvals on `production`.
- Screenshots: the GHCR package with the tags, the identical digests, the approval, the failed release for an untested commit, and the Docker Hub page.
- The scan report and answers.

---

## Task 3: Publishing native packages: NuGet, Maven, npm

Libraries are consumed by **other developers' build tools**, so they are published as native packages. This task uses **GitHub Packages** as the registry (so you can use the built-in `GITHUB_TOKEN`), and you will see the same flow on the public registries in 3.5. The libraries live in `publish-demo`.

### 3.1 .NET: a NuGet package (`.dll` inside `.nupkg`)

**Package metadata** goes in `dotnet-lib/src/LendIt.Pricing/LendIt.Pricing.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>

    <PackageId>LendIt.Pricing</PackageId>
    <Version>1.0.0</Version>                       <!-- overridden by the pipeline: -p:Version=... -->
    <Authors>Your Name</Authors>
    <Description>Late fee calculation for LendIt.</Description>
    <PackageLicenseExpression>MIT</PackageLicenseExpression>
    <PackageReadmeFile>README.md</PackageReadmeFile>
    <RepositoryUrl>https://github.com/<owner>/publish-demo</RepositoryUrl>
    <RepositoryType>git</RepositoryType>
  </PropertyGroup>

  <ItemGroup>
    <None Include="README.md" Pack="true" PackagePath="\" />
  </ItemGroup>

</Project>
```

(`RepositoryUrl` connects the package to your repository on GitHub, so that the repository permissions apply to it.)

Pack and **look inside** the package. A `.nupkg` is a ZIP file:

```bash
cd dotnet-lib
dotnet pack src/LendIt.Pricing -c Release -p:Version=1.0.0-dev.1 -o out
unzip -l out/LendIt.Pricing.1.0.0-dev.1.nupkg
```

Where is the `.dll`? Where are the metadata? Is there anything in the package that should not be there?

**Publish manually** (a PAT with the `write:packages` scope, or later `GITHUB_TOKEN`):

```bash
dotnet nuget push "out/*.nupkg" \
  --source "https://nuget.pkg.github.com/<owner>/index.json" \
  --api-key "$GITHUB_PAT" --skip-duplicate
```

(`--skip-duplicate` makes a repeated publication of the same version not fail. The registry still refuses to overwrite, which is the immutability you want.)

**Consume** it from a new console project. Put a `nuget.config` **without any secret** in the consumer project, and take the token from an environment variable:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <add key="github" value="https://nuget.pkg.github.com/<owner>/index.json" />
  </packageSources>
  <packageSourceCredentials>
    <github>
      <add key="Username" value="<user>" />
      <add key="ClearTextPassword" value="%GITHUB_TOKEN%" />     <!-- PAT with read:packages -->
    </github>
  </packageSourceCredentials>
</configuration>
```

```bash
dotnet new console -n Consumer && cd Consumer      # then add the nuget.config above
dotnet add package LendIt.Pricing                  # latest STABLE version only
dotnet add package LendIt.Pricing --prerelease     # also 1.0.0-dev.N
dotnet add package LendIt.Pricing --version 1.0.0-dev.1
```

Observe: the stable command does not see the dev version. That is how NuGet separates **test** and **production** versions.

> **Dependency confusion.** If a *public* package named `LendIt.Pricing` appeared on nuget.org, a build that searches all sources might pick it up. NuGet's **package source mapping** makes it impossible, by saying which source is allowed to provide which package IDs:
>
> ```xml
> <packageSourceMapping>
>   <packageSource key="github"><package pattern="LendIt.*" /></packageSource>
>   <packageSource key="nuget.org"><package pattern="*" /></packageSource>
> </packageSourceMapping>
> ```

### 3.2 Java: a Maven artifact (`.jar`)

Maven publishing needs three things: the artifact coordinates (`groupId:artifactId:version`) and metadata in `pom.xml`, the *repository* to publish to (`distributionManagement`), and *credentials* (in `settings.xml`, never in the `pom.xml`).

**`java-lib/pom.xml`**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>hr.lendit</groupId>
  <artifactId>lendit-pricing</artifactId>
  <version>1.0.0-SNAPSHOT</version>               <!-- overridden by the pipeline -->
  <packaging>jar</packaging>

  <name>lendit-pricing</name>
  <description>Late fee calculation for LendIt.</description>
  <url>https://github.com/<owner>/publish-demo</url>
  <licenses>
    <license><name>MIT</name><url>https://opensource.org/licenses/MIT</url></license>
  </licenses>
  <scm><url>https://github.com/<owner>/publish-demo</url></scm>

  <properties>
    <maven.compiler.release>21</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter</artifactId>
      <version>5.11.4</version>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>                                    <!-- publish the sources next to the jar -->
        <artifactId>maven-source-plugin</artifactId>
        <version>3.3.1</version>
        <executions>
          <execution><goals><goal>jar-no-fork</goal></goals></execution>
        </executions>
      </plugin>
    </plugins>
  </build>

  <distributionManagement>
    <repository>
      <id>github</id>                             <!-- must match the <server><id> in settings.xml -->
      <name>GitHub Packages</name>
      <url>https://maven.pkg.github.com/<owner>/publish-demo</url>
    </repository>
  </distributionManagement>
</project>
```

**Credentials: `~/.m2/settings.xml`** (this file is on *your* computer and is **not** committed):

```xml
<settings>
  <servers>
    <server>
      <id>github</id>
      <username>${env.GITHUB_ACTOR}</username>
      <password>${env.GITHUB_TOKEN}</password>
    </server>
  </servers>
</settings>
```

Set the environment variables (a PAT in `GITHUB_TOKEN`), build, inspect and publish:

```bash
cd java-lib
./mvnw -B versions:set -DnewVersion=1.0.0-dev.1 -DgenerateBackupPoms=false
./mvnw -B verify
unzip -l target/lendit-pricing-1.0.0-dev.1.jar
./mvnw -B deploy
```

`mvn deploy` runs the whole lifecycle (Lab 6: `validate → … → verify → install → deploy`) and then uploads the `.jar`, the sources `.jar` and the `.pom`.

**Consume** it in another Maven project (in `pom.xml`, plus the same `settings.xml` for the credentials):

```xml
<repositories>
  <repository>
    <id>github</id>
    <url>https://maven.pkg.github.com/<owner>/publish-demo</url>
  </repository>
</repositories>

<dependencies>
  <dependency>
    <groupId>hr.lendit</groupId>
    <artifactId>lendit-pricing</artifactId>
    <version>1.0.0-dev.1</version>
  </dependency>
</dependencies>
```

> Unlike npm and NuGet, Maven has no "give me the newest stable version" in the dependency declaration. **You always state the exact version**, which is good for reproducibility. The ranges and `LATEST` are discouraged.

### 3.3 JavaScript: an npm package

**`js-lib/package.json`**

```json
{
  "name": "@<owner>/lendit-pricing",
  "version": "1.0.0",
  "description": "Late fee calculation for LendIt.",
  "main": "index.js",
  "files": ["index.js", "README.md"],
  "scripts": { "test": "node --test" },
  "repository": {
    "type": "git",
    "url": "git+https://github.com/<owner>/publish-demo.git",
    "directory": "js-lib"
  },
  "license": "MIT"
}
```

The package name must be **scoped** (`@<owner>/…`, lowercase) on GitHub Packages. The `"files"` list is a **whitelist** of what goes into the package. See what would be published before you publish:

```bash
cd js-lib
npm test
npm pack --dry-run
```

Is anything in the list that should not be public (tests, `.env`, `.npmrc`)? Without `"files"` or an `.npmignore`, npm publishes nearly everything in the folder, and leaked tokens in published packages are a real and common incident.

**Registry configuration: `js-lib/.npmrc`** (committed **without a secret**; the token comes from an environment variable):

```text
@<owner>:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
```

```bash
export GITHUB_TOKEN=<PAT with write:packages>
npm version 1.0.0-dev.1 --no-git-tag-version
npm publish --tag dev
```

**dist-tags** are the npm way of separating channels. `npm install @<owner>/lendit-pricing` installs the version that the tag `latest` points at. A **pre-release must never be published as `latest`**. Always give a pre-release an explicit `--tag`, otherwise everyone who installs the package without a version gets your unfinished build (newer npm versions refuse such a publish, but do not rely on that).

| Channel | Publish | Install |
|---------|---------|---------|
| dev | `npm publish --tag dev` | `npm install @<owner>/lendit-pricing@dev` |
| rc | `npm publish --tag next` | `npm install @<owner>/lendit-pricing@next` |
| release | `npm publish` (tag `latest`) | `npm install @<owner>/lendit-pricing` |

```bash
npm view @<owner>/lendit-pricing dist-tags versions
```

### 3.4 One pipeline for the three packages

Create `.github/workflows/publish-packages.yml` in `publish-demo`. The version and the channel come from `version.sh` (Task 1), the test jobs run **before** publishing, and test versus production is a matter of **environment and version**, not of different code.

```yaml
name: publish-packages

on:
  push:
    branches: [main]
    tags: ["v*"]
  workflow_dispatch:

permissions:
  contents: read

jobs:
  version:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.v.outputs.VERSION }}
      channel: ${{ steps.v.outputs.CHANNEL }}
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0
      - id: v
        run: .github/scripts/version.sh >> "$GITHUB_OUTPUT"

  # ---------------------------------------------------------------- NuGet
  nuget:
    needs: version
    runs-on: ubuntu-latest
    environment: ${{ needs.version.outputs.channel == 'release' && 'production' || 'test' }}
    permissions:
      contents: read
      packages: write
    env:
      VERSION: ${{ needs.version.outputs.version }}
      NUGET_KEY: ${{ secrets.GITHUB_TOKEN }}
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-dotnet@v5
        with:
          dotnet-version: 10.0.x

      - name: Test
        run: dotnet test dotnet-lib/tests/LendIt.Pricing.Tests -c Release

      - name: Pack with the computed version
        run: dotnet pack dotnet-lib/src/LendIt.Pricing -c Release -p:Version="$VERSION" -o out

      - name: Publish to GitHub Packages
        run: >
          dotnet nuget push "out/*.nupkg"
          --source "https://nuget.pkg.github.com/${{ github.repository_owner }}/index.json"
          --api-key "$NUGET_KEY" --skip-duplicate

  # ---------------------------------------------------------------- Maven
  maven:
    needs: version
    runs-on: ubuntu-latest
    environment: ${{ needs.version.outputs.channel == 'release' && 'production' || 'test' }}
    permissions:
      contents: read
      packages: write
    env:
      VERSION: ${{ needs.version.outputs.version }}
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}     # read by settings.xml, written by setup-java below
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-java@v5
        with:
          distribution: temurin
          java-version: 21
          cache: maven
          cache-dependency-path: java-lib/pom.xml
          server-id: github                          # generates ~/.m2/settings.xml for the <server> with this id
          server-username: GITHUB_ACTOR
          server-password: GITHUB_TOKEN

      - name: Set the version, test and publish
        working-directory: java-lib
        run: |
          ./mvnw -B -ntp versions:set -DnewVersion="$VERSION" -DgenerateBackupPoms=false
          ./mvnw -B -ntp deploy

  # ---------------------------------------------------------------- npm
  npm:
    needs: version
    runs-on: ubuntu-latest
    environment: ${{ needs.version.outputs.channel == 'release' && 'production' || 'test' }}
    permissions:
      contents: read
      packages: write
    env:
      VERSION: ${{ needs.version.outputs.version }}
      CHANNEL: ${{ needs.version.outputs.channel }}
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v4
        with:
          node-version: lts/*
          registry-url: https://npm.pkg.github.com
          scope: "@${{ github.repository_owner }}"

      - name: Set the version and test
        working-directory: js-lib
        run: |
          npm version "$VERSION" --no-git-tag-version
          npm test

      - name: Publish with the dist-tag of the channel
        working-directory: js-lib
        env:
          NODE_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          case "$CHANNEL" in
            dev) TAG=dev ;;
            rc)  TAG=next ;;
            *)   TAG=latest ;;
          esac
          npm publish --tag "$TAG"
```

> The `scope` and the package `name` must use the **lowercase** GitHub owner name. If your GitHub user name has capital letters, the scope is still lowercase.

Run it in the three ways of Task 2.8 (push to `main`, tag `v1.1.0-rc.1`, tag `v1.1.0`), then check the **Packages** page of the repository, and use the commands from 3.1 to 3.3 to install the dev, rc and release versions from a consumer project.

> **Packages are rebuilt for each version**, while images are *promoted*. Why is that unavoidable for packages? (Where is the version number stored in a `.nupkg` / `.jar` / npm tarball?)

### 3.5 The public registries: Maven Central, nuget.org, npmjs.com

Everything above works the same for the large public registries, with these differences. **Do not publish your lab libraries there.**

| | Maven Central | nuget.org | npmjs.com |
|---|---|---|---|
| Account / access | Sonatype Central Portal account, **verified namespace** (domain or GitHub) | account + **API key** (scoped to package IDs, with expiry) | account with **2FA** + token |
| Signing | **required** (GPG signatures on all files) | optional (author signing) | optional; `--provenance` attaches a build attestation |
| Metadata required | name, description, URL, license, SCM, developers | ID, version, authors, description, license | name, version |
| Delete a published version? | **no** | no (only *unlist*) | restricted (time and usage conditions) |
| Anonymous download | yes | yes | yes |

Consequence: **only release versions** go to public registries, and only from the production channel of a pipeline. A secret or a bug published to them cannot be taken back. You can only publish a fixed version and revoke the secret.

### Questions for your report

1. Which fields of the three package formats contain the version, and where does the pipeline set it?
2. How do consumers *opt in* to the test versions in NuGet, Maven and npm? Which of the three has the most dangerous default, and why?
3. Why does GitHub Packages require authentication even to read a public Maven/NuGet/npm package? What is the cost of that for open-source consumers, and which public registry would you use instead?
4. What does `files` in `package.json` protect against?
5. What is *dependency confusion*, and which setting of which tool prevents it in NuGet (3.1)? What would be the equivalent in npm (hint: scopes; see Task 4)?

### Expected result of Task 3

- Three libraries published to GitHub Packages with a `dev`, an `rc` and a `release` version each, by the pipeline (screenshots of the package pages and of `npm view ... dist-tags`).
- Consumer projects (one per ecosystem is enough for two of them) installing the dev and the stable version.
- Answers to the questions.

---

## Task 4: A self-hosted registry and its security

A self-hosted registry gives you full control over data and access, and full responsibility for running it securely. You will run two: a **container registry** (the open-source Docker `registry`, also called *Distribution*) and an **npm registry** (**Verdaccio**).

### 4.1 Start a registry: the dangerous way

```bash
docker run -d --name registry-insecure -p 5000:5000 registry:3
docker pull alpine:3.20
docker tag alpine:3.20 localhost:5000/demo/alpine:1.0
docker push localhost:5000/demo/alpine:1.0

curl http://localhost:5000/v2/_catalog
curl http://localhost:5000/v2/demo/alpine/tags/list
```

It works immediately: no login, no TLS. **Anybody who can reach the port can push, overwrite and read all images.** (`localhost` is the only host name that Docker allows over plain HTTP.) This is what a registry looks like when somebody follows a five-minute tutorial and leaves it running. Remove it: `docker rm -f registry-insecure`.

(The image tag `registry:3` is the current Distribution release. `registry:2` accepts the same configuration variables used below.)

### 4.2 Add TLS and authentication

1. **A certificate.** TLS encrypts the traffic and proves the server's identity. For the lab, use a self-signed certificate that is valid for `localhost`:

```bash
mkdir -p registry/{auth,certs,data} && cd registry

openssl req -x509 -newkey rsa:4096 -sha256 -days 365 -nodes \
  -keyout certs/registry.key -out certs/registry.crt \
  -subj "/CN=localhost" -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"
```

2. **Users.** The registry's `htpasswd` authentication reads a file of users with **bcrypt**-hashed passwords (`-B`). Create two users: one for CI that pushes, one for a server that only pulls.

```bash
docker run --rm -it --entrypoint htpasswd httpd:2 -Bc /dev/stdout ci-user      > auth/htpasswd
docker run --rm -it --entrypoint htpasswd httpd:2 -B  /dev/stdout deploy-user  >> auth/htpasswd
```

(If the second command overwrites instead of appending on your system, create the second user with `-Bbn deploy-user '<password>'` and append the output with `>>`. Never put real passwords on a command line you will keep in your shell history. Use throw-away passwords for the lab.)

3. **Run the registry** with TLS and authentication turned on:

```bash
docker run -d --name registry --restart=unless-stopped -p 5443:5443 \
  -v "$PWD/data:/var/lib/registry" \
  -v "$PWD/certs:/certs:ro" \
  -v "$PWD/auth:/auth:ro" \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:5443 \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/registry.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/registry.key \
  -e REGISTRY_AUTH=htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_REALM="LendIt registry" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  -e REGISTRY_STORAGE_DELETE_ENABLED=true \
  registry:3
```

(In PowerShell, use `${PWD}` instead of `$PWD`.)

4. **Make Docker trust your certificate.** The Docker daemon does not trust self-signed certificates until you tell it to:

   - **Linux:** `sudo mkdir -p /etc/docker/certs.d/localhost:5443 && sudo cp certs/registry.crt /etc/docker/certs.d/localhost:5443/ca.crt` (no restart needed).
   - **Docker Desktop (Windows/macOS):** add `registry.crt` to the operating system's trusted root certificates, and restart Docker Desktop (see the Docker documentation for your system).

### 4.3 Test the access control

```bash
docker tag alpine:3.20 localhost:5443/demo/alpine:1.0

docker push localhost:5443/demo/alpine:1.0
# → "no basic auth credentials"                       (not logged in)

docker login localhost:5443 -u ci-user
docker push localhost:5443/demo/alpine:1.0            # works

docker logout localhost:5443
docker login localhost:5443 -u deploy-user
docker pull localhost:5443/demo/alpine:1.0            # works

curl --cacert certs/registry.crt -u deploy-user https://localhost:5443/v2/_catalog     # prompts for the password
curl http://localhost:5443/v2/                         # plain HTTP to a TLS port → refused
```

Notice that the password is **prompted** (`-u deploy-user` without `:password`), so it does not end up in the shell history or in the process list.

Now the uncomfortable result. Try to **push as `deploy-user`**:

```bash
docker push localhost:5443/demo/alpine:2.0            # as deploy-user
```

It works, although `deploy-user` was meant to be read-only. **`htpasswd` provides authentication, not authorization**: every valid user may do everything with every repository. Answer in your report:

1. Why is this a problem for a production server that holds the credentials of `deploy-user`?
2. What are three ways to get real authorization? (Options: a **token service** with the registry's token authentication; a **reverse proxy** such as nginx that allows `GET`/`HEAD` for one user group and `POST`/`PUT`/`PATCH`/`DELETE` for another; or a registry product with built-in roles such as **Harbor**, **Nexus**, **Artifactory** or **GitLab**, which add projects, role-based access, robot accounts, scanning and replication.)

### 4.4 Using it from a pipeline

A GitHub-hosted runner cannot reach `localhost:5443` on your laptop. A pipeline that publishes to a self-hosted registry needs:

- the registry reachable from the runner (a public address with TLS and a certificate from a **trusted CA**, e.g. Let's Encrypt, or a **self-hosted runner** inside your network);
- credentials stored as **secrets** (`REGISTRY_USER`, `REGISTRY_PASSWORD`), not in the workflow, and a dedicated **push-only** account for CI;
- the step `docker/login-action` with `registry: registry.example.com`.

Write (without running it) the `login` and `push` steps that would publish the tested image to `registry.example.com/lendit/dotnet`, and answer: why must a **self-hosted runner never run workflows from pull requests of public repositories**? (Hint: who can change the workflow file, and what can the runner reach?)

### 4.5 Deleting and cleaning up

Registries grow. With `REGISTRY_STORAGE_DELETE_ENABLED=true`, a manifest can be deleted through the API (the digest comes from the `Docker-Content-Digest` response header), but the disk space is released only after the registry's **garbage collector** has run. Read about it in the Distribution documentation, and write down your clean-up policy for the test channel (for example, "keep the last 20 dev versions and everything newer than 14 days"). Retention is a security topic as well: an old image with known vulnerabilities, kept and pullable, is a liability.

### 4.6 A self-hosted npm registry (Verdaccio)

```bash
mkdir -p verdaccio/conf verdaccio/storage
cat > verdaccio/conf/config.yaml <<'EOF'
storage: /verdaccio/storage
auth:
  htpasswd:
    file: /verdaccio/conf/htpasswd
    max_users: 1000            # we will change this to -1 (no self-registration) below
uplinks:
  npmjs:
    url: https://registry.npmjs.org/
packages:
  '@<owner>/*':                # OUR packages: login required, and NEVER looked up on npmjs.org
    access: $authenticated
    publish: $authenticated
    unpublish: $authenticated
  '**':                        # everything else: anonymous read, proxied (and cached) from npmjs.org
    access: $all
    publish: $authenticated
    unpublish: $authenticated
    proxy: npmjs
log: { type: stdout, format: pretty, level: http }
EOF

# Verdaccio runs as user 10001; it must be able to write
sudo chown -R 10001:65533 verdaccio

docker run -d --name verdaccio -p 4873:4873 \
  -v "$PWD/verdaccio/conf:/verdaccio/conf" \
  -v "$PWD/verdaccio/storage:/verdaccio/storage" \
  verdaccio/verdaccio
```

Create a user and publish the library from Task 3 to **your own** registry, using the scope mapping in `.npmrc`:

```bash
npm adduser --registry http://localhost:4873 --auth-type=legacy

cd js-lib
# a temporary .npmrc that points the scope to Verdaccio (do not commit it)
echo "@<owner>:registry=http://localhost:4873/" > .npmrc.verdaccio
npm version 1.0.0-rc.1 --no-git-tag-version
npm publish --userconfig .npmrc.verdaccio --tag next 2>&1 | tail -5     # or use --registry http://localhost:4873

npm dist-tag ls @<owner>/lendit-pricing --registry http://localhost:4873
npm dist-tag add @<owner>/lendit-pricing@1.0.0-rc.1 latest --registry http://localhost:4873     # a promotion!
```

Then lock the registry down so nobody else can register: change `max_users: 1000` to `max_users: -1`, `docker restart verdaccio`, and verify that `npm adduser` with a new name is refused.

Observe two security properties in `config.yaml`:

- **Proxy and cache.** Packages that are not yours are fetched from npmjs.org through your registry and cached. That is faster, works when npmjs.org is down, and gives you one controlled place to block bad packages.
- **No `proxy:` for your scope.** A request for `@<owner>/anything` is never forwarded to the public registry. That prevents **dependency confusion**: an attacker publishing a public package with the same name as your internal one, in the hope that your build resolves the public (malicious) one. Scoped names plus "this scope comes only from our registry" are the standard defence. Test it: request `npm view @<owner>/does-not-exist --registry http://localhost:4873` and read the answer.

> Verdaccio on `http://localhost` sends the token without encryption, which is acceptable only on your own computer. Anything reachable over a network needs TLS (a reverse proxy in front of it).

**Optional:** **Sonatype Nexus Repository** (`docker run -d -p 8081:8081 sonatype/nexus3`; the first admin password is in `/nexus-data/admin.password` inside the container, and it needs several GB of memory) hosts **Maven, NuGet, npm and Docker** repositories in one place, with roles and privileges, and offers *proxy* repositories (a cache of Maven Central, nuget.org, npmjs) and *group* repositories (one URL that combines your hosted repository and the proxies, in a controlled order). Try to publish the Maven library from Task 3 to a hosted repository in Nexus by changing `<distributionManagement>` and the `<server>` in `settings.xml`.

### Expected result of Task 4

- A running TLS-protected, password-protected container registry, with the evidence for each access-control experiment (push without login, with login, the successful push by the read-only user).
- A Verdaccio registry with a published package, a dist-tag promotion, closed registration and the scope protection.
- Answers to the questions in 4.3, 4.4 and the clean-up policy.

---

## Task 5: Publishing in your own project

Apply what you learned to the **project repository** from Lab 1.

1. **Decide the artifacts.** Which artifacts does your project produce (a Docker image, a library, a web package)? For each one, choose the registry, with a justification.
2. **Versioning.** Create `docs/versioning.md` (Task 1.1): scheme, channels, tag format, mutable and immutable names, and who may create a release. Add the `version.sh` script (or another tool) to the repository.
3. **Pipeline.** Extend the pipeline from Lab 7 with the `publish` stage: **dev** publishing from `main`, and **rc/release** promotion from tags. Publish only after the tests pass, and never from pull requests. Build once, and promote the same artifact if your artifact type allows it.
4. **Security.** Create the `test` and `production` environments, put the registry secrets in the environments, set `permissions:` to the minimum, and use `GITHUB_TOKEN` or OIDC wherever the registry supports it.
5. **Release.** Create a `v0.1.0-rc.1` and then a `v0.1.0` release, and show it in **GitHub Releases** with generated release notes.
6. **Document it** in `docs/publishing.md`: the artifacts and registries, the channels and tags, the pipeline stages, the credentials and where each is stored, **how to consume** a test and a production version (the exact commands), **how to revoke a leaked credential**, and **how to withdraw a faulty release** (for example by publishing a fixed `PATCH` version and marking the old one as deprecated or deleting it where the registry allows).
7. **Process.** Work as in the earlier labs: issues in the GitHub Project, a branch per issue, pull requests with `Closes #<n>`.

### Expected result of Task 5

- A tagged release of your project, with the artifact in a registry.
- `docs/versioning.md`, `docs/publishing.md`, and a green pipeline with a `publish` stage.

---

## Troubleshooting

| Symptom | Likely cause and fix |
|---------|----------------------|
| `docker push` to GHCR: `denied` or `permission_denied: write_package` | The job lacks `packages: write`, the package belongs to another owner/repository (see **Manage Actions access** in the package settings), or the name is not lowercase |
| `docker buildx imagetools create` says `not found` | The source tag was never pushed: the tested image (`:<sha>`) exists only if the `e2e` job ran on `main` and passed |
| Release fails with *No tested image for commit* | You tagged a commit that never went through the main pipeline. That is the intended safeguard. Tag a commit that did |
| `version.sh`: `Permission denied` | `git update-index --chmod=+x .github/scripts/version.sh` and commit |
| Version script returns `0.1.0-dev.N` although tags exist | The checkout was shallow. Use `fetch-depth: 0` |
| Environment job waits forever | A required reviewer has to approve it (Actions run page → *Review deployments*). A job also cannot start if the tag is not allowed to deploy to that environment |
| NuGet: `401 Unauthorized` on push | Wrong token or scope (`write:packages`), or the package's `RepositoryUrl` points to a repository you cannot write to |
| NuGet: `409 Conflict` | The version already exists. Versions are immutable. Increase it, or use `--skip-duplicate` |
| Maven: `401` or `Could not transfer artifact` | The `<id>` in `distributionManagement` does not match the `<server><id>`, or the environment variables are empty |
| Maven: `Return code is: 409` | That version has already been deployed. Use a new version |
| npm: `404 Not Found - PUT ...` or `E403` | The package name must be scoped (`@owner/name`) with the exact lowercase owner, and the registry must be mapped for that scope |
| npm: a pre-release became `latest` | It was published without `--tag`. Fix it with `npm dist-tag add <pkg>@<good-version> latest` |
| Verdaccio: `EACCES` on start | The mounted folders must be writable by UID 10001: `sudo chown -R 10001:65533 verdaccio` |
| `docker login localhost:5443`: `x509: certificate signed by unknown authority` | The daemon does not trust the self-signed certificate (see 4.2, step 4) |
| Docker Hub: `toomanyrequests` | The anonymous pull limit was reached. Log in, or use a mirror or another registry |

---

## Deliverables

- Repositories `testing-demo` and `publish-demo` (public), with the workflows `release.yml` and `publish-packages.yml`, the changed Lab 7 pipeline, `version.sh` and `docs/versioning.md`.
- A report `docs/publishing-lab.md` in your **project** repository: answers to the questions of every task, the screenshots (package pages, identical digests, approval, failed release for an untested commit).
- The artifacts of your own project in a registry, with `docs/versioning.md` and `docs/publishing.md` in your project repository.

Final checklist:

- [ ] Versioning rules written; `version.sh` produces `dev`, `rc`, `release` and rejects bad tags
- [ ] The Docker image is built once and **promoted** (same digest) through `dev`, `rc` and `release` in GHCR
- [ ] Docker Hub receives release images only; the token lives in the `production` environment
- [ ] Image scan fails the build on fixable HIGH/CRITICAL findings
- [ ] NuGet, Maven and npm packages published by a pipeline with correct pre-release handling (`--prerelease`, dist-tags)
- [ ] A TLS- and password-protected self-hosted registry; the limits of `htpasswd` documented
- [ ] A self-hosted npm registry with closed registration and scope protection against dependency confusion
- [ ] Your project publishes a versioned release to a registry, and the process is documented

---

## Command cheat sheet

| Goal | Command |
|------|---------|
| Log in to a registry (token from stdin) | `echo "$TOKEN" \| docker login <registry> -u <user> --password-stdin` |
| Add a name to an image in the same registry (no rebuild) | `docker buildx imagetools create --tag <registry>/<repo>:<new> <registry>/<repo>:<existing>` |
| Show the digest of a tag | `docker buildx imagetools inspect <image>:<tag>` |
| Copy an image to another registry | `docker pull <src>` → `docker tag <src> <dst>` → `docker push <dst>` |
| NuGet: pack / push | `dotnet pack -c Release -p:Version=<v> -o out` / `dotnet nuget push "out/*.nupkg" --source <url> --api-key <key>` |
| Maven: set the version / publish | `./mvnw versions:set -DnewVersion=<v> -DgenerateBackupPoms=false` / `./mvnw deploy` |
| npm: set the version / publish with a dist-tag | `npm version <v> --no-git-tag-version` / `npm publish --tag <tag>` |
| npm: show / move a dist-tag | `npm dist-tag ls <pkg>` / `npm dist-tag add <pkg>@<v> <tag>` |
| Registry API (Docker `registry`) | `curl --cacert ca.crt -u <user> https://host:5443/v2/_catalog` |
| Scan an image | `docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:<version> image <image>` |
| Create a GitHub Release | `gh release create <tag> --generate-notes [--prerelease]` |

---

## Useful resources

- [GitHub Docs – Working with the Container registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- [GitHub Docs – GitHub Packages: Maven, NuGet and npm registries](https://docs.github.com/en/packages)
- [GitHub Docs – Managing environments for deployment](https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-deployments/managing-environments-for-deployment)
- [GitHub Docs – Security hardening with OpenID Connect (AWS)](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
- [GitHub Docs – Security hardening for GitHub Actions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions)
- [Docker Docs – Docker Hub access tokens and download rate limits](https://docs.docker.com/docker-hub/)
- [Docker Docs – `docker buildx imagetools`](https://docs.docker.com/reference/cli/docker/buildx/imagetools/)
- [Distribution (Docker registry) documentation](https://distribution.github.io/distribution/)
- [Verdaccio documentation](https://verdaccio.org/docs/what-is-verdaccio)
- [Semantic Versioning 2.0.0](https://semver.org)
- [NuGet – package source mapping](https://learn.microsoft.com/en-us/nuget/consume-packages/package-source-mapping)
- [Maven – deploying to a repository](https://maven.apache.org/guides/mini/guide-deployment-security-settings.html)
- [npm – `dist-tag`](https://docs.npmjs.com/cli/commands/npm-dist-tag)
- [Sigstore cosign](https://docs.sigstore.dev/) and [Trivy](https://trivy.dev/)
- [Amazon ECR User Guide](https://docs.aws.amazon.com/AmazonECR/latest/userguide/)
- [AWS CodeArtifact User Guide](https://docs.aws.amazon.com/codeartifact/latest/ug/)
