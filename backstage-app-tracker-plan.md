# Backstage – App Tracker Implementation Plan

> Org-Wide App Delivery Status · Env-Level Tracking · Multi-Stakeholder Sign-off  
> **Stack:** Backstage · PostgreSQL · Keycloak · GitLab  
> **Version:** 1.0 | March 2026

---

## Table of Contents

1. [Overview & Goals](#1-overview--goals)
2. [Data Model & Schema](#2-data-model--schema)
3. [Role Hierarchy & Sign-off Chain](#3-role-hierarchy--sign-off-chain)
4. [Auto-Status Derivation Logic](#4-auto-status-derivation-logic)
5. [API Design](#5-api-design)
6. [Frontend Design](#6-frontend-design)
7. [Import Specification](#7-import-specification)
8. [File Structure](#8-file-structure)
9. [Implementation Steps](#9-implementation-steps)
10. [Quick Reference](#10-quick-reference)

---

## 1. Overview & Goals

The App Tracker is an org-wide delivery status dashboard built as a Backstage plugin. It tracks every application's progression through environments from `dev` to `prd`, with full stakeholder sign-off chains, release version history, and priority management.

### Why This Exists

| Problem | Solution |
|---|---|
| No single view of what is deployed where | Heatmap: app × env status at a glance |
| No audit trail of who approved what | Sign-off chain per app-env-version |
| Status scattered across teams and tools | One source of truth in Backstage |
| Manual status updates prone to error | Auto-status derived from sign-off completions |
| Org-level reporting not possible | Category → Sub-Category → App hierarchy |

### Environments (in order)

```
dev → uat → stg → box → prd
```

### Key Design Decisions

| Decision | Choice | Reason |
|---|---|---|
| Granularity | One row per app + env + release_version | Full history, per-release tracking |
| Upsert key | app + env + release_version | Version keeps changing; same app can have multiple active releases |
| Status model | auto_status + manual_status → display_status | Auto derived from sign-offs; manual override when needed |
| Sign-off storage | Separate child table | Flexible roles, full audit trail, supports multiple assignees |
| Role requirement | Per role per env (configurable matrix) | Lower envs (dev/uat) don't need SRE or Security sign-off |
| User picker | Keycloak-synced Backstage catalog users | Consistent with org identity |
| Data persistence | PostgreSQL (same as LaunchPad) | Survives container restarts, org-wide access |

---

## 2. Data Model & Schema

### 5 Database Tables

```
app_tracker_roles              — configurable role definitions
app_tracker_role_env_config    — required flag per role per env
app_tracker_entries            — main tracking rows (app+env+version)
app_tracker_signoffs           — sign-off records per entry per role
app_tracker_import_log         — audit log of CSV/XLSX imports
```

---

### Table 1: app_tracker_roles

Defines all stakeholder roles. Supports parent-child nesting for grouped roles (e.g. Platform Team → Infra/DevOps/SRE).

```sql
CREATE TABLE IF NOT EXISTS app_tracker_roles (
  id              SERIAL PRIMARY KEY,
  role_name       TEXT NOT NULL,
  parent_role_id  INTEGER REFERENCES app_tracker_roles(id) ON DELETE SET NULL,
  scope           TEXT NOT NULL DEFAULT 'global',   -- 'global' | 'app-family-group'
  scope_value     TEXT,                             -- e.g. 'FinTech-Group' if scope != global
  sort_order      INTEGER NOT NULL DEFAULT 0,
  created_by      TEXT,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

**Seed data (default roles):**

```
id | role_name        | parent_role_id | scope
1  | Product Team     | NULL           | global
2  | Application Arch | NULL           | global
3  | Platform Arch    | NULL           | global
4  | Dev              | NULL           | global
5  | QA               | NULL           | global
6  | Security         | NULL           | global
7  | Red Team         | 6              | global   ← sub of Security
8  | Blue Team        | 6              | global   ← sub of Security
9  | Info Sec         | 6              | global   ← sub of Security
10 | Platform Team    | NULL           | global
11 | Infra            | 10             | global   ← sub of Platform Team
12 | DevOps           | 10             | global   ← sub of Platform Team
13 | SRE              | 10             | global   ← sub of Platform Team
```

> **Note:** Parent roles (Security, Platform Team) never have direct sign-offs — their status is always derived from their children. The schema supports unlimited nesting depth for future extensibility.

---

### Table 2: app_tracker_role_env_config

Controls which roles are required in which environments. Default is `required = true` for all roles in all envs unless explicitly overridden.

```sql
CREATE TABLE IF NOT EXISTS app_tracker_role_env_config (
  id          SERIAL PRIMARY KEY,
  role_id     INTEGER NOT NULL REFERENCES app_tracker_roles(id) ON DELETE CASCADE,
  env         TEXT NOT NULL,   -- 'dev' | 'uat' | 'stg' | 'box' | 'prd'
  required    BOOLEAN NOT NULL DEFAULT true,
  updated_by  TEXT,
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE(role_id, env)
);
```

**Default configuration (platform team manages this via UI matrix):**

```
Role              | dev | uat | stg | box | prd
Product Team      |  ✅  |  ✅  |  ✅  |  ✅  |  ✅
Application Arch  |  ✅  |  ✅  |  ✅  |  ✅  |  ✅
Platform Arch     |  ✅  |  ✅  |  ✅  |  ✅  |  ✅
Dev               |  ✅  |  ✅  |  ✅  |  ✅  |  ✅
QA                |  ❌  |  ✅  |  ✅  |  ✅  |  ✅
Security (parent) |  N/A |  N/A |  N/A |  N/A |  N/A  ← derived from children
  Red Team        |  ❌  |  ❌  |  ✅  |  ✅  |  ✅
  Blue Team       |  ❌  |  ❌  |  ✅  |  ✅  |  ✅
  Info Sec        |  ❌  |  ❌  |  ✅  |  ✅  |  ✅
Platform Team     |  N/A |  N/A |  N/A |  N/A |  N/A  ← derived from children
  Infra           |  ✅  |  ✅  |  ✅  |  ✅  |  ✅
  DevOps          |  ✅  |  ✅  |  ✅  |  ✅  |  ✅
  SRE             |  ❌  |  ❌  |  ✅  |  ✅  |  ✅
```

---

### Table 3: app_tracker_entries

The main tracking table. One row per app + env + release_version.

```sql
CREATE TABLE IF NOT EXISTS app_tracker_entries (
  id                SERIAL PRIMARY KEY,

  -- Hierarchy
  category          TEXT NOT NULL,
  sub_category      TEXT NOT NULL,
  app_family_group  TEXT NOT NULL,
  app_family        TEXT NOT NULL,
  app               TEXT NOT NULL,

  -- Ownership
  business_owner    TEXT,
  tech_owner        TEXT,
  dev_team          TEXT,
  platform_contact  TEXT,

  -- Delivery
  env               TEXT NOT NULL,   -- 'dev' | 'uat' | 'stg' | 'box' | 'prd'
  release_version   TEXT NOT NULL,
  priority          TEXT NOT NULL DEFAULT 'Medium',
                                     -- 'Critical' | 'High' | 'Medium' | 'Low'
  start_date        DATE,
  target_date       DATE,
  delivered_date    DATE,

  -- Status
  manual_status     TEXT,            -- set by platform team (overrides auto)
  auto_status       TEXT NOT NULL DEFAULT 'Not Started',
                                     -- computed from sign-offs
  remarks           TEXT,

  -- Audit
  created_by        TEXT,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),

  UNIQUE(app, env, release_version)  -- upsert key
);
```

**display_status** is a computed value (not stored):
```
display_status = manual_status  (if set and not NULL)
               = auto_status    (otherwise)
```

**Status values:**
```
Not Started   — no sign-offs started
In Progress   — some sign-offs done, not all
Delivered     — all required sign-offs approved
Blocked       — at least one required sign-off rejected
Rolled Back   — manually set by platform team
```

**Priority values (per app-per-env):**
```
Critical | High | Medium | Low
```

---

### Table 4: app_tracker_signoffs

Child table. One row per assignee per role per entry. Min 1 assignee per role when a role is added.

```sql
CREATE TABLE IF NOT EXISTS app_tracker_signoffs (
  id              SERIAL PRIMARY KEY,
  entry_id        INTEGER NOT NULL REFERENCES app_tracker_entries(id) ON DELETE CASCADE,
  role_id         INTEGER NOT NULL REFERENCES app_tracker_roles(id) ON DELETE CASCADE,
  assignee_name   TEXT NOT NULL,
  assignee_email  TEXT,
  status          TEXT NOT NULL DEFAULT 'Pending',
                                -- 'Pending' | 'Approved' | 'Rejected'
  signoff_date    TIMESTAMPTZ,
  remarks         TEXT,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

**Sign-off status values:**
```
Pending   — awaiting action
Approved  — signed off
Rejected  — rejected (entry becomes Blocked)
```

---

### Table 5: app_tracker_import_log

Audit trail for every CSV/XLSX import.

```sql
CREATE TABLE IF NOT EXISTS app_tracker_import_log (
  id            SERIAL PRIMARY KEY,
  imported_by   TEXT,
  file_name     TEXT,
  row_count     INTEGER,
  inserted      INTEGER,
  updated       INTEGER,
  failed        INTEGER,
  status        TEXT NOT NULL DEFAULT 'success',
  error_message TEXT,
  imported_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## 3. Role Hierarchy & Sign-off Chain

### Full Hierarchy

```
Product Team          (standalone — direct sign-off)
Application Arch      (standalone — direct sign-off)
Platform Arch         (standalone — direct sign-off)
Dev                   (standalone — direct sign-off)
QA                    (standalone — direct sign-off)
Security              (parent — derived from children)
  ├── Red Team        (direct sign-off)
  ├── Blue Team       (direct sign-off)
  └── Info Sec        (direct sign-off)
Platform Team         (parent — derived from children)
  ├── Infra           (direct sign-off)
  ├── DevOps          (direct sign-off)
  └── SRE             (direct sign-off)
```

### Sign-off Panel — Per Entry

```
Role              | Assignee(s)       | Status      | Date       | Remarks
─────────────────────────────────────────────────────────────────────────────
Product Team      | Sarah K           | ✅ Approved  | 2026-03-05 | LGTM
Application Arch  | Raj M             | ✅ Approved  | 2026-03-06 | -
Platform Arch     | Nantha P          | ⏳ Pending   | -          | -
Dev               | Kumar S           | ✅ Approved  | 2026-03-04 | -
QA                | Priya R           | ⏳ Pending   | -          | -
Security ─────────────────────────────────────────────────────────────────
  Red Team        | Alex T            | N/A (dev)   | -          | -
  Blue Team       | -                 | N/A (dev)   | -          | -
  Info Sec        | -                 | N/A (dev)   | -          | -
Platform Team ────────────────────────────────────────────────────────────
  Infra           | Nantha P          | ✅ Approved  | 2026-03-07 | -
  DevOps          | Arun K            | ⏳ Pending   | -          | -
  SRE             | -                 | N/A (dev)   | -          | -
```

---

## 4. Auto-Status Derivation Logic

### Leaf Role Status (roles with no children)

```
No assignees assigned           → Not Started
All assignees Approved          → Approved
Any assignee  Rejected          → Rejected
Mix Pending + Approved          → In Progress
```

### Parent Role Status (Security, Platform Team)

```
All required children Approved  → Approved
Any required child   Rejected   → Rejected
Mix                             → In Progress
All Pending / no children       → Not Started
```

### Entry auto_status Calculation

```
Step 1: Resolve required roles for this entry's env
        (look up app_tracker_role_env_config for each leaf role)

Step 2: For each required role — compute leaf status (above)
        For N/A roles (required=false for this env) — skip

Step 3: Compute parent role status from their required children

Step 4: Compute entry auto_status:

  All required roles Approved   → Delivered
  Any required role  Rejected   → Blocked
  Mix Pending + Approved        → In Progress
  All Pending / none assigned   → Not Started

Step 5: display_status = manual_status ?? auto_status
```

### Recalculation Triggers

Auto-status recalculates automatically whenever:
- A sign-off is added, updated, or deleted
- A role-env config changes (required flag toggled)
- A new entry is created

---

## 5. API Design

All endpoints under `/api/app-tracker/`

### Entries

```
GET    /entries                       List all entries (with filters)
GET    /entries/:id                   Get single entry with sign-offs
POST   /entries                       Create new entry
PUT    /entries/:id                   Update entry
DELETE /entries/:id                   Delete entry
GET    /entries/summary               Heatmap data (app × env → display_status)
```

**Query params for GET /entries:**
```
?category=Banking
?app_family_group=FinTech-Group
?app_family=Payments
?env=dev
?status=Blocked
?priority=Critical
?release_version=v1.2.0
```

---

### Sign-offs

```
GET    /entries/:id/signoffs          List sign-offs for an entry
POST   /entries/:id/signoffs          Add sign-off record
PUT    /signoffs/:id                  Update sign-off (approve/reject/remarks)
DELETE /signoffs/:id                  Remove sign-off record
```

**POST /entries/:id/signoffs body:**
```json
{
  "role_id": 4,
  "assignee_name": "Priya R",
  "assignee_email": "priya@company.com",
  "status": "Pending"
}
```

**PUT /signoffs/:id body (approve action):**
```json
{
  "status": "Approved",
  "signoff_date": "2026-03-10T10:00:00Z",
  "remarks": "All tests passed"
}
```

---

### Roles

```
GET    /roles                         List all roles (with children nested)
POST   /roles                         Create role (platform team only)
PUT    /roles/:id                     Update role
DELETE /roles/:id                     Delete role
```

---

### Role-Env Config

```
GET    /roles/env-config              Full matrix (all roles × all envs)
PUT    /roles/env-config              Bulk update matrix (platform team only)
```

**PUT /roles/env-config body:**
```json
{
  "updates": [
    { "role_id": 13, "env": "dev", "required": false },
    { "role_id": 13, "env": "uat", "required": false },
    { "role_id": 13, "env": "stg", "required": true }
  ]
}
```

---

### Import

```
POST   /import                        Bulk upsert from CSV/XLSX
GET    /import/logs                   Import audit log
```

**POST /import body:** `multipart/form-data` with file attachment

**Upsert key:** `app + env + release_version`
- Match found → overwrite all fields
- No match → insert new row

---

### Users (Keycloak proxy)

```
GET    /users                         List Keycloak-synced users for assignee picker
GET    /users?search=priya            Search by name or email
```

Proxies to Backstage catalog API: `GET /api/catalog/entities?filter=kind=User`

---

## 6. Frontend Design

### Page: `/app-tracker`

Accessible from Backstage sidebar under a new **"App Tracker"** nav item.

---

### Tab 1 — Heatmap (Status Overview)

**Purpose:** Instant org-wide delivery status at a glance.

**Layout:**
```
Filter bar: [Category ▼] [App-Family-Group ▼] [App-Family ▼] [Priority ▼] [Version ▼]

                  dev        uat        stg        box        prd
payment-service   🟢          🟡          ⬜          ⬜          ⬜
auth-service      🟢          🔴          ⬜          ⬜          ⬜
ui-portal         🟢          🟢          🟡          ⬜          ⬜
infra-agent       🟢          🟢          🟢          🟡          ⬜
```

**Cell colours:**
```
Not Started   → ⬜ Grey    (#9E9E9E)
In Progress   → 🟡 Blue    (#1976D2)
Delivered     → 🟢 Green   (#2E7D32)
Blocked       → 🔴 Red     (#C62828)
Rolled Back   → 🟠 Amber   (#E65100)
N/A           → ░░ Light grey (striped)
```

**Cell click** → opens slide-out Sign-off Panel for that app+env+version entry

**Latest version per app+env** is shown by default. Version selector available per cell.

---

### Tab 2 — Detailed Table

**Purpose:** Full data view with all 17 columns, inline editing, sign-off expansion.

**Columns displayed:**
```
Category | Sub-Category | App-Family-Group | App-Family | App |
Business Owner | Tech Owner | Dev Team | Platform Contact |
Env | Release Version | Priority | Start Date | Target Date |
Delivered Date | Status | Remarks | Actions
```

**Features:**
- Paginated (20 rows per page)
- Sort by any column
- Filter bar (same as heatmap tab)
- **Expand row (▶)** → inline sign-off panel
- **Edit button** → modal form for all fields
- **Add Entry button** → new entry modal
- **Delete button** (with confirmation)
- Status column shows `display_status` with colour badge

---

### Tab 3 — Import

**Purpose:** Bulk upload entries from CSV or XLSX.

**Flow:**
```
1. Drag & drop or click to upload CSV/XLSX file
2. Preview table shows first 10 rows with column mapping
3. Validation errors highlighted (missing required fields, invalid values)
4. Confirm import button
5. Progress indicator
6. Result summary: "X rows inserted, Y rows updated, Z rows failed"
7. Import log history table below
```

**CSV/XLSX column mapping (header names expected):**
```
category, sub_category, app_family_group, app_family, app,
business_owner, tech_owner, dev_team, platform_contact,
env, release_version, priority, start_date, target_date,
delivered_date, manual_status, remarks
```

**Upsert behaviour:** `app + env + release_version` match → overwrite all fields

---

### Tab 4 — Configure Roles

**Purpose:** Platform team manages role definitions and env-requirement matrix.

**Section 1 — Role List**
```
Role Name         | Parent       | Scope          | Actions
Product Team      | -            | Global         | Edit | Delete
Application Arch  | -            | Global         | Edit | Delete
Platform Arch     | -            | Global         | Edit | Delete
Dev               | -            | Global         | Edit | Delete
QA                | -            | Global         | Edit | Delete
Security          | -            | Global         | Edit | Delete
  Red Team        | Security     | Global         | Edit | Delete
  Blue Team       | Security     | Global         | Edit | Delete
  Info Sec        | Security     | Global         | Edit | Delete
Platform Team     | -            | Global         | Edit | Delete
  Infra           | Platform Team| Global         | Edit | Delete
  DevOps          | Platform Team| Global         | Edit | Delete
  SRE             | Platform Team| Global         | Edit | Delete

[+ Add Role]
```

**Section 2 — Env Requirement Matrix**

Checkbox grid — platform team toggles required/not-required per role per env:

```
Role              | dev | uat | stg | box | prd
Product Team      |  ✅  |  ✅  |  ✅  |  ✅  |  ✅
Application Arch  |  ✅  |  ✅  |  ✅  |  ✅  |  ✅
Platform Arch     |  ✅  |  ✅  |  ✅  |  ✅  |  ✅
Dev               |  ✅  |  ✅  |  ✅  |  ✅  |  ✅
QA                |  ❌  |  ✅  |  ✅  |  ✅  |  ✅
Security          |  -   |  -   |  -   |  -   |  -   (derived)
  Red Team        |  ❌  |  ❌  |  ✅  |  ✅  |  ✅
  Blue Team       |  ❌  |  ❌  |  ✅  |  ✅  |  ✅
  Info Sec        |  ❌  |  ❌  |  ✅  |  ✅  |  ✅
Platform Team     |  -   |  -   |  -   |  -   |  -   (derived)
  Infra           |  ✅  |  ✅  |  ✅  |  ✅  |  ✅
  DevOps          |  ✅  |  ✅  |  ✅  |  ✅  |  ✅
  SRE             |  ❌  |  ❌  |  ✅  |  ✅  |  ✅

[Save Matrix]
```

Changes to the matrix immediately trigger auto-status recalculation for all affected entries.

---

### Sign-off Panel (Shared Component)

Appears as a slide-out drawer from Tab 1 (cell click) and Tab 2 (row expand).

**Header:**
```
payment-service  |  dev  |  v1.2.0
Auto Status: In Progress  |  Manual Override: —
Priority: High  |  Target: 2026-03-15
```

**Sign-off table:**
```
Role              | Assignee          | Status     | Date       | Remarks      | Actions
Product Team      | Sarah K           | ✅ Approved | 2026-03-05 | LGTM         | Edit
Application Arch  | Raj M             | ✅ Approved | 2026-03-06 | -            | Edit
Platform Arch     | Nantha P          | ⏳ Pending  | -          | -            | Approve | Reject
Dev               | Kumar S           | ✅ Approved | 2026-03-04 | -            | Edit
QA                | Priya R           | ⏳ Pending  | -          | -            | Approve | Reject
                  | [+ Add Assignee]  |            |            |              |
Security ─────────────────────────────────────────────────────────────────────────────────
  Red Team        | —                 | N/A        | -          | Not req. dev | -
  Blue Team       | —                 | N/A        | -          | Not req. dev | -
  Info Sec        | —                 | N/A        | -          | Not req. dev | -
Platform Team ────────────────────────────────────────────────────────────────────────────
  Infra           | Nantha P          | ✅ Approved | 2026-03-07 | -            | Edit
  DevOps          | Arun K            | ⏳ Pending  | -          | -            | Approve | Reject
  SRE             | —                 | N/A        | -          | Not req. dev | -
```

**Assignee picker** → searches Keycloak-synced users from Backstage catalog
**Manual status override** → dropdown at top of panel (platform team only)

---

## 7. Import Specification

### Supported Formats

- CSV (UTF-8, comma-separated)
- XLSX (first sheet used)

### Expected Column Headers

```
category          (required)
sub_category      (required)
app_family_group  (required)
app_family        (required)
app               (required)
env               (required) — must be one of: dev, uat, stg, box, prd
release_version   (required)
priority          (optional) — default: Medium
business_owner    (optional)
tech_owner        (optional)
dev_team          (optional)
platform_contact  (optional)
start_date        (optional) — format: YYYY-MM-DD
target_date       (optional) — format: YYYY-MM-DD
delivered_date    (optional) — format: YYYY-MM-DD
manual_status     (optional)
remarks           (optional)
```

### Upsert Logic

```
For each row in the file:
  1. Look up existing entry WHERE app = ? AND env = ? AND release_version = ?
  2. If found → UPDATE all fields with file values (overwrite)
  3. If not found → INSERT new row
  4. Recalculate auto_status for all affected entries
```

### Validation Rules

```
env must be one of: dev, uat, stg, box, prd
priority must be one of: Critical, High, Medium, Low (if provided)
manual_status must be one of: Not Started, In Progress, Delivered, Blocked, Rolled Back (if provided)
dates must be valid YYYY-MM-DD format (if provided)
app + env + release_version combination is the upsert key
```

---

## 8. File Structure

```
backstage/
  packages/
    backend/
      src/
        plugins/
          └── appTracker.ts              ← backend plugin (store + API)
        index.ts                         ← register appTrackerPlugin

    app/
      src/
        components/
          AppTracker/
            ├── AppTrackerPage.tsx        ← main page, 4 tabs, router
            ├── HeatmapTab.tsx            ← env status grid
            ├── DetailedTableTab.tsx      ← full table with expand + edit
            ├── ImportTab.tsx             ← CSV/XLSX upload + log
            ├── ConfigureRolesTab.tsx     ← role list + env matrix
            ├── SignoffPanel.tsx          ← shared slide-out panel
            ├── hooks/
            │     ├── useEntries.ts       ← API calls for entries
            │     ├── useSignoffs.ts      ← API calls for sign-offs
            │     └── useRoles.ts         ← API calls for roles + config
            ├── types.ts                  ← TypeScript interfaces
            └── index.ts                  ← exports

        plugin.ts                         ← Backstage plugin registration
        App.tsx                           ← add route for /app-tracker
```

---

## 9. Implementation Steps

### Step 1 — Backend Plugin (appTracker.ts)

- `AppTrackerStore` class with Knex
- `migrate()` method — creates all 5 tables + indexes on startup
- Seed default roles and env config if tables are empty
- All CRUD methods for entries, signoffs, roles, role-env-config
- `recalculateAutoStatus(entryId)` — called after every signoff change
- Express router with all API endpoints
- `addAuthPolicy` for any public endpoints
- Register in `index.ts`

### Step 2 — Frontend Plugin Registration

- Create `plugin.ts` in `packages/app/src`
- Add route `/app-tracker` in `App.tsx`
- Add nav item in `Root.tsx` sidebar

### Step 3 — HeatmapTab

- Fetch `/api/app-tracker/entries/summary`
- Build app × env grid
- Colour cells by `display_status`
- Filter bar (category, app-family-group, priority)
- Cell click → open SignoffPanel

### Step 4 — DetailedTableTab

- Fetch `/api/app-tracker/entries` with pagination
- Sortable, filterable columns
- Expand row → SignoffPanel
- Edit modal → PUT `/api/app-tracker/entries/:id`
- Add entry modal → POST `/api/app-tracker/entries`

### Step 5 — SignoffPanel

- Fetch `/api/app-tracker/entries/:id/signoffs`
- Fetch `/api/app-tracker/roles` (for hierarchy)
- Fetch `/api/app-tracker/roles/env-config` (for N/A determination)
- User search → `GET /api/app-tracker/users?search=`
- Approve/Reject buttons → PUT `/api/app-tracker/signoffs/:id`
- Add assignee → POST `/api/app-tracker/entries/:id/signoffs`

### Step 6 — ImportTab

- File upload (CSV/XLSX) → POST `/api/app-tracker/import`
- Column preview table
- Validation error display
- Result summary
- Import log → GET `/api/app-tracker/import/logs`

### Step 7 — ConfigureRolesTab

- Role list with parent-child tree → GET `/api/app-tracker/roles`
- Add/Edit/Delete role modals
- Env requirement matrix checkboxes → GET/PUT `/api/app-tracker/roles/env-config`
- Save triggers auto-status recalculation for all entries

### Step 8 — Rebuild & Deploy

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
  -e KEYCLOAK_CLIENT_SECRET=YOUR_SECRET \
  -e GITLAB_TOKEN=YOUR_GITLAB_TOKEN \
  backstage:latest

docker logs backstage | grep -i "app-tracker"
# Expected:
# AppTracker: Database initialized
# AppTracker: Default roles seeded
# AppTracker: API listening at /api/app-tracker
```

---

## 10. Quick Reference

### DB Tables

```
app_tracker_roles              — role definitions (with parent_role_id for nesting)
app_tracker_role_env_config    — required flag per role per env
app_tracker_entries            — main rows (upsert key: app + env + release_version)
app_tracker_signoffs           — sign-off records per entry per role per assignee
app_tracker_import_log         — import audit trail
```

### Entry Columns (17)

```
category | sub_category | app_family_group | app_family | app |
business_owner | tech_owner | dev_team | platform_contact |
env | release_version | priority | start_date | target_date |
delivered_date | manual_status | auto_status | remarks
```

### Status Values

```
Not Started   — no sign-offs started
In Progress   — some approved, not all
Delivered     — all required sign-offs approved
Blocked       — at least one required sign-off rejected
Rolled Back   — manually set
```

### Priority Values

```
Critical | High | Medium | Low    (per app-per-env row)
```

### Environments (in order)

```
dev → uat → stg → box → prd
```

### Role Hierarchy

```
Product Team          (standalone)
Application Arch      (standalone)
Platform Arch         (standalone)
Dev                   (standalone)
QA                    (standalone)
Security              (parent)
  ├── Red Team
  ├── Blue Team
  └── Info Sec
Platform Team         (parent)
  ├── Infra
  ├── DevOps
  └── SRE
```

### Upsert Key (Import)

```
app + env + release_version
```

### API Base URL

```
/api/app-tracker/
```

### Frontend Route

```
/app-tracker
```

### Verification Commands

```bash
# Check plugin started
docker logs backstage | grep -i "app-tracker"

# Get token
TOKEN=$(curl -sk "https://localhost/api/auth/guest/refresh" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['backstageIdentity']['token'])")

# List all entries
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/app-tracker/entries" | python3 -m json.tool

# Get heatmap summary
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/app-tracker/entries/summary" | python3 -m json.tool

# List roles
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://localhost/api/app-tracker/roles" | python3 -m json.tool
```

---

## Implementation Status

| Phase | Task | Status |
|---|---|---|
| Design | Data model finalised | ✅ Complete |
| Design | Role hierarchy defined | ✅ Complete |
| Design | API design complete | ✅ Complete |
| Design | UI design complete | ✅ Complete |
| Build | Backend plugin (appTracker.ts) | ⏳ Pending |
| Build | Frontend — AppTrackerPage.tsx | ⏳ Pending |
| Build | Frontend — HeatmapTab.tsx | ⏳ Pending |
| Build | Frontend — DetailedTableTab.tsx | ⏳ Pending |
| Build | Frontend — SignoffPanel.tsx | ⏳ Pending |
| Build | Frontend — ImportTab.tsx | ⏳ Pending |
| Build | Frontend — ConfigureRolesTab.tsx | ⏳ Pending |
| Build | Plugin registration + routing | ⏳ Pending |
| Deploy | Rebuild & deploy | ⏳ Pending |
| Test | Verification & testing | ⏳ Pending |

---

*Generated: March 2026 | Backstage v1.48.0 | Node.js v24.13.1 | Version 1.0 — App Tracker Implementation Plan*
