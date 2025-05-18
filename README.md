# 🧬 Gramps Web Hosting on GCP e2-micro

> A lightweight, secure, self-hosted family tree app using Gramps Web, Docker, Caddy, and SQLite. Suitable for 2–5 users with optional backup and HTTPS.

---

## 📦 Features

- 🐳 Dockerized setup (Gramps + Caddy)
- 🔒 HTTPS via Caddy & DuckDNS
- 🧑‍💻 Lightweight (e2-micro GCP instance)
- 💾 SQLite database
- ☁️ Optional cloud backups (via `rclone`)
- 🧍 Basic multi-user support (via Caddy Basic Auth)

---

## 🖥️ 1. GCP VM Setup

- **Instance type**: `e2-micro`
- **OS**: Ubuntu 22.04 LTS
- **Disk**: 10–20 GB SSD
- **Firewall rules**:
  - Allow TCP: 22 (SSH), 80 (HTTP), 443 (HTTPS)

### Increase Swap Memory

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```
Helps prevent memory-related crashes, especially when building Docker images or using Caddy/Gramps in a low-RAM environment.

---

## 🐳 2. Install Docker & Docker Compose

```bash
sudo apt update && sudo apt install docker.io docker-compose -y
sudo usermod -aG docker $USER
newgrp docker
```
Reboot or log out/in again to apply Docker group permissions

---

## 🌐 3. DuckDNS Setup (Optional)

Follow https://www.duckdns.org/install.jsp to auto-update your IP.

---

## 📁 4. File Structure

```
gramps-deploy/
├── docker-compose.yml
├── .env
├── caddy/
│   └── Caddyfile
├── gramps/
│   └── media/ (auto-created)
└── backups/
```

---

## 🧾 5. Setup Project

```bash
git clone https://github.com/comrados/gramps-deploy
cd gramps-deploy
nano .env  # Set your admin username/password
docker-compose up -d
```

---

## 🔑 6. Optional Multi-User Auth via Caddy

Generate hashed password:

```bash
docker run --rm caddy caddy hash-password --plaintext "yourpassword"
```

Add to `Caddyfile` like:

```
yourname.duckdns.org {
  reverse_proxy gramps:5000
  basicauth {
    alice <hashed>
    bob   <hashed>
  }
}
```

---

## ♻️ 7. Backups (via cron + rclone)

Install rclone and configure remote:

```bash
sudo apt install rclone -y
rclone config
```

Edit crontab (`crontab -e`):

```cron
0 3 * * 1 docker exec gramps gramps -O <tree_name> -e /data/backups/backup_$(date "+%F").gramps
0 4 * * 1 rclone copy /home/ubuntu/gramps/backups remote:gramps-backups
```
Replace `<tree_name>` with your actual Gramps tree ID (usually visible in the UI).

---

## 🧯 8. Troubleshooting

- If `swap` is low, increase it.
- Check logs with `docker-compose logs -f`
- Ensure DuckDNS IP updates correctly
- Let Caddy handle HTTPS — don’t try to run HTTPS in Flask directly

---