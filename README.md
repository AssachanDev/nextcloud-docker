<div align="center">

# ☁️ Nextcloud Self-Hosted

**Personal cloud storage running on Docker + Tailscale**

![Nextcloud](https://img.shields.io/badge/Nextcloud-0082C9?style=for-the-badge&logo=nextcloud&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![MariaDB](https://img.shields.io/badge/MariaDB-003545?style=for-the-badge&logo=mariadb&logoColor=white)
![Tailscale](https://img.shields.io/badge/Tailscale-241F20?style=for-the-badge&logo=tailscale&logoColor=white)

</div>

---

## 📋 Prerequisites

| Requirement | Details |
|-------------|---------|
| OS | Ubuntu Server |
| Network | Tailscale installed & connected |
| Storage | NAS mounted at `/mnt/nas` |

---

## 🚀 Quick Start

### 1 — Install Docker

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
```

### 2 — Create Project Directory

```bash
mkdir -p ~/Projects/nextcloud
mkdir -p /mnt/nas/nextcloud
cd ~/Projects/nextcloud
```

### 3 — Configure Environment Variables

```bash
cp .env.example .env
nano .env
```

```env
# .env.example
MYSQL_ROOT_PASSWORD=changeme_root
MYSQL_PASSWORD=your_db_password
NEXTCLOUD_ADMIN_PASSWORD=your_admin_password
```

> [!WARNING]
> `.env` is ignored by git — **never commit real passwords.**

### 4 — Start Nextcloud

```bash
docker compose up -d
```

```bash
docker compose ps   # verify containers are running
```

### 5 — Add Trusted Domain (Tailscale IP)

> [!NOTE]
> ถ้า setup ผ่าน browser (ไม่ได้ตั้ง `NEXTCLOUD_ADMIN_USER`/`NEXTCLOUD_ADMIN_PASSWORD` ใน `.env`) Nextcloud จะ auto-add trusted domain ให้เอง ไม่ต้องรัน command นี้
>
> แต่ถ้าตั้ง admin credentials ใน `.env` ไว้ Nextcloud จะ auto-install และต้องรัน occ เองหลัง install เสร็จ

```bash
# If accessing via browser shows "untrusted domain" error:
docker exec -u 33 -it nextcloud php occ config:system:set trusted_domains 1 --value=<tailscale-ip>

# If data directory permission error appears:
sudo chown -R 33:33 /mnt/nas/nextcloud
```

> Replace `<tailscale-ip>` with your server's Tailscale IP e.g. `100.84.x.x`

### 6 — Enable Auto-start on Reboot

```bash
sudo systemctl enable docker
```

> Containers restart automatically via `restart: always` in `docker-compose.yml`

---

## 🌐 Accessing Nextcloud

Open your browser and go to:

```
http://<tailscale-ip>:8080
```

| Field | Value |
|-------|-------|
| Username | `admin` |
| Password | value of `NEXTCLOUD_ADMIN_PASSWORD` in `.env` |

---

## 📱 Desktop & Mobile Client

Download the Nextcloud client from https://nextcloud.com/install/#install-clients

Connect using server URL: `http://<tailscale-ip>:8080`

---

## 📁 File Structure

```
nextcloud/
├── 🐳 docker-compose.yml   # service definitions
├── 🔒 .env                 # your secrets (not committed)
├── 📄 .env.example         # template — safe to commit
├── 🚫 .gitignore
└── 📖 README.md
```

---

## 🛠️ Useful Commands

| Action | Command |
|--------|---------|
| View logs | `docker compose logs -f nextcloud` |
| Stop | `docker compose down` |
| Update | `docker compose pull && docker compose up -d` |
| Restart | `docker compose restart` |

---

## 📝 Notes

- Files are stored on NAS at `/mnt/nas/nextcloud/admin/files/`
- Nextcloud is **only accessible within the Tailscale network**
- On mobile: install Tailscale app and connect before opening Nextcloud

