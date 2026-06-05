# lighthouse-ri-scale

Ansible-based deployment of a [Lighthouse](https://github.com/go-oidfed/lighthouse)
**OpenID Federation Trust Anchor** for the RI / e-Infra AAI federation.

> **Live instance:** `https://trust-anchor.dep.dev.rciam.grnet.gr`

---

## Stack

| Component | Version | Role |
|---|---|---|
| [Lighthouse](https://github.com/go-oidfed/lighthouse) | `v0.20.3` (pinned) | OpenID Federation engine — Trust Anchor, statement signer, resolver |
| [PostgreSQL](https://www.postgresql.org/) | `16-alpine` | Primary storage (entities, signing keys, statistics) |
| [Caddy](https://caddyserver.com/) | `2-alpine` | TLS termination, automatic Let's Encrypt, reverse proxy |
| Docker Compose | — | Container orchestration |
| Ansible | ≥ 2.15 | Automated, idempotent deployment |

---

## Architecture

```
Internet  :443
    │
    ▼
  Caddy  (TLS, Let's Encrypt, rate limiting)
    │  :7672 (Lighthouse main — federation API)
    ▼
  Lighthouse v0.20.3
    │  :5432 (internal)
    ▼
  PostgreSQL 16

  Admin API on 127.0.0.1:7673  ← SSH tunnel only, never exposed publicly
  /enroll                       ← blocked by Caddy (403)
  /api/v1/admin/*               ← blocked by Caddy (403, defense-in-depth)
```

Signing keys are stored on the host filesystem (`/opt/lighthouse-ta/lighthouse/data/signing`)
and inside PostgreSQL (`pk_backend: db`). Caddy obtains and renews TLS certificates automatically.

---

## Repository layout

```
ansible/
  deploy.yml                 # main playbook
  inventory.ini              # target host(s)
  requirements.yml           # Ansible Galaxy dependencies
  group_vars/
    trust_anchors.yml        # deployment variables (entity_id, image, postgres config, etc.)
    vault.yml                # Ansible Vault — secrets (gitignored, see vault.yml.example)
    vault.yml.example        # template for vault.yml
  roles/lighthouse/
    tasks/main.yml           # install Docker, create dirs, render config, (re)start stack
    templates/
      config.yaml.j2         # Lighthouse config template
      docker-compose.yml.j2  # Compose template (postgres + lighthouse + caddy)
      Caddyfile.j2           # Caddy config template
      env.j2                 # .env template (credentials, mode 0600)
caddy/
  Caddyfile                  # static reference — rendered from Caddyfile.j2 on deploy
lighthouse/
  config.yaml                # static reference — rendered from config.yaml.j2 on deploy
  data/                      # signing keys (gitignored, created on deploy)
docker-compose.yml           # static reference — rendered from docker-compose.yml.j2 on deploy
```

---

## Prerequisites

| Where | Requirement |
|---|---|
| Control node | Ansible ≥ 2.15 (`pip install ansible`) |
| Control node | `community.docker` Ansible collection |
| Control node | SSH key with `sudo` access to the target VM |
| Target VM | Debian 12 x86_64, ports 22 / 80 / 443 open |

> Docker is installed automatically by the playbook — no manual prep on the VM needed.

---

## Deploy

### 1. Configure Ansible Vault (first time only)

The database password is stored in an Ansible Vault file and is never written to
any config file in plain text. It is injected into `.env` on the host (mode `0600`).

```bash
cp ansible/group_vars/vault.yml.example ansible/group_vars/trust_anchors/vault.yml
# Edit vault.yml — set vault_postgres_password to a strong random string:
#   python3 -c "import secrets; print(secrets.token_urlsafe(32))"
ansible-vault encrypt ansible/group_vars/trust_anchors/vault.yml
```

### 2. Review deployment variables

Edit `ansible/group_vars/trust_anchors.yml` to confirm:

```yaml
lighthouse_entity_id: "https://trust-anchor.dep.dev.rciam.grnet.gr"
lighthouse_image: "oidfed/lighthouse:0.20.3"
deploy_dir: "/opt/lighthouse-ta"
```

Edit `ansible/inventory.ini` with the VM's hostname or IP.

### 3. Install Ansible dependencies

```bash
ansible-galaxy collection install -r ansible/requirements.yml
```

### 4. Run the playbook

```bash
ansible-playbook -i ansible/inventory.ini ansible/deploy.yml \
  -u YOUR_SSH_USER \
  -e deploy_user=YOUR_SSH_USER \
  -e deploy_group=YOUR_SSH_USER \
  --ask-vault-pass
```

The playbook is **idempotent** — safe to re-run for updates and config changes.
Container restarts are triggered automatically when the rendered config changes.

### 5. Verify

```bash
# Entity Configuration (signed JWT)
curl -s https://trust-anchor.dep.dev.rciam.grnet.gr/.well-known/openid-federation | head -c 200

# List enrolled subordinates
curl -s https://trust-anchor.dep.dev.rciam.grnet.gr/list
```

---

## Admin API

Lighthouse 0.20.x exposes a REST Admin API on port `7673` for managing
subordinate entities, federation metadata, signing key rotation, and statistics.
This is the **primary management interface** — it replaces the legacy `lhcli`
CLI tool and the `/enroll` HTTP endpoint.

**Port 7673 is bound to loopback only** (`127.0.0.1:7673:7673`) and is
additionally blocked by Caddy on port 443. Access it exclusively via SSH tunnel.

> **Full runbook:** [ADMIN_API.md](ADMIN_API.md)

### Quick start

```bash
# 1. Open tunnel (keep open for the session)
ssh -L 7673:localhost:7673 YOUR_USER@trust-anchor.dep.dev.rciam.grnet.gr

# 2. Swagger UI — open in browser
#    http://localhost:7673/api/v1/admin/docs

# 3. Create the first admin user (no auth required for first user only)
curl -s -X POST http://localhost:7673/api/v1/admin/users \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"YOUR_STRONG_PASSWORD"}'

# 4. Enroll a subordinate
curl -s -u "admin:PASSWORD" -X POST \
  http://localhost:7673/api/v1/admin/subordinates \
  -H 'Content-Type: application/json' \
  -d '{"entity_identifier":"https://some-idp.example.org"}'

# 5. List subordinates
curl -s -u "admin:PASSWORD" \
  http://localhost:7673/api/v1/admin/subordinates | python3 -m json.tool

# 6. Remove a subordinate
curl -s -u "admin:PASSWORD" -X DELETE \
  "http://localhost:7673/api/v1/admin/subordinates/https%3A%2F%2Fsome-idp.example.org"
```

See [ADMIN_API.md](ADMIN_API.md) for the full API surface: users, metadata
management, lifetimes, JWT decoding, and the legacy `/enroll` endpoint.

---

## Federation endpoints (public)

| Endpoint | Description |
|---|---|
| `/.well-known/openid-federation` | Signed Entity Configuration JWT |
| `/list` | List of enrolled subordinate entity identifiers |
| `/fetch?sub=<entity_id>` | Signed Entity Statement for a specific subordinate |
| `/resolve?sub=<entity_id>&trust_anchor=<ta>` | Full trust chain resolution |

---

## Secrets summary

| Secret | Location | Vault key |
|---|---|---|
| PostgreSQL password | `ansible/group_vars/trust_anchors/vault.yml` (gitignored) | `vault_postgres_password` |
| Admin API password | Set manually after deploy (not stored in repo) | — |
| TLS private key | Managed by Caddy in `caddy/data/` (gitignored) | — |
| Signing keys | `lighthouse/data/signing/` (gitignored) | — |

---

## Upstream references

| Resource | URL |
|---|---|
| Lighthouse GitHub | <https://github.com/go-oidfed/lighthouse> |
| Lighthouse documentation | <https://go-oidfed.github.io/lighthouse/> |
| Configuration reference | <https://go-oidfed.github.io/lighthouse/config/> |
| Admin API reference | <https://go-oidfed.github.io/lighthouse/api/admin/> |
| Docker Hub image | <https://hub.docker.com/r/oidfed/lighthouse> |
