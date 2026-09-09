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
| POST | `/api/v1/tracesets` | Upload a traceset (small file) |
| POST | `/api/v1/tracesets/upload-url` | Get pre-signed URLs for large file upload |
| POST | `/api/v1/tracesets/upload-url/complete` | Complete multipart upload |
| GET | `/api/v1/tracesets/{id}` | Get traceset details |
| DELETE | `/api/v1/tracesets/{id}` | Delete a traceset |
| GET | `/api/v1/tracesets/{id}/download` | Get download URL |
| POST | `/api/v1/tracesets/{id}/promote` | Promote traceset scope |
| GET | `/api/v1/tracesets/{id}/resolve` | Resolve traceset |
| POST | `/api/v1/tracesets/{id}/validate` | Validate traceset |

---

## List tracesets

<span class="api-method api-method-get">GET</span> `/api/v1/tracesets`

List all tracesets in the workspace. Supports pagination and filtering by status.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `workspaceId` | query | string | No | Workspace ID to scope the query. If not provided, returns all tracesets. |
| `status` | query | string | No | Filter by traceset status |
| `next` | query | string | No | Cursor for pagination (opaque token from previous response's nextCursor field) |
| `limit` | query | integer | No | Maximum number of items to return (default 20, max 100) |

### Responses

**200** - List of tracesets

```json
{
  "items": [
    {
      "createdAt": "2024-01-15T10:30:00Z",
      "fileCount": 1,
      "id": "string",
      "name": "string",
      "status": "uploading"
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

**403** - Forbidden - not a member of workspace

---

## Upload a traceset (small file)

<span class="api-method api-method-post">POST</span> `/api/v1/tracesets`

Upload a trace file directly. For files larger than 100MB, use the upload-url endpoint instead.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `workspaceId` | query | string | Yes | Workspace ID to upload to |

### Request Body

### Responses

**200** - Traceset already exists (de-duplicated)

```json
{
  "createdAt": "2024-01-15T10:30:00Z",
  "description": "string",
  "expiresAt": "2024-01-15T10:30:00Z",
  "files": [
    {
      "contentType": "string",
      "hash": "string",
      "metadataStatus": "pending",
      "name": "string",
      "rankRange": "...",
      "role": "primary",
      "size": 1
    }
  ],
  "id": "string",
  "name": "string",
  "status": "uploading",
  "tags": [
    "string"
  ],
  "totalSize": 1,
  "workspaceId": "string"
}
```

**201** - Traceset created successfully

```json
{
  "createdAt": "2024-01-15T10:30:00Z",
  "description": "string",
  "expiresAt": "2024-01-15T10:30:00Z",
  "files": [
    {
      "contentType": "string",
      "hash": "string",
      "metadataStatus": "pending",
      "name": "string",
      "rankRange": "...",
      "role": "primary",
      "size": 1
    }
  ],
  "id": "string",
  "name": "string",
  "status": "uploading",
  "tags": [
    "string"
  ],
  "totalSize": 1,
  "workspaceId": "string"
}
```

**400** - Invalid request (bad format, validation failed)

**413** - File too large (use upload-url for files >100MB)

---

## Get pre-signed URLs for large file upload

<span class="api-method api-method-post">POST</span> `/api/v1/tracesets/upload-url`

Initiate a multipart upload and get pre-signed URLs for each part. Use this for files larger than 100MB.

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `description` | string | No | - |
| `files` | array[UploadFileSpec] | Yes | Files to upload. Maximum 100 files per request (configurable server-side via max_files_per_upload). |
| `name` | string | Yes | Traceset name |
| `tags` | array[string] | No | - |
| `ttlDays` | integer | No | Days until trace expires since last use. Default: 7 |
| `workloadMetadata` | WorkloadMetadata | No | Workload characterization metadata for traceset promotion |
| `workspaceId` | string | No | Workspace to upload to. If not provided, uses the universal traceset workspace. |

```json
{
  "description": "string",
  "files": [
    {
      "contentType": "string",
      "name": "string",
      "role": "primary",
      "size": 1
    }
  ],
  "name": "string",
  "tags": [
    "string"
  ],
  "ttlDays": 1,
  "workloadMetadata": {
    "accelerator": "string",
    "collectiveOps": [
      "string"
    ],
    "computeModel": "string",
    "etFormatVersion": "string",
    "executionKind": "string",
    "extensions": {},
    "modelName": "string",
    "parallelism": {
      "cp": "...",
      "dp": "...",
      "ep": "...",
      "fsdp": "...",
      "pp": "...",
      "sp": "...",
      "tp": "..."
    },
    "precision": "string",
    "schemaVersion": 1,
    "shape": {
      "globalBatchSize": "...",
      "hiddenDim": "...",
      "numMicrobatches": "...",
      "rankCount": "...",
      "seqLen": "...",
      "trainingSteps": "..."
    },
    "stepKind": "string",
    "traceGenerator": "string",
    "workloadFamily": "string",
    "workloadName": "string",
    "workloadRevision": "string"
  },
  "workspaceId": "string"
}
```

### Responses

**200** - Pre-signed URLs generated

```json
{
  "expiresAt": "2024-01-15T10:30:00Z",
  "files": [
    {
      "name": "string",
      "parts": [
        "..."
      ]
    }
  ],
  "uploadId": "upload_2cKWFtZ2E5gSLHrnZLT0xwM8r8M"
}
```

**400** - Invalid request (invalid/duplicate filenames, path traversal, invalid role, file count exceeds limit, description/tags/metadata exceeds size limits, file too large)

---

## Complete multipart upload

<span class="api-method api-method-post">POST</span> `/api/v1/tracesets/upload-url/complete`

Complete a multipart upload after all parts have been uploaded. This endpoint is idempotent - calling it multiple times with the same uploadId returns the same traceset.

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `files` | array[CompletedFile] | Yes | - |
| `uploadId` | string | Yes | Upload session ID from upload-url response |

```json
{
  "files": [
    {
      "name": "string",
      "parts": [
        "..."
      ]
    }
  ],
  "uploadId": "string"
}
```

### Responses

**200** - Traceset already exists (idempotent retry)

```json
{
  "createdAt": "2024-01-15T10:30:00Z",
  "description": "string",
  "expiresAt": "2024-01-15T10:30:00Z",
  "files": [
    {
      "contentType": "string",
      "hash": "string",
      "metadataStatus": "pending",
      "name": "string",
      "rankRange": "...",
      "role": "primary",
      "size": 1
    }
  ],
  "id": "string",
  "name": "string",
  "status": "uploading",
  "tags": [
    "string"
  ],
  "totalSize": 1,
  "workspaceId": "string"
}
```

**201** - Traceset created successfully

```json
{
  "createdAt": "2024-01-15T10:30:00Z",
  "description": "string",
  "expiresAt": "2024-01-15T10:30:00Z",
  "files": [
    {
      "contentType": "string",
      "hash": "string",
      "metadataStatus": "pending",
      "name": "string",
      "rankRange": "...",
      "role": "primary",
      "size": 1
    }
  ],
  "id": "string",
  "name": "string",
  "status": "uploading",
  "tags": [
    "string"
  ],
  "totalSize": 1,
  "workspaceId": "string"
}
```

**400** - Invalid request (missing/unexpected files, bad parts, empty ETags, failed session)

**404** - Upload session not found

**409** - Upload session is already being completed by another request

**410** - Upload session has expired

**500** - Server error (S3 failure, database error, checksum unavailable)

---

## Get traceset details

<span class="api-method api-method-get">GET</span> `/api/v1/tracesets/{id}`

Get detailed information about a traceset including files, metadata status, and library hash.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `id` | path | string | Yes | Traceset ID |

### Responses

**200** - Traceset details

```json
{
  "createdAt": "2024-01-15T10:30:00Z",
  "description": "string",
  "expiresAt": "2024-01-15T10:30:00Z",
  "files": [
    {
      "contentType": "string",
      "hash": "string",
      "metadataStatus": "pending",
      "name": "string",
      "rankRange": "...",
      "role": "primary",
      "size": 1
    }
  ],
  "id": "string",
  "name": "string",
  "status": "uploading",
  "tags": [
    "string"
  ],
  "totalSize": 1,
  "workspaceId": "string"
}
```

**404** - Traceset not found

---

## Delete a traceset

<span class="api-method api-method-delete">DELETE</span> `/api/v1/tracesets/{id}`

Soft delete a traceset. The traceset will be marked as deleted and cleaned up after TTL expires. Does not affect the library if other tracesets reference it.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `id` | path | string | Yes | Traceset ID |

### Responses

**204** - Traceset deleted successfully

**404** - Traceset not found

---

## Get download URL

<span class="api-method api-method-get">GET</span> `/api/v1/tracesets/{id}/download`

Get a pre-signed URL to download traceset files. Resets TTL on access. Protected by auth proxy.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `id` | path | string | Yes | Traceset ID |
| `file` | query | string | No | Specific file to download (optional, defaults to primary trace file) |

### Responses

**200** - Download URL

```json
{
  "expiresAt": "2024-01-15T10:30:00Z",
  "file": "string",
  "url": "https://example.com"
}
```

**404** - Traceset not found

---

## Promote traceset scope

<span class="api-method api-method-post">POST</span> `/api/v1/tracesets/{id}/promote`

Promote a traceset from workspace to tenant scope, or from tenant to public scope. Requires status=ready. Idempotent if already at target scope.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `id` | path | string | Yes | Traceset ID |

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `description` | string | No | Optional description override |
| `tags` | array[string] | No | Optional tags override |
| `target` | string | No | Target scope. Defaults to 'tenant' if not specified. |
| `workloadMetadata` | WorkloadMetadata | No | Workload metadata (required for promotion, may be pre-populated on traceset) |

```json
{
  "description": "string",
  "tags": [
    "string"
  ],
  "target": "tenant",
  "workloadMetadata": {
    "accelerator": "string",
    "collectiveOps": [
      "string"
    ],
    "computeModel": "string",
    "etFormatVersion": "string",
    "executionKind": "string",
    "extensions": {},
    "modelName": "string",
    "parallelism": {
      "cp": "...",
      "dp": "...",
      "ep": "...",
      "fsdp": "...",
      "pp": "...",
      "sp": "...",
      "tp": "..."
    },
    "precision": "string",
    "schemaVersion": 1,
    "shape": {
      "globalBatchSize": "...",
      "hiddenDim": "...",
      "numMicrobatches": "...",
      "rankCount": "...",
      "seqLen": "...",
      "trainingSteps": "..."
    },
    "stepKind": "string",
    "traceGenerator": "string",
    "workloadFamily": "string",
    "workloadName": "string",
    "workloadRevision": "string"
  }
}
```

### Responses

**200** - Promotion result

```json
{
  "error": "string",
  "id": "string",
  "promotionApplied": true,
  "scope": "string"
}
```

**400** - Invalid target scope

**404** - Traceset not found

**409** - Traceset not ready for promotion

**412** - Promotion precondition failed (invalid transition, missing metadata)

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
  "dra": {
    "bucket": "scala-traces",
    "lustre_path": "/lustre/traces/lib_def456/",
    "s3_path": "s3://scala-traces/libraries/lib_def456/"
  },
  "files": [
    {
      "lustre_path": "/lustre/traces/lib_def456/ab/cdef1234567890",
      "name": "rank_0.chakra.json",
      "role": "primary",
      "s3_key": "libraries/lib_def456/ab/cdef1234567890",
      "s3_path": "s3://scala-traces/libraries/lib_def456/ab/cdef1234567890",
      "size": 1048576
    }
  ],
  "layout": "cas",
  "name": "GPT-175B Training Traces",
  "total_size": 4194304,
  "traceset_id": "ts_abc123xyz"
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

## Validate traceset

<span class="api-method api-method-post">POST</span> `/api/v1/tracesets/{id}/validate`

Validate traceset format and optionally correlate with topology.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `id` | path | string | Yes | Traceset ID |

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `topologyId` | string | No | Optional topology ID for correlation validation |

```json
{
  "topologyId": "string"
}
```

### Responses

**200** - Validation result

```json
{
  "checks": [
    {
      "message": "string",
      "name": "string",
      "passed": true
    }
  ],
  "valid": true
}
```

**404** - Traceset not found

---
