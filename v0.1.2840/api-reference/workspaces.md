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
| GET | `/api/v1/workspaces/{workspace_id}` | Get workspace details |

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
