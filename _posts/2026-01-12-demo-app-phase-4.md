---
title: "Demo App: Phase 4 - Docker & Multi-Arch Builds"
date: 2026-01-12
categories:
  - projects
  - go
tags:
  - demo-app
  - go
  - docker
  - learning-in-public
---

***Quick session tonight. We set up the Dockerfile for multi-arch builds (amd64 and arm64).***

---

## What Changed

With static files now embedded in the Go binary (Phase 3), the Dockerfile needed updates. Previously we only copied Go files — now we copy the static folder too.

```dockerfile
COPY *.go ./
COPY static/ ./static/
```

## Multi-Arch Builds

The goal: one image that works on Intel/AMD servers and M1/M2 Macs.

Docker buildx handles this with `TARGETOS` and `TARGETARCH` build arguments:

```dockerfile
ARG TARGETOS TARGETARCH
RUN CGO_ENABLED=0 GOOS=${TARGETOS:-linux} GOARCH=${TARGETARCH:-amd64} go build -o /demo-app
```

When you run `docker buildx build --platform linux/amd64,linux/arm64`, buildx:
1. Runs the build twice (once per platform)
2. Passes the correct `TARGETOS`/`TARGETARCH` for each
3. Creates a manifest that references both images

### Local Testing with QEMU

Building arm64 on an amd64 machine requires emulation:

```bash
# Enable QEMU for cross-platform builds
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes

# Create a builder that supports multiple platforms
docker buildx create --name multiarch --driver docker-container --use

# Build for both architectures
docker buildx build --platform linux/amd64,linux/arm64 -t demo-app:latest .
```

The arm64 build took ~9 minutes via emulation. In CI/CD with native runners, this will be much faster.

## The Final Dockerfile

```dockerfile
FROM dhi.io/golang:1.25-alpine3.22-dev AS build-stage

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY *.go ./
COPY static/ ./static/

ARG TARGETOS TARGETARCH
RUN CGO_ENABLED=0 GOOS=${TARGETOS:-linux} GOARCH=${TARGETARCH:-amd64} go build -o /demo-app

FROM dhi.io/static:20250911-alpine3.22 AS runtime-stage

COPY --from=build-stage /demo-app /demo-app

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD ["/demo-app", "healthcheck"]

ENTRYPOINT ["/demo-app"]
```

## Phase 4 Complete

- [x] Update Dockerfile for embedded static files
- [x] Multi-arch builds (amd64 + arm64)
- [x] Test containerized deployment

Next up: Terraform Provider.
