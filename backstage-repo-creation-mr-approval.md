# Backstage – Repo Creation via Template with MR Approval

> GitLab Repo Creation · Backstage Scaffolder · MR-Gated Approval · Auto Provisioning  
> **Stack:** Backstage · GitLab · Custom Backend Plugin  
> **Version:** 1.0 | March 2026

---

## Table of Contents

1. [Overview & Architecture](#1-overview--architecture)
2. [Prerequisites](#2-prerequisites)
3. [File Structure](#3-file-structure)
4. [Step 1 – Scaffolder Template (template.yaml)](#4-step-1--scaffolder-template-templateyaml)
5. [Step 2 – Request Skeleton File](#5-step-2--request-skeleton-file)
6. [Step 3 – Backend Plugin (repoProvisioner.ts)](#6-step-3--backend-plugin-repoprovisionerts)
7. [Step 4 – Register Plugin in index.ts](#7-step-4--register-plugin-in-indexts)
8. [Step 5 – app-config.yaml Additions](#8-step-5--app-configyaml-additions)
9. [Step 6 – CODEOWNERS in platform/repo-requests](#9-step-6--codeowners-in-platformrepo-requests)
10. [Step 7 – GitLab Webhook Setup](#10-step-7--gitlab-webhook-setup)
11. [Step 8 – Generate Webhook Secret Token](#11-step-8--generate-webhook-secret-token)
12. [Step 9 – Register Template in Backstage](#12-step-9--register-template-in-backstage)
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
| `Template` added to `catalog.rules` allowlist | 🔲 Required — see Step 9 |
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

This file is what lands in `platform/repo-requests/requests/` as the MR diff. The platform team reviews exactly this content before approving. On merge, the webhook reads this file to know what to provision.

---

## 6. Step 3 – Backend Plugin (repoProvisioner.ts)

Create the file `packages/backend/src/plugins/repoProvisioner.ts`:

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
  // Get files changed in MR
  const res = await fetch(
    `${baseUrl}/api/v4/projects/${encodeURIComponent(requestsProjectId)}/merge_requests/${mrIid}/changes`,
    { headers: { 'PRIVATE-TOKEN': token } },
  );
  const data: { changes: Array<{ new_path: string }> } = await res.json();

  // Find the requests/*.yaml file
  const requestChange = data.changes.find(
    c => c.new_path.startsWith('requests/') && c.new_path.endsWith('.yaml'),
  );
  if (!requestChange) throw new Error('No request file found in MR changes');

  // Fetch file content from main branch (post-merge)
  const fileRes = await fetch(
    `${baseUrl}/api/v4/projects/${encodeURIComponent(requestsProjectId)}/repository/files/${encodeURIComponent(requestChange.new_path)}/raw?ref=main`,
    { headers: { 'PRIVATE-TOKEN': token } },
  );
  const content = await fileRes.text();

  // Extract YAML fields without adding a yaml dependency
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

        const gitlabToken = config.getString('gitlab.token');
        const gitlabBaseUrl =
          config.getOptionalString('gitlab.baseUrl') ?? 'https://gitlab.com';
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
        logger.info(
          'RepoProvisioner: Webhook listening at /api/repo-provisioner/webhook',
        );
      },
    });
  },
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

// existing registrations
backend.add(keycloakCustomModule);
backend.add(
  import('@backstage-community/plugin-catalog-backend-module-gitlab/alpha'),
);
backend.add(import('@immobiliarelabs/backstage-plugin-gitlab-backend'));

// ✅ new
backend.add(repoProvisionerPlugin);

backend.start();
```

---

## 8. Step 5 – app-config.yaml Additions

Add these two blocks to `app-config.yaml`:

```yaml
# app-config.yaml

gitlab:
  token: ${GITLAB_TOKEN}
  baseUrl: https://gitlab.com          # omit for SaaS default; required for self-hosted

repoProvisioner:
  webhookSecret: ${REPO_PROVISIONER_WEBHOOK_SECRET}
  requestsProjectId: cloudopsedge/repo-requests
```

> ℹ️ **INFO:** The `gitlab.token` here is read by the provisioner plugin directly from config. It must have `api` scope — not just `read_api` — since it creates repos and pushes files.

---

## 9. Step 6 – CODEOWNERS in platform/repo-requests

Add a `CODEOWNERS` file to the root of `platform/repo-requests`:

```
# CODEOWNERS
# All MRs in this repo require platform team approval before merge
* @cloudopsedge/platform-team
```

Then in GitLab, enable CODEOWNERS approval:

```
platform/repo-requests →
  Settings → Merge requests → Approvals →
    ✅ Enable "Require approval from code owners"
```

This enforces that no repo request MR can be merged without explicit platform team approval — it is the gate.

---

## 10. Step 7 – GitLab Webhook Setup

In GitLab go to `cloudopsedge/repo-requests` → **Settings → Webhooks → Add new webhook**:

| Field | Value |
|---|---|
| URL | `https://your-backstage-host/api/repo-provisioner/webhook` |
| Secret token | Value generated in Step 8 below |
| Trigger | ✅ Merge request events only — uncheck all others |
| SSL verification | ✅ Enabled |

> ⚠️ **WARNING:** The secret token field in GitLab is shown only at creation time. Store it immediately — if lost you must regenerate and update both GitLab and the Docker env var.

---

## 11. Step 8 – Generate Webhook Secret Token

### Generate the Secret

Run any of the following on your WSL Ubuntu terminal:

```bash
# Option A — openssl (recommended)
openssl rand -hex 32

# Option B — /dev/urandom
cat /dev/urandom | tr -dc 'a-zA-Z0-9' | head -c 40

# Option C — python3
python3 -c "import secrets; print(secrets.token_hex(32))"
```

Example output:
```
a3f8c2d1e9b74506f2a1c8e3d5b97042f1e6a8c3d2b54709e1f8a6c3d2b54709
```

This value goes in **two places** — GitLab webhook config (Step 7) and Docker env var (Step 10).

### How It Works

GitLab sends the secret in every webhook request as the `X-Gitlab-Token` header. The provisioner plugin checks this header on every incoming request:

```typescript
// Inside repoProvisioner.ts
const incoming = req.headers['x-gitlab-token'];
if (incoming !== webhookSecret) {
  res.status(401).json({ error: 'Unauthorized' });
  return;
}
```

Any request without the correct secret is rejected with `401` — prevents unauthorized repo creation if the webhook URL is discovered.

### Rotation

Rotate the secret if a team member with access leaves, or on your security policy schedule:

```bash
# 1. Generate new secret
openssl rand -hex 32

# 2. Update GitLab: repo-requests → Settings → Webhooks → Edit → new secret → Save
# 3. Restart Backstage with new env var value
# Note: do steps 2 and 3 quickly — there is a brief window where webhooks will fail
```

---

## 12. Step 9 – Register Template in Backstage

### Add Template Location to app-config.yaml

```yaml
# app-config.yaml
catalog:
  locations:
    # ... existing locations ...

    # ✅ GitLab repo creation template
    - type: url
      target: https://gitlab.com/cloudopsedge/backstage-templates/-/raw/main/templates/create-gitlab-repo/template.yaml
      rules:
        - allow: [Template]
```

### Add Template to catalog.rules Allowlist

```yaml
# app-config.yaml
catalog:
  rules:
    # ✅ Template added
    - allow: [Component, System, API, Resource, Location, Template]
```

> ⚠️ **WARNING:** Without `Template` in the allowlist, the template file is fetched but silently ignored — it will not appear in the Backstage Create page.

---

## 13. Step 10 – Rebuild & Deploy

```bash
# Step 1: Rebuild
yarn install --immutable
yarn tsc
yarn build:backend

# Step 2: Remove old container
docker rm -f backstage

# Step 3: Build new image
docker image build . -f packages/backend/Dockerfile --tag backstage:latest

# Step 4: Run with new env var
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
  -e REPO_PROVISIONER_WEBHOOK_SECRET=YOUR_GENERATED_SECRET \
  backstage:latest

# Step 5: Verify plugin loaded
docker logs backstage | grep -i "repo-provisioner"
# Expected:
# RepoProvisioner: Webhook listening at /api/repo-provisioner/webhook
```

---

## 14. Verification

### Verify Webhook Endpoint Is Reachable

```bash
# Should return 401 (no secret) — confirms endpoint exists
curl -sk -X POST \
  https://your-backstage-host/api/repo-provisioner/webhook \
  -H "Content-Type: application/json" \
  -d '{}' | python3 -m json.tool
# Expected: {"error": "Unauthorized"}

# With correct secret — returns 200 ignored (not a merge event payload)
curl -sk -X POST \
  https://your-backstage-host/api/repo-provisioner/webhook \
  -H "Content-Type: application/json" \
  -H "X-Gitlab-Token: YOUR_WEBHOOK_SECRET" \
  -d '{"object_kind": "ping"}' | python3 -m json.tool
# Expected: {"status": "ignored"}
```

### Verify Template Appears in Backstage

```
Backstage → Create → search "Create GitLab Repository"
```

If not visible, check:

```bash
# Confirm template entity was ingested
TOKEN=$(curl -sk "https://localhost/api/auth/guest/refresh" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['backstageIdentity']['token'])")

curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/catalog/entities?filter=kind=Template" \
  | python3 -m json.tool | grep '"name"'
```

### Test the GitLab Webhook from GitLab UI

```
platform/repo-requests →
  Settings → Webhooks → (your webhook) →
    Test → Merge request events
```

```bash
# Watch logs during test
docker logs -f backstage | grep -i "repo-provisioner"
# Expected:
# RepoProvisioner: Webhook received — ignored (not a merge event)
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
   → Scaffolder creates branch request/payment-service
   → Commits requests/payment-service.yaml to platform/repo-requests
   → Opens MR: "feat: create repo payment-service"

4. Developer sees output:
   → Link: "View Approval MR" → opens GitLab MR

5. Platform team receives MR notification in GitLab
   → Reviews the YAML diff (repo name, description, owner)
   → Approves (CODEOWNERS enforced)
   → Merges MR

6. Webhook fires → Backstage processes in ~3–5 seconds:
   → Creates https://gitlab.com/cloudopsedge/payment-service
   → Pushes README.md, catalog-info.yaml, mkdocs.yml
   → Posts comment on MR with repo URL

7. Platform team sees MR comment:
   ✅ Repository created successfully
   - Repo URL: https://gitlab.com/cloudopsedge/payment-service
   - Catalog: Will appear in Backstage on next discovery cycle (~30 min)
   - Owner: group:default/infra-team

8. ~30 minutes later:
   → GitLab discovery picks up catalog-info.yaml
   → Component "payment-service" appears in Backstage catalog
   → Docs tab ready (mkdocs.yml already in place)
   → CI/CD tab ready (gitlab annotation already in catalog-info.yaml)
```

---

## 16. Troubleshooting

---

### ❌ Template not appearing in Backstage Create page

**Symptom:** No template visible under Create.

**Diagnosis:**
```bash
# Check if template entity exists in catalog
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/catalog/entities?filter=kind=Template" \
  | python3 -m json.tool | grep '"name"'
```

**Fixes:**
- Confirm `Template` is in `catalog.rules` allowlist
- Confirm the `target` URL in `catalog.locations` points to the raw file URL (not the GitLab UI URL)
- Raw URL format: `https://gitlab.com/<group>/<repo>/-/raw/main/path/to/template.yaml`

---

### ❌ OwnerPicker shows no teams

**Symptom:** Owner Team dropdown is empty.

**Cause:** Keycloak sync has not run yet, or groups are not in the catalog.

**Fix:**
```bash
# Verify groups exist in catalog
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/catalog/entities?filter=kind=Group" \
  | python3 -m json.tool | grep '"name"'
```

If empty, wait for the Keycloak sync cycle or trigger a manual refresh by restarting the container.

---

### ❌ Webhook returns 401 Unauthorized

**Symptom:** GitLab shows webhook delivery failed with 401.

**Cause:** The `X-Gitlab-Token` sent by GitLab does not match `REPO_PROVISIONER_WEBHOOK_SECRET`.

**Fix:**
```bash
# Verify env var is set in container
docker exec -it backstage env | grep WEBHOOK_SECRET

# If missing — restart with correct env var
docker rm -f backstage && docker run -d ... \
  -e REPO_PROVISIONER_WEBHOOK_SECRET=YOUR_SECRET ...
```

---

### ❌ Webhook fires but repo not created — 500 error

**Symptom:** Logs show `RepoProvisioner: Failed to create repo: 403`.

**Cause:** GitLab token has `read_api` scope only — needs `api` scope to create repos.

**Fix:**
```
GitLab → User Settings → Access Tokens → YOUR_TOKEN → Edit
  → Scopes: ✅ api
  → Save
```

Then restart Backstage with the updated token.

---

### ❌ Repo created but skeleton files missing

**Symptom:** Repo exists in GitLab but is empty — no README, catalog-info.yaml, or mkdocs.yml.

**Diagnosis:**
```bash
docker logs backstage | grep -i "skeleton\|push\|commit"
```

**Cause:** The commits API call failed. Common reason — `initialize_with_readme: false` creates a truly empty repo with no default branch, and the first commit to `main` fails if the branch doesn't exist yet.

**Fix:** The `pushSkeletonFiles` function uses the commits API which creates the branch on first push if it doesn't exist. If it still fails, check that the token has `write_repository` scope in addition to `api`.

---

### ❌ Component does not appear in catalog after repo creation

**Symptom:** Repo exists with `catalog-info.yaml` but no component in Backstage.

**Cause:** GitLab discovery runs on a schedule (every 30 minutes in your config). The component will appear on the next scan cycle.

**Speed up for testing:**
```bash
# Restart container to trigger immediate discovery scan
docker restart backstage
```

Or temporarily reduce the discovery frequency during testing:
```yaml
schedule:
  frequency: { minutes: 1 }
```

---

### ❌ MR created but branch name conflicts

**Symptom:** Scaffolder fails with `branch already exists`.

**Cause:** A previous request for the same repo name was made and the branch `request/<repo-name>` still exists in `platform/repo-requests`.

**Fix:** Delete the old branch in GitLab manually, or add a timestamp suffix to the branch name in `template.yaml`:

```yaml
branchName: request/${{ parameters.repoName }}-${{ '' | now | truncate(10, true, '') }}
```

---

## 17. Quick Reference

### File Checklist

| File | Location | Purpose |
|---|---|---|
| `template.yaml` | `gitlab-templates-repo/templates/create-gitlab-repo/` | Backstage scaffolder template |
| `skeleton/requests/${{ values.repoName }}.yaml` | Same repo, skeleton folder | Request file committed to MR |
| `repoProvisioner.ts` | `packages/backend/src/plugins/` | Webhook listener + provisioner |
| `index.ts` | `packages/backend/src/` | Plugin registration |
| `CODEOWNERS` | `platform/repo-requests/` | Approval gate |

### Key URLs

| Resource | URL |
|---|---|
| Backstage Create page | `https://your-host/create` |
| Webhook endpoint | `https://your-host/api/repo-provisioner/webhook` |
| Approval MRs | `https://gitlab.com/cloudopsedge/repo-requests/-/merge_requests` |

### Generate Webhook Secret

```bash
openssl rand -hex 32
```

### Verify Webhook Endpoint

```bash
# Expect 401 — confirms endpoint is live
curl -sk -X POST https://your-host/api/repo-provisioner/webhook \
  -H "Content-Type: application/json" -d '{}'

# Expect 200 ignored — confirms secret is accepted
curl -sk -X POST https://your-host/api/repo-provisioner/webhook \
  -H "Content-Type: application/json" \
  -H "X-Gitlab-Token: YOUR_SECRET" \
  -d '{"object_kind":"ping"}'
```

### Docker Run Command

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

---

*Generated: March 2026 | Backstage v1.48.0 | Node.js v24.13.1 | GitLab SaaS*
