---
title: "Simulations"
description: "Simulation execution and results"
---

Simulation execution and results

{/* AUTO-GENERATED CONTENT BELOW - DO NOT EDIT MANUALLY */}
{/* Generated from OpenAPI spec by generate-api-reference.py */}

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/simulations` | List simulations |
| POST | `/api/v1/simulations` | Start simulation |
| GET | `/api/v1/simulations/{sim_id}` | Get simulation status |
| DELETE | `/api/v1/simulations/{sim_id}` | Delete simulation and free storage |
| POST | `/api/v1/simulations/{sim_id}/terminate` | Terminate running simulation |
| GET | `/api/v1/simulations/{sim_id}/results` | Get simulation results |
| GET | `/api/v1/simulations/{sim_id}/results/download-url` | Generate presigned download URL for result file |
| GET | `/api/v1/simulations/{sim_id}/results/data` | Get computed summary data for simulation results |
| GET | `/api/v1/simulations/{sim_id}/logs` | Get simulation logs |

---

## List simulations

<span class="api-method api-method-get">GET</span> `/api/v1/simulations`

Retrieves a paginated list of simulations with optional filtering by workspace, status, name search, and creation date. Results are sorted by creation date (newest first) by default.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `limit` | query | integer | No | Maximum number of items to return (default: 100, max: 100) |
| `next` | query | string | No | Opaque cursor token from previous response's `nextCursor` for pagination |
| `workspaceId` | query | string | No | Filter by workspace ID (format: `workspace_xxx`) |
| `status` | query | string | No | Filter by simulation status. Supports comma-separated values for OR logic (e.g., `running,completed,failed`). Valid values: `validating`, `provisioning`, `running`, `completed`, `failed`, `terminated`, `invalid`. Invalid values are silently ignored for forward compatibility. |
| `search` | query | string | No | Full-text search on simulation name (case-insensitive, max 100 characters) |
| `createdAfter` | query | string | No | Filter by creation date - items created on or after this time (ISO 8601, inclusive) |
| `createdBefore` | query | string | No | Filter by creation date - items created before this time (ISO 8601, exclusive) |
| `sort` | query | string | No | Sort order: comma-separated list of `field:direction` pairs (e.g., `created:desc,name:asc`). Default: `created:desc` |

### Responses

**200** - List of simulations

```json
{
  "simulations": [
    {
      "id": "sim_abc123xyz",
      "workspaceId": "workspace_def456",
      "configurationId": "config_ghi789",
      "configurationVersion": 1,
      "name": "my-network-simulation",
      "status": "running",
      "createdAt": "2024-01-15T10:30:00Z"
    }
  ],
  "pagination": {
    "count": 1,
    "hasMore": true,
    "nextCursor": "eyJjcmVhdGVkIjoiMjAyNC0wMS0xNVQxMDozMDowMFoifQ=="
  }
}
```

**400** - Bad Request - Invalid query parameters (e.g., malformed cursor, invalid workspaceId format, search query too long)

```json
{
  "error": {
    "message": "Invalid workspaceId format: expected 'workspace_' prefix"
  }
}
```

---

## Start simulation

<span class="api-method api-method-post">POST</span> `/api/v1/simulations`

Creates and starts a new simulation based on the specified configuration. The simulation will begin in `validating` status, then progress through `provisioning` to `running`. Poll the GET endpoint to monitor progress.

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `workspaceId` | string | No | - |
| `configurationId` | string | Yes | - |
| `configurationVersion` | integer | No | - |
| `name` | string | Yes | Simulation name. SECURITY: this value is interpolated into command strings and Slurm entrypoints executed on cluster nodes, so it is restricted to letters, digits, '-' and '_' (1–255 characters), and must start with a letter or digit (no whitespace, other punctuation, or non-ASCII characters, and no leading '-' or '_'). |
| `duration` | string | Yes | - |
| `checkpointing` | CheckpointingConfig | No | - |

```json
{
  "configurationId": "config_def456",
  "configurationVersion": 1,
  "name": "production-network-test",
  "duration": "1h",
  "checkpointing": {
    "enabled": true,
    "interval": "10m"
  }
}
```

### Responses

**201** - Simulation started successfully

```json
{
  "id": "string",
  "workspaceId": "string",
  "configurationId": "string",
  "configurationVersion": 1,
  "name": "string",
  "status": "validating",
  "createdAt": "2024-01-15T10:30:00Z",
  "startedAt": "2024-01-15T10:30:00Z",
  "completedAt": "2024-01-15T10:30:00Z",
  "failureReason": "string",
  "metrics": {
    "simulatedTime": "string",
    "eventsProcessed": 1,
    "throughput": "string"
  }
}
```

**400** - Bad Request - Invalid simulation parameters

**402** - Payment Required - the platform's credit balance is exhausted. Add credits before launching simulations.

**404** - Configuration or workspace not found

**422** - Configuration is not validated — must have status `validated` before starting a simulation

---

## Get simulation status

<span class="api-method api-method-get">GET</span> `/api/v1/simulations/{sim_id}`

Retrieves detailed information about a specific simulation, including its current status, timing information, and metrics (if available).

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `sim_id` | path | string | Yes | Simulation ID in `sim_xxx` format (base32-encoded UUID with prefix) |

### Responses

**200** - Simulation details

```json
{
  "id": "string",
  "workspaceId": "string",
  "configurationId": "string",
  "configurationVersion": 1,
  "name": "string",
  "status": "validating",
  "createdAt": "2024-01-15T10:30:00Z",
  "startedAt": "2024-01-15T10:30:00Z",
  "completedAt": "2024-01-15T10:30:00Z",
  "failureReason": "string",
  "metrics": {
    "simulatedTime": "string",
    "eventsProcessed": 1,
    "throughput": "string"
  }
}
```

**404** - Simulation not found

---

## Delete simulation and free storage

<span class="api-method api-method-delete">DELETE</span> `/api/v1/simulations/{sim_id}`

[Feature-gated: enable_simulation_delete] Permanently deletes a simulation and all associated data including results, logs, and checkpoints. This action cannot be undone. Running simulations must be terminated before deletion. This endpoint is only available on deployments where the enable_simulation_delete feature flag is enabled; callers may receive 404 on deployments where the flag is off.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `sim_id` | path | string | Yes | Simulation ID in `sim_xxx` format |

### Responses

**200** - Simulation scheduled for deletion

```json
{
  "id": "string",
  "status": "string",
  "message": "string"
}
```

**400** - Invalid simulation ID format

**404** - Simulation or workspace not found

---

## Terminate running simulation

<span class="api-method api-method-post">POST</span> `/api/v1/simulations/{sim_id}/terminate`

Initiates graceful termination of a running simulation. The simulation will transition to `terminated` status. Results generated up to the termination point will be available.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `sim_id` | path | string | Yes | Simulation ID in `sim_xxx` format |

### Responses

**200** - Simulation termination initiated

```json
{
  "id": "string",
  "workspaceId": "string",
  "configurationId": "string",
  "configurationVersion": 1,
  "name": "string",
  "status": "validating",
  "createdAt": "2024-01-15T10:30:00Z",
  "startedAt": "2024-01-15T10:30:00Z",
  "completedAt": "2024-01-15T10:30:00Z",
  "failureReason": "string",
  "metrics": {
    "simulatedTime": "string",
    "eventsProcessed": 1,
    "throughput": "string"
  }
}
```

**202** - Termination dispatched for a simulation whose workspace has been deleted. Returned without a body: workspace deletion revokes the simulation's details, so the acknowledgement carries no SimulationDetails payload. Also returned when the backend refuses the request with a 4xx, whose message would describe the revoked simulation; a backend 5xx is reported as itself.

**400** - Bad Request - Invalid simulation ID format, or the simulation has no platform link and cannot be terminated

**404** - Simulation not found

**409** - Conflict - Simulation is not in a running state

**501** - Not Implemented - Termination is not yet supported for this simulation's backend

**503** - Service Unavailable - The termination transport is unavailable

---

## Get simulation results

<span class="api-method api-method-get">GET</span> `/api/v1/simulations/{sim_id}/results`

Retrieves results for a completed simulation including summary metrics, dashboard URL, and downloadable result files (pcap, traces, etc.).

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `sim_id` | path | string | Yes | Simulation ID in `sim_xxx` format |
| `next` | query | string | No | Opaque cursor token from previous response's `nextCursor` for pagination |
| `limit` | query | integer | No | Maximum number of items to return (default: 100, min: 1, max: 100) |
| `type` | query | string | No | Filter by result file type (pcap, csv, json, log, state, unknown) |
| `metrics` | query | string | No | Filter by metrics query parameter |
| `sort` | query | string | No | Sort field (created_at, size). Default: created_at |
| `order` | query | string | No | Sort order (asc, desc). Default: desc |

### Responses

**200** - Simulation results

```json
{
  "simulationId": "string",
  "duration": "string",
  "summary": {
    "totalPackets": 1,
    "avgLatency": "string",
    "throughput": "string",
    "dropRate": 1.0
  },
  "dashboardUrl": "https://example.com",
  "files": [
    {
      "type": "pcap",
      "name": "string",
      "size": "string",
      "downloadUrl": "https://example.com"
    }
  ],
  "pagination": {
    "count": 1,
    "hasMore": true,
    "nextCursor": "string"
  }
}
```

**400** - Invalid simulation ID format, invalid pagination cursor, or limit out of range

**404** - Simulation not found or results not yet available

**500** - Internal server error

---

## Generate presigned download URL for result file

<span class="api-method api-method-get">GET</span> `/api/v1/simulations/{sim_id}/results/download-url`

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `sim_id` | path | string | Yes | Simulation ID in sim_xxx format (base32-encoded UUID with prefix) |
| `fileName` | query | string | Yes | Path of the result file to download, relative to the simulation's results prefix. Nested files use their subdirectory path (e.g. "subdir/file.csv"). |
| `metrics` | query | string | No | Reserved for future use. Filter by metrics type (e.g., nd-stats, perf-stats, pfc-stats, post-annotated, logs) |
| `type` | query | string | No | Reserved for future use. File type filter (e.g., zstd) |

### Responses

**200** - File download URL generated

```json
{
  "simulationId": "string",
  "fileName": "string",
  "contentType": "string",
  "sizeBytes": 1,
  "checksum": "string",
  "downloadUrl": "https://example.com",
  "expiresAt": "2024-01-15T10:30:00Z"
}
```

**400** - Invalid simulation ID format or missing fileName

**404** - Simulation or file not found

**500** - Internal server error

---

## Get computed summary data for simulation results

<span class="api-method api-method-get">GET</span> `/api/v1/simulations/{sim_id}/results/data`

Computes or retrieves cached summary statistics for simulation result CSV files using Polars. Returns a presigned URL to download the zstd-compressed JSON summary.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `sim_id` | path | string | Yes | Simulation ID in sim_xxx format (base32-encoded UUID with prefix) |
| `metric` | query | string (enum) | Yes | Metric type to summarize |
| `mode` | query | string (enum) | No | Output mode (default: summary). Only 'summary' is currently supported. |
| `tier` | query | string (enum) | No | Tier filter for network device results (default: all) |

### Responses

**200** - Summary data with presigned download URL

```json
{
  "simulationId": "string",
  "metric": "nd-stats",
  "mode": "summary",
  "tier": "all",
  "file": {
    "name": "string",
    "size": 1,
    "contentType": "string",
    "downloadUrl": "https://example.com",
    "expiresAt": "2024-01-15T10:30:00Z"
  },
  "metadata": {
    "cached": true,
    "computedAt": "2024-01-15T10:30:00Z",
    "rowsProcessed": 1,
    "computeTimeMs": 1
  }
}
```

**400** - Invalid parameters (bad metric, mode, or tier value)

**404** - Simulation not found or no data files available for the specified metric

**500** - Internal server error

---

## Get simulation logs

<span class="api-method api-method-get">GET</span> `/api/v1/simulations/{sim_id}/logs`

Retrieves paginated logs from a simulation with optional filtering by log level and component. Logs are returned in chronological order.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `sim_id` | path | string | Yes | Simulation ID in `sim_xxx` format |
| `level` | query | string (enum) | No | Filter logs by minimum severity level |
| `component` | query | string | No | Filter logs by component name (e.g., `router`, `switch`, `host`) |
| `limit` | query | integer | No | Maximum number of log entries to return (default: 100, max: 100) |
| `next` | query | string | No | Opaque cursor token from previous response's `nextCursor` for pagination |

### Responses

**200** - Simulation logs

```json
{
  "logs": [
    {
      "timestamp": "2024-01-15T10:30:00Z",
      "level": "debug",
      "component": "string",
      "message": "string"
    }
  ],
  "pagination": {
    "count": 1,
    "hasMore": true,
    "nextCursor": "string"
  }
}
```

**404** - Simulation not found

---
