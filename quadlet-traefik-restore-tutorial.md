# Restoring Rootless Podman / Quadlet / Traefik from Git and Backups

## Table of Contents

1. [Introduction](#introduction)
2. [Prerequisites](#prerequisites)
3. [Setup Steps](#setup-steps)
   - [1. Clone the Git Repository](#1-clone-the-git-repository)
   - [2. Create the Symlinks](#2-create-the-symlinks)
   - [3. Configure DuckDNS](#3-configure-duckdns)
   - [4. Recreate Podman Secrets](#4-recreate-podman-secrets)
   - [5. Restore the ACME Volume](#5-restore-the-acme-volume)
   - [6. Create the Proxy Network](#6-create-the-proxy-network)
   - [7. Deploy Traefik](#7-deploy-traefik)
4. [Verification and Testing](#verification-and-testing)
5. [Update the Repository](#update-the-repository)

---

## Introduction

This tutorial walks through restoring the Quadlet / Traefik setup on a **new or rebuilt VM** using the configuration that was previously backed up to GitHub. It is a direct continuation of the [Rootless Podman Container Management with Quadlet and Traefik](quadlet-traefik-tutorial.md) tutorial.

All configuration files already exist in the remote git repository. The steps below focus on cloning that repository, recreating the runtime state that git does not track (secrets, Podman volumes, symlinks), and getting Traefik back up and running.

---

## Prerequisites

The same system prerequisites apply as in the original tutorial:

- ✅ Non-root user created and configured
- ✅ Podman installed (version 5.0+)
- ✅ Git installed and configured (username and email set)
- ✅ SSH key configured for GitHub access
- ✅ Linger enabled for the rootless user
- ✅ `podman-auto-update.timer` and `podman.socket` enabled
- ✅ Firewall services enabled: `http`, `https`, and `http3` (UDP 443)
- ✅ `net.ipv4.ip_unprivileged_port_start=80` configured
- ✅ SELinux enabled
- ✅ DuckDNS account with the `smithoo4` subdomain

**⚠️ Important**: Before proceeding, ensure the **original VM is shut down or destroyed**. Running both the old and new VM at the same time will cause DuckDNS to resolve to the wrong IP and Let's Encrypt certificate challenges will fail.

---

## Setup Steps

### 1. Clone the Git Repository

Clone the configuration repository into the home directory:

```bash
git clone git@github.com:Smithoo4/quadlet-traefik-tutorial.git ~/containers-config
```

Verify the clone was successful:

```bash
ls ~/containers-config
```

You should see the `quadlet/`, `apps/`, `README.md`, and tutorial markdown files.

---

### 2. Create the Symlinks

The git repository stores the configuration files, but Quadlet and the application containers expect them at standard XDG paths. Create the symlinks to wire everything together.

First, ensure the parent directory exists:

```bash
mkdir -p ~/.config/containers
```

Then create the symlinks:

```bash
ln -s ~/containers-config/quadlet ~/.config/containers/systemd
ln -s ~/containers-config/apps ~/.config/containers/apps
```

Verify the symlinks were created correctly:

```bash
ls -la ~/.config/containers
```

You should see `systemd` and `apps` as symbolic links pointing to `~/containers-config/quadlet` and `~/containers-config/apps` respectively.

---

### 3. Configure DuckDNS

If the new VM has a **different public IP address** than the original, you must update DuckDNS before deploying Traefik. Let's Encrypt DNS-01 challenges will fail if DuckDNS resolves to the wrong IP.

Log in to [DuckDNS](https://www.duckdns.org/) and update the `smithoo4` subdomain to point to the new IP address.

---

### 4. Recreate Podman Secrets

Podman secrets are stored in the container runtime, not in git. They must be recreated manually on every new system.

#### DuckDNS Token

```bash
echo -n "your-duckdns-token-here" | podman secret create duckdns_token -
```

**Important**: Replace `your-duckdns-token-here` with your actual DuckDNS token. This is the same token used during the original setup.

#### Verify the Secret

```bash
podman secret ls
```

You should see `duckdns_token` in the list.

---

### 5. Restore the ACME Volume

The `traefik-acme` Podman volume holds the `acme.json` file where Traefik stores its Let's Encrypt certificates. This file is not tracked in git. You have two options:

---

#### Option A — Create a Fresh acme.json (Recommended for most cases)

Use this option if you do not have an `acme.json` backup, or if you are happy to let Traefik request new certificates.

```bash
podman run --rm \
  -v traefik-acme:/acme:Z \
  docker.io/library/alpine:latest \
  sh -c "touch /acme/acme.json && chmod 600 /acme/acme.json"
```

This creates an empty `acme.json` with the correct `600` permissions that Traefik requires. New certificates will be automatically requested from Let's Encrypt when Traefik starts.

**⚠️ Rate Limit Warning**: Let's Encrypt production allows 50 certificates per registered domain per week. If you have been recreating VMs frequently during testing, use Option B.

---

#### Option B — Restore acme.json from Backup

Use this option if you have a previously saved `acme.json` backup and want to avoid requesting new certificates (e.g., to stay within Let's Encrypt rate limits).

**1. Upload the backup from your local machine to the new VM:**

From your local machine (host OS):

```bash
scp ./acme.json.backup username@vm-hostname:~/acme.json.backup
```

Replace `username` with your VM username and `vm-hostname` with your VM's hostname or IP address.

**2. Create the volume:**

```bash
podman volume create traefik-acme
```

**3. Restore the backup into the volume:**

```bash
podman unshare cp ~/acme.json.backup \
  ~/.local/share/containers/storage/volumes/traefik-acme/_data/acme.json
```

**4. Set the correct permissions:**

Traefik requires `acme.json` to be readable only by its owner. Set this from within the user namespace:

```bash
podman unshare chmod 600 \
  ~/.local/share/containers/storage/volumes/traefik-acme/_data/acme.json
```

**5. Clean up the temporary upload:**

```bash
rm ~/acme.json.backup
```

---

Verify the volume exists regardless of which option you chose:

```bash
podman volume ls
```

You should see `traefik-acme` in the list.

---

### 6. Create the Proxy Network

Start the proxy network service so it is available before Traefik starts:

```bash
systemctl --user daemon-reload
systemctl --user start proxy-network.service
```

Verify the network was created:

```bash
podman network ls
```

You should see `proxy` in the list.

---

### 7. Deploy Traefik

Start the Traefik service:

```bash
systemctl --user start traefik.service
```

Check that the service started without errors:

```bash
systemctl --user status traefik.service
```

The service status should show `active (running)`.

Verify the container is running:

```bash
podman ps
```

You should see the `traefik` container in the list.

---

## Verification and Testing

### Check Traefik Logs

Monitor the logs to confirm Traefik is running and certificate activity is as expected:

```bash
journalctl --user -u traefik.service -f
```

**If you used Option A (fresh acme.json)**: Watch for log lines indicating that Traefik is requesting certificates from Let's Encrypt. Certificate acquisition typically takes 10–30 seconds.

**If you used Option B (restored backup)**: Watch for log lines indicating that Traefik loaded existing certificates from `acme.json`. You should **not** see new certificate requests unless the restored certificates are expired or missing domains.

### Verify the Dashboard

Visit `https://traefik.smithoo4.duckdns.org` in your browser:

**Expected Results**:
- HTTP Basic Authentication prompt
- Traefik dashboard loads successfully
- No certificate warning (production certificate trusted by browser)

In your browser, check the certificate details:
1. Click the padlock icon in the address bar
2. View certificate details
3. Confirm:
   - Issued by: Let's Encrypt (not "Fake LE")
   - Valid for ~90 days
   - Subject Alternative Name: `traefik.smithoo4.duckdns.org`
   - Public Key: ECDSA P-384

### Verify Web Applications

If you have other services configured (e.g., `webapp`), start them and verify they are accessible:

```bash
systemctl --user start webapp.service
```

Check that the service started without errors:

```bash
systemctl --user status webapp.service
```

Then visit the service URL to confirm HTTPS is working correctly:

```
https://webapp.smithoo4.duckdns.org
```

---

## Update the Repository

Now that the restore is confirmed working, add this tutorial to the repository, update the README, and push to GitHub.

### 1. Add This Tutorial

Copy this tutorial into the repository directory. From your local machine:

```bash
scp quadlet-traefik-restore-tutorial.md username@vm-hostname:~/containers-config/
```

### 2. Update README.md

Edit `~/containers-config/README.md` to reference the new tutorial and reflect the current state of the repository:

```markdown
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
```

### 3. Commit and Push

```bash
cd ~/containers-config
git add .
git commit -m "Add restore tutorial and update README"
git push
```

Verify the push was successful by visiting your repository on GitHub and confirming the new files appear.

---

**Restore complete!** You now have:
- ✅ Configuration cloned from git and symlinked to the correct XDG paths
- ✅ DuckDNS updated to the new VM's IP address
- ✅ Podman secrets recreated
- ✅ ACME volume restored or freshly initialized with correct permissions
- ✅ Traefik running with valid HTTPS certificates
- ✅ Restore tutorial committed to the repository for future reference

---

**Tutorial Version**: 1.0  
**Last Updated**: February 2026  
**Tested On**: OpenSUSE MicroOS with Podman 5.7.1
