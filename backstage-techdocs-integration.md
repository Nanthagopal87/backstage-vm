# Backstage – TechDocs Integration Guide

> Hosted Documentation from GitLab Repos · Local Builder · Local Storage  
> **Plugin:** `@backstage/plugin-techdocs`  
> **Version:** 1.0 | March 2026

---

## Table of Contents

1. [Overview](#1-overview)
2. [Prerequisites](#2-prerequisites)
3. [Dockerfile – Install mkdocs](#3-dockerfile--install-mkdocs)
4. [app-config.yaml – TechDocs Configuration](#4-app-configyaml--techdocs-configuration)
5. [app-config.production.yaml – TechDocs Block](#5-app-configproductionyaml--techdocs-block)
6. [catalog-info.yaml – Required Annotation](#6-catalog-infoyaml--required-annotation)
7. [Repo Structure – mkdocs.yml and docs/](#7-repo-structure--mkdocsyml-and-docs)
8. [README.md vs docs/index.md](#8-readmemd-vs-docsindexmd)
9. [Rebuild & Deploy](#9-rebuild--deploy)
10. [Verification](#10-verification)
11. [Troubleshooting](#11-troubleshooting)
12. [Quick Reference](#12-quick-reference)

---

## 1. Overview

TechDocs turns markdown files in your GitLab repos into a hosted documentation site rendered inside Backstage. Every component with the `techdocs-ref` annotation gets a **Docs tab** that builds and serves docs on demand.

### How It Works

```
repo/
  ├── catalog-info.yaml        (annotation: backstage.io/techdocs-ref: dir:.)
  ├── mkdocs.yml               (tells mkdocs what to build)
  └── docs/
        └── index.md           (or point directly at README.md)

        ↓  Developer clicks Docs tab in Backstage
        ↓  TechDocs backend checks if docs are built
        ↓  mkdocs-techdocs-core runs inside Backstage container
        ↓  Generated HTML stored locally in container (/tmp/techdocs)
        ↓  TechDocs UI renders the docs inline
```

### What Gets Added to the UI

| Element | Location | Shows |
|---|---|---|
| Docs tab | Every component page | Full mkdocs-rendered documentation site |
| ReportIssue addon | Docs tab toolbar | Lets readers flag issues inline |
| Search integration | Backstage global search | Doc content is searchable across all components |

### Setup Mode (This Guide)

| Setting | Value | Notes |
|---|---|---|
| Builder | `local` | mkdocs runs inside Backstage container |
| Generator | `local` | No Docker-in-Docker required |
| Publisher | `local` | Docs stored at `/tmp/techdocs` in container |
| Migration path | External storage + CI build | Covered in future guide |

---

## 2. Prerequisites

| Requirement | Why Needed |
|---|---|
| Backstage running on Docker with PostgreSQL | Local publisher stores metadata in the database |
| GitLab integration already configured | TechDocs fetches repo content via the same GitLab token |
| Components already in catalog with `catalog-info.yaml` | TechDocs annotation is added to existing catalog entries |
| `node:24-trixie-slim` base image | Python3 and pip3 availability confirmed on this image |

---

## 3. Dockerfile – Install mkdocs

This is the most critical change. Since `generator.runIn: local` runs mkdocs directly inside the container, `python3`, `python3-pip`, and `mkdocs-techdocs-core` must all be installed in the image.

### Root Cause of Common Build Failure

`node:24-trixie-slim` includes `python3` but NOT `python3-pip`. Running `pip3` or `pip` without installing `python3-pip` first will fail:

```
/bin/sh: 1: pip3: not found
```

### Fix — Add python3-pip to the First apt Layer

`python3-pip` must be added to the **same** `apt-get install` block as `python3`. Adding it in a separate `RUN` step causes issues with the shared apt cache mounts.

```dockerfile
# packages/backend/Dockerfile

FROM node:24-trixie-slim

ENV PYTHON=/usr/bin/python3

# ✅ python3-pip added here alongside python3
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && \
    apt-get install -y --no-install-recommends \
      python3 \
      python3-pip \        
      g++ \
      build-essential && \
    rm -rf /var/lib/apt/lists/*

# Install sqlite3 dependencies
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && \
    apt-get install -y --no-install-recommends libsqlite3-dev && \
    rm -rf /var/lib/apt/lists/*

# ✅ Install mkdocs-techdocs-core for TechDocs local builder
# python3-pip must be installed above before this step runs
RUN pip3 install mkdocs-techdocs-core --break-system-packages

# From here on we use the least-privileged node user
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

CMD ["node", "packages/backend", \
     "--config", "app-config.yaml", \
     "--config", "app-config.production.yaml"]
```

> ⚠️ **WARNING:** Use `mkdocs-techdocs-core` — not plain `mkdocs`. The core package bundles the exact plugins and theme that Backstage TechDocs requires. Plain mkdocs will build but the output will be unstyled and missing features.

> ⚠️ **WARNING:** Always use `--break-system-packages` with pip3 on Debian/Ubuntu-based images. Without it, pip3 refuses to install into the system Python on newer Debian versions (trixie and above).

---

## 4. app-config.yaml – TechDocs Configuration

Your `app-config.yaml` already has a `techdocs` block. The only required change is `generator.runIn` from `docker` to `local`.

```yaml
# app-config.yaml

# ── BEFORE ──────────────────────────────────────────────
techdocs:
  builder: 'local'
  generator:
    runIn: 'docker'     # ← causes Docker-in-Docker issues
  publisher:
    type: 'local'

# ── AFTER ───────────────────────────────────────────────
techdocs:
  builder: 'local'
  generator:
    runIn: 'local'      # ← runs mkdocs directly in container
  publisher:
    type: 'local'
```

### Why `runIn: docker` Fails in This Setup

`runIn: docker` tells TechDocs to spin up a separate Docker container to run mkdocs. Since Backstage itself runs inside Docker, this requires Docker-in-Docker (mounting the Docker socket into the container), which is a security risk and adds operational complexity. `runIn: local` runs mkdocs as a subprocess inside the existing container — simpler, safer, and sufficient for local publisher mode.

---

## 5. app-config.production.yaml – TechDocs Block

Your production config had no `techdocs` block. Add it so production deployments use the same local builder settings:

```yaml
# app-config.production.yaml — add this block

techdocs:
  builder: 'local'
  generator:
    runIn: 'local'
  publisher:
    type: 'local'
    local:
      publishDirectory: /tmp/techdocs   # generated docs stored here in container
```

> ℹ️ **INFO:** `/tmp/techdocs` is ephemeral — docs are cleared when the container restarts and rebuilt on next access. This is acceptable for local mode. When you migrate to external storage (S3/GCS), the `publishDirectory` setting is replaced by the cloud storage config and docs persist across restarts.

---

## 6. catalog-info.yaml – Required Annotation

Add `backstage.io/techdocs-ref` to every component that should have a Docs tab. The value `dir:.` tells TechDocs to look for `mkdocs.yml` at the root of the repo.

```yaml
# catalog-info.yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: my-service
  description: My service description
  annotations:
    gitlab.com/project-slug: cloudopsedge/my-service
    backstage.io/techdocs-ref: dir:.    # ✅ add this line
spec:
  type: service
  lifecycle: production
  owner: group:default/my-team
```

### What dir:. Means

```
dir:.          → mkdocs.yml is at the repo root          (most common)
dir:./docs     → mkdocs.yml is inside a docs/ subfolder
url:<gitlab>   → fetch mkdocs.yml from a specific URL    (used in CI/CD mode)
```

---

## 7. Repo Structure – mkdocs.yml and docs/

### Minimum Required Structure

```
my-repo/
  ├── catalog-info.yaml     (with techdocs-ref annotation)
  ├── mkdocs.yml            ← required at root
  └── docs/
        └── index.md        ← minimum one page required
```

### mkdocs.yml – Minimum Config

```yaml
# mkdocs.yml
site_name: 'my-service'
site_description: 'Documentation for my-service'

nav:
  - Home: index.md

plugins:
  - techdocs-core           # required — do not remove or rename
```

> ⚠️ **WARNING:** The `techdocs-core` plugin entry is mandatory. Without it the build succeeds but Backstage cannot render the output correctly — the Docs tab will appear broken or unstyled.

### mkdocs.yml – Multi-Page Example

```yaml
# mkdocs.yml — expanded structure for services with more docs
site_name: 'payment-service'
site_description: 'Documentation for the Payment Service'

nav:
  - Home: index.md
  - Architecture: architecture.md
  - API Reference: api.md
  - Runbook: runbook.md
  - Changelog: changelog.md

plugins:
  - techdocs-core
```

Each entry in `nav` maps to a file under `docs/`:

```
docs/
  ├── index.md
  ├── architecture.md
  ├── api.md
  ├── runbook.md
  └── changelog.md
```

---

## 8. README.md vs docs/index.md

When repos only have a `README.md`, there are three ways to connect it to TechDocs. Choose the one that fits your workflow.

### Option 1 — Keep Separate (Current Default)

`README.md` is the GitLab repo landing page. `docs/index.md` is maintained independently as TechDocs content. Simple but creates duplication — both files need to be updated separately.

```
my-repo/
  ├── README.md          ← GitLab landing page (untouched by TechDocs)
  ├── mkdocs.yml
  └── docs/
        └── index.md     ← TechDocs content (separate from README)
```

### Option 2 — Point mkdocs.yml Directly at README.md ✅ Recommended

No `docs/` folder needed. mkdocs reads `README.md` directly from the repo root. One file, zero duplication.

```yaml
# mkdocs.yml
site_name: 'my-service'
docs_dir: .                   # ← scan repo root instead of docs/
nav:
  - Home: README.md           # ← use README.md directly
plugins:
  - techdocs-core
```

```
my-repo/
  ├── README.md          ← serves as BOTH GitLab landing page AND TechDocs home
  ├── mkdocs.yml
  └── catalog-info.yaml
```

> 💡 **TIP:** This is the recommended approach for repos that only have a README. When a repo grows to need proper multi-page docs, switch back to `docs_dir: docs` and add structured content then.

### Option 3 — Symlink docs/index.md → README.md

```bash
# Run in repo root
mkdir -p docs
cd docs
ln -s ../README.md index.md
```

`docs/index.md` always mirrors `README.md` automatically. No duplication, but symlinks can cause issues on Windows developer machines.

### Comparison

| Option | Duplication | Windows Safe | Multi-page Ready |
|---|---|---|---|
| Option 1 — Separate files | Yes | Yes | Yes |
| Option 2 — docs_dir: . | No | Yes | Partial |
| Option 3 — Symlink | No | No | Yes |

---

## 9. Rebuild & Deploy

After updating the Dockerfile, configs, and repo files:

```bash
# Step 1: Rebuild Backstage
yarn install --immutable
yarn tsc
yarn build:backend

# Step 2: Remove old container
docker rm -f backstage

# Step 3: Build new image
# Use --no-cache the first time to force the apt layer to re-run with python3-pip
docker image build --no-cache . -f packages/backend/Dockerfile --tag backstage:latest

# Step 4: Run container
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
  -e GITLAB_TOKEN=YOUR_GITLAB_TOKEN \
  backstage:latest

# Step 5: Watch logs
docker logs -f backstage | grep -i "techdocs\|mkdocs"
```

> 💡 **TIP:** `--no-cache` is only needed the first time after adding `python3-pip` to the Dockerfile. Subsequent builds can omit it and use the layer cache normally.

### Verify mkdocs Is Installed in the Image

```bash
# Before running the full container, verify mkdocs is present
docker run --rm backstage:latest python3 -m mkdocs --version
# Expected: mkdocs, version 1.x.x from /usr/lib/python3/...
```

---

## 10. Verification

### Check Docs Tab Appears

1. Navigate to any component in Backstage that has `backstage.io/techdocs-ref: dir:.`
2. Click the **Docs** tab
3. On first load a spinner appears while mkdocs builds — this is normal
4. Subsequent loads are served from cache instantly

### Check Build Logs

```bash
# Watch for TechDocs build activity
docker logs -f backstage | grep -i "techdocs\|mkdocs"

# Expected on successful first build:
# TechDocs: Building docs for entity component:default/my-service
# INFO    -  Running mkdocs build
# INFO    -  Documentation built successfully
# TechDocs: Successfully built docs for component:default/my-service
```

### Check mkdocs Is Reachable Inside Container

```bash
docker exec -it backstage python3 -m mkdocs --version
# Expected: mkdocs, version 1.x.x
```

### Check Annotation Is Present on Component

```bash
TOKEN=$(curl -sk "https://localhost/api/auth/guest/refresh" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['backstageIdentity']['token'])")

curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/catalog/entities/by-name/component/default/my-service" \
  | python3 -m json.tool | grep "techdocs"

# Expected:
# "backstage.io/techdocs-ref": "dir:."
```

---

## 11. Troubleshooting

---

### ❌ pip3: not found during Docker build

**Symptom:**
```
/bin/sh: 1: pip3: not found
```

**Cause:** `python3-pip` was not installed before the `pip3` command runs. On `node:24-trixie-slim`, pip is not bundled with python3.

**Fix:** Add `python3-pip` to the first `apt-get install` block:

```dockerfile
# ✅ Correct — python3-pip in same apt layer as python3
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && \
    apt-get install -y --no-install-recommends \
      python3 \
      python3-pip \
      g++ \
      build-essential && \
    rm -rf /var/lib/apt/lists/*

RUN pip3 install mkdocs-techdocs-core --break-system-packages
```

---

### ❌ Docs tab shows "Documentation not yet generated" and never loads

**Symptom:** Clicking Docs tab shows a message that docs haven't been generated, and it never progresses.

**Diagnosis:**
```bash
docker logs backstage | grep -i "error\|techdocs\|mkdocs" | tail -30
```

**Common causes:**

| Cause | Fix |
|---|---|
| `mkdocs.yml` missing from repo root | Add `mkdocs.yml` to repo root |
| `techdocs-ref` annotation missing | Add `backstage.io/techdocs-ref: dir:.` to `catalog-info.yaml` |
| `docs/` folder missing | Create `docs/index.md` or use `docs_dir: .` approach |
| `runIn: docker` still set | Change to `runIn: local` in `app-config.yaml` |
| mkdocs not installed in image | Run `docker exec -it backstage python3 -m mkdocs --version` to verify |

---

### ❌ Docs tab renders but content is unstyled / broken layout

**Symptom:** Docs tab loads but looks like raw HTML with no Backstage theming.

**Cause:** Plain `mkdocs` was installed instead of `mkdocs-techdocs-core`, or the `techdocs-core` plugin is missing from `mkdocs.yml`.

**Fix:**
```bash
# Verify correct package is installed
docker exec -it backstage pip3 show mkdocs-techdocs-core
```

```yaml
# Ensure mkdocs.yml has the plugin
plugins:
  - techdocs-core    # ← must be present, exact name
```

---

### ❌ mkdocs build fails — nav references missing file

**Symptom in logs:**
```
WARNING - A relative path to 'architecture.md' is included in the 'nav'
configuration, which is not found in the docs directory.
```

**Fix:** Every file listed in `nav` in `mkdocs.yml` must exist under `docs/`:

```yaml
# mkdocs.yml
nav:
  - Home: index.md          # docs/index.md must exist
  - Architecture: arch.md   # docs/arch.md must exist ← create this file
```

---

### ❌ Docs disappear after container restart

**Symptom:** Docs were working, container restarted, Docs tab shows "not yet generated" again.

**Cause:** This is expected behaviour with `publisher: local`. The generated docs are stored at `/tmp/techdocs` which is ephemeral — it does not persist across container restarts. Docs are rebuilt on next access.

**Fix for persistence:** Migrate to external storage (S3/GCS/Azure Blob) with a CI-based builder. This is the recommended production setup and will be covered in a separate guide.

As a temporary workaround, mount a volume:

```bash
docker run -d \
  --name backstage \
  -v backstage-techdocs:/tmp/techdocs \    # ← named volume persists docs
  # ... rest of your run flags ...
  backstage:latest
```

---

### ❌ TechDocs builds on every page load instead of using cache

**Symptom:** Each time you open the Docs tab, mkdocs runs again causing a slow load.

**Cause:** The cache check uses entity metadata stored in PostgreSQL. If the catalog entity is being re-processed frequently (e.g., short catalog sync frequency), TechDocs may treat the entity as stale and rebuild.

**Fix:** Increase the catalog sync frequency in `app-config.yaml`:

```yaml
catalog:
  providers:
    gitlab:
      catalog-scan:
        schedule:
          frequency: { minutes: 30 }   # increase from 1 minute
          timeout: { minutes: 5 }
```

---

### ❌ Error: This action requires 'catalog.entity.read' permission

**Symptom:** Docs tab shows a permission error.

**Fix:** Ensure permissions are disabled in both config files:

```yaml
# app-config.yaml and app-config.production.yaml
permission:
  enabled: false
```

---

## 12. Quick Reference

### Dockerfile Changes Summary

```dockerfile
# Add python3-pip to first apt block
apt-get install -y --no-install-recommends \
  python3 \
  python3-pip \        ← add this
  g++ \
  build-essential

# Add after apt blocks
RUN pip3 install mkdocs-techdocs-core --break-system-packages
```

### app-config.yaml TechDocs Block

```yaml
techdocs:
  builder: 'local'
  generator:
    runIn: 'local'      # not 'docker'
  publisher:
    type: 'local'
```

### app-config.production.yaml TechDocs Block

```yaml
techdocs:
  builder: 'local'
  generator:
    runIn: 'local'
  publisher:
    type: 'local'
    local:
      publishDirectory: /tmp/techdocs
```

### catalog-info.yaml Annotation

```yaml
metadata:
  annotations:
    backstage.io/techdocs-ref: dir:.
```

### mkdocs.yml — Option A (with docs/ folder)

```yaml
site_name: 'my-service'
nav:
  - Home: index.md
plugins:
  - techdocs-core
```

### mkdocs.yml — Option B (README.md as docs)

```yaml
site_name: 'my-service'
docs_dir: .
nav:
  - Home: README.md
plugins:
  - techdocs-core
```

### Verify mkdocs Installed

```bash
docker run --rm backstage:latest python3 -m mkdocs --version
docker exec -it backstage python3 -m mkdocs --version
```

### Watch TechDocs Build Logs

```bash
docker logs -f backstage | grep -i "techdocs\|mkdocs"
```

### Repo File Checklist Per Component

| File | Required | Content |
|---|---|---|
| `catalog-info.yaml` | ✅ | Must include `backstage.io/techdocs-ref: dir:.` |
| `mkdocs.yml` | ✅ | Must include `plugins: [techdocs-core]` |
| `docs/index.md` | ✅ (Option A) | At least one markdown page |
| `README.md` | ✅ (Option B) | Used directly as docs home |

---

*Generated: March 2026 | Backstage v1.48.0 | mkdocs-techdocs-core | Node.js v24.13.1*
