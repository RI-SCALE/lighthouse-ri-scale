# Lighthouse Admin API Runbook

> **Lighthouse version:** 0.20.x  
> **Admin API base URL (via SSH tunnel):** `http://localhost:7673/api/v1/admin`  
> **Interactive docs (Swagger UI):** `http://localhost:7673/api/v1/admin/docs`

The Admin API is the primary management interface for Lighthouse 0.20.x — it
replaces the old `lhcli` CLI tool and the legacy `/enroll` HTTP endpoint for
day-to-day operations.

---

## 1. Access — SSH Tunnel

Port 7673 is bound to the VM's loopback interface only and is additionally
blocked by Caddy on port 443 as defense-in-depth. The only way to reach it
is through an SSH tunnel.

**Open the tunnel** (keep this terminal open for the entire session):

```bash
ssh -L 7673:localhost:7673 -i ~/.ssh/trust_anchor_ed25519 \
    user.name@trust-anchor.dep.dev.rciam.grnet.gr
```

With an `~/.ssh/config` entry this shortens to:

```
Host trust-anchor
    HostName trust-anchor.dep.dev.rciam.grnet.gr
    User user.name
    IdentityFile ~/.ssh/trust_anchor_key
```

```bash
ssh -L 7673:localhost:7673 trust-anchor
```

Verify the tunnel is working:

```bash
curl -s -o /dev/null -w '%{http_code}' http://localhost:7673/api/v1/admin/docs
# Expected: 200
```

---

## 2. Swagger UI

With the tunnel open, browse to:

```
http://localhost:7673/api/v1/admin/docs
```

Click **Authorize** in the top-right corner, enter your username and password,
and explore or test any endpoint interactively.

---

## 3. First-Time Setup — Create the Admin User

No authentication is required to create the **very first** user.  After the
first user is created, every subsequent API call requires HTTP Basic Auth.

```bash
curl -s -X POST http://localhost:7673/api/v1/admin/users \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"YOUR_STRONG_PASSWORD"}'
# Expected: 201 Created
```

> Store the password in a secrets manager (Bitwarden, 1Password, etc.) —
> the Admin API password is never stored in the repository or Ansible Vault.

---

## 4. Users

All calls require Basic Auth after the first user is created.

```bash
CREDS="admin:YOUR_ADMIN_PASSWORD"

# List users
curl -s -u "$CREDS" http://localhost:7673/api/v1/admin/users | python3 -m json.tool

# Create another admin user
curl -s -u "$CREDS" -X POST http://localhost:7673/api/v1/admin/users \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin2","password":"another-strong-password"}'

# Change a user's password
curl -s -u "$CREDS" -X PUT http://localhost:7673/api/v1/admin/users/admin \
  -H 'Content-Type: application/json' \
  -d '{"password":"new-strong-password"}'

# Delete a user
curl -s -u "$CREDS" -X DELETE http://localhost:7673/api/v1/admin/users/admin2
```

---

## 5. Subordinate Entities

### List

```bash
curl -s -u "$CREDS" \
  http://localhost:7673/api/v1/admin/subordinates | python3 -m json.tool
```

### Enroll a new subordinate

Lighthouse fetches the subordinate's Entity Configuration from their
`/.well-known/openid-federation`, verifies it, and stores the JWKS in the
database. No downtime; the server keeps running.

```bash
curl -s -u "$CREDS" \
  -X POST http://localhost:7673/api/v1/admin/subordinates \
  -H 'Content-Type: application/json' \
  -d '{"entity_identifier":"https://some-idp.example.org"}'
# Expected: 201 Created
```

**Common error responses:**

| HTTP | Meaning |
|------|---------|
| `400` | Could not fetch or verify the subordinate's Entity Configuration — check their `/.well-known/openid-federation` is reachable and returns a valid JWT |
| `409` | Already enrolled — use a re-enroll if they rotated their keys |

### Inspect one subordinate

```bash
# URL-encode the entity identifier
curl -s -u "$CREDS" \
  "http://localhost:7673/api/v1/admin/subordinates/https%3A%2F%2Fsome-idp.example.org" \
  | python3 -m json.tool
```

### Re-enroll (after subordinate key rotation)

`POST` is idempotent — it updates an existing record if one exists:

```bash
curl -s -u "$CREDS" \
  -X POST http://localhost:7673/api/v1/admin/subordinates \
  -H 'Content-Type: application/json' \
  -d '{"entity_identifier":"https://some-idp.example.org"}'
```

### Remove a subordinate

```bash
# URL-encode the entity identifier in the path
curl -s -u "$CREDS" -X DELETE \
  "http://localhost:7673/api/v1/admin/subordinates/https%3A%2F%2Fsome-idp.example.org"
# Expected: 204 No Content
```

---

## 6. Entity Configuration

The Entity Configuration is the signed JWT served at
`/.well-known/openid-federation`.  Its metadata, lifetime, and signing
parameters are managed through the Admin API.

### View current configuration

```bash
curl -s -u "$CREDS" \
  http://localhost:7673/api/v1/admin/entity-configuration | python3 -m json.tool
```

### Update federation_entity metadata

```bash
curl -s -u "$CREDS" \
  -X PUT http://localhost:7673/api/v1/admin/entity-configuration/metadata/federation_entity \
  -H 'Content-Type: application/json' \
  -d '{
    "organization_name": "GRNET – Greek Research and Technology Network",
    "display_name":      "RI Federation Trust Anchor",
    "contacts":          ["aai-support@grnet.gr"],
    "homepage_uri":      "https://aai.grnet.gr"
  }'
```

### Set entity configuration lifetime

How long the signed Entity Configuration JWT is valid before clients must
re-fetch it:

```bash
curl -s -u "$CREDS" \
  -X PUT http://localhost:7673/api/v1/admin/entity-configuration/lifetime \
  -H 'Content-Type: application/json' \
  -d '"1d"'
```

### Set subordinate statement lifetime

How long the Trust Anchor's signed statements about subordinates are valid:

```bash
curl -s -u "$CREDS" \
  -X PUT http://localhost:7673/api/v1/admin/statement-lifetime \
  -H 'Content-Type: application/json' \
  -d '"7d"'
```

---

## 7. Verify the Trust Anchor (public endpoints, no tunnel)

These endpoints are served by Caddy on port 443 and are publicly accessible:

```bash
BASE="https://trust-anchor.dep.dev.rciam.grnet.gr"

# Signed Entity Configuration JWT
curl -s "${BASE}/.well-known/openid-federation" | head -c 200

# List enrolled subordinate entity identifiers
curl -s "${BASE}/list"

# Fetch a signed statement for a specific subordinate
curl -s "${BASE}/fetch?sub=https://some-idp.example.org"

# Resolve full trust chain end-to-end
curl -s "${BASE}/resolve?sub=https://some-idp.example.org&trust_anchor=${BASE}"
```

The `/resolve` call is the definitive end-to-end test — `200 OK` with a JWT
means the full chain from the subordinate to the Trust Anchor is valid.

---

## 8. Decode the Entity Configuration JWT

```bash
curl -s https://trust-anchor.dep.dev.rciam.grnet.gr/.well-known/openid-federation \
  | python3 -c "
import sys, json, base64
t = sys.stdin.read().strip()
p = t.split('.')[1]; p += '=' * (-len(p) % 4)
print(json.dumps(json.loads(base64.urlsafe_b64decode(p)), indent=2))
"
```

Expected top-level fields: `iss`, `sub` (both = entity_id), `jwks`,
`metadata.federation_entity.*` with all endpoint URLs.

---

## 9. Legacy enrollment endpoint (`/enroll`)

> **This is a legacy interface retained for backward compatibility.**  
> Prefer the Admin API (§5 above) for all new workflows.

`/enroll` is a simple HTTP GET endpoint that fetches and enrolls a
subordinate in a single call.  It is blocked by Caddy on port 443 (HTTP 403)
and is reachable only via an SSH tunnel to the **main server port 7672**.

```bash
# Open a tunnel to the main server port (separate from the Admin API tunnel)
ssh -L 7672:localhost:7672 trust-anchor

# Enroll in a second terminal
curl -i "http://localhost:7672/enroll?sub=https://some-idp.example.org"
# Expected: 201 Created
```

**When to use `/enroll` vs the Admin API:**

| | Admin API (7673) | `/enroll` (7672) |
|---|---|---|
| Authentication | HTTP Basic Auth | None (admin access = open tunnel) |
| Enroll | `POST /api/v1/admin/subordinates` | `GET /enroll?sub=...` |
| Remove | `DELETE /api/v1/admin/subordinates/{id}` | Not available |
| Metadata mgmt | Full CRUD | Not available |
| Recommended for | All new operations | Quick one-off enroll |
