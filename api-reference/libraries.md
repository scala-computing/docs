---
title: "Libraries"
description: "Content-addressed trace libraries (feature-gated: enable_trace_routes)"
---

Content-addressed trace libraries (feature-gated: enable_trace_routes)

{/* AUTO-GENERATED CONTENT BELOW - DO NOT EDIT MANUALLY */}
{/* Generated from OpenAPI spec by generate-api-reference.py */}

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/libraries` | List libraries |
| GET | `/api/v1/libraries/{hash}` | Get library by hash |
| GET | `/api/v1/libraries/{id}` | Get library |

---

## List libraries

<span class="api-method api-method-get">GET</span> `/api/v1/libraries`

List all libraries accessible in the workspace, including global libraries.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `workspaceId` | query | string | No | Workspace ID to scope the query. If not provided, returns only global libraries. |
| `scope` | query | string | No | Filter by library scope |
| `next` | query | string | No | Cursor for pagination (opaque token from previous response's nextCursor field) |
| `limit` | query | integer | No | Maximum number of items to return |

### Responses

**200** - List of libraries

```json
{
  "items": [
    {
      "format": "string",
      "hash": "string",
      "hash_short": "string",
      "id": "string",
      "layout": "cas",
      "reference_count": 1,
      "scope": "global",
      "storage_url": "string",
      "total_size": 1,
      "workload_type": "string"
    }
  ],
  "pagination": {
    "count": 1,
    "hasMore": true,
    "nextCursor": "string"
  }
}
```

---

## Get library by hash

<span class="api-method api-method-get">GET</span> `/api/v1/libraries/{hash}`

Get library details by content hash. Accepts 12-character prefix or full 64-character Blake3 hash.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `hash` | path | string | Yes | Library hash (12-char prefix or full 64-char Blake3) |
| `workspaceId` | query | string | No | Workspace ID for access control. If not provided, only global libraries are accessible. |

### Responses

**200** - Library details

```json
{
  "content_type": "string",
  "created_at": "2024-01-15T10:30:00Z",
  "file_count": 1,
  "format": "string",
  "hash": "string",
  "hash_short": "string",
  "id": "string",
  "layout": "cas",
  "metadata_status": "string",
  "reference_count": 1,
  "scope": "global",
  "storage_url": "string",
  "total_size": 1,
  "tracesets": [
    {
      "id": "string",
      "name": "string",
      "workspace_id": "string"
    }
  ],
  "workload_type": "string"
}
```

**300** - Ambiguous hash prefix - multiple matches

**404** - Library not found

---

## Get library

<span class="api-method api-method-get">GET</span> `/api/v1/libraries/{id}`

Returns a library by ID (`lib_xxx` format) or content hash prefix (minimum 8 characters). When using a hash prefix, if the prefix matches multiple libraries, a 300 Multiple Choices response is returned.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `id` | path | string | Yes | Library ID (`lib_xxx` format) or content hash prefix (minimum 8 hex characters) |
| `workspaceId` | query | string | No | Workspace ID (format: `workspace_xxx`). If not provided, only global-scoped libraries are accessible. |

### Responses

**200** - Library details

```json
{
  "created_at": "2026-01-15T10:30:00Z",
  "file_count": 256,
  "hash": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2",
  "hash_short": "a1b2c3d4",
  "id": "lib_abc123xyz",
  "layout": "cas",
  "metadata_status": "complete",
  "reference_count": 3,
  "scope": "global",
  "storage_url": "s3://scala-traces/libraries/lib_abc123xyz/",
  "total_size": 104857600,
  "tracesets": [
    {
      "id": "ts_def456",
      "name": "GPT-175B Traces",
      "workspace_id": "workspace_ghi789"
    }
  ]
}
```

**300** - Multiple choices — hash prefix matches more than one library. Provide a longer prefix.

**400** - Invalid request — hash prefix too short (minimum 8 characters)

```json
{
  "error": {
    "message": "Hash prefix 'a1b2' is too short. Minimum 8 characters required."
  }
}
```

**404** - Library not found

```json
{
  "error": {
    "message": "Library 'lib_notfound' not found"
  }
}
```

---
