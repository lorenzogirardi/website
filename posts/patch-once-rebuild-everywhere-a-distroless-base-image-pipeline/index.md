# Patch Once, Rebuild Everywhere: A Distroless Base Image Pipeline

### Table of Contents

- The Problem: N containers, one CVE
- What I Had
- Naaa... the alternatives
- The Idea: Layers That Patch Themselves
- Building a Distroless Base Without Google
- The Single Source of Versions
- The Cascade
- Trivy as the Gate
- Who Keeps It Alive: Renovate and the Agents
- The Test: pytbak on the New Base
- What Broke on the Way
- Security Considerations
- Monitoring and Observability
- Conclusion
- Reflections

Here we are.

Last week I scanned `pytbak`, my debug/test API, with Trivy. The image was `lgirardi/pytbak:v2.0.425`, built on `python:3.15.2-slim`, and the report said **444 vulnerabilities, 4 of them CRITICAL**. The application code had nothing to do with it. Almost all of it was the operating system sitting underneath: 436 findings in Debian packages that the app never calls, plus 8 in Python dependencies.

Now multiply that by the number of containers you run. I have a handful in the homelab. A company has hundreds. Every one of them carries its own copy of the same glibc, the same OpenSSL, the same tzdata, and every one of them needs a rebuild the day one of those gets a CVE. That is the real problem of this post: not "how do I make one image smaller", but **how do I patch the operating system of many containers without touching each one by hand**.

## The Problem: N containers, one CVE

Think about what happens when OpenSSL publishes a fix.

1. Debian ships the patched package.
2. Nothing happens to your running containers. They still have the old copy baked in.
3. Somebody has to rebuild every image, and every rebuild has to pick up the new package.

If each service has its own Dockerfile with `FROM python:3.15.2-slim` and its own `apt-get install`, step 3 is N separate decisions. Some will be forgotten. Some will be rebuilt from a cached layer that still has the old package. And you find out which ones when the next scan runs.

The fix is not a better scanner. The fix is to **move the operating system out of the applications** into one place that is rebuilt on a schedule, and let the applications inherit from it.

## What I Had

I already had a repository for this, `ACT-lg-baseimg`, with a `base/` image on `debian:bookworm-slim` and a `flask/` image on top of it. Built every Sunday. The idea was right. The execution had aged:

- `python3=3.11.2-1+b1`, pinned through apt: an old Python, and an exact apt pin that breaks the build the day Debian updates that package.
- A missing `&&` in a `RUN`, so the line `rm /usr/lib/python3.11/EXTERNALLY-MANAGED` ended up as an argument of the previous `rm -rf` and never ran.
- One mutable tag, `:edge`. No dated tags, so no rollback.
- `procps`, `lsof`, `net-tools` in the base: handy for debugging, but they are attack surface in production.
- A workflow with an invalid `run:` input and a digest output that was always empty.

Still a shell, still `apt`, still `dpkg`, in every container that inherited from it.

## Naaa... the alternatives

- **Keep the Debian base, just patch it more often.** Better, but the image still has a shell and a package manager. Most of the 436 OS findings were in packages the app never touches.
- **Alpine.** Smaller, but musl instead of glibc. Several Python wheels are missing or have to be compiled. I did not want to trade CVEs for build surprises.
- **Google distroless or Chainguard.** Good images, but the Python version is whatever they ship, and I want **the exact version I chose**. Pinned versions on Chainguard are a paid feature, the free tier only gives `:latest`.
- **Build my own distroless layers.** More work once, full control after.

Naaaa on the first three. Enter the fourth.

## The Idea: Layers That Patch Themselves

Distroless means an image with no shell, no package manager, only what the program needs to run. If it isn't there, it can't break. And it can't be exploited either.

The repository now looks like this:

```
ACT-lg-baseimg/
├── versions.yaml            # the ONLY place a version is written
├── images/
│   ├── base/Dockerfile      # lg-static and lg-base targets
│   ├── python/Dockerfile    # lg-python on top of lg-base
│   ├── node/Dockerfile      # lg-node on top of lg-base
│   ├── go/Dockerfile        # lg-go: build-stage toolchain
│   └── mkrootfs.sh          # root filesystem from Debian packages
├── scripts/                 # build.sh, test-images.sh, versions.py
└── legacy/                  # the old images, still rebuilt weekly
```

| Image | What it is | Size |
|---|---|---|
| `lg-static` | CA certificates, tzdata, nonroot user. No libc | 1.5 MB |
| `lg-base` | `lg-static` + glibc, libstdc++, OpenSSL | 17 MB |
| `lg-python` | CPython 3.15.2 on `lg-base` | 54 MB |
| `lg-node` | Node.js 24.22.3 on `lg-base` | 141 MB |
| `lg-go` | Go 1.27.4, build stage only | 479 MB |

Go needs no runtime image: a program built with `CGO_ENABLED=0` runs straight on `lg-static`. The 479 MB toolchain stays in the build stage and never ships.

The point is the arrows between them, not the sizes:

{{< mermaid >}}
flowchart TD
    A[Debian packages updated] --> B[lg-base rebuilt]
    B --> C[lg-python]
    B --> D[lg-node]
    B --> E[lg-static]
    E --> F[lg-go builds run here]
    C --> G[pytbak and every Python service]
    D --> H[every Node service]
{{< /mermaid >}}

## Building a Distroless Base Without Google

The trick is that a distroless root filesystem is nothing magical: it is the files that a few Debian packages install, copied onto `scratch`.

`images/mkrootfs.sh` takes a list of packages, installs them in a throwaway Debian container, and copies **only the files each package owns** into `/rootfs`:

```bash
for pkg in "$@"; do
  dpkg -L "$pkg" | while IFS= read -r path; do
    [ -f "$path" ] || [ -L "$path" ] || continue    # directories are created on demand
    dir=$(readlink -f "$(dirname "$path")")          # resolve the usr-merge symlinks
    mkdir -p "$R$dir"
    cp -a "$path" "$R$dir/"
  done
  dpkg-query -s "$pkg" > "$R/var/lib/dpkg/status.d/$pkg"
done
```

Two details matter:

1. **`status.d` records.** Each package leaves its control file in `/var/lib/dpkg/status.d/`, the convention distroless uses. Without them an image has no package database and Trivy sees nothing, which looks great in the report and means nothing. With them, the scanner lists what is really inside.
2. **Pruning.** glibc ships about 200 character-set converters, 20 MB that no web service uses. I keep the Unicode ones and two Western ones. That alone took `lg-base` from 35 MB to 17 MB.

The two targets of `images/base/Dockerfile` differ only by the package list:

```dockerfile
FROM debian:${DEBIAN_TAG} AS rootfs-base
COPY images/mkrootfs.sh /usr/local/bin/mkrootfs.sh
RUN mkrootfs.sh base-files ca-certificates tzdata netbase \
                libc6 libgcc-s1 libstdc++6 zlib1g libssl3t64 libzstd1

FROM scratch AS base
COPY --from=rootfs-base /rootfs/ /
ENV SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt
USER 65532:65532
WORKDIR /home/nonroot
```

`lg-python` does not use Debian's Python. It copies the **official CPython build** of the pinned version, so the interpreter is exactly `3.15.2`, and strips what a runtime never needs: pip, the test suites, idle, ensurepip.

## The Single Source of Versions

This is the part that answers "the versions must be pinned and chosen". One file, no exceptions:

```yaml
# versions.yaml
# renovate: datasource=docker depName=debian versioning=regex:^trixie-(?<major>\d{8})-slim$
debian: trixie-20261104-slim

# renovate: datasource=docker depName=python versioning=docker
python: 3.15.2

# renovate: datasource=docker depName=node versioning=node
node: 24.22.3

# renovate: datasource=docker depName=golang versioning=docker
go: 1.27.4
```

Dockerfiles receive these as build args, workflows read them through a tiny `scripts/versions.py`. A unit test fails if a runtime is not an exact `major.minor.patch`, if the Debian tag is not a dated snapshot, or if a line has no `# renovate:` comment above it. That last test sounds pedantic until you remember that a version Renovate cannot see is a version that rots in silence.

Mind the distinction, because it is the whole design:

- **Pinned** (changes only through a pull request): the Debian snapshot, Python, Node, Go.
- **Floating on purpose** (changes on every build): the Debian *packages* on top of that snapshot, so glibc, OpenSSL, tzdata, the CA bundle. That is what the weekly rebuild is for.

## The Cascade

Now the flow I care about. Whenever the base is rebuilt, everything on top of it is rebuilt, **on the exact base that was just published**.

{{< mermaid >}}
flowchart TD
    A[Sunday 03:00 UTC] --> B
    C[push touching images or versions.yaml] --> B
    D[manual dispatch] --> B
    B[build lg-static and lg-base] --> S{Trivy gate}
    S -->|pass| P[push amd64 and arm64]
    S -->|CRITICAL with fix| X[stop, nothing published]
    P -->|base digest| PY[lg-python]
    P -->|base digest| ND[lg-node]
    P --> GO[lg-go]
{{< /mermaid >}}

The runtimes start from the **digest** of the base, not from `:edge`:

```yaml
build-args: |
  BASE_IMAGE=ghcr.io/lorenzogirardi/lg-base@${{ needs.base.outputs.base_digest }}
  PYTHON_VERSION=${{ needs.versions.outputs.python_version }}
```

A tag moves. A digest does not. So every `lg-python` is tied to one precise base, the build is reproducible, and if the base fails its scan, the runtimes simply never start.

Tags make the rollback trivial:

| Tag | Moves? | Use it for |
|---|---|---|
| `lg-python:3.15` | with every patch | development |
| `lg-python:3.15.2` | with every weekly rebuild | staging |
| `lg-python:3.15.2-20261121` | never | **production** |

In production I pin the dated tag, or the digest. Adopting a new base becomes my decision, and going back is a one-line change.

## Trivy as the Gate

Every image is built once for amd64, scanned, and only if the scan passes is it built for amd64 and arm64 and pushed with SBOM and provenance. The gate is a composite action, used by every image:

```yaml
- name: Trivy gate (CRITICAL with a fix available blocks the publish)
  uses: aquasecurity/trivy-action@ed142fd0673e97e23eac54620cfb913e5ce36c25 # v0.36.0
  with:
    image-ref: scan/${{ inputs.name }}:ci
    scanners: vuln
    severity: CRITICAL
    ignore-unfixed: true
    exit-code: '1'
```

Why `ignore-unfixed`? Because a gate that blocks on something nobody can fix is a gate people learn to bypass. A vulnerability with no patch cannot be solved by rebuilding. A vulnerability **with** a patch can, so that one stops the line. Everything else, every severity, goes into the run summary as awareness.

## Who Keeps It Alive: Renovate and the Agents

A repository that needs a human to remember it will be forgotten by February. So the maintenance is automated in two layers.

**Renovate**, every Monday morning, reads `versions.yaml` through a regex manager and opens pull requests:

| Update | What happens |
|---|---|
| Patch of Python, Node or Go | One grouped PR, no approval needed |
| Minor or major of a runtime | Waits on the Dependency Dashboard until I tick the box |
| Odd Node major | Never proposed, LTS only |
| Debian snapshot | PR for the next dated tag |
| GitHub Actions | One grouped PR, pinned by digest |

Renovate merges nothing itself. The merge authority is the second layer.

**The agent pipeline**, the same one I use for `pytbak`, lives in a shared repository (`ci-shared`, tag `v3`) and is wired into this repo with a handful of small workflow files:

- every pull request, Renovate's included, gets an architect's brief, a developer, a second developer, QA, SRE and security roles, and one comment;
- when `PR Checks` fails, an agent reads the real logs and pushes a fix to the branch, three attempts at most;
- when the PR is certified and the checks are green, it is squash-merged;
- if the build on `main` breaks after a merge, a guard reverts it, with a circuit breaker after three reverts in 24 hours.

`PR Checks` itself is deterministic: it builds **all five images**, in cascade order, with the same script I run on my laptop, then runs smoke tests. Because a distroless image has no shell, each runtime is tested by running the runtime itself:

```bash
out=$(docker run --rm lg-python:local -c "import sys; print('.'.join(map(str, sys.version_info[:3])))")
[ "$out" = "$PYTHON_VERSION" ] || fail "lg-python is $out, pinned $PYTHON_VERSION"
docker run --rm lg-python:local -c "
import ssl, sqlite3, ctypes, zlib, bz2, lzma, uuid, decimal, hashlib, pyexpat, zoneinfo
ssl.create_default_context().load_default_certs()
print('ok')" | grep -qx ok || fail "an extension module or certificate is missing"
```

That second block is not decoration. It is the reason the next section is a short one instead of a night of production debugging.

## The Test: pytbak on the New Base

Theory is cheap. I took `pytbak`, multi-stage, dependencies installed on `python:3.15.2-slim-trixie` into a venv, venv copied onto `lg-python`:

```dockerfile
FROM python:3.15.2-slim-trixie AS build
RUN python -m venv /venv
COPY requirements.txt .
RUN /venv/bin/pip install --no-cache-dir -r requirements.txt

FROM lgirardi/lg-python:3.15.2
COPY --from=build /venv /venv
COPY --chown=65532:65532 app/ /app/app/
WORKDIR /app
ENTRYPOINT ["/venv/bin/python", "-m", "uvicorn"]
CMD ["app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Started next to PostgreSQL and Redis, the log said what I wanted to read:

```
Startup complete | pg=connected redis=connected env=development
```

`/api/mgmt/ready`, `/api/mgmt/health` and `/metrics` all answered 200. Then I ran Trivy on both versions, the old one and the new one:

| | `pytbak:v2.0.425` (python slim) | `pytbak` on `lg-python` |
|---|---|---|
| Image size | 356 MB | 241 MB |
| Base | Debian 13.7 | Debian 13.8 |
| OS packages | 436 | **74** |
| Python packages | 8 | 8 |
| CRITICAL | 4 | **0** |
| HIGH | 125 | **15** |
| MEDIUM | 148 | 45 |
| LOW | 167 | 22 |
| **Total** | **444** | **82** |

From 444 to 82, with no change to a single line of application code. And the 4 CRITICAL are gone. The 8 Python findings are identical in both columns because they come from `requirements.txt`, and a base image cannot patch those.

Of the 15 HIGH that remain:

- **10 are OS packages with no fix upstream** (`libuuid1`, `libsqlite3-0`, `libncursesw6`, `libtinfo6`, `libssl3t64`). Debian has not shipped a patched package yet, so the only answer is the weekly rebuild when it does. The gate does not block on these on purpose.
- **5 are Python packages, all with a fix available**: `msgpack` (2 CVEs), `urllib3` (2) and `setuptools` (1). Bumping two lines in `requirements.txt` removes four of them.

That is the honest picture: a smaller surface, not a clean one.

## What Broke on the Way

Three things, in the order they bit me. All three would have been invisible without the smoke tests.

1. **`libzstd.so.1` not found, on `import ssl`.** OpenSSL 3.5 links zstd for certificate compression. My package list had `libssl3t64` and not its new dependency. Python started fine and died the first time something touched TLS.
2. **`libsqlite3.so.0` not found, and an empty package list.** To find the libraries CPython needs, I run `ldd` on every extension module and map each library to its Debian package with `dpkg -S`. On a usr-merged Debian `ldd` prints `/lib/aarch64-linux-gnu/...` while dpkg recorded `/usr/lib/...`, so `dpkg -S` found nothing and returned nothing, silently. The fix is `readlink -f` before the lookup. The lesson is the guard I added afterwards: `test -n "$pkgs"` makes the build fail loudly when the list is empty.
3. **`ping` and `traceroute` gone.** pytbak has opt-in debug endpoints (`/api/debug/network/scan`) that run `ping` and `traceroute` as subprocesses. In a distroless image those binaries do not exist. Before: ping and traceroute output. After: `[Errno 2] No such file or directory` and `traceroute not installed`. The DNS part keeps working because it is pure Python.

The third one is not a bug of the image, it is the price of the image. The options are to reimplement the tools in Python (ICMP raw sockets need `NET_RAW`), to keep a `:tools` variant for the few places that need it, or to disable the debug endpoints in production, where they are already off by default. A distroless image will always show you which dependencies on the operating system you did not know you had.

## Security Considerations

1. **No shell, no package manager, no `curl`.** An attacker who gets code execution in the container finds nothing to work with. No `sh -c`, no `apt install`, no `wget`.
2. **Non-root by default.** `lg-base` and everything on it run as uid 65532. Kubernetes manifests need no `runAsUser` tricks.
3. **Scan before publish, not after.** The gate sits between build and push. A vulnerable base never reaches a registry that other images pull from, and the runtimes on top of it never start.
4. **Digest pinning inside the cascade.** A runtime is built from one specific base. A moving tag cannot swap what is underneath it between two builds.
5. **Real package records.** Because `status.d` is populated, scanners see what is actually in the image. An empty database would make the report green and meaningless.
6. **SBOM and provenance on every push**, so "what is inside this tag?" has an answer that does not depend on my memory.
7. **A weekly rebuild is a security control, not a chore.** It is how a glibc fix reaches every runtime with no change in any repository.

## Monitoring and Observability

I would watch four things:

- **The age of the base in production.** The dated tag tells you: a `lg-python` older than two weeks means somebody stopped following the base.
- **The Trivy summary of the weekly run.** Every run writes the findings by severity into the job summary. A trend of the unfixed HIGH count is a good indicator of how long Debian is taking.
- **The pipeline watching itself.** A scheduled health check, with no model involved, goes red when a commit on `main` has no build, when a pull request is stuck, or when one of the agent workflows keeps failing.
- **A nightly canary.** A known change to a harmless module goes through the whole agent pipeline, and the outcome is checked. If the agents start doing something other than what they should, I find out from a red run and not from a bad merge.

## Conclusion

The container image is not the unit to patch. The **layer** is. Put the operating system in one place, build it from the packages and nothing else, pin the versions you choose, rebuild on a schedule, scan before you publish, and let a bot propose the version bumps while a pipeline reviews them. Then a CVE in OpenSSL is one rebuild of one base, not N tickets.

For `pytbak` the first proof is encouraging: 444 vulnerabilities down to 82, 4 CRITICAL down to zero, 356 MB down to 241 MB, same application, same endpoints, except the ones that shelled out to `ping`.

## Reflections

This is not finished, and I would rather say so.

**What is still missing?**

- **The applications do not follow the base by themselves yet.** The cascade updates the images in `ACT-lg-baseimg`. The services that use `lg-python` still need a pull request in their own repository to move to a new tag. Renovate can open it, the Docker datasource does exactly that for a `FROM` line, but I have not wired it up for `pytbak` yet. Until I do, the cascade stops at the base images and the last mile is manual.
- **The first run of the full pipeline on GitHub runners is still in progress while I write this.** Everything above was built and tested locally on arm64. amd64 and the registry push are exactly the parts I have not seen with my own eyes yet.
- **Ten HIGH findings have no fix and I cannot make one.** That is the honest floor of "Debian trixie today".
- **Debugging a distroless container is a new habit.** `kubectl debug` with an ephemeral container works, but it is not a shell in the pod, and it takes a minute to stop reaching for `exec`.

The pattern is still the right one: fewer things in the image, one place to patch, and a pipeline that does not need me to remember.

