# Backstage – GitLab CI/CD Integration Guide

> Pipeline Status · Merge Request Visibility · Language Breakdown  
> **Plugin:** `@immobiliarelabs/backstage-plugin-gitlab`  
> **Version:** 1.0 | March 2026

---

## Table of Contents

1. [Overview](#1-overview)
2. [Prerequisites](#2-prerequisites)
3. [Install Packages](#3-install-packages)
4. [Backend Wiring](#4-backend-wiring)
5. [Frontend Wiring – EntityPage.tsx](#5-frontend-wiring--entitypagetsx)
6. [Configuration – app-config.yaml](#6-configuration--app-configyaml)
7. [catalog-info.yaml – Required Annotation](#7-catalog-infoyaml--required-annotation)
8. [Rebuild & Deploy](#8-rebuild--deploy)
9. [Verification](#9-verification)
10. [Troubleshooting](#10-troubleshooting)
11. [Quick Reference](#11-quick-reference)

---

## 1. Overview

This guide adds GitLab CI/CD visibility to Backstage component pages. Once integrated, every component with a `gitlab.com/project-slug` annotation gets a **CI/CD tab** showing pipeline runs and merge requests, and a **language breakdown card** on the overview tab.

### What Gets Added to the UI

| Element | Location | Shows |
|---|---|---|
| `EntityGitlabPipelinesTable` | CI/CD tab | Recent pipeline runs, status, branch, duration, trigger |
| `EntityGitlabMergeRequestsTable` | CI/CD tab | Open and recent MRs with status and author |
| `EntityGitlabLanguageCard` | Overview tab | Repo language breakdown by percentage |

### How It Works

```
catalog-info.yaml annotation
        ↓
  isGitlabAvailable() check
        ↓
  Backstage backend plugin
        ↓ (proxies token — never exposed to browser)
  GitLab API (SaaS or self-hosted)
        ↓
  Pipeline / MR data rendered in UI
```

---

## 2. Prerequisites

Before starting, confirm the following are already in place:

| Requirement | Why Needed |
|---|---|
| GitLab integration block in `app-config.yaml` | Plugin reuses the same tokens configured for catalog discovery |
| `read_api` scope on both GitLab tokens | Required to query pipeline and MR data via GitLab API |
| Components already in catalog with `catalog-info.yaml` | Plugin needs the `project-slug` annotation to know which repo to query |
| Backstage running on Node.js v24+ | ESM compatibility requirement of the plugin |

### Verify Token Scopes

```bash
# Verify SaaS token has read_api access to pipelines
curl -H "PRIVATE-TOKEN: $GITLAB_SAAS_TOKEN" \
  "https://gitlab.com/api/v4/projects/my-org%2Fmy-repo/pipelines?per_page=5" \
  | python3 -m json.tool

# Verify self-hosted token
curl -H "PRIVATE-TOKEN: $GITLAB_SH_TOKEN" \
  "https://gitlab.internal/api/v4/projects/my-group%2Fmy-repo/pipelines?per_page=5" \
  | python3 -m json.tool
```

> ⚠️ **WARNING:** If either returns `401 Unauthorized` or `403 Forbidden`, fix the token scope before proceeding. The plugin will silently show empty tables if the token lacks access.

---

## 3. Install Packages

Two packages are required — one for the frontend UI and one for the backend API proxy.

```bash
# Frontend plugin — installs into packages/app
yarn --cwd packages/app add @immobiliarelabs/backstage-plugin-gitlab

# Backend plugin — installs into packages/backend
yarn --cwd packages/backend add @immobiliarelabs/backstage-plugin-gitlab-backend
```

> ⚠️ **WARNING:** Note the argument order — `yarn --cwd <path> add <package>`. This is Yarn Berry (v2+) syntax. The Yarn v1 syntax `yarn add --cwd` does **not** work and will install to the wrong workspace silently.

### Verify Installation

```bash
# Confirm frontend plugin is in packages/app
cat packages/app/package.json | grep immobiliarelabs

# Confirm backend plugin is in packages/backend
cat packages/backend/package.json | grep immobiliarelabs
```

### Check Available Exports (Useful When Docs Are Outdated)

Since the plugin uses ESM, `require()` won't work to inspect exports. Use the type definitions instead:

```bash
# Dump all exported names from the installed version
grep "^export" node_modules/@immobiliarelabs/backstage-plugin-gitlab/dist/index.d.ts
```

This is the ground truth for what the installed version actually exports — useful when TypeScript errors suggest a component name has changed between versions.

---

## 4. Backend Wiring

Register the backend plugin in `packages/backend/src/index.ts`. Add it alongside your existing registrations:

```typescript
// packages/backend/src/index.ts

import { createBackend } from '@backstage/backend-defaults';

const backend = createBackend();

// ... existing registrations ...
backend.add(keycloakCustomModule);                                        // already present
backend.add(import('@backstage-community/plugin-catalog-backend-module-gitlab/alpha')); // already present

// ✅ Add GitLab backend plugin
backend.add(import('@immobiliarelabs/backstage-plugin-gitlab-backend'));

backend.start();
```

> ℹ️ **INFO:** The backend plugin acts as a secure proxy. All GitLab API calls go through it — the GitLab token is never sent to the browser.

---

## 5. Frontend Wiring – EntityPage.tsx

This is the most involved step. Three things change in `EntityPage.tsx`:

1. Import the GitLab components
2. Replace the `cicdContent` block to use GitLab pipelines with a fallback
3. Add `EntityGitlabLanguageCard` to `serviceEntityPage` and `websiteEntityPage` overviews

### 5.1 Imports

Add to the existing imports block:

```typescript
import {
  isGitlabAvailable,
  EntityGitlabPipelinesTable,
  EntityGitlabMergeRequestsTable,
  EntityGitlabLanguageCard,
} from '@immobiliarelabs/backstage-plugin-gitlab';
```

> ℹ️ **INFO:** `EntityGitlabLanguageCard` is the correct export name in current versions. Older docs may reference `EntityGitlabProjectLanguagesCard` which no longer exists and will cause a TypeScript error.

### 5.2 cicdContent Block

Replace the existing `cicdContent` entirely:

```tsx
// ── BEFORE (original placeholder) ──────────────────────────────────────────
const cicdContent = (
  <EntitySwitch>
    <EntitySwitch.Case>
      <EmptyState
        title="No CI/CD available for this entity"
        missing="info"
        description="You need to add an annotation..."
        action={<Button ...>Read more</Button>}
      />
    </EntitySwitch.Case>
  </EntitySwitch>
);

// ── AFTER (with GitLab pipelines) ──────────────────────────────────────────
const cicdContent = (
  <EntitySwitch>
    <EntitySwitch.Case if={isGitlabAvailable}>
      <Grid container spacing={3}>
        <Grid item xs={12}>
          <EntityGitlabPipelinesTable />
        </Grid>
        <Grid item xs={12}>
          <EntityGitlabMergeRequestsTable />
        </Grid>
      </Grid>
    </EntitySwitch.Case>

    <EntitySwitch.Case>
      <EmptyState
        title="No CI/CD available for this entity"
        missing="info"
        description="You need to add an annotation to your component if you want to enable CI/CD for it. You can read more about annotations in Backstage by clicking the button below."
        action={
          <Button
            variant="contained"
            color="primary"
            href="https://backstage.io/docs/features/software-catalog/well-known-annotations"
          >
            Read more
          </Button>
        }
      />
    </EntitySwitch.Case>
  </EntitySwitch>
);
```

### 5.3 Language Card in serviceEntityPage Overview

Inside `serviceEntityPage`, expand the overview route to include the language card. Replace the shared `overviewContent` reference with an inline grid that conditionally renders the card:

```tsx
const serviceEntityPage = (
  <EntityLayout>
    <EntityLayout.Route path="/" title="Overview">
      <Grid container spacing={3} alignItems="stretch">
        {entityWarningContent}
        <Grid item md={6}>
          <EntityAboutCard variant="gridItem" />
        </Grid>
        <Grid item md={6} xs={12}>
          <EntityCatalogGraphCard variant="gridItem" height={400} />
        </Grid>

        {/* ✅ Added: only renders when gitlab annotation is present */}
        <EntitySwitch>
          <EntitySwitch.Case if={isGitlabAvailable}>
            <Grid item md={4} xs={12}>
              <EntityGitlabLanguageCard />
            </Grid>
          </EntitySwitch.Case>
        </EntitySwitch>

        <Grid item md={4} xs={12}>
          <EntityLinksCard />
        </Grid>
        <Grid item md={8} xs={12}>
          <EntityHasSubcomponentsCard variant="gridItem" />
        </Grid>
      </Grid>
    </EntityLayout.Route>

    <EntityLayout.Route path="/ci-cd" title="CI/CD">
      {cicdContent}
    </EntityLayout.Route>

    {/* ... rest of routes unchanged ... */}
  </EntityLayout>
);
```

### 5.4 Language Card in websiteEntityPage Overview

Apply the same change to `websiteEntityPage`:

```tsx
const websiteEntityPage = (
  <EntityLayout>
    <EntityLayout.Route path="/" title="Overview">
      <Grid container spacing={3} alignItems="stretch">
        {entityWarningContent}
        <Grid item md={6}>
          <EntityAboutCard variant="gridItem" />
        </Grid>
        <Grid item md={6} xs={12}>
          <EntityCatalogGraphCard variant="gridItem" height={400} />
        </Grid>

        {/* ✅ Added: only renders when gitlab annotation is present */}
        <EntitySwitch>
          <EntitySwitch.Case if={isGitlabAvailable}>
            <Grid item md={4} xs={12}>
              <EntityGitlabLanguageCard />
            </Grid>
          </EntitySwitch.Case>
        </EntitySwitch>

        <Grid item md={4} xs={12}>
          <EntityLinksCard />
        </Grid>
        <Grid item md={8} xs={12}>
          <EntityHasSubcomponentsCard variant="gridItem" />
        </Grid>
      </Grid>
    </EntityLayout.Route>

    <EntityLayout.Route path="/ci-cd" title="CI/CD">
      {cicdContent}
    </EntityLayout.Route>

    {/* ... rest of routes unchanged ... */}
  </EntityLayout>
);
```

> 💡 **TIP:** The `defaultEntityPage` (used by `library` and other types) intentionally keeps using the shared `overviewContent` without the GitLab card. This is correct — not all component types need it.

---

## 6. Configuration – app-config.yaml

The plugin reuses the `integrations.gitlab` block already configured for catalog discovery. No new top-level block is needed. Confirm both entries exist:

```yaml
# app-config.yaml
integrations:
  gitlab:
    # ── GitLab SaaS ──────────────────────────────────────
    - host: gitlab.com
      token: ${GITLAB_SAAS_TOKEN}

    # ── Self-Hosted GitLab ───────────────────────────────
    - host: gitlab.internal
      token: ${GITLAB_SH_TOKEN}
      apiBaseUrl: https://gitlab.internal/api/v4
      baseUrl: https://gitlab.internal
```

> ℹ️ **INFO:** `apiBaseUrl` and `baseUrl` are optional for `gitlab.com` but required for self-hosted instances. Without them the plugin won't know how to reach your internal GitLab API.

---

## 7. catalog-info.yaml – Required Annotation

This is the most commonly missed step. The plugin uses the annotation to know which GitLab project to fetch data for. Without it, `isGitlabAvailable` returns `false` and the CI/CD tab shows the empty state fallback.

### For GitLab SaaS

```yaml
metadata:
  annotations:
    gitlab.com/project-slug: my-org/platform/payment-service
```

### For Self-Hosted GitLab

```yaml
metadata:
  annotations:
    gitlab.internal/project-slug: internal-platform/infra/deploy-service
```

### Annotation Pattern

```
<gitlab-host>/project-slug: <group>/<subgroup>/<repo-name>
```

The value must match the project's full URL path in GitLab — not the display name.

```bash
# Find the correct slug from the GitLab project URL:
# https://gitlab.com/my-company/platform/payment-service
#                   └─────────────────────────────────┘
#                   my-company/platform/payment-service  ← this is the slug
```

### Full catalog-info.yaml Example

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: payment-service
  description: Handles payment processing and refunds
  annotations:
    gitlab.com/project-slug: my-org/platform/payment-service
    backstage.io/techdocs-ref: dir:.
spec:
  type: service
  lifecycle: production
  owner: group:default/infra-team
```

---

## 8. Rebuild & Deploy

After all changes, rebuild the Backstage image and restart the container:

```bash
# Step 1: Rebuild
yarn install --immutable
yarn tsc
yarn build:backend

# Step 2: Remove old container
docker rm -f backstage

# Step 3: Build new image
docker image build . -f packages/backend/Dockerfile --tag backstage:latest

# Step 4: Run with tokens
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

# Step 5: Watch logs
docker logs -f backstage
```

---

## 9. Verification

### Check Backend Plugin Loaded

```bash
docker logs backstage | grep -i gitlab
# Expected:
# GitLab backend plugin initialized
# GitLab integration found for host: gitlab.com
# GitLab integration found for host: gitlab.internal
```

### Check a Component Has the Annotation

```bash
TOKEN=$(curl -sk "https://localhost/api/auth/guest/refresh" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['backstageIdentity']['token'])")

# Check specific component annotations
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/catalog/entities/by-name/component/default/payment-service" \
  | python3 -m json.tool | grep "project-slug"
```

### Check GitLab API Reachable from Container

```bash
# Test SaaS pipeline API directly
docker exec -it backstage curl -s \
  -H "PRIVATE-TOKEN: $GITLAB_SAAS_TOKEN" \
  "https://gitlab.com/api/v4/projects/my-org%2Fmy-repo/pipelines?per_page=3" \
  | python3 -m json.tool

# Test self-hosted
docker exec -it backstage curl -s \
  -H "PRIVATE-TOKEN: $GITLAB_SH_TOKEN" \
  "https://gitlab.internal/api/v4/projects/my-group%2Fmy-repo/pipelines?per_page=3" \
  | python3 -m json.tool
```

---

## 10. Troubleshooting

---

### ❌ CI/CD tab shows empty state despite annotation being present

**Symptom:** Component has the `project-slug` annotation but the CI/CD tab shows "No CI/CD available for this entity."

**Diagnosis:**
```bash
# Confirm annotation key matches the GitLab host exactly
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/catalog/entities/by-name/component/default/my-service" \
  | python3 -m json.tool | grep "project-slug"
```

**Fix:** The annotation key must exactly match the host configured in `integrations.gitlab`.

```yaml
# If integrations.gitlab host is: gitlab.internal
# Annotation key must be:
gitlab.internal/project-slug: my-group/my-repo   # ✅ correct

# NOT:
gitlab.com/project-slug: my-group/my-repo         # ❌ wrong host
```

---

### ❌ Pipelines table is empty — no data shown

**Symptom:** CI/CD tab renders (not showing empty state) but the pipelines table has no rows.

**Diagnosis:**
```bash
# Test token can access this specific project's pipelines
curl -H "PRIVATE-TOKEN: $GITLAB_SAAS_TOKEN" \
  "https://gitlab.com/api/v4/projects/my-org%2Fmy-repo/pipelines" \
  | python3 -m json.tool
```

**Possible causes:**

| Cause | Fix |
|---|---|
| Token lacks `read_api` scope | Regenerate token with `read_api` checked |
| Service account has < Reporter access on the project | Add the service account as Reporter on the GitLab project |
| Project has no pipeline runs yet | Run a pipeline manually in GitLab first |
| `project-slug` path is wrong | Check the exact URL path in GitLab — copy from the browser address bar |

---

### ❌ TypeScript error: has no exported member named 'EntityGitlabProjectLanguagesCard'

**Symptom:**
```
'@immobiliarelabs/backstage-plugin-gitlab' has no exported member
named 'EntityGitlabProjectLanguagesCard'
```

**Fix:** The component was renamed in a newer version. Use the correct name:

```typescript
// ❌ Old name (removed)
import { EntityGitlabProjectLanguagesCard } from '@immobiliarelabs/backstage-plugin-gitlab';

// ✅ Current name
import { EntityGitlabLanguageCard } from '@immobiliarelabs/backstage-plugin-gitlab';
```

To check what the installed version actually exports:
```bash
grep "^export" node_modules/@immobiliarelabs/backstage-plugin-gitlab/dist/index.d.ts
```

---

### ❌ Error: Cannot find module 'dayjs/plugin/relativeTime'

**Symptom:**
```
Error [ERR_MODULE_NOT_FOUND]: Cannot find module 'dayjs/plugin/relativeTime'
imported from .../backstage-plugin-gitlab/dist/components/utils.esm.js
Did you mean to import "dayjs/plugin/relativeTime.js"?
```

**Cause:** This error appears when trying to `require()` the plugin directly in Node.js. The plugin is ESM-only and cannot be loaded via CommonJS `require()`.

**Fix:** Do not use `node -e "require(...)"` to inspect this package. Use the `.d.ts` approach instead:

```bash
# ❌ Won't work — ESM module
node -e "console.log(Object.keys(require('@immobiliarelabs/backstage-plugin-gitlab')))"

# ✅ Works — read type definitions directly
grep "^export" node_modules/@immobiliarelabs/backstage-plugin-gitlab/dist/index.d.ts
```

---

### ❌ Self-hosted GitLab pipelines not loading — CERTIFICATE error

**Symptom:**
```
Error: UNABLE_TO_VERIFY_LEAF_SIGNATURE
```

**Fix:** Mount your GitLab instance's CA certificate into the container:

```bash
docker run -d \
  --name backstage \
  --network backstage-network \
  -v /path/to/ca.crt:/usr/local/share/ca-certificates/gitlab-ca.crt:ro \
  -e NODE_EXTRA_CA_CERTS=/usr/local/share/ca-certificates/gitlab-ca.crt \
  -e GITLAB_SH_TOKEN=YOUR_SH_TOKEN \
  backstage:latest
```

---

### ❌ yarn add installs to wrong workspace

**Symptom:** Package installs but TypeScript can't find the import, or the import works in one package but not the other.

**Cause:** Using Yarn v1 syntax (`yarn add --cwd packages/app`) instead of Yarn Berry syntax.

**Fix:**
```bash
# ❌ Yarn v1 syntax — does not work with Yarn Berry
yarn add --cwd packages/app @immobiliarelabs/backstage-plugin-gitlab

# ✅ Yarn Berry syntax — correct
yarn --cwd packages/app add @immobiliarelabs/backstage-plugin-gitlab
```

---

### ❌ MR table shows but pipeline table is empty (or vice versa)

**Cause:** The token has access to MRs but not pipelines, or pipelines exist on protected branches with restricted visibility.

**Fix:** Ensure the service account tied to the token has at least **Reporter** role on the project in GitLab. Developer or Maintainer role is needed for protected branch pipeline visibility.

```
GitLab → Project → Manage → Members → Add your service account → Role: Reporter (minimum)
```

---

## 11. Quick Reference

### Yarn Install Commands

```bash
yarn --cwd packages/app add @immobiliarelabs/backstage-plugin-gitlab
yarn --cwd packages/backend add @immobiliarelabs/backstage-plugin-gitlab-backend
```

### index.ts Registration

```typescript
backend.add(import('@immobiliarelabs/backstage-plugin-gitlab-backend'));
```

### Correct Imports for EntityPage.tsx

```typescript
import {
  isGitlabAvailable,
  EntityGitlabPipelinesTable,
  EntityGitlabMergeRequestsTable,
  EntityGitlabLanguageCard,          // ← not EntityGitlabProjectLanguagesCard
} from '@immobiliarelabs/backstage-plugin-gitlab';
```

### catalog-info.yaml Annotation

```yaml
# GitLab SaaS
annotations:
  gitlab.com/project-slug: <group>/<subgroup>/<repo>

# Self-hosted
annotations:
  gitlab.internal/project-slug: <group>/<subgroup>/<repo>
```

### Check Exports of Installed Version

```bash
grep "^export" node_modules/@immobiliarelabs/backstage-plugin-gitlab/dist/index.d.ts
```

### Verify Token Access to Pipelines

```bash
curl -H "PRIVATE-TOKEN: $GITLAB_SAAS_TOKEN" \
  "https://gitlab.com/api/v4/projects/<group>%2F<repo>/pipelines?per_page=3" \
  | python3 -m json.tool
```

---

*Generated: March 2026 | Backstage v1.48.0 | @immobiliarelabs/backstage-plugin-gitlab | Node.js v24.13.1*
