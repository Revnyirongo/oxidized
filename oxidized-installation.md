# UbuntuNet Oxidized Installation Guide

This document describes the exact Oxidized setup deployed at UbuntuNet, which differs significantly from other online tutorials. This installation uses **Docker Compose with Traefik** for SSL termination and reverse proxy.

## Key Differences from Other Guides

| Aspect | Other Guides (e.g., apjone.uk) | UbuntuNet Setup |
|--------|--------------------------------|-----------------|
| **Installation Method** | Native Ruby installation | Docker Compose |
| **Web Server** | Apache/Nginx | Traefik reverse proxy |
| **SSL/TLS** | Manual certificate setup | Automatic Let's Encrypt |
| **Configuration Location** | `/home/oxidized/.config/oxidized/` | `/root/oxidized-traefik/oxidized-config/` |
| **Service Management** | systemd service | Docker containers |
| **Database Storage** | SQLite in user home | File-based in mounted volume |
| **Network Access** | Direct port access | Domain-based routing |
| **Backup Strategy** | Manual file copying | Automated Docker volume backup |

## System Overview

```
Internet → Traefik (SSL termination) → Oxidized Container
                ↓
        Let's Encrypt (automatic SSL)
                ↓  
        oxid.ubuntunet.net (public access)
```

## Installation Architecture

### **File Structure**
```
/root/oxidized-traefik/                           # Main project directory
├── docker-compose.yml                           # Container orchestration
├── oxidized-config/                             # Oxidized configuration
│   ├── config                                   # Main Oxidized config file
│   ├── router.db                                # Device inventory (CSV format)
│   ├── configs/                                 # Device backup storage
│   │   ├── device1.cfg                         # Individual device configs
│   │   └── device2.cfg
│   ├── logs/                                    # Application logs
│   └── pid                                      # Process ID file
└── letsencrypt/                                 # SSL certificate storage
    └── acme.json                                # Let's Encrypt certificates
```

### **Network Architecture**
```
traefik-network (Docker network)
├── traefik container (reverse proxy)
└── oxidized container (application)
```

## Complete Installation Procedure

### **Prerequisites**
- Ubuntu 20.04+ server
- Root access
- Domain name pointing to server IP
- Ports 80, 443, 8080 open

### **Step 1: Install Docker and Docker Compose**

```bash
# Update system
sudo apt update

# Install Docker dependencies
sudo apt install -y ca-certificates curl gnupg lsb-release

# Add Docker GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Add Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Install standalone docker-compose
sudo apt install docker-compose

# Verify installation
docker --version
docker-compose --version
```

### **Step 2: Create Project Structure**

```bash
# Create main directory
mkdir -p /root/oxidized-traefik
cd /root/oxidized-traefik

# Create subdirectories
mkdir -p oxidized-config/configs
mkdir -p letsencrypt
```

### **Step 3: Create Docker Compose Configuration**

Create `/root/oxidized-traefik/docker-compose.yml`:

```yaml
version: '3'
services:
  oxidized:
    image: oxidized/oxidized:latest
    container_name: oxidized
    environment:
      - OXIDIZED_HOME=/home/oxidized/.config/oxidized
      - OXIDIZED_WEB=true
      - OXIDIZED_OPTS=--web-host 0.0.0.0 --web-port 8888
    ports:
      - "8888:8888"
    volumes:
      - ./oxidized-config:/home/oxidized/.config/oxidized
    restart: always
    networks:
      - web
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.oxidized.rule=Host(`oxid.ubuntunet.net`)"
      - "traefik.http.routers.oxidized.entrypoints=websecure"
      - "traefik.http.routers.oxidized.tls.certresolver=myresolver"
      - "traefik.http.services.oxidized.loadbalancer.server.port=8888"
      # Remove the following line to allow public access
      # - "traefik.http.routers.oxidized.middlewares=ipwhitelist@docker"
      - "traefik.http.routers.oxidized-http.rule=Host(`oxid.ubuntunet.net`)"
      - "traefik.http.routers.oxidized-http.entrypoints=web"
      - "traefik.http.routers.oxidized-http.middlewares=https-redirect"
      - "traefik.http.middlewares.https-redirect.redirectscheme.scheme=https"

  traefik:
    image: traefik:v2.5
    container_name: traefik
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    volumes:
      - ./letsencrypt:/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock:ro
    command:
      - "--log.level=DEBUG"
      - "--api=true"
      - "--api.dashboard=true"
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.myresolver.acme.httpchallenge=true"
      - "--certificatesresolvers.myresolver.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.myresolver.acme.email=revelation.nyirongo@ubuntunet.net"
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
    restart: always
    networks:
      - web
    labels:
      - "traefik.enable=true"
      # IP whitelist for specific networks (modify as needed)
      - "traefik.http.middlewares.ipwhitelist.ipwhitelist.sourcerange=197.239.12.189/32,196.32.208.0/21,41.70.13.32/32,41.210.154.130/32,197.239.0.0/24"
      - "traefik.http.routers.dashboard.rule=Host(`traefik.oxid.ubuntunet.net`)"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.entrypoints=websecure"
      - "traefik.http.routers.dashboard.tls.certresolver=myresolver"
      - "traefik.http.routers.dashboard.middlewares=ipwhitelist@docker"

networks:
  web:
    name: traefik-network
```

### **Step 4: Create Oxidized Configuration**

Create `/root/oxidized-traefik/oxidized-config/config`:

```yaml
---
username: admin
password: admin
model: ios
interval: 3600
debug: false
input:
  default: ssh
  ssh:
    secure: false
output:
  default: file
  file:
    directory: "/home/oxidized/.config/oxidized/configs"
source:
  default: csv
  csv:
    file: "/home/oxidized/.config/oxidized/router.db"
    delimiter: ":"
    map:
      name: 0
      model: 1
      group: 2
rest: "0.0.0.0:8888"
```

### **Step 5: Create Device Inventory**

Create `/root/oxidized-traefik/oxidized-config/router.db`:

```
# Format: hostname:model:group
# Examples:
router1.ubuntunet.net:ios:core
switch1.ubuntunet.net:ios:access
firewall1.ubuntunet.net:junos:security
196.32.211.235:ios:network
```

### **Step 6: Set Permissions**

```bash
# Set correct ownership for Oxidized container
sudo chown -R 30000:30000 /root/oxidized-traefik/oxidized-config/
```

### **Step 7: Deploy Services**

```bash
# Navigate to project directory
cd /root/oxidized-traefik

# Start services
docker-compose up -d

# Verify containers are running
docker-compose ps

# Check logs
docker-compose logs traefik
docker-compose logs oxidized
```

## Configuration Details

### **Oxidized Configuration Specifics**

#### **Authentication**
- **Global credentials**: Set in `/root/oxidized-traefik/oxidized-config/config`
- **Username**: `admin` (default)
- **Password**: `admin` (default)
- **Per-device credentials**: Not used in this setup

#### **Device Inventory Format**
```
# File: /root/oxidized-traefik/oxidized-config/router.db
# Format: name:model:group
device.example.com:ios:production
192.168.1.1:junos:core
switch.local:ios:access
```

#### **Output Configuration**
- **Type**: File-based storage
- **Location**: `/root/oxidized-traefik/oxidized-config/configs/`
- **Format**: Individual `.cfg` files per device

#### **Web Interface**
- **Internal port**: 8888 (container)
- **External access**: Via Traefik reverse proxy
- **Public URL**: `https://oxid.ubuntunet.net`
- **SSL**: Automatic Let's Encrypt certificates

### **Traefik Configuration Specifics**

#### **Routing Rules**
```yaml
# HTTPS access
traefik.http.routers.oxidized.rule=Host(`oxid.ubuntunet.net`)

# HTTP to HTTPS redirect
traefik.http.routers.oxidized-http.middlewares=https-redirect
```

#### **SSL Certificates**
- **Provider**: Let's Encrypt
- **Challenge**: HTTP-01
- **Storage**: `/root/oxidized-traefik/letsencrypt/acme.json`
- **Email**: `revelation.nyirongo@ubuntunet.net`

#### **Access Control**
```yaml
# IP whitelist (optional)
traefik.http.middlewares.ipwhitelist.ipwhitelist.sourcerange=197.239.12.189/32,196.32.208.0/21,41.70.13.32/32,41.210.154.130/32,197.239.0.0/24
```

## Device Management

### **Adding New Devices**

#### **Method 1: Edit router.db directly**
```bash
# Add device to inventory
echo "newdevice.example.com:ios:production" >> /root/oxidized-traefik/oxidized-config/router.db

# Restart Oxidized to pick up changes
cd /root/oxidized-traefik
docker-compose restart oxidized
```

#### **Method 2: Via Web Interface**
1. Navigate to `https://oxid.ubuntunet.net`
2. Click "Add Node"
3. Fill in device details:
   - **Name**: Device hostname or IP
   - **Model**: Device type (ios, junos, etc.)
   - **Group**: Logical grouping
4. Save configuration

#### **Method 3: Via API**
```bash
curl -X POST -H "Content-Type: application/json" \
  -d '{"name":"device.example.com","model":"ios","group":"production"}' \
  https://oxid.ubuntunet.net/node/create
```

### **Supported Device Models**
- **ios**: Cisco IOS devices
- **junos**: Juniper devices  
- **eos**: Arista devices
- **nxos**: Cisco Nexus devices
- **iosxr**: Cisco IOS-XR devices

### **Device Authentication**
The system uses global authentication configured in the main config file. Per-device authentication is not implemented in this setup.

## Operational Procedures

### **Service Management**

```bash
# Start services
cd /root/oxidized-traefik
docker-compose up -d

# Stop services
docker-compose down

# Restart specific service
docker-compose restart oxidized
docker-compose restart traefik

# View logs
docker-compose logs -f oxidized
docker-compose logs -f traefik

# Check service status
docker-compose ps
```

### **Configuration Updates**

#### **Updating Oxidized Configuration**
```bash
# Edit main config
nano /root/oxidized-traefik/oxidized-config/config

# Restart Oxidized to apply changes
cd /root/oxidized-traefik
docker-compose restart oxidized
```

#### **Updating Device List**
```bash
# Edit device inventory
nano /root/oxidized-traefik/oxidized-config/router.db

# Restart Oxidized to pick up changes
cd /root/oxidized-traefik
docker-compose restart oxidized
```

#### **Updating Docker Configuration**
```bash
# Edit Docker Compose file
nano /root/oxidized-traefik/docker-compose.yml

# Recreate containers with new configuration
cd /root/oxidized-traefik
docker-compose down
docker-compose up -d
```

### **Monitoring and Troubleshooting**

#### **Check System Status**
```bash
# Container status
docker-compose ps

# Resource usage
docker stats

# Network connectivity
docker network ls
docker network inspect traefik-network
```

#### **View Device Configurations**
```bash
# List backed up devices
ls -la /root/oxidized-traefik/oxidized-config/configs/

# View specific device config
cat /root/oxidized-traefik/oxidized-config/configs/device.example.com
```

#### **Common Issues and Solutions**

| Issue | Symptoms | Solution |
|-------|----------|----------|
| **SSL Certificate Issues** | Browser warnings, HTTPS not working | Check Let's Encrypt logs, verify domain DNS |
| **Device Not Backing Up** | Missing config files | Verify device connectivity, check SSH credentials |
| **Web Interface Not Accessible** | Connection refused | Check Traefik routing, verify domain configuration |
| **Permission Errors** | Container startup failures | Run `sudo chown -R 30000:30000 /root/oxidized-traefik/oxidized-config/` |

#### **Log Analysis**
```bash
# Oxidized application logs
docker-compose logs oxidized | grep ERROR

# Traefik routing logs  
docker-compose logs traefik | grep oxid.ubuntunet.net

# System logs for Docker
journalctl -u docker.service

# Container inspection
docker inspect oxidized
docker inspect traefik
```

## Backup and Recovery

### **Automated Backup Setup**

Create backup script at `/root/scripts/oxidized-backup.sh`:

```bash
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/root/backups/oxidized"
mkdir -p "$BACKUP_DIR"

# Create backup
tar -czf "$BACKUP_DIR/oxidized-backup-$DATE.tar.gz" -C /root oxidized-traefik/

# Log backup
echo "$(date) - Backup created: oxidized-backup-$DATE.tar.gz" >> /var/log/oxidized-backup.log

# Cleanup old backups (keep 30 days)
find "$BACKUP_DIR" -name "oxidized-backup-*.tar.gz" -mtime +30 -delete
```

Set up daily backups:
```bash
# Make script executable
chmod +x /root/scripts/oxidized-backup.sh

# Add to crontab (daily at 2 AM)
(crontab -l 2>/dev/null; echo "0 2 * * * /root/scripts/oxidized-backup.sh") | crontab -
```

### **Recovery Procedure**

```bash
# Stop current services
cd /root/oxidized-traefik
docker-compose down

# Backup current state
mv /root/oxidized-traefik /root/oxidized-traefik-backup-$(date +%Y%m%d)

# Restore from backup
cd /root
tar -xzf /root/backups/oxidized/oxidized-backup-TIMESTAMP.tar.gz

# Set permissions
sudo chown -R 30000:30000 /root/oxidized-traefik/oxidized-config/

# Start services
cd /root/oxidized-traefik
docker-compose up -d
```

## Security Considerations

### **Access Control**
- **Web Interface**: Accessible via domain name only
- **IP Restrictions**: Configurable via Traefik middleware
- **SSL/TLS**: Enforced for all connections
- **Container Isolation**: Services run in isolated Docker network

### **Network Security**
```yaml
# Restrict access to specific IP ranges
traefik.http.middlewares.ipwhitelist.ipwhitelist.sourcerange=YOUR_IP_RANGES
```

### **Authentication**
Currently uses basic authentication configured in Oxidized. For enhanced security, consider:
- Implementing Traefik basic auth middleware
- Integrating with external authentication systems
- Using SSH key-based device authentication

## Maintenance

### **Regular Maintenance Tasks**

#### **Weekly**
- Review backup logs
- Check SSL certificate status
- Monitor disk usage in `/root/oxidized-traefik/oxidized-config/configs/`

#### **Monthly**
- Update Docker images
- Review device connectivity
- Clean up old logs

#### **Update Procedure**
```bash
# Pull latest images
cd /root/oxidized-traefik
docker-compose pull

# Recreate containers with updated images
docker-compose down
docker-compose up -d

# Verify services are working
docker-compose ps
```

## Support and Documentation

### **Web Interfaces**
- **Oxidized**: `https://oxid.ubuntunet.net`
- **Traefik Dashboard**: `https://traefik.oxid.ubuntunet.net` (IP restricted)

### **Configuration Files Reference**
- **Main Config**: `/root/oxidized-traefik/docker-compose.yml`
- **Oxidized Config**: `/root/oxidized-traefik/oxidized-config/config`
- **Device List**: `/root/oxidized-traefik/oxidized-config/router.db`
- **Device Configs**: `/root/oxidized-traefik/oxidized-config/configs/`

### **Port Reference**
- **80**: HTTP (redirects to HTTPS)
- **443**: HTTPS (Oxidized web interface)
- **8080**: Traefik dashboard
- **8888**: Oxidized internal port (not directly accessible)

This documentation reflects the actual UbuntuNet Oxidized deployment and should be used instead of generic online tutorials when working with this specific installation.
