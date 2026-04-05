<div align="center">

# ☁️ Nextcloud Self-Hosted

**Production-ready Nextcloud on Docker + Tailscale**

[![Nextcloud](https://img.shields.io/badge/Nextcloud_28-0082C9?style=for-the-badge&logo=nextcloud&logoColor=white)](https://nextcloud.com)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://docker.com)
[![MariaDB](https://img.shields.io/badge/MariaDB_10.11-003545?style=for-the-badge&logo=mariadb&logoColor=white)](https://mariadb.org)
[![Tailscale](https://img.shields.io/badge/Tailscale-241F20?style=for-the-badge&logo=tailscale&logoColor=white)](https://tailscale.com)

</div>

---

## What This Is

A self-hosted Nextcloud setup that runs entirely inside a private Tailscale network — no public internet exposure. All credentials are managed through environment variables, nothing is hardcoded.

This repo ships two compose files:

| File | What it does |
|------|-------------|
| `docker-compose.yml` | Spins up **one** Nextcloud instance |
| `docker-compose.multi.yml` | Spins up **multiple isolated** Nextcloud instances on the same server |

---

## When to Use Which

**Use `docker-compose.yml` when:**
- You need one cloud storage for one person or one team
- Simple setup, one URL, one admin panel

**Use `docker-compose.multi.yml` when:**
- You have multiple organizations or departments on the same server
- Each group needs **completely separate storage** — they cannot see each other's files
- You want one server to serve many tenants, each with their own login, their own data, their own admin

> Each tenant in the multi setup is a fully independent Nextcloud — separate database, separate filesystem, separate port. Adding more tenants is just copying a block in the compose file and adding the corresponding variables to `.env`.

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| OS | Ubuntu Server 22.04+ |
| Runtime | Docker + Docker Compose |
| Network | Tailscale installed and connected |
| Storage | NAS or any mounted path for data persistence |

**Install Docker:**
```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER && newgrp docker
```

---

## Single Instance — `docker-compose.yml`

```
server
└── nextcloud  →  :8080  →  /your/data/path/
```

**1. Configure environment:**
```bash
cp .env.example .env
nano .env
```

Fill in the single-instance section:
```env
PORT=8080
ADMIN_USER=admin
ADMIN_PASSWORD=your_strong_password

MYSQL_ROOT_PASSWORD=your_strong_password
MYSQL_PASSWORD=your_strong_password

DATA_PATH=/mnt/nas/nextcloud
```

**2. Create data directory:**
```bash
sudo mkdir -p /mnt/nas/nextcloud
sudo chown -R 33:33 /mnt/nas/nextcloud
```

**3. Start:**
```bash
docker compose up -d
docker compose ps
```

**4. Add trusted domain:**
```bash
docker exec -u 33 -it nextcloud php occ \
  config:system:set trusted_domains 1 --value=<tailscale-ip>
```

**5. Access:** `http://<tailscale-ip>:8080`

---

## Multi-Tenant — `docker-compose.multi.yml`

Each tenant is a completely independent Nextcloud instance. They share the same server but nothing else — not the database, not the filesystem, not the admin panel.

```
server
├── tenant-a  →  :9090  →  /your/data/path/company-a/
├── tenant-b  →  :9091  →  /your/data/path/company-b/
└── tenant-c  →  :9092  →  /your/data/path/company-c/   ← just copy the block
```

**1. Configure environment:**

Fill in the multi-tenant section in `.env`:
```env
DATA_PATH=/mnt/nas/nextcloud

TENANT_A_NAME=company-a
TENANT_A_PORT=9090
TENANT_A_ADMIN_PASSWORD=your_strong_password
TENANT_A_MYSQL_ROOT_PASSWORD=your_strong_password
TENANT_A_MYSQL_PASSWORD=your_strong_password

TENANT_B_NAME=company-b
TENANT_B_PORT=9091
TENANT_B_ADMIN_PASSWORD=your_strong_password
TENANT_B_MYSQL_ROOT_PASSWORD=your_strong_password
TENANT_B_MYSQL_PASSWORD=your_strong_password
```

> **Tenant names** must be lowercase with no spaces — they are used as database names and Docker container names.

**2. Create data directories:**
```bash
sudo mkdir -p /mnt/nas/nextcloud/company-a
sudo mkdir -p /mnt/nas/nextcloud/company-b
sudo chown -R 33:33 /mnt/nas/nextcloud/
```

**3. Start:**
```bash
docker compose -f docker-compose.multi.yml up -d
docker compose -f docker-compose.multi.yml ps
```

**4. Add trusted domain for each tenant:**
```bash
docker exec -u 33 -it company-a-nextcloud php occ \
  config:system:set trusted_domains 1 --value=<tailscale-ip>

docker exec -u 33 -it company-b-nextcloud php occ \
  config:system:set trusted_domains 1 --value=<tailscale-ip>
```

**5. Access:**

| Tenant | URL |
|--------|-----|
| company-a | `http://<tailscale-ip>:9090` |
| company-b | `http://<tailscale-ip>:9091` |

### Adding More Tenants

To add a third (or fourth, or fifth) tenant:

1. Copy a tenant block in `docker-compose.multi.yml` and rename `tenant-b` → `tenant-c`
2. Add the corresponding variables to `.env` (`TENANT_C_NAME`, `TENANT_C_PORT`, etc.)
3. Create the data directory and fix ownership
4. Run `docker compose -f docker-compose.multi.yml up -d` again
5. Set the trusted domain for the new container

---

## User & Permission Management

Each instance has its own admin panel. Login with `admin` and the password you set in `.env`.

**Recommended group structure:**

| Group | Access |
|-------|--------|
| `admin` | Full access — sees all files across all users |
| `employee` | Sees only their own folder and files explicitly shared with them |

**To create groups and users:**
1. Login as admin → **Settings → Users**
2. Add groups from the left sidebar
3. Create user accounts and assign them to a group

---

## File Structure

```
nextcloud/
├── docker-compose.yml        # single instance
├── docker-compose.multi.yml  # multi-tenant
├── .env                      # your secrets — not committed
├── .env.example              # template — safe to commit
├── .gitignore
└── README.md
```

---

## Useful Commands

| Action | Single instance | Multi-tenant |
|--------|----------------|--------------|
| Start | `docker compose up -d` | `docker compose -f docker-compose.multi.yml up -d` |
| Stop | `docker compose down` | `docker compose -f docker-compose.multi.yml down` |
| Logs | `docker compose logs -f nextcloud` | `docker compose -f docker-compose.multi.yml logs -f tenant-a` |
| Update | `docker compose pull && docker compose up -d` | same with `-f docker-compose.multi.yml` |
| Restart | `docker compose restart` | `docker compose -f docker-compose.multi.yml restart` |

**Enable auto-start on reboot:**
```bash
sudo systemctl enable docker
# Containers use restart: always — they come back up automatically
```

> [!TIP]
> Generate strong passwords with: `openssl rand -base64 16`

> [!WARNING]
> `.env` is git-ignored. **Never commit real credentials.**

---

## Notes

- Nextcloud is only reachable inside the Tailscale network — never exposed to the public internet
- In multi-tenant mode, tenants are isolated at both the database and filesystem level — no data can cross between them
- Data persists across container restarts and image updates as long as `DATA_PATH` is intact
- Mobile access: install the [Tailscale app](https://tailscale.com/download) first, then connect via the [Nextcloud client](https://nextcloud.com/install/#install-clients)
