---
title: "Demo App: From Idea to Published Terraform Provider"
date: 2026-01-27
categories:
  - go
  - projects
tags:
  - demo-app
  - go
  - terraform
  - learning-in-public
---

***A month ago I started learning Go by building a demo app. Today I have a published Terraform provider on registry.terraform.io. Here's what I built and what I learned.***

---

## What is Demo App?

As a Solutions Engineer, I constantly need something to deploy when demoing infrastructure tools. The app itself isn't the point — it's a vehicle to show that Terraform provisioned something, Vault injected secrets, or traffic is flowing through a network appliance.

Most options are either too simple (nginx default page) or too complex (real apps with dependencies). Demo App sits in the sweet spot:

- Single binary with embedded frontend
- REST API for CRUD operations
- Display panel that accepts arbitrary JSON
- System info panel showing hostname, IPs, environment
- Prometheus metrics endpoint
- Structured JSON logging

## Try It Yourself

The whole point is that this works with one command. If you have Terraform and Docker installed:

```bash
git clone https://github.com/billgrant/demo-app-examples.git
cd demo-app-examples/baseline
terraform init
terraform apply
```

Open http://localhost:8080 and you'll see:

![Demo App Baseline](/assets/images/demo-app-baseline.png)

That's it. Terraform spins up the container, waits for it to be healthy, then uses the DemoApp provider to populate data. If the app crashes, `terraform apply` restores everything from state.

## The Three Repos

| Repo | Purpose |
|------|---------|
| [demo-app](https://github.com/billgrant/demo-app) | The application itself |
| [terraform-provider-demoapp](https://github.com/billgrant/terraform-provider-demoapp) | Terraform provider for managing app resources |
| [demo-app-examples](https://github.com/billgrant/demo-app-examples) | Ready-to-run Terraform configurations |

## What I Learned About Providers

I've been using Terraform for almost 10 years and selling it for four. I always wanted to understand how providers actually work at the code level. Building one taught me:

**Providers are simpler than they look.** It's HTTP calls wrapped in a framework interface. The plugin framework handles all the gRPC plumbing — you just implement CRUD methods.

**The mental model that clicked:** Providers are like printer drivers. Terraform Core doesn't know how to talk to your specific API. The provider speaks both the Terraform interface and your API's protocol.

**What maintainers deal with:** Schema design, state drift handling, error messages, documentation for the registry. I have more appreciation for the providers I use daily.

## The Stack

Built entirely with Claude as my coding partner:

- **Go** — stdlib `net/http`, `log/slog`, `embed`
- **BadgerDB** — embedded K/V store (replaced SQLite for concurrent writes)
- **Docker Hardened Images** — security-first containers
- **GitHub Actions** — CI/CD for both repos
- **GoReleaser** — GPG-signed provider releases

## What's Next

The baseline demo works. Next up is expanding the example library:

- Vault integration (inject secrets into display panel)
- Observability stack (ship logs to Grafana/Loki)
- Multi-tier demo (two app instances talking to each other)
- MCP server (AI agent managing app state)

---

*This project is documented in the [DEVLOG](https://github.com/billgrant/demo-app/blob/main/DEVLOG.md) if you want the session-by-session breakdown of what I learned.*

*P.S. Claude helped me write this post.*
