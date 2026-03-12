# Backstage Implementation Guide

> Complete Step-by-Step Setup with Troubleshooting  
> **Stack:** WSL Ubuntu • Docker • Keycloak • PostgreSQL • Nginx  
> **Version:** 1.0 | March 2026

---

## Architecture Overview

```
Browser (HTTPS 443)
        ↓
   Nginx (SSL termination)
        ↓
Docker Backstage (7007)
        ↓
Docker Keycloak (8080)
        ↓
Docker PostgreSQL (5432)
    ├── DB: backstage
    └── DB: keycloak
```

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Backstage Installation](#2-backstage-installation)
3. [Docker Containerization](#3-docker-containerization)
4. [Nginx Reverse Proxy with HTTPS](#4-nginx-reverse-proxy-with-https)
5. [Docker Network and PostgreSQL Setup](#5-docker-network-and-postgresql-setup)
6. [Keycloak Setup](#6-keycloak-setup)
7. [Keycloak Users/Groups Sync to Backstage](#7-keycloak-usersgroups-sync-to-backstage)
8. [Troubleshooting Guide](#8-troubleshooting-guide)
9. [Quick Reference](#9-quick-reference)

---

## 1. Prerequisites

### 1.1 System Requirements

| Component | Requirement | Notes |
|-----------|-------------|-------|
| OS | Ubuntu (WSL2 or VM) | Windows WSL2 Ubuntu recommended |
| Node.js | v24.13.1 | Install via nvm |
| nvm | v0.40.1+ | Node Version Manager |
| Docker | Latest | Docker Desktop with WSL2 integration |
| RAM | 8GB minimum | 16GB recommended |
| Disk | 20GB free | For Docker images and data |

### 1.2 Install nvm and Node.js

```bash
# Install nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash

# Reload shell
source ~/.bashrc

# Install Node.js v24.13.1
nvm install 24.13.1
nvm alias default 24.13.1

# Verify
node --version   # v24.13.1
nvm --version    # v0.40.1
```

### 1.3 Install Build Tools

> ⚠️ **Critical:** Without build tools, native modules like `cpu-features` and `isolated-vm` will fail to compile. This is the most common first error on a fresh Ubuntu setup.

```bash
sudo apt-get update && sudo apt-get install -y \
  build-essential python3 python3-pip gcc g++ make cmake
```

### 1.4 Enable Docker Desktop WSL2 Integration

1. Open Docker Desktop on Windows
2. Go to **Settings → Resources → WSL Integration**
3. Enable **"Enable integration with my default WSL distro"**
4. Toggle ON your specific distro (e.g., Ubuntu)
5. Click **"Apply & Restart"**

```bash
# Verify Docker works in WSL
docker --version
docker images
```

---

## 2. Backstage Installation

### 2.1 Create Backstage App

```bash
# Create new Backstage app
npx @backstage/create-app@latest

# Follow prompts:
# Enter a name for the app: my-backstage

# Navigate to project
cd my-backstage
```

### 2.2 Run Locally (Development Mode)

```bash
# Install dependencies
yarn install

# Start the app
yarn start

# Access at:
# http://localhost:3000
```

> 💡 **WSL Browser Tip:** If you cannot access `localhost:3000` in your Windows browser after `yarn start`, try clearing the browser cache (`Ctrl + Shift + R`).

---

## 3. Docker Containerization

### 3.1 Build Steps

Backstage requires a specific build order before Docker image creation:

```bash
# Step 1: Install dependencies
yarn install --immutable

# Step 2: Generate TypeScript type definitions
yarn tsc

# Step 3: Build backend bundle
yarn build:backend
```

### 3.2 app-config.yaml

Update the following key settings:

```yaml
app:
  title: Scaffolded Backstage App
  baseUrl: http://localhost:7007

organization:
  name: My Company

backend:
  baseUrl: http://localhost:7007
  listen:
    port: 7007
  csp:
    connect-src: ["'self'", 'http:', 'https:']
  cors:
    origin: http://localhost:7007
    methods: [GET, HEAD, PATCH, POST, PUT, DELETE]
    credentials: true
  database:
    client: better-sqlite3
    connection: ':memory:'

integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}

techdocs:
  builder: 'local'
  generator:
    runIn: 'docker'
  publisher:
    type: 'local'

auth:
  dangerouslyDisableDefaultAuthPolicy: true
  providers:
    guest: {}

catalog:
  import:
    entityFilename: catalog-info.yaml
    pullRequestBranchName: backstage-integration
  rules:
    - allow: [Component, System, API, Resource, Location]
  locations:
    - type: file
      target: ../../examples/entities.yaml
    - type: file
      target: ../../examples/template/template.yaml
      rules:
        - allow: [Template]
    - type: file
      target: ../../examples/org.yaml
      rules:
        - allow: [User, Group]

permission:
  enabled: false
```

### 3.3 app-config.production.yaml

```yaml
app:
  baseUrl: http://localhost:7007

backend:
  baseUrl: http://localhost:7007
  listen: ':7007'
  database:
    client: pg
    connection:
      host: ${POSTGRES_HOST}
      port: ${POSTGRES_PORT}
      user: ${POSTGRES_USER}
      password: ${POSTGRES_PASSWORD}
      database: ${POSTGRES_DB}

auth:
  dangerouslyDisableDefaultAuthPolicy: true
  providers:
    guest:
      dangerouslyAllowOutsideDevelopment: true

catalog:
  locations:
    - type: file
      target: ./examples/entities.yaml
    - type: file
      target: ./examples/template/template.yaml
      rules:
        - allow: [Template]
    - type: file
      target: ./examples/org.yaml
      rules:
        - allow: [User, Group]
  providers:
    keycloakOrg:
      default:
        baseUrl: http://keycloak:8080/keycloak
        loginRealm: backstage
        realm: backstage
        clientId: backstage-client
        clientSecret: ${KEYCLOAK_CLIENT_SECRET}
        schedule:
          frequency:
            minutes: 5
          timeout:
            minutes: 3
          initialDelay:
            seconds: 15

permission:
  enabled: false
```

### 3.4 Dockerfile (packages/backend/Dockerfile)

The default Dockerfile already includes everything needed. No changes required for basic setup.

```dockerfile
FROM node:24-trixie-slim

ENV PYTHON=/usr/bin/python3

# Install native module build dependencies
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && \
    apt-get install -y --no-install-recommends python3 g++ build-essential && \
    rm -rf /var/lib/apt/lists/*

# Install SQLite dependencies
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && \
    apt-get install -y --no-install-recommends libsqlite3-dev && \
    rm -rf /var/lib/apt/lists/*

USER node
WORKDIR /app

COPY --chown=node:node .yarn ./.yarn
COPY --chown=node:node .yarnrc.yml ./
COPY --chown=node:node backstage.json ./

ENV NODE_ENV=production
ENV NODE_OPTIONS="--no-node-snapshot"

COPY --chown=node:node yarn.lock package.json packages/backend/dist/skeleton.tar.gz ./
RUN tar xzf skeleton.tar.gz && rm skeleton.tar.gz

RUN --mount=type=cache,target=/home/node/.cache/yarn,sharing=locked,uid=1000,gid=1000 \
    yarn workspaces focus --all --production && rm -rf "$(yarn cache clean)"

COPY --chown=node:node examples ./examples
COPY --chown=node:node packages/backend/dist/bundle.tar.gz app-config*.yaml ./
RUN tar xzf bundle.tar.gz && rm bundle.tar.gz

CMD ["node", "packages/backend", "--config", "app-config.yaml", "--config", "app-config.production.yaml"]
```

### 3.5 Build and Run Docker Image

```bash
# Build Docker image
docker image build . -f packages/backend/Dockerfile --tag backstage:latest

# Run container (basic - SQLite)
docker run -it \
  -p 7007:7007 \
  --name backstage \
  backstage:latest

# Access at:
# http://localhost:7007
```

---

## 4. Nginx Reverse Proxy with HTTPS

### 4.1 Generate Self-Signed Certificate

```bash
# Create certs directory
mkdir -p ~/certs

# Generate self-signed certificate (valid 365 days)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout ~/certs/backstage.key \
  -out ~/certs/backstage.crt \
  -subj "/C=US/ST=Local/L=Local/O=Backstage/CN=localhost"
```

### 4.2 Install Nginx

```bash
sudo apt-get install -y nginx
```

### 4.3 Nginx Configuration

Create `/etc/nginx/sites-available/backstage`:

```nginx
# HTTP → HTTPS redirect
server {
    listen 80;
    server_name localhost;
    return 301 https://$host$request_uri;
}

# Backstage + Keycloak on same port 443
server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate     /home/YOUR_USER/certs/backstage.crt;
    ssl_certificate_key /home/YOUR_USER/certs/backstage.key;

    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    # Keycloak
    location /keycloak {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port 443;
        proxy_buffer_size 128k;
        proxy_buffers 4 256k;
        proxy_busy_buffers_size 256k;
    }

    # Backstage
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

### 4.4 Enable and Restart Nginx

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

### 4.5 Access URLs

| Service | URL |
|---------|-----|
| Backstage | `https://localhost` |
| Keycloak Admin | `https://localhost/keycloak/admin` |

> 🔒 Browser will show a security warning for self-signed cert. Click **Advanced → Proceed to localhost**.

---

## 5. Docker Network and PostgreSQL Setup

### 5.1 Create Docker Network

A shared Docker network allows containers to communicate using container names instead of IP addresses.

```bash
docker network create backstage-network
```

### 5.2 Run Shared PostgreSQL

```bash
docker run -d \
  --name backstage-postgres \
  --network backstage-network \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres123 \
  -v postgres-data:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:16
```

### 5.3 Create Databases

```bash
# Create Keycloak database
docker exec -it backstage-postgres psql -U postgres -c "CREATE DATABASE keycloak;"

# Create Backstage database
docker exec -it backstage-postgres psql -U postgres -c "CREATE DATABASE backstage;"

# Verify both databases exist
docker exec -it backstage-postgres psql -U postgres -c "\l"
```

### 5.4 Run Backstage with PostgreSQL

```bash
docker run -d \
  --name backstage \
  --network backstage-network \
  -p 7007:7007 \
  -e POSTGRES_HOST=backstage-postgres \
  -e POSTGRES_PORT=5432 \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres123 \
  -e POSTGRES_DB=backstage \
  -e KEYCLOAK_CLIENT_SECRET=YOUR_SECRET \
  backstage:latest
```

---

## 6. Keycloak Setup

### 6.1 Run Keycloak with PostgreSQL

```bash
docker run -d \
  --name keycloak \
  --network backstage-network \
  -e KC_DB=postgres \
  -e KC_DB_URL=jdbc:postgresql://backstage-postgres:5432/keycloak \
  -e KC_DB_USERNAME=postgres \
  -e KC_DB_PASSWORD=postgres123 \
  -e KC_BOOTSTRAP_ADMIN_USERNAME=admin \
  -e KC_BOOTSTRAP_ADMIN_PASSWORD=admin123 \
  -e KC_HOSTNAME=localhost \
  -e KC_HOSTNAME_PATH=/keycloak \
  -e KC_HTTP_ENABLED=true \
  -e KC_HTTP_RELATIVE_PATH=/keycloak \
  -e KC_PROXY_HEADERS=xforwarded \
  -p 8080:8080 \
  quay.io/keycloak/keycloak:latest \
  start-dev
```

### 6.2 Create Backstage Realm

1. Login to `https://localhost/keycloak/admin`
2. Click dropdown at top left (shows "Keycloak")
3. Click **"Create Realm"**
4. Set **Realm name:** `backstage`
5. Click **Create**

### 6.3 Create Client

Inside the **backstage** realm, go to **Clients → Create client**:

**General Settings:**

| Field | Value |
|-------|-------|
| Client type | OpenID Connect |
| Client ID | `backstage-client` |

**Capability Config:**

| Field | Value |
|-------|-------|
| Client authentication | ON |
| Service accounts roles | ON |
| Standard flow | ON |
| Direct access grants | ON |

**Login Settings:**

| Field | Value |
|-------|-------|
| Valid redirect URIs | `https://localhost/*` |
| Valid post logout redirect URIs | `https://localhost/*` |
| Web origins | `https://localhost` |

After saving, go to **Credentials** tab and copy the **Client secret**.

### 6.4 Assign Service Account Roles

Go to **Clients → backstage-client → Service accounts roles → Assign role → Filter by clients → realm-management**:

- `view-users`
- `query-users`
- `view-groups`
- `query-groups`
- `manage-users`
- `view-realm`

### 6.5 Add Groups Mapper

Go to **Clients → backstage-client → Client scopes → backstage-client-dedicated → Add mapper → By configuration → Group Membership**:

| Field | Value |
|-------|-------|
| Name | `groups` |
| Token Claim Name | `groups` |
| Full group path | OFF |
| Add to ID token | ON |
| Add to access token | ON |
| Add to userinfo | ON |

### 6.6 Create Groups (Hierarchical)

1. Go to **Groups → Create group** → Name: `platform-common-team` → Save
2. Click on `platform-common-team` → **Children** tab → **Create child group**
3. Create child groups: `infra-team`, `devops-team`, `sre-team`

Also create flat groups for Backstage RBAC:
- `backstage-admins`
- `backstage-developers`
- `backstage-viewers`

### 6.7 Create Users

Go to **Users → Create new user** for each:

| Username | Email | Password | Group |
|----------|-------|----------|-------|
| `backstage-admin` | `admin@backstage.local` | `admin123` | `backstage-admins` |
| `backstage-dev` | `dev@backstage.local` | `dev123` | `backstage-developers` |
| `backstage-viewer` | `viewer@backstage.local` | `viewer123` | `backstage-viewers` |

After creating each user:
1. Go to **Credentials** tab → **Set password** → Set password → Temporary: **OFF**
2. Go to **Groups** tab → **Join Group** → select group

### 6.8 Verify Token

```bash
curl -s -X POST \
  http://localhost:8080/keycloak/realms/backstage/protocol/openid-connect/token \
  -d "client_id=backstage-client" \
  -d "client_secret=YOUR_CLIENT_SECRET" \
  -d "username=backstage-admin" \
  -d "password=admin123" \
  -d "grant_type=password" | python3 -m json.tool
```

Expected response includes:
```json
{
  "groups": ["backstage-admins"],
  "preferred_username": "backstage-admin",
  "email": "admin@backstage.local"
}
```

---

## 7. Keycloak Users/Groups Sync to Backstage

### 7.1 Install Plugin

```bash
yarn --cwd packages/backend add \
  @backstage-community/plugin-catalog-backend-module-keycloak
```

### 7.2 Custom Provider for Subgroup Hierarchy

> ⚠️ **Important:** The default Keycloak plugin does not support subgroups in Keycloak 23+. Keycloak 23+ no longer embeds subgroups in the default `/groups` API response. A custom provider is required.

Create `packages/backend/src/keycloakProvider.ts`:

```typescript
import {
  EntityProvider,
  EntityProviderConnection,
} from '@backstage/plugin-catalog-node';
import { GroupEntity, UserEntity } from '@backstage/catalog-model';
import { LoggerService } from '@backstage/backend-plugin-api';

interface KeycloakGroup {
  id: string;
  name: string;
  path: string;
  subGroupCount?: number;
  parentId?: string;
}

interface KeycloakUser {
  id: string;
  username: string;
  email?: string;
  firstName?: string;
  lastName?: string;
}

export class KeycloakCustomProvider implements EntityProvider {
  private connection?: EntityProviderConnection;
  private readonly baseUrl: string;
  private readonly realm: string;
  private readonly clientId: string;
  private readonly clientSecret: string;
  private readonly logger: LoggerService;

  constructor(options: {
    baseUrl: string;
    realm: string;
    clientId: string;
    clientSecret: string;
    logger: LoggerService;
  }) {
    this.baseUrl = options.baseUrl;
    this.realm = options.realm;
    this.clientId = options.clientId;
    this.clientSecret = options.clientSecret;
    this.logger = options.logger;
  }

  getProviderName(): string {
    return 'KeycloakCustomProvider';
  }

  async connect(connection: EntityProviderConnection): Promise<void> {
    this.connection = connection;
  }

  private async getToken(): Promise<string> {
    const response = await fetch(
      `${this.baseUrl}/realms/${this.realm}/protocol/openid-connect/token`,
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
        body: new URLSearchParams({
          client_id: this.clientId,
          client_secret: this.clientSecret,
          grant_type: 'client_credentials',
        }),
      },
    );
    const data = await response.json();
    return data.access_token;
  }

  private async fetchAllGroups(token: string): Promise<KeycloakGroup[]> {
    // Fetch top-level groups
    const response = await fetch(
      `${this.baseUrl}/admin/realms/${this.realm}/groups?max=500`,
      { headers: { Authorization: `Bearer ${token}` } },
    );
    const topGroups: KeycloakGroup[] = await response.json();
    const allGroups: KeycloakGroup[] = [];

    for (const group of topGroups) {
      allGroups.push(group);
      // Fetch subgroups via /children API if any exist
      if (group.subGroupCount && group.subGroupCount > 0) {
        const subResponse = await fetch(
          `${this.baseUrl}/admin/realms/${this.realm}/groups/${group.id}/children?max=500`,
          { headers: { Authorization: `Bearer ${token}` } },
        );
        const subGroups: KeycloakGroup[] = await subResponse.json();
        for (const sub of subGroups) {
          allGroups.push({ ...sub, parentId: group.id });
        }
      }
    }
    return allGroups;
  }

  private async fetchUsers(token: string): Promise<KeycloakUser[]> {
    const response = await fetch(
      `${this.baseUrl}/admin/realms/${this.realm}/users?max=500`,
      { headers: { Authorization: `Bearer ${token}` } },
    );
    return response.json();
  }

  private async fetchUserGroups(token: string, userId: string): Promise<KeycloakGroup[]> {
    const response = await fetch(
      `${this.baseUrl}/admin/realms/${this.realm}/users/${userId}/groups`,
      { headers: { Authorization: `Bearer ${token}` } },
    );
    return response.json();
  }

  async refresh(): Promise<void> {
    if (!this.connection) return;

    this.logger.info('KeycloakCustomProvider: Reading users and groups');
    const token = await this.getToken();
    const allGroups = await this.fetchAllGroups(token);
    const users = await this.fetchUsers(token);

    // Build group entities
    const groupEntities: GroupEntity[] = allGroups.map(group => {
      const pathParts = group.path.split('/').filter(Boolean);
      const parentName = pathParts.length > 1
        ? pathParts[pathParts.length - 2]
        : undefined;

      return {
        apiVersion: 'backstage.io/v1alpha1',
        kind: 'Group',
        metadata: {
          name: group.name,
          namespace: 'default',
          annotations: {
            'backstage.io/managed-by-location': `keycloak:${this.realm}`,
            'backstage.io/managed-by-origin-location': `keycloak:${this.realm}`,
          },
        },
        spec: {
          type: 'team',
          profile: {},
          children: allGroups
            .filter(g => g.parentId === group.id)
            .map(g => g.name),
          ...(parentName ? { parent: parentName } : {}),
          members: [],
        },
      };
    });

    // Build user entities with group membership
    const userEntities: UserEntity[] = [];
    for (const user of users) {
      const userGroups = await this.fetchUserGroups(token, user.id);
      const memberOf = userGroups.map(g => g.name);

      // Populate group members
      for (const g of userGroups) {
        const groupEntity = groupEntities.find(ge => ge.metadata.name === g.name);
        if (groupEntity && !groupEntity.spec.members?.includes(user.username)) {
          groupEntity.spec.members?.push(user.username);
        }
      }

      userEntities.push({
        apiVersion: 'backstage.io/v1alpha1',
        kind: 'User',
        metadata: {
          name: user.username,
          namespace: 'default',
          annotations: {
            'backstage.io/managed-by-location': `keycloak:${this.realm}`,
            'backstage.io/managed-by-origin-location': `keycloak:${this.realm}`,
          },
        },
        spec: {
          profile: {
            email: user.email,
            displayName: `${user.firstName ?? ''} ${user.lastName ?? ''}`.trim(),
          },
          memberOf,
        },
      });
    }

    this.logger.info(
      `KeycloakCustomProvider: Ingesting ${userEntities.length} users and ${groupEntities.length} groups`,
    );

    await this.connection.applyMutation({
      type: 'full',
      entities: [...groupEntities, ...userEntities].map(entity => ({
        entity,
        locationKey: `keycloak:${this.realm}`,
      })),
    });
  }
}
```

### 7.3 Update packages/backend/src/index.ts

```typescript
import { createBackend } from '@backstage/backend-defaults';
import { coreServices, createBackendModule } from '@backstage/backend-plugin-api';
import { catalogProcessingExtensionPoint } from '@backstage/plugin-catalog-node/alpha';
import { KeycloakCustomProvider } from './keycloakProvider';

const keycloakCustomModule = createBackendModule({
  pluginId: 'catalog',
  moduleId: 'keycloak-custom-provider',
  register(reg) {
    reg.registerInit({
      deps: {
        catalog: catalogProcessingExtensionPoint,
        logger: coreServices.logger,
        config: coreServices.rootConfig,
        scheduler: coreServices.scheduler,
      },
      async init({ catalog, logger, config, scheduler }) {
        const provider = new KeycloakCustomProvider({
          baseUrl: config.getString('catalog.providers.keycloakOrg.default.baseUrl'),
          realm: config.getString('catalog.providers.keycloakOrg.default.realm'),
          clientId: config.getString('catalog.providers.keycloakOrg.default.clientId'),
          clientSecret: config.getString('catalog.providers.keycloakOrg.default.clientSecret'),
          logger,
        });

        catalog.addEntityProvider(provider);

        await scheduler.scheduleTask({
          id: 'keycloak-custom-provider-refresh',
          frequency: { minutes: 5 },
          timeout: { minutes: 3 },
          initialDelay: { seconds: 15 },
          fn: async () => {
            await provider.refresh();
          },
        });
      },
    });
  },
});

const backend = createBackend();

backend.add(import('@backstage/plugin-app-backend'));
backend.add(import('@backstage/plugin-proxy-backend'));
backend.add(import('@backstage/plugin-scaffolder-backend'));
backend.add(import('@backstage/plugin-scaffolder-backend-module-github'));
backend.add(import('@backstage/plugin-scaffolder-backend-module-notifications'));
backend.add(import('@backstage/plugin-techdocs-backend'));
backend.add(import('@backstage/plugin-auth-backend'));
backend.add(import('@backstage/plugin-auth-backend-module-guest-provider'));
backend.add(import('@backstage/plugin-catalog-backend'));
backend.add(import('@backstage/plugin-catalog-backend-module-scaffolder-entity-model'));
backend.add(import('@backstage/plugin-catalog-backend-module-logs'));
backend.add(import('@backstage/plugin-permission-backend'));
backend.add(import('@backstage/plugin-permission-backend-module-allow-all-policy'));
backend.add(import('@backstage/plugin-search-backend'));
backend.add(import('@backstage/plugin-search-backend-module-pg'));
backend.add(import('@backstage/plugin-search-backend-module-catalog'));
backend.add(import('@backstage/plugin-search-backend-module-techdocs'));
backend.add(import('@backstage/plugin-kubernetes-backend'));
backend.add(import('@backstage/plugin-notifications-backend'));
backend.add(import('@backstage/plugin-signals-backend'));
backend.add(keycloakCustomModule); // ← Custom Keycloak provider with subgroup support

backend.start();
```

### 7.4 Rebuild and Restart

```bash
yarn build:backend

docker rm -f backstage

docker image build . -f packages/backend/Dockerfile --tag backstage:latest

docker run -d \
  --name backstage \
  --network backstage-network \
  -p 7007:7007 \
  -e POSTGRES_HOST=backstage-postgres \
  -e POSTGRES_PORT=5432 \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres123 \
  -e POSTGRES_DB=backstage \
  -e KEYCLOAK_CLIENT_SECRET=YOUR_SECRET \
  backstage:latest
```

### 7.5 Verify Sync

```bash
docker logs -f backstage | grep -i keycloak
```

Expected log output:
```
KeycloakCustomProvider: Reading users and groups
KeycloakCustomProvider: Ingesting 7 users and 6 groups
```

Check in Backstage UI:
- `https://localhost/catalog?kind=Group`
- `https://localhost/catalog?kind=User`

---

## 8. Troubleshooting Guide

### 8.1 Installation Issues

---

#### ❌ Error: cpu-features and isolated-vm build failures

**Symptom:**
```
YN0009: cpu-features@npm couldn't be built successfully (exit code 1)
YN0009: isolated-vm@npm couldn't be built successfully (exit code 1)
```

**Fix:**
```bash
sudo apt-get update && sudo apt-get install -y \
  build-essential python3 python3-pip gcc g++ make cmake
```

---

#### ❌ Error: Docker command not found in WSL

**Symptom:**
```
The command 'docker' could not be found in this WSL 2 distro.
```

**Fix:**
1. Open Docker Desktop → Settings → Resources → WSL Integration
2. Enable integration with your WSL distro
3. Click Apply & Restart

---

#### ❌ Error: Yarn peer dependency warnings

**Symptom:**
```
YN0060: @testing-library/react doesn't satisfy what @backstage/test-utils requests
YN0002: app@workspace doesn't provide @types/react
```

**Fix:** These are warnings, not blockers. Add to `.yarnrc.yml`:
```yaml
nodeLinker: node-modules
```

---

### 8.2 Docker Container Issues

---

#### ❌ Error: 403 Forbidden on guest auth

**Symptom:**
```
GET /api/auth/guest/refresh 403 Forbidden
Failed to sign in as a guest
```

**Fix:** Add to `app-config.production.yaml`:
```yaml
auth:
  dangerouslyDisableDefaultAuthPolicy: true
  providers:
    guest:
      dangerouslyAllowOutsideDevelopment: true
```

**Root cause:** In production mode, the guest provider is blocked unless explicitly allowed outside development.

---

#### ❌ Error: 401 Unauthorized on all API calls

**Symptom:**
```
POST /api/permission/authorize 401 Unauthorized
GET /api/catalog/entities 401 Unauthorized
```

**Fix:** Add to `app-config.yaml`:
```yaml
permission:
  enabled: false
```

**Root cause:** The permission framework blocks unauthenticated requests even with `dangerouslyDisableDefaultAuthPolicy`.

---

#### ❌ Error: ECONNREFUSED 127.0.0.1:5432

**Symptom:**
```
connect ECONNREFUSED 127.0.0.1:5432
Failed to connect to the database
```

**Fix:** Pass PostgreSQL env vars to `docker run`:
```bash
docker run -d \
  --network backstage-network \
  -e POSTGRES_HOST=backstage-postgres \
  -e POSTGRES_PORT=5432 \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres123 \
  -e POSTGRES_DB=backstage \
  backstage:latest
```

**Root cause:** The config uses `${POSTGRES_HOST}` env vars but they weren't passed to the container. Also ensure `--network backstage-network` is set so the container can resolve `backstage-postgres` by name.

---

#### ❌ Error: CORS mismatch

**Symptom:**
```
Access-Control-Allow-Origin error
```

**Fix:** Ensure `app.baseUrl` and `cors.origin` match the port you're accessing from:
```yaml
app:
  baseUrl: http://localhost:7007  # Must match access URL
backend:
  cors:
    origin: http://localhost:7007  # Must match app.baseUrl
```

---

### 8.3 Keycloak Issues

---

#### ❌ Error: Page not found at /keycloak

**Symptom:**
```
https://localhost/keycloak
We are sorry... Page not found
```

**Fix:** Use the correct admin URL:
```
https://localhost/keycloak/admin
https://localhost/keycloak/admin/master/console
```

**Root cause:** `/keycloak` alone doesn't redirect to the admin console.

---

#### ❌ Error: Deprecated environment variables

**Symptom:**
```
WARN: Environment variable 'KEYCLOAK_ADMIN' is deprecated
use 'KC_BOOTSTRAP_ADMIN_USERNAME' instead
```

**Fix:**
```bash
# Old (deprecated)
-e KEYCLOAK_ADMIN=admin \
-e KEYCLOAK_ADMIN_PASSWORD=admin123 \

# New (correct for Keycloak 26+)
-e KC_BOOTSTRAP_ADMIN_USERNAME=admin \
-e KC_BOOTSTRAP_ADMIN_PASSWORD=admin123 \
```

---

#### ❌ Error: unauthorized_client when syncing

**Symptom:**
```
Error: unauthorized_client
Failed to authenticate with Keycloak
```

**Fix:**
1. Go to **Clients → backstage-client → Settings**
2. Turn ON **Service accounts roles**
3. Save
4. Go to **Service accounts roles** tab
5. Assign: `view-users`, `query-users`, `view-groups`, `query-groups`, `manage-users`

---

#### ❌ Error: HTTP 403 Forbidden when syncing users/groups

**Symptom:**
```
Error while syncing Keycloak users and groups HTTP 403 Forbidden
```

**Fix:** Assign the `manage-users` role to the service account (in addition to view/query roles).

---

#### ❌ Error: Subgroups not appearing in Backstage catalog

**Symptom:**
- Only top-level groups appear in catalog
- Child groups like `infra-team`, `devops-team`, `sre-team` are missing
- Logs show correct count (e.g., 6 groups) but UI only shows top-level

**Root cause:** Keycloak 23+ no longer embeds subgroups in the default `/groups` API response (`subGroups: []` is always empty). The official plugin relies on embedded subgroups.

**Diagnosis:**
```bash
# Verify subgroups exist in Keycloak
TOKEN=$(curl -s -X POST \
  http://localhost:8080/keycloak/realms/backstage/protocol/openid-connect/token \
  -d "client_id=backstage-client" \
  -d "client_secret=YOUR_SECRET" \
  -d "grant_type=client_credentials" | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

# Check /children API
curl -s -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8080/keycloak/admin/realms/backstage/groups/PARENT_GROUP_ID/children" \
  | python3 -m json.tool
```

**Fix:** Use the custom `KeycloakCustomProvider` (see Section 7) which explicitly calls the `/children` API.

---

### 8.4 Nginx Issues

---

#### ❌ Error: Nginx test fails

**Symptom:**
```
nginx: configuration file test failed
```

**Fix:**
```bash
# Check for syntax errors
sudo nginx -t

# Check logs
sudo journalctl -u nginx -n 50

# Common fixes:
# 1. Ensure cert paths are correct
# 2. Ensure no duplicate server blocks
# 3. Remove default site: sudo rm /etc/nginx/sites-enabled/default
```

---

## 9. Quick Reference

### 9.1 Full Rebuild Command

```bash
# Rebuild and restart everything
yarn install --immutable
yarn tsc
yarn build:backend

docker rm -f backstage

docker image build . -f packages/backend/Dockerfile --tag backstage:latest

docker run -d \
  --name backstage \
  --network backstage-network \
  -p 7007:7007 \
  -e POSTGRES_HOST=backstage-postgres \
  -e POSTGRES_PORT=5432 \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres123 \
  -e POSTGRES_DB=backstage \
  -e KEYCLOAK_CLIENT_SECRET=YOUR_SECRET \
  backstage:latest

docker logs -f backstage
```

### 9.2 Docker Management

```bash
# View running containers
docker ps

# View container logs
docker logs -f backstage
docker logs -f keycloak
docker logs backstage --tail=50

# Restart container
docker restart backstage

# Remove container
docker rm -f backstage

# Check container env vars
docker exec -it backstage env | grep POSTGRES
docker exec -it keycloak env | grep KC_
```

### 9.3 Verification Commands

```bash
# Get guest token
TOKEN=$(curl -sk "https://localhost/api/auth/guest/refresh" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['backstageIdentity']['token'])")

# List all groups
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/catalog/entities?filter=kind=Group" \
  | python3 -m json.tool | grep '"name"'

# List all users
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/catalog/entities?filter=kind=User" \
  | python3 -m json.tool | grep '"name"'

# Check specific group
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/catalog/entities/by-name/group/default/infra-team" \
  | python3 -m json.tool

# Test Keycloak token
curl -s -X POST \
  http://localhost:8080/keycloak/realms/backstage/protocol/openid-connect/token \
  -d "client_id=backstage-client" \
  -d "client_secret=YOUR_SECRET" \
  -d "grant_type=client_credentials" | python3 -m json.tool

# Check Keycloak subgroups
curl -s -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8080/keycloak/admin/realms/backstage/groups/GROUP_ID/children" \
  | python3 -m json.tool

# Check configs inside container
docker run --rm backstage:latest cat app-config.yaml
docker run --rm backstage:latest cat app-config.production.yaml
```

### 9.4 Service URLs

| Service | URL | Credentials |
|---------|-----|-------------|
| Backstage | `https://localhost` | Guest login |
| Keycloak Admin | `https://localhost/keycloak/admin` | admin / admin123 |
| Backstage API | `https://localhost/api` | Bearer token |
| PostgreSQL | `localhost:5432` | postgres / postgres123 |
| Keycloak HTTP | `http://localhost:8080/keycloak` | Internal only |

### 9.5 Implementation Status

| Phase | Task | Status |
|-------|------|--------|
| Phase 1 | Backstage installation on WSL Ubuntu | ✅ Complete |
| Phase 1 | Docker containerization | ✅ Complete |
| Phase 1 | Nginx HTTPS reverse proxy | ✅ Complete |
| Phase 1 | PostgreSQL shared database | ✅ Complete |
| Phase 1 | Keycloak Docker setup | ✅ Complete |
| Phase 2 | Keycloak realm and client setup | ✅ Complete |
| Phase 2 | Users and groups creation | ✅ Complete |
| Phase 2 | Flat group sync to Backstage | ✅ Complete |
| Phase 2 | Hierarchical subgroup sync (custom provider) | ✅ Complete |
| Phase 3 | Azure AD authentication | ⏳ Pending |
| Phase 4 | RBAC with Keycloak groups | ⏳ Pending |

---

*Generated: March 2026 | Backstage v1.48.0 | Keycloak 26.5.4 | Node.js v24.13.1*
