# Quadlet Traefik Configuration

Rootless Podman container management with Quadlet and Traefik reverse proxy.

## Structure

- `quadlet/` - Quadlet unit files (symlinked from `~/.config/containers/systemd`)
- `apps/` - Application configurations (symlinked from `~/.config/containers/apps`)
- `quadlet-traefik-tutorial.md` - Complete initial setup tutorial
- `quadlet-traefik-restore-tutorial.md` - Restore tutorial (new VM / disaster recovery)

## Services

- **Traefik** - Reverse proxy with automatic HTTPS (Let's Encrypt)
- **webapp** - Demo web application

## Features

- HTTP/3 support
- Automatic SSL/TLS certificates via Let's Encrypt DNS-01 challenge
- TLS hardening with modern cipher suites
- Security headers middleware
- Basic authentication for Traefik dashboard
- Rootless containers with SELinux enabled
- Version controlled configuration

## Documentation

See `quadlet-traefik-tutorial.md` for complete initial setup instructions.

See `quadlet-traefik-restore-tutorial.md` for restore and disaster recovery instructions.
