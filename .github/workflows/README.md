# CI/CD pipeline

This directory implements the review/release process for bookinfo:

```
feature/* branch ──PR──▶ master branch ──merge──▶ Docker Hub
     │                        │
     ▼                        ▼
 pr.yml                  release.yml
 (ci-checks.yml)          (ci-checks.yml again)
                          + build, Trivy scan, SBOM, push
```

## Workflows

| File | Trigger | Purpose |
|---|---|---|
| [ci-checks.yml](./ci-checks.yml) | `workflow_call` (reusable) | Unit tests per service, CodeQL (SAST) |
| [pr.yml](./pr.yml) | `pull_request` → `master` | Enforces `feature/*` branch naming, then calls `ci-checks.yml` |
| [release.yml](./release.yml) | `push` → `master` (i.e. after merge) | Re-runs `ci-checks.yml`, then per service: build → Trivy scan → SBOM → push to Docker Hub |

Fortify was intentionally **not** included — it requires a paid license.
SonarCloud was also considered and dropped — its free tier only covers public
repos unconditionally (private repos are free up to 50,000 lines of code, and
its onboarding flow defaults to a paid "Team" plan unless you explicitly pick
"Free"), and the goal here was a zero-account, zero-billing pipeline. CodeQL
(free, built into GitHub, no account/signup needed) covers SAST instead, and
Trivy covers container scanning.

## Required repository secrets

| Secret | Used by | Notes |
|---|---|---|
| `DOCKERHUB_USERNAME` | `release.yml` | Docker Hub username/org |
| `DOCKERHUB_TOKEN` | `release.yml` | Docker Hub access token (not your password) |

## Branch protection (must be configured manually in GitHub, not via YAML)

GitHub Actions workflows cannot by themselves stop someone from merging or
approving a PR — that enforcement comes from a **branch protection rule**,
configured under *Settings → Branches → Add rule* for `master`:

1. **Require a pull request before merging**, with **Require approvals ≥ 1**.
2. **Require status checks to pass before merging**, and select the checks
   once they've run at least once on a PR:
   - `branch-naming-convention`
   - `checks / test-productpage`, `checks / test-details`, `checks / test-ratings`, `checks / test-reviews`
   - `checks / codeql (python)`, `(javascript-typescript)`, `(ruby)`, `(java-kotlin)`
3. **Require branches to be up to date before merging** (recommended).
4. Optionally **Do not allow bypassing the above settings** so it also applies to admins.

> Note: GitHub's "require approvals" blocks the **merge button**, not the
> reviewer's ability to click *Approve* on a PR with failing checks — a
> reviewer can still approve, they just can't merge until checks are green.
> If you need to hard-block approval itself, that requires an org policy /
> CODEOWNERS + a required review from a bot that only approves on green CI,
> which is out of scope for a plain Actions workflow.

## Container scanning policy

`release.yml` fails the job (blocking the Docker Hub push) if Trivy finds any
`CRITICAL` vulnerability. To also fail on `HIGH`, change `severity: CRITICAL`
to `severity: CRITICAL,HIGH` in `release.yml`. To report without blocking,
set `exit-code: "0"`.

## Docker build time & caching

Benchmarked locally (`docker build`, BuildKit) for each service's image:

| Service | Cold build (no cache) | Rebuild, unchanged (cache hit) |
|---|---|---|
| mongodb | ~13s | ~3.6s |
| mysql | ~15s | ~3.4s |
| details (ruby) | ~6.7s | ~5.7s |
| ratings (node) | ~14.2s | <1s* |
| productpage (python) | ~21.9s | <1s* |
| reviews (java/gradle, multi-stage) | ~30.4s | <1s* |

\* once every layer hits cache (same base image, same manifests, same source),
BuildKit skips re-executing `RUN`/`COPY` entirely — the "rebuild" is close to
instant. **reviews** is the slowest cold build because it's a multi-stage
build that runs a full Gradle build plus Open Liberty feature installation.

**Why this matters for CI:** GitHub Actions runners are ephemeral — every job
starts with an empty Docker build cache unless you explicitly persist it.
Without caching, every single merge to `master` would pay the full "cold
build" cost for all 6 images (~100s combined, serialized), or worse if
dependencies need to be re-downloaded from slow upstream mirrors.

`release.yml` solves this with the **GitHub Actions cache backend** for
BuildKit:

```yaml
cache-from: type=gha,scope=${{ matrix.service }}
cache-to: type=gha,mode=max,scope=${{ matrix.service }}
```

- `type=gha` stores/retrieves layers from GitHub's built-in Actions cache
  (fast, free, no extra setup) instead of `type=registry` (pushes a cache
  image to Docker Hub — more storage, slightly slower, but not subject to the
  ~10GB per-repo GHA cache eviction limit) or `type=local`/`actions/cache`
  (older approach, works but less efficient with BuildKit's native exporter).
- `scope=<service>` keeps each service's cache independent so a change to
  `reviews` doesn't evict the `productpage` cache.
- `mode=max` also caches intermediate build-stage layers (important for
  `reviews`, which is multi-stage — the Gradle builder stage's dependency
  download layer benefits the most).

With this in place, a `master` push where only `ratings.js` changed will
rebuild **only** the `ratings` image's final `COPY ratings.js` layer — the
`npm install` layer and every other service's build stay fully cached. (This
also required reordering `ratings/Dockerfile` to install dependencies
*before* copying source, so source-only changes don't bust the dependency
install layer — see the diff in that file.)

If you later find dependency installs (not just full-layer cache misses) are
still slow when `package.json`/`requirements.txt`/`Gemfile` actually change,
the next lever is **BuildKit cache mounts** for the package manager's
download cache, e.g.:

```dockerfile
RUN --mount=type=cache,target=/root/.npm npm install
RUN --mount=type=cache,target=/root/.cache/pip pip3 install -r requirements.txt
```

These don't replace full layer caching (the mounted cache isn't part of the
final image layer, so the install step still runs), but they avoid
re-downloading packages from the network on every dependency bump.
