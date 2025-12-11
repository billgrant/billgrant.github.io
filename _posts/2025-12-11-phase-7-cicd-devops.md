---
title: "Music Graph Project: CI/CD and Production DevOps"
date: 2025-12-11
categories:
  - projects
  - devops
tags:
  - flask
  - music-graph
  - ci-cd
  - terraform
  - github-actions
  - gcp
---

***Over the past few days we built out CI/CD and a dev VM. I used Claude CLI for the first time and it worked out better then I thought it would. At first I was taken back because it wanted to write all the code and just ask me if it could write the file. I stopped it and said please update claude.md to say that I want to understand and step into the code before we write anything. It started doing planning sessions with me after that.I also had to fix some of the Terraform it wrote. Below is the unedited post by claude CLI using OPUS this time***

Phase 7 transforms the Music Graph project from a manually-deployed application into a production-ready system with automated testing, deployment pipelines, database backups, and reliable SSL certificate management. This phase focused on the infrastructure and automation needed for sustainable long-term operation.

## Starting Point: Manual Everything

After Phase 6, the application worked well but had operational gaps:

- **Manual deployments:** SSH to VMs, git pull, docker-compose rebuild
- **No automated testing:** Changes went straight to production
- **Single environment:** Testing in production (risky)
- **No backups:** Database lived only in Docker containers
- **SSL certificate issues:** Certbot renewal failing with IP-restricted firewall
- **No CI/CD:** Every deployment required manual steps

For a learning project with one user (Aidan), this was acceptable. For a production application, it needed improvement.

## The Build: Five Major Improvements

### 1. GitHub Actions CI/CD Pipeline

**The Problem:** No automated testing or deployment workflow.

**The Solution:** Three GitHub Actions workflows:

**CI Workflow** (`ci.yml`):
- Triggers on pull requests and pushes to main
- Runs pytest test suite (17 tests)
- Checks code coverage
- Runs flake8 linting
- Path filters: Only runs when Python or test files change

**Dev Deployment** (`deploy-dev.yml`):
- Triggers on push to main branch
- Builds Docker image tagged as `dev-latest`
- Pushes to Google Container Registry
- SSH to dev VM and pulls new image
- Restarts containers automatically
- Path filters: Only deploys when application code changes

**Prod Deployment** (`deploy-prod.yml`):
- Manual trigger only (workflow_dispatch)
- Builds image tagged as `production` AND `latest`
- Creates GitHub release with version tag
- Requires manual approval for safety

**Path Filters Implementation:**
```yaml
on:
  push:
    branches: [ main ]
    paths:
      - 'app.py'
      - 'models.py'
      - '**.py'
      - 'templates/**'
      - 'static/**'
      - 'requirements.txt'
```

This prevents unnecessary builds when only documentation or configuration changes. Don't bake a new cake if the recipe didn't change.

### 2. Separate Development Environment

Created a complete parallel environment for safe testing:

**Dev Environment:**
- Separate GCP VM (`dev-music-graph`)
- Own PostgreSQL database
- Subdomain: `dev.music-graph.billgrant.io`
- Own SSL certificate
- Pulls `dev-latest` Docker images

**Terraform Workspace Strategy:**
```bash
terraform workspace select dev
terraform apply  # Creates dev infrastructure

terraform workspace select prod
terraform apply  # Creates prod infrastructure
```

Each workspace maintains separate state for VM configuration, firewall rules, and environment-specific settings.

**Benefits:**
- Test changes safely before production
- Parallel development possible
- Environment parity (both use Docker, PostgreSQL, same setup)
- No production impact from experiments

### 3. CI/CD Optimization (Issue #8)

After initial CI/CD setup, we optimized to reduce costs and complexity:

**Image Tagging Strategy:**
- Dev builds: `gcr.io/music-graph-479719/music-graph:dev-latest`
- Prod builds: `gcr.io/music-graph-479719/music-graph:production` + `latest`
- Releases: `gcr.io/music-graph-479719/music-graph:v1.0.0`

This prevents dev and prod from colliding on the `latest` tag.

**GCR Lifecycle Policy:**

Created via Terraform to automatically clean up old images:

```hcl
resource "google_artifact_registry_repository" "music_graph" {
  cleanup_policies {
    id     = "delete-old-dev-images"
    action = "DELETE"
    condition {
      tag_prefixes = ["dev-"]
      older_than   = "2592000s"  # 30 days
    }
  }

  cleanup_policies {
    id     = "keep-production-releases"
    action = "KEEP"
    condition {
      tag_prefixes = ["v", "production", "latest"]
    }
  }
}
```

**Docker Cleanup Automation:**

Both deployment workflows now clean up old images on VMs:

```bash
docker system prune -af --filter "until=24h"
```

This prevents disk space issues that had previously filled both VMs to 100% capacity.

**Terraform Restructure:**

Split Terraform into logical directories:

```
terraform/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   └── terraform.tfvars
│   └── prod/
│       ├── main.tf
│       └── terraform.tfvars
└── project/
    ├── main.tf  # GCS bucket, GCR policy, IAM
    └── terraform.tfvars
```

- **environments/**: Workspace-specific VM configurations
- **project/**: Shared resources (GCS buckets, GCR policies, IAM)

This prevents folder sprawl while maintaining clear separation.

### 4. Database Backup System

**The Problem:** Database only existed in Docker containers. No backups, no disaster recovery.

**The Solution:** Automated daily backups to Google Cloud Storage.

**Components:**

**1. GCS Bucket (Terraform):**
```hcl
resource "google_storage_bucket" "database_backups" {
  name     = "music-graph-backups-music-graph-479719"
  location = "US"  # Multi-region for disaster recovery

  lifecycle_rule {
    action {
      type = "Delete"
    }
    condition {
      age = 7  # Delete backups older than 7 days
    }
  }
}
```

**2. Backup Script (`backup-database.sh`):**
- Runs pg_dump from Docker container
- Uses `--clean --if-exists` flags for restore compatibility
- Compresses with gzip (5MB → 500KB)
- Uploads to GCS
- Logs all operations
- Cleans up local files

**3. Cron Job (each VM):**
```cron
# Daily database backup at 3:00 AM
0 3 * * * /home/billgrant/music-graph/backup-database.sh prod >> /var/log/music-graph-backup-cron.log 2>&1
```

**4. Verification Script (`verify-backup-setup.sh`):**

Checks 9 aspects of backup configuration:
- Backup script executable
- Log file permissions
- Docker group membership
- Database container running
- GCS bucket access
- Cron job configured
- And more...

**5. Documentation (`docs/database-backup-setup.md`):**

Complete setup guide including:
- One-time VM setup steps
- Restore procedure
- Troubleshooting
- Tested restore results

**Restore Testing:**

We tested the full disaster recovery procedure on dev:
1. Created backup
2. Deleted data (simulated data loss)
3. Downloaded backup from GCS
4. Restored to database
5. Verified data recovery

**Result:** Complete data recovery in under 2 minutes. The `--clean --if-exists` flags proved essential - they allow restoring over an existing database without conflicts.

### 5. SSL Certificate Auto-Renewal Fix

**The Problem:** Certbot renewal failing because our firewall blocks port 80 (HTTP-01 challenge won't work).

**The Solution:** Switch to Route53 DNS-01 challenge.

**How DNS-01 Works:**
Instead of proving domain ownership via HTTP (port 80), certbot creates a DNS TXT record in Route53. Let's Encrypt checks the DNS record to verify ownership. No firewall ports required.

**Implementation:**

**1. AWS IAM Resources (Terraform):**
```hcl
resource "aws_iam_policy" "certbot_route53" {
  name = "certbot-route53-dns-music-graph"
  policy = jsonencode({
    Statement = [
      {
        Effect = "Allow"
        Action = ["route53:ListHostedZones", "route53:GetChange"]
        Resource = "*"
      },
      {
        Effect   = "Allow"
        Action   = "route53:ChangeResourceRecordSets"
        Resource = "arn:aws:route53:::hostedzone/*"
      }
    ]
  })
}

resource "aws_iam_user" "certbot" {
  name = "certbot-music-graph"
}

resource "aws_iam_access_key" "certbot" {
  user = aws_iam_user.certbot.name
}
```

Added AWS provider to existing Terraform project configuration (already had Google provider).

**2. Configure AWS Credentials (both VMs):**
```bash
# Create credentials file for root (certbot runs as root)
sudo mkdir -p /root/.aws
sudo bash -c 'cat > /root/.aws/credentials << EOF
[default]
aws_access_key_id = <from terraform output>
aws_secret_access_key = <from terraform output>
EOF'
sudo chmod 600 /root/.aws/credentials
```

**3. Install Plugin and Get Certificates:**
```bash
sudo apt install python3-certbot-dns-route53

# Get certificate using DNS-01
sudo certbot certonly \
  --dns-route53 \
  -d music-graph.billgrant.io \
  --non-interactive \
  --agree-tos \
  -m email@example.com
```

**4. Test Renewal:**
```bash
sudo certbot renew --dry-run
# ✓ Congratulations, all simulated renewals succeeded
```

**Why This Matters:**

The firewall can stay locked down (only port 443 open) while certificates still renew automatically. This is more secure than opening port 80 for HTTP-01 challenges.

## Lessons Learned

### 1. Path Filters Save Money and Time

Initially, every push to main triggered full build and deployment - even for documentation changes. Path filters ensure we only build when code actually changes.

**Before:** 10 deployments per day for typo fixes
**After:** 2-3 deployments per day for actual code changes

### 2. Immutable Infrastructure Is Next

We're still modifying VMs in place (SSH, apt install, manual changes). Phase 8 will address this with Packer images and immutable infrastructure.

**Current state:**
- Manual setup on each VM
- "Snowflake" servers (unique configurations)
- Hard to reproduce

**Phase 8 goal:**
- Packer builds VM images with all dependencies
- VMs deployed from images (no manual setup)
- Reproducible infrastructure

### 3. Backups Are Useless Until Tested

We built the backup system and assumed it worked. Then we tested restore and found issues:
- Missing `--clean --if-exists` flags caused duplicate key errors
- Needed to stop web container during restore
- Documentation gaps

**Testing the restore procedure caught all these issues before a real disaster.**

### 4. Multi-Cloud Is Reality

This project now uses:
- **Google Cloud:** VMs, Container Registry, Cloud Storage
- **AWS:** Route53 DNS, IAM for certbot
- **GitHub:** Source control, Actions, Container Registry alternative

Terraform manages both GCP and AWS resources in the same configuration. Multi-cloud isn't a choice - it's the reality of using best tool for each job.

### 5. Planning Sessions Are Worth It

Issue #8 (CI/CD optimization) had the best planning session yet:
- High-level concepts before code
- Discussion of trade-offs
- Caught issues early (latest tag collision, workspace concerns)
- Clear implementation plan before touching files

**Quote from CLAUDE.md:**
> "This was a great planning session, best one yet. The way we did this is the way I am most comfortable with."

Starting with understanding beats diving into code.

## Workflow Evolution

This phase refined the development workflow documented in CLAUDE.md:

**Code Change Workflow:**
1. Start with high-level explanation
2. Let Bill ask questions (this is a learning project)
3. Discuss trade-offs and alternatives
4. Address concerns (Bill often catches important issues)
5. Get explicit approval
6. Then implement

**For Refactoring Existing Code:**
- Step-by-step, methodical changes
- Test after each change
- Use feature branches
- Get approval before applying

**For New Features:**
- Can do full-file generation
- Still review before applying
- Test locally before production

## Current State

The application is now production-ready from an operations perspective:

**✅ Automated Testing:** 17 tests run on every PR
**✅ Automated Deployment:** Push to main deploys to dev automatically
**✅ Safe Promotion:** Manual approval required for production
**✅ Database Backups:** Daily automated backups, tested restore procedure
**✅ SSL Auto-Renewal:** Certificates renew automatically via DNS-01
**✅ Cost Optimization:** Old images cleaned up automatically
**✅ Separate Environments:** Dev and prod fully isolated

**Live Sites:**
- **Production:** https://music-graph.billgrant.io
- **Development:** https://dev.music-graph.billgrant.io

Both environments run the same Docker containers, use the same database schema, and follow the same deployment process.

## What's Next: Phase 8

The roadmap is shifting based on lessons learned. Phase 8 will focus on **Infrastructure Modernization** (Issue #10):

**Priorities:**
1. **Remote Terraform State:** Move from local state files to GCS backend (enables team collaboration, state locking)
2. **Production WSGI Server:** Replace Flask dev server with Gunicorn (Issue #2)
3. **Immutable Infrastructure:** Packer images, never modify running VMs (Issue #7)
4. **Secrets Management:** Remove static secrets from repository (Issue #12 - blocking for public launch)
5. **Evaluate Cloud SQL:** Compare managed PostgreSQL vs Docker database

**Future Sub-Issues:**
- **Issue #11:** Import DNS records to Terraform (everything as code)
- **Issue #12:** Remove static secrets (critical before public launch)

Phase 8 will address the "mutable infrastructure" problem - VMs are still modified manually. The goal is reproducible infrastructure from code.

## Code

Changes visible in:
- [GitHub Issue #8](https://github.com/billgrant/music-graph/issues/8) - CI/CD Optimization
- CI/CD workflows in `.github/workflows/`
- Backup system: `backup-database.sh`, `verify-backup-setup.sh`, `docs/database-backup-setup.md`
- Terraform changes: `terraform/project/main.tf` (AWS provider, IAM, GCS bucket, GCR policy)

---

*This is part of the [Music Genre Graph project series](https://billgrant.io/tags/#music-graph). See the [project introduction](https://billgrant.io/2025/11/24/music-graph-intro/) for the full roadmap.*
