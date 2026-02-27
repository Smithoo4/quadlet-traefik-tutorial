# Rootless Podman Container Management with Quadlet and Traefik

## Table of Contents

1. [Introduction](#introduction)
2. [Prerequisites](#prerequisites)
3. [Directory Structure](#directory-structure)
4. [Setup Steps](#setup-steps)
   - [1. Create the Proxy Network](#1-create-the-proxy-network)
   - [2. Create Configuration Directory](#2-create-configuration-directory)
   - [3. Configure DuckDNS](#3-configure-duckdns)
   - [4. Create Podman Secrets](#4-create-podman-secrets)
   - [5. Initialize the ACME Volume](#5-initialize-the-acme-volume)
   - [6. Create Traefik Static Configuration](#6-create-traefik-static-configuration)
   - [7. Create TLS Hardening Configuration](#7-create-tls-hardening-configuration)
   - [8. Create Security Headers Middleware](#8-create-security-headers-middleware)
   - [9. Create Basic Authentication Middleware](#9-create-basic-authentication-middleware)
   - [10. Create Traefik Quadlet Container File](#10-create-traefik-quadlet-container-file)
   - [11. Deploy Traefik](#11-deploy-traefik)
5. [Verification and Testing](#verification-and-testing)
6. [Adding a Proxied Web Service](#adding-a-proxied-web-service)
7. [Moving to Production](#moving-to-production)
8. [Backup Configuration to Remote Git Repository](#backup-configuration-to-remote-git-repository)
9. [Viewing Logs](#viewing-logs)

---

## Introduction

This tutorial demonstrates how to use **Quadlet** to manage rootless Podman containers with **Traefik** as a reverse proxy using ACME DNS-01 challenges with DuckDNS for automatic SSL certificate management.

### Key Principles

- **Rootless containers**: All containers run without root privileges
- **XDG Base Directory compliance**: Organized configuration following Linux standards
- **Declarative configuration**: Infrastructure as code with Quadlet unit files
- **Automatic updates**: Containers stay current with minimal manual intervention
- **Secrets management**: Sensitive data handled securely via Podman secrets
- **Version control**: Track all configuration changes with git for easy backup and rollback

---

## Prerequisites

This tutorial assumes the following are already configured on an OpenSUSE MicroOS system:

- ✅ Non-root user created and configured
- ✅ Podman installed (version 5.0+)
- ✅ Git installed and configured (username and email set)
- ✅ SSH key configured for GitHub access
- ✅ Remote empty git repository created at `git@github.com:Smithoo4/quadlet-traefik-tutorial.git`
- ✅ Linger enabled for the rootless user
- ✅ `podman-auto-update.timer` and `podman.socket` enabled
- ✅ Firewall services enabled: `http`, `https`, and `http3` (UDP 443)
- ✅ `net.ipv4.ip_unprivileged_port_start=80` configured
- ✅ SELinux enabled
- ✅ DuckDNS account with a subdomain (`smithoo4`)

**Note**: The base system installation and configuration can be done using: https://github.com/Smithoo4/MicroOS-fuel-ignition

---

## Directory Structure

This tutorial uses a git-tracked directory with symlinks to the standard XDG locations:

```
~/containers-config/          # Git repository for configuration backup
├── .git/
├── quadlet/                  # Quadlet unit files (symlinked to ~/.config/containers/systemd)
│   ├── proxy.network
│   ├── traefik.container
│   └── webapp.container
└── apps/                     # Application configurations (symlinked to ~/.config/containers/apps)
    ├── traefik/
    │   ├── traefik.yml
    │   └── dynamic/
    │       ├── tls-options.yml
    │       ├── security-headers.yml
    │       └── basic-auth.yml
    └── webapp/
        ├── index.html
        └── nginx.conf

~/.config/containers/
├── systemd/  -> ~/containers-config/quadlet  # Symlink
└── apps/     -> ~/containers-config/apps     # Symlink
```

**Why This Structure?**
- **Git tracking**: All configuration files are version controlled
- **XDG compliance**: Symlinks maintain standard paths for Quadlet and applications
- **Easy backup**: The entire configuration is in one git repository
- **Podman volumes**: Runtime data (like `acme.json`) stored separately in named volumes

---

## Initialize Git Repository

Before creating any configuration files, set up the git-tracked directory structure.

### Create Directory Structure and Symlinks

```bash
mkdir -p ~/.config/containers
mkdir -p ~/containers-config/quadlet
mkdir -p ~/containers-config/apps
ln -s ~/containers-config/quadlet ~/.config/containers/systemd
ln -s ~/containers-config/apps ~/.config/containers/apps
```

Verify the symlinks were created correctly:

```bash
ls -la ~/.config/containers
```

You should see `systemd` and `apps` as symbolic links pointing to `~/containers-config/quadlet` and `~/containers-config/apps` respectively.

### Initialize Git Repository

```bash
cd ~/containers-config
git init
```

### Create .gitignore

Create `~/containers-config/.gitignore` to exclude sensitive or temporary files:

```
# Exclude any files that might contain secrets in the future
*.key
*.pem
```

### Initial Commit

```bash
git add .gitignore
git commit -m "Initial commit: Setup container configuration repository"
```

---

## Setup Steps

### 1. Create the Proxy Network

Create `~/.config/containers/systemd/proxy.network`:

```ini
[Network]
NetworkName=proxy
Driver=bridge
```

This defines a bridge network that Traefik and backend services will use to communicate.

---

### 2. Create Configuration Directory

```bash
mkdir -p ~/.config/containers/apps/traefik/dynamic
```

The `dynamic` directory will be used for future middleware, routers, and TLS options.

---

### 3. Configure DuckDNS

Log in to [DuckDNS](https://www.duckdns.org/) and set the `smithoo4` subdomain to point to the IP address of the VM.

---

### 4. Create Podman Secrets

Store sensitive data as Podman secrets:

#### DuckDNS Token

```bash
echo -n "your-duckdns-token-here" | podman secret create duckdns_token -
```

**Important**: Replace `your-duckdns-token-here` with your actual DuckDNS token.

#### Verify Secret

```bash
podman secret ls
```

You should see `duckdns_token` in the list.

---

### 5. Initialize the ACME Volume

Create the volume and initialize `acme.json` with correct permissions:

```bash
podman run --rm \
  -v traefik-acme:/acme:Z \
  docker.io/library/alpine:latest \
  sh -c "rm -f /acme/acme.json && touch /acme/acme.json && chmod 600 /acme/acme.json"
```

This command removes any existing `acme.json` file, creates a fresh empty one, and sets mode `600` (read/write for owner only), which Traefik requires.

**Note**: This command is also useful if you need to clear your ACME certificates (e.g., when switching from staging to production).

Verify the volume was created:

```bash
podman volume ls
```

You should see `traefik-acme` in the list.

---

### 6. Create Traefik Static Configuration

Create `~/.config/containers/apps/traefik/traefik.yml`:

```yaml
# Global Configuration
global:
  checkNewVersion: true
  sendAnonymousUsage: false

# API and Dashboard
api:
  dashboard: true
  insecure: false  # Dashboard is only accessible via Traefik routing

# Entry Points
entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
          permanent: true

  websecure:
    address: ":443"
    http3: {}
    http:
      middlewares:
        - security-headers@file
      tls:
        certResolver: letsencrypt

# Certificate Resolvers
certificatesResolvers:
  letsencrypt:
    acme:
      storage: /acme/acme.json
      caServer: https://acme-staging-v02.api.letsencrypt.org/directory  # Staging for testing
      keyType: EC384
      dnsChallenge:
        provider: duckdns
        delayBeforeCheck: 0
        resolvers:
          - "1.1.1.1:53"
          - "8.8.8.8:53"

# Providers
providers:
  docker:
    endpoint: "unix:///var/run/podman/podman.sock"
    exposedByDefault: false
    network: proxy

  file:
    directory: /etc/traefik/dynamic
    watch: true

# Logging
log:
  level: DEBUG  # Set to WARN for production
  format: common

# Access Logs
accessLog:
  format: common
```

**Key Configuration Notes**:
- **caServer**: Using Let's Encrypt staging for testing (switch to production later)
- **keyType: EC384**: Uses ECDSA with P-384 curve for stronger, more efficient certificates
- **http3: {}**: Enables HTTP/3 (QUIC) support on the websecure entrypoint for improved performance
- **middlewares - security-headers@file**: Applies global security headers to all HTTPS traffic
- **log.level**: Set to `DEBUG` for initial setup; change to `WARN` for production

---

### 7. Create TLS Hardening Configuration

Create TLS configuration to enforce strong encryption standards.

Create `~/.config/containers/apps/traefik/dynamic/tls-options.yml`:

```yaml
tls:
  options:
    default:
      minVersion: VersionTLS13
      sniStrict: true
```

**Security Impact**:
- **minVersion: VersionTLS13**: Enforces TLS 1.3 only, removing legacy protocol vulnerabilities
- **sniStrict: true**: Prevents fallback to default certificates and stops certificate enumeration attacks
- **Cipher/Curve Selection**: Relies on Go's secure defaults (no custom configuration needed)

**Note**: Naming the TLS options `default` means Traefik will automatically apply these settings to all HTTPS connections without requiring explicit configuration on each router. This ensures consistent security across all services.

---

### 8. Create Security Headers Middleware

Create global security headers to protect all routes.

Create `~/.config/containers/apps/traefik/dynamic/security-headers.yml`:

```yaml
http:
  middlewares:
    security-headers:
      headers:
        frameDeny: true
        contentTypeNosniff: true
        referrerPolicy: "strict-origin-when-cross-origin"
        permissionsPolicy: "camera=(), microphone=(), geolocation=()"
        stsSeconds: 63072000
        stsIncludeSubdomains: true
        stsPreload: true
        forceSTSHeader: true
```

**Security Impact**:
- **frameDeny**: Prevents clickjacking attacks by blocking iframe embedding
- **contentTypeNosniff**: Prevents MIME type sniffing vulnerabilities
- **referrerPolicy**: Protects privacy by limiting referrer information leakage
- **permissionsPolicy**: Locks down browser features (camera, microphone, geolocation)
- **HSTS (HTTP Strict Transport Security)**:
  - `stsSeconds: 63072000` - 2 years (industry standard)
  - `stsIncludeSubdomains: true` - Protects all subdomains
  - `stsPreload: true` - Ready for HSTS preload list
  - `forceSTSHeader: true` - Always enforces HTTPS

**Note**: These headers are applied globally to all HTTPS traffic via the `websecure` entrypoint configuration in `traefik.yml`.

---

### 9. Create Basic Authentication Middleware

Create HTTP Basic Authentication middleware as a dynamic configuration file. This will be used to protect services like the Traefik dashboard.

#### Generate htpasswd Hash

Use an htpasswd generator like [https://codeshack.io/htpasswd-generator/](https://codeshack.io/htpasswd-generator/) to create your password hash.

**Recommended Algorithm**: **Bcrypt** - This is the current industry standard for htpasswd hashing. It's more secure than MD5, SHA-1, or crypt algorithms.

Example:
- Username: `admin`
- Password: `your-secure-password`
- Algorithm: **Bcrypt**

The generator will produce a hash like:
```
admin:$2y$10$abcdefghijklmnopqrstuvwxyz1234567890ABCDEFGHIJKLMNOP
```

#### Create the Middleware Configuration

Create `~/.config/containers/apps/traefik/dynamic/basic-auth.yml`:

```yaml
http:
  middlewares:
    basic-auth:
      basicAuth:
        users:
          - "admin:$2y$10$abcdefghijklmnopqrstuvwxyz1234567890ABCDEFGHIJKLMNOP"
```

**Security Note**: For this tutorial, the password hash is stored directly in the configuration file. In production environments, consider using Traefik's [file provider with external files](https://doc.traefik.io/traefik/middlewares/http/basicauth/#usersfile) or integrate with an authentication service.

**Applying the Middleware**: To use this middleware on any service, add the following label to your container:
```ini
Label=traefik.http.routers.your-router-name.middlewares=basic-auth@file
```

This middleware will be applied to the Traefik dashboard in the next section.

---

### 10. Create Traefik Quadlet Container File

Create `~/.config/containers/systemd/traefik.container`:

```ini
[Unit]
Description=Traefik Reverse Proxy
Wants=network-online.target
After=network-online.target
RequiresMountsFor=%h/.config/containers/apps/traefik

[Container]
Image=docker.io/traefik:v3
AutoUpdate=registry

# Container Name
ContainerName=traefik

# Network
Network=proxy

# Published Ports
PublishPort=80:80
PublishPort=443:443
PublishPort=443:443/udp

# Volume Mounts
Volume=%h/.config/containers/apps/traefik/traefik.yml:/etc/traefik/traefik.yml:ro,Z
Volume=%h/.config/containers/apps/traefik/dynamic:/etc/traefik/dynamic:ro,Z
Volume=traefik-acme:/acme:Z
Volume=%t/podman/podman.sock:/var/run/podman/podman.sock:ro

# Secrets
Secret=duckdns_token,type=env,target=DUCKDNS_TOKEN

# SELinux
SecurityLabelDisable=true

# Labels for Traefik Dashboard
Label=traefik.enable=true
Label=traefik.http.routers.dashboard.rule=Host(`traefik.smithoo4.duckdns.org`)
Label=traefik.http.routers.dashboard.entrypoints=websecure
Label=traefik.http.routers.dashboard.tls.certresolver=letsencrypt
Label=traefik.http.routers.dashboard.service=api@internal
Label=traefik.http.routers.dashboard.middlewares=basic-auth@file

[Service]
Restart=always
TimeoutStartSec=900

[Install]
WantedBy=default.target
```

**Key Configuration Explanations**:

- **AutoUpdate=registry**: Enables automatic updates when new images are available
- **PublishPort=443:443/udp**: Publishes UDP port 443 for HTTP/3 (QUIC) support
- **SecurityLabelDisable=true**: Disables SELinux labeling for the Traefik container. This is required because Traefik needs access to the Podman socket (`/var/run/podman/podman.sock`) to discover containers, but SELinux blocks this access by default. An alternative approach would be to bind-mount the socket with the `:z` option, but that would change the socket's SELinux context and could affect other programs using the socket. See [Red Hat Bug 1495053](https://bugzilla.redhat.com/show_bug.cgi?id=1495053#c2) for more details.
- **Volume mounts**: `:Z` flag enables SELinux relabeling; `:ro` for read-only
- **Labels**: Configure dashboard access via Traefik itself (port 8080 not published to host)
  - The `middlewares=basic-auth@file` label applies HTTP Basic Authentication to the dashboard using the middleware defined in `dynamic/basic-auth.yml`

---

### Commit Configuration to Git

Before deploying, commit all the configuration files you've created:

```bash
cd ~/containers-config
git add .
git commit -m "Add Traefik configuration with TLS hardening, security headers, and basic auth"
```

This creates a snapshot of your configuration before deployment.

---

### 11. Deploy Traefik

#### Reload Systemd

Tell systemd to discover and convert the Quadlet files:

```bash
systemctl --user daemon-reload
```

#### Start the Proxy Network

```bash
systemctl --user start proxy-network.service
```

Verify the network was created:

```bash
podman network ls
```

You should see a network named `proxy`.

#### Start Traefik

```bash
systemctl --user start traefik.service
```

Verify the container is running:

```bash
podman ps
```

You should see the `traefik` container running with ports 80 and 443 published.

---

## Verification and Testing

### 1. View Traefik Logs

Monitor Traefik startup and ACME certificate acquisition:

```bash
journalctl --user -u traefik.service -n 50
```

**On first run (certificate acquisition)**, look for these key messages in sequence:

1. `Building ACME client... providerName=letsencrypt.acme`
2. `Register... providerName=letsencrypt.acme`
3. `Using DNS Challenge provider: duckdns`
4. `[traefik.smithoo4.duckdns.org] acme: Obtaining bundled SAN certificate`
5. `[traefik.smithoo4.duckdns.org] acme: use dns-01 solver`
6. `[traefik.smithoo4.duckdns.org] acme: Trying to solve DNS-01`
7. `Wait for propagation [timeout: 1m0s, interval: 2s]`
8. `[traefik.smithoo4.duckdns.org] The server validated our request`
9. `[traefik.smithoo4.duckdns.org] acme: Validations succeeded; requesting certificates`
10. `[traefik.smithoo4.duckdns.org] Server responded with a certificate.`
11. `Certificates obtained for domains [traefik.smithoo4.duckdns.org]`
12. `Adding certificate for domain(s) traefik.smithoo4.duckdns.org`

**On subsequent restarts** (certificate already exists):

- `No ACME certificate generation required for domains` - This means the certificate is already valid and being reused

### 2. Verify ACME Certificate Acquisition

After a few moments, check if the certificate was obtained by inspecting the `acme.json` file:

```bash
podman unshare cat ~/.local/share/containers/storage/volumes/traefik-acme/_data/acme.json
```

You should see JSON data with certificate information for `traefik.smithoo4.duckdns.org`.

**Note**: Use `podman unshare` to access files in Podman volumes with correct permissions.

### 3. Access the Traefik Dashboard

Open a web browser and navigate to:

```
https://traefik.smithoo4.duckdns.org
```

**Expected Results**:
- Browser shows a certificate warning (normal for Let's Encrypt staging certificates)
- Accept the certificate and proceed
- **HTTP Basic Authentication prompt will appear** - Enter the username and password you configured in `basic-auth.yml`
- After successful authentication, you should see the Traefik dashboard with the `dashboard@internal` router

### 4. Verify Automatic HTTP to HTTPS Redirect

Visit the HTTP endpoint:

```
http://traefik.smithoo4.duckdns.org
```

You should be automatically redirected to HTTPS.

### 5. Check Service Status

```bash
systemctl --user status traefik.service
```

Service should be `active (running)`.

---

## Adding a Proxied Web Service

Now that Traefik is running and verified, let's add a web service to demonstrate how Traefik proxies backend containers. We'll deploy a simple web application using Nginx.

### 1. Create Application Configuration Directory

Following the XDG Base Directory pattern, create a directory for the web application's configuration:

```bash
mkdir -p ~/.config/containers/apps/webapp
```

This keeps application configurations organized and separate from Traefik's configuration.

### 2. Create Web Content

Create a simple HTML page to serve:

Create `~/.config/containers/apps/webapp/index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Demo Web App</title>
    <style>
        body {
            font-family: system-ui, -apple-system, sans-serif;
            max-width: 800px;
            margin: 50px auto;
            padding: 20px;
            background: #f5f5f5;
        }
        .container {
            background: white;
            padding: 40px;
            border-radius: 8px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        }
        h1 {
            color: #333;
            margin-top: 0;
        }
        .info {
            background: #e3f2fd;
            padding: 15px;
            border-radius: 4px;
            margin: 20px 0;
        }
        .success {
            color: #2e7d32;
            font-weight: bold;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>🎉 Success!</h1>
        <p class="success">Your web application is being proxied through Traefik.</p>
        <div class="info">
            <h3>What's Happening:</h3>
            <ul>
                <li>This page is served by an Nginx container</li>
                <li>Traefik is reverse proxying the request</li>
                <li>HTTPS is automatically handled by Traefik</li>
                <li>Certificate is issued by Let's Encrypt</li>
                <li>Security headers are applied globally</li>
            </ul>
        </div>
        <p><strong>Subdomain:</strong> webapp.smithoo4.duckdns.org</p>
        <p><strong>Served by:</strong> Nginx (rootless container)</p>
        <p><strong>Proxied by:</strong> Traefik v3</p>
    </div>
</body>
</html>
```

### 3. Create Nginx Configuration

Create `~/.config/containers/apps/webapp/nginx.conf`:

```nginx
server {
    listen 80;
    server_name _;
    
    root /usr/share/nginx/html;
    index index.html;
    
    location / {
        try_files $uri $uri/ =404;
    }
    
    # Security headers (Traefik adds global ones, but these are nginx-specific)
    add_header X-Served-By "Nginx-Container" always;
    
    # Logging
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;
}
```

### 4. Create the Web Application Quadlet Container File

Create `~/.config/containers/systemd/webapp.container`:

```ini
[Unit]
Description=Demo Web Application
Wants=network-online.target
After=network-online.target traefik.service
RequiresMountsFor=%h/.config/containers/apps/webapp

[Container]
Image=docker.io/library/nginx:alpine
AutoUpdate=registry

# Container Name
ContainerName=webapp

# Network
Network=proxy

# Volume Mounts
Volume=%h/.config/containers/apps/webapp/index.html:/usr/share/nginx/html/index.html:ro,Z
Volume=%h/.config/containers/apps/webapp/nginx.conf:/etc/nginx/conf.d/default.conf:ro,Z

# Traefik Labels
Label=traefik.enable=true
Label=traefik.http.routers.webapp.rule=Host(`webapp.smithoo4.duckdns.org`)
Label=traefik.http.routers.webapp.entrypoints=websecure
Label=traefik.http.routers.webapp.tls.certresolver=letsencrypt
Label=traefik.http.services.webapp.loadbalancer.server.port=80

[Service]
Restart=always
TimeoutStartSec=300

[Install]
WantedBy=default.target
```

**Key Configuration Explanations**:

- **After=traefik.service**: Ensures Traefik starts before this container
- **Network=proxy**: Connects to the same network as Traefik for communication
- **Volume mounts**: Bind mounts application files from `~/.config/containers/apps/webapp/`
- **Traefik Labels**:
  - `traefik.enable=true`: Explicitly enable Traefik for this container
  - `rule=Host(...)`: Routes requests for `webapp.smithoo4.duckdns.org` to this container
  - `entrypoints=websecure`: Only accessible via HTTPS (port 443)
  - `tls.certresolver=letsencrypt`: Automatically get a Let's Encrypt certificate
  - `loadbalancer.server.port=80`: Tell Traefik the container listens on port 80

**Security Notes**:
- The container does NOT publish any ports to the host (no `PublishPort=`)
- Only accessible through Traefik
- Automatically gets HTTPS with Let's Encrypt
- Global security headers are applied automatically
- Rootless container (no privilege escalation)

### Commit Web Application Configuration

Before deploying, commit the web application configuration to git:

```bash
cd ~/containers-config
git add .
git commit -m "Add webapp configuration and Quadlet file"
```

### 6. Deploy the Web Application

#### Reload Systemd

```bash
systemctl --user daemon-reload
```

#### Start the Web Application

```bash
systemctl --user start webapp.service
```

#### Verify Container is Running

```bash
podman ps
```

You should see both `traefik` and `webapp` containers running.

### 7. Verify the Web Application

#### Check Service Status

```bash
systemctl --user status webapp.service
```

Service should be `active (running)`.

#### View Logs

```bash
journalctl --user -u webapp.service -n 50
```

Look for Nginx starting successfully.

#### Monitor Traefik Logs for Certificate Acquisition

```bash
journalctl --user -u traefik.service -n 50
```

Look for:
- `Certificates obtained for domains [webapp.smithoo4.duckdns.org]`
- `Adding certificate for domain(s) webapp.smithoo4.duckdns.org`

### 8. Access the Web Application

Open a web browser and navigate to:

```
https://webapp.smithoo4.duckdns.org
```

**Expected Results**:
- Browser shows a certificate warning (normal for Let's Encrypt staging certificates)
- Accept the certificate and proceed
- You should see the demo web application page
- Check the certificate details - it should be a separate certificate for `webapp.smithoo4.duckdns.org`

**Verify HTTP to HTTPS Redirect**:

Visit `http://webapp.smithoo4.duckdns.org` - you should be automatically redirected to HTTPS.

---

**Success!** You now have:
- ✅ Traefik reverse proxy running
- ✅ A web application being proxied
- ✅ Automatic HTTPS with Let's Encrypt
- ✅ Separate certificates per subdomain
- ✅ Global security headers applied
- ✅ Network isolation (no direct host access)

This pattern can be repeated for any additional services you want to add. Each service gets:
- Its own config directory in `~/.config/containers/apps/`
- Its own Quadlet `.container` file
- Its own subdomain
- Its own certificate
- Automatic HTTPS

---

## Moving to Production

Once you've verified everything works with Let's Encrypt staging certificates, you're ready to switch to production certificates.

**⚠️ Warning**: Let's Encrypt production has strict rate limits (50 certificates per registered domain per week). Only proceed after confirming everything works with staging.

### 1. Stop Traefik

Stop the service before making configuration changes:

```bash
systemctl --user stop traefik.service
```

### 2. Update Traefik Configuration

Edit `~/.config/containers/apps/traefik/traefik.yml` and make the following changes:

**Remove or comment out the staging caServer line:**

```yaml
certificatesResolvers:
  letsencrypt:
    acme:
      storage: /acme/acme.json
      # caServer: https://acme-staging-v02.api.letsencrypt.org/directory  # Commented out - defaults to production
      keyType: EC384
```

By removing or commenting out the `caServer` line, Traefik defaults to the production Let's Encrypt server.

**Change log level from DEBUG to WARN:**

```yaml
log:
  level: WARN  # Production logging
  format: common
```

### 3. Commit Production Configuration

Commit the production configuration changes to git:

```bash
cd ~/containers-config
git add .
git commit -m "Switch to Let's Encrypt production certificates"
```

This documents the switch to production in your configuration history.

### 4. Clear the Staging Certificate Data

Remove the staging certificates from `acme.json`:

```bash
podman run --rm \
  -v traefik-acme:/acme:Z \
  docker.io/library/alpine:latest \
  sh -c "rm -f /acme/acme.json && touch /acme/acme.json && chmod 600 /acme/acme.json"
```

This deletes the old staging certificate data and creates a fresh `acme.json` file.

### 5. Reload and Start Traefik

Apply the configuration changes and start the service:

```bash
systemctl --user daemon-reload
systemctl --user start traefik.service
```

### 6. Verify Production Certificate

Verify the production certificate was obtained by viewing the `acme.json` file:

```bash
podman unshare cat ~/.local/share/containers/storage/volumes/traefik-acme/_data/acme.json
```

You should see JSON data with certificate information for your domains. The certificate acquisition process should take 10-30 seconds.

### 7. Access the Dashboard

Visit `https://traefik.smithoo4.duckdns.org` in your browser:

**Expected Results**:
- **No certificate warning** - Production certificates are trusted by all browsers
- HTTP Basic Authentication prompt
- Traefik dashboard loads successfully

### 8. Verify Certificate Details

In your browser, check the certificate:
1. Click the padlock icon in the address bar
2. View certificate details
3. Confirm:
   - Issued by: Let's Encrypt (not "Fake LE")
   - Valid for ~90 days
   - Subject Alternative Name: `traefik.smithoo4.duckdns.org`
   - Public Key: ECDSA P-384

### 9. Check and Restart Web Applications

**Important**: After switching to production certificates, you should also verify and potentially restart any web applications behind Traefik:

```bash
# Check if your web app is running
podman ps

# Restart the webapp container
systemctl --user restart webapp.service
```

Then verify the application is accessible and using the new production certificate:

```
https://webapp.smithoo4.duckdns.org
```

Some applications may cache the old staging certificate and require a restart to pick up the new production certificate from Traefik.

**You're now running with production-grade security!**

---

## Backup Configuration to Remote Git Repository

Now that your configuration is complete and working, it's time to back it up to a remote git repository for safekeeping and future reference.

### 1. Transfer Tutorial to VM

From your host OS (where you have this tutorial saved), transfer it to the VM using SSH:

```bash
scp quadlet-traefik-tutorial.md username@vm-hostname:~/containers-config/
```

Replace `username` with your VM username and `vm-hostname` with your VM's hostname or IP address.

### 2. Create README.md

Create `~/containers-config/README.md`:

```markdown
# Quadlet Traefik Configuration

Rootless Podman container management with Quadlet and Traefik reverse proxy.

## Structure

- `quadlet/` - Quadlet unit files (symlinked from `~/.config/containers/systemd`)
- `apps/` - Application configurations (symlinked from `~/.config/containers/apps`)
- `quadlet-traefik-tutorial.md` - Complete setup tutorial

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

See `quadlet-traefik-tutorial.md` for complete setup instructions.
```

### 3. Add Remote Origin

Set the remote origin to your GitHub repository:

```bash
cd ~/containers-config
git remote add origin git@github.com:Smithoo4/quadlet-traefik-tutorial.git
```

### 4. Commit and Push

Add all files and push to the remote repository:

```bash
git add .
git commit -m "Complete Traefik configuration with tutorial and README"
git push -u origin
```

---

## Viewing Logs

### Real-time Log Monitoring

```bash
journalctl --user -u traefik.service -f
```

### View Last N Entries

```bash
journalctl --user -u traefik.service -n 50
```

### View Logs Since a Time

```bash
journalctl --user -u traefik.service --since "10 minutes ago"
journalctl --user -u traefik.service --since today
```

### Search Logs

```bash
journalctl --user -u traefik.service | grep -i "acme"
journalctl --user -u traefik.service | grep -i "error"
```

### Alternative: Podman Logs

```bash
podman logs traefik
podman logs -f traefik  # Follow mode
podman logs --tail 50 traefik
```

---

**Tutorial Version**: 2.8  
**Last Updated**: February 2026  
**Tested On**: OpenSUSE MicroOS with Podman 5.7.1
