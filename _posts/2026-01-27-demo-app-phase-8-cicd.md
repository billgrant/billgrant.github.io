---
layout: post
title: "Demo App: Phase 8 — CI/CD & Publishing"
date: 2026-01-27
tags: demo-app go learning-in-public
---

***In this session we covered unit tests, CI/CD, binary and container publishing, and Terraform provider publishing. This session was probably the easiest one for me to follow as it was relatively light on the Go. Really interesting to learn what it looks like to publish and maintain a provider on registry.terraform.io. I will let Claude explain what we did in detail below***

## What We Built

Phase 8 had two tracks running in parallel:

**Phase 8 (demo-app):**
- Unit tests for all API handlers
- CI pipeline — build, test, lint, Docker verification
- Release automation — multi-arch binaries and container images

**Phase 8b (terraform-provider-demoapp):**
- GPG signing for provider binaries
- GoReleaser release workflow
- Published to the Terraform Registry
- Updated examples to use published artifacts

The end result: the entire project is now consumable by anyone with Terraform and Docker installed.

## Writing Tests in Go

Go's stdlib includes `net/http/httptest`, which lets you test HTTP handlers without starting a real server.

```go
req := httptest.NewRequest("GET", "/health", nil)
rr := httptest.NewRecorder()

healthHandler(rr, req)

if rr.Code != http.StatusOK {
    t.Errorf("expected status 200, got %d", rr.Code)
}
```

`httptest.NewRequest` creates a fake request, `httptest.NewRecorder` captures the response. You call the handler directly — no network, no ports, no server startup. It's fast and deterministic.

### Test Setup with TestMain

Our handlers depend on package-level variables (`db`, `itemSeq`) that need to be initialized. Go provides `TestMain` for this:

```go
func TestMain(m *testing.M) {
    var err error
    db, err = initStore(":memory:")
    if err != nil {
        panic("failed to init test database: " + err.Error())
    }
    defer db.Close()

    itemSeq, err = db.GetSequence([]byte("seq:items"), 100)
    if err != nil {
        panic("failed to init test sequence: " + err.Error())
    }
    defer itemSeq.Release()

    os.Exit(m.Run())
}
```

`TestMain` runs once before all tests in the package. `m.Run()` executes every `Test*` function and returns an exit code. This is Go's equivalent of pytest's `conftest.py` fixtures.

### What We Test

14 tests covering all API endpoints:

| Endpoint | Tests |
|----------|-------|
| `/health` | Returns 200 with status and timestamp |
| `/api/items` | Create, list, get, update, delete |
| `/api/items` errors | Non-existent (404), invalid ID (400), bad JSON (400), missing name (400) |
| `/api/display` | Empty default, set and get, invalid JSON |
| `/api/system` | Expected fields present, POST rejected (405) |

Coverage landed at ~48% of statements — solid coverage of the core API paths without over-testing.

## CI Pipeline

The CI workflow (`.github/workflows/ci.yml`) runs on every push to `main` and on pull requests. Two jobs run in parallel:

### Build and Test

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: actions/setup-go@v5
    with:
      go-version-file: go.mod   # reads version from go.mod, not hardcoded

  - run: go build -o demo-app .
  - run: go test -v -cover ./...
  - run: go vet ./...
  - run: staticcheck ./...
```

Two linters:
- **`go vet`** — stdlib, catches suspicious constructs (wrong printf formats, unreachable code)
- **`staticcheck`** — most popular Go linter, catches deprecated APIs, unnecessary conversions, and code simplifications

### Docker Verification

```yaml
steps:
  - uses: docker/login-action@v3  # DHI base images need auth
    with:
      registry: dhi.io
      username: {% raw %}${{ secrets.DHI_USERNAME }}{% endraw %}
      password: {% raw %}${{ secrets.DHI_PASSWORD }}{% endraw %}

  - run: docker build -t demo-app:ci .
  - run: docker run -d --name demo-app -p 8080:8080 demo-app:ci

  # Poll health endpoint until it responds
  - run: |
      for i in $(seq 1 30); do
        if curl -sf http://localhost:8080/health > /dev/null 2>&1; then
          echo "Health check passed"
          exit 0
        fi
        sleep 1
      done
      echo "Health check failed after 30 seconds"
      docker logs demo-app
      exit 1
```

This catches real deployment issues — the binary compiles, the container builds, the app starts, and the health endpoint responds.

### Skipping CI for Docs

One early catch: CI was running on README changes. Since docs don't affect the binary, we added path filters:

```yaml
on:
  push:
    branches: [main]
    paths-ignore:
      - "*.md"
      - "docs/**"
```

## Release Automation

The release workflow triggers on version tags (`git tag v0.8.0 && git push --tags`).

### Multi-Arch Binary Builds

A strategy matrix builds 6 binaries in parallel:

| OS | Arch |
|----|------|
| linux | amd64, arm64 |
| darwin (macOS) | amd64, arm64 |
| windows | amd64, arm64 |

```yaml
- name: Build binary
  env:
    GOOS: {% raw %}${{ matrix.goos }}{% endraw %}
    GOARCH: {% raw %}${{ matrix.goarch }}{% endraw %}
    CGO_ENABLED: "0"
  run: |
    EXT=""
    if [ "$GOOS" = "windows" ]; then EXT=".exe"; fi
    go build -ldflags="-s -w" -o demo-app-${GOOS}-${GOARCH}${EXT} .
```

The `-ldflags="-s -w"` strips debug symbols and DWARF info — smaller binaries with no functional difference.

### Multi-Arch Docker Images

```yaml
- uses: docker/build-push-action@v6
  with:
    platforms: linux/amd64,linux/arm64
    push: true
    tags: |
      ghcr.io/billgrant/demo-app:v0.8.0
      ghcr.io/billgrant/demo-app:latest
```

Docker Buildx with QEMU handles cross-compilation. The image is pushed to GitHub Container Registry with both a version tag and `latest`.

### First Bug Fix: v0.8.1

The initial release failed at the artifact download step. The `docker/build-push-action` creates an internal buildx metadata artifact, and our download step was grabbing *everything*:

```
Found 7 artifact(s)  # expected 6 binaries, got 7
Error: Unable to download artifact(s)
```

Fix: add a pattern filter to only download our binary artifacts:

```yaml
- uses: actions/download-artifact@v4
  with:
    pattern: demo-app-*    # skip the buildx artifact
    merge-multiple: true
```

Tagged `v0.8.1` — the versioning scheme validated itself on day one.

## Versioning Scheme

We adopted a phase-based semver approach:

| Segment | Meaning | Example |
|---------|---------|---------|
| `MINOR` | Maps to project phase | Phase 8 = `v0.8.0` |
| `PATCH` | Bug fixes within a phase | `v0.8.1` |
| `MAJOR` | Production-ready | `v1.0.0` (someday) |

This gives clear traceability — you can look at a version and know which phase it corresponds to.

## Phase 8b: Terraform Provider CI/CD

### GPG Signing

The Terraform Registry requires GPG-signed checksums to verify provider binaries. The setup:

1. Generate a GPG key (`gpg --full-generate-key`, RSA 4096)
2. Public key → Terraform Registry (Settings > Signing Keys)
3. Private key → GitHub secret (`GPG_PRIVATE_KEY`)
4. GoReleaser signs checksums at release time

**One important decision:** use a personal email for the GPG key and Terraform Registry account, not a corporate one. If you leave your job, you don't want to lose control of your signing key and published providers.

### GoReleaser Workflow

The provider uses GoReleaser instead of raw `go build` because the Terraform Registry expects a specific release format (zipped binaries, SHA256 checksums, detached GPG signature).

```yaml
- uses: crazy-max/ghaction-import-gpg@v6
  id: import_gpg
  with:
    gpg_private_key: {% raw %}${{ secrets.GPG_PRIVATE_KEY }}{% endraw %}

- uses: goreleaser/goreleaser-action@v6
  with:
    args: release --clean
  env:
    GITHUB_TOKEN: {% raw %}${{ secrets.GITHUB_TOKEN }}{% endraw %}
    GPG_FINGERPRINT: {% raw %}${{ steps.import_gpg.outputs.fingerprint }}{% endraw %}
```

GoReleaser handles binary builds, zip archives, checksums, GPG signing, and GitHub Release creation — all from a single config file.

### Publishing to the Registry

The Terraform Registry auto-discovers providers from GitHub repos named `terraform-provider-*`. One catch: you need at least one GitHub Release before the registry will accept the provider. We had to create the workflow, tag `v0.1.0`, let it build, then publish.

After that, the registry watches for new releases and indexes them automatically.

## The Zero-Setup Experience

With everything published, the demo-app-examples repo now delivers on the Phase 6 vision:

```bash
git clone https://github.com/billgrant/demo-app-examples.git
cd demo-app-examples/baseline
terraform init    # downloads provider from registry
terraform apply   # pulls container from ghcr.io, deploys, populates data
```

No local builds. No dev overrides. No prerequisites beyond Terraform and Docker.

The `main.tf` changes were minimal:

```hcl
# Before: required local image build
resource "docker_image" "demo_app" {
  name         = "demo-app:latest"
  keep_locally = true
}

# After: pulls from GitHub Container Registry
resource "docker_image" "demo_app" {
  name = "ghcr.io/billgrant/demo-app:latest"
}
```

## Phase 8 Complete

Both tracks done:

**Phase 8 (demo-app) `v0.8.1`:**
- [x] 14 unit tests, ~48% coverage
- [x] CI pipeline (build, test, lint, Docker verify)
- [x] Release automation (6 binaries + multi-arch Docker image)

**Phase 8b (terraform-provider-demoapp) `v0.1.0`:**
- [x] GPG signing + GoReleaser release workflow
- [x] Published to Terraform Registry
- [x] Examples updated to zero-setup experience

Next up: Phase 9 — Distribution & Documentation (Kubernetes manifests, Helm chart, architecture diagrams).
