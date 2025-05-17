# Traefik as an SSL router with HTTPS
Here’s a **complete, self-contained example** with `traefik.yml` and `docker-compose.yml` to set up Traefik as an SSL router with HTTPS. Replace placeholders (e.g., `example.com`, `your-email@example.com`, `8080`) with your values.

---

## File Structure
```
your-project/
├── docker-compose.yml
├── traefik.yml
└── letsencrypt/         # Directory for SSL certificates
    └── acme.json        # Traefik will create this (ensure permissions: chmod 600)
```

---

## 1. `traefik.yml` (Static Configuration)
```yaml
# Static configuration (entrypoints, certificates, providers)
entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: ":443"

certificatesResolvers:
  letsencrypt:
    acme:
      email: your-email@example.com   # Replace with your email
      storage: /letsencrypt/acme.json
      httpChallenge:
        entryPoint: web

providers:
  docker:
    exposedByDefault: false   # Only containers with `traefik.enable=true` are exposed
    network: traefik-public   # Shared network
```

---

## 2. `docker-compose.yml`
```yaml
version: "3.8"

services:
  # Traefik Reverse Proxy
  traefik:
    image: traefik:v2.10
    container_name: traefik
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro  # Docker discovery
      - ./traefik.yml:/etc/traefik/traefik.yml        # Static config
      - ./letsencrypt:/letsencrypt                    # SSL certificates
    networks:
      - traefik-public

  # Your Application
  your-app:
    image: your-app-image        # Replace with your app's image
    restart: unless-stopped
    networks:
      - traefik-public
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.https-router.rule=Host(`example.com`)"  # Your domain
      - "traefik.http.routers.https-router.entrypoints=websecure"
      - "traefik.http.routers.https-router.tls=true"
      - "traefik.http.routers.https-router.tls.certresolver=letsencrypt"
      - "traefik.http.services.your-app.loadbalancer.server.port=8080"  # Your app's port

networks:
  traefik-public:
    external: true   # Create it first: `docker network create traefik-public`
```

---

## 3. Setup Steps

1. **Create the network**:
   ```bash
   docker network create traefik-public
   ```

2. **Create `letsencrypt` directory and set permissions**:
   ```bash
   mkdir -p letsencrypt && touch letsencrypt/acme.json && chmod 600 letsencrypt/acme.json
   ```

3. **Start Traefik and your app**:
   ```bash
   docker-compose up -d
   ```

---

## How It Works
1. **Traefik Configuration**:
   - Listens on ports `80` (HTTP) and `443` (HTTPS).
   - Automatically redirects HTTP → HTTPS using the `web` entrypoint redirection.
   - Uses Let’s Encrypt to issue/renew SSL certificates via the `letsencrypt` resolver.
   - Discovers containers on the `traefik-public` network.

2. **Your Application**:
   - Exposed via Traefik with TLS using the domain `example.com`.
   - Traffic is routed to the container’s internal port `8080`.

---

## Notes
- Replace `example.com` with your domain and ensure DNS points to your server’s IP.
- Replace `your-email@example.com` with your email (for Let’s Encrypt expiry notices).
- Replace `8080` with your app’s internal port (e.g., `3000` for Node.js, `80` for Nginx).

# Setup Secure traefik Dashboard Access via IP
## Configuration Files
> [!IMPORTANT]
> Replace `[IP]` and `[password_hash]` with applicable values.
### **1. `docker-compose.yml`**
```yaml
services:
  traefik:
    image: "traefik:v3.4"
    container_name: "web-tools-traefik"
    command:
      - "--configFile=/etc/traefik/traefik.yml"  # Load Traefik config from file
    ports:
      - "80:80"    # HTTP entrypoint (e.g., redirects)
      - "443:443"  # HTTPS entrypoint (dashboard/API)
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"  # Docker provider
      - "./certs:/certs"                                # Self-signed certs
      - "./traefik.yml:/etc/traefik/traefik.yml"         # Main config
    labels:
      # Route HTTPS traffic to Traefik's dashboard
      - "traefik.enable=true"
      - "traefik.http.routers.traefik_dashboard.rule=Host(`[IP]`) && (PathPrefix(`/dashboard/`) || PathPrefix(`/api`))"
      - "traefik.http.routers.traefik_dashboard.entrypoints=websecure"
      - "traefik.http.routers.traefik_dashboard.service=api@internal"  # Internal dashboard service
      # Basic authentication (replace [password_hash] with your own)
      - "traefik.http.routers.traefik_dashboard.middlewares=basic-auth-global"
      - "traefik.http.middlewares.basic-auth-global.basicauth.users=test:[password_hash]"
    networks:
      - traefiknet

networks:
  traefiknet:
    name: web-tools-traefik-net  # Dedicated network for Traefik
```

### **2. `traefik.yml`**
```yaml
# Entrypoints for HTTP/HTTPS
entryPoints:
  web:
    address: ":80"        # HTTP (port 80)
  websecure:
    address: ":443"       # HTTPS (port 443)
    http:
      tls: {}             # Enable TLS for HTTPS

# TLS configuration (self-signed certificate)
tls:
  stores:
    default:
      defaultCertificate:
        certFile: /certs/selfsigned.crt  # Certificate file
        keyFile: /certs/selfsigned.key   # Private key

# Docker provider settings
providers:
  docker:
    exposedByDefault: false  # Only expose containers with traefik.enable=true

# API and dashboard configuration
api:
  insecure: false   # Disable insecure HTTP API access
  dashboard: true   # Enable built-in dashboard
```

---

## **Key Generation & Setup**
1. **Generate Self-Signed Certificate** (replace `[IP]` with your server IP):
   ```bash
   mkdir -p certs
   openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 -nodes \
     -keyout certs/selfsigned.key -out certs/selfsigned.crt \
     -subj "/CN=[IP]" \
     -addext "subjectAltName=IP:[IP]"
   ```

2. **Create Password Hash** (replace `[password]`):
   ```bash
   htpasswd -nb test [password] | sed -e 's/\$/\$\$/g'
   ```
   Update the `basicauth.users` label in `docker-compose.yml` with the output.

### Explanation of `sed -e 's/\$/\$\$/g`
This command ensures `$` characters in the password hash are **escaped** for Docker Compose:
- **Why?** Docker Compose uses `$` for environment variables. To preserve `$` in the hash (e.g., `$apr1$...`), they must be doubled (`$$`).
- **What it does**: Replaces every `$` with `$$` in the `htpasswd` output.

### Example:
```bash
# Original hash from htpasswd:
test:$apr1$665445$kRMEc.dKeK7zcMK3h/7MS/

# After sed:
test:$$apr1$$665445$$kRMEc.dKeK7zcMK3h/7MS/
```

This ensures Docker Compose treats `$$` as a literal `$` in the label.

---

## **Access Instructions**
1. **Dashboard URL**:
   Navigate to `https://[IP]/dashboard/` (ensure trailing `/`).

2. **Browser Warning**:
   Bypass the "unsafe connection" warning (self-signed cert).

3. **Credentials**:
   Use `test` as the username and `[password]` (replace with your chosen password).

---

## **Notes**
- **IP Access**: Self-signed certs require the IP in `subjectAltName` (see key generation step).
- **Security**: This setup is for testing. For production, use a domain and Let’s Encrypt.
