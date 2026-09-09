---
title: "Topology"
description: "Real-time topology validation"
---

Real-time topology validation

{/* AUTO-GENERATED CONTENT BELOW - DO NOT EDIT MANUALLY */}
{/* Generated from OpenAPI spec by generate-api-reference.py */}

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/topology/validate` | Validate topology without saving |

---

## Validate topology without saving

<span class="api-method api-method-post">POST</span> `/api/v1/topology/validate`

Validates the provided topology components, containers, applications, and links using the same validation path as stored configurations. This enables real-time validation in the UI wizard before a configuration is saved.

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `applications` | array[ApplicationDefinition] | Yes | Application definitions |
| `components` | array[ComponentDeclaration] | Yes | Component declarations (switches, NICs, servers) |
| `containers` | array[ContainerDefinition] | Yes | Container definitions (racks, pods) |
| `links` | array[ComponentDeclaration] | No | Layer components defining spine plane linking configuration |

```json
{
  "applications": [
    {
      "model": "ScalaTrafficGeneratorApplication",
      "name": "TrafficGen",
      "trafficType": "non-coordinated",
      "type": "application",
      "typedParameters": {
        "MessageSize": {
          "type": "bytes",
          "unit": "KB",
          "value": 16
        },
        "TrafficPattern": {
          "type": "string",
          "value": "uniform-random"
        }
      },
      "version": "3.1.1"
    }
  ],
  "components": [
    {
      "model": "ScalaSwitch",
      "name": "FabricSwitch",
      "type": "switch",
      "typedParameters": {
        "NumDownLinks": {
          "type": "uint",
          "value": 8
        },
        "NumUpLinks": {
          "type": "uint",
          "value": 1
        },
        "SwitchingCapacity": {
          "type": "datarate",
          "unit": "Gbps",
          "value": 3200
        }
      },
      "version": "3.2.1"
    },
    {
      "model": "ScalaSwitch",
      "name": "RackSwitch",
      "type": "switch",
      "typedParameters": {
        "NumDownLinks": {
          "type": "uint",
          "value": 4
        },
        "NumUpLinks": {
          "type": "uint",
          "value": 2
        },
        "SwitchingCapacity": {
          "type": "datarate",
          "unit": "Gbps",
          "value": 1600
        }
      },
      "version": "3.2.1"
    },
    {
      "model": "ScalaNic",
      "name": "Nic",
      "type": "nic",
      "typedParameters": {
        "NetworkInterface": {
          "UplinkNetworkInterface": {
            "DataRate": {
              "type": "datarate",
              "unit": "Gbps",
              "value": 100
            }
          }
        }
      },
      "version": "3.1.1"
    },
    {
      "allowedComponents": [
        "nic"
      ],
      "components": [
        {
          "count": 1,
          "name": "Nic",
          "type": "nic"
        }
      ],
      "model": "ScalaHost",
      "name": "Host",
      "type": "server",
      "typedParameters": {},
      "version": "3.1.1"
    }
  ],
  "containers": [
    {
      "allowedComponents": [
        "switch",
        "server"
      ],
      "components": [
        {
          "count": 1,
          "model": "ScalaSwitch",
          "name": "RackSwitch",
          "type": "switch"
        },
        {
          "count": 4,
          "model": "ScalaHost",
          "name": "Host",
          "type": "server"
        }
      ],
      "name": "Rack",
      "type": "rack"
    },
    {
      "allowedComponents": [
        "switch",
        "rack"
      ],
      "components": [
        {
          "count": 2,
          "model": "ScalaSwitch",
          "name": "FabricSwitch",
          "type": "switch"
        },
        {
          "count": 2,
          "name": "Rack",
          "type": "rack"
        }
      ],
      "name": "Pod",
      "type": "pod"
    }
  ],
  "links": []
}
```

### Responses

**200** - Validation result with topology-specific data

```json
{
  "checksum": "abc123def456",
  "simulationParameters": [
    {
      "label": "Total Hosts",
      "name": "TotalHosts",
      "value": 8
    },
    {
      "label": "Total NICs",
      "name": "TotalNics",
      "value": 8
    }
  ],
  "switchBandwidth": [
    {
      "model": "ScalaSwitch",
      "name": "FabricSwitch",
      "networkTier": 1,
      "switchingCapacity": 3200,
      "switchingCapacityUnit": "Gbps"
    }
  ],
  "topologyMap": "",
  "validation": {
    "errors": null,
    "info": [
      {
        "code": "PREPROCESSOR_VALIDATION_NOTE",
        "component": null,
        "message": "Preprocessor validation (PASS): Topology validated successfully",
        "parameter": null,
        "path": null,
        "severity": "info"
      }
    ],
    "status": "valid",
    "valid": true,
    "warnings": null
  }
}
```

**400** - Invalid request body

**401** - Authentication required

---
