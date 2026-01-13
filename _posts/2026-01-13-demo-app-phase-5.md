---
title: "Demo App: Phase 5 - Building My First Terraform Provider"
date: 2026-01-13
categories:
  - projects
  - go
tags:
  - demo-app
  - go
  - terraform
  - learning-in-public
---

***I have been using Terraform for the better part of 10 years. First as a practitioner, today I use it as a Solutions Engineer helping out HashiCorp customers. I have always wanted to create my own provider to understand more about how the sausage is made. My lack of Go understanding has always held me back. After working on this Demo App project for a while I felt comfortable enough to give it a go. With Claude as my teacher explaining everything we wrote line by line, we got it done and I feel like I have a better understanding of how Terraform works under the hood. I will let Claude take it from here.***

---

## Why Build a Provider?

Demo App is intentionally stateless — it uses an in-memory SQLite database that resets on restart. Great for demos, but what if the app crashes mid-presentation?

The solution: **Terraform becomes the persistence layer.**

```bash
# App crashed? No problem.
./demo-app &
terraform apply
# Your demo data is restored from Terraform state
```

The provider manages two resources:
- `demoapp_item` — CRUD for inventory items
- `demoapp_display` — Posts arbitrary JSON to the display panel

## How Providers Actually Work

After years of using providers, I finally learned what's happening behind `terraform apply`:

```
┌─────────────────┐         gRPC          ┌─────────────────┐
│  Terraform CLI  │ ◄──────────────────► │    Provider     │
│  (Core)         │                       │  (Your Code)    │
└─────────────────┘                       └─────────────────┘
```

**Terraform Core** and **Provider** are separate processes communicating over gRPC. Core handles state, dependency graphs, and the plan/apply lifecycle. The provider is just a "driver" that answers questions:

- "What resources do you support?"
- "What's the schema for `demoapp_item`?"
- "Create this resource with these attributes"
- "Read the current state of this resource"

The best mental model: **Providers are like printer drivers.** The OS (Core) doesn't know how to talk to your specific printer. The driver (Provider) speaks both the OS interface and the printer's protocol.

## The Plugin Framework

HashiCorp provides two SDKs for building providers:
- **SDKv2** — The original, being phased out
- **Plugin Framework** — Modern approach, what we used

The framework handles all the gRPC plumbing. You implement interfaces:

```go
type Resource interface {
    Metadata(ctx, req, resp)  // "What's your name?"
    Schema(ctx, req, resp)    // "What attributes do you have?"
    Create(ctx, req, resp)    // terraform apply (new resource)
    Read(ctx, req, resp)      // terraform plan (check current state)
    Update(ctx, req, resp)    // terraform apply (changed resource)
    Delete(ctx, req, resp)    // terraform destroy
}
```

Each CRUD method follows the same pattern:
1. Read from Terraform (plan or state)
2. Make HTTP request to the API
3. Write back to Terraform state

## Key Implementation Details

### Two Data Models

Terraform and REST APIs speak different languages:

```go
// Terraform model — handles null/unknown states
type ItemResourceModel struct {
    ID          types.String `tfsdk:"id"`
    Name        types.String `tfsdk:"name"`
    Description types.String `tfsdk:"description"`
}

// API model — plain Go types for JSON
type itemAPIModel struct {
    ID          int    `json:"id"`
    Name        string `json:"name"`
    Description string `json:"description"`
}
```

Why `types.String` instead of `string`? Terraform values can be:
- **Set** — user provided a value
- **Null** — user didn't provide it (optional)
- **Unknown** — "(known after apply)"

Go's `string` can only be one thing. The framework types handle all three.

### Handling Drift

What if someone deletes a resource outside Terraform?

```go
func (r *ItemResource) Read(ctx, req, resp) {
    // ... make HTTP request ...

    if httpResp.StatusCode == http.StatusNotFound {
        // Tell Terraform: this resource no longer exists
        resp.State.RemoveResource(ctx)
        return
    }

    // ... update state with current values ...
}
```

Next `terraform plan` will show it needs to be recreated.

### Provider Configuration

Users configure the endpoint in their provider block or via environment variable:

```hcl
provider "demoapp" {
  endpoint = "http://localhost:8080"
}
```

```go
func (p *DemoAppProvider) Configure(ctx, req, resp) {
    endpoint := os.Getenv("DEMOAPP_ENDPOINT")

    if !config.Endpoint.IsNull() {
        endpoint = config.Endpoint.ValueString()
    }

    // Create HTTP client, pass to resources
    client := &DemoAppClient{
        HTTPClient: &http.Client{Timeout: 30 * time.Second},
        Endpoint:   endpoint,
    }
    resp.ResourceData = client
}
```

## The Result

```hcl
terraform {
  required_providers {
    demoapp = {
      source = "billgrant/demoapp"
    }
  }
}

provider "demoapp" {
  endpoint = "http://localhost:8080"
}

resource "demoapp_item" "web_server" {
  name        = "Web Server"
  description = "Provisioned by Terraform"
}

resource "demoapp_display" "status" {
  data = jsonencode({
    provisioned_by = "terraform"
    timestamp      = timestamp()
  })
}
```

All CRUD operations work:

| Operation | Test | Result |
|-----------|------|--------|
| Create | `terraform apply` | Resources created in demo-app |
| Read | `terraform plan` | Drift detection works |
| Update | Change attribute | In-place update |
| Delete | Remove from config | Deleted from API |

## What I Learned

1. **Providers are simpler than they look** — It's just HTTP calls wrapped in a framework interface

2. **The framework does the heavy lifting** — gRPC, state management, plan diffing are all handled

3. **Separation of concerns matters** — Keep Terraform models and API models separate

4. **Go patterns clicked** — Factory functions, context propagation, interface checks all make more sense now

## Next Steps

The provider works locally with dev overrides. Phase 8 (CI/CD) will add:
- GPG signing for releases
- GitHub Actions workflow
- Publishing to registry.terraform.io

Then anyone can use it with just:

```hcl
source = "billgrant/demoapp"
```

---

The full provider code is at [github.com/billgrant/terraform-provider-demoapp](https://github.com/billgrant/terraform-provider-demoapp).
