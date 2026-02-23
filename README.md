# HomeLab — Core Stack

I like my infrastructure the same way I like CI/CD: **boring, repeatable, recoverable**.

This repo contains the core building blocks that make everything else sane:

- **AdGuard Home** — internal DNS (clean URLs)
- **Traefik** — ingress + TLS (HTTPS on LAN, DNS-01 via Cloudflare)
- **Authentik** — SSO (one login gate, policies in one place)
- **Vaultwarden** — self-hosted password vault under my backup policy
- **Portainer** — quick container admin when I'm not at a real terminal

> Public repo note: hostnames and IPs are intentionally example values. The real ones live in a private inventory.
> For ACME/DNS-01 in real deployments, replace `.local` with your real registered domain.

---

## What this repo includes

- DNS, TLS, and SSO wired together as one stack (not random services glued together)
- Reproducible deploys with Ansible + templates + inventory-driven vars
- Real day-2 operations documented in [`RUNBOOK.md`](RUNBOOK.md)

---

## Network Architecture

```
Client (LAN/WiFi)
  │
  │  DNS → clean names, no ip:port
  ▼
AdGuard Home  (10.0.0.10:53)
  │
  │  HTTPS everywhere on LAN (Cloudflare DNS-01, no inbound ports)
  ▼
Traefik  ─────────────────────┐
  │                           │ ForwardAuth (optional per-service)
  ▼                           │
Authentik  ◄──────────────────┘   SSO + policies
  │
  ├──► Vaultwarden     https://vaultwarden.homelab.local
  ├──► Portainer       https://portainer.homelab.local
  └──► (anything you add later)
```

I keep routing/TLS in Traefik and identity in Authentik, so adding a new service is usually DNS + labels.

---

## Architecture Overview

I split responsibilities by hardware profile and power usage.

### Node 1: NAS / Identity Layer (Intel N100)

| Item | Value |
|---|---|
| Hardware | Intel N100 (4-core, ~6W TDP) |
| OS | TrueNAS SCALE |
| Role | 24/7 always-on core services: DNS, reverse proxy, identity, management |

The N100 stays on 24/7. In my tests it usually idles around 10W and goes up to ~35W under stress, so it is the right place for always-on services.
This core profile deploys Traefik, AdGuard, Authentik, Vaultwarden, and Portainer.

ZFS ARC is intentionally protected. Docker containers on the NAS are kept memory-conscious to avoid competing with filesystem cache.




---

## Decision log

| Decision | Why |
|---|---|
| **AdGuard Home** | Internal DNS = clean names, one place to manage them. I don't want to memorize `10.0.0.10:1234`. |
| **Traefik + Cloudflare DNS-01** | Valid certs on LAN without inbound ports. No self-signed pain. |
| **Authentik over Authelia** | One identity source for users, groups, and policies. Easier to reason about access. |
| **Vaultwarden self-hosted** | Password vault under my control, my backup policy. |
| **Portainer** | "Phone-grade" admin when I'm away from a full shell. Complements, doesn't replace, Ansible. |
| **Break-glass local admins kept** | SSO outage shouldn't mean locked out. Recovery beats purity. |

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

> **Rule:** if recovery is harder than purity, recovery wins.

---

## "Read this before deploy"

These are the things that look trivial until they ruin your evening:

- **AdGuard first-run wizard** — fresh installs need one-time bootstrap on `:3000` before the UI is normal. See runbook.
- **Traefik `acme.json`** — must exist, `0600`, treated carefully. `state: touch` with preserved timestamps to avoid false `changed`.
- **Authentik outpost config** — if SSO feels "almost working", the answer is usually in outpost or worker logs.

Full failure modes, incident playbooks, and deep dives → [`RUNBOOK.md`](RUNBOOK.md).

---

## Service endpoints

| Service | URL | Node |
|---|---|---|
| Traefik dashboard | `http://10.0.0.10:8090` *(LAN-only)* | NAS |
| Authentik | `https://authentik.homelab.local` | NAS |
| AdGuard Home | `https://adguard.homelab.local` | NAS |
| Vaultwarden | `https://vaultwarden.homelab.local` | NAS |
| Portainer | `https://portainer.homelab.local` | NAS |

These URLs use `.homelab.local` as documentation examples. In real deployments with Cloudflare DNS-01, use your real registered domain.

---

## Ansible structure

```
roles/
├── core_services/   # Traefik, AdGuard Home, Authentik, Vaultwarden, Portainer
└── ssh_hardening/   # SSH policy
```

```
group_vars/
├── all.yml          # Global: domains, IPs, shared endpoints, DNS resolvers
└── n100.yml         # NAS-specific: paths, versions, compatibility toggles

secrets.yml          # Vault: tokens, passwords, API keys — never in git
```

Roles are independently deployable via tags. IPs, domains, versions → `group_vars`. Credentials → `secrets.yml` (vault).

---

## Deploy

```bash
# Prerequisites
ansible-galaxy collection install -r requirements.yml

# Dry run first
ansible-playbook -i inventory.ini deploy_n100.yml --check --diff

# Full core deploy
ansible-playbook -i inventory.ini deploy_n100.yml

# Single service
ansible-playbook -i inventory.ini deploy_n100.yml --tags traefik
ansible-playbook -i inventory.ini deploy_n100.yml --tags adguard
ansible-playbook -i inventory.ini deploy_n100.yml --tags authentik
ansible-playbook -i inventory.ini deploy_n100.yml --tags vaultwarden
ansible-playbook -i inventory.ini deploy_n100.yml --tags portainer
ansible-playbook -i inventory.ini deploy_n100.yml --tags hardening
```

---

## Security

- **No inbound WAN/NAT ports** — HTTPS via Cloudflare DNS-01. Router/firewall remains closed; LAN service ports are intentionally exposed on the NAS.
- **SSO via Authentik** — ForwardAuth on Traefik, toggled per-service via `enable_authentik_protection`.
- **SSH hardening** — `ssh_hardening` role, `--tags hardening`.
- **Secrets** — ansible-vault only. Never plaintext, never in git.
- **Safe diffs** — `no_log: true` on secret-bearing templates. `.env` files at `0600`.
- **Break-glass** — lockout safety beats purity.


---

## See also

[`RUNBOOK.md`](RUNBOOK.md) — operational commands, incident playbooks, engineering deep dives, smoke checklist

---

## License

Licensed under the Apache License, Version 2.0.
See [`LICENSE`](LICENSE) and [`NOTICE`](NOTICE).
