# 11 - Docker Setup

Docker is installed on container 106 (Grafana, 10.2.7.108) running Ubuntu 24.04.

## Installation

Installed via the standard Docker convenience script on Ubuntu 24.04 LXC.

## Known Issues

### AppArmor

Unprivileged LXC containers can't load AppArmor profiles for Docker:

```json
{
  "apparmor": false,
  "storage-driver": "overlay2"
}
```

Fix: Create `/etc/docker/daemon.json` with the above, then `systemctl restart docker`.

### Storage Driver

`overlay2` is the recommended storage driver for modern Linux and works well in LXC containers.

## Verifying Docker

```bash
docker ps                     # List running containers
docker info | grep Storage    # Check storage driver
docker run hello-world        # Verify installation
```

## Running Docker Containers

```bash
# Start compose stack
docker compose -f grafana.yml up -d

# Check logs
docker compose -f grafana.yml logs -f

# Stop
docker compose -f grafana.yml down
```
