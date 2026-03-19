# Backstage – Repo Creation via Template with MR Approval

> GitLab Repo Creation · Backstage Scaffolder · MR-Gated Approval · Auto Provisioning  
> **Stack:** Backstage · GitLab · Custom Backend Plugin  
> **Version:** 1.1 | March 2026

---

## Table of Contents

1. [Overview & Architecture](#1-overview--architecture)
2. [Prerequisites](#2-prerequisites)
3. [File Structure](#3-file-structure)
4. [Step 1 – Scaffolder Template (template.yaml)](#4-step-1--scaffolder-template-templateyaml)
5. [Step 2 – Request Skeleton File](#5-step-2--request-skeleton-file)
6. [Step 3 – Backend Plugin (repoProvisioner.ts)](#6-step-3--backend-plugin-repoprovisionerts)
7. [Step 4 – Register Plugin in index.ts](#7-step-4--register-plugin-in-indexts)
8. [Step 5 – app-config.yaml Changes](#8-step-5--app-configyaml-changes)
9. [Step 6 – app-config.production.yaml Changes](#9-step-6--app-configproductionyaml-changes)
10. [Step 7 – CODEOWNERS in platform/repo-requests](#10-step-7--codeowners-in-platformrepo-requests)
11. [Step 8 – GitLab Webhook Setup](#11-step-8--gitlab-webhook-setup)
12. [Step 9 – Generate Webhook Secret Token](#12-step-9--generate-webhook-secret-token)
13. [Step 10 – Rebuild & Deploy](#13-step-10--rebuild--deploy)
14. [Verification](#14-verification)
15. [End-to-End Developer Flow](#15-end-to-end-developer-flow)
16. [Troubleshooting](#16-troubleshooting)
17. [Quick Reference](#17-quick-reference)

---

## 1. Overview & Architecture

This guide implements a GitLab repo creation flow where a developer fills out a Backstage Software Template form, which raises a GitLab MR for platform team approval. On MR merge, a custom backend plugin automatically creates the repo, pushes skeleton files, and the component registers itself in the Backstage catalog via GitLab discovery.

### Flow Diagram

```
Developer fills Backstage Template form
  (repo name · description · owner team)
        ↓
Backstage Scaffolder runs
  Step 1: Generates requests/<repo-name>.yaml
  Step 2: Pushes to platform/repo-requests as new branch
  Step 3: Opens MR with structured description
        ↓
Platform team reviews MR diff in GitLab
  (CODEOWNERS enforces approval requirement)
        ↓
Platform team merges MR
        ↓
GitLab Webhook → Backstage /api/repo-provisioner/webhook
        ↓
  → Reads request YAML from merged MR
  → Creates repo under cloudopsedge/<repo-name>
  → Pushes README.md + catalog-info.yaml + mkdocs.yml
  → Comments result back on MR
        ↓
GitLab Discovery picks up catalog-info.yaml (~30 min)
        ↓
Component appears in Backstage catalog
```

### What Gets Created Automatically

| Artifact | Location | Purpose |
|---|---|---|
| `requests/<repo-name>.yaml` | `platform/repo-requests` | Approval artifact — platform team reviews this |
| `README.md` | New repo root | Populated with name and description |
| `catalog-info.yaml` | New repo root | Registers component in Backstage catalog |
| `mkdocs.yml` | New repo root | TechDocs ready from day one |

### Token Scope Requirement

| Token | Scope Required | Used For |
|---|---|---|
| `GITLAB_TOKEN` | `api` (not just `read_api`) | Creating repos, pushing files, posting MR comments |
| `REPO_PROVISIONER_WEBHOOK_SECRET` | N/A — random string | Verifying webhook authenticity |

> ⚠️ **WARNING:** The existing `GITLAB_TOKEN` likely has `read_api` scope only. Update it to `api` scope in GitLab before proceeding, or create a separate provisioner token.

---

## 2. Prerequisites

| Requirement | Status |
|---|---|
| Backstage running on Docker with PostgreSQL | ✅ Already done |
| GitLab integration configured in `app-config.yaml` | ✅ Already done |
| `platform/repo-requests` repo exists in GitLab | ✅ Already created |
| GitLab catalog discovery running | ✅ Already done |
| TechDocs configured | ✅ Already done |
| `Template` added to `catalog.rules` allowlist | 🔲 Required — see Step 5 |
| GitLab token updated to `api` scope | 🔲 Required — see Section 1 |

---

## 3. File Structure

```
backstage/
  packages/backend/src/
    ├── index.ts                              ← add plugin registration
    └── plugins/
          └── repoProvisioner.ts              ← new custom backend plugin

gitlab-templates-repo/
  templates/
    └── create-gitlab-repo/
          ├── template.yaml                   ← scaffolder template (form + steps)
          └── skeleton/
                └── requests/
                      └── ${{ values.repoName }}.yaml   ← request file committed to MR

platform/repo-requests/                       ← already exists
  └── CODEOWNERS                              ← approval gate
```

---

## 4. Step 1 – Scaffolder Template (template.yaml)

Create this file at `templates/create-gitlab-repo/template.yaml` in your GitLab templates repo:

```yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: create-gitlab-repo
  title: Create GitLab Repository
  description: Request a new GitLab repository. Creates an MR for platform team approval.
  tags:
    - gitlab
    - repository
    - recommended
spec:
  owner: group:default/platform-common-team
  type: service

  # ── Input form ──────────────────────────────────────────────────────────────
  parameters:
    - title: Repository Details
      required:
        - repoName
        - description
        - owner
      properties:
        repoName:
          title: Repository Name
          type: string
          description: Name of the new GitLab repo (lowercase, hyphens only)
          pattern: '^[a-z0-9-]+$'
          ui:autofocus: true

        description:
          title: Description
          type: string
          description: Short description of what this repo is for
          ui:widget: textarea
          ui:options:
            rows: 3

        owner:
          title: Owner Team
          type: string
          description: Team that will own this repository
          ui:field: OwnerPicker
          ui:options:
            catalogFilter:
              kind: Group

  # ── Scaffolder steps ─────────────────────────────────────────────────────────
  steps:
    - id: fetch-template
      name: Fetch Request Template
      action: fetch:template
      input:
        url: ./skeleton
        values:
          repoName: ${{ parameters.repoName }}
          description: ${{ parameters.description }}
          owner: ${{ parameters.owner }}
          requestedBy: ${{ user.entity.metadata.name }}
          namespace: cloudopsedge

    - id: create-mr
      name: Open Approval MR
      action: publish:gitlab:merge-request
      input:
        repoUrl: gitlab.com?owner=cloudopsedge&repo=repo-requests
        title: "feat: create repo ${{ parameters.repoName }}"
        description: |
          ## New Repository Request

          | Field | Value |
          |---|---|
          | **Repo Name** | `${{ parameters.repoName }}` |
          | **Description** | ${{ parameters.description }} |
          | **Owner Team** | ${{ parameters.owner }} |
          | **Requested By** | ${{ user.entity.metadata.name }} |

          ---
          ✅ Merge this MR to trigger automatic repo creation.
          ❌ Close without merging to reject the request.
        branchName: request/${{ parameters.repoName }}
        commitMessage: "feat: add repo request for ${{ parameters.repoName }}"
        targetBranchName: main

  # ── Output ───────────────────────────────────────────────────────────────────
  output:
    links:
      - title: View Approval MR
        url: ${{ steps['create-mr'].output.mergeRequestUrl }}
        icon: gitlab
    text:
      - title: Next Steps
        content: |
          Your repository request has been submitted.
          The platform team will review and approve the MR.
          Once approved, your repo will be created at:
          `https://gitlab.com/cloudopsedge/${{ parameters.repoName }}`
```

### Form Fields Explained

| Field | Type | Validation | Notes |
|---|---|---|---|
| `repoName` | string | `^[a-z0-9-]+$` | Lowercase and hyphens only — maps directly to GitLab repo path |
| `description` | string | Required | Textarea — populates README and catalog-info.yaml |
| `owner` | OwnerPicker | Required | Dropdown from live Backstage catalog groups — no free text |

---

## 5. Step 2 – Request Skeleton File

Create this file at `templates/create-gitlab-repo/skeleton/requests/${{ values.repoName }}.yaml`:

```yaml
apiVersion: platform.io/v1
kind: RepoRequest
metadata:
  name: ${{ values.repoName }}
  requestedBy: ${{ values.requestedBy }}
spec:
  repoName: ${{ values.repoName }}
  description: ${{ values.description }}
  ownerTeam: ${{ values.owner }}
  namespace: ${{ values.namespace }}
  type: service
  visibility: private
  defaultBranch: main
  status: pending
```

This is what lands in `platform/repo-requests/requests/` as the MR diff. The platform team reviews exactly this content. On merge, the webhook reads this file to know what to provision.

---

## 6. Step 3 – Backend Plugin (repoProvisioner.ts)

Create `packages/backend/src/plugins/repoProvisioner.ts`:

```typescript
import {
  createBackendPlugin,
  coreServices,
} from '@backstage/backend-plugin-api';
import { Router } from 'express';

// ── Skeleton file generators ──────────────────────────────────────────────────

function generateReadme(repoName: string, description: string): string {
  return `# ${repoName}

${description}

## Overview

Add your service overview here.

## Getting Started

Add setup instructions here.
`;
}

function generateCatalogInfo(
  repoName: string,
  description: string,
  ownerTeam: string,
): string {
  const owner = ownerTeam.startsWith('group:')
    ? ownerTeam
    : `group:default/${ownerTeam}`;

  return `apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: ${repoName}
  description: "${description}"
  annotations:
    gitlab.com/project-slug: cloudopsedge/${repoName}
    backstage.io/techdocs-ref: dir:.
spec:
  type: service
  lifecycle: experimental
  owner: ${owner}
`;
}

function generateMkdocs(repoName: string, description: string): string {
  return `site_name: '${repoName}'
site_description: '${description}'
docs_dir: .
nav:
  - Home: README.md
plugins:
  - techdocs-core
`;
}

// ── GitLab API helpers ─────────────────────────────────────────────────────────

async function gitlabPost(
  path: string,
  body: Record<string, unknown>,
  token: string,
  baseUrl: string,
): Promise<Response> {
  return fetch(`${baseUrl}/api/v4${path}`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'PRIVATE-TOKEN': token,
    },
    body: JSON.stringify(body),
  });
}

async function resolveNamespaceId(
  namespace: string,
  token: string,
  baseUrl: string,
): Promise<number> {
  const res = await fetch(
    `${baseUrl}/api/v4/namespaces?search=${namespace}`,
    { headers: { 'PRIVATE-TOKEN': token } },
  );
  const namespaces: Array<{ id: number; path: string }> = await res.json();
  const found = namespaces.find(n => n.path === namespace);
  if (!found) throw new Error(`Namespace not found: ${namespace}`);
  return found.id;
}

async function createRepo(
  repoName: string,
  description: string,
  namespace: string,
  token: string,
  baseUrl: string,
): Promise<{ id: number; web_url: string }> {
  const namespaceId = await resolveNamespaceId(namespace, token, baseUrl);

  const res = await gitlabPost(
    '/projects',
    {
      name: repoName,
      path: repoName,
      description,
      namespace_id: namespaceId,
      visibility: 'private',
      initialize_with_readme: false,
      default_branch: 'main',
    },
    token,
    baseUrl,
  );

  if (!res.ok) {
    const err = await res.text();
    throw new Error(`Failed to create repo: ${res.status} ${err}`);
  }

  return res.json();
}

async function pushSkeletonFiles(
  projectId: number,
  files: Array<{ filePath: string; content: string }>,
  token: string,
  baseUrl: string,
): Promise<void> {
  const res = await gitlabPost(
    `/projects/${projectId}/repository/commits`,
    {
      branch: 'main',
      commit_message: 'chore: initial scaffold by Backstage',
      actions: files.map(f => ({
        action: 'create',
        file_path: f.filePath,
        content: Buffer.from(f.content).toString('base64'),
        encoding: 'base64',
      })),
    },
    token,
    baseUrl,
  );

  if (!res.ok) {
    const err = await res.text();
    throw new Error(`Failed to push skeleton files: ${res.status} ${err}`);
  }
}

async function postMrComment(
  mrIid: number,
  message: string,
  token: string,
  baseUrl: string,
  requestsProjectId: string,
): Promise<void> {
  await gitlabPost(
    `/projects/${encodeURIComponent(requestsProjectId)}/merge_requests/${mrIid}/notes`,
    { body: message },
    token,
    baseUrl,
  );
}

async function readRequestFile(
  mrIid: number,
  token: string,
  baseUrl: string,
  requestsProjectId: string,
): Promise<Record<string, string>> {
  const res = await fetch(
    `${baseUrl}/api/v4/projects/${encodeURIComponent(requestsProjectId)}/merge_requests/${mrIid}/changes`,
    { headers: { 'PRIVATE-TOKEN': token } },
  );
  const data: { changes: Array<{ new_path: string }> } = await res.json();

  const requestChange = data.changes.find(
    c => c.new_path.startsWith('requests/') && c.new_path.endsWith('.yaml'),
  );
  if (!requestChange) throw new Error('No request file found in MR changes');

  const fileRes = await fetch(
    `${baseUrl}/api/v4/projects/${encodeURIComponent(requestsProjectId)}/repository/files/${encodeURIComponent(requestChange.new_path)}/raw?ref=main`,
    { headers: { 'PRIVATE-TOKEN': token } },
  );
  const content = await fileRes.text();

  const extract = (key: string): string => {
    const match = content.match(new RegExp(`${key}:\\s*(.+)`));
    return match ? match[1].trim().replace(/^["']|["']$/g, '') : '';
  };

  return {
    repoName: extract('repoName'),
    description: extract('description'),
    ownerTeam: extract('ownerTeam'),
    namespace: extract('namespace'),
  };
}

// ── Plugin definition ──────────────────────────────────────────────────────────

export const repoProvisionerPlugin = createBackendPlugin({
  pluginId: 'repo-provisioner',
  register(env) {
    env.registerInit({
      deps: {
        logger: coreServices.logger,
        config: coreServices.rootConfig,
        httpRouter: coreServices.httpRouter,
      },
      async init({ logger, config, httpRouter }) {
        const router = Router();
        router.use(require('express').json());

        // ── Read GitLab token via getConfigArray — bracket notation not supported
        const gitlabConfigs = config.getConfigArray('integrations.gitlab');
        const gitlabToken = gitlabConfigs[0].getString('token');
        const gitlabBaseUrl =
          gitlabConfigs[0].getOptionalString('baseUrl') ?? 'https://gitlab.com';

        const webhookSecret = config.getOptionalString(
          'repoProvisioner.webhookSecret',
        );
        const requestsProjectId =
          config.getOptionalString('repoProvisioner.requestsProjectId') ??
          'cloudopsedge/repo-requests';

        router.post('/webhook', async (req, res) => {
          try {
            // ── 1. Verify webhook secret ────────────────────────────────────
            if (webhookSecret) {
              const incoming = req.headers['x-gitlab-token'];
              if (incoming !== webhookSecret) {
                logger.warn('RepoProvisioner: Invalid webhook secret — rejected');
                res.status(401).json({ error: 'Unauthorized' });
                return;
              }
            }

            const event = req.body;

            // ── 2. Only handle MR merge events ──────────────────────────────
            if (
              event.object_kind !== 'merge_request' ||
              event.object_attributes?.state !== 'merged' ||
              event.project?.path_with_namespace !== requestsProjectId
            ) {
              res.status(200).json({ status: 'ignored' });
              return;
            }

            const mrIid: number = event.object_attributes.iid;
            logger.info(`RepoProvisioner: Processing merged MR !${mrIid}`);

            // ── 3. Read request YAML from merged MR ─────────────────────────
            const params = await readRequestFile(
              mrIid,
              gitlabToken,
              gitlabBaseUrl,
              requestsProjectId,
            );

            logger.info(
              `RepoProvisioner: Creating repo ${params.namespace}/${params.repoName}`,
            );

            // ── 4. Create the GitLab repo ────────────────────────────────────
            const newRepo = await createRepo(
              params.repoName,
              params.description,
              params.namespace,
              gitlabToken,
              gitlabBaseUrl,
            );

            // ── 5. Push skeleton files ───────────────────────────────────────
            await pushSkeletonFiles(
              newRepo.id,
              [
                {
                  filePath: 'README.md',
                  content: generateReadme(params.repoName, params.description),
                },
                {
                  filePath: 'catalog-info.yaml',
                  content: generateCatalogInfo(
                    params.repoName,
                    params.description,
                    params.ownerTeam,
                  ),
                },
                {
                  filePath: 'mkdocs.yml',
                  content: generateMkdocs(params.repoName, params.description),
                },
              ],
              gitlabToken,
              gitlabBaseUrl,
            );

            logger.info(
              `RepoProvisioner: Successfully created ${newRepo.web_url}`,
            );

            // ── 6. Comment result back on MR ─────────────────────────────────
            await postMrComment(
              mrIid,
              `✅ **Repository created successfully**\n\n` +
                `- **Repo URL:** ${newRepo.web_url}\n` +
                `- **Catalog:** Will appear in Backstage on next discovery cycle (~30 min)\n` +
                `- **Owner:** ${params.ownerTeam}`,
              gitlabToken,
              gitlabBaseUrl,
              requestsProjectId,
            );

            res.status(200).json({
              status: 'created',
              repoUrl: newRepo.web_url,
            });
          } catch (err: unknown) {
            const message =
              err instanceof Error ? err.message : 'Unknown error';
            logger.error(`RepoProvisioner: ${message}`);
            res.status(500).json({ error: message });
          }
        });

        httpRouter.use(router);

        // ── Mark /webhook as unauthenticated — required so Backstage middleware
        // does not block GitLab webhook calls before they reach the handler.
        // Plugin-level security is handled by X-Gitlab-Token check above.
        httpRouter.addAuthPolicy({
          path: '/webhook',
          allow: 'unauthenticated',
        });

        logger.info(
          'RepoProvisioner: Webhook listening at /api/repo-provisioner/webhook',
        );
      },
    });
  },
});
```

### Two Critical Implementation Notes

**1 — Config array syntax**

Backstage `ConfigReader` does NOT support bracket notation for arrays:

```typescript
// ❌ Wrong — causes "Invalid config key" TypeError at startup
const token = config.getString('integrations.gitlab[0].token');

// ✅ Correct — use getConfigArray() then index the result
const gitlabConfigs = config.getConfigArray('integrations.gitlab');
const token = gitlabConfigs[0].getString('token');
```

**2 — Webhook auth policy**

Without `addAuthPolicy`, Backstage's own middleware intercepts the webhook and returns `AuthenticationError: Missing credentials` before the handler runs:

```typescript
// ❌ Missing addAuthPolicy — Backstage returns 401 on every GitLab webhook call
httpRouter.use(router);

// ✅ Correct — allow GitLab to call without a Backstage token
httpRouter.use(router);
httpRouter.addAuthPolicy({
  path: '/webhook',
  allow: 'unauthenticated',
});
```

---

## 7. Step 4 – Register Plugin in index.ts

```typescript
// packages/backend/src/index.ts

import { createBackend } from '@backstage/backend-defaults';
import { keycloakCustomModule } from './keycloakProvider';
import { repoProvisionerPlugin } from './plugins/repoProvisioner';  // ← add

const backend = createBackend();

backend.add(keycloakCustomModule);
backend.add(
  import('@backstage-community/plugin-catalog-backend-module-gitlab/alpha'),
);
backend.add(import('@immobiliarelabs/backstage-plugin-gitlab-backend'));

// ✅ Add repo provisioner
backend.add(repoProvisionerPlugin);

backend.start();
```

---

## 8. Step 5 – app-config.yaml Changes

Two changes required:

**Change 1 — `repoProvisioner` must be a top-level key**

```yaml
# app-config.yaml

integrations:
  gitlab:
    - host: gitlab.com
      apiBaseUrl: https://gitlab.com/api/v4
      token: ${GITLAB_TOKEN}
  # ❌ Do NOT put repoProvisioner here under integrations

# ✅ Top-level key — not nested under integrations
repoProvisioner:
  webhookSecret: ${REPO_PROVISIONER_WEBHOOK_SECRET}
  requestsProjectId: cloudopsedge/repo-requests
```

**Change 2 — Add template URL to `catalog.locations`**

```yaml
catalog:
  rules:
    # ✅ Template must be in the allowlist
    - allow: [Component, System, API, Resource, Location, Template, Domain, Group, User]

  locations:
    # ✅ Add this — use /-/raw/main/ not /-/blob/main/
    - type: url
      target: https://gitlab.com/cloudopsedge/backstage-templates/-/raw/main/templates/create-gitlab-repo/template.yaml
      rules:
        - allow: [Template]

    # keep existing
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
```

---

## 9. Step 6 – app-config.production.yaml Changes

> ⚠️ **CRITICAL:** In production mode, `app-config.production.yaml` overrides `catalog.locations` completely. Any location only in `app-config.yaml` will NOT be loaded at runtime. All three items below must be in the production config.

**Add to app-config.production.yaml:**

```yaml
# app-config.production.yaml

# ── 1. GitLab integration (token must resolve in production) ─────────────────
integrations:
  gitlab:
    - host: gitlab.com
      apiBaseUrl: https://gitlab.com/api/v4
      token: ${GITLAB_TOKEN}

# ── 2. repoProvisioner top-level block ───────────────────────────────────────
repoProvisioner:
  webhookSecret: ${REPO_PROVISIONER_WEBHOOK_SECRET}
  requestsProjectId: cloudopsedge/repo-requests

# ── 3. Template URL in catalog.locations ─────────────────────────────────────
catalog:
  locations:
    # ✅ Template URL — must be here not just in app-config.yaml
    - type: url
      target: https://gitlab.com/cloudopsedge/backstage-templates/-/raw/main/templates/create-gitlab-repo/template.yaml
      rules:
        - allow: [Template]

    # keep existing
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
```

---

## 10. Step 7 – CODEOWNERS in platform/repo-requests

```
# CODEOWNERS
* @cloudopsedge/platform-team
```

Enable in GitLab:
```
platform/repo-requests →
  Settings → Merge requests → Approvals →
    ✅ Enable "Require approval from code owners"
```

---

## 11. Step 8 – GitLab Webhook Setup

Go to `cloudopsedge/repo-requests` → **Settings → Webhooks → Add new webhook**:

| Field | Value |
|---|---|
| URL | `https://your-backstage-host/api/repo-provisioner/webhook` |
| Secret token | Value generated in Step 9 |
| Trigger | ✅ Merge request events only — uncheck everything else |
| SSL verification | ✅ Enabled |

---

## 12. Step 9 – Generate Webhook Secret Token

```bash
# Recommended
openssl rand -hex 32

# Alternative
python3 -c "import secrets; print(secrets.token_hex(32))"
```

Use the same value in two places:
- GitLab webhook → Secret token field
- Docker run → `-e REPO_PROVISIONER_WEBHOOK_SECRET=<value>`

**Rotation:**
```bash
# 1. Generate new secret
openssl rand -hex 32
# 2. Update GitLab: repo-requests → Settings → Webhooks → Edit → new secret
# 3. Restart Backstage with new env var
# Do steps 2 and 3 quickly to minimise the gap
```

---

## 13. Step 10 – Rebuild & Deploy

```bash
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
  -e GITLAB_TOKEN=YOUR_GITLAB_TOKEN \
  -e REPO_PROVISIONER_WEBHOOK_SECRET=YOUR_WEBHOOK_SECRET \
  backstage:latest

# Verify plugin started
docker logs backstage | grep -i "repo-provisioner"
# Expected:
# RepoProvisioner: Webhook listening at /api/repo-provisioner/webhook
```

---

## 14. Verification

### Verify Plugin Started

```bash
docker logs backstage | grep -i "repo-provisioner"
# Expected:
# RepoProvisioner: Webhook listening at /api/repo-provisioner/webhook
```

### Verify Webhook Endpoint Security

```bash
# No secret — expect {"error": "Unauthorized"} from plugin
curl -sk -X POST https://localhost/api/repo-provisioner/webhook \
  -H "Content-Type: application/json" \
  -d '{}' | python3 -m json.tool

# With correct secret — expect {"status": "ignored"}
curl -sk -X POST https://localhost/api/repo-provisioner/webhook \
  -H "Content-Type: application/json" \
  -H "X-Gitlab-Token: YOUR_WEBHOOK_SECRET" \
  -d '{"object_kind": "ping"}' | python3 -m json.tool
```

### Verify Template Is in Catalog

```bash
TOKEN=$(curl -sk "https://localhost/api/auth/guest/refresh" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['backstageIdentity']['token'])")

curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/catalog/entities?filter=kind=Template" \
  | python3 -m json.tool | grep '"name"'
# Expected: "name": "create-gitlab-repo"
```

### Test Webhook from GitLab UI

```
cloudopsedge/repo-requests →
  Settings → Webhooks → (your webhook) → Test → Merge request events
```

```bash
docker logs -f backstage | grep -i "repo-provisioner"
# Expected: {"status": "ignored"}
```

---

## 15. End-to-End Developer Flow

```
1. Developer opens Backstage
   → Create → "Create GitLab Repository"

2. Fills 3 fields:
   → Repo Name:    payment-service
   → Description:  Handles payment processing and refunds
   → Owner Team:   group:default/infra-team  (picked from dropdown)

3. Clicks "Create Repository"
   → Scaffolder creates branch: request/payment-service
   → Commits: requests/payment-service.yaml to platform/repo-requests
   → Opens MR: "feat: create repo payment-service"

4. Developer sees output page:
   → Link: "View Approval MR" → opens GitLab MR

5. Platform team receives MR notification
   → Reviews YAML diff (repo name, description, owner)
   → Approves (CODEOWNERS enforced)
   → Merges MR

6. Webhook fires → Backstage processes in ~3–5 seconds:
   → Creates https://gitlab.com/cloudopsedge/payment-service
   → Pushes README.md, catalog-info.yaml, mkdocs.yml
   → Posts comment on MR with repo URL

7. MR gets auto-comment:
   ✅ Repository created successfully
   - Repo URL: https://gitlab.com/cloudopsedge/payment-service
   - Catalog: Will appear in Backstage on next discovery cycle (~30 min)
   - Owner: group:default/infra-team

8. ~30 minutes later (or on docker restart):
   → GitLab discovery picks up catalog-info.yaml
   → Component appears in Backstage catalog
   → Docs tab ready (mkdocs.yml in place)
   → CI/CD tab ready (gitlab annotation in catalog-info.yaml)
```

---

## 16. Troubleshooting

---

### ❌ Missing required config value at 'gitlab.token'

**Symptom:**
```
Error: Missing required config value at 'gitlab.token' in 'app-config.production.yaml'
```

**Cause:** Two problems — the plugin was reading `gitlab.token` (non-existent path) and the GitLab integration block was missing from `app-config.production.yaml`.

**Fix 1 — Use correct config path in repoProvisioner.ts:**
```typescript
// ❌ Wrong
const gitlabToken = config.getString('gitlab.token');

// ✅ Correct
const gitlabConfigs = config.getConfigArray('integrations.gitlab');
const gitlabToken = gitlabConfigs[0].getString('token');
```

**Fix 2 — Add to app-config.production.yaml:**
```yaml
integrations:
  gitlab:
    - host: gitlab.com
      apiBaseUrl: https://gitlab.com/api/v4
      token: ${GITLAB_TOKEN}
```

---

### ❌ Invalid config key 'integrations.gitlab[0].token'

**Symptom:**
```
TypeError: Invalid config key 'integrations.gitlab[0].token'
```

**Cause:** Backstage `ConfigReader` does not support array bracket notation. This is a hard limitation of the config system.

**Fix:**
```typescript
// ❌ Wrong — bracket notation not supported
const token = config.getString('integrations.gitlab[0].token');

// ✅ Correct
const gitlabConfigs = config.getConfigArray('integrations.gitlab');
const token = gitlabConfigs[0].getString('token');
```

---

### ❌ Webhook returns AuthenticationError: Missing credentials

**Symptom:**
```json
{
  "error": {
    "name": "AuthenticationError",
    "message": "Missing credentials"
  },
  "response": { "statusCode": 401 }
}
```

**Cause:** Backstage's own authentication middleware is blocking the request before it reaches the plugin. The `/webhook` route was not marked as unauthenticated.

**Fix — add `addAuthPolicy` in repoProvisioner.ts:**
```typescript
httpRouter.use(router);

// ✅ Required — without this every webhook call returns AuthenticationError
httpRouter.addAuthPolicy({
  path: '/webhook',
  allow: 'unauthenticated',
});
```

---

### ❌ repoProvisioner config not found

**Symptom:**
```
Error: Missing required config value at 'repoProvisioner.webhookSecret'
```

**Cause:** The `repoProvisioner` block was nested under `integrations` instead of being a top-level key.

**Fix:**
```yaml
# ❌ Wrong — nested under integrations
integrations:
  gitlab:
    - host: gitlab.com
  repoProvisioner:
    webhookSecret: ...

# ✅ Correct — top-level key
integrations:
  gitlab:
    - host: gitlab.com

repoProvisioner:
  webhookSecret: ${REPO_PROVISIONER_WEBHOOK_SECRET}
  requestsProjectId: cloudopsedge/repo-requests
```

---

### ❌ Template not appearing in Backstage Create page

**Symptom:** No template visible under Create.

**Diagnosis:**
```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/catalog/entities?filter=kind=Template" \
  | python3 -m json.tool | grep '"name"'
```

**Common causes:**

| Cause | Fix |
|---|---|
| Template URL only in `app-config.yaml`, not `app-config.production.yaml` | Add URL to `catalog.locations` in production config |
| URL uses `/-/blob/main/` | Change to `/-/raw/main/` |
| `Template` not in `catalog.rules` | Add `Template` to the allowlist |
| Template YAML has syntax errors | Check logs for catalog processing errors |

**Correct URL format:**
```
✅ https://gitlab.com/group/repo/-/raw/main/path/to/template.yaml
❌ https://gitlab.com/group/repo/-/blob/main/path/to/template.yaml
```

---

### ❌ Webhook returns 401 — wrong secret

**Symptom:** GitLab webhook delivery shows 401, logs show `Invalid webhook secret — rejected`

**Fix:**
```bash
# Verify env var is present in container
docker exec -it backstage env | grep WEBHOOK_SECRET

# Test with correct secret
curl -sk -X POST https://localhost/api/repo-provisioner/webhook \
  -H "Content-Type: application/json" \
  -H "X-Gitlab-Token: YOUR_SECRET" \
  -d '{"object_kind":"ping"}' | python3 -m json.tool
# Expected: {"status": "ignored"}
```

---

### ❌ Webhook fires but repo not created — 403

**Symptom:**
```
RepoProvisioner: Failed to create repo: 403
```

**Cause:** `GITLAB_TOKEN` has `read_api` scope only.

**Fix:**
```
GitLab → User Settings → Access Tokens → YOUR_TOKEN → Edit
  → Scopes: ✅ api
  → Save → copy new token value → restart Backstage with new token
```

---

### ❌ Repo created but no skeleton files

**Symptom:** Repo exists in GitLab but is empty.

**Diagnosis:**
```bash
docker logs backstage | grep -i "skeleton\|push\|commit\|error" | tail -20
```

**Fix:** Ensure token has both `api` and `write_repository` scopes. The commits API creates the `main` branch on first push — if it still fails, the exact API error response will be in the logs.

---

### ❌ OwnerPicker dropdown is empty

**Cause:** Keycloak groups not yet synced to catalog.

**Fix:**
```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/catalog/entities?filter=kind=Group" \
  | python3 -m json.tool | grep '"name"'
```

If empty, restart the container to trigger an immediate Keycloak sync.

---

### ❌ MR branch already exists

**Symptom:**
```
Branch 'request/my-repo' already exists
```

**Fix — Option A:** Delete the old branch in GitLab manually.

**Fix — Option B:** Add uniqueness to branch name in `template.yaml`:
```yaml
branchName: request/${{ parameters.repoName }}-${{ '' | now | truncate(10, true, '') }}
```

---

### ❌ Component not in catalog after repo creation

**Cause:** GitLab discovery runs on schedule — not immediately.

**Speed up:**
```bash
docker restart backstage   # triggers immediate scan on startup
```

---

## 17. Quick Reference

### Critical Config Rules

| Rule | Detail |
|---|---|
| `repoProvisioner` placement | Top-level key in both config files — never nested under `integrations` |
| `integrations.gitlab` in production | Must be in `app-config.production.yaml` — not just `app-config.yaml` |
| Template URL in production | Must be in `app-config.production.yaml` `catalog.locations` |
| Config array reads | Use `getConfigArray()` — bracket notation `[0]` not supported |
| Webhook auth policy | Must call `addAuthPolicy({ path: '/webhook', allow: 'unauthenticated' })` |
| Template URL format | Use `/-/raw/main/` not `/-/blob/main/` |

### Config Read Pattern

```typescript
// ✅ Only correct way to read GitLab token from Backstage config
const gitlabConfigs = config.getConfigArray('integrations.gitlab');
const gitlabToken = gitlabConfigs[0].getString('token');
const gitlabBaseUrl = gitlabConfigs[0].getOptionalString('baseUrl') ?? 'https://gitlab.com';
```

### Webhook Auth Policy

```typescript
// ✅ Must be added — prevents Backstage from blocking GitLab webhook calls
httpRouter.addAuthPolicy({
  path: '/webhook',
  allow: 'unauthenticated',
});
```

### Generate Webhook Secret

```bash
openssl rand -hex 32
```

### Verify Webhook

```bash
# Expect {"error": "Unauthorized"}
curl -sk -X POST https://localhost/api/repo-provisioner/webhook \
  -H "Content-Type: application/json" -d '{}'

# Expect {"status": "ignored"}
curl -sk -X POST https://localhost/api/repo-provisioner/webhook \
  -H "Content-Type: application/json" \
  -H "X-Gitlab-Token: YOUR_SECRET" \
  -d '{"object_kind":"ping"}'
```

### Docker Run

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
  -e GITLAB_TOKEN=YOUR_GITLAB_TOKEN \
  -e REPO_PROVISIONER_WEBHOOK_SECRET=YOUR_WEBHOOK_SECRET \
  backstage:latest
```

### Watch Provisioner Logs

```bash
docker logs -f backstage | grep -i "repo-provisioner"
```

### File Checklist

| File | Location | Purpose |
|---|---|---|
| `template.yaml` | `gitlab-templates-repo/templates/create-gitlab-repo/` | Scaffolder template |
| `skeleton/requests/${{ values.repoName }}.yaml` | Same repo, skeleton folder | Request file committed to MR |
| `repoProvisioner.ts` | `packages/backend/src/plugins/` | Webhook listener + provisioner |
| `index.ts` | `packages/backend/src/` | Plugin registration |
| `CODEOWNERS` | `platform/repo-requests/` | Approval gate |

---

*Generated: March 2026 | Backstage v1.48.0 | Node.js v24.13.1 | GitLab SaaS | Version 1.1 — includes all troubleshooting from implementation session*
