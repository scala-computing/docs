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
| POST | `/api/v1/simulations/compare` | Compare simulations |
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
  },
  "transportStats": [
    "uet"
  ]
}
```

**400** - Bad Request - Invalid simulation parameters

**402** - Payment Required - the platform's credit balance is exhausted. Add credits before launching simulations.

**404** - Configuration or workspace not found

**422** - Configuration is not validated — must have status `validated` before starting a simulation

---

## Compare simulations

<span class="api-method api-method-post">POST</span> `/api/v1/simulations/compare`

Compares two to four simulations and returns aligned summaries, per-metric deltas and a diff context. The result is returned inline as JSON rather than as a presigned download envelope.

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `simulationIds` | array[string] | Yes | - |
| `tier` | string | Yes | - |

```json
{
  "simulationIds": [
    "string"
  ],
  "tier": "all"
}
```

### Responses

**200** - Comparison result

```json
{
  "simulations": [
    {
      "simulationId": "string",
      "summaryByMetric": "..."
    }
  ],
  "deltas": [
    {
      "metric": "nd-stats",
      "fieldPath": "string",
      "baseline": null,
      "comparisonValues": [
        "..."
      ],
      "percentChange": [
        "..."
      ]
    }
  ],
  "diffContext": {
    "changedParams": [
      "..."
    ],
    "heldConstant": [
      "string"
    ]
  },
  "confidence": "singleChange"
}
```

**400** - Invalid request body (simulationIds count outside 2..4, malformed simulation id, or unknown tier)

**404** - Simulation not found, or its workspace has been deleted

**500** - Internal server error

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
  },
  "transportStats": [
    "uet"
  ]
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
  },
  "transportStats": [
    "uet"
  ]
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
| `sort` | query | string (enum) | No | Field to sort the result files by (default: createdAt) |
| `order` | query | string (enum) | No | Sort direction (default: desc) |

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

**400** - Invalid simulation ID format, invalid pagination cursor, limit out of range, or an unrecognized sort or order value

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
| `metric` | query | string (enum) | Yes | Metric type to summarize. `rank-analysis` is published but returns 501 Not Implemented until its compute path ships. |
| `mode` | query | string (enum) | No | Output mode (default: summary). `timeSeries` and `sampled` are published but return 501 Not Implemented until their compute paths ship. |
| `tier` | query | string (enum) | No | Tier filter for network device results (default: all) |
| `nodeId` | query | array | No | Filter to these node ids. Comma-separated on the wire. An empty list is not accepted — omit the parameter instead. |
| `ifid` | query | array | No | Filter to these interface ids. Comma-separated on the wire. An empty list is not accepted — omit the parameter instead. |
| `nicType` | query | string | No | Filter to a NIC class. |
| `metricName` | query | array | No | Filter to these metric names. Comma-separated on the wire; values may not contain commas. An empty list is not accepted — omit the parameter instead. |
| `application` | query | array | No | Filter to these application names. Comma-separated on the wire; values may not contain commas. An empty list is not accepted — omit the parameter instead. |
| `priority` | query | array | No | Filter to these PFC priorities. Comma-separated on the wire. An empty list is not accepted — omit the parameter instead. |
| `uldl` | query | string (enum) | No | Filter to uplink or downlink direction. |
| `preset` | query | string (enum) | No | Rank-analysis grouping preset (only consumed when metric=rank-analysis). Omit to get the server default, `balanced`. Carries no schema `default`: a schema default makes generated clients send the value on every request, which lists it in X-Ignored-Params on every mode and metric that does not consume it. |
| `tolerance` | query | number | No | Dimensionless multiplier on the per-dimension tolerance bands (only consumed when metric=rank-analysis). Omit to get the server default, 0.0. Carries no schema `default`: a schema default makes generated clients send the value on every request, which lists it in X-Ignored-Params on every mode and metric that does not consume it. |
| `outlierThreshold` | query | number | No | Outlier deviation-score fraction (only consumed when metric=rank-analysis). Omit to get the server default, 0.10. Carries no schema `default`: a schema default makes generated clients send the value on every request, which lists it in X-Ignored-Params on every mode and metric that does not consume it. |

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

**413** - Payload Too Large - The summarized result set exceeds the server's size limit (its row count is over the ceiling, or summarizing it would exceed the memory budget). This condition is permanent for the request and must not be retried.

**500** - Internal server error

**501** - The requested mode or metric is recognized but its compute path has not shipped yet

**503** - Service Unavailable - Several conditions render this status and they differ in whether a retry resolves them; the ones named here are the known ones rather than a closed set. The common one is a transient fault against the results object store on any mode - a network error, throttling, or a 5xx from the dependency - which is retriable: a client that backs off and retries will usually succeed. A second retriable one is a lost cache-write race the server could not recover from: a concurrent request stored the summary object first, and when this request went back to read it the object was no longer there, so a later attempt normally succeeds against the object the next writer stores. The last is a required backend dependency that is not available or not configured, so the simulation's artifact location could not be resolved. Retrying does not help if the dependency is unconfigured. The response body does not distinguish them, so a client should retry with backoff under a bounded attempt count rather than indefinitely.

**504** - Gateway Timeout - Summary computation exceeded the server's time budget. The request may be retried.

---

## Get simulation logs

<span class="api-method api-method-get">GET</span> `/api/v1/simulations/{sim_id}/logs`

Retrieves paginated logs from a simulation with optional filtering by log level and component. Logs are returned in chronological order.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `sim_id` | path | string | Yes | Simulation ID in `sim_xxx` format |
| `level` | query | string (enum) | No | Filter logs by minimum severity level |
| `component` | query | string | No | Filter logs by component name (e.g., `simulator`, `flownizer`, `annotator`) |
| `since` | query | string | No | Filter logs to entries at or after this time (ISO 8601 / RFC 3339) |
| `limit` | query | integer | No | Maximum number of log entries to return (default: 100, max: 1000). A truncated read may return one entry more than `limit`: the in-band truncation notice does not count against it. |
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

**400** - Bad Request - Invalid simulation ID format, invalid `since` timestamp, invalid `level` or `component`, or invalid pagination cursor

**404** - Simulation not found

**500** - Internal server error

**503** - Orchestrator database not configured

---
