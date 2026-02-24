# Ansible Homelab — Runbook (Full Stack)

Operational reference for the NAS core node.
For architecture deep dive → [`ARCHITECTURE.md`](ARCHITECTURE.md)
For quick deployment overview → [`README.md`](README.md)

---

## Table of Contents

- [If it's on fire (60 seconds)](#if-its-on-fire-60-seconds)
- [Scope](#scope)
- [Infrastructure](#infrastructure)
- [Common Commands](#common-commands)
- [Variable Map](#variable-map)
- [Common Issues & Solutions](#common-issues--solutions)
- [Incident Playbooks](#incident-playbooks)
- [Engineering Notes](#engineering-notes)
- [Ansible Gotchas](#ansible-gotchas)
- [Go-live Smoke Checklist](#go-live-smoke-checklist)


---

## If it's on fire (60 seconds)

Run these first to get a fast signal on container health, ingress/auth logs, DNS, and ACME:


```bash
# 1) Are the core containers up on the NAS?
ssh <user>@10.0.0.10 "sudo docker ps --format '{{.Names}}\t{{.Status}}' | egrep 'traefik|adguard|authentik|vaultwarden|portainer|jellyfin|homepage'"

# 2) Traefik: routing / ACME / forward auth errors
ssh <user>@10.0.0.10 "sudo docker logs --tail 200 traefik | tail -n 200"

# 3) Authentik: server/worker health
ssh <user>@10.0.0.10 "sudo docker logs --tail 200 authentik-server"
ssh <user>@10.0.0.10 "sudo docker logs --tail 200 authentik-worker"

# 4) DNS sanity (AdGuard)
dig +short authentik.<domain> @10.0.0.10

# 5) ACME sanity (DNS-01) — use your real registered domain here
dig TXT _acme-challenge.<domain> @1.1.1.1
```

---

## Scope

**Included in this profile:**
- Traefik
- AdGuard Home
- Authentik (+ LDAP outpost)
- Vaultwarden
- Portainer
- Jellyfin
- Homepage
- Docker host role on Ryzen
- Prod apps on Ryzen (Immich, Paperless-ngx, DB backups)
- SSH hardening role

---

## Infrastructure

| Node | IP | OS | Role | Ansible group |
|---|---|---|---|---|
| NAS | 10.0.0.10 | TrueNAS SCALE | Core + media services | `n100` |
| Ryzen VM | 10.0.0.30 | Debian (Proxmox VM) | Docker host + prod apps | `docker_nodes` |
| Proxmox | 10.0.0.20 | Proxmox VE | Hypervisor (hosts Ryzen VM) | *(unmanaged by Ansible)* |

> This public branch uses sanitized example values:
> - SSH target: `<user>@10.0.0.10`
> - Ryzen SSH target: `<user>@10.0.0.30`
> - `docker_config_dir`: `/mnt/example_apps/appdata/docker/config`
> - `docker_data_dir`: `/mnt/example_apps/appdata/docker/data`
> - Domain suffix `.homelab.local` is for documentation only; ACME DNS-01 requires a real registered domain.

---

## Common Commands

### Deploy

```bash
# Full deploy
ansible-playbook -i inventory.ini deploy_n100.yml
ansible-playbook -i inventory.ini deploy_docker_nodes.yml

# Single service (NAS)
ansible-playbook -i inventory.ini deploy_n100.yml --tags traefik
ansible-playbook -i inventory.ini deploy_n100.yml --tags adguard
ansible-playbook -i inventory.ini deploy_n100.yml --tags authentik
ansible-playbook -i inventory.ini deploy_n100.yml --tags vaultwarden
ansible-playbook -i inventory.ini deploy_n100.yml --tags portainer
ansible-playbook -i inventory.ini deploy_n100.yml --tags media

# Entire core role
ansible-playbook -i inventory.ini deploy_n100.yml --tags core

# Single service (Ryzen)
ansible-playbook -i inventory.ini deploy_docker_nodes.yml --tags immich
ansible-playbook -i inventory.ini deploy_docker_nodes.yml --tags paperless
ansible-playbook -i inventory.ini deploy_docker_nodes.yml --tags backups

# SSH hardening (both nodes)
ansible-playbook -i inventory.ini deploy_n100.yml --tags hardening
ansible-playbook -i inventory.ini deploy_docker_nodes.yml --tags hardening

# Dry run (always run before applying changes)
ansible-playbook -i inventory.ini deploy_n100.yml --check --diff
ansible-playbook -i inventory.ini deploy_docker_nodes.yml --check --diff
```

### Dependencies

```bash
ansible-galaxy collection install -r requirements.yml
```

### Vault

```bash
ansible-vault encrypt secrets.yml
ansible-vault edit secrets.yml
ansible-playbook -i inventory.ini deploy_n100.yml --ask-vault-pass
ansible-playbook -i inventory.ini deploy_n100.yml --vault-password-file ~/.vault_pass
```

### Diagnostics

```bash
ansible n100 -m ping
ansible docker_nodes -m ping
ansible-playbook -i inventory.ini deploy_n100.yml --list-tasks
ansible-playbook -i inventory.ini deploy_n100.yml --list-tags
ansible-playbook -i inventory.ini deploy_n100.yml --list-hosts
ansible-playbook -i inventory.ini deploy_docker_nodes.yml --list-tags
```

### Lint / Syntax

```bash
# Optional local tools (not installed by requirements.yml)
python3 -m pip install --user ansible-lint yamllint

ansible-playbook -i inventory.ini deploy_n100.yml --syntax-check
ansible-playbook -i inventory.ini deploy_docker_nodes.yml --syntax-check
ansible-lint deploy_n100.yml
ansible-lint deploy_docker_nodes.yml
yamllint .
```

---

## Variable Map

| File | Contains |
|---|---|
| `group_vars/all.yml` | Domains, NAS IP, shared ports/endpoints |
| `group_vars/n100.yml` | Paths (`docker_config_dir`, `docker_data_dir`), image tags, compatibility toggles |
| `group_vars/docker_nodes.yml` | Docker node SSH user, NFS source paths, mount points, prod app versions |
| `roles/prod_apps/defaults/main.yml` | Default stack layout for Immich/Paperless/backups |
| `secrets.yml` | `cert_email`, `cloudflare_token`, `authentik_*`, app DB passwords, widget tokens |

---

## Common Issues & Solutions

### 1) AdGuard Home fails to start — port 53 already in use

**Symptom:** `failed to bind host port 0.0.0.0:53/tcp: address already in use`

**Cause:** On Debian/Ubuntu hosts, `systemd-resolved` listens on `127.0.0.53:53`.
AdGuard binds `0.0.0.0:53` (all local addresses, including loopback), so bind fails.
This conflict does not apply to TrueNAS SCALE.

**Fix:**
```bash
sudo sed -i 's/^#\?DNSStubListener=.*/DNSStubListener=no/' /etc/systemd/resolved.conf
sudo systemctl restart systemd-resolved
ansible-playbook -i inventory.ini deploy_n100.yml --tags adguard
```

---

### 2) AdGuard Home returns 502 immediately after first deploy

**Symptom:** `https://adguard.homelab.local` returns 502 right after fresh deploy.

**Cause:** First boot uses setup wizard on `:3000`. Traefik forwards to `:80`. Wizard hasn't run yet.

**Fix:**
```yaml
# group_vars/n100.yml
adguard_setup_mode: true
```
```bash
ansible-playbook -i inventory.ini deploy_n100.yml --tags adguard
# Complete wizard at http://10.0.0.10:3000
# Then set adguard_setup_mode: false in group_vars/n100.yml
ansible-playbook -i inventory.ini deploy_n100.yml --tags adguard
```

> Default in this repo is `adguard_setup_mode: false`. Enable it only for first-run wizard, then disable and redeploy.

---

### 3) Traefik fails to detect containers on Docker Engine 27+

**Symptom:** Services return 404. Traefik logs show:
`client version 1.24 is too old. Minimum supported API version is 1.44`

**Cause:** `traefik:v3.0` is too old for newer Docker API.

**Fix:** Keep `traefik_version: "v3"` in `group_vars/n100.yml` and redeploy:
```bash
ansible-playbook -i inventory.ini deploy_n100.yml --tags traefik
```

---

### 4) `acme.json` always shows `changed`

**Cause:** `state: touch` updates timestamps unless `preserve` is set.

**Fix:** Verify the task uses:
```yaml
- name: Ensure acme.json exists
  ansible.builtin.file:
    path: "/mnt/example_apps/appdata/docker/data/traefik/acme.json"
    state: touch
    mode: '0600'
    access_time: preserve
    modification_time: preserve
```

---

### 5) Fresh machine — module not found

**Symptom:** `community.docker.*` or `ansible.posix.mount` modules unavailable.

**Cause:** required Ansible collections not installed.

**Fix:**
```bash
ansible-galaxy collection install -r requirements.yml
```

---

### 6) Paperless returns 403 behind Traefik

**Symptom:** Paperless container is up, but Traefik returns 403.

**Cause:** Paperless is a Django app. Behind reverse proxy, Django enforces host/origin checks; if `ALLOWED_HOSTS` / `CSRF_TRUSTED_ORIGINS` are wrong, you get 403 even when containers are healthy.

**Fix:** ensure these variables are present in Paperless `.env` template:
```env
PAPERLESS_URL=https://{{ paperless_domain }}
PAPERLESS_CSRF_TRUSTED_ORIGINS=https://{{ paperless_domain }}
PAPERLESS_ALLOWED_HOSTS={{ paperless_domain }}
```
Then redeploy Paperless:
```bash
ansible-playbook -i inventory.ini deploy_docker_nodes.yml --tags paperless
```

---

### 7) Traefik cannot route to Immich/Paperless on Ryzen

**Symptom:** `immich.<domain>` / `paperless.<domain>` return 404 from Traefik.

**Cause:** Docker provider is local-only (NAS socket). Ryzen services must be exposed through Traefik file provider routes.

**Fix:** verify these variables and redeploy Traefik:
- `enable_remote_apps_routes: true` in `group_vars/n100.yml`
- `ryzen_ip`, `immich_service_port`, `paperless_service_port` in `group_vars/all.yml`

```bash
ansible-playbook -i inventory.ini deploy_n100.yml --tags traefik
```

---

### 8) Authentik Redis unhealthy — `Can't handle RDB format version`

**Symptom:** Authentik stack fails to start; Redis logs contain:
`Can't handle RDB format version ...`

**Cause:** persisted Redis dump was created by a newer Redis format than the current Redis image can read (effectively a downgrade mismatch).

**Fix:** keep `authentik_redis_image` on a compatible/newer tag.  
If you accept losing Redis cache/sessions, archive old dump and recreate Redis state:

```bash
ssh <user>@10.0.0.10
sudo docker compose -f /mnt/example_apps/appdata/docker/config/authentik/docker-compose.yml -f /mnt/example_apps/appdata/docker/config/authentik/ldap-outpost.yml down
sudo mv /mnt/example_apps/appdata/docker/config/authentik/redis/dump.rdb /mnt/example_apps/appdata/docker/config/authentik/redis/dump.rdb.bak.$(date +%F_%H-%M-%S) || true
sudo docker compose -f /mnt/example_apps/appdata/docker/config/authentik/docker-compose.yml -f /mnt/example_apps/appdata/docker/config/authentik/ldap-outpost.yml up -d --remove-orphans
```

---

## Incident Playbooks

### SSO outage (Authentik down)

```bash
ssh <user>@10.0.0.10
sudo docker ps --format '{{.Names}} {{.Status}}' | grep -E 'authentik|traefik'
cd /mnt/example_apps/appdata/docker/config/authentik
sudo docker compose -f docker-compose.yml -f ldap-outpost.yml up -d --remove-orphans
sudo docker logs --tail 200 authentik-server
```

Temporary bypass for protected services while recovering:
```bash
ansible-playbook -i inventory.ini deploy_n100.yml --tags adguard,portainer,homepage -e enable_authentik_protection=false
```

Re-enable after recovery:
```bash
ansible-playbook -i inventory.ini deploy_n100.yml --tags adguard,portainer,homepage -e enable_authentik_protection=true
```

---

### DNS outage

```bash
dig +short adguard.homelab.local @10.0.0.10
dig +short authentik.homelab.local @10.0.0.10
ssh <user>@10.0.0.10 "sudo docker logs --tail 200 adguard"
```

---

### TLS / certificate outage

```bash
ssh <user>@10.0.0.10 "sudo docker logs --tail 400 traefik | grep -iE 'acme|challenge|cert|error'"
dig TXT _acme-challenge.<domain> @1.1.1.1  # real domain used for ACME DNS-01
ssh <user>@10.0.0.10 "cd /mnt/example_apps/appdata/docker/config/traefik && sudo docker compose up -d --remove-orphans"
```

---

### RAM pressure on NAS

```bash
ssh <user>@10.0.0.10 "free -h"
ssh <user>@10.0.0.10 "sudo docker stats --no-stream --format 'table {{.Name}}\t{{.MemUsage}}\t{{.MemPerc}}'"
ssh <user>@10.0.0.10 "cat /proc/spl/kstat/zfs/arcstats | grep -E '^(size|c_max|memory_throttle_count)'"
ssh <user>@10.0.0.10 "dmesg -T | grep -iE 'out of memory|killed process|oom' | tail -n 30"
```

---

## Engineering Notes

### Traefik — `acme.json` path and permissions

- ACME storage must use absolute path `/acme.json` inside the container.
- File permissions must be `0600` before Traefik starts or it may fall back to self-signed behavior.

```bash
ssh <user>@10.0.0.10 "sudo ls -l /mnt/example_apps/appdata/docker/data/traefik/acme.json"
ssh <user>@10.0.0.10 "sudo docker logs --tail 200 traefik | grep -iE 'acme|cert|resolver|error'"
```

---

### AdGuard Home — local DNS rewrites

For all LAN clients to route through NAS Traefik, configure a DNS rewrite in AdGuard Home:

```
*.homelab.local → 10.0.0.10
```

Without this, clients may resolve to public IP and bypass the local path.
VPN clients may override DNS — split-tunneling or custom DNS config may be required.

---

### Proxmox — Traefik reverse proxy + Authentik OIDC

Traffic flow in this setup:

```
proxmox.homelab.local
  -> AdGuard DNS rewrite -> 10.0.0.10 (NAS / Traefik)
  -> Traefik router/service
  -> https://10.0.0.20:8006 (Proxmox)
```

The Traefik side is defined in `roles/core_services/templates/traefik/dynamic_conf.yml.j2`
using `proxmox_domain` and `proxmox_internal_url`.

OIDC setup checklist:
1. Create OAuth2/OpenID provider in Authentik for Proxmox.
2. In Authentik, set Redirect URI exactly to `https://proxmox.homelab.local/` (note trailing slash).
3. Configure Proxmox realm with the Authentik issuer/client.

Common error: `Redirect URI mismatch` — compare Authentik redirect URI and what Proxmox sends byte-for-byte.

Proxmox CLI example:
```bash
pveum realm add authentik --type openid \
  --issuer-url https://authentik.homelab.local/application/o/proxmox/ \
  --client-id proxmox \
  --username-claim preferred_username \
  --autocreate 1

# autocreate creates user object but does not grant admin ACL
pveum acl modify / --user <login>@authentik --role Administrator
pveum realm list
```

TLS sanity check (bypasses browser cache):
```bash
curl -vI https://proxmox.homelab.local
```

---

### Authentik — worker permissions (`/media/public`)

Common failure: `authentik-worker` restart loop with permission errors.

Root cause is volume ownership mismatch. This repo creates required paths with configured `authentik_uid`/`authentik_gid` before startup.

```bash
ssh <user>@10.0.0.10 "sudo ls -ld /mnt/example_apps/appdata/docker/config/authentik/media"
ssh <user>@10.0.0.10 "sudo docker logs --tail 200 authentik-worker"
```

---

### Authentik — PostgreSQL "too many clients"

During worker restart loops, connections can spike and exhaust Postgres defaults.

Compose config sets:
```yaml
command: ["postgres", "-c", "max_connections=200"]
```

Check active connections:
```bash
ssh <user>@10.0.0.10 "sudo docker exec -i authentik-db psql -U authentik -d authentik -c 'SELECT count(*) FROM pg_stat_activity;'"
```

---

### LDAP Outpost — dual-network requirement

LDAP outpost must attach to:
- `default` network — to reach Authentik core
- `proxy_net` — to reach Traefik/clients

Missing one produces routing/lookup errors that are hard to trace.

---

### LDAP integration — Base DN exact match

If outpost logs show `No provider found for request`, check Base DN consistency.
Use exactly the same normalized format everywhere (case differences can break matching):

```
dc=homelab,dc=local
```

---

### LDAP integration — username appears as hash (TrueNAS/SSSD)

If username resolves as UUID-like values, map LDAP username to `cn`:

```bash
midclt call ldap.update '{"auxiliary_parameters": "ldap_user_name = cn\nldap_user_gecos = cn\n"}'
sudo sss_cache -E || true
sudo systemctl restart sssd || true
sudo systemctl restart middlewared
```

Verify:
```bash
getent passwd <expected_username>
```

---

### LDAP integration — operational checklist and diagnostics

Before deep debugging, verify:
- Outpost is running and healthy.
- Outpost has the LDAP application assigned.
- Port mapping is correct (`389 -> 3389`), and LDAPS is exposed through Traefik on `636`.
- Base DN is identical everywhere (`dc=homelab,dc=local`, lowercase, no spaces).
- Bind user exists and bind flow has no MFA step.
- LDAP provider search mode is `Direct querying` when you need immediate visibility for new users.
- Validate plain LDAP first (`ldap://:389`), then enable/validate TLS (`ldaps://:636`).

Useful checks:
```bash
ssh <user>@10.0.0.10 "sudo docker ps --format '{{.Names}} {{.Status}}' | grep authentik-ldap-outpost"
ssh <user>@10.0.0.10 "sudo ss -ltnp | egrep ':389|:636'"
ssh <user>@10.0.0.10 "sudo docker logs --tail 200 authentik-ldap-outpost"
```

Optional direct bind test from NAS host network:
```bash
ssh <user>@10.0.0.10 "sudo docker run --rm --network host alpine:3.19 sh -lc 'apk add --no-cache openldap-clients >/dev/null && ldapwhoami -x -H ldap://127.0.0.1:389 -D \"cn=<bind-user>,ou=users,dc=homelab,dc=local\" -W'"
```

Error interpretation quick map:
- `No provider found for request` -> Base DN mismatch or provider routing mismatch
- `Invalid credentials (49)` -> wrong password / bind user / MFA in bind flow
- `SERVER_DOWN` -> host/port/firewall/TLS problem

If new users do not appear immediately:
```bash
sudo sss_cache -E
```

---

### SSH hardening — verification

```bash
# From your workstation: expected denied, no password fallback
ssh -o PubkeyAuthentication=no root@10.0.0.10

# Then SSH to NAS and validate sshd syntax there
ssh <user>@10.0.0.10
sudo sshd -t
```

---

## Ansible Gotchas

**`import_tasks` vs `include_tasks`:**
- `import_tasks` — static, tags work at import level, easier to debug
- `include_tasks` — dynamic, useful with loops/conditions, tags must be repeated inside

**`notify` + handlers:**
- Handler runs once at end of play, even if notified multiple times
- Handler only fires on `changed`, not `ok`

**`failed_when` + `changed_when`:**
- Useful for shell commands with non-standard exit semantics (e.g., `docker network create` errors if network already exists)
- `--check --diff` compares rendered templates from local temp paths (e.g., `~/.ansible/tmp/...`) to remote files; this is expected behavior.

```yaml
register: result
failed_when:
  - result.rc != 0
  - "'already exists' not in result.stderr"
changed_when: result.rc == 0
```

**`become: true` vs `ansible_become: true`:**
- Both work. Pick one convention and don't mix them in the same project.

---

## Go-live Smoke Checklist

```bash
# Syntax check
ansible-playbook -i inventory.ini deploy_n100.yml --syntax-check
ansible-playbook -i inventory.ini deploy_docker_nodes.yml --syntax-check

# Dry run
ansible-playbook -i inventory.ini deploy_n100.yml --check --diff
ansible-playbook -i inventory.ini deploy_docker_nodes.yml --check --diff

# Deploy
ansible-playbook -i inventory.ini deploy_n100.yml
ansible-playbook -i inventory.ini deploy_docker_nodes.yml

# Verify containers are running
ssh <user>@10.0.0.10 "sudo docker ps --format '{{.Names}} {{.Status}}'"
ssh <user>@10.0.0.30 "sudo docker ps --format '{{.Names}} {{.Status}}'"

# Verify RAM caps are enforced
ssh <user>@10.0.0.10 "sudo docker inspect --format '{{.Name}} mem={{.HostConfig.Memory}}' traefik adguard vaultwarden portainer jellyfin homepage"
```

Manual checks:
- [ ] `https://authentik.homelab.local` — Authentik login page loads
- [ ] `https://adguard.homelab.local` — AdGuard Home UI accessible
- [ ] `https://vaultwarden.homelab.local` — Vaultwarden loads
- [ ] `https://portainer.homelab.local` — Portainer accessible
- [ ] `https://homepage.homelab.local` — Homepage accessible
- [ ] `https://jellyfin.homelab.local` — Jellyfin accessible
- [ ] `https://immich.homelab.local` — Immich accessible
- [ ] `https://paperless.homelab.local` — Paperless accessible
- [ ] `https://nas.homelab.local` — TrueNAS UI accessible
- [ ] `http://10.0.0.10:8090` — Traefik dashboard (LAN only)

---
