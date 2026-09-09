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
  "pagination": {
    "count": 1,
    "hasMore": true,
    "nextCursor": "string"
  },
  "workspaces": [
    {
      "createdAt": "2024-01-15T10:30:00Z",
      "description": "string",
      "id": "string",
      "modifiedAt": "2024-01-15T10:30:00Z",
      "name": "string",
      "owner": "string"
    }
  ]
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
| `description` | string | No | - |
| `name` | string | Yes | - |

```json
{
  "description": "string",
  "name": "string"
}
```

### Responses

**201** - Workspace created

```json
{
  "createdAt": "2024-01-15T10:30:00Z",
  "description": "string",
  "id": "string",
  "modifiedAt": "2024-01-15T10:30:00Z",
  "name": "string",
  "owner": "string",
  "stats": {
    "activeSimulations": 1,
    "configurationCount": 1,
    "simulationCount": 1
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
  "createdAt": "2024-01-15T10:30:00Z",
  "description": "string",
  "id": "string",
  "modifiedAt": "2024-01-15T10:30:00Z",
  "name": "string",
  "owner": "string",
  "stats": {
    "activeSimulations": 1,
    "configurationCount": 1,
    "simulationCount": 1
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
| `description` | string | No | - |
| `name` | string | Yes | - |

```json
{
  "description": "string",
  "name": "string"
}
```

### Responses

**200** - Workspace updated

```json
{
  "createdAt": "2024-01-15T10:30:00Z",
  "description": "string",
  "id": "string",
  "modifiedAt": "2024-01-15T10:30:00Z",
  "name": "string",
  "owner": "string",
  "stats": {
    "activeSimulations": 1,
    "configurationCount": 1,
    "simulationCount": 1
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
  "created": true,
  "legacyProjectId": "550e8400-e29b-41d4-a716-446655440000",
  "workspaceId": "workspace_abc123"
}
```

**400** - Bad request

**401** - Unauthorized

**403** - Forbidden — requires service (M2M) token

**404** - Workspace not found

**409** - Legacy project ID already set to a different value

**500** - Internal server error

---
