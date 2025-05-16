# Setup Secure traefik Dashboard Access via IP
## Configuration Files
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
