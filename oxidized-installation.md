# Oxidized with Traefik Documentation

This document provides a comprehensive guide for managing the Oxidized network configuration backup system with Traefik reverse proxy.

## System Overview

This installation uses:
- **Oxidized**: Network device configuration backup tool
- **Traefik**: Reverse proxy handling SSL termination and routing
- **Docker & Docker Compose**: Container management

## Installation Details

### Directory Structure
```
~/oxidized-traefik/
├── docker-compose.yml         # Container configuration
├── letsencrypt/               # SSL certificates
└── oxidized-config/           # Oxidized configuration
    ├── config                 # Main configuration file
    ├── configs/               # Backed up device configurations
    └── router.db              # Device inventory
```

### Configuration Files

#### Main Oxidized Configuration (`oxidized-config/config`)
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

#### Docker Compose Configuration (`docker-compose.yml`)
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
      - "traefik.http.routers.oxidized-http.rule=Host(`oxid.xxxx.net`)"
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
      - "--certificatesresolvers.myresolver.acme.email=devops@xxxx.com"
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
    restart: always
    networks:
      - web
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.dashboard.rule=Host(`traefik.oxid.xxxxx.net`)"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.entrypoints=websecure"
      - "traefik.http.routers.dashboard.tls.certresolver=myresolver"
      - "traefik.http.middlewares.auth.basicauth.users=admin:$apr1$ruca84Hq$mbjdMZBAG.KWn7vfN/SNK/"

networks:
  web:
    name: traefik-network
```

## Managing Network Devices

### Adding Devices

To add network devices for backup:

1. Edit the router.db file:
   ```bash
   nano ~/oxidized-traefik/oxidized-config/router.db
   ```

2. Add entries in the format: `hostname_or_ip:device_model:group`
   
   Example:
   ```
   192.168.1.1:ios:routers
   10.0.0.1:junos:switches
   172.16.0.1:procurve:datacenter
   ```

3. Update the credentials in the main config file if necessary:
   ```bash
   nano ~/oxidized-traefik/oxidized-config/config
   ```

4. Force Oxidized to reload its configuration:
   ```bash
   docker exec -it oxidized kill -HUP 1
   ```

### Supported Device Models

Oxidized supports many device types including:
- `ios`: Cisco IOS
- `iosxr`: Cisco IOS XR
- `junos`: Juniper JunOS
- `eos`: Arista EOS
- `routeros`: MikroTik RouterOS
- `procurve`: HP ProCurve
- `fortigate`: Fortinet FortiOS
- `asr`: Cisco ASR
- `panos`: Palo Alto Networks PAN-OS

For a complete list, refer to the [Oxidized GitHub repository](https://github.com/ytti/oxidized/tree/master/lib/oxidized/model).

### Authentication Methods

You can configure different authentication methods in the main config file:

```yaml
username: admin
password: admin

# For SSH keys
input:
  default: ssh
  ssh:
    secure: false
    keys: /home/oxidized/.ssh/id_rsa
```

## System Maintenance

### Viewing Logs

```bash
# Oxidized logs
docker-compose logs -f oxidized

# Traefik logs
docker-compose logs -f traefik
```

### Updating the System

```bash
# Pull the latest images
docker-compose pull

# Restart the services
docker-compose down
docker-compose up -d
```

### Backing Up Configuration

It's recommended to back up your configuration files regularly:

```bash
# Create a backup
tar -czf oxidized-backup-$(date +%Y%m%d).tar.gz ~/oxidized-traefik/oxidized-config ~/oxidized-traefik/docker-compose.yml
```

## Firewall Configuration (UFW)

Docker manages its own iptables rules, which can bypass UFW. For proper security:

1. Allow only necessary ports through UFW:
   ```bash
   sudo ufw allow 80/tcp   # HTTP for Let's Encrypt challenges
   sudo ufw allow 443/tcp  # HTTPS for web interface
   ```

2. Block Docker from setting iptables rules that bypass UFW by editing `/etc/docker/daemon.json`:
   ```json
   {
     "iptables": false
   }
   ```

3. Restart Docker:
   ```bash
   sudo systemctl restart docker
   ```

With this setup, all Docker container ports will be properly controlled by UFW.

## Troubleshooting

### Connection Issues

If devices aren't connecting:
1. Check network connectivity: `ping device_ip`
2. Verify credentials in the config file
3. Make sure the device model is supported
4. Check Oxidized logs: `docker-compose logs oxidized`

### Web Interface Issues

If the web interface isn't working:
1. Check if Oxidized is running: `docker-compose ps`
2. Check if the web interface is accessible locally: `curl http://localhost:8888`
3. Verify Traefik logs: `docker-compose logs traefik`
4. Check DNS configuration: `dig oxid.xxxx.net`

### SSL Certificate Issues

If SSL certificates aren't working:
1. Make sure ports 80 and 443 are accessible from the internet
2. Check Traefik logs for certificate issuance errors
3. Verify the acme.json file: `docker exec -it traefik cat /letsencrypt/acme.json`

## Security Considerations

For production use, consider:
1. Removing the Traefik dashboard or securing it properly
2. Using environment variables for sensitive information
3. Implementing proper authentication for the Oxidized web interface
4. Restricting network access to management interfaces

## Advanced Configuration

### Using Git for Configuration Storage

To store configurations in Git instead of files:

1. Edit the main config file:
   ```yaml
   output:
     default: git
     git:
       user: oxidized
       email: oxidized@example.com
       repo: /home/oxidized/.config/oxidized/configs.git
   ```

2. Initialize the Git repository:
   ```bash
   docker exec -it oxidized bash -c "mkdir -p /home/oxidized/.config/oxidized/configs.git && cd /home/oxidized/.config/oxidized/configs.git && git init --bare"
   ```

### Custom User Credentials

To use different credentials for different devices:

1. Create a users.yml file:
   ```bash
   nano ~/oxidized-traefik/oxidized-config/users.yml
   ```

2. Add content:
   ```yaml
   ---
   username: xxx      # Default username
   password: xxx      # Default password
   
   groups:
     routers:
       username: router_admin
       password: router_pass
     switches:
       username: switch_admin
       password: switch_pass
   
   models:
     junos:
       username: juniper_user
       password: juniper_pass
   ```

3. Update the main config to reference this file:
   ```yaml
   source:
     default: csv
     csv:
       file: "/home/oxidized/.config/oxidized/router.db"
       delimiter: ":"
       map:
         name: 0
         model: 1
         group: 2
   
   model_map:
     juniper: junos
     cisco: ios
   
   rest: "0.0.0.0:8888"
   
   username: admin
   password: admin
   
   vars_map:
     enable: enable
   
   groups:
     routers:
       username: router_admin
       password: router_pass
     switches:
       username: switch_admin
       password: switch_pass
   
   models:
     junos:
       username: juniper_user
       password: juniper_pass
   ```

## Additional Resources

- [Oxidized Documentation](https://github.com/ytti/oxidized/tree/master/docs)
- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
