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
| `components` | array[ComponentDeclaration] | Yes | Component declarations (switches, NICs, servers) |
| `containers` | array[ContainerDefinition] | Yes | Container definitions (racks, pods) |
| `applications` | array[ApplicationDefinition] | Yes | Application definitions |
| `links` | array[ComponentDeclaration] | No | Layer components defining spine plane linking configuration |

```json
{
  "components": [
    {
      "name": "FabricSwitch",
      "type": "switch",
      "model": "ScalaSwitch",
      "version": "3.2.1",
      "typedParameters": {
        "NumUpLinks": {
          "type": "uint",
          "value": 1
        },
        "NumDownLinks": {
          "type": "uint",
          "value": 8
        },
        "SwitchingCapacity": {
          "type": "datarate",
          "unit": "Gbps",
          "value": 3200
        }
      }
    },
    {
      "name": "RackSwitch",
      "type": "switch",
      "model": "ScalaSwitch",
      "version": "3.2.1",
      "typedParameters": {
        "NumUpLinks": {
          "type": "uint",
          "value": 2
        },
        "NumDownLinks": {
          "type": "uint",
          "value": 4
        },
        "SwitchingCapacity": {
          "type": "datarate",
          "unit": "Gbps",
          "value": 1600
        }
      }
    },
    {
      "name": "Nic",
      "type": "nic",
      "model": "ScalaNic",
      "version": "3.1.1",
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
      }
    },
    {
      "name": "Host",
      "type": "server",
      "model": "ScalaHost",
      "version": "3.1.1",
      "typedParameters": {},
      "allowedComponents": [
        "nic"
      ],
      "components": [
        {
          "name": "Nic",
          "count": 1,
          "type": "nic"
        }
      ]
    }
  ],
  "containers": [
    {
      "type": "rack",
      "name": "Rack",
      "allowedComponents": [
        "switch",
        "server"
      ],
      "components": [
        {
          "name": "RackSwitch",
          "count": 1,
          "type": "switch",
          "model": "ScalaSwitch"
        },
        {
          "name": "Host",
          "count": 4,
          "type": "server",
          "model": "ScalaHost"
        }
      ]
    },
    {
      "type": "pod",
      "name": "Pod",
      "allowedComponents": [
        "switch",
        "rack"
      ],
      "components": [
        {
          "name": "FabricSwitch",
          "count": 2,
          "type": "switch",
          "model": "ScalaSwitch"
        },
        {
          "name": "Rack",
          "count": 2,
          "type": "rack"
        }
      ]
    }
  ],
  "applications": [
    {
      "name": "TrafficGen",
      "type": "application",
      "model": "ScalaTrafficGeneratorApplication",
      "version": "3.1.1",
      "trafficType": "non-coordinated",
      "typedParameters": {
        "TrafficPattern": {
          "type": "string",
          "value": "uniform-random"
        },
        "MessageSize": {
          "type": "bytes",
          "unit": "KB",
          "value": 16
        }
      }
    }
  ],
  "links": []
}
```

### Responses

**200** - Validation result with topology-specific data

```json
{
  "validation": {
    "valid": true,
    "status": "valid",
    "errors": null,
    "warnings": null,
    "info": [
      {
        "message": "Preprocessor validation (PASS): Topology validated successfully",
        "severity": "info",
        "code": "PREPROCESSOR_VALIDATION_NOTE",
        "path": null,
        "component": null,
        "parameter": null
      }
    ]
  },
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
  "checksum": "abc123def456"
}
```

**400** - Invalid request body

**401** - Authentication required

---
