# App Tracker — Rebuild, Deploy & Verification Guide

> Version: 1.1 | March 2026  
> Backstage v1.48.0 | Node.js v24.13.1

---

## Prerequisites

Complete all items below before building or deploying the App Tracker.

---

### 1. Backstage Infrastructure (Already in place)

| Requirement | Status |
|---|---|
| Backstage running on Docker with PostgreSQL | ✅ Already done |
| Keycloak users/groups synced to Backstage catalog | ✅ Already done |
| GitLab integration configured | ✅ Already done |
| LaunchPad plugin implemented | ✅ Already done |
| Node.js v24.13.1 | ✅ Already done |
| Backstage v1.48.0 | ✅ Already done |

---

### 2. New npm Packages

```bash
yarn --cwd packages/backend add multer xlsx
yarn --cwd packages/backend add --dev @types/multer
```

---

### 3. Keycloak Groups

The App Tracker sign-off panel filters assignee picker results by Keycloak group.
All groups below must exist in Keycloak **before** the sign-off picker will work correctly.

#### 3.1 Rename Existing Group

```
Groups → platform-common-team → Edit → Name: platform-team → Save
```

#### 3.2 Create New Parent Groups

```
Groups → Create group:
  security-team          ← parent for Red Team, Blue Team, Info Sec
```

#### 3.3 Create Child Groups Under platform-team

```
Groups → platform-team → Children → Create child group:
  platform-arch-team
```

#### 3.4 Create Child Groups Under security-team

```
Groups → security-team → Children → Create child group:
  red-team
  blue-team
  info-sec
```

#### 3.5 Create Full Team Groups (for catalog ownership)

```
Groups → Create group (one by one):
  product-team
  application-arch-team
  dev-fintech-team         ← FinTech engineers
  dev-insuretech-team      ← InsureTech engineers
  ml-team                  ← AI/ML engineers
  data-team                ← Data engineers
  qa-team
  risk-compliance-team
```

#### 3.6 Create Approval Groups (for sign-off picker)

These groups contain **managers and leads only** — not all team members.
Only members of these groups will appear in the sign-off assignee picker.

```
Groups → Create group (one by one):
  product-managers         ← Product Manager(s)
  application-arch-leads   ← Lead Architect(s)
  dev-fintech-em           ← FinTech Engineering Manager (Raj K)
  dev-insuretech-em        ← InsureTech Engineering Manager (Kumar S)
  ml-leads                 ← ML Lead(s)
  data-leads               ← Data Lead(s)
  qa-leads                 ← QA Lead(s)
  rc-leads                 ← Risk & Compliance Lead(s)
  red-team-leads           ← Red Team Lead
  blue-team-leads          ← Blue Team Lead
  info-sec-leads           ← Info Sec Lead
  platform-arch-leads      ← Platform Architect(s)
  infra-em                 ← Infra Engineering Manager
  devops-em                ← DevOps Engineering Manager
  sre-em                   ← SRE Engineering Manager
```

#### 3.7 Add Users to Approval Groups

For each approval group go to:
`Groups → (group name) → Members tab → Add member`

| Approval Group | Who to Add |
|---|---|
| `product-managers` | Product Manager(s) |
| `application-arch-leads` | Lead Architect(s) |
| `dev-fintech-em` | Raj K (FinTech EM) |
| `dev-insuretech-em` | Kumar S (InsureTech EM) |
| `ml-leads` | ML Lead(s) |
| `data-leads` | Data Lead(s) |
| `qa-leads` | QA Lead(s) |
| `rc-leads` | Risk & Compliance Lead(s) |
| `red-team-leads` | Red Team Lead |
| `blue-team-leads` | Blue Team Lead |
| `info-sec-leads` | Info Sec Lead |
| `platform-arch-leads` | Platform Architect(s) |
| `infra-em` | Infra EM |
| `devops-em` | DevOps EM |
| `sre-em` | SRE EM |

#### 3.8 Verify Groups Synced to Backstage

After creating all groups, restart Backstage to trigger an immediate Keycloak sync:

```bash
docker restart backstage

# Watch sync logs
docker logs -f backstage | grep -i keycloak
# Expected:
# KeycloakCustomProvider: Reading users and groups
# KeycloakCustomProvider: Ingesting X users and Y groups

# Verify groups in catalog
TOKEN=$(curl -sk "https://localhost/api/auth/guest/refresh" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['backstageIdentity']['token'])")

curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/catalog/entities?filter=kind=Group" \
  | python3 -m json.tool | grep '"name"'
```

---

### 4. Role → Keycloak Group Mapping

This is the complete mapping used by the App Tracker sign-off panel.
Each role's assignee picker shows **only** members of its linked Keycloak approval group.

| # | Role | Type | Keycloak Approval Group | Env Required |
|---|---|---|---|---|
| 1 | Product Team | standalone | `product-managers` | all envs |
| 2 | Dev | parent | — (derived) | — |
| 3 | Application Arch | child of Dev | `application-arch-leads` | all envs |
| 4 | Dev - FinTech | child of Dev | `dev-fintech-em` | all envs |
| 5 | Dev - InsureTech | child of Dev | `dev-insuretech-em` | all envs |
| 6 | ML Engineer | child of Dev | `ml-leads` | off by default |
| 7 | Data Engineer | child of Dev | `data-leads` | off by default |
| 8 | QA | standalone | `qa-leads` | uat onwards |
| 9 | Risk & Compliance | standalone | `rc-leads` | prd only |
| 10 | Security | parent | — (derived) | — |
| 11 | Red Team | child of Security | `red-team-leads` | stg onwards |
| 12 | Blue Team | child of Security | `blue-team-leads` | stg onwards |
| 13 | Info Sec | child of Security | `info-sec-leads` | stg onwards |
| 14 | Platform Team | parent | — (derived) | — |
| 15 | Platform Arch | child of Platform Team | `platform-arch-leads` | all envs |
| 16 | Infra | child of Platform Team | `infra-em` | all envs |
| 17 | DevOps | child of Platform Team | `devops-em` | all envs |
| 18 | SRE | child of Platform Team | `sre-em` | stg onwards |

---

### 5. SQL Migration

Run the migration to add `keycloak_group` column and re-seed roles:

```bash
docker exec -i backstage-postgres psql -U postgres \
  -d "backstage_plugin_app-tracker" \
  < packages/backend/src/plugins/appTracker/migrations/002_add_keycloak_group_to_roles.sql

# Verify
docker exec -it backstage-postgres psql -U postgres \
  -d "backstage_plugin_app-tracker" \
  -c "SELECT id, role_name, keycloak_group FROM app_tracker_roles ORDER BY id;"
```

---

### 6. Sample Data (Optional)

Use this CSV to import 13 sample entries across 3 app-family-groups for testing:

```csv
category,sub_category,app_family_group,app_family,app,env,release_version,priority,business_owner,tech_owner,dev_team,platform_contact,remarks
Banking,Retail,FinTech-Group,Payments,payment-service,dev,v1.2.0,High,John B,Raj K,dev-fintech-team,Nantha P,Dev deployment
Banking,Retail,FinTech-Group,Payments,payment-service,uat,v1.2.0,High,John B,Raj K,dev-fintech-team,Nantha P,UAT in progress
Banking,Retail,FinTech-Group,Payments,fraud-detection,dev,v0.9.0,Critical,John B,Kumar S,dev-fintech-team,Nantha P,ML model integration
Banking,Core,FinTech-Group,Identity,auth-service,dev,v2.1.0,High,Sarah K,Raj K,dev-fintech-team,Arun K,OAuth2 upgrade
Banking,Core,FinTech-Group,Identity,auth-service,uat,v2.1.0,High,Sarah K,Raj K,dev-fintech-team,Arun K,UAT sign-off pending
Banking,Retail,FinTech-Group,Frontend,customer-portal,dev,v3.0.0,Medium,Mike T,Priya R,dev-fintech-team,Nantha P,React 18 upgrade
Insurance,Claims,InsureTech-Group,Claims,claims-processor,dev,v1.0.0,High,David L,Kumar S,dev-insuretech-team,Arun K,Initial release
Insurance,Policy,InsureTech-Group,Policy,policy-manager,dev,v1.1.0,Medium,David L,Priya R,dev-insuretech-team,Arun K,Policy engine v2
Insurance,Self-Service,InsureTech-Group,Frontend,insure-portal,dev,v1.0.0,Low,Emma W,Raj K,dev-insuretech-team,Nantha P,Beta launch
Platform,Data,Platform-Group,Data,data-pipeline,dev,v0.5.0,High,Alex T,Kumar S,dev-platform-team,Arun K,Spark upgrade
Platform,Monitoring,Platform-Group,Observability,grafana,dev,v9.0.0,Medium,Nantha P,Arun K,sre-team,Nantha P,Dashboard migration
Platform,DevOps,Platform-Group,DevOps-Tools,gitlab,dev,v16.0.0,Critical,Nantha P,Arun K,devops-team,Nantha P,Major version upgrade
Platform,IaC,Platform-Group,DevOps-Tools,terraform,dev,v1.5.0,High,Nantha P,Arun K,infra-team,Nantha P,Provider upgrade
```

---



```bash
yarn --cwd packages/backend add multer xlsx
yarn --cwd packages/backend add --dev @types/multer
```

### Verify packages installed correctly

```bash
cat packages/backend/package.json | grep -E "multer|xlsx"
# Expected:
# "multer": "x.x.x"
# "xlsx": "x.x.x"
# "@types/multer": "x.x.x"
```

---

## Step 2 — Verify All Files Are In Place

Before rebuilding, confirm every file is placed correctly:

```bash
# Backend
ls packages/backend/src/plugins/appTracker/
# Expected:
# appTracker.ts
# migrations/001_create_app_tracker_tables.sql

# Frontend components
ls packages/app/src/components/AppTracker/
# Expected:
# index.ts
# types.ts
# AppTrackerPage.tsx
# HeatmapTab.tsx
# DetailedTableTab.tsx
# SignoffPanel.tsx
# ImportTab.tsx
# ConfigureRolesTab.tsx
# hooks/

ls packages/app/src/components/AppTracker/hooks/
# Expected:
# useEntries.ts
# useSignoffs.ts
# useRoles.ts

# Plugin registration
ls packages/app/src/plugin.ts
# Expected: plugin.ts

# Verify index.ts has appTrackerPlugin import
grep "appTracker" packages/backend/src/index.ts
# Expected:
# import { appTrackerPlugin } from './plugins/appTracker/appTracker';
# backend.add(appTrackerPlugin);

# Verify App.tsx has the route
grep "app-tracker" packages/app/src/App.tsx
# Expected:
# <Route path="/app-tracker" element={<AppTrackerPage />} />

# Verify Root.tsx has the sidebar item
grep "App Tracker" packages/app/src/components/Root/Root.tsx
# Expected:
# <SidebarItem icon={TrackChangesIcon} to="app-tracker" text="App Tracker" />
```

---

## Step 3 — Rebuild

```bash
# Step 3.1 — Install all dependencies (immutable = no lockfile changes)
yarn install --immutable

# Step 3.2 — Generate TypeScript type definitions
yarn tsc

# Step 3.3 — Build backend bundle
yarn build:backend
```

> ⚠️ **WARNING:** All three steps must complete without errors before proceeding.
> If `yarn tsc` fails, check for TypeScript errors in the new files.
> Common causes: wrong import paths, missing type exports from `types.ts`.

### Common TypeScript errors and fixes

```bash
# Error: Cannot find module './components/AppTracker'
# Fix: ensure index.ts exists at packages/app/src/components/AppTracker/index.ts

# Error: Module 'multer' has no exported member
# Fix: yarn --cwd packages/backend add --dev @types/multer

# Error: Cannot find module '@material-ui/lab'
# Fix: yarn --cwd packages/app add @material-ui/lab

# Error: Property 'addAuthPolicy' does not exist on type 'HttpRouterService'
# Fix: ensure you are on Backstage v1.48.0 — addAuthPolicy was added in v1.24+
```

---

## Step 4 — Remove Old Container & Build New Image

```bash
# Step 4.1 — Remove old container
docker rm -f backstage

# Step 4.2 — Build new image
# Use --no-cache only if you changed the Dockerfile (not required here)
docker image build . -f packages/backend/Dockerfile --tag backstage:latest
```

> ℹ️ **INFO:** The build may take 3–8 minutes depending on your machine.
> Watch for any build errors — particularly in the npm install layer.

---

## Step 5 — Run Container

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

> ℹ️ **INFO:** No new environment variables are required for App Tracker.
> It uses the same PostgreSQL connection as LaunchPad and the existing plugins.

---

## Step 6 — Watch Startup Logs

```bash
docker logs -f backstage
```

### Expected log sequence on successful startup

```
# Backstage core
Backend started successfully

# Keycloak sync
KeycloakCustomProvider: Reading users and groups
KeycloakCustomProvider: Ingesting X users and Y groups

# LaunchPad
LaunchPad: Database initialized
LaunchPad: API listening at /api/launchpad

# Repo Provisioner
RepoProvisioner: Webhook listening at /api/repo-provisioner/webhook

# ✅ App Tracker — new
AppTracker: Database initialized and default roles seeded
AppTracker: API listening at /api/app-tracker
```

> ⚠️ **WARNING:** If you see `AppTracker: Database initialized` but NOT
> `Default roles seeded` — the roles table already existed from a previous run.
> This is normal on container restart. Roles are seeded only on first startup.

---

## Step 7 — Verify Backend Plugin

### 7.1 Confirm plugin started

```bash
docker logs backstage | grep -i "app-tracker"
# Expected:
# AppTracker: Database initialized and default roles seeded
# AppTracker: API listening at /api/app-tracker
```

### 7.2 Confirm DB tables were created

```bash
docker exec -it backstage-postgres psql -U postgres -d backstage \
  -c "\dt app_tracker*"
# Expected:
#  public | app_tracker_entries       | table | postgres
#  public | app_tracker_import_log    | table | postgres
#  public | app_tracker_role_env_config | table | postgres
#  public | app_tracker_roles         | table | postgres
#  public | app_tracker_signoffs      | table | postgres
```

### 7.3 Confirm default roles were seeded

```bash
docker exec -it backstage-postgres psql -U postgres -d backstage \
  -c "SELECT id, role_name, parent_role_id FROM app_tracker_roles ORDER BY id;"
# Expected:
#  1 | Product Team     | NULL
#  2 | Application Arch | NULL
#  3 | Platform Arch    | NULL
#  4 | Dev              | NULL
#  5 | QA               | NULL
#  6 | Security         | NULL
#  7 | Red Team         | 6
#  8 | Blue Team        | 6
#  9 | Info Sec         | 6
# 10 | Platform Team    | NULL
# 11 | Infra            | 10
# 12 | DevOps           | 10
# 13 | SRE              | 10
```

### 7.4 Confirm env config was seeded

```bash
docker exec -it backstage-postgres psql -U postgres -d backstage \
  -c "SELECT role_id, env, required FROM app_tracker_role_env_config ORDER BY role_id, env;"
# Verify SRE (id=13) is false in dev and uat:
docker exec -it backstage-postgres psql -U postgres -d backstage \
  -c "SELECT role_id, env, required FROM app_tracker_role_env_config WHERE role_id=13;"
# Expected:
#  13 | box | t
#  13 | dev | f
#  13 | prd | t
#  13 | stg | t
#  13 | uat | f
```

---

## Step 8 — Verify API Endpoints

### 8.1 Get auth token

```bash
TOKEN=$(curl -sk "https://localhost/api/auth/guest/refresh" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['backstageIdentity']['token'])")

echo $TOKEN | head -c 50
# Expected: eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
```

### 8.2 Get roles

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/app-tracker/roles" \
  | python3 -m json.tool | grep "role_name"
# Expected: 13 role names including Product Team, Security, SRE etc.
```

### 8.3 Get env config matrix

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/app-tracker/roles/env-config" \
  | python3 -m json.tool | python3 -c "
import sys, json
data = json.load(sys.stdin)
print(f'Total config entries: {len(data[\"config\"])}')
# Expected: Total config entries: 55 (11 leaf roles × 5 envs)
"
```

### 8.4 Get entries (empty on first run)

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/app-tracker/entries" \
  | python3 -m json.tool
# Expected: {"entries": []}
```

### 8.5 Get heatmap summary (empty on first run)

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/app-tracker/entries/summary" \
  | python3 -m json.tool
# Expected: {"data": []}
```

### 8.6 Get catalog users (Keycloak-synced)

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/app-tracker/users" \
  | python3 -m json.tool | grep "displayName"
# Expected: list of Keycloak-synced users
```

---

## Step 9 — Verify Frontend

### 9.1 Check App Tracker page loads

```bash
# Open in browser
https://localhost/app-tracker
```

Expected:
- App Tracker page renders with 4 tabs: Status Heatmap, Detailed Table, Import, Configure Roles
- No console errors in browser DevTools
- "App Tracker" nav item visible in the sidebar

### 9.2 Smoke test — Add a test entry via API

```bash
curl -sk -X POST "https://localhost/api/app-tracker/entries" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "category": "Banking",
    "subCategory": "Retail",
    "appFamilyGroup": "FinTech-Group",
    "appFamily": "Payments",
    "app": "payment-service",
    "env": "dev",
    "releaseVersion": "v1.0.0",
    "priority": "High",
    "businessOwner": "John B",
    "techOwner": "Raj K",
    "remarks": "Test entry"
  }' | python3 -m json.tool
# Expected: {"entry": {"id": 1, "app": "payment-service", "auto_status": "Not Started", ...}}
```

### 9.3 Verify entry appears in heatmap

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/app-tracker/entries/summary" \
  | python3 -m json.tool
# Expected: payment-service / dev / Not Started cell
```

### 9.4 Add a signoff and verify auto-status recalculates

```bash
# Step 1: Get the entry id
ENTRY_ID=$(curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/app-tracker/entries" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['entries'][0]['id'])")

echo "Entry ID: $ENTRY_ID"

# Step 2: Add a signoff for Product Team (role_id=1)
curl -sk -X POST "https://localhost/api/app-tracker/entries/${ENTRY_ID}/signoffs" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "roleId": 1,
    "assigneeName": "backstage-admin",
    "assigneeEmail": "admin@backstage.local",
    "status": "Pending"
  }' | python3 -m json.tool
# Expected: {"signoff": {"id": 1, "status": "Pending", ...}}

# Step 3: Approve the signoff
SIGNOFF_ID=$(curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/app-tracker/entries/${ENTRY_ID}/signoffs" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['signoffs'][0]['id'])")

curl -sk -X PUT "https://localhost/api/app-tracker/signoffs/${SIGNOFF_ID}" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"status": "Approved", "remarks": "LGTM"}' \
  | python3 -m json.tool
# Expected: {"signoff": {"status": "Approved", ...}}

# Step 4: Check auto_status — should now be "In Progress"
# (Not all required roles approved yet — just one)
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/app-tracker/entries/${ENTRY_ID}" \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
print('auto_status:', data['entry']['auto_status'])
# Expected: In Progress
"
```

---

## Step 10 — Test Import

### 10.1 Create a test CSV

```bash
cat > /tmp/test_import.csv << 'EOF'
category,sub_category,app_family_group,app_family,app,env,release_version,priority,business_owner,tech_owner,dev_team,platform_contact,remarks
Banking,Retail,FinTech-Group,Payments,payment-service,uat,v1.0.0,High,John B,Raj K,infra-team,Nantha P,UAT deployment
Banking,Retail,FinTech-Group,Payments,payment-service,stg,v1.0.0,High,John B,Raj K,infra-team,Nantha P,Staging deployment
Banking,Retail,FinTech-Group,Auth,auth-service,dev,v2.1.0,Critical,Sarah K,Kumar S,devops-team,Nantha P,Initial deploy
Insurance,Claims,InsureTech-Group,Claims,claims-processor,dev,v0.5.0,Medium,Mike T,Priya R,sre-team,Arun K,Beta testing
EOF
echo "Test CSV created"
```

### 10.2 Import via API

```bash
curl -sk -X POST "https://localhost/api/app-tracker/import" \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@/tmp/test_import.csv" \
  -F "importedBy=test-user" \
  | python3 -m json.tool
# Expected:
# {
#   "status": "success",
#   "inserted": 4,
#   "updated": 0,
#   "failed": 0,
#   "errors": []
# }
```

### 10.3 Verify import log

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/app-tracker/import/logs" \
  | python3 -m json.tool | grep -E "file_name|status|inserted|updated"
# Expected: file_name: test_import.csv, status: success, inserted: 4
```

### 10.4 Verify heatmap now has data

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/app-tracker/entries/summary" \
  | python3 -m json.tool | grep '"app"'
# Expected: multiple app entries visible
```

---

## Troubleshooting

---

### ❌ AppTracker not appearing in logs

**Symptom:**
```
AppTracker: Database initialized and default roles seeded   ← missing
```

**Fix:** Check `index.ts` has the import and registration:
```bash
grep "appTracker" packages/backend/src/index.ts
# Must show both import and backend.add(appTrackerPlugin)
```

---

### ❌ App Tracker page shows blank / 404

**Symptom:** Navigating to `https://localhost/app-tracker` shows nothing or 404.

**Fix — Check App.tsx has the route:**
```bash
grep "app-tracker" packages/app/src/App.tsx
# Must show: <Route path="/app-tracker" element={<AppTrackerPage />} />
```

**Fix — Check plugin.ts exists:**
```bash
ls packages/app/src/plugin.ts
```

---

### ❌ TypeScript error: Cannot find module '@material-ui/lab'

**Symptom:**
```
Cannot find module '@material-ui/lab' or its corresponding type declarations
```

**Fix:**
```bash
yarn --cwd packages/app add @material-ui/lab
```

---

### ❌ TypeScript error: multer has no exported member

**Symptom:**
```
Module 'multer' has no exported member 'X'
```

**Fix:**
```bash
yarn --cwd packages/backend add --dev @types/multer
```

---

### ❌ DB tables not created — relation does not exist

**Symptom:**
```
error: relation "app_tracker_roles" does not exist
```

**Cause:** `migrate()` failed silently. Check logs:
```bash
docker logs backstage | grep -i "error\|app-tracker"
```

**Fix:** Most common cause is a permissions issue on the PostgreSQL `backstage` DB:
```bash
docker exec -it backstage-postgres psql -U postgres -d backstage \
  -c "GRANT ALL PRIVILEGES ON DATABASE backstage TO postgres;"
docker restart backstage
```

---

### ❌ Roles not seeded — table exists but empty

**Symptom:**
```
AppTracker: Database initialized and default roles seeded
```
But:
```bash
docker exec -it backstage-postgres psql -U postgres -d backstage \
  -c "SELECT count(*) FROM app_tracker_roles;"
# Returns: 0
```

**Cause:** `seedDefaults()` skips seeding if the table already has rows. If you manually deleted rows, the seed will not re-run.

**Fix — manually re-seed:**
```bash
docker exec -it backstage-postgres psql -U postgres -d backstage \
  -c "TRUNCATE app_tracker_roles CASCADE;"
docker restart backstage
```

---

### ❌ Import fails — multer not processing file

**Symptom:**
```
{"error": "No file uploaded"}
```

**Cause:** The `multer` package is missing or not installed in `packages/backend`.

**Fix:**
```bash
yarn --cwd packages/backend add multer xlsx
yarn --cwd packages/backend add --dev @types/multer
yarn build:backend
docker image build . -f packages/backend/Dockerfile --tag backstage:latest
```

---

## Quick Reference — All Verification Commands

```bash
# Get token
TOKEN=$(curl -sk "https://localhost/api/auth/guest/refresh" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['backstageIdentity']['token'])")

# Check plugin started
docker logs backstage | grep -i "app-tracker"

# List roles
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/app-tracker/roles" | python3 -m json.tool

# List entries
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/app-tracker/entries" | python3 -m json.tool

# Get heatmap
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/app-tracker/entries/summary" | python3 -m json.tool

# Get import logs
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/app-tracker/import/logs" | python3 -m json.tool

# Check DB tables
docker exec -it backstage-postgres psql -U postgres -d backstage -c "\dt app_tracker*"

# Watch live logs
docker logs -f backstage | grep -i "app-tracker"
```

---

*Generated: March 2026 | Backstage v1.48.0 | Node.js v24.13.1 | App Tracker v1.0*
