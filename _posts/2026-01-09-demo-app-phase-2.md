---
title: "Demo App: Phase 2 - Building the REST API"
date: 2026-01-09
categories:
  - projects
  - go
tags:
  - demo-app
  - go
  - learning-in-public
  - api
  - sqlite
---


***Tonight we built the API for our demo app. I had a real moment where Go just clicked for me. It was when we built the display API. Instead of reading Claude's explanation I read the code and wrote my description to Claude of exactly what each line was doing. I was dead on with my description, for the most part some typos and logic errors. All in all, with enough time I could have written that myself without AI's help. This is something that has never happened for me in the past when I tried to teach myself Go. I would get frustrated and go back to Python. I will let Claude cover the fine details of our implementation below.***

---

## What We Built

Phase 2 turned the foundation into a real API. Three endpoint groups, each serving a different purpose:

| Endpoint | Purpose |
|----------|---------|
| `/api/items` | Full CRUD — proves the database works, generates realistic logs |
| `/api/display` | Accepts arbitrary JSON — inject Terraform outputs, Vault responses, anything |
| `/api/system` | Returns hostname, IPs, env vars — shows where the app is running |

By the end, we had a complete REST API in about 400 lines of Go.

## Manual Routing: The stdlib Limitation

The standard library's `net/http` doesn't support path parameters like `/api/items/:id`. Frameworks like Gin or Chi handle this automatically, but we're learning fundamentals first.

Our solution: parse the URL manually.

```go
func itemsHandler(w http.ResponseWriter, r *http.Request) {
    // Extract ID from path: /api/items/123 -> "123"
    path := strings.TrimPrefix(r.URL.Path, "/api/items")
    path = strings.TrimPrefix(path, "/")

    if path == "" {
        // /api/items (no ID)
        switch r.Method {
        case http.MethodGet:
            listItems(w, r)
        case http.MethodPost:
            createItem(w, r)
        default:
            http.Error(w, `{"error":"method not allowed"}`, 405)
        }
    } else {
        // /api/items/:id
        id, err := strconv.ParseInt(path, 10, 64)
        if err != nil {
            http.Error(w, `{"error":"invalid id"}`, 400)
            return
        }
        // Route to getItem, updateItem, or deleteItem based on method
    }
}
```

It's more code than a framework would require, but now I understand exactly what routers do under the hood.

## The Variable Shadowing Trap

This one bit me. We have a package-level database variable:

```go
var db *sql.DB  // package-level, handlers need access
```

In `main()`, I initially wrote:

```go
db, err := initDB(dbPath)  // WRONG
```

The `:=` operator *always* creates a new variable in the current scope. This created a local `db` that shadowed the package-level one. The handlers were using the package-level `db` — which was still `nil`.

The fix:

```go
var err error
db, err = initDB(dbPath)  // RIGHT: assigns to existing package-level db
```

Use `:=` for new variables, `=` for existing ones. Go won't warn you about shadowing — you just get nil pointer panics at runtime.

## Database Operations: Query vs QueryRow vs Exec

Go's `database/sql` has three methods for running queries:

| Method | Use When | Returns |
|--------|----------|---------|
| `db.Query()` | Multiple rows expected | `*Rows` to iterate |
| `db.QueryRow()` | Single row expected | `*Row` to scan once |
| `db.Exec()` | No rows returned (INSERT, UPDATE, DELETE) | `Result` with affected rows |

For listing items:

```go
rows, err := db.Query("SELECT id, name, description, created_at FROM items")
defer rows.Close()

for rows.Next() {
    var item Item
    rows.Scan(&item.ID, &item.Name, &item.Description, &item.CreatedAt)
    items = append(items, item)
}
```

For getting a single item:

```go
err := db.QueryRow("SELECT ... WHERE id = ?", id).
    Scan(&item.ID, &item.Name, &item.Description, &item.CreatedAt)

if err == sql.ErrNoRows {
    // 404
}
```

The `Scan` function needs pointers (`&item.ID`) because it writes values into your variables. Without the `&`, it would get copies and your struct would stay empty.

## Storing Arbitrary JSON

The `/api/display` endpoint accepts any valid JSON — Terraform outputs, Vault responses, custom demo data. We don't know the structure ahead of time.

Go's solution: `json.RawMessage`.

```go
var displayData json.RawMessage  // package-level, in-memory

func setDisplay(w http.ResponseWriter, r *http.Request) {
    var data json.RawMessage
    if err := json.NewDecoder(r.Body).Decode(&data); err != nil {
        http.Error(w, `{"error":"invalid json"}`, 400)
        return
    }
    displayData = data  // store whatever they sent
    w.WriteHeader(http.StatusCreated)
    w.Write(displayData)
}
```

`json.RawMessage` holds raw JSON bytes without parsing them into a struct. It validates that the input is valid JSON, but doesn't care about the structure. Perfect for a "store anything" endpoint.

We chose in-memory storage (a package variable) over database storage because this data is transient — demo content that doesn't need to survive restarts.

## Type Assertions: Extracting Concrete Types

The `/api/system` endpoint returns the server's IP addresses. Go's `net` package gives us network interfaces, but the addresses come back as an interface type.

```go
for _, addr := range addrs {
    if ipnet, ok := addr.(*net.IPNet); ok {
        if ipnet.IP.To4() != nil {  // IPv4 only
            ips = append(ips, ipnet.IP.String())
        }
    }
}
```

That `addr.(*net.IPNet)` is a type assertion — "try to extract this as a `*net.IPNet`". It returns two values: the converted value and a boolean indicating success.

This is like Python's `isinstance()` check, but it also does the conversion in one step. If the assertion fails, `ok` is false and we skip it.

## Environment Variable Filtering

The system endpoint also exposes environment variables — but not all of them. Dumping `os.Environ()` would leak secrets.

```go
func getFilteredEnvVars() map[string]string {
    allowed := []string{
        "PORT", "DB_PATH", "HOSTNAME",
        "POD_NAME", "POD_NAMESPACE", "NODE_NAME",
    }

    result := make(map[string]string)
    for _, key := range allowed {
        if val := os.Getenv(key); val != "" {
            result[key] = val
        }
    }
    return result
}
```

An allowlist approach: only expose what's explicitly safe. The `make()` call is required — `var m map[string]string` creates a nil map that panics when you try to write to it.

## The API in Action

```bash
# Create an item
curl -X POST http://localhost:8080/api/items \
  -H "Content-Type: application/json" \
  -d '{"name":"Demo Item","description":"Created during Phase 2"}'

# Inject display data (like Terraform output)
curl -X POST http://localhost:8080/api/display \
  -H "Content-Type: application/json" \
  -d '{"region":"us-east-1","instance_count":3}'

# Check system info
curl http://localhost:8080/api/system
# {"hostname":"pop-os","ips":["192.168.1.100","172.17.0.1"],"environment":{}}
```

## Phase 2 Complete

The backend is done:
- Full CRUD API with SQLite persistence
- Display panel for arbitrary JSON injection
- System info endpoint for deployment verification
- All endpoints producing structured JSON logs

Next up: Phase 3, where we build a frontend dashboard to visualize all of this. Vanilla JavaScript, no frameworks — same "learn the fundamentals" philosophy.
