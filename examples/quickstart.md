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
      "id": "cfg_1234",
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
curl -X POST "$API_BASE/templates/cfg_1234/clone" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "My Data Center Config"
  }'
```

Response:

```json
{
  "id": "cfg_xyz789",
  "workspaceId": "ws_abc123",
  "name": "My Data Center Config",
  "status": "draft",
  "version": 1,
  "createdAt": "2024-01-15T10:35:00Z",
  "modifiedAt": "2024-01-15T10:35:00Z"
}
```

## Step 3: Validate the Configuration

Before running a simulation, validate your configuration:

```bash
curl -X POST "$API_BASE/configurations/cfg_xyz789/validate" \
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
    "configurationId": "cfg_xyz789",
    "name": "my-first-simulation",
    "duration": "60s"
  }'
```

Response:

```json
{
  "id": "sim_def456",
  "workspaceId": "ws_abc123",
  "configurationId": "cfg_xyz789",
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
1. `validating` - Checking configuration
2. `provisioning` - Allocating resources
3. `running` - Simulation in progress
4. `completed` - Finished successfully

Or error states: `failed`, `terminated`

## Step 6: Get Results

Once the simulation completes, retrieve the results:

```bash
curl -X GET "$API_BASE/simulations/sim_def456/results" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

Response:

```json
{
  "simulationId": "sim_def456",
  "duration": "60s",
  "summary": {
    "totalPackets": 1500000,
    "avgLatency": "2.5ms",
    "throughput": "10Gbps",
    "dropRate": 0.001
  },
  "files": [
    {
      "type": "pcap",
      "name": "capture.pcap",
      "size": "150MB",
      "downloadUrl": "https://..."
    }
  ]
}
```

## Next Steps

- Learn more about [configurations](../api-reference/configurations.md) to customize your simulations
- Explore [available models](../api-reference/models.md) for different network components
- See the [full simulation example](./run-simulation.md) for advanced usage
