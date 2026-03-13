# Backstage – GitLab Integration Guide

> Catalog Management via Repo Scanning · Component Auto-Creation  
> **GitLab SaaS (gitlab.com) + Self-Hosted GitLab**  
> **Version:** 1.0 | March 2026

---

## Table of Contents

1. [Overview & Architecture](#1-overview--architecture)
2. [Prerequisites – GitLab Access Tokens](#2-prerequisites--gitlab-access-tokens)
3. [Install Required Packages](#3-install-required-packages)
4. [app-config.yaml – GitLab Integration Block](#4-app-configyaml--gitlab-integration-block)
5. [Catalog Discovery: Scanning Repos](#5-catalog-discovery-scanning-repos)
6. [catalog-info.yaml – Component Authoring Guide](#6-catalog-infoyaml--component-authoring-guide)
7. [Scheduled Auto-Discovery Setup](#7-scheduled-auto-discovery-setup)
8. [Connecting Components to Keycloak Teams](#8-connecting-components-to-keycloak-teams)
9. [Verification & Testing](#9-verification--testing)
10. [Troubleshooting](#10-troubleshooting)
11. [Quick Reference Cheatsheet](#11-quick-reference-cheatsheet)

---

## 1. Overview & Architecture

This guide extends the existing Backstage + Keycloak setup to add GitLab as a catalog source. The integration covers two distinct GitLab environments running simultaneously.

| Field | Value |
|---|---|
| Backstage Version | v1.48.0 |
| GitLab SaaS | gitlab.com (any tier) |
| Self-Hosted GitLab | GitLab CE / EE 16+ |
| Node.js | v24.13.1 (via nvm) |
| Stack Context | WSL Ubuntu · Docker · Keycloak · PostgreSQL · Nginx |

### Dual-GitLab Architecture

```
Browser (HTTPS 443)
        ↓
   Nginx (SSL termination)
        ↓
Docker Backstage (7007)
        ├── Catalog Backend
        │       ├── GitLab SaaS Discovery (gitlab.com)
        │       │       └── Groups / Namespaces → scans repos → catalog-info.yaml
        │       ├── GitLab Self-Hosted Discovery (gitlab.internal)
        │       │       └── Groups / Namespaces → scans repos → catalog-info.yaml
        │       └── KeycloakCustomProvider (users + teams)
        └── Catalog Store → PostgreSQL (5432)
```

### What This Guide Enables

| Capability | Trigger | Result |
|---|---|---|
| Full namespace scan | Scheduled (configurable) | Every repo in a GitLab group is crawled |
| catalog-info.yaml discovery | File present in repo root | Component registered automatically |
| Component types | `type:` field in catalog-info.yaml | `service` · `website` · `library` |
| Owner linking | `owner:` field references Keycloak team | Team ownership visible in Backstage |
| Dual host support | Two `integrations` blocks in config | SaaS + self-hosted run in parallel |

---

## 2. Prerequisites – GitLab Access Tokens

Both GitLab environments require a Personal Access Token (PAT) or Group Access Token with the correct scopes. A service/bot account PAT is recommended over a personal one for production use.

### 2.1 GitLab SaaS (gitlab.com)

**Steps:** In gitlab.com → User Settings → Access Tokens → Add new token

| Setting | Value |
|---|---|
| Token name | `backstage-catalog-saas` |
| Expiration | Set a long expiry and rotate annually |
| Scopes required | `read_api` (minimum) |
| Optional scope | `read_repository` (if scanning file contents) |

> ⚠️ **WARNING:** Store the token immediately after creation — GitLab only shows it once. Save it as `GITLAB_SAAS_TOKEN` in your Docker run environment or secret manager.

### 2.2 Self-Hosted GitLab

**Steps:** In your GitLab instance → User Settings → Access Tokens → Add new token

| Setting | Value |
|---|---|
| Token name | `backstage-catalog-selfhosted` |
| Scopes required | `read_api` |
| Network requirement | Backstage Docker container must be able to reach the GitLab instance hostname |
| TLS | If self-signed cert, see Section 4.4 for CA config |

> 💡 **TIP:** If you use a Group Access Token instead of a PAT, create it under the top-level group. All subgroups will be accessible with the same token.

### 2.3 Network Verification

Before proceeding, verify Backstage can reach both GitLab hosts from inside Docker:

```bash
# Test SaaS connectivity
docker exec -it backstage curl -I https://gitlab.com

# Test self-hosted connectivity (replace with your host)
docker exec -it backstage curl -I https://gitlab.internal

# Test token validity – SaaS
curl -H "PRIVATE-TOKEN: $GITLAB_SAAS_TOKEN" \
  https://gitlab.com/api/v4/user | python3 -m json.tool

# Test token validity – Self-hosted
curl -H "PRIVATE-TOKEN: $GITLAB_SH_TOKEN" \
  https://gitlab.internal/api/v4/user | python3 -m json.tool
```

---

## 3. Install Required Packages

The official Backstage GitLab integration uses two packages: one for the config/API integration and one for catalog discovery. Both are maintained by the Backstage community.

### 3.1 Install Packages

```bash
# From your Backstage project root
yarn add --cwd packages/backend \
  @backstage/integration \
  @backstage-community/plugin-catalog-backend-module-gitlab
```

> ℹ️ **INFO:** The `@backstage/integration` package provides the base GitLab integration (API calls, auth). The catalog module adds the GitLab entity provider for auto-discovery. Both are needed.

### 3.2 Register the Module in packages/backend/src/index.ts

Add the GitLab catalog module to the backend. This is similar to how you registered the custom Keycloak module:

```typescript
// packages/backend/src/index.ts

import { createBackend } from '@backstage/backend-defaults';

const backend = createBackend();

// ... your existing registrations ...
backend.add(keycloakCustomModule);  // already present

// ✅ Add GitLab catalog discovery
backend.add(
  import('@backstage-community/plugin-catalog-backend-module-gitlab/alpha')
);

backend.start();
```

> ⚠️ **WARNING:** The `/alpha` import path is required for the entity provider features (auto-discovery). Without it, only manual URL-based catalog locations work.

---

## 4. app-config.yaml – GitLab Integration Block

Configuration lives in two places: the `integrations` block (auth/API settings) and the `catalog.providers` block (discovery rules). Both must be set for each GitLab host.

### 4.1 integrations Block (app-config.yaml)

Add under the top-level `integrations` key. If you previously only had a `github` block, add `gitlab` alongside it:

```yaml
# app-config.yaml
integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}   # keep existing

  gitlab:
    # ── GitLab SaaS ──────────────────────────────────────
    - host: gitlab.com
      token: ${GITLAB_SAAS_TOKEN}

    # ── Self-Hosted GitLab ───────────────────────────────
    - host: gitlab.internal          # replace with your hostname
      token: ${GITLAB_SH_TOKEN}
      apiBaseUrl: https://gitlab.internal/api/v4
      baseUrl: https://gitlab.internal
```

> ℹ️ **INFO:** `apiBaseUrl` and `baseUrl` are optional for `gitlab.com` (Backstage knows the defaults), but required for self-hosted instances.

### 4.2 catalog.providers Block (app-config.production.yaml)

Add the GitLab entity provider configuration under `catalog.providers`. This is where discovery rules are defined — which groups to scan, how often, and what files to look for:

```yaml
# app-config.production.yaml
catalog:
  providers:
    # ── Existing Keycloak provider ───────────────────────
    keycloakOrg:
      default:
        baseUrl: http://keycloak:8080/keycloak
        # ... existing config ...

    # ── GitLab SaaS Discovery ────────────────────────────
    gitlab:
      saas:                              # logical name (arbitrary)
        host: gitlab.com
        group: my-org-group              # top-level GitLab group to scan
        entityFilename: catalog-info.yaml
        projectPattern: '.*'             # all repos (use regex to filter)
        branch: main                     # default branch to scan
        schedule:
          frequency: { minutes: 30 }
          timeout: { minutes: 10 }
          initialDelay: { seconds: 30 }

      selfhosted:                        # logical name (arbitrary)
        host: gitlab.internal
        group: internal-platform         # top-level group on self-hosted
        entityFilename: catalog-info.yaml
        projectPattern: '.*'
        branch: main
        schedule:
          frequency: { minutes: 30 }
          timeout: { minutes: 10 }
          initialDelay: { seconds: 60 }
```

> 💡 **TIP:** The logical names (`saas`, `selfhosted`) are arbitrary — they become part of the location key in Backstage logs. Use names that describe your environment clearly.

### 4.3 Filtering Projects with projectPattern

The `projectPattern` field accepts a regular expression against the full project path (`group/subgroup/repo-name`). Examples:

| Pattern | Matches |
|---|---|
| `.*` | All repos in the group (default — scan everything) |
| `^my-org/backend-.*` | Only repos whose path starts with `backend-` |
| `^my-org/(api\|service)-.*` | Repos starting with `api-` or `service-` |
| `^(?!.*archived).*$` | Exclude repos with "archived" in the path |

### 4.4 Self-Signed TLS (Self-Hosted Only)

If your self-hosted GitLab uses a self-signed certificate, Node.js will reject the connection. Add your CA cert to the Docker container:

```bash
# Mount your CA cert into the Backstage container
docker run -d \
  --name backstage \
  --network backstage-network \
  -v /path/to/ca.crt:/usr/local/share/ca-certificates/gitlab-ca.crt:ro \
  -e NODE_EXTRA_CA_CERTS=/usr/local/share/ca-certificates/gitlab-ca.crt \
  -e GITLAB_SAAS_TOKEN=$GITLAB_SAAS_TOKEN \
  -e GITLAB_SH_TOKEN=$GITLAB_SH_TOKEN \
  backstage:latest
```

---

## 5. Catalog Discovery: Scanning Repos

The GitLab entity provider supports three discovery modes that can work together. Understanding how they interact avoids duplicates and missed components.

### 5.1 Discovery Mode Comparison

| Mode | How it Works | Best For |
|---|---|---|
| Full namespace scan | Crawls all repos in a group matching `projectPattern` | Bootstrap — find all repos at once |
| catalog-info.yaml gating | Only registers repos where the file exists at `entityFilename` path | Steady-state — repos opt-in by adding the file |
| Scheduled polling | Repeats discovery on a cron-like schedule | Keeping catalog fresh as new repos are created |

> 💡 **TIP:** All three modes are active simultaneously when configured as shown in Section 4. The scan finds repos → checks for `catalog-info.yaml` → registers if found → repeats on schedule.

### 5.2 Subgroup Scanning

By default, the provider scans all subgroups recursively under the configured group. No extra config is needed — Backstage uses the GitLab API's recursive group listing.

```
# Example group structure — all scanned automatically
my-org-group/
  ├── platform/
  │     ├── infra-service        ← scanned
  │     └── monitoring-service   ← scanned
  ├── frontend/
  │     ├── web-app              ← scanned
  │     └── design-system        ← scanned
  └── libraries/
        └── shared-utils         ← scanned
```

### 5.3 Scanning Multiple Groups

To scan more than one top-level group (on either GitLab host), define additional named provider entries:

```yaml
# app-config.production.yaml
catalog:
  providers:
    gitlab:
      # Group 1
      saas-platform:
        host: gitlab.com
        group: my-org/platform
        entityFilename: catalog-info.yaml
        schedule: { frequency: { minutes: 30 }, timeout: { minutes: 10 } }

      # Group 2 (same host, different group)
      saas-products:
        host: gitlab.com
        group: my-org/products
        entityFilename: catalog-info.yaml
        schedule: { frequency: { minutes: 30 }, timeout: { minutes: 10 } }

      # Self-hosted group
      selfhosted-internal:
        host: gitlab.internal
        group: internal-platform
        entityFilename: catalog-info.yaml
        schedule: { frequency: { minutes: 30 }, timeout: { minutes: 10 } }
```

### 5.4 Allow Rules for Catalog Entity Kinds

Backstage's catalog enforces an allowlist of entity kinds. Ensure `Component`, `System`, `API`, and `Resource` are allowed in `app-config.yaml`:

```yaml
# app-config.yaml
catalog:
  rules:
    - allow: [Component, System, API, Resource, Location, Template]
```

> ⚠️ **WARNING:** Without `Component` in the rules list, discovered `catalog-info.yaml` files will be silently ignored and no components will appear in the catalog.

---

## 6. catalog-info.yaml – Component Authoring Guide

Each repo that should appear in the Backstage catalog needs a `catalog-info.yaml` file at its root. This section covers templates for all three component types.

### 6.1 File Location

```
my-repo/
  ├── catalog-info.yaml   ← required at root
  ├── src/
  ├── README.md
  └── ...
```

> ℹ️ **INFO:** The file must be at the exact path configured in `entityFilename` (default: `catalog-info.yaml`). Subdirectory placement is supported but requires adjusting `entityFilename` in the provider config.

### 6.2 Component Type: Service (Backend API)

Use for backend services, APIs, microservices, and workers.

```yaml
# catalog-info.yaml – Service
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: payment-service          # unique name in the catalog
  description: Handles payment processing and refunds
  tags:
    - nodejs
    - payment
    - backend
  annotations:
    # Links back to this exact GitLab repo
    gitlab.com/project-slug: my-org/platform/payment-service
    # Optional: link to TechDocs
    backstage.io/techdocs-ref: dir:.
spec:
  type: service
  lifecycle: production          # production | experimental | deprecated
  owner: group:default/infra-team    # matches Keycloak group name
  system: payment-platform       # logical system grouping (optional)
  providesApis:
    - payment-api                # references an API entity (optional)
```

### 6.3 Component Type: Website (Frontend App)

Use for frontend applications, SPAs, portals, and static sites.

```yaml
# catalog-info.yaml – Website
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: customer-portal
  description: Customer-facing portal built with React
  tags:
    - react
    - typescript
    - frontend
  annotations:
    gitlab.com/project-slug: my-org/frontend/customer-portal
spec:
  type: website
  lifecycle: production
  owner: group:default/devops-team
  system: customer-platform
  consumesApis:
    - payment-api                # APIs this frontend calls (optional)
```

### 6.4 Component Type: Library (Shared Package)

Use for internal npm/pip/maven packages and shared utility libraries.

```yaml
# catalog-info.yaml – Library
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: shared-utils
  description: Common utilities shared across all backend services
  tags:
    - library
    - nodejs
  annotations:
    gitlab.com/project-slug: my-org/libraries/shared-utils
spec:
  type: library
  lifecycle: production
  owner: group:default/platform-common-team
```

### 6.5 Self-Hosted GitLab Annotation

For repos on self-hosted GitLab, the annotation key changes to reflect your instance hostname:

```yaml
# For self-hosted GitLab (replace host in the annotation key)
metadata:
  annotations:
    gitlab.internal/project-slug: internal-platform/infra/deploy-service

# Pattern: <gitlab-host>/project-slug: <group>/<subgroup>/<repo>
```

### 6.6 lifecycle Values Reference

| Value | Meaning |
|---|---|
| `production` | Live, customer-facing, on-call coverage expected |
| `experimental` | In development or beta — not yet stable |
| `deprecated` | Scheduled for retirement — avoid new dependencies |

### 6.7 owner Field – Linking to Keycloak Teams

The `owner` field connects catalog components to your Keycloak-synced groups. Use the exact group name that exists in the Backstage catalog (populated by `KeycloakCustomProvider`):

| Keycloak Group Name | catalog-info.yaml owner value |
|---|---|
| `infra-team` | `group:default/infra-team` |
| `devops-team` | `group:default/devops-team` |
| `sre-team` | `group:default/sre-team` |
| `platform-common-team` | `group:default/platform-common-team` |

---

## 7. Scheduled Auto-Discovery Setup

The `schedule` block in each provider entry controls how often Backstage polls GitLab. Discovery is incremental — only new or changed repos are re-processed on subsequent runs.

### 7.1 Schedule Parameters

| Parameter | Type | Description | Recommended Value |
|---|---|---|---|
| `frequency` | Duration | How often to run discovery | 30 minutes |
| `timeout` | Duration | Max time one discovery run may take | 10 minutes |
| `initialDelay` | Duration | Wait after Backstage starts before first run | 30–60 seconds |

### 7.2 Recommended Schedule Configuration

```yaml
# Recommended production schedule
schedule:
  frequency:
    minutes: 30          # poll every 30 minutes
  timeout:
    minutes: 10          # fail if discovery takes > 10 min
  initialDelay:
    seconds: 30          # wait 30s after Backstage starts

# For large GitLab instances (500+ repos), increase timeout:
schedule:
  frequency: { minutes: 60 }
  timeout: { minutes: 20 }
  initialDelay: { seconds: 60 }
```

### 7.3 Stagger Multiple Providers

When running multiple provider instances (SaaS + self-hosted), use different `initialDelay` values to prevent simultaneous discovery runs from overloading the catalog:

```yaml
gitlab:
  saas:
    host: gitlab.com
    # ...
    schedule:
      frequency: { minutes: 30 }
      timeout: { minutes: 10 }
      initialDelay: { seconds: 30 }    # ← starts first

  selfhosted:
    host: gitlab.internal
    # ...
    schedule:
      frequency: { minutes: 30 }
      timeout: { minutes: 10 }
      initialDelay: { seconds: 90 }    # ← starts 60s later
```

---

## 8. Connecting Components to Keycloak Teams

Since your team and user data is already synced from Keycloak (via `KeycloakCustomProvider`), you can link GitLab-discovered components to those teams. No additional code changes are needed — it's purely a matter of the `owner` field in `catalog-info.yaml`.

### 8.1 How Entity Linking Works

```
Keycloak Group "infra-team"
    └── KeycloakCustomProvider syncs → Group entity: group:default/infra-team

GitLab repo "infra-service" with catalog-info.yaml:
    spec:
      owner: group:default/infra-team    ← references the Keycloak group

Result in Backstage:
    Component "infra-service" → owned by → Group "infra-team"
    Group "infra-team" → shows all owned components
```

### 8.2 Verify Available Group Names

Before writing `catalog-info.yaml` files, confirm the exact group names available in your catalog:

```bash
# Get token
TOKEN=$(curl -sk "https://localhost/api/auth/guest/refresh" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['backstageIdentity']['token'])")

# List all groups (shows the names to use in owner: field)
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/catalog/entities?filter=kind=Group" \
  | python3 -m json.tool | grep '"name"'
```

### 8.3 RBAC Groups as owners

You can also use RBAC groups (`type: rbac-group` from Keycloak) as owners. This is useful for shared infrastructure components not owned by a single team:

```yaml
# catalog-info.yaml for a shared platform component
spec:
  type: service
  lifecycle: production
  owner: group:default/backstage-admins   # rbac-group as owner
```

---

## 9. Verification & Testing

After configuration, rebuild and restart Backstage. Use these commands to confirm discovery is working correctly.

### 9.1 Rebuild After Config Changes

```bash
# Rebuild Backstage image after adding packages and config
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
  -e KEYCLOAK_CLIENT_SECRET=YOUR_KEYCLOAK_SECRET \
  -e GITLAB_SAAS_TOKEN=YOUR_SAAS_TOKEN \
  -e GITLAB_SH_TOKEN=YOUR_SH_TOKEN \
  backstage:latest
```

### 9.2 Watch Discovery Logs

```bash
# Watch live logs for GitLab discovery activity
docker logs -f backstage | grep -i gitlab

# Expected output when working:
# GitLabDiscoveryProcessor: Processing GitLab projects...
# Discovered 12 projects from gitlab.com group my-org
# Ingesting 8 entities from gitlab.com
```

### 9.3 Verify Components in Catalog via API

```bash
# Get guest token
TOKEN=$(curl -sk "https://localhost/api/auth/guest/refresh" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['backstageIdentity']['token'])")

# List all discovered components
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/catalog/entities?filter=kind=Component" \
  | python3 -m json.tool | grep -E '"name"|"type"|"owner"'

# Check a specific component
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/catalog/entities/by-name/component/default/payment-service" \
  | python3 -m json.tool

# Filter by type
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/catalog/entities?filter=kind=Component,spec.type=service" \
  | python3 -m json.tool | grep '"name"'
```

---

## 10. Troubleshooting

### 10.1 GitLab Discovery Errors

---

#### ❌ No components appearing after startup

**Symptom:** Catalog shows no GitLab components even after waiting for `initialDelay`.

Diagnosis steps:
- Check logs: `docker logs backstage | grep -i "gitlab\|error\|warn"`
- Verify token is in env: `docker exec -it backstage env | grep GITLAB`
- Confirm `catalog.rules` includes `Component` in `app-config.yaml`
- Confirm the `gitlab` provider block is under `catalog.providers` (not `catalog.locations`)

---

#### ❌ Error: 401 Unauthorized from GitLab API

```
# Symptom in logs:
GitLab API error: 401 Unauthorized
Failed to fetch projects for group my-org
```

**Fix:** The token is missing, expired, or lacks `read_api` scope.

```bash
# Test token manually
curl -H "PRIVATE-TOKEN: $GITLAB_SAAS_TOKEN" \
  https://gitlab.com/api/v4/groups/my-org | python3 -m json.tool

# If 401: regenerate token and ensure read_api scope is checked
# Then restart container with new token:
docker rm -f backstage && docker run -d ... -e GITLAB_SAAS_TOKEN=NEW_TOKEN ...
```

---

#### ❌ Error: UNABLE_TO_VERIFY_LEAF_SIGNATURE (Self-Hosted)

```
# Symptom in logs:
Error: UNABLE_TO_VERIFY_LEAF_SIGNATURE
certificate chain verification failed for gitlab.internal
```

**Fix:** Node.js cannot validate the self-hosted GitLab TLS certificate.
- Obtain your GitLab instance's CA certificate (`.crt` or `.pem` file)
- Mount it into the Docker container and set `NODE_EXTRA_CA_CERTS` (see Section 4.4)
- Restart the container

---

#### ❌ Error: 404 – Group not found

```
# Symptom in logs:
GitLab API returned 404 for group: my-org-group
```

**Fix:** The `group` value in the config must match the GitLab group's URL path, not its display name.

```yaml
# GitLab group URL: https://gitlab.com/my-company/platform-team
# Correct config:
group: my-company/platform-team    # full URL path without hostname

# Wrong:
group: Platform Team               # display name does not work
```

---

#### ❌ catalog-info.yaml files present but components not created

```
# Symptom in logs:
Skipping entity from location: not allowed by catalog rules
```

**Fix:** The entity kind in `catalog-info.yaml` is not in the `catalog.rules` allowlist.

```yaml
# app-config.yaml
catalog:
  rules:
    - allow: [Component, System, API, Resource, Location, Template]
    # ↑ Ensure Component is listed
```

---

#### ❌ Components appear but owner is unresolved (shows as plain string)

**Symptom:** Components show `infra-team` instead of the linked Group entity in the catalog UI.

**Root cause:** The `owner` value format is wrong, or the Keycloak sync has not run yet.

```yaml
# Wrong (missing group: prefix):
owner: infra-team

# Wrong (wrong namespace):
owner: group:production/infra-team

# Correct:
owner: group:default/infra-team
```

---

#### ❌ Conflict warnings between GitLab and another provider

```
# Symptom in logs:
Source gitlab:saas detected conflicting entityRef
component:default/my-service already referenced by gitlab:selfhosted
```

**Fix:** The same repo is being picked up by two GitLab provider instances. Narrow the `projectPattern` regex on one or both providers so their scanned groups do not overlap.

---

#### ❌ Discovery times out for large GitLab groups

```
# Symptom in logs:
GitLab discovery timed out after 10 minutes
```

**Fix:** Increase the timeout and stagger with `initialDelay`:

```yaml
schedule:
  frequency: { minutes: 60 }
  timeout: { minutes: 30 }     # increase for large instances
  initialDelay: { seconds: 60 }
```

---

## 11. Quick Reference Cheatsheet

### 11.1 Full Docker Run Command (with GitLab tokens)

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
  -e KEYCLOAK_CLIENT_SECRET=YOUR_KEYCLOAK_SECRET \
  -e GITLAB_SAAS_TOKEN=YOUR_SAAS_PAT \
  -e GITLAB_SH_TOKEN=YOUR_SELFHOSTED_PAT \
  backstage:latest
```

### 11.2 catalog-info.yaml Minimal Template

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: <repo-name>
  description: <one-line description>
  annotations:
    gitlab.com/project-slug: <group>/<subgroup>/<repo>
spec:
  type: service | website | library
  lifecycle: production | experimental | deprecated
  owner: group:default/<keycloak-group-name>
```

### 11.3 Useful Verification Commands

```bash
# Watch discovery logs
docker logs -f backstage | grep -i gitlab

# List all components
TOKEN=$(curl -sk https://localhost/api/auth/guest/refresh | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['backstageIdentity']['token'])")

curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/catalog/entities?filter=kind=Component" \
  | python3 -m json.tool | grep '"name"'

# Force re-scan (restart container)
docker restart backstage
```

### 11.4 Config File Summary

| File | What to add |
|---|---|
| `app-config.yaml` | `integrations.gitlab` block (both hosts) + `catalog.rules` |
| `app-config.production.yaml` | `catalog.providers.gitlab` block (both provider entries with schedule) |
| `packages/backend/src/index.ts` | `backend.add(import(...catalog-backend-module-gitlab/alpha))` |
| `packages/backend/package.json` | `@backstage-community/plugin-catalog-backend-module-gitlab` |
| Each repo root | `catalog-info.yaml` with `kind`, `type`, `owner`, `annotations` |

### 11.5 Implementation Status

| Phase | Task | Status |
|---|---|---|
| Phase 1 | Backstage installation on WSL Ubuntu | ✅ Complete |
| Phase 1 | Docker containerization | ✅ Complete |
| Phase 1 | Nginx HTTPS reverse proxy | ✅ Complete |
| Phase 1 | PostgreSQL shared database | ✅ Complete |
| Phase 1 | Keycloak Docker setup | ✅ Complete |
| Phase 2 | Keycloak realm and client setup | ✅ Complete |
| Phase 2 | Users and groups creation | ✅ Complete |
| Phase 2 | Hierarchical subgroup sync (custom provider) | ✅ Complete |
| Phase 3 | GitLab SaaS catalog integration | ✅ Complete |
| Phase 3 | GitLab self-hosted catalog integration | ✅ Complete |
| Phase 3 | catalog-info.yaml component authoring | ✅ Complete |
| Phase 3 | Scheduled auto-discovery | ✅ Complete |
| Phase 4 | Azure AD authentication | ⏳ Pending |
| Phase 4 | RBAC with Keycloak groups | ⏳ Pending |

---

*Generated: March 2026 | Backstage v1.48.0 | Keycloak 26.5.4 | Node.js v24.13.1*
