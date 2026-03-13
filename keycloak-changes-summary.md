# Keycloak Integration Changes Summary

> Changes implemented on top of the base Backstage + Keycloak setup  
> **Date:** March 2026

---

## Table of Contents

1. [Overview](#1-overview)
2. [Change 1: Custom Keycloak Provider (Subgroup Hierarchy)](#2-change-1-custom-keycloak-provider-subgroup-hierarchy)
3. [Change 2: Group Type Differentiation via Keycloak Attributes](#3-change-2-group-type-differentiation-via-keycloak-attributes)
4. [Change 3: User DisplayName Fallback Fix](#4-change-3-user-displayname-fallback-fix)
5. [Final File Reference](#5-final-file-reference)
6. [Troubleshooting](#6-troubleshooting)

---

## 1. Overview

After the base Backstage + Keycloak setup, three additional changes were implemented to handle real-world scenarios:

| # | Change | Reason |
|---|--------|--------|
| 1 | Custom Keycloak Provider | Official plugin doesn't support subgroups in Keycloak 23+ |
| 2 | Group type differentiation | Differentiate teams, orgs, and rbac groups in Backstage catalog |
| 3 | User displayName fallback | Users without firstName/lastName failed Backstage validation |

---

## 2. Change 1: Custom Keycloak Provider (Subgroup Hierarchy)

### Problem

The official `@backstage-community/plugin-catalog-backend-module-keycloak` plugin only fetches top-level groups. Keycloak 23+ no longer embeds subgroups in the default `/groups` API response — `subGroups` is always returned as `[]`.

**Diagnosis:**
```bash
# Keycloak API returns subGroupCount: 3 but subGroups: []
curl -s -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8080/keycloak/admin/realms/backstage/groups?briefRepresentation=false" \
  | python3 -m json.tool
```

```json
{
  "name": "platform-common-team",
  "subGroupCount": 3,
  "subGroups": []   ← always empty in Keycloak 23+
}
```

### Solution

Created a custom `KeycloakCustomProvider` class that explicitly calls the `/groups/{id}/children` API for each parent group.

**New file:** `packages/backend/src/keycloakProvider.ts`

**Key logic:**
```typescript
private async fetchAllGroups(token: string): Promise<KeycloakGroup[]> {
  // Fetch top-level groups
  const response = await fetch(
    `${this.baseUrl}/admin/realms/${this.realm}/groups?max=500&briefRepresentation=false`,
    { headers: { Authorization: `Bearer ${token}` } },
  );
  const topGroups: KeycloakGroup[] = await response.json();
  const allGroups: KeycloakGroup[] = [];

  for (const group of topGroups) {
    allGroups.push(group);
    // ← Explicitly fetch subgroups via /children API
    if (group.subGroupCount && group.subGroupCount > 0) {
      const subResponse = await fetch(
        `${this.baseUrl}/admin/realms/${this.realm}/groups/${group.id}/children?max=500&briefRepresentation=false`,
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
```

### Changes to index.ts

Removed the official plugin and registered the custom module instead:

```typescript
// ❌ Removed
backend.add(import('@backstage-community/plugin-catalog-backend-module-keycloak'));

// ✅ Added
backend.add(keycloakCustomModule);
```

### Result

```
platform-common-team  (organization)
    ├── infra-team    (team)
    ├── devops-team   (team)
    └── sre-team      (team)
```

---

## 3. Change 2: Group Type Differentiation via Keycloak Attributes

### Problem

All groups in Backstage were showing as `type: team` by default. There was no way to differentiate:
- Org-level parent groups
- Actual teams
- RBAC access control groups

### Solution

**Step 1: Add `backstage-type` attribute in Keycloak**

For each group go to **Groups → (group) → Attributes tab** and add:

| Group | Attribute Key | Attribute Value |
|-------|--------------|-----------------|
| `platform-common-team` | `backstage-type` | `organization` |
| `infra-team` | `backstage-type` | `team` |
| `devops-team` | `backstage-type` | `team` |
| `sre-team` | `backstage-type` | `team` |
| `backstage-admins` | `backstage-type` | `rbac-group` |
| `backstage-developers` | `backstage-type` | `rbac-group` |
| `backstage-viewers` | `backstage-type` | `rbac-group` |

**Step 2: Update KeycloakGroup interface**

```typescript
// ← Added attributes field
interface KeycloakGroup {
  id: string;
  name: string;
  path: string;
  subGroupCount?: number;
  parentId?: string;
  attributes?: Record<string, string[]>;  // ← New
}
```

**Step 3: Add getGroupType() helper**

```typescript
// ← New helper method
private getGroupType(group: KeycloakGroup): string {
  const attr = group.attributes?.['backstage-type'];
  if (attr && attr.length > 0) {
    return attr[0];
  }
  return 'team'; // default fallback
}
```

**Step 4: Use in group entity creation**

```typescript
const groupType = this.getGroupType(group); // ← reads backstage-type attribute

return {
  spec: {
    type: groupType, // ← organization | team | rbac-group
    ...
  }
}
```

**Step 5: Add `briefRepresentation=false` to API calls**

Required to include attributes in the Keycloak API response:

```typescript
// ← briefRepresentation=false added to both calls
`/groups?max=500&briefRepresentation=false`
`/groups/${group.id}/children?max=500&briefRepresentation=false`
```

### Result

Backstage catalog now shows groups with correct types:

| Group | Type |
|-------|------|
| `platform-common-team` | `organization` |
| `infra-team` | `team` |
| `devops-team` | `team` |
| `sre-team` | `team` |
| `backstage-admins` | `rbac-group` |
| `backstage-developers` | `rbac-group` |
| `backstage-viewers` | `rbac-group` |

---

## 4. Change 3: User DisplayName Fallback Fix

### Problem

Users created in Keycloak without `firstName` or `lastName` failed Backstage entity validation:

```
TypeError: /spec/profile/displayName must NOT have fewer than 1 characters - limit: 1
```

Affected users: `infra-user1`, `demo-user1`, `nanthagopal`, `sre-user1`

### Root Cause

The displayName was computed as:
```typescript
displayName: `${user.firstName ?? ''} ${user.lastName ?? ''}`.trim()
// Results in "" (empty string) when both are missing → fails validation
```

### Fix

Added `|| user.username` fallback:

```typescript
// ❌ Before
displayName: `${user.firstName ?? ''} ${user.lastName ?? ''}`.trim()

// ✅ After
displayName: `${user.firstName ?? ''} ${user.lastName ?? ''}`.trim() || user.username
```

This ensures `displayName` always has a value — falling back to the Keycloak username when no name is set.

### Result

No more validation errors. All users are now ingested successfully with `displayName` set to either their full name or username.

---

## 5. Final File Reference

### packages/backend/src/keycloakProvider.ts (complete)

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
  attributes?: Record<string, string[]>;
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

  private getGroupType(group: KeycloakGroup): string {
    const attr = group.attributes?.['backstage-type'];
    if (attr && attr.length > 0) {
      return attr[0];
    }
    return 'team';
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
    const response = await fetch(
      `${this.baseUrl}/admin/realms/${this.realm}/groups?max=500&briefRepresentation=false`,
      { headers: { Authorization: `Bearer ${token}` } },
    );
    const topGroups: KeycloakGroup[] = await response.json();
    const allGroups: KeycloakGroup[] = [];

    for (const group of topGroups) {
      allGroups.push(group);
      if (group.subGroupCount && group.subGroupCount > 0) {
        const subResponse = await fetch(
          `${this.baseUrl}/admin/realms/${this.realm}/groups/${group.id}/children?max=500&briefRepresentation=false`,
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

  private async fetchUserGroups(
    token: string,
    userId: string,
  ): Promise<KeycloakGroup[]> {
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

    const groupEntities: GroupEntity[] = allGroups.map(group => {
      const pathParts = group.path.split('/').filter(Boolean);
      const parentName =
        pathParts.length > 1 ? pathParts[pathParts.length - 2] : undefined;
      const groupType = this.getGroupType(group);

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
          type: groupType,
          profile: {},
          children: allGroups
            .filter(g => g.parentId === group.id)
            .map(g => g.name),
          ...(parentName ? { parent: parentName } : {}),
          members: [],
        },
      };
    });

    const userEntities: UserEntity[] = [];
    for (const user of users) {
      const userGroups = await this.fetchUserGroups(token, user.id);
      const memberOf = userGroups.map(g => g.name);

      for (const g of userGroups) {
        const groupEntity = groupEntities.find(
          ge => ge.metadata.name === g.name,
        );
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
            displayName:
              `${user.firstName ?? ''} ${user.lastName ?? ''}`.trim() ||
              user.username,
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

---

## 6. Troubleshooting

### Conflict warnings between two providers

**Symptom:**
```
Source KeycloakCustomProvider detected conflicting entityRef group:default/backstage-admins
already referenced by keycloak-org-provider:default
```

**Fix:** Remove the official plugin from `index.ts`:
```typescript
// Remove this line
backend.add(import('@backstage-community/plugin-catalog-backend-module-keycloak'));
```

---

### Groups showing as `team` instead of correct type

**Symptom:** All groups show `type: team` even after adding Keycloak attributes.

**Fix:** Ensure `briefRepresentation=false` is in the API calls:
```typescript
`/groups?max=500&briefRepresentation=false`
```
Without this, Keycloak does not return the `attributes` field.

---

### Users missing from catalog

**Symptom:**
```
TypeError: /spec/profile/displayName must NOT have fewer than 1 characters
```

**Fix:** Add firstName/lastName to the user in Keycloak, or ensure the fallback is in the provider:
```typescript
displayName: `${user.firstName ?? ''} ${user.lastName ?? ''}`.trim() || user.username
```

---

*Generated: March 2026*
