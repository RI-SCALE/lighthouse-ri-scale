# lighthouse-ri-scale

Ansible-based deployment of a [Lighthouse](https://github.com/go-oidfed/lighthouse)
**OpenID Federation Trust Anchor** for the RI / e-Infra AAI federation.

> **Live instance:** `https://trust-anchor.dep.dev.rciam.grnet.gr`

---

## What this is

This repository contains everything needed to deploy and operate a production-grade
OpenID Federation Trust Anchor. The stack is:

- **[Lighthouse](https://github.com/go-oidfed/lighthouse)** — the federation engine
  (Trust Anchor, entity statement signer, resolver)
- **[Caddy](https://caddyserver.com/)** — reverse proxy handling TLS termination
  and automatic certificate renewal via Let's Encrypt
- **Docker Compose** — container orchestration
- **Ansible** — automated, idempotent deployment recipe

```
Internet  :443
    │
    ▼
  Caddy  (TLS, Let's Encrypt)
    │  :7672 (internal)
    ▼
  Lighthouse  (OpenID Federation TA)
    ← admin access via SSH tunnel on 127.0.0.1:7672
```

---

## Repository layout

```
ansible/
  deploy.yml              # main playbook
  inventory.ini           # target host(s)
  group_vars/
    trust_anchors.yml     # deployment variables (entity_id, image tags, etc.)
  roles/lighthouse/
    tasks/main.yml        # install Docker, create dirs, render config, start stack
    templates/            # Jinja2 templates for config.yaml, Caddyfile, docker-compose.yml
caddy/
  Caddyfile               # static reference — rendered from templates/Caddyfile.j2 on deploy
lighthouse/
  config.yaml             # static reference — rendered from templates/config.yaml.j2 on deploy
  data/                   # signing keys + Badger DB (gitignored, created on deploy)
docker-compose.yml        # static reference — rendered from templates/docker-compose.yml.j2
setup.md                  # full operational guide
```

---

## Quick start

### Prerequisites

| Where | Requirement |
|---|---|
| Control node | Ansible ≥ 2.15 (`pip install ansible`) |
| Control node | `community.docker` collection (`ansible-galaxy collection install -r ansible/requirements.yml`) |
| Control node | SSH key with `sudo` access to the target VM |
| Target VM | Debian 12 x86_64, ports 22 / 80 / 443 open |

> Docker is installed automatically by the playbook — no manual prep on the VM needed.

### 1. Configure

Edit `ansible/group_vars/trust_anchors.yml`:

```yaml
lighthouse_entity_id: "https://your-domain.example.org"   # must match DNS + TLS cert
deploy_dir: "/opt/lighthouse-ta"
lighthouse_image: "oidfed/lighthouse:main"
```

Edit `ansible/inventory.ini` with your VM's hostname/IP.

### 2. Deploy

```bash
ansible-galaxy collection install -r ansible/requirements.yml

ansible-playbook -i ansible/inventory.ini ansible/deploy.yml \
    -u YOUR_SSH_USER \
    -e deploy_user=YOUR_SSH_USER \
    -e deploy_group=YOUR_SSH_USER \
    --private-key ~/.ssh/your_key
```

### 3. Verify

```bash
# Entity Configuration (self-signed JWT)
curl -s https://your-domain.example.org/.well-known/openid-federation

# List enrolled subordinates
curl -s https://your-domain.example.org/list

# Full trust chain resolution
curl -s "https://your-domain.example.org/resolve?sub=https://some-idp.example.org&trust_anchor=https://your-domain.example.org"
```

---

## Enrolling subordinates

Subordinate enrollment is done via the `/enroll` HTTP endpoint.

The endpoint is **blocked by Caddy on port 443**. Access it via SSH tunnel:

```bash
# 1. Open tunnel (keep open)
ssh -L 7672:localhost:7672 YOUR_USER@your-domain.example.org

# 2. Enroll (in another terminal)
curl -i "http://localhost:7672/enroll?sub=https://some-idp.example.org"
# Expected: HTTP 201 Created + signed Entity Statement JWT
```

See [`setup.md § 7`](setup.md#7-enrolling-subordinate-entities) for full details,
error reference, and the removal procedure.

---

## Documentation

Full operational documentation is in **[setup.md](setup.md)**:

- Architecture and design decisions
- First-deploy vs. re-deploy behaviour
- Signing key management and backup
- Public key extraction and distribution to federation members
- Subordinate enrollment and removal
- Key rotation procedure
- Operations, maintenance, and log access
- Hardening checklist
- Troubleshooting

---

## Upstream references

| Resource | URL |
|---|---|
| Lighthouse GitHub | <https://github.com/go-oidfed/lighthouse> |
| Lighthouse documentation | <https://go-oidfed.github.io/lighthouse/> |
| Configuration reference | <https://go-oidfed.github.io/lighthouse/config/> |
| Docker Hub image | <https://hub.docker.com/r/oidfed/lighthouse> |
