---
title: "Demo App: Phase 7 — Observability & Polish"
date: 2026-01-22
categories:
  - projects
  - go
tags:
  - demo-app
  - go
  - observability
  - prometheus
  - learning-in-public
---

***This phase started off simple create metrics endpoint and create a webhook that ships generic logs. We ended up refactoring the code into multiple files given the size of main.go. Something about Go really clicked for me when we dicussed structs and interfaces at length. I will let Claude explain the rest*** 

## What We Built
Phase 7 focused on making demo-app production-ready from an observability standpoint:

- **Prometheus metrics endpoint** (`/metrics`)
- **Code refactoring** — split monolithic `main.go` into focused files
- **Request headers & client info** — visible in System Info panel
- **Log webhook shipping** — optional HTTP POST for log aggregation
- **Environment variable filtering** — regex-based control via `ENV_FILTER`
- **Configuration documentation** — comprehensive `docs/CONFIGURATION.md`

## Prometheus Metrics

The `/metrics` endpoint exposes both application-specific and Go runtime metrics in Prometheus format.

### Application Metrics

| Metric | Type | Purpose |
|--------|------|---------|
| `demoapp_http_requests_total` | Counter | Request counts by method, path, status |
| `demoapp_http_request_duration_seconds` | Histogram | Response time distribution |
| `demoapp_items_total` | Gauge | Current item count in database |
| `demoapp_display_updates_total` | Counter | Display panel POST count |
| `demoapp_info` | Gauge | Build info (version label) |

### Path Normalization

One subtle but important detail: we normalize paths to prevent cardinality explosion.

```go
func normalizePath(path string) string {
    if strings.HasPrefix(path, "/api/items/") {
        return "/api/items/:id"
    }
    return path
}
```

Without this, every unique item ID (`/api/items/1`, `/api/items/2`, etc.) would create a separate metric series. In Prometheus, high cardinality means high memory and storage costs. Normalizing to `/api/items/:id` keeps the metric series bounded.

### Why Prometheus Over OpenTelemetry

We chose Prometheus format for simplicity and compatibility. Most observability platforms (Splunk, Datadog, Grafana, New Relic) can ingest Prometheus metrics natively or via collectors. OpenTelemetry is powerful but adds complexity we don't need for a demo app.

## Code Refactoring

Before adding more features, we split `main.go` (~730 lines) into focused files:

| File | Lines | Responsibility |
|------|-------|----------------|
| `handlers.go` | 455 | HTTP endpoint logic |
| `main.go` | 149 | Startup, configuration, routing |
| `middleware.go` | 90 | Request logging, metrics recording |
| `metrics.go` | 74 | Prometheus definitions |
| `store.go` | 67 | Data model, database setup |

### Go's Package Scope

A key learning: all `.go` files in the same directory with `package main` share variables and functions automatically. No imports needed between files in the same package.

```go
// store.go
var db *badger.DB  // package-level

// handlers.go
func listItems(w http.ResponseWriter, r *http.Request) {
    db.View(...)  // uses db from store.go directly
}
```

This is exactly how Terraform works — all `.tf` files in a directory share the same namespace. Same design pattern, since Terraform is written in Go.

### The `:=` Shadowing Trap

One gotcha that caught us: `:=` creates a NEW variable even if a package-level one exists.

```go
var db *badger.DB  // package-level

func main() {
    db, err := initStore()  // WRONG: creates local db, package-level stays nil!

    var err error
    db, err = initStore()   // RIGHT: assigns to existing package-level db
}
```

## Log Webhook Shipping

For demos where you're running the binary directly (not in a container with log agents), we added optional webhook shipping.

```bash
LOG_WEBHOOK_URL="https://splunk.example.com:8088/services/collector" \
LOG_WEBHOOK_TOKEN="Splunk your-hec-token" \
./demo-app
```

### Implementation: Custom slog.Handler

Go's `log/slog` package uses handlers to control where logs go. We created a custom handler that wraps the standard JSONHandler:

```go
type webhookHandler struct {
    jsonHandler slog.Handler
    webhookURL  string
    authToken   string
    client      *http.Client
}

func (w *webhookHandler) Handle(ctx context.Context, record slog.Record) error {
    // Always write to stdout via wrapped handler
    if err := w.jsonHandler.Handle(ctx, record); err != nil {
        return err
    }

    // Async POST to webhook (fire and forget)
    go w.postToWebhook(record)
    return nil
}
```

Key design decisions:
- **Async calls** — webhooks POST in a goroutine so slow endpoints don't block HTTP responses
- **Fire and forget** — failed webhook calls log to stderr but don't affect the app
- **Vendor neutral** — no Splunk SDK, no Loki SDK, just HTTP POST with JSON

## Environment Variable Filtering

The System Info panel displays environment variables, but which ones? Originally we had a hardcoded allowlist. Now users can control it with `ENV_FILTER`.

```bash
# Show only vars starting with DEMO_
ENV_FILTER="^DEMO_" ./demo-app

# Show Kubernetes-related vars
ENV_FILTER="^(POD_|NODE_|KUBERNETES_)" ./demo-app
```

### Implementation

```go
func getFilteredEnvVars() map[string]string {
    if pattern := os.Getenv("ENV_FILTER"); pattern != "" {
        // Case-insensitive regex matching
        re, err := regexp.Compile("(?i)" + pattern)
        if err != nil {
            slog.Error("invalid ENV_FILTER regex", "pattern", pattern, "error", err)
            return make(map[string]string)  // safe fallback
        }

        result := make(map[string]string)
        for _, envVar := range os.Environ() {
            parts := strings.SplitN(envVar, "=", 2)
            if re.MatchString(parts[0]) {
                result[parts[0]] = parts[1]
            }
        }
        return result
    }

    // Default: safe allowlist
    // ...
}
```

### Learning: Where Do Environment Variables Come From?

A question that came up: how does `os.Getenv()` work?

Every process inherits environment variables from its parent (your shell). When you run `./demo-app`, it gets a snapshot of the shell's environment. You can see the same data with `env` or `printenv` in bash.

This is different from `os.Hostname()`, which makes a syscall to the kernel. The `HOSTNAME` environment variable may or may not be set (Docker sets it, desktop Linux often doesn't), but `os.Hostname()` always works.

## Configuration Documentation

We created `docs/CONFIGURATION.md` with comprehensive documentation for all environment variables. The README has a quick reference table linking to the full docs.

This structure keeps the README scannable while providing depth for users who need it.

## Bug Fix: CSS Overflow

While testing `ENV_FILTER`, we noticed long variable names (like `DEMO_LONG_VARIABLE_NAME`) overlapped with values in the dashboard. The fix was simple:

```css
/* Before */
#system-content .info-label {
    width: 100px;  /* fixed width, text overflows */
}

/* After */
#system-content .info-label {
    min-width: 100px;  /* minimum width, can grow */
    margin-right: 1rem;
}
```

## Phase 7 Complete

All observability features are in place:

- [x] Prometheus `/metrics` endpoint
- [x] Code refactoring
- [x] Request headers & client info
- [x] Log webhook shipping
- [x] Environment variable filtering
- [x] Configuration documentation

Next up: Phase 8 — CI/CD with tests, GitHub Actions, and automated releases.
