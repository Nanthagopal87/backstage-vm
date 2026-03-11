Great choices! Here's the complete setup plan:

## Architecture
```
Browser (443) → Nginx (SSL termination) → Docker Backstage (7007)
```

## Step 1: Generate Self-Signed Certificate

```bash
# Create certs directory
mkdir -p ~/certs

# Generate self-signed cert (valid for 365 days)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout ~/certs/backstage.key \
  -out ~/certs/backstage.crt \
  -subj "/C=US/ST=Local/L=Local/O=Backstage/CN=localhost"
```

## Step 2: Install Nginx

```bash
sudo apt-get install -y nginx
```

## Step 3: Create Nginx Config

```bash
sudo nano /etc/nginx/sites-available/backstage
```

Paste this:

```nginx
server {
    listen 80;
    server_name localhost;
    # Redirect HTTP to HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate     /root/certs/backstage.crt;
    ssl_certificate_key /root/certs/backstage.key;

    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    # Proxy to Backstage Docker container
    location / {
        proxy_pass http://localhost:7007;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

## Step 4: Enable the Nginx Config

```bash
# Enable site
sudo ln -s /etc/nginx/sites-available/backstage /etc/nginx/sites-enabled/

# Remove default site
sudo rm /etc/nginx/sites-enabled/default

# Test config
sudo nginx -t

# Restart Nginx
sudo systemctl restart nginx
```

## Step 5: Update `app-config.production.yaml`

Since Nginx is now handling HTTPS, update the baseUrls:

```yaml
app:
  baseUrl: https://localhost

backend:
  baseUrl: https://localhost
  listen: ':7007'
  cors:
    origin: https://localhost
    methods: [GET, HEAD, PATCH, POST, PUT, DELETE]
    credentials: true
  database:
    client: better-sqlite3
    connection: ':memory:'

auth:
  dangerouslyDisableDefaultAuthPolicy: true
  providers:
    guest:
      dangerouslyAllowOutsideDevelopment: true

permission:
  enabled: false
```

## Step 6: Rebuild and Run Docker

```bash
yarn build:backend

docker rm backstage

docker image build . -f packages/backend/Dockerfile --tag backstage:latest

docker run -d \
  -p 7007:7007 \
  --name backstage \
  backstage:latest
```

## Step 7: Access

```
https://localhost
```
> Browser will show a security warning for self-signed cert — click **"Advanced" → "Proceed to localhost"**

---

## For GCP VM

Same steps, but replace `localhost` with your **GCP VM's external IP** in:
- `openssl` command → `-subj "/CN=YOUR_GCP_IP"`
- Nginx `server_name YOUR_GCP_IP`
- `app-config.production.yaml` → `baseUrl: https://YOUR_GCP_IP`
- Also open port 443 in **GCP Firewall rules**

---

## Summary

| Component | Role |
|-----------|------|
| Nginx | SSL termination on port 443 |
| Self-signed cert | HTTPS without a domain |
| Docker Backstage | Runs on 7007 internally |
| HTTP → HTTPS redirect | Port 80 redirects to 443 |

Start with Step 1 and let me know how it goes!
