# 11 - Docker Setup

Docker is installed on container 106 (Grafana, 10.2.7.108) running Ubuntu 24.04.

## How to Install Docker (Step by Step)

### Prerequisites
- Ubuntu/Debian LXC container
- `nesting=1` feature enabled on the container
- SSH access as root

### Installation

```bash
# 1️⃣ Update system
apt update && apt upgrade -y

# 2️⃣ Install prerequisites
apt install -y ca-certificates curl

# 3️⃣ Install Docker via convenience script
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# 4️⃣ Fix AppArmor (REQUIRED for unprivileged LXC)
# Docker can't load AppArmor profiles inside unprivileged LXC containers
mkdir -p /etc/docker
cat > /etc/docker/daemon.json << 'EOF'
{
  "apparmor": false,
  "storage-driver": "overlay2"
}
EOF

# 5️⃣ Restart Docker
systemctl restart docker

# 6️⃣ Fix permissions (optional, for non-root user)
usermod -aG docker $USER
# Log out and back in for this to take effect, or use `newgrp docker`

# 7️⃣ Verify installation
docker ps                    # Should show empty list (no errors)
docker info | grep Storage   # Should show Storage Driver: overlay2
docker run --rm hello-world  # Should pull and run successfully

# 8️⃣ Install Docker Compose plugin (usually included, but verify)
docker compose version       # Should show version info
```

## Common Docker Commands

```bash
# Container management
docker ps                          # List running containers
docker ps -a                       # List all containers (including stopped)
docker logs <container_name>       # View logs
docker logs -f <container_name>    # Follow log output
docker restart <container_name>    # Restart a single container
docker stop <container_name>       # Stop a container
docker rm <container_name>         # Remove a container

# Image management
docker pull <image:tag>            # Download an image
docker images                      # List downloaded images
docker rmi <image_id>              # Remove an image

# Compose (multi-container)
docker compose up -d               # Start all services in background
docker compose down                # Stop and remove all containers
docker compose restart <service>   # Restart one service
docker compose ps                  # List service status
docker compose logs -f             # Follow all service logs
docker compose logs <service>      # Logs for one service
```

## Running Compose Stacks

The monitoring stack lives at `/opt/monitoring/` on CT 106:

```bash
cd /opt/monitoring

# Deploy everything
docker compose up -d

# Check status
docker compose ps

# View logs
docker compose logs -f

# Restart one service (e.g., after config change)
docker compose restart prometheus

# Full restart
docker compose down && docker compose up -d

# Rebuild and recreate
docker compose up -d --force-recreate <service>
```

## Known Issues

### AppArmor (LXC)

Unprivileged LXC containers can't load AppArmor profiles for Docker. All containers
in the compose stack must have `security_opt: - apparmor:unconfined` to start.

Without this fix, you'll see:
```
Error response from daemon: AppArmor enabled on system but the docker-default
profile could not be loaded: apparmor_parser: Access denied.
```

### Volume Mounts

When bind-mounting files (not directories), make sure the file exists on the host
before the container starts. Docker creates directories but NOT individual files.
Example:
```yaml
# ✅ This works (directory already exists)
volumes:
  - /opt/monitoring/:/etc/app/

# ❌ This fails if /opt/monitoring/config.yml doesn't exist yet
volumes:
  - /opt/monitoring/config.yml:/etc/app/config.yml:ro
```
