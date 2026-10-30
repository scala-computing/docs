---
title: "Running a Simulation: End-to-End Example"
description: "This example demonstrates a complete workflow for running a network simulation, including custom configurations and result analysis."
---

This example demonstrates a complete workflow for running a network simulation, including custom configurations and result analysis.

## Scenario

We'll simulate a data center network with:
- A leaf-spine topology
- Multiple host nodes

## 1. Set Up Environment

First, set your API token and base URL:

```bash
export SCALA_API_TOKEN="your-token-here"
export API_BASE="https://api.scalacomputing.com/api/v1"
```

See [Authentication](../authentication.md) for how to obtain a token.

## 2. Explore Available Components

List component schemas to understand available network elements:

```bash
curl -X GET "$API_BASE/components" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

```json
{
  "components": [
    {
      "id": "model_37w6uUurBkspg7voji4julJIPSP",
      "model": "ScalaSwitch",
      "name": "ScalaSwitch",
      "category": "model",
      "type": "switch",
      "description": "Configurable L2/L3 switch",
      "latestVersion": "1.1",
      "versions": ["1.1", "1.0"]
    },
    {
      "id": "model_2nE5kQ8xVb1Rt4Yc7Wm0Lp3Hs9D",
      "model": "ScalaRoceNic",
      "name": "ScalaRoceNic",
      "category": "model",
      "type": "nic",
      "description": "RoCE NIC for compute hosts",
      "latestVersion": "1.0",
      "versions": ["1.0"]
    }
  ],
  "pagination": {
    "count": 2,
    "hasMore": false
  }
}
```

Get detailed schema for a component:

```bash
curl -X GET "$API_BASE/components/model_37w6uUurBkspg7voji4julJIPSP" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

## 3. Create a Custom Configuration

Build a configuration with topology, components, and applications

```bash
curl -X POST "$API_BASE/configurations" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Leaf-Spine 48 Hosts",
    "components": [
      {
        "type": "switch",
        "model": "ScalaSwitch",
        "name": "spine",
        "typedParameters": {}
      },
      {
        "type": "switch",
        "model": "ScalaSwitch",
        "name": "leaf",
        "typedParameters": {}
      },
      {
        "type": "nic",
        "model": "ScalaRoceNic",
        "name": "host-nic",
        "typedParameters": {}
      }
    ],
    "containers": [],
    "applications": [],
    "links": [],
    "topology": {
      "type": "clos",
      "topologyTiers": 2,
      "allowedComponents": ["switch", "pod"],
      "components": [
        { "name": "spine", "type": "switch", "count": 2 },
        { "name": "leaf",  "type": "switch", "count": 4 }
      ]
    },
    "globalParameters": {}
  }'
```

```json
{
  "id": "config_9012",
  "workspaceId": "workspace_prod123",
  "name": "Leaf-Spine 48 Hosts",
  "status": "draft",
  "version": 1,
  "createdAt": "2024-01-15T10:35:00Z",
  "modifiedAt": "2024-01-15T10:35:00Z",
  "componentCount": 3
}
```

The response is the full configuration. This example shows only its summary fields. The full response also carries `components`, `containers`, `applications`, `links`, `topology`, and `globalParameters`.

## 4. Validate Configuration

```bash
curl -X POST "$API_BASE/configurations/config_9012/validate" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

If there are validation issues, fix them and update. Pass the configuration's current version, in quotes, as `If-Match`. A version that no longer matches returns 412.

```bash
curl -X PATCH "$API_BASE/configurations/config_9012" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "If-Match: \"1\"" \
  -H "Content-Type: application/json" \
  -d '{
    "topology": { ... }
  }'
```

## 5. Start the Simulation

With a validated configuration, start the simulation:

```bash
curl -X POST "$API_BASE/simulations" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "configurationId": "config_9012",
    "name": "leaf-spine-stress-test",
    "duration": "3600s"
  }'
```

The simulation is created asynchronously—this endpoint returns immediately after queuing. The initial status is `validating` or `provisioning`. Some simulations skip `validating` and start in `provisioning` while resources are prepared.

```json
{
  "id": "sim_stress001",
  "workspaceId": "workspace_prod123",
  "configurationId": "config_9012",
  "configurationVersion": 1,
  "name": "leaf-spine-stress-test",
  "status": "validating",
  "createdAt": "2024-01-15T10:40:00Z"
}
```

Save the simulation ID for subsequent calls:

```bash
export SIM_ID="sim_stress001"
```

## 6. Monitor Simulation Progress

### Check Status

```bash
curl -X GET "$API_BASE/simulations/$SIM_ID" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

### Fetch Logs

```bash
# Get logs with filtering
curl -X GET "$API_BASE/simulations/$SIM_ID/logs?level=error" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

Response:

```json
{
  "logs": [
    {
      "timestamp": "2024-01-15T10:42:00Z",
      "level": "error",
      "component": "simulator",
      "message": "Buffer overflow on port 3"
    }
  ],
  "pagination": {
    "count": 1,
    "hasMore": true,
    "nextCursor": "abc123"
  }
}
```

### Polling Script

```bash
#!/bin/bash
# Uses $SIM_ID and $SCALA_API_TOKEN from environment

while true; do
  STATUS=$(curl -s "$API_BASE/simulations/$SIM_ID" \
    -H "Authorization: Bearer $SCALA_API_TOKEN" \
    | jq -r '.status')
  
  echo "$(date): Status = $STATUS"
  
  case $STATUS in
    completed|failed|terminated|invalid)
      echo "Simulation finished with status: $STATUS"
      break
      ;;
  esac
  
  sleep 30
done
```

## 7. Terminate If Needed

If you need to terminate the simulation:

```bash
curl -X POST "$API_BASE/simulations/$SIM_ID/terminate" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

The response is the simulation's details. A simulation that has already finished keeps the status it ended with. Otherwise the status can keep reading `running` for a while after the request, until the run stops; it then becomes `terminated`, and results written up to that point stay available.

## 8. Retrieve and Analyze Results

### List Result Files

```bash
curl -X GET "$API_BASE/simulations/$SIM_ID/results" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

```json
{
  "simulationId": "sim_stress001",
  "duration": "01:02:17",
  "files": [
    {
      "type": "csv",
      "name": "sim_stress001-nd-stats-0.csv",
      "size": "2621440",
      "downloadUrl": "https://..."
    },
    {
      "type": "csv",
      "name": "sim_stress001-ecn-pfc-stats-0.csv",
      "size": "524288",
      "downloadUrl": "https://..."
    },
    {
      "type": "json",
      "name": "sim_stress001-global-sim-report-0.json",
      "size": "4096",
      "downloadUrl": "https://..."
    }
  ],
  "pagination": {
    "count": 3,
    "hasMore": false
  }
}
```

The list returns at most 100 files at a time. When `pagination.hasMore` is `true`, pass `pagination.nextCursor` as the `next` query parameter to get the next page. [Simulation output files](../simulation-output-files.md) describes each file.

### Download Result Files

Set `FILE_NAME` to a name from the list:

```bash
# Download a specific file
curl -o "$FILE_NAME" "$(curl -s "$API_BASE/simulations/$SIM_ID/results/download-url?fileName=$FILE_NAME" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" | jq -r '.downloadUrl')"
```

### Get Computed Results

The platform can compute results from the files for you. The response links to a zstd-compressed JSON document:

```bash
URL=$(curl -s "$API_BASE/simulations/$SIM_ID/results/data?metric=nd-stats&tier=all" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" | jq -r '.file.downloadUrl')
curl -s "$URL" | zstd -d > nd-stats-summary.json
```

## 9. Cleanup

Delete the simulation when done:

```bash
curl -X DELETE "$API_BASE/simulations/$SIM_ID" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

## Error Handling Best Practices

Always check response status codes and handle errors:

```bash
response=$(curl -s -w "\n%{http_code}" -X POST "$API_BASE/simulations" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"configurationId": "config_9012", "name": "error-handling-demo", "duration": "3600s"}')

body=$(echo "$response" | head -n -1)
status=$(echo "$response" | tail -n 1)

if [ "$status" -ge 400 ]; then
  echo "Error $status: $(echo "$body" | jq -r '.error.message')"
  exit 1
fi
```

## Next Steps

- Explore the [API Reference](../api-reference/index.md) for all available endpoints
- Check [available models](../api-reference/models.md) for specialized network components
