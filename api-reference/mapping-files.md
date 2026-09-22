---
title: "Mapping-Files"
description: "Chakra mapping files (feature-gated: enable_trace_routes)"
---

Chakra mapping files (feature-gated: enable_trace_routes)

{/* AUTO-GENERATED CONTENT BELOW - DO NOT EDIT MANUALLY */}
{/* Generated from OpenAPI spec by generate-api-reference.py */}

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/mapping-files` | List mapping files |
| GET | `/api/v1/mapping-files/{mapping_file_id}` | Get mapping file |
| DELETE | `/api/v1/mapping-files/{mapping_file_id}` | Delete mapping file |
| GET | `/api/v1/mapping-files/{mapping_file_id}/download` | Get mapping file download URL |
| POST | `/api/v1/mapping-files/upload-url` | Initiate mapping-file upload |
| POST | `/api/v1/mapping-files/upload-url/complete` | Complete mapping-file upload |

---

## List mapping files

<span class="api-method api-method-get">GET</span> `/api/v1/mapping-files`

Lists the Chakra mapping files in a workspace, newest first, with cursor pagination. Deleted files are omitted.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `workspaceId` | query | string | Yes | Workspace to list (format: `workspace_xxx`). |
| `limit` | query | integer | No | Maximum number of items to return. |
| `next` | query | string | No | Opaque cursor token from the previous response's nextCursor field. |

### Responses

**200** - Paginated list of mapping files

```json
{
  "items": [
    {
      "id": "mapfile_2cVQ8f0YyQ3nT1hKpR7mBz4LsXd",
      "name": "gpt-175b-256n.txt",
      "workspaceId": "workspace_def456",
      "sizeBytes": 4096,
      "rankCount": 256,
      "scaleUpGroupCount": 2,
      "hasScaleUp": true,
      "createdAt": "2026-09-16T14:30:00Z"
    }
  ],
  "pagination": {
    "count": 1,
    "hasMore": false
  }
}
```

**400** - Bad Request — workspaceId is absent or blank (there is no tenant-wide fallback), or the cursor is invalid or expired

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "workspaceId is required"
  }
}
```

---

## Get mapping file

<span class="api-method api-method-get">GET</span> `/api/v1/mapping-files/{mapping_file_id}`

Returns a mapping file's metadata, including the stored CRC64NVME checksum of its bytes.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `mapping_file_id` | path | string | Yes | Mapping file ID (format: `mapfile_xxx`). |

### Responses

**200** - Mapping file details

```json
{
  "id": "mapfile_2cVQ8f0YyQ3nT1hKpR7mBz4LsXd",
  "name": "gpt-175b-256n.txt",
  "workspaceId": "workspace_def456",
  "sizeBytes": 4096,
  "rankCount": 256,
  "scaleUpGroupCount": 2,
  "hasScaleUp": true,
  "createdAt": "2026-09-16T14:30:00Z",
  "description": "Reverse placement for the 256-rank study",
  "hash": "crc64nvme:vK3Jd8Qm1pA",
  "etag": "\"0\"",
  "duplicateOfExisting": false
}
```

**404** - Mapping file not found

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Mapping file 'mapfile_notfound' not found"
  }
}
```

**410** - Mapping file has been deleted

```json
{
  "error": {
    "code": "MAPPING_FILE_DELETED",
    "message": "Mapping file 'gpt-175b-256n.txt' has been deleted"
  }
}
```

---

## Delete mapping file

<span class="api-method api-method-delete">DELETE</span> `/api/v1/mapping-files/{mapping_file_id}`

Soft-deletes a mapping file. The delete succeeds whether or not a configuration refers to the file; a configuration that refers to it then reports it as deleted, and creating a simulation from that configuration is refused until the file is detached or another is attached. Simulations already created are unaffected.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `mapping_file_id` | path | string | Yes | Mapping file ID (format: `mapfile_xxx`). |

### Responses

**204** - Mapping file deleted

**404** - Mapping file not found, or already deleted

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Mapping file 'mapfile_notfound' not found"
  }
}
```

---

## Get mapping file download URL

<span class="api-method api-method-get">GET</span> `/api/v1/mapping-files/{mapping_file_id}/download`

Returns a presigned URL for the mapping file's stored bytes. The served bytes are byte-for-byte the uploaded bytes.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `mapping_file_id` | path | string | Yes | Mapping file ID (format: `mapfile_xxx`). |

### Responses

**200** - Presigned download URL

```json
{
  "url": "https://scala-traces.s3.amazonaws.com/library/crc64nvme:vK3Jd8Qm1pA/gpt-175b-256n.txt?X-Amz-Algorithm=...",
  "expiresAt": "2026-09-16T15:30:00Z"
}
```

**404** - Mapping file not found

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Mapping file 'mapfile_notfound' not found"
  }
}
```

**410** - Mapping file has been deleted

```json
{
  "error": {
    "code": "MAPPING_FILE_DELETED",
    "message": "Mapping file 'gpt-175b-256n.txt' has been deleted"
  }
}
```

---

## Initiate mapping-file upload

<span class="api-method api-method-post">POST</span> `/api/v1/mapping-files/upload-url`

Creates an upload session and returns presigned S3 multipart upload URLs for one Chakra mapping file. After PUTting every part, call POST /api/v1/mapping-files/upload-url/complete to validate and store it. Nothing is stored until that call succeeds.

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | No | Optional display name; absent, it is chakra-mapping.txt. Used verbatim as the final segment of the stored object's key, so it must be a single safe path segment of 1 to 255 bytes of [A-Za-z0-9._-] and must not be `.` or `..`. Display metadata only: inside the simulator the file is always named chakra-mapping.txt. |
| `sizeBytes` | integer | Yes | Upper bound on the file's byte length. The 64,000,000-byte ceiling is the 64 MB per-file limit. The stored sizeBytes is the completed object's actual length, which may be smaller. |
| `workspaceId` | string | Yes | Workspace the file belongs to (format: `workspace_xxx`). Must not be blank or padded with whitespace; it is stored exactly as sent. |
| `description` | string | No | - |

```json
{
  "workspaceId": "workspace_def456",
  "name": "gpt-175b-256n.txt",
  "sizeBytes": 4096,
  "description": "Reverse placement for the 256-rank study"
}
```

### Responses

**200** - Presigned upload URLs for each part

```json
{
  "uploadId": "upload_2cVQ8f0YyQ3nT1hKpR7mBz4LsXd",
  "checksumAlgorithm": "crc64nvme",
  "parts": [
    {
      "partNumber": 1,
      "url": "https://scala-traces.s3.amazonaws.com/uploads/upload_2cVQ8f0YyQ3nT1hKpR7mBz4LsXd/gpt-175b-256n.txt?partNumber=1&X-Amz-Algorithm=..."
    }
  ],
  "expiresAt": "2026-09-17T14:30:00Z"
}
```

**400** - Invalid request — name is not a single safe path segment, sizeBytes is outside 1..64000000, workspaceId is absent, blank, or padded with whitespace, or description exceeds 2048 characters

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "sizeBytes must be between 1 and 64000000 bytes (the 64 MB per-file limit); got 64000001"
  }
}
```

---

## Complete mapping-file upload

<span class="api-method api-method-post">POST</span> `/api/v1/mapping-files/upload-url/complete`

Finalises the multipart upload, validates the file against the simulator's grammar, stores it once under its content checksum, and creates the mapping file. A file whose bytes already exist as a live mapping file in the workspace returns that file with duplicateOfExisting set and creates nothing. A file that fails validation is refused with the offending line number and nothing is stored. Not yet available: until the completion handler ships in the follow-on change, this operation answers 501 NOT_IMPLEMENTED and changes nothing; that response is removed when the handler lands.

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `uploadId` | string | Yes | - |
| `parts` | array[CompletedPart] | Yes | - |

```json
{
  "uploadId": "upload_2cVQ8f0YyQ3nT1hKpR7mBz4LsXd",
  "parts": [
    {
      "partNumber": 1,
      "etag": "\"d41d8cd98f00b204e9800998ecf8427e\""
    }
  ]
}
```

### Responses

**200** - The uploaded bytes already exist as a live mapping file in this workspace; that file is returned with duplicateOfExisting true and no new mapping file was created

```json
{
  "id": "mapfile_2cVQ8f0YyQ3nT1hKpR7mBz4LsXd",
  "name": "gpt-175b-256n.txt",
  "workspaceId": "workspace_def456",
  "sizeBytes": 4096,
  "rankCount": 256,
  "scaleUpGroupCount": 2,
  "hasScaleUp": true,
  "createdAt": "2026-09-16T14:30:00Z",
  "description": "Reverse placement for the 256-rank study",
  "hash": "crc64nvme:vK3Jd8Qm1pA",
  "etag": "\"0\"",
  "duplicateOfExisting": true
}
```

**201** - Mapping file stored and created

```json
{
  "id": "mapfile_2cVQ8f0YyQ3nT1hKpR7mBz4LsXd",
  "name": "gpt-175b-256n.txt",
  "workspaceId": "workspace_def456",
  "sizeBytes": 4096,
  "rankCount": 256,
  "scaleUpGroupCount": 2,
  "hasScaleUp": true,
  "createdAt": "2026-09-16T14:30:00Z",
  "description": "Reverse placement for the 256-rank study",
  "hash": "crc64nvme:vK3Jd8Qm1pA",
  "etag": "\"0\"",
  "duplicateOfExisting": false
}
```

**400** - Invalid request — the upload session is not a mapping-file upload, the parts do not match the session, or the session has failed

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Expected 1 part(s), got 2"
  }
}
```

**404** - Upload session not found

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Upload session not found"
  }
}
```

**409** - The upload session named by uploadId is already being completed or was already completed or aborted (CONFLICT), or a different mapping file with this display name is already stored under this content checksum (MAPPING_FILE_HASH_COLLISION)

```json
{
  "error": {
    "code": "MAPPING_FILE_HASH_COLLISION",
    "message": "A different mapping file is already stored under this content checksum. Re-upload under a different display name."
  }
}
```

**410** - Upload session expired

```json
{
  "error": {
    "code": "GONE",
    "message": "Upload session has expired"
  }
}
```

**413** - The staged object is larger than the declared sizeBytes, or above the 64 MB limit; staging is deleted and nothing is stored

```json
{
  "error": {
    "code": "PAYLOAD_TOO_LARGE",
    "message": "Staged object is 70000000 bytes, above the declared 4096 bytes and/or the 64 MB per-file limit of 64000000 bytes"
  }
}
```

**422** - The file does not parse as a Chakra mapping file; the message names the offending line

```json
{
  "error": {
    "code": "MAPPING_FILE_INVALID",
    "message": "line 3: expected an integer server id, found 'x'"
  }
}
```

**501** - Not yet available: the upload-complete handler ships in the follow-on change, which removes this response. Nothing is stored and the upload session is left untouched

```json
{
  "error": {
    "code": "NOT_IMPLEMENTED",
    "message": "Internal server error"
  }
}
```

**503** - Another upload of the same content held the content lock for too long; the request is safe to retry

```json
{
  "error": {
    "code": "SERVICE_UNAVAILABLE",
    "message": "Internal server error"
  }
}
```

---
