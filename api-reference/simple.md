---
title: "Simple"
description: "Simple collective-operation simulation"
---

Simple collective-operation simulation

{/* AUTO-GENERATED CONTENT BELOW - DO NOT EDIT MANUALLY */}
{/* Generated from OpenAPI spec by generate-api-reference.py */}

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/simple/simulations` | Submit a collective-operation workload for simple network simulation |
| POST | `/api/v1/simple/simulations/preview` | Preview the derived topology for a simple simulation request |
| GET | `/api/v1/simple/compatibility` | Get the operation/algorithm compatibility matrix |

---

## Submit a collective-operation workload for simple network simulation

<span class="api-method api-method-post">POST</span> `/api/v1/simple/simulations`

Validates the workload against operation/algorithm compatibility rules, resolves or synthesizes a topology and traceset, creates a configuration, and launches a simulation. Returns immediately with status `accepted`; poll `GET /api/v1/simulations/{simulationId}` to track progress. Note: the poll response uses a different schema (`SimulationDetails`) — the simulation ID is returned as `id` (not `simulationId`), the workload is not echoed back, and the status transitions from `accepted` to `provisioning` on the next poll. Identical workloads are deduplicated via content-addressed tracesets.

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | No | Optional human-readable name for the simulation. Auto-generated when omitted. SECURITY: this value is interpolated into command strings and Slurm entrypoints executed on cluster nodes, so it is restricted to letters, digits, '-' and '_' (1–255 characters), and must start with a letter or digit (no whitespace, other punctuation, or non-ASCII characters, and no leading '-' or '_'). |
| `topology` | string | No | Optional topology reference (reserved for K2 orchestration). |
| `topologyConstraints` | TopologyConstraints | No | Hardware constraints for topology derivation. The solver derives a valid CLOS topology from the specified switch radix and oversubscription ratio. A 2-tier topology is used when ranks fit within the radix capacity (ports × hosts_per_leaf); otherwise a 3-tier pod-based topology (RSW → FSW → SSW) is derived with all switches fully utilized. When present, rank count is not restricted to the reference values — any count satisfying the constraints is valid. |
| `workload` | SimpleWorkload | No | Sequential workload specification. All ranks execute the same ordered phases. Mutually exclusive with groups. Not all algorithms are valid for every operation; see GET /api/v1/simple/compatibility for the full operation/algorithm compatibility matrix including defaults and power-of-two constraints. |
| `groups` | array[SimpleGroupSpec] | No | Parallel groups: ranks partitioned into independent groups executing concurrently. Mutually exclusive with workload. |

**Example: 128-rank AllReduce with default topology**

```json
{
  "workload": {
    "ranks": 128,
    "phases": [
      {
        "operationType": "allReduce",
        "dataSize": 1048576
      }
    ]
  }
}
```

**Example: Custom topology with 192 ranks**

```json
{
  "name": "custom-topology-test",
  "topologyConstraints": {
    "switchRadix": "128x400G",
    "subscriptionRatio": "3:1"
  },
  "workload": {
    "ranks": 192,
    "phases": [
      {
        "operationType": "allReduce",
        "algorithm": "ring",
        "dataSize": 1048576
      }
    ]
  }
}
```

**Example: Multiple phases in one submission**

```json
{
  "workload": {
    "ranks": 256,
    "phases": [
      {
        "operationType": "allReduce",
        "dataSize": 1024
      },
      {
        "operationType": "allGather",
        "dataSize": 1048576
      },
      {
        "operationType": "reduceScatter",
        "dataSize": 67108864
      }
    ]
  }
}
```

**Example: AllToAll with sequential algorithm**

```json
{
  "workload": {
    "ranks": 128,
    "phases": [
      {
        "operationType": "allToAll",
        "algorithm": "sequential",
        "dataSize": 524288
      }
    ]
  }
}
```

**Example: Broadcast followed by scatter**

```json
{
  "workload": {
    "ranks": 64,
    "phases": [
      {
        "operationType": "broadcast",
        "algorithm": "interleaved",
        "dataSize": 2097152
      },
      {
        "operationType": "scatter",
        "algorithm": "direct",
        "dataSize": 1048576
      }
    ]
  }
}
```

**Example: Parallel groups executing concurrently**

```json
{
  "name": "multi-group-benchmark",
  "groups": [
    {
      "count": 4,
      "groupSize": 64,
      "phases": [
        {
          "operationType": "allReduce",
          "algorithm": "ring",
          "dataSize": 1048576
        }
      ]
    }
  ]
}
```

**Example: Multiple group specs with different operations**

```json
{
  "name": "mixed-workload-benchmark",
  "topologyConstraints": {
    "switchRadix": "64x800G",
    "subscriptionRatio": "1:1"
  },
  "groups": [
    {
      "count": 2,
      "groupSize": 128,
      "phases": [
        {
          "operationType": "allReduce",
          "algorithm": "recursiveHalvingDoubling",
          "dataSize": 4194304
        }
      ]
    },
    {
      "count": 4,
      "groupSize": 32,
      "phases": [
        {
          "operationType": "allGather",
          "dataSize": 1048576
        },
        {
          "operationType": "reduceScatter",
          "algorithm": "halving",
          "dataSize": 2097152
        }
      ]
    }
  ]
}
```

### Responses

**202** - Simulation accepted for processing

```json
{
  "simulationId": "sim_abc123",
  "status": "accepted",
  "mode": "sequential",
  "workload": {
    "ranks": 128,
    "phases": [
      {
        "operationType": "allReduce",
        "algorithm": "ring",
        "dataSize": 1048576
      }
    ]
  }
}
```

**400** - Validation error — invalid rank count, unsupported operation/algorithm combination, both topology and topologyConstraints set, or constraint solver cannot satisfy the request

*Rank count not in reference set (no constraints):*

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "ranks must be one of [8, 16, 32, 64, 128, 256, 512, 1024, 2048, 4096], got 100"
  }
}
```

*Algorithm not valid for operation:*

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "algorithm 'ring' is not compatible with operation 'allToAll'; allowed algorithms: sequential, interleaved"
  }
}
```

*Both topology and topologyConstraints provided:*

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "topologyConstraints and topology are mutually exclusive"
  }
}
```

**401** - Missing or invalid authentication token

---

## Preview the derived topology for a simple simulation request

<span class="api-method api-method-post">POST</span> `/api/v1/simple/simulations/preview`

Runs the full validation and topology derivation pipeline against the provided workload or groups, then returns the physical network topology that would be produced — without creating a simulation, traceset, or configuration. The request body is identical to POST /api/v1/simple/simulations. The workload/groups payload is required because the total rank count drives topology derivation (rack count, tier selection, switch allocation), but no workload details are echoed in the response. The response describes only the physical network: tier structure, switch counts, link counts, data rates, and resolved constraint values.

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | No | Optional human-readable name for the simulation. Auto-generated when omitted. SECURITY: this value is interpolated into command strings and Slurm entrypoints executed on cluster nodes, so it is restricted to letters, digits, '-' and '_' (1–255 characters), and must start with a letter or digit (no whitespace, other punctuation, or non-ASCII characters, and no leading '-' or '_'). |
| `topology` | string | No | Optional topology reference (reserved for K2 orchestration). |
| `topologyConstraints` | TopologyConstraints | No | Hardware constraints for topology derivation. The solver derives a valid CLOS topology from the specified switch radix and oversubscription ratio. A 2-tier topology is used when ranks fit within the radix capacity (ports × hosts_per_leaf); otherwise a 3-tier pod-based topology (RSW → FSW → SSW) is derived with all switches fully utilized. When present, rank count is not restricted to the reference values — any count satisfying the constraints is valid. |
| `workload` | SimpleWorkload | No | Sequential workload specification. All ranks execute the same ordered phases. Mutually exclusive with groups. Not all algorithms are valid for every operation; see GET /api/v1/simple/compatibility for the full operation/algorithm compatibility matrix including defaults and power-of-two constraints. |
| `groups` | array[SimpleGroupSpec] | No | Parallel groups: ranks partitioned into independent groups executing concurrently. Mutually exclusive with workload. |

**Example: 128-rank AllReduce — produces 2-tier topology**

```json
{
  "workload": {
    "ranks": 128,
    "phases": [
      {
        "operationType": "allReduce",
        "dataSize": 1048576
      }
    ]
  }
}
```

**Example: 8 ranks with constraints — produces 1-tier (single rack)**

```json
{
  "topologyConstraints": {
    "switchRadix": "128x400G",
    "subscriptionRatio": "1:1"
  },
  "workload": {
    "ranks": 8,
    "phases": [
      {
        "operationType": "allReduce",
        "dataSize": 1024
      }
    ]
  }
}
```

**Example: 16384 ranks — produces 3-tier pod topology**

```json
{
  "topologyConstraints": {
    "switchRadix": "128x400G",
    "subscriptionRatio": "1:1"
  },
  "workload": {
    "ranks": 16384,
    "phases": [
      {
        "operationType": "allReduce",
        "dataSize": 1048576
      }
    ]
  }
}
```

### Responses

**200** - Topology preview computed successfully

```json
{
  "topology": {
    "tierCount": 2,
    "totalHosts": 128,
    "ranksPerHost": 1,
    "ranksPerRack": 64,
    "tiers": [
      {
        "name": "RSW",
        "switchCount": 2,
        "downlinksPerSwitch": 64,
        "uplinksPerSwitch": 64,
        "downlinkDataRateGbps": 400,
        "uplinkDataRateGbps": 400,
        "effectiveSubscriptionRatio": "1:1",
        "switchCapacityUtilization": 100.0
      },
      {
        "name": "FSW",
        "switchCount": 1,
        "downlinksPerSwitch": 128,
        "uplinksPerSwitch": 0,
        "downlinkDataRateGbps": 400,
        "uplinkDataRateGbps": 0,
        "switchCapacityUtilization": 100.0
      }
    ],
    "links": {
      "hostToRsw": 128,
      "rswToFsw": 128,
      "fswToSsw": 0
    },
    "resolvedConstraints": {
      "switchRadix": "128x400G",
      "subscriptionRatio": "1:1",
      "nicNetworkPortSpeed": 400,
      "effectiveNicSpeed": 400,
      "switchLinkSpeed": 400,
      "switchingCapacityGbps": 51200
    }
  }
}
```

**400** - Validation error — identical to POST /api/v1/simple/simulations

*Rank count not in reference set (no constraints):*

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "ranks must be one of [8, 16, 32, 64, 128, 256, 512, 1024, 2048, 4096], got 100"
  }
}
```

**401** - Missing or invalid authentication token

---

## Get the operation/algorithm compatibility matrix

<span class="api-method api-method-get">GET</span> `/api/v1/simple/compatibility`

Returns the full compatibility matrix mapping each collective operation type to its allowed algorithms, platform default, and power-of-two constraints. This is a static reference endpoint — the matrix does not change between requests.

### Responses

**200** - Compatibility matrix

```json
{
  "entries": [
    {
      "operationType": "...",
      "allowedAlgorithms": [
        "..."
      ],
      "defaultAlgorithm": "...",
      "powerOfTwoRequired": [
        "..."
      ]
    }
  ]
}
```

**401** - Missing or invalid authentication token

---
