# HomeLab with Ansible

I like my infrastructure the same way I like CI/CD: **boring, repeatable, recoverable**.

This is the **v1.1 release** — from core networking and identity to media and compute:

- **AdGuard Home** — internal DNS (clean URLs)
- **Traefik** — ingress + TLS (HTTPS on LAN, DNS-01 via Cloudflare)
- **Authentik** — SSO (one login gate, policies in one place)
- **Vaultwarden** — self-hosted password vault under my backup policy
- **Portainer** — quick container admin when I'm not at a real terminal

In v1.1, this baseline is shipped together with media services (Jellyfin, Homepage) and the Ryzen app lane (Immich, Paperless-ngx, backups).

> Public repo note: hostnames and IPs are intentionally example values. The real ones live in a private inventory.
> For ACME/DNS-01 in real deployments, replace `.local` with your real registered domain.

---

## What this repo is actually showing

If you're scanning this from LinkedIn, this is what I wanted to show:

- DNS, TLS, and SSO wired together as one stack (not random services glued together)
- Reproducible deploys with Ansible + templates + inventory-driven vars
- Real day-2 operations documented in [`RUNBOOK.md`](RUNBOOK.md)

---

## Network Architecture

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

I split the lab by responsibility and power profile:

```
                    ┌───────────────┐
Client (LAN/WiFi) → │ AdGuard (DNS) │ → https://service.<domain>
                    └──────┬────────┘
                           │
                           v
                    ┌───────────────┐
                    │ Traefik (TLS) │  Cloudflare DNS-01 → valid certs on LAN
                    └──────┬────────┘
                           │ ForwardAuth (optional)
                           v
                    ┌───────────────┐
                    │  Authentik    │  SSO + policies
                    └──────┬────────┘
                           │
          ┌────────────────┴────────────────┐
          v                                 v
   NAS-local apps                      Remote apps (Ryzen VM)
 (Vaultwarden, Jellyfin, etc.)        (Immich, Paperless, DBs)
```

I keep routing/TLS in Traefik and identity in Authentik, so adding a new service is usually DNS + labels.

---

## Architecture Overview

I split the lab by responsibility and power profile.

### Node 1 — NAS / Identity Layer (Intel N100)

| | |
|---|---|
| **Hardware** | Intel N100 (4-core, ~6W TDP) |
| **OS** | TrueNAS SCALE |
| **Role** | 24/7 backbone: storage + DNS + ingress + identity |

This box stays on. It's the "control plane" of the home network.

Key detail: **ZFS stability wins**. Containers on the NAS are capped with Docker/Compose memory limits (and I verify the caps via `docker inspect … HostConfig.Memory`) so they don't starve ZFS ARC when something spikes. 

Jellyfin lives here on purpose: Intel Quick Sync makes transcoding (even 4K → 1080p) cheap and predictable.

This repo deploys the NAS stack as core + media. In v1.1, core services (Traefik, AdGuard Home, Authentik, Vaultwarden, Portainer), media (Jellyfin + Homepage), and the Ryzen app lane ship as one release.

### Node 2 — Compute lane (AMD Ryzen 5 6600H)

| | |
|---|---|
| **Hardware** | AMD Ryzen 5 6600H |
| **OS** | Debian VM on Proxmox VE |
| **Role** | "production" apps that actually want CPU/RAM |

Immich and Paperless-ngx are not NAS-friendly workloads (AI/OCR + databases). I'd rather keep the NAS boring and let the Ryzen do the noisy work. This compute lane is included in v1.1.


---

## Constraints & success criteria

- **Clean URLs**: `https://service.<domain>` instead of `10.0.0.10:8096` — no `ip:port` (wife's nightmare)
- **No inbound exposure**: TLS via Cloudflare DNS-01, no open WAN ports
- **HTTPS for user-facing apps on LAN**: fewer weird edge cases, cleaner trust model
- **Storage first**: ZFS shouldn't crash because an app got hungry
- **Household usability**: "wife test" passes via one homepage / dashboard

> Docs note: `.homelab.local` is used in examples as a placeholder only. ACME DNS-01 requires a real registered domain.

---

## Storage — what lives where and why

The 1Gbit link between nodes is fine for photos and documents. It's not fine for Postgres round-trips or model loading.

| Data | Location | Why |
|---|---|---|
| PostgreSQL databases | Ryzen NVMe | DB queries shouldn't pay network latency tax |
| Immich AI model cache | Ryzen NVMe | Avoid 1Gbit bottleneck on model startup |
| Photos library | NAS HDD pool via NFS | ZFS integrity, capacity, snapshots |
| Documents archive | NAS HDD pool via NFS | Central storage + cross-device access |
| App configs | NAS SSD pool via NFS | Fast + snapshot-friendly |

---

## Decision log

| Decision | Why |
|---|---|
| **AdGuard Home** | Internal DNS = clean names, one place to manage them. I don't want to memorize `10.0.0.10:1234`. |
| **Traefik + Cloudflare DNS-01** | Valid certs on LAN without inbound ports. No self-signed pain. |
| **Authentik over Authelia** | One identity source for users, groups, and policies. Easier to reason about access. |
| **Vaultwarden self-hosted** | Password vault under my control, my backup policy. |
| **Portainer** | "Phone-grade" admin when I'm away from a full shell. Complements, doesn't replace, Ansible. |
| **Homepage** | One landing page for daily use + household usability. Non-technical users navigate without help. |
| **Break-glass local admins kept** | SSO outage shouldn't mean locked out. Recovery beats purity. |
| **TrueNAS SCALE on N100** | 24/7 low-power host with native container support. |
| **Jellyfin on N100** | Intel Quick Sync = best performance-per-watt for transcoding. |
| **Container RAM limits on NAS** | Protect ZFS ARC and keep storage stable under load. |
| **Immich + Paperless-ngx on Ryzen** | AI/OCR + DB workloads get real compute; NAS stays responsive. |
| **DB/cache on local NVMe (Ryzen)** | Lower latency + higher IOPS; avoid 1Gbit as the bottleneck. |

---

## Trade-offs & mitigations

This isn’t a “threats list”. It’s a list of things I **consciously accept**, because in return I get a simpler system and better day-2 operations.

| Trade-off | Benefit (what I get) | Cost (what I accept) | Mitigation (how I keep it sane) |
|---|---|---|---|
| **SSO becomes part of the control plane** | One place for access policy, groups, and a single “front door” for services. Easier to reason about access. | If SSO goes down, I can lock myself out. Recovery beats purity. | Break-glass local admin; per-service auth toggle (ForwardAuth isn’t mandatory everywhere); quick health check + step-by-step runbook. |
| **Internal DNS becomes critical infra** | Clean URLs and consistent naming. No port memorization. | DNS down = “everything down” from a user’s perspective, even if services are still running. | Fallback resolver plan (router/clients); ability to access by IP during incidents; simple first-step tests (`dig/nslookup`). |
| **DNS-01 for certs = more moving parts** | Valid certs on LAN without opening inbound internet ports. No self-signed pain. | Dependency on DNS API/tokens, edge cases, and rate limits. | Minimal-scope token (single zone); rotation; logs + clear troubleshooting steps in the runbook. |
| **Reverse proxy as a choke point** | One routing/TLS/headers layer. One place that “does HTTPS” for the whole network. | A bad proxy change can break access to multiple services at once. | Config in code (review/diff); dashboard kept local; quick rollback to last known-good config. |
| **GUI tools vs IaC drift** | Fast admin actions when I’m away from a full shell. Convenience without hacks. | GUIs tempt “quick fixes” → drift and non-reproducible state. | GUI is an ops console, not the source of truth: changes end up in code; in incidents I note the hotfix and backport it to the repo. |
| **`latest` tags during bootstrap** | Faster initial rollout while I validate the stack. | Reproducibility suffers and an update can surprise you. | After stabilization I pin exact tags; version bumps go through release notes/changelog, not by accident. |

> **Rule:** if recovery is harder than purity, purity loses.

---

## Operational targets

- **Power baseline**: N100 stays online 24/7 for core services. Ryzen is a VM — can be stopped without affecting core services.
- **Storage stability**: ZFS ARC target ~4.7 GiB; containers are capped to avoid ARC starvation.
- **Latency-sensitive paths**: DB and AI model caches stay on Ryzen NVMe — no 1Gbit bottleneck for every query.

---

## "Read this before deploy"

These are the things that look trivial until they ruin your evening:

- **AdGuard first-run wizard** — fresh installs need one-time bootstrap on `:3000` before the UI is normal. See runbook.
- **Traefik `acme.json`** — must exist, `0600`, treated carefully. `state: touch` with preserved timestamps to avoid false `changed`.
- **Authentik outpost config** — if SSO feels "almost working", the answer is usually in outpost or worker logs.

Full failure modes, incident playbooks, and deep dives → [`RUNBOOK.md`](RUNBOOK.md).

---

## Service endpoints

Application services are behind Traefik with automatic HTTPS via Cloudflare DNS-01 (no inbound ports). The Traefik dashboard is intentionally LAN-only on `http://<nas-ip>:8090`.

| Service | URL | Node |
|---|---|---|
| Traefik dashboard | `http://<nas-ip>:8090` *(LAN-only)* | NAS |
| Authentik | `https://authentik.<domain>` | NAS |
| AdGuard Home | `https://adguard.<domain>` | NAS |
| Vaultwarden | `https://vaultwarden.<domain>` | NAS |
| Portainer | `https://portainer.<domain>` | NAS |
| Homepage | `https://homepage.<domain>` | NAS |
| Jellyfin | `https://jellyfin.<domain>` | NAS |
| Immich | `https://immich.<domain>` | Ryzen VM |
| Paperless-ngx | `https://paperless.<domain>` | Ryzen VM |
| TrueNAS UI | `https://nas.<domain>` | NAS |
| Proxmox | `https://proxmox.<domain>` | NAS → Proxmox |

Immich and Paperless-ngx run on the Ryzen VM but route through the NAS Traefik instance via file provider — Docker socket provider only sees local containers.

---

## Ansible structure

```
roles/
├── core_services/   # Traefik, AdGuard Home, Authentik, Vaultwarden, Portainer
├── media_stack/     # Jellyfin, Homepage
├── prod_apps/       # Immich, Paperless-ngx, DB backups
├── common/          # APT baseline, qemu-guest-agent
├── ssh_hardening/   # Shared SSH policy — both nodes
└── docker_host/     # Docker engine, Portainer Agent
```

```
group_vars/
├── all.yml               # Global: domains, IPs, shared endpoints, DNS resolvers
├── n100.yml              # NAS-specific: paths, versions, compatibility toggles
└── docker_nodes.yml      # Ryzen: Docker user, NFS mounts, node-specific values

roles/prod_apps/defaults/main.yml   # Default stack paths for Immich/Paperless — override in docker_nodes.yml
secrets.yml                         # Vault: tokens, passwords, API keys — never in git
```

Roles are independently deployable via tags. IPs, domains, versions → `group_vars`. Credentials → `secrets.yml` (vault).

---

## Deploy

```bash
# Prerequisites
ansible-galaxy collection install -r requirements.yml

# Dry run first
ansible-playbook -i inventory.ini deploy_n100.yml --check --diff
ansible-playbook -i inventory.ini deploy_docker_nodes.yml --check --diff

# Full deploy
ansible-playbook -i inventory.ini deploy_n100.yml             # NAS: core services + media
ansible-playbook -i inventory.ini deploy_docker_nodes.yml     # Ryzen: Docker host + prod apps

# Single service — NAS
ansible-playbook -i inventory.ini deploy_n100.yml --tags traefik
ansible-playbook -i inventory.ini deploy_n100.yml --tags adguard
ansible-playbook -i inventory.ini deploy_n100.yml --tags authentik
ansible-playbook -i inventory.ini deploy_n100.yml --tags vaultwarden
ansible-playbook -i inventory.ini deploy_n100.yml --tags portainer
ansible-playbook -i inventory.ini deploy_n100.yml --tags core    # entire core_services role
ansible-playbook -i inventory.ini deploy_n100.yml --tags media   # Jellyfin + Homepage

# Single service — Ryzen
ansible-playbook -i inventory.ini deploy_docker_nodes.yml --tags immich
ansible-playbook -i inventory.ini deploy_docker_nodes.yml --tags paperless
ansible-playbook -i inventory.ini deploy_docker_nodes.yml --tags backups

# SSH hardening — both nodes
ansible-playbook -i inventory.ini deploy_n100.yml --tags hardening
ansible-playbook -i inventory.ini deploy_docker_nodes.yml --tags hardening
```

`site.yml` is a compatibility alias → imports `deploy_docker_nodes.yml`.

---

## Security

- **No inbound WAN/NAT ports** — HTTPS via Cloudflare DNS-01. Router/firewall remains closed; only selected LAN ports are exposed on NAS/Ryzen where needed.
- **SSO via Authentik** — ForwardAuth on Traefik, toggled per-service via `enable_authentik_protection`.
- **SSH hardening** — `ssh_hardening` role, `--tags hardening`.
- **Secrets** — ansible-vault only. Never plaintext, never in git.
- **Safe diffs** — `no_log: true` on secret-bearing templates. `.env` files at `0600`.
- **Break-glass** — lockout safety beats purity.


---

## See also

[`RUNBOOK.md`](RUNBOOK.md) — operational commands, incident playbooks, engineering deep dives, smoke checklist
