Ai is fucking crazy for them readmes

```bash
kubectl create secret generic auth-key \            
  --from-literal=TS_AUTHKEY=<key> \
  -n headscale

kubectl create secret generic tailscale-auth \
  --from-literal=auth-key=YOUR_NEW_KEY \
  -n headscale
```

# Complete Headscale VPS Setup on DigitalOcean

## Phase 1: Create DigitalOcean Droplet

### 1. Sign Up & Create Droplet
1. Go to [DigitalOcean](https://digitalocean.com) and create an account
2. Click **"Create"** → **"Droplets"**
3. Choose these settings:
   - **Image**: Ubuntu 22.04 (LTS) x64
   - **Plan**: Basic ($6/month - 1GB RAM, 1 vCPU, 25GB SSD)
   - **Region**: Choose closest to your location
   - **Authentication**: SSH Key (recommended) or Password
   - **Hostname**: `headscale-server`

### 2. Configure SSH Key (Recommended)
If you don't have an SSH key:
```bash
# On your local machine
ssh-keygen -t ed25519 -C "your-email@example.com"
cat ~/.ssh/id_ed25519.pub
```
Copy the output and paste it in DigitalOcean's SSH key field.

## Phase 2: Initial VPS Setup

### 1. Connect to Your VPS
```bash
ssh root@YOUR_VPS_IP
```

### 2. Update System & Install Essentials
```bash
# Update package list and upgrade
apt update && apt upgrade -y

# Install essential packages
apt install -y curl wget git nano ufw fail2ban

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Install Docker Compose
apt install -y docker-compose-plugin

# Start and enable Docker
systemctl start docker
systemctl enable docker
```

### 3. Secure the Server
```bash
# Configure firewall
ufw default deny incoming
ufw default allow outgoing
ufw allow ssh
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 3478/udp  # STUN for Headscale DERP
ufw --force enable

# Configure fail2ban for SSH protection
systemctl start fail2ban
systemctl enable fail2ban
```

### 4. Create Non-Root User (Optional but Recommended)
```bash
# Create user
adduser headscale
usermod -aG sudo headscale # or wheel
usermod -aG headscale
usermod -aG docker headscale

# Copy SSH keys to new user
mkdir -p /home/headscale/.ssh
cp /root/.ssh/authorized_keys /home/headscale/.ssh/
chown -R headscale:headscale /home/headscale/.ssh
chmod 700 /home/headscale/.ssh
chmod 600 /home/headscale/.ssh/authorized_keys
```

## Phase 3: Domain Setup

### 1. Configure DNS
In your DNS provider (Cloudflare, etc.):
- Create an **A record** pointing `vpn.yourdomain.com` to your VPS IP
- **Important**: Set to "DNS only" (gray cloud) if using Cloudflare - don't proxy it

### 2. Test DNS Resolution
```bash
# On your VPS, test the domain resolves
nslookup vpn.yourdomain.com
```

## Phase 4: Headscale Installation

### 1. Create Directory Structure
```bash
# Switch to headscale user (or stay as root)
su - headscale  # Skip if staying as root

# Create directories
mkdir -p /opt/headscale/{config,data}
cd /opt/headscale
```

### 2. Create Docker Compose File
```yaml
# /opt/headscale/docker-compose.yml
version: '3.8'

services:
  headscale:
    image: headscale/headscale:v0.26.0
    container_name: headscale
    restart: unless-stopped
    ports:
      - "8080:8080"  # Headscale API
      - "9090:9090"  # Metrics (optional)
      - "3478:3478/udp"  # STUN
    volumes:
      - ./config:/etc/headscale
      - ./data:/var/lib/headscale
    environment:
      - TZ=America/Denver  # Change to your timezone
    command: headscale serve
    
  caddy:
    image: caddy:2-alpine
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config
    environment:
      - DOMAIN=vpn.yourdomain.com  # Change this!

volumes:
  caddy_data:
  caddy_config:
```

### 3. Create Headscale Configuration
```yaml
# /opt/headscale/config/config.yaml
server_url: https://vpn.yourdomain.com  # Change this!
listen_addr: 0.0.0.0:8080
metrics_listen_addr: 0.0.0.0:9090

# Disable built-in TLS (Caddy handles it)
tls_cert_path: ""
tls_key_path: ""

# DERP settings
derp:
  server:
    enabled: true
    region_id: 999
    region_code: "homelab"
    region_name: "Home Lab"
    stun_listen_addr: "0.0.0.0:3478"

# Database
database:
  type: sqlite3
  sqlite:
    path: /var/lib/headscale/db.sqlite

# Logging
log:
  level: info

# IP prefixes
ip_prefixes:
  - fd7a:115c:a1e0::/48
  - 100.64.0.0/10

# DNS settings
dns:
  magic_dns: true
  base_domain: yourdomain.com  # Change this!
  nameservers:
    global:
      - 1.1.1.1
      - 8.8.8.8

# Disable built-in HTTPS redirect
disable_check_updates: true
```

### 4. Create Caddyfile
```caddyfile
# /opt/headscale/Caddyfile
vpn.yourdomain.com {  # Change this!
    reverse_proxy headscale:8080
    
    # Headers for WebSocket support
    header_up Host {host}
    header_up X-Real-IP {remote_host}
    header_up X-Forwarded-For {remote_host}
    header_up X-Forwarded-Proto {scheme}
    
    # Logging
    log {
        output file /var/log/caddy/access.log {
            roll_size 100mb
            roll_keep 3
        }
    }
}
```

## Phase 5: Start Services

### 1. Set Correct Permissions
```bash
# Make sure headscale user owns the files
sudo chown -R headscale:headscale /opt/headscale
```

### 2. Start Everything
```bash
cd /opt/headscale
docker compose up -d
```

### 3. Check Status
```bash
docker compose logs -f
```

## Phase 6: Configure Headscale

### 1. Create Your First User
```bash
docker compose exec headscale headscale users create admin
```

### 2. Generate Pre-auth Key
```bash
docker compose exec headscale headscale preauthkeys create --user admin --reusable --expiration 24h
```
Save this key - you'll need it to connect devices!

### 3. Test Web Interface
Visit `https://vpn.yourdomain.com` - you should see Headscale's info page with no errors.

## Phase 7: Connect Your First Device

### 1. Install Tailscale on Your Device
```bash
# Linux/Mac
curl -fsSL https://tailscale.com/install.sh | sh

# Or download from https://tailscale.com/download
```

### 2. Connect Using Your Headscale Server
```bash
sudo tailscale up --login-server=https://vpn.yourdomain.com --authkey=YOUR_PREAUTH_KEY
```

### 3. Verify Connection
```bash
# On your VPS
docker compose exec headscale headscale nodes list

# On your device
tailscale status
```

## Phase 8: Monitoring & Maintenance

### 1. Set Up Log Rotation
```bash
# Create logrotate config
sudo tee /etc/logrotate.d/headscale <<EOF
/opt/headscale/logs/*.log {
    weekly
    missingok
    rotate 52
    compress
    delaycompress
    notifempty
    create 644 headscale headscale
}
EOF
```

### 2. Create Backup Script
```bash
#!/bin/bash
# /opt/headscale/backup.sh
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/opt/headscale/backups"
mkdir -p $BACKUP_DIR

# Backup database and config
tar -czf $BACKUP_DIR/headscale_backup_$DATE.tar.gz \
    /opt/headscale/config \
    /opt/headscale/data

# Keep only last 7 backups
find $BACKUP_DIR -name "headscale_backup_*.tar.gz" -mtime +7 -delete

echo "Backup completed: headscale_backup_$DATE.tar.gz"
```

```bash
chmod +x /opt/headscale/backup.sh

# Add to crontab for daily backups
crontab -e
# Add: 0 2 * * * /opt/headscale/backup.sh
```

## Phase 9: Useful Commands

### Managing Users
```bash
# List users
docker compose exec headscale headscale users list

# Create user
docker compose exec headscale headscale users create USERNAME

# Delete user
docker compose exec headscale headscale users destroy USERNAME
```

### Managing Devices
```bash
# List all devices
docker compose exec headscale headscale nodes list

# Delete device
docker compose exec headscale headscale nodes delete NODE_ID

# Move device to different user
docker compose exec headscale headscale nodes move NODE_ID NEW_USER
```

### Managing Pre-auth Keys
```bash
# List keys
docker compose exec headscale headscale preauthkeys list

# Create reusable key (good for multiple devices)
docker compose exec headscale headscale preauthkeys create --user USER --reusable --expiration 168h

# Create one-time key
docker compose exec headscale headscale preauthkeys create --user USER --expiration 1h
```

### Updating Headscale
```bash
cd /opt/headscale
docker compose pull
docker compose up -d
```

## Troubleshooting

### Check Logs
```bash
# All services
docker compose logs

# Specific service
docker compose logs headscale
docker compose logs caddy
```

### Test Connectivity
```bash
# Test from outside
curl -I https://vpn.yourdomain.com

# Test WebSocket (should return upgrade info)
curl -H "Upgrade: websocket" -H "Connection: upgrade" https://vpn.yourdomain.com/ts2021
```

### Common Issues
- **Certificate issues**: Check domain DNS, ensure A record points to VPS IP
- **Firewall blocking**: Verify ufw rules with `ufw status`
- **Port conflicts**: Make sure ports 80/443 aren't used by other services
- **WebSocket issues**: This setup should work unlike Cloudflare Tunnel!

## Cost Breakdown
- **VPS**: $6/month (DigitalOcean Basic Droplet)
- **Domain**: $10-15/year (if you don't have one)
- **Total**: ~$6-7/month

This gives you a professional, always-available Tailscale alternative that works perfectly with all devices and doesn't have the Cloudflare Tunnel limitations!
