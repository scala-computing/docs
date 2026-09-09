---
title: "Running a Simulation: End-to-End Example"
description: "This example demonstrates a complete workflow for running a network simulation, including custom configurations and result analysis."
---

This example demonstrates a complete workflow for running a network simulation, including custom configurations and result analysis.

## Scenario

We'll simulate a data center network with:
- A leaf-spine topology
- Multiple host nodes
- Checkpointing enabled for long-running simulations

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
      "id": "comp_switch",
      "model": "ScalaSwitch",
      "name": "ScalaSwitch",
      "category": "model",
      "type": "switch",
      "description": "Configurable L2/L3 switch",
      "latestVersion": "1.1",
      "versions": ["1.1", "1.0"]
    },
    {
      "id": "comp_nic",
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
curl -X GET "$API_BASE/components/comp_switch" \
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
  "id": "cfg_spineleaf",
  "workspaceId": "ws_prod123",
  "name": "Leaf-Spine 48 Hosts",
  "status": "draft",
  "version": 1,
  "createdAt": "2024-01-15T10:35:00Z",
  "modifiedAt": "2024-01-15T10:35:00Z",
  "componentCount": 3
}
```

## 4. Validate Configuration

```bash
curl -X POST "$API_BASE/configurations/cfg_spineleaf/validate" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

If there are validation issues, fix them and update:

```bash
curl -X PATCH "$API_BASE/configurations/cfg_spineleaf" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "If-Match: \"v1\"" \
  -H "Content-Type: application/json" \
  -d '{
    "topology": { ... }
  }'
```

## 5. Start Long-Running Simulation with Checkpointing

With a validated configuration, start the simulation:

```bash
curl -X POST "$API_BASE/simulations" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "configurationId": "cfg_spineleaf",
    "name": "leaf-spine-stress-test",
    "duration": "3600s",
    "checkpointing": {
      "enabled": true,
      "interval": "300s"
    }
  }'
```

The simulation is created asynchronously—this endpoint returns immediately after queuing. The initial status will be `provisioning` while resources are prepared.

```json
{
  "id": "sim_stress001",
  "workspaceId": "ws_prod123",
  "configurationId": "cfg_spineleaf",
  "configurationVersion": 1,
  "name": "leaf-spine-stress-test",
  "status": "provisioning",
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
      "component": "spine-0",
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
    completed|failed|terminated)
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

## 8. Retrieve and Analyze Results

### Get Summary

```bash
curl -X GET "$API_BASE/simulations/$SIM_ID/results" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

```json
{
  "simulationId": "sim_stress001",
  "duration": "3600s",
  "summary": {
    "totalPackets": 450000000,
    "avgLatency": "4.2ms",
    "throughput": "1.2Tbps",
    "dropRate": 0.0005
  },
  "dashboardUrl": "https://app.scalacomputing.com/simulations/sim_stress001/dashboard",
  "files": [
    {
      "type": "csv",
      "name": "latency_histogram.csv",
      "size": "2.5MB",
      "downloadUrl": "https://..."
    },
    {
      "type": "pcap",
      "name": "sample_capture.pcap",
      "size": "500MB",
      "downloadUrl": "https://..."
    },
    {
      "type": "json",
      "name": "flow_completion_times.json",
      "size": "15MB",
      "downloadUrl": "https://..."
    }
  ]
}
```

### Download Result Files

```bash
# Download a specific file
curl -o latency_histogram.csv "$(curl -s "$API_BASE/simulations/$SIM_ID/results" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  | jq -r '.files[] | select(.name == "latency_histogram.csv") | .downloadUrl')"
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
  -d '{"configurationId": "cfg_spineleaf", "name": "error-handling-demo", "duration": "3600s"}')

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
