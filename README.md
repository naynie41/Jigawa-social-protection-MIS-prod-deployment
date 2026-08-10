# Production Deployment — Containerised Web Platform on a Single VPS

Taking a bare Ubuntu VPS to a **hardened, CI/CD-driven, monitored, backed-up production deployment** for a multi-service containerised web application.

This repository documents the infrastructure and delivery layer I built for a **social-protection management information system (SP-MIS)** — a multi-tenant government platform handling sensitive personal data under national data-protection law. The application code is private; **this repo is the DevOps work**, generalised and stripped of any client-specific detail so it can be read, cloned, and reused.

> **Scope:** infrastructure, provisioning, CI/CD, TLS, backups, and observability — not application code. All hostnames, IPs, domains, and credentials here are placeholders.

---

## Architecture

![Deployment architecture](docs/architecture.svg)

A single VPS runs the full stack as **seven Docker containers** on one private bridge network. The design goal throughout: **minimise public attack surface, make every deploy reproducible and reversible, and treat the data as if a breach were a career-ending event** — because under NDPA/NDPR, it effectively is.

| Layer | Technology | Public? |
|---|---|---|
| Reverse proxy / TLS | Nginx | **Yes — the only public service** (80/443) |
| API | Laravel 12 (PHP) | No — internal network |
| Web | React + TypeScript (SPA, served by Nginx) | No |
| Background jobs | Laravel queue worker (same image as API) | No |
| Database | PostgreSQL + PostGIS | **No public port** |
| Cache / sessions | Redis | **No public port** |
| Message broker | RabbitMQ | **No public port** |

---

## Key design decisions

The commands are the easy part. These are the decisions that make it production-grade:

### Only the reverse proxy is public
Postgres, Redis, and RabbitMQ have **no published ports at all** — they're reachable only by service name on the internal Docker network. This matters more than it looks: Docker manipulates `iptables` directly and can **bypass a UFW firewall** for any port it publishes. Relying on the host firewall alone to protect the database is a trap. The mitigation is architectural, not firewall config — don't publish the port in the first place.

### The host never builds images
CI builds and pushes to the registry; the **production host only ever pulls**. This keeps the server's toolchain minimal (smaller attack surface), makes builds reproducible, and means the exact artifact tested in CI is the one that runs in prod — no "works on my machine" drift.

### Deploy from pinned tags, never `latest`
Every deploy references an explicit version tag (`:1.4.0`, or a commit SHA). `latest` gives you no way to say *"put back exactly what ran yesterday."* Rollback is then just: repin the previous tag, pull, `up -d`. See [`scripts/rollback.sh`](scripts/rollback.sh).

### SSH hardening that can't lock you out
Root login and password auth are disabled, key-only. But the [provisioning script](scripts/provision.sh) validates the SSH config with `sshd -t` **before** restarting the daemon, and reverts on failure — a typo can't leave you locked out of your own box. Hardening lives in a `sshd_config.d/` drop-in so it survives package updates that rewrite the main config.

### Backups are worthless until a restore is tested
Nightly `pg_dump` → encrypted → offsite, with local pruning. But the important discipline is that **an untested backup is a hypothesis, not a backup** — the restore path is verified into a scratch container before go-live. Provider snapshots exist too, but as coarse whole-machine rollback points, *not* as the data-protection backup. Different tools, different jobs.

### Observability lives partly off-host
An uptime monitor that runs *on* the box it watches can't tell you the box is down. External probing plus on-host healthchecks and capped logging give a complete picture without one blind spot covering for another.

---

## The deployment, stage by stage

The full path from bare server to live, in dependency order. Each stage gates the next.

| # | Stage | Artifact |
|---|---|---|
| 0 | Verify SSH access, sanity-check specs | — |
| 1 | System update + non-root `deploy` user | [`provision.sh`](scripts/provision.sh) |
| 2 | SSH hardening + UFW + fail2ban + auto-patches | [`provision.sh`](scripts/provision.sh) |
| 3 | Docker Engine + Compose (official repo) | [`provision.sh`](scripts/provision.sh) |
| — | **Snapshot #1** — clean hardened baseline | — |
| 4 | Registry (GHCR) authentication — read-only token | — |
| 5 | Production Docker Compose | [`docker-compose.prod.yml`](docker-compose.prod.yml) |
| 6 | Nginx reverse proxy | [`nginx/conf.d/app.conf`](nginx/conf.d/app.conf) |
| 7 | DNS A-record → server IP | — |
| 8 | HTTPS via Let's Encrypt / Certbot | [`nginx/`](nginx/) |
| 9 | Manual deploy (pull + migrate) | [`deploy.sh`](scripts/deploy.sh) |
| — | **Backups live + restore-tested** (before real data) | [`backup-db.sh`](scripts/backup-db.sh) |
| 10 | GitHub Actions CI/CD | [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml) |
| 11 | Rollback procedure | [`rollback.sh`](scripts/rollback.sh) |
| 12 | Auto-start on reboot | [`systemd/app.service`](systemd/app.service) |
| 13 | Monitoring / logging | [`monitoring/`](monitoring/) |
| 14 | Pre-go-live smoke test | [`docs/runbook.md`](docs/runbook.md) |

> Backups are physically set up wherever convenient, but must be **live and restore-tested before any real data loads** — so they appear late in the *build* order but early in the *readiness* order.

---

## CI/CD pipeline

![CI/CD pipeline](docs/cicd-pipeline.svg)

On push to `main`, GitHub Actions builds each service image, tags it with the commit SHA, and pushes to the container registry. A deploy job then SSHes to the host, which **pulls the pinned images and restarts only the changed containers** — Postgres/Redis/RabbitMQ keep running untouched while API/web are swapped. Migrations run with `--force`; old images are pruned.

The host authenticates to the registry with a **read-only** token — it can pull but never push or delete. Deploy credentials live in GitHub Actions secrets, never in the repo.

---

## Repository structure

```
.
├── README.md                      ← you are here
├── docker-compose.prod.yml        ← production topology (dummy values)
├── .env.example                   ← every variable documented, no real secrets
├── nginx/
│   └── conf.d/app.conf            ← reverse proxy + TLS 1.2+ config
├── .github/workflows/
│   └── deploy.yml                 ← build → registry → SSH pull pipeline
├── scripts/
│   ├── provision.sh               ← Stages 1–3: harden + firewall + Docker
│   ├── deploy.sh                  ← Stage 9: pull + migrate + cache
│   ├── backup-db.sh               ← nightly pg_dump + offsite + prune
│   └── rollback.sh                ← repin previous tag
├── systemd/
│   └── app.service                ← auto-start the stack on reboot
├── monitoring/
│   └── ...                        ← healthchecks, uptime, log config
└── docs/
    ├── architecture.svg           ← the diagram above
    ├── cicd-pipeline.svg
    └── runbook.md                 ← full step-by-step deployment guide
```

---

## Reproducing this

This is a real topology with placeholder values — you can adapt it to any multi-service containerised app.

```bash
# 1. Provision a fresh Ubuntu 22.04/24.04 VPS
scp scripts/provision.sh root@<server-ip>:/root/
ssh root@<server-ip>
DEPLOY_USER=deploy ./provision.sh

# 2. Verify key login works for the deploy user BEFORE closing root
ssh deploy@<server-ip>        # must succeed
ssh root@<server-ip>          # must be refused

# 3. Copy compose + env, fill in real values
cp .env.example .env          # then edit; chmod 600 .env

# 4. Authenticate to the registry (read-only token), pull, start
docker login ghcr.io -u <user> --password-stdin < token.txt
docker compose -f docker-compose.prod.yml pull
docker compose -f docker-compose.prod.yml up -d
```

Full walkthrough including TLS issuance and CI/CD wiring in [`docs/runbook.md`](docs/runbook.md).

---

## Security & data-protection posture

Because the underlying system handles personal data under **NDPA/NDPR**, the deployment treats data protection as a first-class requirement, not an afterthought:

- **TLS 1.2+** enforced at the edge; HTTP redirects to HTTPS.
- **No public ports** on any data service — DB/cache/broker are network-isolated.
- **Encryption at rest** (provider volume) for the database; **encrypted** offsite backups.
- **Key-only SSH**, root login disabled, brute-force protection, unattended security patches.
- **Least-privilege registry access** — the host token is read-only.
- **Secrets never in Git** — `.env` is git-ignored; only `.env.example` is committed.
- **Immutable audit logging** at the application layer, included in the backup scope.

---

## What I'd do next (scaling path)

This is deliberately a **single-VPS** topology — correct for the current stage, and honest about it. The moment the platform needs true zero-downtime or horizontal scale, the next moves are:

- Split the app tier across two hosts behind a load balancer (the data-services-are-private design already supports this).
- Move Postgres to a managed/replicated instance with automated failover.
- Add a full metrics stack (Prometheus + Grafana + Alertmanager) as traffic justifies the maintenance cost.
- Container orchestration (Kubernetes / Nomad) *only* once multi-host scheduling is a real requirement — not before.

Knowing when *not* to reach for the heavier tool is part of the job.

---

*Placeholders (`<server-ip>`, `ghcr.io/yourorg`, `app.example.gov.ng`) stand in for real values. No client names, IPs, domains, or credentials appear in this repository or its history.*
