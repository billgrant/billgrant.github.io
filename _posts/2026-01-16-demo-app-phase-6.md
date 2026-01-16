---
title: "Demo App: Phase 6 - From SQLite to BadgerDB (The Demo for the Demo)"
date: 2026-01-16
categories:
  - projects
  - go
tags:
  - demo-app
  - go
  - terraform
  - badgerdb
  - learning-in-public
---


***We used the Terraform provider we created in Phase 5 to create a baseline demo. I have been calling it the "demo for the demo". The provider introduced a database locking issue. We were running SQLite in memory which followed the stateless design principles of this demo app. However Terraform does writes in parallel, this caused TF applies to fail. We were going to set up a WAL, but this added state to our stateless app. We ended up refactoring and moving to BadgerDB. I learned a bit about how this K/V type of database worked and I think it is the right database design for this app. Claude will go into more detail about the session below.***

---

## The Concurrency Problem

Phase 5 worked great for single-threaded operations. But Terraform creates resources in parallel by default. When I added 10 items to the example Terraform config and ran `terraform apply`, two consistently failed:

```
Error: failed to create item: 500 Internal Server Error
```

The server logs showed the real issue:

```
database is locked
```

SQLite's in-memory mode (`:memory:`) doesn't support concurrent writes. Every parallel request fights for the same lock.

## Attempted Fixes

**Option A: WAL Mode + Busy Timeout**

SQLite's Write-Ahead Logging (WAL) mode handles concurrent writes better. We added:

```go
db.Exec("PRAGMA journal_mode=WAL")
db.Exec("PRAGMA busy_timeout=5000")
```

Result: Still failed. WAL mode doesn't work with `:memory:` databases — it requires a file on disk.

**Option B: File-based SQLite**

We could just use a file. But this changes the user experience:
- Binary runs differently than container (where does the file go?)
- Need to manage persistence/cleanup
- Breaks the "ephemeral by default" design

**Option C: Different Database**

BadgerDB is an embeddable key-value store written in Go. Key features:
- Native concurrent write support
- In-memory mode works identically to file mode
- No CGO dependencies (pure Go)
- Used in production by Dgraph

We chose BadgerDB.

## Key-Value vs SQL: A Mental Model Shift

SQL databases organize data in tables with rows and columns:

```sql
SELECT * FROM items WHERE id = 1;
-- Returns: id=1, name="Widget", description="A thing"
```

Key-value databases are simpler — just keys mapping to values:

```
"item:1" → {"id":1,"name":"Widget","description":"A thing"}
"item:2" → {"id":2,"name":"Gadget","description":"Another thing"}
```

No schema, no tables, no SQL. You define the key format and store JSON blobs.

## BadgerDB Implementation

### Initialization

```go
var db *badger.DB
var itemSeq *badger.Sequence

func initStore(dbPath string) (*badger.DB, error) {
    var opts badger.Options
    if dbPath == "" || dbPath == ":memory:" {
        opts = badger.DefaultOptions("").WithInMemory(true)
    } else {
        opts = badger.DefaultOptions(dbPath)
    }
    opts = opts.WithLoggingLevel(badger.WARNING)
    return badger.Open(opts)
}
```

The API is similar to SQLite — open with options, get a connection. In-memory vs file is just a flag.

### Auto-Increment IDs with Sequences

SQL has `AUTOINCREMENT`. BadgerDB has sequences:

```go
itemSeq, err = db.GetSequence([]byte("seq:items"), 100)

// In createItem:
id, err := itemSeq.Next()  // atomic, concurrent-safe
```

The second parameter (100) is the batch size — it pre-allocates IDs for performance. `Next()` is atomic, so concurrent creates get unique IDs.

### Writing Data

```go
func createItem(w http.ResponseWriter, r *http.Request) {
    // ... parse input ...

    id, _ := itemSeq.Next()
    item := Item{
        ID:          int64(id),
        Name:        input.Name,
        Description: input.Description,
        CreatedAt:   time.Now().UTC(),
    }

    value, _ := json.Marshal(item)
    key := []byte(fmt.Sprintf("item:%d", id))

    err = db.Update(func(txn *badger.Txn) error {
        return txn.Set(key, value)
    })
    // ...
}
```

The pattern:
1. Generate ID from sequence
2. Marshal struct to JSON
3. Build key with prefix (`item:`) + ID
4. `db.Update()` wraps a write transaction
5. `txn.Set(key, value)` stores the data

### Reading Data

```go
func getItem(w http.ResponseWriter, r *http.Request, idStr string) {
    id, _ := strconv.ParseInt(idStr, 10, 64)
    key := []byte(fmt.Sprintf("item:%d", id))

    var item Item
    err := db.View(func(txn *badger.Txn) error {
        dbItem, err := txn.Get(key)
        if err != nil {
            return err
        }
        return dbItem.Value(func(val []byte) error {
            return json.Unmarshal(val, &item)
        })
    })
    // ...
}
```

`db.View()` is a read-only transaction — multiple can run concurrently. The nested callback pattern (`Value(func(val []byte)...)`) is BadgerDB's way of managing memory safely.

### Listing All Items

Without SQL's `SELECT *`, we iterate over keys with a prefix:

```go
func listItems(w http.ResponseWriter, r *http.Request) {
    var items []Item

    err := db.View(func(txn *badger.Txn) error {
        opts := badger.DefaultIteratorOptions
        it := txn.NewIterator(opts)
        defer it.Close()

        prefix := []byte("item:")
        for it.Seek(prefix); it.ValidForPrefix(prefix); it.Next() {
            item := it.Item()
            err := item.Value(func(val []byte) error {
                var i Item
                json.Unmarshal(val, &i)
                items = append(items, i)
                return nil
            })
            if err != nil {
                return err
            }
        }
        return nil
    })
    // ...
}
```

`Seek(prefix)` jumps to the first matching key. `ValidForPrefix(prefix)` continues only while keys match. This is the K/V equivalent of a table scan.

## Testing Results

After the refactor:

| Test | SQLite | BadgerDB |
|------|--------|----------|
| 10 parallel creates | 2 failures | All succeed |
| `-parallelism=1` workaround | Required | Not needed |

No more database locks.

## The Demo for the Demo

With concurrency fixed, we built `demo-app-examples` — a repository showing how to use demo-app in real demos.

### Baseline Demo Architecture

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Docker         │     │  HTTP           │     │  DemoApp        │
│  Provider       │────►│  Provider       │────►│  Provider       │
│                 │     │  (data source)  │     │                 │
│  Creates        │     │  Fetches from   │     │  Posts to       │
│  container      │     │  /api/system    │     │  /api/display   │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

A single `terraform apply`:
1. Docker provider creates the demo-app container
2. Time provider waits for health check
3. HTTP provider fetches `/api/system` (hostname, IPs)
4. DemoApp provider posts that data to `/api/display`
5. DemoApp provider creates example items

Four providers working together, demonstrating the full ecosystem.

### Key Design Decision: Lifecycle Ignore Changes

```hcl
resource "demoapp_display" "system_snapshot" {
  data = jsonencode({
    source       = "terraform"
    captured_at  = timestamp()
    system_info  = jsondecode(data.http.system_info.response_body)
  })

  lifecycle {
    ignore_changes = [data]
  }
}
```

The `timestamp()` function generates a new value every time Terraform runs. Without `ignore_changes`, every `terraform plan` would show a diff.

But more importantly: **Terraform is the persistence layer.**

Demo-app is intentionally stateless — data lives only in memory. If the container crashes mid-presentation:
1. Restart the container (data is gone)
2. Run `terraform apply`
3. The *same* data is restored from Terraform state

Without `ignore_changes`, Terraform would fetch *new* system info and post *new* timestamps. With it, the original captured data is restored — exactly what you want when recovering a demo.

The default is opt-out. Engineers who want fresh data on every apply can remove the lifecycle block.

## What We Learned

1. **K/V databases are simpler than they seem** — No schema, no migrations. Just keys and JSON blobs. The tradeoff is you manage structure yourself.

2. **Concurrent writes need the right tool** — SQLite is great, but not for in-memory concurrent writes. BadgerDB handles this natively.

3. **Terraform as persistence** — A stateless app with Terraform managing state is a powerful pattern for demos. Crash recovery is just `terraform apply`.

4. **Multi-provider demos** — Showing four providers working together in one apply is more impressive than any single provider demo.

## Next Steps

The provider still uses dev overrides (not published). Phase 8 (CI/CD) will add:
- GitHub Actions for automated builds
- Multi-arch Docker images pushed to ghcr.io
- Provider publishing to registry.terraform.io

Then the baseline demo becomes `git clone` → `terraform init` → `terraform apply` with no prerequisites.

---

Code: [github.com/billgrant/demo-app](https://github.com/billgrant/demo-app)
Examples: [github.com/billgrant/demo-app-examples](https://github.com/billgrant/demo-app-examples)
Provider: [github.com/billgrant/terraform-provider-demoapp](https://github.com/billgrant/terraform-provider-demoapp)
