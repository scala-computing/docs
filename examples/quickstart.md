---
title: "Quickstart Guide"
description: "This guide walks you through the basic workflow of using the Network Simulation API to run your first simulation."
---

This guide walks you through the basic workflow of using the Network Simulation API to run your first simulation.

## Prerequisites

- API access token (see [Authentication](../authentication.md))
- `curl` or your preferred HTTP client
- Basic familiarity with REST APIs

## Setup

Set your API token as an environment variable:

```bash
export SCALA_API_TOKEN="your-token-here"
export API_BASE="https://api.scalacomputing.com/api/v1"
```

## Step 1: Browse Available Templates

Templates provide pre-configured starting points for common simulation scenarios:

```bash
curl -X GET "$API_BASE/templates" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

Response:

```json
{
  "templates": [
    {
      "id": "config_1234",
      "name": "Data Center Network",
      "description": "Clos topology with configurable scale",
      "category": "getting-started"
    }
  ],
  "pagination": {
    "count": 1,
    "hasMore": false
  }
}
```

## Step 2: Create a Configuration from Template

Clone a template to create your own configuration

```bash
curl -X POST "$API_BASE/templates/config_1234/clone" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "My Data Center Config"
  }'
```

Response:

```json
{
  "id": "config_5678",
  "workspaceId": "workspace_abc123",
  "name": "My Data Center Config",
  "status": "draft",
  "version": 1,
  "createdAt": "2024-01-15T10:35:00Z",
  "modifiedAt": "2024-01-15T10:35:00Z"
}
```

The response is the full configuration. This example shows only its summary fields. The full response also carries `components`, `containers`, `applications`, `links`, `topology`, and `globalParameters`.

## Step 3: Validate the Configuration

Before running a simulation, validate your configuration:

```bash
curl -X POST "$API_BASE/configurations/config_5678/validate" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

Response:

```json
{
  "valid": true,
  "status": "valid",
  "errors": [],
  "warnings": [],
  "info": []
}
```

If there are issues, they'll appear in the `errors` or `warnings` arrays:

```json
{
  "valid": false,
  "status": "error",
  "errors": [
    {
      "severity": "error",
      "component": "switch-0",
      "parameter": "portSpeed",
      "message": "Invalid port speed value"
    }
  ],
  "warnings": [],
  "info": []
}
```

## Step 4: Start a Simulation

Now run your first simulation:

```bash
curl -X POST "$API_BASE/simulations" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "configurationId": "config_5678",
    "name": "my-first-simulation",
    "duration": "60s"
  }'
```

Response:

```json
{
  "id": "sim_def456",
  "workspaceId": "workspace_abc123",
  "configurationId": "config_5678",
  "configurationVersion": 1,
  "name": "my-first-simulation",
  "status": "validating",
  "createdAt": "2024-01-15T10:40:00Z"
}
```

## Step 5: Monitor Progress

Check the simulation status:

```bash
curl -X GET "$API_BASE/simulations/sim_def456" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

The status will progress through:
1. `validating` - Checking configuration. Some simulations skip this status and start in `provisioning`.
2. `provisioning` - Allocating resources
3. `running` - Simulation in progress
4. `completed` - Finished successfully

A simulation can also end in one of these statuses:
- `failed` - The run, or the export of its results, stopped on an error. `failureReason`, when present, says why.
- `terminated` - The run was stopped before it finished: you terminated it, or the platform stopped it, for example after a node failure, an out-of-memory kill or a time limit. A simulation that started in `provisioning` reports those platform stops as `failed` instead.
- `invalid` - The platform could not determine the run's state.

## Step 6: Get Results

Once the simulation completes, list the files it wrote:

```bash
curl -X GET "$API_BASE/simulations/sim_def456/results?type=csv" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

Response:

```json
{
  "simulationId": "sim_def456",
  "duration": "00:01:04",
  "files": [
    {
      "type": "csv",
      "name": "sim_def456-nd-stats-0.csv",
      "size": "1843200",
      "downloadUrl": "https://..."
    },
    {
      "type": "csv",
      "name": "sim_def456-perf-0.csv",
      "size": "52214",
      "downloadUrl": "https://..."
    }
  ],
  "pagination": {
    "count": 2,
    "hasMore": false
  }
}
```

Download a file from its `downloadUrl`, which is valid for 15 minutes. `size` is in bytes. [Simulation output files](../simulation-output-files.md) describes each file and shows how to get results computed from them.

## Next Steps

- Learn more about [configurations](../api-reference/configurations.md) to customize your simulations
- Explore [available models](../api-reference/models.md) for different network components
- See the [full simulation example](./run-simulation.md) for advanced usage
