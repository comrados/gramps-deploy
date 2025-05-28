# Gramps Web Hosting on Oracle Cloud VM.Standard.E2.1.Micro

A lightweight, secure, self-hosted family tree app using Gramps Web, Docker, Caddy, and SQLite. Suitable for 2–5 users with optional backup and HTTPS.



## Features

- Dockerized setup (Gramps + Caddy)
- HTTPS via Caddy & DuckDNS
- Lightweight (VM.Standard.E2.1.Micro Always Free instance)
- SQLite database
- Optional cloud backups (via `rclone`)
- Basic multi-user support (via Caddy Basic Auth)



## 1. Oracle Cloud VM Setup

- **Instance type**: `VM.Standard.E2.1.Micro` (Always Free-eligible)
- **Memory**: 1 GB RAM
- **OS**: Ubuntu 22.04 Minimal
- **Boot Volume**: 47 GB (Always Free eligible)
- **Network Security List Rules**:
  - Allow TCP: 22 (SSH), 80 (HTTP), 443 (HTTPS)

### Create Instance

1. Sign in to Oracle Cloud Console
2. Navigate to Compute > Instances
3. Click "Create Instance"
4. Select "VM.Standard.E2.1.Micro" shape
5. Choose Ubuntu 22.04 Minimal image
6. Configure VCN and subnet (use default)
7. Add your SSH public key
8. Create the instance

### Configure Network Security

1. Go to Networking > Virtual Cloud Networks
2. Select your VCN > Security Lists > Default Security List
3. Add Ingress Rules:
   - Source: 0.0.0.0/0, Protocol: TCP, Port: 80
   - Source: 0.0.0.0/0, Protocol: TCP, Port: 443

### Increase Swap Memory

```bash
# Create 2GB swap file (essential for small instances with only 1GB RAM)
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Make swap permanent
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Verify swap is active
free -h
```

This helps prevent memory-related crashes in the 1GB RAM environment, especially when building Docker images or running Caddy/Gramps simultaneously.

### Install necessary packages

```bash
sudo apt install -y curl git nano ufw cron
```



## 2. Install Docker & Docker Compose

`Ubuntu 22.04` offers multiple ways to install Docker. Here's the recommended approach using the official Docker repository for the latest stable version:

```bash
# Install Docker using the convenience script (simplest method)
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Verify Docker installation
sudo docker --version

# Install Docker Compose
sudo apt update
sudo apt install -y docker-compose-plugin

# Verify Docker Compose installation
docker compose version

# Add your user to the docker group (optional, recommended)
sudo usermod -aG docker $USER
# Note: You'll need to log out and back in for this to take effect
# Until then, you can use 'sudo docker' instead of just 'docker'
```

If you prefer the traditional Docker Compose standalone binary instead of the plugin:

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/v2.24.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version
```



## 3. Configure Ubuntu Firewall

```bash
sudo ufw allow ssh
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```



## 4. DuckDNS Setup (Optional)

Follow https://www.duckdns.org/install.jsp to auto-update your IP address for dynamic DNS.



## 5. Setup Project

### Download & Prepare

```bash
git clone https://github.com/comrados/gramps-deploy
cd gramps-deploy
nano .env  # Set your admin username/password
```

Create `.env` file:

```
GRAMPSWEB_USERNAME=admin
GRAMPSWEB_PASSWORD=securepassowrd
```

Modify `Caddyfile`:

```
<yourname>.duckdns.org {
    reverse_proxy http://gramps:5000
}
```

Project Structure:

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

### (Optional) Watchtower

By default, Watchtower is disabled in docker-compose.yml to ensure stable operation and avoid unexpected container restarts during active use.

If you want to enable automatic updates for Gramps Web (and other services), uncomment the following block in your docker-compose.yml:

```
# watchtower:
#   image: containrrr/watchtower
#   container_name: watchtower
#   restart: always
#   volumes:
#     - /var/run/docker.sock:/var/run/docker.sock
#   environment:
#     - WATCHTOWER_CLEANUP=true
#     - TZ=Europe/Moscow
#   command: --schedule "0 5 * * *"
```

> With this configuration, Watchtower will check for updates daily at 5:00 AM and automatically restart any containers with new versions.

### Run the project

```bash
docker compose up -d
```


## 6. Optional Multi-User Auth via Caddy

Generate hashed password:

```bash
docker run --rm caddy caddy hash-password --plaintext "yourpassword"
```

Add to `Caddyfile`:

```
<yourname>.duckdns.org {
  reverse_proxy http://gramps:5000
  basicauth {
    alice <hashed_password>
    bob   <hashed_password>
  }
}
```



## 7. Backups (via cron + rclone)

Install rclone and configure remote storage:

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


## 8. Oracle Cloud Specific Considerations

### Always Free Tier Limits
- 1 GB RAM (shared with system processes)
- 1/8 OCPU (burstable)
- 47 GB boot volume storage
- 10 TB outbound data transfer per month

### Memory Optimization
Due to the 1 GB RAM limitation, monitor memory usage:

```bash
# Check memory usage
free -h
# Monitor Docker containers
docker stats
```

Consider stopping unused services and optimizing Docker images for minimal memory footprint.

### Instance Management
- Always Free instances can be stopped/started without losing the Always Free status
- Avoid accidentally terminating the instance (use stop instead)
- Monitor Oracle Cloud usage to stay within Always Free limits



## 9. Troubleshooting

- If memory usage is high, increase swap space or restart containers
- Check logs with `docker-compose logs -f`
- Ensure DuckDNS IP updates correctly
- Let Caddy handle HTTPS automatically
- Monitor Oracle Cloud Always Free usage limits
- If instance becomes unresponsive, use Oracle Cloud Console to restart

### Common Oracle Cloud Issues
- **SSH Connection Refused**: Check Security List rules and UFW settings
- **Out of Memory**: Increase swap, restart containers, or optimize configuration
- **HTTPS Not Working**: Verify port 443 is open in both Security Lists and UFW



## 10. Monitoring Always Free Usage

Regularly check Oracle Cloud Console for:
- Compute usage hours
- Storage usage
- Network data transfer
- Ensure you stay within Always Free tier limits to avoid charges