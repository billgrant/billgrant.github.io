---
title: "Demo App: Phase 1 - Building the Foundation in Go"
date: 2026-01-07
categories:
  - projects
  - go
tags:
  - demo-app
  - go
  - learning-in-public
  - docker
  - sqlite
---


***I think I am becoming a serial vibe coder. For a number of years I have been looking for the perfect demo app. I build a lot of labs and often live demo solutions. The apps I use are boring and "Hello World" style. I want to build something universal that could be used to demo a wide array of products and solutions. It should add value to the demo without taking away from the solution. I have no idea what that actually would look like but I am going to build it anyway. I am using this opportunity to learn Go with Claude as my teacher. The content below was written by Claude. It covers our Phase 1 sessions in detail.***

---

## Why This Project?

When demoing infrastructure tools, you need something to deploy. Most demo apps are either:
- **Too simple** — "Hello World" doesn't impress anyone
- **Too complex** — Full production apps with tons of dependencies
- **Too specific** — Built for one vendor's demo

I wanted something in the sweet spot: real REST API, database, structured logging, single binary deployment.

## The Tech Decisions

Before writing code, Claude and I talked through the options:

| Decision | Choice | Why |
|----------|--------|-----|
| Router | stdlib `net/http` | Learn fundamentals before frameworks |
| Logging | `log/slog` | Stdlib, structured JSON, no dependencies |
| SQLite driver | `modernc.org/sqlite` | Pure Go, no CGO for easy cross-compilation |
| Docker base | Docker Hardened Images | Shift-left security, CVE-free baseline |

The theme: **start simple, add abstractions only when needed**.

## The Mental Shift: ResponseWriter

The biggest "aha" moment was understanding Go's HTTP handler pattern.

In Flask, you return a response:
```python
@app.route('/health')
def health():
    return jsonify({"status": "ok"})  # Flask handles the rest
```

In Go, you're handed the wire and you write to it:
```go
func healthHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(response)
    // No return — you wrote directly to the connection
}
```

`ResponseWriter` IS the return. Flask abstracts this; Go hands you the raw mechanism. More code, but no mystery about what's actually being sent.

## Pointers: The Python Developer's Confusion

Go makes pointers explicit. In the handler signature:

```go
func healthHandler(w http.ResponseWriter, r *http.Request)
//                                         ^ this asterisk
```

That `*http.Request` means "pointer to an http.Request" — a memory address, not a copy of the data.

Why it matters:
- **Without pointer:** function gets a copy, changes stay local
- **With pointer:** function gets reference, changes affect original

Python does this automatically for objects. In Go, you're explicit about it. The syntax `r *http.Request` reads as "r is a pointer to http.Request."

## Middleware: Wrapping Handlers

To log every request, we needed middleware — a function that wraps handlers to add behavior:

```go
func loggingMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        // Call the actual handler
        next(w, r)

        // Log after it completes
        slog.Info("request",
            "method", r.Method,
            "path", r.URL.Path,
            "latency_ms", time.Since(start).Milliseconds(),
        )
    }
}
```

The trick: `ResponseWriter` doesn't expose the status code after you write it. We had to wrap it:

```go
type responseRecorder struct {
    http.ResponseWriter
    statusCode int
}

func (r *responseRecorder) WriteHeader(code int) {
    r.statusCode = code              // intercept and save
    r.ResponseWriter.WriteHeader(code) // pass through
}
```

This struct embedding pattern — putting `http.ResponseWriter` without a field name — gives our struct all the original methods, letting us override just `WriteHeader`.

## SQLite: The Underscore Import

Adding SQLite introduced a Go idiom that looked strange at first:

```go
import (
    "database/sql"
    _ "modernc.org/sqlite"  // underscore import
)
```

The `_` means "import for side effects only." The driver's `init()` function registers itself with `database/sql` when imported. You never call it directly:

```go
db, err := sql.Open("sqlite", ":memory:")
//                  ^^^^^^^^ looks up the registered driver
```

## Error Handling: Verbose but Explicit

Go doesn't have exceptions. Every function that can fail returns an error:

```go
db, err := initDB(dbPath)
if err != nil {
    slog.Error("failed to initialize database", "error", err)
    os.Exit(1)
}
defer db.Close()
```

You'll write `if err != nil` hundreds of times. It's verbose, but errors are always visible — no hidden control flow.

That `defer db.Close()` schedules `Close()` to run when the function exits, like Python's `with` statement.

## Docker Hardened Images

For the container, I wanted to try Docker's new Hardened Images (DHI) — minimal, CVE-free base images. Multi-stage build:

```dockerfile
# Build stage: full Go SDK
FROM dhi.io/golang:1.25-alpine3.22-dev AS build-stage
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY *.go ./
RUN CGO_ENABLED=0 GOOS=linux go build -o /demo-app

# Runtime stage: minimal static image
FROM dhi.io/static:20250911-alpine3.22 AS runtime-stage
COPY --from=build-stage /demo-app /demo-app
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD ["/demo-app", "healthcheck"]
ENTRYPOINT ["/demo-app"]
```

The static image has no curl or wget, so we added a healthcheck mode to the binary itself:

```go
func main() {
    if len(os.Args) > 1 && os.Args[1] == "healthcheck" {
        resp, err := http.Get("http://localhost:8080/health")
        if err != nil || resp.StatusCode != 200 {
            os.Exit(1)
        }
        os.Exit(0)
    }
    // ... normal server startup
}
```

Now `docker ps` shows the container as `(healthy)`.

## Phase 1 Complete

What we built:
- HTTP server with `/health` endpoint
- Structured JSON logging with request middleware
- SQLite database (in-memory or file-based)
- Docker container with hardened images
- Self-contained healthcheck

All in about 130 lines of Go.

## What's Next

Phase 2: the actual API endpoints — CRUD for items, display panel for injected demo content, system info endpoint. The foundation is solid; now we build on it.

---

***Three sessions in and I'm starting to feel the Go patterns clicking. The explicit error handling felt tedious at first, but I can see how it prevents the "exception hiding somewhere" problem. The ResponseWriter thing was the biggest shift — Flask spoiled me by hiding the HTTP mechanics. Still getting used to pointers, but the "is this a copy or reference?" question is clearer when it's explicit in the code. On to Phase 2.***
