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
| GET | `/api/v1/libraries/{id}` | Get library |

---

## List libraries

<span class="api-method api-method-get">GET</span> `/api/v1/libraries`

List content-addressed trace libraries. Libraries are de-duplicated by content hash and may be shared across multiple tracesets. Supports pagination, filtering by scope and workspace, full-text search on hash prefix or metadata, and date range filtering.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `workspaceId` | query | string | No | Filter by workspace ID (format: `workspace_xxx`). If not provided, returns only global-scoped libraries. |
| `scope` | query | string | No | Filter by scope. Valid values: `global`, `workspace`. |
| `search` | query | string | No | Search by content hash prefix or metadata fields (case-insensitive) |
| `sort` | query | string | No | Sort order: comma-separated list of `field:direction` pairs (e.g., `created:desc`). Default: `created:desc` |
| `limit` | query | integer | No | Maximum number of items to return |
| `next` | query | string | No | Opaque cursor token from previous response's nextCursor field |
| `createdAfter` | query | string | No | Filter by creation date — items created on or after this time (ISO 8601, inclusive) |
| `createdBefore` | query | string | No | Filter by creation date — items created before this time (ISO 8601, exclusive) |

### Responses

**200** - Paginated list of libraries

```json
{
  "items": [
    {
      "id": "lib_abc123xyz",
      "hash_short": "a1b2c3d4",
      "storage_url": "s3://scala-traces/libraries/lib_abc123xyz/",
      "layout": "cas",
      "scope": "global",
      "total_size": 104857600,
      "reference_count": 3
    }
  ],
  "pagination": {
    "count": 1,
    "hasMore": false
  }
}
```

**400** - Bad Request — Invalid query parameters (bad cursor, invalid sort field)

```json
{
  "error": {
    "message": "Invalid sort field: 'unknown'. Valid fields: created, total_size, reference_count"
  }
}
```

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
  "id": "lib_abc123xyz",
  "hash": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2",
  "hash_short": "a1b2c3d4",
  "storage_url": "s3://scala-traces/libraries/lib_abc123xyz/",
  "layout": "cas",
  "scope": "global",
  "total_size": 104857600,
  "metadata_status": "complete",
  "reference_count": 3,
  "tracesets": [
    {
      "id": "ts_def456",
      "name": "GPT-175B Traces",
      "workspace_id": "workspace_ghi789"
    }
  ],
  "created_at": "2026-01-15T10:30:00Z",
  "file_count": 256
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
