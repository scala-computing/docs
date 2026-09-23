---
title: "Tracesets"
description: "Trace file management (feature-gated: enable_trace_routes)"
---

Trace file management (feature-gated: enable_trace_routes)

{/* AUTO-GENERATED CONTENT BELOW - DO NOT EDIT MANUALLY */}
{/* Generated from OpenAPI spec by generate-api-reference.py */}

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/tracesets` | List tracesets |
| POST | `/api/v1/tracesets` | Create traceset (direct upload) |
| GET | `/api/v1/tracesets/{id}` | Get traceset |
| DELETE | `/api/v1/tracesets/{id}` | Delete traceset |
| GET | `/api/v1/tracesets/{id}/download` | Get traceset download URL |
| POST | `/api/v1/tracesets/{id}/validate` | Validate traceset |
| GET | `/api/v1/tracesets/{id}/resolve` | Resolve traceset |
| POST | `/api/v1/tracesets/upload-url` | Initiate multipart traceset upload |
| POST | `/api/v1/tracesets/upload-url/complete` | Complete multipart traceset upload |

---

## List tracesets

<span class="api-method api-method-get">GET</span> `/api/v1/tracesets`

List all tracesets. Supports pagination, filtering by workspace/status, full-text name search, and date range filtering. Results are sorted by creation date (newest first) by default.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `workspaceId` | query | string | No | Filter by workspace ID (format: `workspace_xxx`). If not provided, returns all tracesets accessible to the authenticated user. |
| `status` | query | string | No | Filter by traceset status. Valid values: `uploading`, `processing`, `ready`, `metadata_failed`, `upload_failed`, `deleted`. |
| `search` | query | string | No | Full-text search on traceset name (case-insensitive) |
| `sort` | query | string | No | Sort order: comma-separated list of `field:direction` pairs (e.g., `created:desc,name:asc`). Default: `created:desc` |
| `limit` | query | integer | No | Maximum number of items to return |
| `next` | query | string | No | Opaque cursor token from previous response's nextCursor field |
| `createdAfter` | query | string | No | Filter by creation date — items created on or after this time (ISO 8601, inclusive) |
| `createdBefore` | query | string | No | Filter by creation date — items created before this time (ISO 8601, exclusive) |

### Responses

**200** - Paginated list of tracesets

```json
{
  "items": [
    {
      "id": "ts_abc123xyz",
      "name": "GPT-175B Training Traces",
      "status": "ready",
      "fileCount": 4,
      "createdAt": "2026-02-20T14:30:00Z"
    }
  ],
  "pagination": {
    "count": 1,
    "hasMore": true,
    "nextCursor": "eyJjcmVhdGVkIjoiMjAyNi0wMi0yMFQxNDozMDowMFoifQ=="
  }
}
```

**400** - Bad Request — Invalid query parameters (bad cursor, invalid sort field)

```json
{
  "error": {
    "message": "Invalid sort field: 'unknown'. Valid fields: created, name, status"
  }
}
```

---

## Create traceset (direct upload)

<span class="api-method api-method-post">POST</span> `/api/v1/tracesets`

Direct single-file traceset upload. Not yet available for external use — returns 501 Not Implemented. Use POST /api/v1/tracesets/upload-url for multipart upload.

### Responses

**501** - Not implemented — use POST /api/v1/tracesets/upload-url instead

---

## Get traceset

<span class="api-method api-method-get">GET</span> `/api/v1/tracesets/{id}`

Returns a traceset by ID including all file metadata, tags, and expiration information.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `id` | path | string | Yes | Traceset ID (format: `ts_xxx`) |

### Responses

**200** - Traceset details

```json
{
  "id": "ts_abc123xyz",
  "name": "GPT-175B Training Traces",
  "description": "Chakra traces for GPT-175B on 256 ranks",
  "status": "ready",
  "files": [
    {
      "name": "rank_0.chakra.json",
      "size": 1048576,
      "role": "primary",
      "metadataStatus": "complete"
    }
  ],
  "totalSize": 4194304,
  "tags": [
    "gpt",
    "training"
  ],
  "workspaceId": "workspace_def456",
  "createdAt": "2026-02-20T14:30:00Z"
}
```

**404** - Traceset not found

```json
{
  "error": {
    "message": "Traceset 'ts_notfound' not found"
  }
}
```

---

## Delete traceset

<span class="api-method api-method-delete">DELETE</span> `/api/v1/tracesets/{id}`

Soft-deletes a traceset. Associated library files are reference-counted and cleaned up asynchronously. Excluded when trace_routes_read_only is enabled.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `id` | path | string | Yes | Traceset ID (format: `ts_xxx`) |

### Responses

**204** - Traceset deleted

**404** - Traceset not found

```json
{
  "error": {
    "message": "Traceset 'ts_notfound' not found"
  }
}
```

---

## Get traceset download URL

<span class="api-method api-method-get">GET</span> `/api/v1/tracesets/{id}/download`

Returns a presigned S3 download URL for a specific file within a traceset, or for the entire traceset archive if no file is specified. The URL expires after a short TTL.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `id` | path | string | Yes | Traceset ID (format: `ts_xxx`) |
| `file` | query | string | No | Specific file name within the traceset. If omitted, returns a URL for the entire traceset archive. |

### Responses

**200** - Presigned download URL

```json
{
  "url": "https://scala-traces.s3.amazonaws.com/ts_abc123xyz/rank_0.chakra.json?X-Amz-Algorithm=...",
  "expiresAt": "2026-02-23T20:30:00Z",
  "file": "rank_0.chakra.json"
}
```

**400** - File uses an external URL that cannot be downloaded via presigned URL

```json
{
  "error": {
    "message": "File 'external.dat' uses an external URL and cannot be downloaded via presigned URL"
  }
}
```

**404** - Traceset not found

```json
{
  "error": {
    "message": "Traceset 'ts_notfound' not found"
  }
}
```

---

## Validate traceset

<span class="api-method api-method-post">POST</span> `/api/v1/tracesets/{id}/validate`

Runs validation checks on a traceset including file integrity and metadata completeness. Optionally accepts a topologyId for compatibility validation, but the topology_correlation check currently always returns passed (not yet implemented).

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `id` | path | string | Yes | Traceset ID (format: `ts_xxx`) |

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `topologyId` | string | No | - |

```json
{
  "topologyId": "config_ghi789"
}
```

### Responses

**200** - Validation result

```json
{
  "valid": true,
  "checks": [
    {
      "name": "file_integrity",
      "passed": true
    },
    {
      "name": "metadata_complete",
      "passed": true
    },
    {
      "name": "topology_correlation",
      "passed": true,
      "message": "Not yet implemented \u2014 always passes"
    }
  ]
}
```

**404** - Traceset not found

```json
{
  "error": {
    "message": "Traceset 'ts_notfound' not found"
  }
}
```

---

## Resolve traceset

<span class="api-method api-method-get">GET</span> `/api/v1/tracesets/{id}/resolve`

Resolves a traceset to its S3 paths and DRA (Data Resource Accessor) mounting information for use by the simulation orchestrator. The response format depends on the library layout: CAS (content-addressed) layout returns individual file S3 keys; prefix layout returns a file count and root path.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `id` | path | string | Yes | Traceset ID (format: `ts_xxx`) |

### Responses

**200** - Resolved traceset with S3 paths and DRA mount info

```json
{
  "traceset_id": "ts_abc123xyz",
  "name": "GPT-175B Training Traces",
  "layout": "cas",
  "dra": {
    "bucket": "scala-traces",
    "s3_path": "s3://scala-traces/libraries/lib_def456/",
    "lustre_path": "/lustre/traces/lib_def456/"
  },
  "files": [
    {
      "name": "rank_0.chakra.json",
      "role": "primary",
      "s3_key": "libraries/lib_def456/ab/cdef1234567890",
      "s3_path": "s3://scala-traces/libraries/lib_def456/ab/cdef1234567890",
      "lustre_path": "/lustre/traces/lib_def456/ab/cdef1234567890",
      "size": 1048576
    }
  ],
  "total_size": 4194304
}
```

**400** - Traceset cannot be resolved (no files, missing library entry for prefix layout)

```json
{
  "error": {
    "message": "Traceset has no files to resolve"
  }
}
```

**404** - Traceset not found

```json
{
  "error": {
    "message": "Traceset 'ts_notfound' not found"
  }
}
```

---

## Initiate multipart traceset upload

<span class="api-method api-method-post">POST</span> `/api/v1/tracesets/upload-url`

Creates a traceset record and returns presigned S3 multipart upload URLs for each file. Each file is split into parts based on size. After uploading all parts to their presigned URLs, call POST /api/v1/tracesets/upload-url/complete to finalize the upload.

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `workspaceId` | string | No | Workspace ID. If not provided, uses the universal traceset workspace. |
| `name` | string | Yes | - |
| `description` | string | No | - |
| `tags` | array[string] | No | - |
| `ttlDays` | integer | No | Custom TTL in days (1-365). Uses server default if omitted. |
| `files` | array[UploadFileSpec] | Yes | - |

```json
{
  "workspaceId": "workspace_def456",
  "name": "GPT-175B Training Traces",
  "description": "Chakra traces for GPT-175B on 256 ranks",
  "tags": [
    "gpt",
    "training"
  ],
  "files": [
    {
      "name": "rank_0.chakra.json",
      "size": 1048576,
      "role": "primary"
    },
    {
      "name": "rank_1.chakra.json",
      "size": 1048576,
      "role": "primary"
    }
  ]
}
```

### Responses

**200** - Presigned upload URLs for each file part

```json
{
  "uploadId": "upload_ghi789",
  "checksumAlgorithm": "SHA256",
  "files": [
    {
      "name": "rank_0.chakra.json",
      "parts": [
        {
          "partNumber": 1,
          "url": "https://scala-traces.s3.amazonaws.com/...?X-Amz-Algorithm=..."
        }
      ]
    }
  ],
  "expiresAt": "2026-02-24T14:30:00Z"
}
```

**400** - Invalid request (empty files list, missing required fields)

```json
{
  "error": {
    "message": "At least one file is required"
  }
}
```

---

## Complete multipart traceset upload

<span class="api-method api-method-post">POST</span> `/api/v1/tracesets/upload-url/complete`

Finalizes a multipart upload session by providing the ETags returned by S3 for all uploaded parts. The traceset transitions from `uploading` to `processing` state while metadata extraction runs asynchronously. Returns 201 on first completion or 200 on idempotent retry.

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `uploadId` | string | Yes | - |
| `files` | array[CompletedFileInput] | Yes | - |

```json
{
  "uploadId": "upload_ghi789",
  "files": [
    {
      "name": "rank_0.chakra.json",
      "parts": [
        {
          "partNumber": 1,
          "etag": "\"d41d8cd98f00b204e9800998ecf8427e\""
        }
      ]
    }
  ]
}
```

### Responses

**201** - Upload completed and traceset created

```json
{
  "id": "ts_abc123xyz",
  "name": "GPT-175B Training Traces",
  "status": "processing",
  "files": [
    {
      "name": "rank_0.chakra.json",
      "size": 1048576,
      "role": "primary"
    }
  ],
  "totalSize": 1048576
}
```

**200** - Idempotent retry — upload was already completed

```json
{
  "id": "ts_abc123xyz",
  "name": "GPT-175B Training Traces",
  "status": "processing",
  "files": [
    {
      "name": "rank_0.chakra.json",
      "size": 1048576,
      "role": "primary"
    }
  ],
  "totalSize": 1048576
}
```

**400** - Invalid request (missing parts, bad ETags, or an upload session minted by POST /api/v1/mapping-files/upload-url, which only POST /api/v1/mapping-files/upload-url/complete accepts)

```json
{
  "error": {
    "message": "Missing parts for file 'rank_0.chakra.json'"
  }
}
```

**404** - Upload session not found

```json
{
  "error": {
    "message": "Upload session 'upload_notfound' not found"
  }
}
```

**410** - Upload session expired

```json
{
  "error": {
    "message": "Upload session 'upload_ghi789' has expired"
  }
}
```

---
