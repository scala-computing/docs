---
title: "Workspaces"
description: "Workspace management"
---

Workspace management

{/* AUTO-GENERATED CONTENT BELOW - DO NOT EDIT MANUALLY */}
{/* Generated from OpenAPI spec by generate-api-reference.py */}

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/workspaces` | List workspaces |
| POST | `/api/v1/workspaces` | Create workspace |
| GET | `/api/v1/workspaces/{workspace_id}` | Get workspace details |
| PATCH | `/api/v1/workspaces/{workspace_id}` | Update workspace |
| DELETE | `/api/v1/workspaces/{workspace_id}` | Delete workspace |
| POST | `/api/v1/workspaces/{workspace_id}/legacy-project` | Set legacy project ID for workspace |

---

## List workspaces

<span class="api-method api-method-get">GET</span> `/api/v1/workspaces`

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `limit` | query | integer | No | - |
| `next` | query | string | No | Opaque cursor token to fetch results after a specific item |
| `owner` | query | string | No | Filter by owner user ID |
| `search` | query | string | No | Full-text search across name and description fields (case-insensitive) |
| `createdAfter` | query | string | No | Filter to items created on or after this timestamp (ISO 8601) |
| `createdBefore` | query | string | No | Filter to items created before this timestamp (ISO 8601) |
| `modifiedAfter` | query | string | No | Filter to items modified on or after this timestamp (ISO 8601) |
| `modifiedBefore` | query | string | No | Filter to items modified before this timestamp (ISO 8601) |

### Responses

**200** - List of workspaces

```json
{
  "workspaces": [
    {
      "id": "string",
      "name": "string",
      "description": "string",
      "owner": "string",
      "createdAt": "2024-01-15T10:30:00Z",
      "modifiedAt": "2024-01-15T10:30:00Z"
    }
  ],
  "pagination": {
    "count": 1,
    "hasMore": true,
    "nextCursor": "string"
  }
}
```

**401** - Unauthorized

**500** - Internal server error

---

## Create workspace

<span class="api-method api-method-post">POST</span> `/api/v1/workspaces`

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | - |
| `description` | string | No | - |

```json
{
  "name": "string",
  "description": "string"
}
```

### Responses

**201** - Workspace created

```json
{
  "id": "string",
  "name": "string",
  "description": "string",
  "owner": "string",
  "createdAt": "2024-01-15T10:30:00Z",
  "modifiedAt": "2024-01-15T10:30:00Z",
  "stats": {
    "configurationCount": 1,
    "simulationCount": 1,
    "activeSimulations": 1
  }
}
```

**400** - Bad request

**401** - Unauthorized

**403** - Forbidden — requires service (M2M) token

**500** - Internal server error

---

## Get workspace details

<span class="api-method api-method-get">GET</span> `/api/v1/workspaces/{workspace_id}`

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `workspace_id` | path | string | Yes | - |

### Responses

**200** - Workspace details

```json
{
  "id": "string",
  "name": "string",
  "description": "string",
  "owner": "string",
  "createdAt": "2024-01-15T10:30:00Z",
  "modifiedAt": "2024-01-15T10:30:00Z",
  "stats": {
    "configurationCount": 1,
    "simulationCount": 1,
    "activeSimulations": 1
  }
}
```

**401** - Unauthorized

**404** - Workspace not found

**500** - Internal server error

---

## Update workspace

<span class="api-method api-method-patch">PATCH</span> `/api/v1/workspaces/{workspace_id}`

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `workspace_id` | path | string | Yes | - |
| `If-Match` | header | string | No | ETag for optimistic concurrency (optional in development, required in production) |

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | - |
| `description` | string | No | - |

```json
{
  "name": "string",
  "description": "string"
}
```

### Responses

**200** - Workspace updated

```json
{
  "id": "string",
  "name": "string",
  "description": "string",
  "owner": "string",
  "createdAt": "2024-01-15T10:30:00Z",
  "modifiedAt": "2024-01-15T10:30:00Z",
  "stats": {
    "configurationCount": 1,
    "simulationCount": 1,
    "activeSimulations": 1
  }
}
```

**400** - Bad request

**401** - Unauthorized

**403** - Forbidden — requires service (M2M) token

**404** - Workspace not found

**412** - Precondition failed (ETag mismatch)

**500** - Internal server error

---

## Delete workspace

<span class="api-method api-method-delete">DELETE</span> `/api/v1/workspaces/{workspace_id}`

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `workspace_id` | path | string | Yes | - |

### Responses

**202** - Workspace deletion in progress

```json
{
  "id": "string",
  "status": "deleting"
}
```

**401** - Unauthorized

**403** - Forbidden — requires service (M2M) token

**404** - Workspace not found

**409** - Conflict — the workspace still holds an active simulation that is running platform compute. Deleting a workspace revokes reads of every simulation inside it, including the queries that are the only way to discover a simulation id, so those simulations must be terminated first. Active simulations with no platform link do not block deletion: they have no compute to strand and cannot be terminated.

**500** - Internal server error

---

## Set legacy project ID for workspace

<span class="api-method api-method-post">POST</span> `/api/v1/workspaces/{workspace_id}/legacy-project`

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `workspace_id` | path | string | Yes | - |

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `legacyProjectId` | string | Yes | - |

```json
{
  "legacyProjectId": "550e8400-e29b-41d4-a716-446655440000"
}
```

### Responses

**200** - Legacy project ID set

```json
{
  "workspaceId": "workspace_abc123",
  "legacyProjectId": "550e8400-e29b-41d4-a716-446655440000",
  "created": true
}
```

**400** - Bad request

**401** - Unauthorized

**403** - Forbidden — requires service (M2M) token

**404** - Workspace not found

**409** - Legacy project ID already set to a different value

**500** - Internal server error

---
