# LaunchPad – Repo Creation Pipeline Dashboard

> GitLab Repo Request Tracking · ECharts Dashboard · PostgreSQL Persistence  
> **Stack:** Backstage · GitLab · Custom Backend Plugin · ECharts  
> **Version:** 1.0 | March 2026

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Database Schema](#3-database-schema)
4. [File Structure](#4-file-structure)
5. [Backend – launchpad.ts](#5-backend--launchpadts)
6. [Backend – repoProvisioner.ts Changes](#6-backend--repoprovisionerts-changes)
7. [Register Plugins in index.ts](#7-register-plugins-in-indexts)
8. [Frontend – Install ECharts](#8-frontend--install-echarts)
9. [Frontend – api.ts](#9-frontend--apits)
10. [Frontend – KpiCards.tsx](#10-frontend--kpicardstsx)
11. [Frontend – FunnelChart.tsx](#11-frontend--funnelcharttsx)
12. [Frontend – StageDurationChart.tsx](#12-frontend--stagedurationcharttsx)
13. [Frontend – TimelineChart.tsx](#13-frontend--timelinecharttsx)
14. [Frontend – TeamChart.tsx](#14-frontend--teamcharttsx)
15. [Frontend – LaunchPadPage.tsx](#15-frontend--launchpadpagetsx)
16. [Wire into App.tsx and Root.tsx](#16-wire-into-apptsx-and-roottsx)
17. [Rebuild & Deploy](#17-rebuild--deploy)
18. [Verification](#18-verification)
19. [Troubleshooting](#19-troubleshooting)
20. [Quick Reference](#20-quick-reference)

---

## 1. Overview

LaunchPad is a Backstage dashboard that tracks the full lifecycle of GitLab repo creation requests — from when a developer submits a Backstage template through to when the repo is provisioned and registered in the catalog. It gives the platform team visibility into where requests are delayed and how long each stage takes.

### What It Tracks

```
Developer submits template
        ↓ submitted_at recorded (status: pending)
MR opened in platform/repo-requests
        ↓
Platform team reviews and merges MR
        ↓ approved_at recorded (status: provisioning)
Webhook fires → repo created + skeleton files pushed
        ↓ provisioned_at recorded (status: provisioned)
GitLab discovery picks up catalog-info.yaml
        ↓
Component appears in Backstage catalog
```

### Dashboard Features

| Feature | What It Shows |
|---|---|
| KPI Cards | Total / Pending / Provisioned / Failed + avg stage times |
| Pipeline Funnel | How many requests are at each stage right now |
| Stage Duration Bar | Avg time at each stage across all requests |
| Request Timeline | Per-request Gantt — exactly where time is spent |
| Team Pie Chart | Which teams are creating the most repos |
| Requests Table | Full list with timestamps, durations, MR + repo links |

---

## 2. Architecture

```
Backstage Template (MR opened)
        ↓ webhook: action=open
repoProvisioner.ts
        ↓ recordSubmission() → status: pending
launchpad_requests table (PostgreSQL)

Platform team merges MR
        ↓ webhook: state=merged
repoProvisioner.ts
        ↓ recordSubmission() upsert → full details
        ↓ recordProvisioning() → status: provisioning
        ↓ createRepo() + pushSkeletonFiles()
        ↓ recordSuccess() → status: provisioned
launchpad_requests table (PostgreSQL)

Frontend LaunchPadPage
        ↓ polls /api/launchpad/stats + /requests + /stats/by-team
        ↓ renders ECharts + table
```

### PostgreSQL Database Isolation

> ℹ️ **Important:** Backstage creates a **separate database per plugin** — not a schema. The `launchpad_requests` table lives in `backstage_plugin_launchpad`, not the `backstage` database.

```
PostgreSQL server
  ├── backstage                          ← main app DB
  ├── backstage_plugin_catalog           ← catalog tables
  ├── backstage_plugin_launchpad         ← LaunchPad tables ✅
  │     └── public.launchpad_requests    ← our table
  └── backstage_plugin_<others>          ← other plugin DBs
```

**Connecting to the correct database for manual inspection:**

```bash
# ✅ Correct
docker exec -it backstage-postgres psql -U postgres -d backstage_plugin_launchpad

# ❌ Wrong — table won't be found here
docker exec -it backstage-postgres psql -U postgres -d backstage
```

---

## 3. Database Schema

### Table: launchpad_requests

| Column | Type | Notes |
|---|---|---|
| `id` | SERIAL PK | Auto-increment |
| `repo_name` | TEXT NOT NULL | Repo name from request |
| `namespace` | TEXT NOT NULL | GitLab namespace (e.g. cloudopsedge) |
| `owner_team` | TEXT | Keycloak group reference |
| `requested_by` | TEXT | Username or `via-template` |
| `description` | TEXT | Repo description |
| `mr_iid` | INTEGER | GitLab MR number |
| `mr_url` | TEXT | Full MR URL |
| `repo_url` | TEXT | Created repo URL (set on success) |
| `submitted_at` | TIMESTAMPTZ | When MR was opened |
| `approved_at` | TIMESTAMPTZ | When MR was merged |
| `provisioned_at` | TIMESTAMPTZ | When repo + files created |
| `failed_at` | TIMESTAMPTZ | When provisioning failed |
| `status` | TEXT | `pending` / `provisioning` / `provisioned` / `failed` |
| `error_message` | TEXT | Error detail on failure |
| `created_at` | TIMESTAMPTZ | Row created |
| `updated_at` | TIMESTAMPTZ | Row last updated |

### Indexes

```sql
CREATE INDEX idx_launchpad_status       ON launchpad_requests(status);
CREATE INDEX idx_launchpad_owner_team   ON launchpad_requests(owner_team);
CREATE INDEX idx_launchpad_submitted_at ON launchpad_requests(submitted_at);
CREATE INDEX idx_launchpad_repo_name    ON launchpad_requests(repo_name);
```

### Status Lifecycle

```
pending → provisioning → provisioned
                      ↘ failed
```

### recordSubmission Upsert Behaviour

`recordSubmission` is called **twice** per request:

1. **MR open** — inserts row with `repo_name` from branch name, `ownerTeam: 'pending'`, `description: ''`
2. **MR merge** — upserts existing row with full details from the request YAML

```typescript
// If row exists for mrIid → UPDATE with full details
// If no row → INSERT new row
```

---

## 4. File Structure

```
backstage/
  packages/
    backend/src/
      ├── index.ts                              ← register launchPadPlugin + repoProvisionerPlugin
      └── plugins/
            ├── launchpad.ts                    ← DB store + API endpoints
            └── repoProvisioner.ts              ← webhook handler (updated)

    app/src/
      ├── App.tsx                               ← add /launchpad route
      ├── components/
      │     ├── Root/
      │     │     └── Root.tsx                 ← add LaunchPad sidebar item
      │     └── LaunchPad/
      │           ├── api.ts                   ← fetch helpers + TypeScript types
      │           ├── KpiCards.tsx             ← 6 summary KPI cards
      │           ├── FunnelChart.tsx          ← ECharts funnel
      │           ├── StageDurationChart.tsx   ← ECharts horizontal bar
      │           ├── TimelineChart.tsx        ← ECharts Gantt
      │           ├── TeamChart.tsx            ← ECharts donut pie
      │           └── LaunchPadPage.tsx        ← main page with tabs
```

---

## 5. Backend – launchpad.ts

Place at `packages/backend/src/plugins/launchpad.ts`.

### Key Points

**Database client — use `getConfigArray` not bracket notation:**

```typescript
// ❌ Wrong — bracket notation not supported by Backstage ConfigReader
const token = config.getString('integrations.gitlab[0].token');

// ✅ Correct
const gitlabConfigs = config.getConfigArray('integrations.gitlab');
const token = gitlabConfigs[0].getString('token');
```

**deps must include `config`:**

```typescript
deps: {
  logger: coreServices.logger,
  database: coreServices.database,
  httpRouter: coreServices.httpRouter,
  config: coreServices.rootConfig,   // ← required for /provision endpoint
},
async init({ logger, database, httpRouter, config }) {
```

**`/submit` route must be unauthenticated:**

```typescript
httpRouter.addAuthPolicy({
  path: '/submit',
  allow: 'unauthenticated',
});
```

### API Endpoints

| Method | Path | Description |
|---|---|---|
| GET | `/api/launchpad/stats` | Summary KPIs — counts + avg durations |
| GET | `/api/launchpad/requests` | All requests ordered by created_at desc |
| GET | `/api/launchpad/requests/recent` | Last 20 requests |
| GET | `/api/launchpad/requests/:status` | Filter by status |
| GET | `/api/launchpad/stats/by-team` | Request counts grouped by owner_team |
| POST | `/api/launchpad/submit` | Record submission (called by template) |
| PATCH | `/api/launchpad/submit/:repoName` | Update mrIid on pending record |
| POST | `/api/launchpad/provision` | Manual trigger — mimics webhook flow |

### Test API Endpoints

```bash
TOKEN=$(curl -sk "https://localhost/api/auth/guest/refresh" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['backstageIdentity']['token'])")

# Stats
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/launchpad/stats" | python3 -m json.tool

# All requests
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/launchpad/requests" | python3 -m json.tool

# By team
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/launchpad/stats/by-team" | python3 -m json.tool

# Pending only
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/launchpad/requests/pending" | python3 -m json.tool
```

---

## 6. Backend – repoProvisioner.ts Changes

The key change is handling **two webhook events** — MR open and MR merge.

### Webhook Event Handling

```typescript
const mrAction = event.object_attributes?.action;
const mrState  = event.object_attributes?.state;

// MR OPENED → record pending submission
if (mrAction === 'open') {
  // Extract repo name from branch: "request/my-repo" → "my-repo"
  const sourceBranch = event.object_attributes?.source_branch ?? '';
  const repoName = sourceBranch.replace('request/', '').trim();

  await getLaunchPadStore().recordSubmission({
    repoName,
    namespace: 'cloudopsedge',
    ownerTeam: 'pending',      // updated on merge with real value
    requestedBy: 'via-template',
    description: '',           // updated on merge with real value
    mrIid: mrIid,
    mrUrl: event.object_attributes.url ?? '',
  });

  res.status(200).json({ status: 'submission-recorded' });
  return;
}

// MR MERGED → provision the repo
if (mrState !== 'merged') {
  res.status(200).json({ status: 'ignored' });
  return;
}
```

### Why Branch Name Instead of YAML File for Open Event

When the MR is first opened, the request YAML file is on the **source branch** (not yet on main). Reading it via the GitLab API at this point is unreliable — the file may not be accessible immediately. Instead, the repo name is extracted directly from the branch name (`request/my-repo` → `my-repo`), which is always available in the webhook payload.

On MR **merge**, `readRequestFile` reads from `main` (where the file lands after merge) and gets the full details — owner team, description, namespace.

### recordSubmission Flow

```
MR opened:  INSERT  (repoName from branch, ownerTeam='pending')
MR merged:  UPDATE  (full details from YAML — ownerTeam, description, namespace)
```

The `LaunchPadStore.recordSubmission` is an **upsert** — it checks if a row exists for the `mrIid` first:

```typescript
const existing = await this.db('launchpad_requests')
  .where({ mr_iid: params.mrIid })
  .first();

if (existing) {
  // UPDATE existing row with full details
} else {
  // INSERT new row
}
```

### Webhook Order of Operations (MR Merge)

```
1. readRequestFile()          ← get full params from YAML on main
2. recordSubmission() upsert  ← update pending row with real details
3. recordProvisioning()       ← status: provisioning, approved_at: NOW()
4. createRepo()               ← GitLab API: create repo
5. pushSkeletonFiles()        ← GitLab API: commit README + catalog-info + mkdocs
6. recordSuccess()            ← status: provisioned, provisioned_at: NOW()
7. postMrComment()            ← comment back on MR with repo URL
```

### readRequestFile ref Parameter

`readRequestFile` accepts an optional `ref` parameter:

```typescript
// MR open  → read from source branch
// MR merge → read from main (default)
async function readRequestFile(
  mrIid: number,
  token: string,
  baseUrl: string,
  requestsProjectId: string,
  ref = 'main',   // ← defaults to main
): Promise<Record<string, string>>
```

---

## 7. Register Plugins in index.ts

```typescript
// packages/backend/src/index.ts

import { createBackend } from '@backstage/backend-defaults';
import { keycloakCustomModule } from './keycloakProvider';
import { launchPadPlugin } from './plugins/launchpad';
import { repoProvisionerPlugin } from './repoProvisioner';

const backend = createBackend();

backend.add(keycloakCustomModule);
backend.add(
  import('@backstage-community/plugin-catalog-backend-module-gitlab/alpha'),
);
backend.add(import('@immobiliarelabs/backstage-plugin-gitlab-backend'));
backend.add(import('@backstage/plugin-scaffolder-backend-module-gitlab'));

// ✅ LaunchPad must be registered BEFORE repoProvisionerPlugin
// so the store is initialized before the webhook handler tries to use it
backend.add(launchPadPlugin);
backend.add(repoProvisionerPlugin);

backend.start();
```

> ⚠️ **WARNING:** `launchPadPlugin` must come before `repoProvisionerPlugin`. The provisioner calls `getLaunchPadStore()` at runtime — if LaunchPad hasn't initialized yet, the store will be null and throw.

---

## 8. Frontend – Install ECharts

```bash
yarn --cwd packages/app add echarts echarts-for-react
```

### Verify Installation

```bash
cat packages/app/package.json | grep -E "echarts"
# Expected:
# "echarts": "^5.x.x",
# "echarts-for-react": "^3.x.x"
```

---

## 9. Frontend – api.ts

Place at `packages/app/src/components/LaunchPad/api.ts`.

Handles token fetch automatically — all API calls use the guest token from `/api/auth/guest/refresh`. Exports:

- `LaunchPadRequest` — full request entity interface
- `SummaryStats` — KPI stats interface
- `TeamStat` — team count interface
- `launchPadApi` — object with `getStats`, `getRequests`, `getRecentRequests`, `getByTeam`, `getByStatus`

---

## 10. Frontend – KpiCards.tsx

Place at `packages/app/src/components/LaunchPad/KpiCards.tsx`.

Six KPI cards in a row:

| Card | Value | Colour |
|---|---|---|
| Total Requests | total count | Blue |
| Pending Approval | pending count | Orange |
| Provisioned | provisioned count | Green |
| Failed | failed count | Red |
| Avg Approval Time | avgSubmitToApprove | Purple |
| Avg End-to-End | avgEndToEnd | Teal |

Duration auto-formats: `< 1m` → seconds, `< 1h` → minutes, `≥ 1h` → hours.

---

## 11. Frontend – FunnelChart.tsx

Place at `packages/app/src/components/LaunchPad/FunnelChart.tsx`.

ECharts funnel with 4 stages:

| Stage | Colour | Value |
|---|---|---|
| Submitted | Blue | total |
| Approved | Purple | provisioning + provisioned + failed |
| Provisioning | Orange | provisioned + failed |
| Provisioned | Green | provisioned |

---

## 12. Frontend – StageDurationChart.tsx

Place at `packages/app/src/components/LaunchPad/StageDurationChart.tsx`.

ECharts horizontal bar chart with 3 bars:

| Bar | Colour | Metric |
|---|---|---|
| Submit → Approve | Purple | `avgSubmitToApprove` |
| Approve → Provision | Orange | `avgApproveToProvision` |
| End-to-End | Blue | `avgEndToEnd` |

Labels auto-format seconds / minutes / hours.

---

## 13. Frontend – TimelineChart.tsx

Place at `packages/app/src/components/LaunchPad/TimelineChart.tsx`.

ECharts stacked horizontal bar (Gantt-style) — one row per repo:

| Bar | Colour | Stage |
|---|---|---|
| Waiting Approval | Purple | submitted_at → approved_at |
| Provisioning | Orange | approved_at → provisioned_at |
| Failed | Red | provisioning bar if status === failed |

Chart height auto-scales: `52px × number of requests + 80px`.

---

## 14. Frontend – TeamChart.tsx

Place at `packages/app/src/components/LaunchPad/TeamChart.tsx`.

ECharts donut pie chart with vertical legend.

- `cleanTeamName()` strips `group:default/` and title-cases: `group:default/infra-team` → `Infra Team`
- Legend shows: `Team Name (count · percentage%)`
- Hover tooltip shows name, count, share %
- Cycles through 10 distinct colours

---

## 15. Frontend – LaunchPadPage.tsx

Place at `packages/app/src/components/LaunchPad/LaunchPadPage.tsx`.

### Three Tabs

| Tab | Content |
|---|---|
| Overview | Funnel + Stage Duration + Team Pie + Recent Requests (top 5) |
| Timeline | Full Gantt chart for all requests |
| All Requests | Complete table with all fields and links |

### Requests Table Columns

| Column | Shows |
|---|---|
| Repo | Name + error message if failed |
| Status | Colour-coded chip |
| Owner Team | Cleaned team name (no `group:default/` prefix) |
| Submitted | Date + time |
| Wait (Submit→Approve) | Duration |
| Provision Time | Duration |
| End-to-End | Duration |
| Links | MR link + Repo link icons (open in new tab) |

### Status Chip Colours

| Status | Background | Text |
|---|---|---|
| pending | Amber light | Orange |
| provisioning | Purple light | Purple |
| provisioned | Green light | Green |
| failed | Red light | Red |

### Auto-Refresh

Dashboard auto-refreshes every 60 seconds. Manual refresh button top right. Last refreshed timestamp shown below the title.

---

## 16. Wire into App.tsx and Root.tsx

### App.tsx — two changes only

```typescript
// 1. Add import at top
import { LaunchPadPage } from './components/LaunchPad/LaunchPadPage';

// 2. Add route inside <FlatRoutes>
<Route path="/launchpad" element={<LaunchPadPage />} />
```

### Root.tsx — two changes only

```typescript
// 1. Add import at top
import RocketLaunchIcon from '@material-ui/icons/FlightTakeoff';

// 2. Add sidebar item after Create...
<SidebarItem icon={RocketLaunchIcon} to="launchpad" text="LaunchPad" />
```

---

## 17. Rebuild & Deploy

```bash
# Install ECharts (if not done)
yarn --cwd packages/app add echarts echarts-for-react

# Rebuild
yarn tsc
yarn build:backend

# Remove old container
docker rm -f backstage

# Build new image
docker image build . -f packages/backend/Dockerfile --tag backstage:latest

# Run
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

# Verify
docker logs backstage | grep -i "launchpad\|repo-provisioner"
# Expected:
# LaunchPad: Database initialized
# LaunchPad: API listening at /api/launchpad
# RepoProvisioner: Webhook listening at /api/repo-provisioner/webhook
```

---

## 18. Verification

### Verify DB Table Exists

```bash
docker exec -it backstage-postgres psql -U postgres \
  -d backstage_plugin_launchpad -c "\dt"
# Expected: launchpad_requests table present

docker exec -it backstage-postgres psql -U postgres \
  -d backstage_plugin_launchpad \
  -c "SELECT repo_name, status, submitted_at, approved_at, provisioned_at FROM launchpad_requests ORDER BY id DESC LIMIT 5;"
```

### Simulate MR Open (pending entry)

```bash
curl -sk -X POST \
  "https://localhost/api/repo-provisioner/webhook" \
  -H "Content-Type: application/json" \
  -H "X-Gitlab-Token: YOUR_WEBHOOK_SECRET" \
  -d '{
    "object_kind": "merge_request",
    "project": {
      "path_with_namespace": "cloudopsedge/platform/repo-requests"
    },
    "object_attributes": {
      "iid": 10,
      "state": "opened",
      "action": "open",
      "source_branch": "request/my-test-repo",
      "url": "https://gitlab.com/cloudopsedge/platform/repo-requests/-/merge_requests/10"
    }
  }' | python3 -m json.tool
# Expected: {"status": "submission-recorded"}
```

### Simulate MR Merge (provisioning)

```bash
curl -sk -X POST \
  "https://localhost/api/repo-provisioner/webhook" \
  -H "Content-Type: application/json" \
  -H "X-Gitlab-Token: YOUR_WEBHOOK_SECRET" \
  -d '{
    "object_kind": "merge_request",
    "project": {
      "path_with_namespace": "cloudopsedge/platform/repo-requests"
    },
    "object_attributes": {
      "iid": 10,
      "state": "merged",
      "action": "merge",
      "source_branch": "request/my-test-repo",
      "url": "https://gitlab.com/cloudopsedge/platform/repo-requests/-/merge_requests/10"
    }
  }' | python3 -m json.tool
# Expected: {"status": "created", "repoUrl": "https://gitlab.com/cloudopsedge/my-test-repo"}
```

### Manual Provision (no MR needed)

```bash
TOKEN=$(curl -sk "https://localhost/api/auth/guest/refresh" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['backstageIdentity']['token'])")

curl -sk -X POST \
  "https://localhost/api/launchpad/provision" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "repoName": "test-service-001",
    "description": "Test via LaunchPad provision API",
    "ownerTeam": "group:default/infra-team",
    "namespace": "cloudopsedge",
    "requestedBy": "nanthagopal"
  }' | python3 -m json.tool
# Expected: {"status": "provisioned", "repoUrl": "..."}
```

### Verify Dashboard Loads

```
https://localhost/launchpad
  → KPI cards visible              ✅
  → Overview tab: funnel + charts  ✅
  → Timeline tab: Gantt chart      ✅
  → All Requests tab: full table   ✅
  → LaunchPad in sidebar           ✅
```

---

## 19. Troubleshooting

---

### ❌ LaunchPadStore not initialized

**Symptom:**
```
Error: LaunchPadStore not initialized
```

**Cause:** `repoProvisionerPlugin` is registered before `launchPadPlugin` in `index.ts`.

**Fix:** Ensure `launchPadPlugin` is registered first:

```typescript
backend.add(launchPadPlugin);          // ← first
backend.add(repoProvisionerPlugin);    // ← second
```

---

### ❌ Table not found in backstage database

**Symptom:**
```
relation "launchpad_requests" does not exist
```

**Cause:** Looking in wrong database. Backstage creates a separate DB per plugin.

**Fix:**
```bash
# ✅ Correct database
docker exec -it backstage-postgres psql -U postgres -d backstage_plugin_launchpad -c "\dt"

# ❌ Wrong database
docker exec -it backstage-postgres psql -U postgres -d backstage -c "\dt launchpad*"
```

---

### ❌ MR open event not creating pending entry

**Symptom:** Webhook returns `{"status": "submission-recorded"}` but no DB row created.

**Cause:** `source_branch` in the webhook payload doesn't follow `request/<repo-name>` pattern — `repoName` extracts as empty string and `recordSubmission` is skipped.

**Diagnosis:**
```bash
docker logs backstage | grep -i "repo-provisioner"
# Look for: Could not extract repo name from branch
```

**Fix:** Ensure the template `branchName` follows the pattern:
```yaml
branchName: request/${{ parameters.repoName }}
```

---

### ❌ Duplicate rows in DB

**Symptom:** Two rows for the same repo in `launchpad_requests`.

**Cause:** `recordSubmission` inserts instead of upserts — this happens if `mr_iid` is 0 for both rows (multiple manual provisions or simulated events with `iid: 0`).

**Fix — clean up duplicates:**
```bash
docker exec -it backstage-postgres psql -U postgres \
  -d backstage_plugin_launchpad \
  -c "DELETE FROM launchpad_requests WHERE repo_name = '' OR repo_name IS NULL;"

# Delete specific duplicates keeping the latest
docker exec -it backstage-postgres psql -U postgres \
  -d backstage_plugin_launchpad \
  -c "DELETE FROM launchpad_requests a USING launchpad_requests b
      WHERE a.id < b.id AND a.repo_name = b.repo_name;"
```

---

### ❌ Charts not rendering — ECharts error

**Symptom:** Blank chart area or console error about ECharts.

**Cause:** ECharts not installed or wrong import.

**Fix:**
```bash
# Verify installed
cat packages/app/package.json | grep echarts

# If missing
yarn --cwd packages/app add echarts echarts-for-react
```

```typescript
// Correct import
import ReactECharts from 'echarts-for-react';
```

---

### ❌ Dashboard shows no data after rebuild

**Symptom:** KPI cards show 0, charts empty.

**Diagnosis:**
```bash
# Check API returns data
TOKEN=$(curl -sk "https://localhost/api/auth/guest/refresh" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['backstageIdentity']['token'])")

curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/launchpad/stats" | python3 -m json.tool
```

**If API returns data but UI is empty:** Browser cache issue — hard refresh with `Ctrl + Shift + R`.

**If API returns empty:** Seed test data:

```bash
docker exec -it backstage-postgres psql -U postgres \
  -d backstage_plugin_launchpad -c "
INSERT INTO launchpad_requests
  (repo_name, namespace, owner_team, requested_by, description,
   mr_iid, mr_url, repo_url, submitted_at, approved_at, provisioned_at, status)
VALUES
  ('payment-service', 'cloudopsedge', 'group:default/infra-team',
   'nanthagopal', 'Payment service', 101,
   'https://gitlab.com/cloudopsedge/platform/repo-requests/-/merge_requests/101',
   'https://gitlab.com/cloudopsedge/payment-service',
   NOW() - INTERVAL '2 hours', NOW() - INTERVAL '1 hour 30 minutes',
   NOW() - INTERVAL '1 hour', 'provisioned'),
  ('auth-service', 'cloudopsedge', 'group:default/devops-team',
   'nanthagopal', 'Auth service', 102, 
   'https://gitlab.com/cloudopsedge/platform/repo-requests/-/merge_requests/102',
   NULL, NOW() - INTERVAL '3 hours', NULL, NULL, 'pending');
"
```

---

### ❌ LaunchPad not in sidebar

**Symptom:** No LaunchPad item in the left navigation.

**Fix:** Verify `Root.tsx` has both the import and the `SidebarItem`:

```typescript
import RocketLaunchIcon from '@material-ui/icons/FlightTakeoff';

<SidebarItem icon={RocketLaunchIcon} to="launchpad" text="LaunchPad" />
```

---

### ❌ /launchpad route shows 404

**Symptom:** Navigating to `/launchpad` shows a not found page.

**Fix:** Verify `App.tsx` has the route:

```typescript
import { LaunchPadPage } from './components/LaunchPad/LaunchPadPage';

<Route path="/launchpad" element={<LaunchPadPage />} />
```

---

## 20. Quick Reference

### DB Queries

```bash
# Connect to LaunchPad database
docker exec -it backstage-postgres psql -U postgres -d backstage_plugin_launchpad

# View all requests
SELECT repo_name, status, submitted_at, approved_at, provisioned_at
FROM launchpad_requests ORDER BY id DESC;

# View pending requests
SELECT repo_name, submitted_at, mr_url
FROM launchpad_requests WHERE status = 'pending';

# View failed requests
SELECT repo_name, failed_at, error_message
FROM launchpad_requests WHERE status = 'failed';

# Clean ghost rows
DELETE FROM launchpad_requests WHERE repo_name = '' OR repo_name IS NULL;

# Avg stage durations
SELECT
  AVG(EXTRACT(EPOCH FROM (approved_at - submitted_at)) / 60) AS avg_approval_mins,
  AVG(EXTRACT(EPOCH FROM (provisioned_at - approved_at)) / 60) AS avg_provision_mins,
  AVG(EXTRACT(EPOCH FROM (provisioned_at - submitted_at)) / 60) AS avg_total_mins
FROM launchpad_requests
WHERE provisioned_at IS NOT NULL;
```

### Simulate Webhook Events

```bash
# MR opened (pending)
curl -sk -X POST https://localhost/api/repo-provisioner/webhook \
  -H "Content-Type: application/json" \
  -H "X-Gitlab-Token: YOUR_SECRET" \
  -d '{"object_kind":"merge_request","project":{"path_with_namespace":"cloudopsedge/platform/repo-requests"},"object_attributes":{"iid":10,"state":"opened","action":"open","source_branch":"request/my-repo","url":"https://gitlab.com/..."}}'

# MR merged (provision)
curl -sk -X POST https://localhost/api/repo-provisioner/webhook \
  -H "Content-Type: application/json" \
  -H "X-Gitlab-Token: YOUR_SECRET" \
  -d '{"object_kind":"merge_request","project":{"path_with_namespace":"cloudopsedge/platform/repo-requests"},"object_attributes":{"iid":10,"state":"merged","action":"merge","source_branch":"request/my-repo","url":"https://gitlab.com/..."}}'
```

### Manual Provision via API

```bash
TOKEN=$(curl -sk "https://localhost/api/auth/guest/refresh" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['backstageIdentity']['token'])")

curl -sk -X POST "https://localhost/api/launchpad/provision" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"repoName":"my-service","description":"My service","ownerTeam":"group:default/infra-team","namespace":"cloudopsedge","requestedBy":"nanthagopal"}'
```

### Watch Logs

```bash
docker logs -f backstage | grep -i "launchpad\|repo-provisioner"
```

### Dashboard URL

```
https://localhost/launchpad
```

---

*Generated: March 2026 | Backstage v1.48.0 | Node.js v24.13.1 | ECharts v5 | GitLab SaaS*
