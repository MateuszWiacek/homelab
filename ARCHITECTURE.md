# HomeLab Architecture

The "why" behind the stack. For incidents and day-2 procedures, go to [`RUNBOOK.md`](RUNBOOK.md).

---

## Table of Contents

- [If you're here from LinkedIn](#if-youre-here-from-linkedin)
- [Node Roles](#node-roles)
- [Constraints and Success Criteria](#constraints-and-success-criteria)
- [Storage Placement](#storage-placement)
- [Decision Log](#decision-log)
- [Trade-offs and Mitigations](#trade-offs-and-mitigations)
- [Operational Targets](#operational-targets)
- [Service Endpoints](#service-endpoints)
- [Ansible Structure](#ansible-structure)

---

## If you're here from LinkedIn

The short version of what this file shows:

- A homelab designed like a platform — DNS, TLS, identity as one coherent stack, not bolted-on services
- Every decision documented with a reason. Not "I saw it on YouTube", but actual trade-off thinking
- Storage split by workload type — ZFS on the NAS, NVMe for latency-sensitive paths
- Ansible structure that scales: roles, group_vars, vault. Adding a new service is DNS + labels + a role

If you want the operational side — how I recover from failures, what breaks first, what to check — that's in [`RUNBOOK.md`](RUNBOOK.md).

---

## Node Roles

### Node 1 — NAS / Control Plane (Intel N100)

| | |
|---|---|
| **Hardware** | Intel N100 (4-core, ~6W TDP) |
| **OS** | TrueNAS SCALE |
| **Role** | 24/7 backbone: storage + DNS + ingress + identity |

This box stays on. It's the control plane of the home network — DNS, TLS termination, SSO, storage. Low power by design.

ZFS stability is the constraint here. Containers are capped with memory limits and I verify the caps via `docker inspect … HostConfig.Memory`. ZFS ARC doesn't care about your app spike.

Jellyfin lives here on purpose: Intel Quick Sync makes transcoding cheap and predictable, even 4K → 1080p.

### Node 2 — Compute Lane (AMD Ryzen 5 6600H)

| | |
|---|---|
| **Hardware** | AMD Ryzen 5 6600H |
| **OS** | Debian VM on Proxmox VE |
| **Role** | CPU/RAM-hungry workloads |

Immich and Paperless-ngx are not NAS-friendly. AI inference, OCR, and Postgres don't belong next to ZFS. The Ryzen takes the noisy work and can go down without taking core services with it.

---

## Constraints and Success Criteria

- **Clean URLs**: `https://service.<domain>` not `10.0.0.10:8096`. Nobody wants to memorize ports.
- **No inbound exposure**: TLS via Cloudflare DNS-01. Router stays closed.
- **HTTPS everywhere on LAN**: fewer edge cases, cleaner trust model.
- **Storage first**: ZFS shouldn't crash because an app got hungry.
- **Wife test**: non-technical household member can navigate without help. One dashboard, clean URLs, nothing breaks silently.

> `.homelab.local` is a placeholder. DNS-01 requires a real registered domain.

---

## Storage Placement

1Gbit between nodes is fine for photos and documents. It's not a database link.

| Data | Location | Why |
|---|---|---|
| PostgreSQL databases | Ryzen NVMe | No network latency on every query |
| Immich AI model cache | Ryzen NVMe | Avoid 1Gbit bottleneck on model startup |
| Photos library | NAS HDD pool via NFS | ZFS integrity, capacity, snapshots |
| Documents archive | NAS HDD pool via NFS | Central storage + cross-device access |
| App configs | NAS SSD pool via NFS | Fast + snapshot-friendly |

---

## Decision Log

| Decision | Reasoning |
|---|---|
| **AdGuard Home** | Internal DNS. Clean names, one place to manage. I don't memorize `10.0.0.10:1234`. |
| **Traefik + Cloudflare DNS-01** | Valid certs on LAN without opening inbound ports. No self-signed pain. |
| **Authentik over Authelia** | One identity source for users, groups, policies. Easier to reason about access. |
| **Vaultwarden self-hosted** | My vault, my backup policy. |
| **Portainer** | Phone-grade admin when I'm away from a real terminal. Complements Ansible, doesn't replace it. |
| **Homepage** | One dashboard for the whole household. Wife test passes. |
| **Break-glass local admins** | SSO outage shouldn't mean lockout. Recovery beats purity. |
| **TrueNAS SCALE on N100** | 24/7 low-power host with native container support. |
| **Jellyfin on N100** | Intel Quick Sync. Cheap transcoding, 24/7, low power. |
| **Container RAM limits on NAS** | ZFS ARC doesn't care about your app spike. Caps enforced, verified via `docker inspect`. |
| **Immich + Paperless on Ryzen** | AI/OCR and databases want real compute. NAS stays stable. |
| **DB and cache on NVMe** | 1Gbit is not a database link. |
| **`latest` tags during bootstrap** | Ship fast, validate, then pin. Not the other way around. |

---

## Trade-offs and Mitigations

Things I consciously accept. Not a threats list — a list of known costs with mitigations.

| Trade-off | Benefit | Cost | Mitigation |
|---|---|---|---|
| **SSO in the control plane** | One access policy, single front door | SSO down = locked out | Break-glass local admin; ForwardAuth toggleable per-service |
| **Internal DNS as critical infra** | Clean URLs, no port memorization | DNS down = "everything's broken" from a user's perspective | Fallback resolver on router; IP access still works |
| **DNS-01 for certs** | Valid LAN certs, no open ports | DNS API dependency, rate limits | Minimal-scope token, rotation, runbook steps |
| **Reverse proxy as choke point** | One TLS/routing layer | Bad config breaks multiple services at once | Config in code, quick rollback |
| **GUI tools (Portainer)** | Fast admin from phone | Drift risk if changes bypass Ansible | GUI is an ops console only — changes get backported to code |
| **`latest` tags during bootstrap** | Faster initial rollout | An update can surprise you | Pin after stabilization; version bumps go through changelog |

> Rule: if recovery is harder than purity, purity loses.

---

## Operational Targets

- **N100** online 24/7. Ryzen VM stoppable without affecting core services.
- **ZFS ARC** target ~4.7 GiB. Container memory caps enforced and verified.
- **DB and AI cache** on Ryzen NVMe — no 1Gbit roundtrip per query.

---

## Service Endpoints

Everything behind Traefik, HTTPS via Cloudflare DNS-01. No inbound ports. Traefik dashboard is LAN-only.

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

Immich and Paperless-ngx run on the Ryzen VM but route through NAS Traefik via file provider — Docker socket provider only sees local containers.

---

## Ansible Structure

```text
roles/
├── core_services/   # Traefik, AdGuard Home, Authentik, Vaultwarden, Portainer
├── media_stack/     # Jellyfin, Homepage
├── prod_apps/       # Immich, Paperless-ngx, DB backups
├── common/          # APT baseline, qemu-guest-agent
├── ssh_hardening/   # Shared SSH policy — both nodes
└── docker_host/     # Docker engine, Portainer Agent
```

```text
group_vars/
├── all.yml               # Global: domains, IPs, shared endpoints, DNS resolvers
├── n100.yml              # NAS-specific: paths, versions, compatibility toggles
└── docker_nodes.yml      # Ryzen: Docker user, NFS mounts, node-specific values

secrets.yml               # Vault: tokens, passwords, API keys — never in git
```

Roles are independently deployable via tags. IPs, domains, versions → `group_vars`. Credentials → `secrets.yml` (vault).


