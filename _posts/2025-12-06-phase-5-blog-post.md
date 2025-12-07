---
title: "Music Graph Project: Production Deployment to GCP"
date: 2025-11-26
categories:
  - projects
  - devops
  - cloud
tags:
  - flask
  - music-graph
  - gcp
  - docker
  - terraform
  - nginx
  - ssl
---

***The application is up and running in an almost production ready way. I went with GCP for hosting after asking Claude for options. The reason for me was because I work with other CSPs (AWS and Azure) on daily basis. I don't have a great deal of exposure to GCP and I wanted to learn more about it. I found it interesting that Claude kept forgetting that we had to rebuild the docker container for the changes to take affect. I guess Claude missed the class on how a docker container is immutable / how layers work. Below is the post written by claude.***  

Phase 5 is complete - the application is now deployed to Google Cloud Platform and accessible at [https://music-graph.billgrant.io](https://music-graph.billgrant.io). This phase transformed a localhost prototype into a production service.

## Why Cloud Deployment

After Phase 4, the application was secure and functional, but only accessible on localhost. To make it usable by Aidan (and eventually others), we needed:
- Public accessibility (not just my local machine)
- Proper infrastructure (not just Flask's dev server)
- SSL/HTTPS (secure connections)
- Persistent storage (database survives restarts)
- Reliable uptime (cloud infrastructure, not homelab)

## Why GCP Over Homelab

The original plan was homelab deployment, but we pivoted to Google Cloud Platform for several reasons:

**Learning goals:**
- I wanted GCP experience (already proficient with AWS/Azure)
- Cloud deployment teaches Docker, Nginx, SSL setup without homelab complexity
- CI/CD infrastructure easier in the cloud

**Practical concerns:**
- Homelab crashes would disrupt the project
- Cloud has better uptime for user testing
- Can always migrate to homelab later (but why?)

**The decision:** Keep this app cloud-native, do homelab projects with apps that make sense there (home automation, media servers, etc.)

## Infrastructure as Code with Terraform

Instead of clicking through the GCP console, everything is defined in Terraform. This makes the infrastructure:
- **Repeatable** - spin up dev environment with one command
- **Version controlled** - infrastructure changes tracked in Git
- **Documented** - code explains what exists and why

**Key resources:**
```hcl
resource "google_compute_address" "music_graph_ip" {
  name   = "prod-music-graph-ip"
  region = var.region
}

resource "google_compute_instance" "music_graph" {
  name         = "prod-music-graph"
  machine_type = "e2-micro"  # Free tier eligible
  # ... configuration ...
}
```

**Static IP was critical** - without it, VM restarts would change the IP and break DNS. Terraform reserves a static IP and assigns it to the instance.

**Firewall rules** restrict access:
```hcl
resource "google_compute_firewall" "http" {
  allow {
    protocol = "tcp"
    ports    = ["80"]
  }
  source_ranges = var.allowed_ips  # Only my home IP
}
```

For now, only my home IP can access the site. This provides basic security while using simple authentication.

## Dockerization

The application runs in Docker containers managed by docker-compose.

**Two services:**
- **PostgreSQL** - Database (migrated from SQLite for production)
- **Flask App** - Web application

**Why PostgreSQL over SQLite:**
- Better for production (concurrent users, reliability)
- Proper database backups
- Cloud SQL migration path (future)

**docker-compose.prod.yml:**
```yaml
services:
  db:
    image: postgres:16-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped
    
  web:
    build: .
    ports:
      - "127.0.0.1:5000:5000"
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped
```

**Database persistence was tricky.** Initially, the app ran `init_db.py` on every startup, which does `db.drop_all()` - wiping all data! 

**Solution - conditional initialization:**
```bash
# Check if tables exist
python -c "
from app import app
from models import db, Genre
with app.app_context():
    try:
        Genre.query.first()  # If this works, DB is initialized
        exit(0)
    except:
        exit(1)  # Need to initialize
"

if [ $? -eq 1 ]; then
    python init_db.py
fi
```

Now data persists across container restarts.

## Nginx Reverse Proxy

Flask's development server isn't production-ready. Nginx sits in front of Flask and:
- Handles SSL termination
- Serves as reverse proxy
- Adds security headers
- Could serve static files directly (future optimization)

**Configuration:**
```nginx
server {
    listen 80;
    server_name music-graph.billgrant.io;
    
    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        # ... security headers ...
    }
}
```

Nginx forwards requests to Flask on localhost:5000, then returns responses to clients.

## SSL with Let's Encrypt

HTTPS is mandatory for production apps. Let's Encrypt provides free SSL certificates.

**Setup with Certbot:**
```bash
sudo certbot --nginx -d music-graph.billgrant.io
```

Certbot automatically:
- Validates domain ownership (HTTP challenge)
- Gets certificate from Let's Encrypt
- Updates Nginx configuration
- Sets up auto-renewal

**Challenge:** The firewall was IP-restricted, blocking Let's Encrypt's validation servers.

**Temporary fix:** Opened firewall for initial cert, then locked back down.

**Future fix (Phase 6):** Use DNS-01 validation with Route53 plugin - validates via DNS TXT records instead of HTTP, works with firewall restrictions.

## Secrets Management

Production requires proper secrets handling. No hardcoded passwords in code.

**Environment variables in `.env.prod`:**
```bash
DATABASE_URL=postgresql://musicgraph:SECURE_PASSWORD@db:5432/musicgraph
SECRET_KEY=RANDOM_64_CHAR_HEX_STRING
POSTGRES_PASSWORD=SECURE_PASSWORD
```

**Generated secure secrets:**
```bash
python3 -c 'import secrets; print(secrets.token_hex(32))'
```

**Critical:** `.env.prod` is in `.gitignore` - never committed to Git.

## Deployment Workflow

**Current process:**
1. Develop locally
2. Commit and push to GitHub
3. SSH to GCP VM
4. `git pull origin main`
5. `docker-compose -f docker-compose.prod.yml up -d --build`

**Manual but works.** Phase 6 will add CI/CD for automated deployments.

## Production Readiness Checklist

Cleaned up the application for user access:

**✅ Security:**
- HTTPS only (Certbot auto-redirects HTTP)
- IP-restricted access
- Secure password hashing
- Environment variables for secrets

**✅ Reliability:**
- Database persistence (data survives restarts)
- Container auto-restart (`restart: unless-stopped`)
- Static IP (DNS won't break)

**✅ User Experience:**
- Removed debug UI elements (genre/band list cards)
- Clean interface (just the graph)
- Admin-only CRUD operations

**⏳ Still needed (Phase 6):**
- Automated deployments (CI/CD)
- Database backups
- Monitoring and logging
- Dev environment
- Certbot auto-renewal fix

## Cost Analysis

**Current monthly cost: ~$6-8**

- Compute Engine (e2-micro): ~$6/month
- Static IP: $0 (free while attached to running instance)
- Bandwidth: Minimal (free tier covers it)

GCP's free tier includes $300 credit for 90 days, so actually $0 for now.

## What I Learned

**Terraform workflow:**
- Infrastructure as code prevents configuration drift
- Static resources (IPs) need explicit allocation
- Variables make multi-environment easy

**Docker in production:**
- Multi-container apps need docker-compose
- Conditional initialization prevents data loss
- Health checks ensure dependencies are ready

**Nginx configuration:**
- Reverse proxy pattern is standard for production
- SSL termination at proxy simplifies application code
- Security headers add defense-in-depth

**GCP specifics:**
- Compute Engine vs managed services trade-offs
- Firewall tags for flexible security groups
- Startup scripts automate instance configuration

## Current State

The application is live at https://music-graph.billgrant.io with:
- Dockerized Flask + PostgreSQL
- Terraform-managed GCP infrastructure
- Nginx reverse proxy with SSL
- IP-restricted access
- Production-ready authentication
- Clean user interface

Aidan can now access the application remotely and start adding bands.

## What's Next: Phase 6

**CI/CD and DevOps:**
- GitHub Actions for automated testing and deployment
- Separate dev environment (testing without breaking prod)
- Database backup strategy
- Monitoring and logging (know when things break)
- Fix certbot auto-renewal for IP-restricted setup

The goal: develop and deploy new features without disrupting Aidan's usage.

## Code

This release is tagged as [v0.4.0-alpha](https://github.com/billgrant/music-graph/releases/tag/v0.4.0-alpha).

---

*This is part of the [Music Genre Graph project series](https://billgrant.io/tags/#music-graph). See the [project introduction](https://billgrant.io/2025/11/24/music-graph-intro/) for the full roadmap.*
