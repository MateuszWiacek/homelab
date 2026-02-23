# Ansible Homelab — Runbook (Core Stack v1.0)

Operational reference for the NAS core node.
For architecture and deployment overview → [`README.md`](README.md)

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
ssh admin@10.0.0.10 "sudo docker ps --format 'table {{.Names}}\t{{.Status}}'"
ssh admin@10.0.0.10 "sudo docker logs --tail 120 traefik"
ssh admin@10.0.0.10 "sudo docker logs --tail 120 authentik-server"
dig +short authentik.homelab.local @10.0.0.10
ssh admin@10.0.0.10 "sudo docker logs --tail 300 traefik | grep -iE 'acme|challenge|cert|error'"
```

---

## Scope

**Included in this profile:**
- Traefik
- AdGuard Home
- Authentik (+ LDAP outpost)
- Vaultwarden
- Portainer
- SSH hardening role

**Not included in this v1.0 profile:**
- `media_stack` (Jellyfin)
- `prod_apps` (Immich, Paperless-ngx)
- `docker_host` / `docker_nodes`

---

## Infrastructure

| Node | IP | OS | Role | Ansible group |
|---|---|---|---|---|
| NAS | 10.0.0.10 | TrueNAS SCALE | Core services | `n100` |

> This public branch uses sanitized example values:
> - SSH target: `admin@10.0.0.10`
> - `docker_config_dir`: `/mnt/ssd_apps/appdata/docker/config`
> - `docker_data_dir`: `/mnt/ssd_apps/appdata/docker/data`
> - Domain suffix `.homelab.local` is for documentation only; ACME DNS-01 requires a real registered domain.

---

## Common Commands

### Deploy

```bash
# Full core deploy
ansible-playbook -i inventory.ini deploy_n100.yml

# Single service
ansible-playbook -i inventory.ini deploy_n100.yml --tags traefik
ansible-playbook -i inventory.ini deploy_n100.yml --tags adguard
ansible-playbook -i inventory.ini deploy_n100.yml --tags authentik
ansible-playbook -i inventory.ini deploy_n100.yml --tags vaultwarden
ansible-playbook -i inventory.ini deploy_n100.yml --tags portainer

# Entire core role
ansible-playbook -i inventory.ini deploy_n100.yml --tags core

# SSH hardening
ansible-playbook -i inventory.ini deploy_n100.yml --tags hardening

# Dry run (always run before applying changes)
ansible-playbook -i inventory.ini deploy_n100.yml --check --diff
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
ansible-playbook -i inventory.ini deploy_n100.yml --list-tasks
ansible-playbook -i inventory.ini deploy_n100.yml --list-tags
ansible-playbook -i inventory.ini deploy_n100.yml --list-hosts
```

### Lint / Syntax

```bash
ansible-playbook -i inventory.ini deploy_n100.yml --syntax-check
ansible-lint deploy_n100.yml
yamllint .
```

---

## Variable Map

| File | Contains |
|---|---|
| `group_vars/all.yml` | Domains, NAS IP, shared ports/endpoints |
| `group_vars/n100.yml` | Paths (`docker_config_dir`, `docker_data_dir`), image tags, compatibility toggles |
| `secrets.yml` | `cert_email`, `cloudflare_token`, `authentik_*` secrets |

---

## Common Issues & Solutions

### 1) AdGuard Home fails to start — port 53 already in use

**Symptom:** `failed to bind host port 0.0.0.0:53/tcp: address already in use`

**Cause:** `systemd-resolved` DNS stub listener on non-TrueNAS hosts (Debian/Ubuntu).

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

> This repo keeps `adguard_setup_mode: true` and `adguard_publish_http_port: true` for production-compat behavior.

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
    path: "{{ docker_data_dir }}/traefik/acme.json"
    state: touch
    mode: '0600'
    access_time: preserve
    modification_time: preserve
```

---

### 5) Fresh machine — module not found

**Symptom:** `community.docker.*` modules unavailable.

**Cause:** Required Ansible collection not installed.

**Fix:**
```bash
ansible-galaxy collection install -r requirements.yml
```

---

## Incident Playbooks

### SSO outage (Authentik down)

```bash
ssh admin@10.0.0.10
sudo docker ps --format '{{.Names}} {{.Status}}' | grep -E 'authentik|traefik'
cd /mnt/ssd_apps/appdata/docker/config/authentik
sudo docker compose -f docker-compose.yml -f ldap-outpost.yml up -d --remove-orphans
sudo docker logs --tail 200 authentik-server
```

Temporary bypass for protected services while recovering:
```bash
ansible-playbook -i inventory.ini deploy_n100.yml --tags adguard,portainer -e enable_authentik_protection=false
```

Re-enable after recovery:
```bash
ansible-playbook -i inventory.ini deploy_n100.yml --tags adguard,portainer -e enable_authentik_protection=true
```

---

### DNS outage

```bash
dig +short adguard.homelab.local @10.0.0.10
dig +short authentik.homelab.local @10.0.0.10
ssh admin@10.0.0.10 "sudo docker logs --tail 200 adguard"
```

---

### TLS / certificate outage

```bash
ssh admin@10.0.0.10 "sudo docker logs --tail 400 traefik | grep -iE 'acme|challenge|cert|error'"
dig TXT _acme-challenge.homelab.local @1.1.1.1
ssh admin@10.0.0.10 "cd /mnt/ssd_apps/appdata/docker/config/traefik && sudo docker compose up -d --remove-orphans"
```

---

### RAM pressure on NAS

```bash
ssh admin@10.0.0.10 "free -h"
ssh admin@10.0.0.10 "sudo docker stats --no-stream --format 'table {{.Name}}\t{{.MemUsage}}\t{{.MemPerc}}'"
ssh admin@10.0.0.10 "cat /proc/spl/kstat/zfs/arcstats | grep -E '^(size|c_max|memory_throttle_count)'"
ssh admin@10.0.0.10 "dmesg -T | grep -iE 'out of memory|killed process|oom' | tail -n 30"
```

---

## Engineering Notes

### Traefik — `acme.json` path and permissions

- ACME storage must use absolute path `/acme.json` inside the container.
- File permissions must be `0600` before Traefik starts or it may fall back to self-signed behavior.

```bash
ssh admin@10.0.0.10 "sudo ls -l /mnt/ssd_apps/appdata/docker/data/traefik/acme.json"
ssh admin@10.0.0.10 "sudo docker logs --tail 200 traefik | grep -iE 'acme|cert|resolver|error'"
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

### Authentik — worker permissions (`/media/public`)

Common failure: `authentik-worker` restart loop with permission errors.

Root cause is volume ownership mismatch. This repo creates required paths with configured `authentik_uid`/`authentik_gid` before startup.

```bash
ssh admin@10.0.0.10 "sudo ls -ld /mnt/ssd_apps/appdata/docker/config/authentik/media"
ssh admin@10.0.0.10 "sudo docker logs --tail 200 authentik-worker"
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
ssh admin@10.0.0.10 "sudo docker exec -i authentik-db psql -U authentik -d authentik -c 'SELECT count(*) FROM pg_stat_activity;'"
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

### SSH hardening — verification

```bash
# From your workstation: expected denied, no password fallback
ssh -o PubkeyAuthentication=no root@10.0.0.10

# Then SSH to NAS and validate sshd syntax there
ssh admin@10.0.0.10
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

# Dry run
ansible-playbook -i inventory.ini deploy_n100.yml --check --diff

# Deploy
ansible-playbook -i inventory.ini deploy_n100.yml

# Verify containers are running
ssh admin@10.0.0.10 "sudo docker ps --format '{{.Names}} {{.Status}}'"

# Verify RAM caps are enforced
ssh admin@10.0.0.10 "sudo docker inspect --format '{{.Name}} mem={{.HostConfig.Memory}}' traefik adguard vaultwarden portainer"
```

Manual checks:
- [ ] `https://authentik.homelab.local` — Authentik login page loads
- [ ] `https://adguard.homelab.local` — AdGuard Home UI accessible
- [ ] `https://vaultwarden.homelab.local` — Vaultwarden loads
- [ ] `https://portainer.homelab.local` — Portainer accessible
- [ ] `http://10.0.0.10:8090` — Traefik dashboard (LAN only)

---
