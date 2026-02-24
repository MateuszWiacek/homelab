# HomeLab with Ansible

A DevOps engineer's homelab. Built once, never touched again if possible.

DNS, TLS, SSO, self-hosted media, photos, documents — automated end to end with Ansible. No open ports, no self-signed certs, no "just click through the warning". If something breaks at 2am and I'm not around, my wife can follow the runbook and fix it. That's the bar.

This repo is a design + ops artifact. If you want the why: [`ARCHITECTURE.md`](ARCHITECTURE.md). If you want the how-to-fix-at-2am: [`RUNBOOK.md`](RUNBOOK.md).

> Hostnames and IPs are example values. Real ones live in a private inventory.

---

## If you're here from LinkedIn

This is a working homelab, not a demo. Everything here runs at home daily.

- DNS, TLS, and SSO wired as one stack — not services glued together with hope
- Reproducible deploys: Ansible roles, Jinja2 templates, inventory-driven vars
- No manual steps in production. If it's not in code, it doesn't exist
- Day-2 operations documented in [`RUNBOOK.md`](RUNBOOK.md) — not in someone's memory
- Design decisions and trade-offs → [`ARCHITECTURE.md`](ARCHITECTURE.md)

If you're here to build your own open-source homestack, jump straight to the Quick Start section.

---

## Stack

| Service | What it does |
|---|---|
| AdGuard Home | Internal DNS — clean URLs, no port memorization |
| Traefik | Ingress + TLS via Cloudflare DNS-01 |
| Authentik | SSO — one login gate for everything |
| Vaultwarden | Self-hosted password vault |
| Portainer | Container admin when I'm not at a real terminal |
| Jellyfin | Self-hosted media server |
| Homepage | Dashboard — one place for the whole household |
| Immich | Self-hosted Google Photos replacement |
| Paperless-ngx | Self-hosted document archive |

---

## Network

```
                    ┌───────────────┐
Client (LAN/WiFi) → │ AdGuard (DNS) │ → https://service.<domain>
                    └──────┬────────┘
                           │
                           v
                    ┌───────────────┐
                    │ Traefik (TLS) │  Cloudflare DNS-01 → valid certs on LAN
                    └──────┬────────┘
                           │ ForwardAuth (optional per-service)
                           v
                    ┌───────────────┐
                    │  Authentik    │  SSO + policies
                    └──────┬────────┘
                           │
          ┌────────────────┴────────────────┐
          v                                 v
   NAS-local apps                      Ryzen VM apps
 (Vaultwarden, Jellyfin, etc.)        (Immich, Paperless, DBs)
```

Two nodes, split by responsibility and power profile:

```
┌─────────────────────────────────┐     ┌──────────────────────────────────┐
│         NAS — Intel N100        │     │       Compute — Ryzen 5 6600H    │
│         TrueNAS SCALE           │     │       Debian VM on Proxmox       │
│                                 │     │                                  │
│  Traefik    AdGuard Home        │     │  Immich       Paperless-ngx      │
│  Authentik  Vaultwarden         │◄────│  PostgreSQL   AI model cache     │
│  Jellyfin   Homepage            │ NFS │                                  │
│  Portainer                      │     │  Heavy CPU/RAM workloads         │
│                                 │     │  Local NVMe for latency-         │
│  24/7, low power, ZFS storage   │     │  sensitive data                  │
└─────────────────────────────────┘     └──────────────────────────────────┘
```



## Quick Start

```bash
ansible-galaxy collection install -r requirements.yml

# Dry run first — always
ansible-playbook -i inventory.ini deploy_n100.yml --check --diff
ansible-playbook -i inventory.ini deploy_docker_nodes.yml --check --diff

# Full deploy
ansible-playbook -i inventory.ini deploy_n100.yml             # NAS: core + media
ansible-playbook -i inventory.ini deploy_docker_nodes.yml     # Ryzen: prod apps

# Single service
ansible-playbook -i inventory.ini deploy_n100.yml --tags traefik
ansible-playbook -i inventory.ini deploy_n100.yml --tags authentik
ansible-playbook -i inventory.ini deploy_docker_nodes.yml --tags immich
```

Secrets: copy `secrets.yml.example` → `secrets.yml`, fill in, encrypt with ansible-vault. Never plaintext, never in git.

---

## Repo Layout

```
roles/          # core_services / media_stack / prod_apps / common / ssh_hardening / docker_host
group_vars/     # all.yml / n100.yml / docker_nodes.yml
deploy_n100.yml           # NAS: core + media
deploy_docker_nodes.yml   # Ryzen: prod apps
ARCHITECTURE.md           # the why
RUNBOOK.md                # the how-to-fix-at-2am
```

---

## Read This Before Deploy

Things that look trivial until they ruin your evening:

- **AdGuard first-run wizard** — fresh installs need one-time bootstrap on `:3000` before the UI is normal. See runbook.
- **Traefik `acme.json`** — must exist, `0600`, handled carefully. `state: touch` with preserved timestamps to avoid false `changed`.
- **Authentik outpost config** — if SSO feels "almost working", the answer is usually in outpost or worker logs.

Full failure modes and incident playbooks → [`RUNBOOK.md`](RUNBOOK.md).

---

## Docs

- [`ARCHITECTURE.md`](ARCHITECTURE.md) — design decisions, trade-offs, node roles, storage layout
- [`RUNBOOK.md`](RUNBOOK.md) — operational commands, incident playbooks, smoke checklist
