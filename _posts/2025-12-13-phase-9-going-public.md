---
title: "Music Graph Project: Going Public"
date: 2025-12-13
categories:
  - projects
  - python
tags:
  - flask
  - music-graph
  - gcp
  - terraform
  - security
  - secrets-management
---

***Today we took the site live. Some minor UI changes but mostly it was getting the backend production ready. As someone who works with Hashicorp Vault daily, I found it interesting to setup and learn GCP Secrets manager. We changed the release tags from alpha to beta which makes this a milestone. Read below to see what Claude has to say about it.***

---

## The Pivot: Phase 9 "Lite"

The original Phase 9 plan was ambitious: remote Terraform state, immutable infrastructure with Packer, Cloud SQL migration, and more. But with only two users and a desire to get the site publicly accessible, we split the plan:

- **Phase 9 (this phase)**: Minimum viable changes to go public safely
- **Phase 11 (future)**: Deep infrastructure modernization for learning

This pragmatic approach gets the site live without over-engineering for a two-user application.

## Production WSGI: Gunicorn

Flask's built-in server explicitly warns against production use. The fix was simple:

```bash
# Before (entrypoint.sh)
exec python app.py

# After
exec gunicorn --bind 0.0.0.0:5000 --workers 2 app:app
```

Two workers is conservative for an e2-micro instance (1GB RAM), but appropriate for our scale. Each Gunicorn worker is a separate process (~50-100MB), and we're sharing the VM with PostgreSQL.

The change only affects containerized deployments. Local development still uses `python app.py` with Flask's dev server and hot reloading.

## The Logout Bug

Users reported that non-admin users couldn't log out. The culprit:

```python
@app.route('/logout')
@admin_required  # Bug: should be @login_required
def logout():
    logout_user()
    ...
```

A simple decorator mistake. Non-admins were blocked from even reaching the logout function. The fix was one line, but it's a good reminder to test all user roles.

## Authentication Hardening

Before going public, we needed to lock down authentication:

### Hidden Login UI

Rather than prominent Login/Register buttons, we added a subtle "Admin" link in the footer. Aidan bookmarks the login URL directly. This isn't security through obscurity - it's just reducing noise for public visitors who have no reason to log in.

### Disabled Registration

Registration is disabled entirely:

```python
@app.route('/register', methods=['GET', 'POST'])
def register():
    flash('Registration is currently disabled.', 'error')
    return redirect(url_for('login'))
```

With only two admin users, public registration serves no purpose. When we're ready for public users (Phase 10+), we'll build it properly with email verification, 2FA, and role-based permissions.

### Rate Limiting

Flask-Limiter protects the login endpoint:

```python
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

limiter = Limiter(get_remote_address, app=app, storage_uri="memory://")

@app.route('/login', methods=['GET', 'POST'])
@limiter.limit("5 per minute")
def login():
    ...
```

Five attempts per minute per IP stops brute force attacks without affecting legitimate users. We're using in-memory storage for now; Redis is a future enhancement.

## GCP Secret Manager

This was the most significant change. Before Phase 9, secrets were handled poorly:

- Database passwords in docker-compose files
- Hardcoded `SECRET_KEY` fallbacks in code
- `.env` files created manually on VMs

### The New Architecture

Terraform creates secrets per environment:

```hcl
resource "random_password" "flask_secret_key" {
  length  = 64
  special = true
}

resource "google_secret_manager_secret" "flask_secret_key" {
  secret_id = "music-graph-${var.environment}-secret-key"
  replication { auto {} }
}

resource "google_secret_manager_secret_version" "flask_secret_key" {
  secret      = google_secret_manager_secret.flask_secret_key.id
  secret_data = random_password.flask_secret_key.result
}
```

The VM's service account gets IAM access to read these secrets.

### Fetching Secrets at Runtime

The entrypoint script fetches secrets before starting the app:

```bash
# Check if running in GCP
if curl -s -f -m 2 "http://metadata.google.internal/computeMetadata/v1/" \
    -H "Metadata-Flavor: Google" > /dev/null 2>&1; then

    # Get OAuth token from metadata server
    TOKEN=$(curl -s "http://metadata.google.internal/.../token" \
        -H "Metadata-Flavor: Google" | python3 -c "...")

    # Fetch secrets via Secret Manager API
    export SECRET_KEY=$(fetch_gcp_secret "music-graph-${ENV}-secret-key" ...)
    export POSTGRES_PASSWORD=$(fetch_gcp_secret "music-graph-${ENV}-db-password" ...)
else
    echo "Not running in GCP - using environment variables"
fi
```

Key design decisions:

1. **No gcloud CLI** - Uses raw REST API calls to avoid bloating the container
2. **Metadata server auth** - No service account keys to manage
3. **Graceful fallback** - Local development still works with docker-compose env vars
4. **Portable app code** - The Flask app just reads environment variables; it doesn't know about GCP

### Why Not Fetch in Application Code?

We considered using the `google-cloud-secret-manager` Python library directly in the app. We chose the entrypoint approach because:

- **Portability**: App code has no cloud-specific dependencies
- **Testability**: Local testing doesn't need GCP credentials or mocking
- **Flexibility**: Can swap secret backends (Vault, AWS, etc.) by changing only the entrypoint

This is the same pattern many organizations struggle with when adopting HashiCorp Vault - if secrets are fetched in application code, switching providers requires code changes. With infrastructure-layer injection, the app is agnostic.

## Firewall Split

The original Terraform used a single `allowed_ips` variable for all firewall rules. Opening HTTP/HTTPS to the public would have also opened SSH:

```hcl
# Before - single variable for all traffic
source_ranges = var.allowed_ips
```

We split this into two variables:

```hcl
# HTTP/HTTPS - public
source_ranges = var.allowed_web_ips  # ["0.0.0.0/0"]

# SSH - restricted
source_ranges = var.allowed_ssh_ips  # ["my.home.ip/32"]
```

Simple change, but critical for security. Future enhancement: HashiCorp Boundary for SSH access from any network.

## What's Next

Phase 9 achieves the goal: the site is publicly accessible at [music-graph.billgrant.io](https://music-graph.billgrant.io).

**Phase 10** will focus on UI enhancements:
- Detail panel showing all band/genre relationships
- Graph scalability improvements
- Base template pattern for cleaner HTML

**Phase 11** will tackle the deferred infrastructure work:
- Remote Terraform state
- Packer golden images
- Redis for rate limiting
- Automated database migrations

The "lite" approach worked well. We're live with proper security, and the deeper infrastructure learning can happen at a comfortable pace.

---

*This post is part of my [Music Graph Project](/tags/#music-graph) series, documenting the journey of building a full-stack application with AI assistance.*
