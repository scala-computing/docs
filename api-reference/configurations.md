---
title: "Configurations"
description: "Workload configuration management"
---

Workload configuration management

{/* AUTO-GENERATED CONTENT BELOW - DO NOT EDIT MANUALLY */}
{/* Generated from OpenAPI spec by generate-api-reference.py */}

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/configurations` | List configurations |
| POST | `/api/v1/configurations` | Create configuration |
| GET | `/api/v1/configurations/{config_id}` | Get configuration |
| PATCH | `/api/v1/configurations/{config_id}` | Update configuration |
| DELETE | `/api/v1/configurations/{config_id}` | Delete configuration |
| GET | `/api/v1/configurations/{config_id}/applications` | List configuration applications |
| POST | `/api/v1/configurations/{config_id}/applications` | Add application to configuration |
| DELETE | `/api/v1/configurations/{config_id}/applications/{app_name}` | Remove application from configuration |
| PATCH | `/api/v1/configurations/{config_id}/applications/{app_name}/parameters` | Update application parameters |
| GET | `/api/v1/configurations/{config_id}/components` | List configuration components |
| POST | `/api/v1/configurations/{config_id}/components` | Add component to configuration |
| DELETE | `/api/v1/configurations/{config_id}/components/{component_name}` | Remove component from configuration |
| PATCH | `/api/v1/configurations/{config_id}/components/{component_name}/parameters` | Update component parameters |
| PATCH | `/api/v1/configurations/{config_id}/configurations/{component_name}` | Add child to component |
| PATCH | `/api/v1/configurations/{config_id}/configurations/{component_name}/components/{child_name}` | Update child in component |
| DELETE | `/api/v1/configurations/{config_id}/configurations/{component_name}/components/{child_name}` | Remove child from component |
| GET | `/api/v1/configurations/{config_id}/containers` | List configuration containers |
| PATCH | `/api/v1/configurations/{config_id}/containers/{container_name}` | Add component to container |
| GET | `/api/v1/configurations/{config_id}/containers/{container_name}/allowed` | Get allowed component types for container |
| PATCH | `/api/v1/configurations/{config_id}/containers/{container_name}/components/{component_name}` | Update component in container |
| DELETE | `/api/v1/configurations/{config_id}/containers/{container_name}/components/{component_name}` | Remove component from container |
| PATCH | `/api/v1/configurations/{config_id}/distribution` | Update distribution entry value |
| GET | `/api/v1/configurations/{config_id}/links` | List configuration links |
| PATCH | `/api/v1/configurations/{config_id}/links/{link_name}/parameters` | Update link parameters |
| PATCH | `/api/v1/configurations/{config_id}/parameters` | Update global parameters |
| GET | `/api/v1/configurations/{config_id}/software-version` | Get software version |
| PATCH | `/api/v1/configurations/{config_id}/topology` | Add component to topology |
| PATCH | `/api/v1/configurations/{config_id}/topology/{component_name}` | Update topology component |
| DELETE | `/api/v1/configurations/{config_id}/topology/{component_name}` | Remove component from topology |
| GET | `/api/v1/configurations/{config_id}/traceset` | List available tracesets for configuration |
| PATCH | `/api/v1/configurations/{config_id}/traceset` | Attach traceset to configuration |
| POST | `/api/v1/configurations/{config_id}/validate` | Validate configuration |
| GET | `/api/v1/configurations/{config_id}/versions` | List configuration versions |
| GET | `/api/v1/configurations/{config_id}/versions/{version}` | Get specific configuration version |

---

## List configurations

<span class="api-method api-method-get">GET</span> `/api/v1/configurations`

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `limit` | query | integer | No | - |
| `next` | query | string | No | Opaque cursor token to fetch results after a specific item |
| `workspaceId` | query | string | No | - |
| `status` | query | string (enum) | No | Filter by configuration status (comma-separated for multiple) |
| `search` | query | string | No | Full-text search across name and description fields (case-insensitive) |
| `createdAfter` | query | string | No | Filter to items created on or after this timestamp (ISO 8601) |
| `createdBefore` | query | string | No | Filter to items created before this timestamp (ISO 8601) |

### Responses

**200** - List of configurations

```json
{
  "configurations": [
    {
      "componentCount": 1,
      "createdAt": "2024-01-15T10:30:00Z",
      "id": "string",
      "modifiedAt": "2024-01-15T10:30:00Z",
      "name": "string",
      "status": "...",
      "version": 1,
      "workspaceId": "string"
    }
  ],
  "pagination": {
    "count": 1,
    "hasMore": true,
    "nextCursor": "string"
  }
}
```

---

## Create configuration

<span class="api-method api-method-post">POST</span> `/api/v1/configurations`

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `activeTraceset` | any | No | Active traceset for chakra-based simulations. Required when any application has trafficType 'chakra'. |
| `applications` | array[ApplicationDefinition] | Yes | Traffic generator and workload applications |
| `components` | array[ComponentDeclaration] | Yes | Component declarations |
| `containers` | array[ContainerDefinition] | Yes | Container definitions |
| `globalParameters` | TypedParameters | Yes | Hierarchical parameters structure. Leaf nodes are ParameterValue objects, branch nodes are nested TypedParameters representing ModelComponents. |
| `links` | array[ComponentDeclaration] | Yes | Layer components defining spine plane linking configuration (type='layer') |
| `name` | string | Yes | Configuration name |
| `topology` | TopologyDefinition | Yes | Active network topology definition |
| `workspaceId` | string | No | Workspace identifier. If omitted, defaults to the well-known default workspace. |

```json
{
  "activeTraceset": {},
  "applications": [
    {
      "model": "ChakraWorkload",
      "name": "ScalaChakraGenerator",
      "trafficType": "...",
      "type": "...",
      "typedParameters": "...",
      "version": "4.0.1"
    }
  ],
  "components": [
    {
      "allowedComponents": [
        "..."
      ],
      "components": [
        "..."
      ],
      "model": "ScalaSwitch",
      "name": "spine-switch-1",
      "trafficType": "...",
      "type": "...",
      "typedParameters": "...",
      "version": "3.2.1"
    }
  ],
  "containers": [
    {
      "allowedComponents": [
        "..."
      ],
      "components": [
        "..."
      ],
      "configuration": null,
      "model": "ScalaRack",
      "name": "roce-rack",
      "type": "...",
      "version": "4.0.0"
    }
  ],
  "globalParameters": {},
  "links": [
    {
      "allowedComponents": [
        "..."
      ],
      "components": [
        "..."
      ],
      "model": "ScalaSwitch",
      "name": "spine-switch-1",
      "trafficType": "...",
      "type": "...",
      "typedParameters": "...",
      "version": "3.2.1"
    }
  ],
  "name": "string",
  "topology": {
    "allowedComponents": [
      "string"
    ],
    "appDistribution": {
      "description": "...",
      "randomApplicationDistribution": "..."
    },
    "components": [
      "..."
    ],
    "name": null,
    "topologyTiers": 1,
    "type": "clos"
  },
  "workspaceId": "string"
}
```

### Responses

**201** - Configuration created

```json
{
  "activeTraceset": {},
  "applications": [
    {
      "model": "ChakraWorkload",
      "name": "ScalaChakraGenerator",
      "trafficType": "...",
      "type": "...",
      "typedParameters": "...",
      "version": "4.0.1"
    }
  ],
  "componentCount": 1,
  "components": [
    {
      "allowedComponents": [
        "..."
      ],
      "components": [
        "..."
      ],
      "model": "ScalaSwitch",
      "name": "spine-switch-1",
      "trafficType": "...",
      "type": "...",
      "typedParameters": "...",
      "version": "3.2.1"
    }
  ],
  "containers": [
    {
      "allowedComponents": [
        "..."
      ],
      "components": [
        "..."
      ],
      "configuration": null,
      "model": "ScalaRack",
      "name": "roce-rack",
      "type": "...",
      "version": "4.0.0"
    }
  ],
  "createdAt": "2024-01-15T10:30:00Z",
  "globalParameters": {},
  "id": "config_10045",
  "links": [
    {
      "allowedComponents": [
        "..."
      ],
      "components": [
        "..."
      ],
      "model": "ScalaSwitch",
      "name": "spine-switch-1",
      "trafficType": "...",
      "type": "...",
      "typedParameters": "...",
      "version": "3.2.1"
    }
  ],
  "modifiedAt": "2024-01-15T10:30:00Z",
  "name": "string",
  "status": "draft",
  "topology": {
    "allowedComponents": [
      "string"
    ],
    "appDistribution": {
      "description": "...",
      "randomApplicationDistribution": "..."
    },
    "components": [
      "..."
    ],
    "name": null,
    "topologyTiers": 1,
    "type": "clos"
  },
  "version": 1,
  "workspaceId": "string"
}
```

---

## Get configuration

<span class="api-method api-method-get">GET</span> `/api/v1/configurations/{config_id}`

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |

### Responses

**200** - Configuration details

```json
{
  "activeTraceset": {},
  "applications": [
    {
      "model": "ChakraWorkload",
      "name": "ScalaChakraGenerator",
      "trafficType": "...",
      "type": "...",
      "typedParameters": "...",
      "version": "4.0.1"
    }
  ],
  "componentCount": 1,
  "components": [
    {
      "allowedComponents": [
        "..."
      ],
      "components": [
        "..."
      ],
      "model": "ScalaSwitch",
      "name": "spine-switch-1",
      "trafficType": "...",
      "type": "...",
      "typedParameters": "...",
      "version": "3.2.1"
    }
  ],
  "containers": [
    {
      "allowedComponents": [
        "..."
      ],
      "components": [
        "..."
      ],
      "configuration": null,
      "model": "ScalaRack",
      "name": "roce-rack",
      "type": "...",
      "version": "4.0.0"
    }
  ],
  "createdAt": "2024-01-15T10:30:00Z",
  "globalParameters": {},
  "id": "config_10045",
  "links": [
    {
      "allowedComponents": [
        "..."
      ],
      "components": [
        "..."
      ],
      "model": "ScalaSwitch",
      "name": "spine-switch-1",
      "trafficType": "...",
      "type": "...",
      "typedParameters": "...",
      "version": "3.2.1"
    }
  ],
  "modifiedAt": "2024-01-15T10:30:00Z",
  "name": "string",
  "status": "draft",
  "topology": {
    "allowedComponents": [
      "string"
    ],
    "appDistribution": {
      "description": "...",
      "randomApplicationDistribution": "..."
    },
    "components": [
      "..."
    ],
    "name": null,
    "topologyTiers": 1,
    "type": "clos"
  },
  "version": 1,
  "workspaceId": "string"
}
```

---

## Update configuration

<span class="api-method api-method-patch">PATCH</span> `/api/v1/configurations/{config_id}`

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |
| `If-Match` | header | string | Yes | ETag for optimistic concurrency |

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `activeTraceset` | any | No | Active traceset for chakra-based simulations. Required when any application has trafficType 'chakra'. |
| `applications` | array[ApplicationDefinition] | No | Traffic generator and workload applications |
| `components` | array[ComponentDeclaration] | No | Component declarations |
| `containers` | array[ContainerDefinition] | No | Container definitions |
| `globalParameters` | TypedParameters | No | Hierarchical parameters structure. Leaf nodes are ParameterValue objects, branch nodes are nested TypedParameters representing ModelComponents. |
| `links` | array[ComponentDeclaration] | No | Layer components defining spine plane linking configuration (type='layer') |
| `name` | ['string', 'null'] | No | Configuration name |
| `topology` | TopologyDefinition | No | Active network topology definition |

```json
{
  "activeTraceset": {},
  "applications": [
    {
      "model": "ChakraWorkload",
      "name": "ScalaChakraGenerator",
      "trafficType": "...",
      "type": "...",
      "typedParameters": "...",
      "version": "4.0.1"
    }
  ],
  "components": [
    {
      "allowedComponents": [
        "..."
      ],
      "components": [
        "..."
      ],
      "model": "ScalaSwitch",
      "name": "spine-switch-1",
      "trafficType": "...",
      "type": "...",
      "typedParameters": "...",
      "version": "3.2.1"
    }
  ],
  "containers": [
    {
      "allowedComponents": [
        "..."
      ],
      "components": [
        "..."
      ],
      "configuration": null,
      "model": "ScalaRack",
      "name": "roce-rack",
      "type": "...",
      "version": "4.0.0"
    }
  ],
  "globalParameters": {},
  "links": [
    {
      "allowedComponents": [
        "..."
      ],
      "components": [
        "..."
      ],
      "model": "ScalaSwitch",
      "name": "spine-switch-1",
      "trafficType": "...",
      "type": "...",
      "typedParameters": "...",
      "version": "3.2.1"
    }
  ],
  "name": null,
  "topology": {
    "allowedComponents": [
      "string"
    ],
    "appDistribution": {
      "description": "...",
      "randomApplicationDistribution": "..."
    },
    "components": [
      "..."
    ],
    "name": null,
    "topologyTiers": 1,
    "type": "clos"
  }
}
```

### Responses

**200** - Configuration updated

```json
{
  "activeTraceset": {},
  "applications": [
    {
      "model": "ChakraWorkload",
      "name": "ScalaChakraGenerator",
      "trafficType": "...",
      "type": "...",
      "typedParameters": "...",
      "version": "4.0.1"
    }
  ],
  "componentCount": 1,
  "components": [
    {
      "allowedComponents": [
        "..."
      ],
      "components": [
        "..."
      ],
      "model": "ScalaSwitch",
      "name": "spine-switch-1",
      "trafficType": "...",
      "type": "...",
      "typedParameters": "...",
      "version": "3.2.1"
    }
  ],
  "containers": [
    {
      "allowedComponents": [
        "..."
      ],
      "components": [
        "..."
      ],
      "configuration": null,
      "model": "ScalaRack",
      "name": "roce-rack",
      "type": "...",
      "version": "4.0.0"
    }
  ],
  "createdAt": "2024-01-15T10:30:00Z",
  "globalParameters": {},
  "id": "config_10045",
  "links": [
    {
      "allowedComponents": [
        "..."
      ],
      "components": [
        "..."
      ],
      "model": "ScalaSwitch",
      "name": "spine-switch-1",
      "trafficType": "...",
      "type": "...",
      "typedParameters": "...",
      "version": "3.2.1"
    }
  ],
  "modifiedAt": "2024-01-15T10:30:00Z",
  "name": "string",
  "status": "draft",
  "topology": {
    "allowedComponents": [
      "string"
    ],
    "appDistribution": {
      "description": "...",
      "randomApplicationDistribution": "..."
    },
    "components": [
      "..."
    ],
    "name": null,
    "topologyTiers": 1,
    "type": "clos"
  },
  "version": 1,
  "workspaceId": "string"
}
```

**412** - Precondition failed (ETag mismatch)

---

## Delete configuration

<span class="api-method api-method-delete">DELETE</span> `/api/v1/configurations/{config_id}`

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |

### Responses

**204** - Configuration deleted

---

## List configuration applications

<span class="api-method api-method-get">GET</span> `/api/v1/configurations/{config_id}/applications`

Returns all applications in the configuration's topology

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |
| `withDetails` | query | boolean | No | When true, includes full typedParameters for each application. Defaults to false. |

### Responses

**200** - List of applications

```json
{
  "applications": [
    {
      "model": "ChakraWorkload",
      "name": "ScalaChakraGenerator",
      "trafficType": "...",
      "type": "...",
      "typedParameters": "...",
      "version": "4.0.1"
    }
  ]
}
```

**404** - Configuration not found

---

## Add application to configuration

<span class="api-method api-method-post">POST</span> `/api/v1/configurations/{config_id}/applications`

Adds an application to the configuration's topology by looking up the model from the database. Server resolves traffic type, version, and default parameters from the model.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |
| `modelId` | query | string | Yes | Application model identifier — KSUID (model_xxx) or model name (e.g., ScalaChakraGenerator). Server resolves traffic type, version, and default parameters. |
| `name` | query | string | Yes | Unique name for this application in the configuration |
| `version` | query | string | No | Optional model version. If omitted, uses latest stable version. |

### Responses

**201** - Application added

```json
{
  "model": "ChakraWorkload",
  "name": "ScalaChakraGenerator",
  "trafficType": "chakra",
  "type": "switch",
  "typedParameters": {},
  "version": "4.0.1"
}
```

**400** - Invalid request (distribution sum exceeds 1.0)

**404** - Configuration not found

---

## Remove application from configuration

<span class="api-method api-method-delete">DELETE</span> `/api/v1/configurations/{config_id}/applications/{app_name}`

Removes an application from the configuration's topology. Remaining applications are auto-scaled to maintain distribution sum of 1.0.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |
| `app_name` | path | string | Yes | - |

### Responses

**200** - Application removed successfully

```json
{
  "id": "string",
  "message": "string",
  "status": "string"
}
```

**404** - Configuration or application not found

---

## Update application parameters

<span class="api-method api-method-patch">PATCH</span> `/api/v1/configurations/{config_id}/applications/{app_name}/parameters`

Performs a validated merge of the provided TypedParameters patch into the named application's typed_parameters. Only existing parameter keys can be updated, and types must match.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |
| `app_name` | path | string | Yes | - |
| `If-Match` | header | string | No | ETag for optimistic concurrency control |

### Request Body

### Responses

**200** - Parameters updated successfully. Returns the full updated TypedParameters for the application.

**400** - Invalid request (empty patch, unknown parameter key, type mismatch, structural mismatch)

**404** - Configuration or application not found

**409** - Conflict (serialization failure from concurrent modification)

**412** - Precondition failed (ETag mismatch)

**409** - Traceset rank count not yet resolved — metadata extraction is still in progress. Retry after extraction completes.

**422** - Traceset rank count is present but invalid (zero or negative). The traceset data is malformed and cannot be used for a Chakra simulation.

---

## List configuration components

<span class="api-method api-method-get">GET</span> `/api/v1/configurations/{config_id}/components`

Returns all components from the configuration's components array with optional type filtering

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |
| `type` | query | string | No | Filter by component type (switch, nic, server, host) |
| `limit` | query | integer | No | - |
| `next` | query | string | No | Opaque cursor token for pagination |
| `withDetails` | query | boolean | No | When true, includes full typedParameters for each component. Defaults to false. |

### Responses

**200** - List of components

```json
{
  "components": [
    {
      "container": null,
      "model": "ScalaSwitch",
      "name": "string",
      "type": "...",
      "typedParameters": "...",
      "version": "3.2.1"
    }
  ],
  "pagination": {
    "count": 1,
    "hasMore": true,
    "nextCursor": "string"
  }
}
```

**404** - Configuration not found

---

## Add component to configuration

<span class="api-method api-method-post">POST</span> `/api/v1/configurations/{config_id}/components`

Adds a component, container, or layer to the configuration by looking up the model from the database. For switch/nic/server: adds to components array. For rack/pod: adds to containers array. For layer: adds to links array.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |
| `modelId` | query | string | Yes | Model identifier — KSUID (model_xxx) or model name (e.g., ScalaSwitch). Server resolves version and default parameters. |
| `type` | query | string | Yes | Component type: switch, nic, server, rack, pod, or layer |
| `name` | query | string | Yes | Unique name for this component in the configuration |
| `version` | query | string | No | Optional model version. If omitted, uses latest stable version. |

### Responses

**201** - Component added

```json
{
  "allowedComponents": [
    "string"
  ],
  "components": [
    {
      "count": 4,
      "group": null,
      "model": "ScalaSwitch",
      "name": "spine-1",
      "parentApplicationName": null,
      "type": "switch",
      "version": "3.2.1"
    }
  ],
  "model": "ScalaSwitch",
  "name": "spine-switch-1",
  "trafficType": "chakra",
  "type": "switch",
  "typedParameters": {},
  "version": "3.2.1"
}
```

**400** - Invalid request (model not found, invalid parameters)

**404** - Configuration not found

---

## Remove component from configuration

<span class="api-method api-method-delete">DELETE</span> `/api/v1/configurations/{config_id}/components/{component_name}`

Removes a component from the configuration's components array. Cascades removal to all container references, topology references, and child references in other components.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |
| `component_name` | path | string | Yes | - |
| `If-Match` | header | string | No | ETag for optimistic concurrency control |

### Responses

**204** - Component removed from configuration

**404** - Configuration or component not found

**412** - Precondition failed (ETag mismatch)

---

## Update component parameters

<span class="api-method api-method-patch">PATCH</span> `/api/v1/configurations/{config_id}/components/{component_name}/parameters`

Performs a validated merge of the provided TypedParameters patch into the named component's typed_parameters. Only existing parameter keys can be updated, and types must match.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |
| `component_name` | path | string | Yes | - |
| `If-Match` | header | string | No | ETag for optimistic concurrency control |

### Request Body

### Responses

**200** - Parameters updated successfully. Returns the full updated TypedParameters for the component.

**400** - Invalid request (empty patch, unknown parameter key, type mismatch, structural mismatch)

**404** - Configuration or component not found

**409** - Conflict (serialization failure from concurrent modification)

**412** - Precondition failed (ETag mismatch)

---

## Add child to component

<span class="api-method api-method-patch">PATCH</span> `/api/v1/configurations/{config_id}/configurations/{component_name}`

Adds a child component (NIC, GPU, or application) to the specified host (server) component. Validates that the child type is in the host's allowedComponents list.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |
| `component_name` | path | string | Yes | - |
| `If-Match` | header | string | No | ETag for optimistic concurrency control |

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `count` | integer | No | Number of instances (default 1) |
| `group` | string | No | Logical group for relating components |
| `name` | string | Yes | Name of an existing NIC, GPU, or application component in the configuration |
| `parentApplicationName` | string | No | Parent application name (required for application type children) |
| `type` | string | Yes | Child component type (nic, gpu, application) |

```json
{
  "count": 1,
  "group": "string",
  "name": "string",
  "parentApplicationName": "string",
  "type": "nic"
}
```

### Responses

**200** - Child component added to host

```json
{
  "allowedComponents": [
    "string"
  ],
  "components": [
    {
      "count": 4,
      "group": null,
      "model": "ScalaSwitch",
      "name": "spine-1",
      "parentApplicationName": null,
      "type": "switch",
      "version": "3.2.1"
    }
  ],
  "model": "ScalaSwitch",
  "name": "spine-switch-1",
  "trafficType": "chakra",
  "type": "switch",
  "typedParameters": {},
  "version": "3.2.1"
}
```

**400** - Invalid request (child type not allowed, already attached)

**404** - Configuration or component not found

---

## Update child in component

<span class="api-method api-method-patch">PATCH</span> `/api/v1/configurations/{config_id}/configurations/{component_name}/components/{child_name}`

Updates count and/or group on an existing child component reference within the specified host (server) component. At least one of count or group must be provided.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |
| `component_name` | path | string | Yes | - |
| `child_name` | path | string | Yes | - |
| `If-Match` | header | string | No | ETag for optimistic concurrency control |

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `count` | integer | No | Updated instance count |
| `group` | string | No | Updated logical group assignment |

```json
{
  "count": 1,
  "group": "string"
}
```

### Responses

**200** - Child component updated

```json
{
  "allowedComponents": [
    "string"
  ],
  "components": [
    {
      "count": 4,
      "group": null,
      "model": "ScalaSwitch",
      "name": "spine-1",
      "parentApplicationName": null,
      "type": "switch",
      "version": "3.2.1"
    }
  ],
  "model": "ScalaSwitch",
  "name": "spine-switch-1",
  "trafficType": "chakra",
  "type": "switch",
  "typedParameters": {},
  "version": "3.2.1"
}
```

**400** - Invalid request (empty patch, count < 1)

**404** - Configuration, component, or child not found

**412** - Precondition failed (ETag mismatch)

---

## Remove child from component

<span class="api-method api-method-delete">DELETE</span> `/api/v1/configurations/{config_id}/configurations/{component_name}/components/{child_name}`

Removes a child component (NIC, GPU, or application) from the specified host (server) component.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |
| `component_name` | path | string | Yes | - |
| `child_name` | path | string | Yes | - |
| `If-Match` | header | string | No | ETag for optimistic concurrency control |

### Responses

**204** - Child component removed from host

**404** - Configuration, component, or child not found

**412** - Precondition failed (ETag mismatch)

---

## List configuration containers

<span class="api-method api-method-get">GET</span> `/api/v1/configurations/{config_id}/containers`

Returns all containers from the configuration with optional type filtering

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |
| `type` | query | string | No | Filter by container type (rack, pod) |
| `limit` | query | integer | No | - |
| `next` | query | string | No | Opaque cursor token for pagination |

### Responses

**200** - List of containers

```json
{
  "containers": [
    {
      "allowedComponents": [
        "..."
      ],
      "components": [
        "..."
      ],
      "configuration": null,
      "model": "ScalaRack",
      "name": "roce-rack",
      "type": "...",
      "version": "4.0.0"
    }
  ],
  "pagination": {
    "count": 1,
    "hasMore": true,
    "nextCursor": "string"
  }
}
```

**404** - Configuration not found

---

## Add component to container

<span class="api-method api-method-patch">PATCH</span> `/api/v1/configurations/{config_id}/containers/{container_name}`

Adds a component reference to the specified container. Validates that the component type is in the container's allowed list.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |
| `container_name` | path | string | Yes | - |

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `count` | integer | No | Number of instances (default 1) |
| `group` | string | No | Logical group for relating components |
| `name` | string | Yes | Name of an existing component or container in the configuration to add to this container |
| `type` | string | Yes | Component type (switch, nic, server, rack, pod) |

```json
{
  "count": 1,
  "group": "string",
  "name": "my-spine-switch",
  "type": "switch"
}
```

### Responses

**200** - Component added to container

```json
{
  "allowedComponents": [
    "string"
  ],
  "components": [
    {
      "count": 4,
      "group": null,
      "model": "ScalaSwitch",
      "name": "spine-1",
      "parentApplicationName": null,
      "type": "switch",
      "version": "3.2.1"
    }
  ],
  "configuration": null,
  "model": "ScalaRack",
  "name": "roce-rack",
  "type": "rack",
  "version": "4.0.0"
}
```

**400** - Component type not allowed in container

**404** - Configuration or container not found

---

## Get allowed component types for container

<span class="api-method api-method-get">GET</span> `/api/v1/configurations/{config_id}/containers/{container_name}/allowed`

Returns the list of component types allowed in this container based on its type

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |
| `container_name` | path | string | Yes | - |

### Responses

**200** - Allowed component types

```json
{
  "allowedComponents": [
    "switch",
    "nic",
    "server",
    "application",
    "layer"
  ],
  "containerName": "string",
  "containerType": "rack"
}
```

**404** - Configuration or container not found

---

## Update component in container

<span class="api-method api-method-patch">PATCH</span> `/api/v1/configurations/{config_id}/containers/{container_name}/components/{component_name}`

Updates count and/or group on an existing component reference in the specified container. At least one of count or group must be provided.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |
| `container_name` | path | string | Yes | - |
| `component_name` | path | string | Yes | - |
| `If-Match` | header | string | No | ETag for optimistic concurrency control |

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `count` | integer | No | Updated instance count |
| `group` | string | No | Updated logical group assignment |

```json
{
  "count": 1,
  "group": "string"
}
```

### Responses

**200** - Component updated in container

```json
{
  "allowedComponents": [
    "string"
  ],
  "components": [
    {
      "count": 4,
      "group": null,
      "model": "ScalaSwitch",
      "name": "spine-1",
      "parentApplicationName": null,
      "type": "switch",
      "version": "3.2.1"
    }
  ],
  "configuration": null,
  "model": "ScalaRack",
  "name": "roce-rack",
  "type": "rack",
  "version": "4.0.0"
}
```

**400** - Invalid request (empty patch, count < 1)

**404** - Configuration, container, or component not found

**412** - Precondition failed (ETag mismatch)

---

## Remove component from container

<span class="api-method api-method-delete">DELETE</span> `/api/v1/configurations/{config_id}/containers/{container_name}/components/{component_name}`

Removes a component reference from the specified container

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |
| `container_name` | path | string | Yes | - |
| `component_name` | path | string | Yes | - |

### Responses

**204** - Component removed from container

**404** - Configuration, container, or component not found

---

## Update distribution entry value

<span class="api-method api-method-patch">PATCH</span> `/api/v1/configurations/{config_id}/distribution`

Updates the value field of a specific entry in topology.appDistribution.randomApplicationDistribution[]. The entry is identified by parentApplication + distributionId.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |
| `If-Match` | header | string | No | ETag for optimistic concurrency control |

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `distributionId` | string | Yes | Distribution identifier linking to prClientVariable (e.g., prScalaTrafficServer) |
| `parentApplication` | string | Yes | Root application name (e.g., ScalaTrafficGeneratorApp) |
| `value` | number | Yes | New distribution percentage value (0.0 to 1.0 inclusive) |

```json
{
  "distributionId": "string",
  "parentApplication": "string",
  "value": 1.0
}
```

### Responses

**200** - Distribution value updated

```json
{
  "description": null,
  "randomApplicationDistribution": [
    {
      "applicationType": "client",
      "distributionPercent": "...",
      "name": "string",
      "parentApplication": "string"
    }
  ]
}
```

**400** - Invalid request (value out of range, empty fields, no app_distribution)

**404** - Configuration or distribution entry not found

**409** - Concurrent modification conflict, please retry

**412** - Precondition failed (ETag mismatch)

---

## List configuration links

<span class="api-method api-method-get">GET</span> `/api/v1/configurations/{config_id}/links`

Returns all links (layer components) from the configuration

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |
| `withDetails` | query | boolean | No | When true, includes full typedParameters for each link. Defaults to false. |

### Responses

**200** - List of links

```json
{
  "links": [
    {
      "allowedComponents": [
        "..."
      ],
      "components": [
        "..."
      ],
      "model": "ScalaSwitch",
      "name": "spine-switch-1",
      "trafficType": "...",
      "type": "...",
      "typedParameters": "...",
      "version": "3.2.1"
    }
  ]
}
```

**404** - Configuration not found

---

## Update link parameters

<span class="api-method api-method-patch">PATCH</span> `/api/v1/configurations/{config_id}/links/{link_name}/parameters`

Performs a validated merge of the provided TypedParameters patch into the named link's typed_parameters. Only existing parameter keys can be updated, and types must match.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |
| `link_name` | path | string | Yes | - |
| `If-Match` | header | string | No | ETag for optimistic concurrency control |

### Request Body

### Responses

**200** - Parameters updated successfully. Returns the full updated TypedParameters for the link.

**400** - Invalid request (empty patch, unknown parameter key, type mismatch, structural mismatch)

**404** - Configuration or link not found

**409** - Conflict (serialization failure from concurrent modification)

**412** - Precondition failed (ETag mismatch)

---

## Update global parameters

<span class="api-method api-method-patch">PATCH</span> `/api/v1/configurations/{config_id}/parameters`

Performs a validated merge of the provided TypedParameters patch into the configuration's global_parameters. Backend-managed sections (ComputeConfiguration, ParallelizationParameters, etc.), readonly sections (SoftwareVersion), and hidden parameters are rejected with 400.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |
| `If-Match` | header | string | No | ETag for optimistic concurrency control |

### Request Body

### Responses

**200** - Parameters updated successfully. Returns the full updated global TypedParameters (with backend params stripped).

**400** - Invalid request (empty patch, unknown parameter key, type mismatch, structural mismatch, attempt to modify backend-managed/readonly/hidden params)

**404** - Configuration not found

**409** - Conflict (serialization failure from concurrent modification)

**412** - Precondition failed (ETag mismatch)

---

## Get software version

<span class="api-method api-method-get">GET</span> `/api/v1/configurations/{config_id}/software-version`

Returns the simulator software version from the SoftwareVersion section of globalParameters.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |

### Responses

**200** - Software version retrieved successfully

```json
{
  "version": "string"
}
```

**404** - Configuration not found

---

## Add component to topology

<span class="api-method api-method-patch">PATCH</span> `/api/v1/configurations/{config_id}/topology`

Adds a component reference to the configuration's topology.components array. Validates the named entity exists in the correct collection by type (switch→components, layer→links, pod→containers).

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |
| `If-Match` | header | string | No | ETag for optimistic concurrency control |

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `count` | integer | No | Number of instances (default 1) |
| `name` | string | Yes | Name of existing component, container, or link to reference |
| `type` | string | Yes | Type of the referenced entity |

```json
{
  "count": 1,
  "name": "my-spine-switch",
  "type": "switch"
}
```

### Responses

**200** - Component added to topology

```json
{
  "allowedComponents": [
    "string"
  ],
  "appDistribution": {
    "description": null,
    "randomApplicationDistribution": [
      "..."
    ]
  },
  "components": [
    {
      "count": 4,
      "group": null,
      "model": "ScalaSwitch",
      "name": "spine-1",
      "parentApplicationName": null,
      "type": "switch",
      "version": "3.2.1"
    }
  ],
  "name": null,
  "topologyTiers": 1,
  "type": "clos"
}
```

**400** - Invalid request (type not allowed, duplicate, invalid type)

**404** - Configuration or referenced entity not found

---

## Update topology component

<span class="api-method api-method-patch">PATCH</span> `/api/v1/configurations/{config_id}/topology/{component_name}`

Updates the count on an existing component reference in the topology.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |
| `component_name` | path | string | Yes | - |
| `If-Match` | header | string | No | ETag for optimistic concurrency control |

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `count` | integer | Yes | Updated instance count |

```json
{
  "count": 1
}
```

### Responses

**200** - Topology component updated

```json
{
  "allowedComponents": [
    "string"
  ],
  "appDistribution": {
    "description": null,
    "randomApplicationDistribution": [
      "..."
    ]
  },
  "components": [
    {
      "count": 4,
      "group": null,
      "model": "ScalaSwitch",
      "name": "spine-1",
      "parentApplicationName": null,
      "type": "switch",
      "version": "3.2.1"
    }
  ],
  "name": null,
  "topologyTiers": 1,
  "type": "clos"
}
```

**400** - Invalid request (count < 1)

**404** - Configuration or component not found in topology

**412** - Precondition failed (ETag mismatch)

---

## Remove component from topology

<span class="api-method api-method-delete">DELETE</span> `/api/v1/configurations/{config_id}/topology/{component_name}`

Removes a component reference from the configuration's topology.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |
| `component_name` | path | string | Yes | - |
| `If-Match` | header | string | No | ETag for optimistic concurrency control |

### Responses

**204** - Component removed from topology

**404** - Configuration or component not found in topology

**412** - Precondition failed (ETag mismatch)

---

## List available tracesets for configuration

<span class="api-method api-method-get">GET</span> `/api/v1/configurations/{config_id}/traceset`

Returns tracesets that can be attached to configuration applications. Use scope=global to list all tracesets across all workspaces, or scope=workspace to filter by the configuration's workspace only.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |
| `status` | query | string | No | Filter by traceset status (completed, uploading, failed) |
| `limit` | query | integer | No | - |
| `next` | query | string | No | Opaque cursor token for pagination |
| `scope` | query | string (enum) | No | Scope filter for tracesets. 'global' returns all tracesets across all workspaces (default). 'workspace' returns only tracesets in the configuration's workspace. |

### Responses

**200** - List of available tracesets

```json
{
  "pagination": {
    "count": 1,
    "hasMore": true,
    "nextCursor": "string"
  },
  "tracesets": [
    {
      "createdAt": "2024-01-15T10:30:00Z",
      "description": null,
      "format": "chakra_json_v1",
      "hash": "crc64nvme:abc123...",
      "hashShort": "abc123def456",
      "id": "ts_2RKHfGD5Z8vW9pL3NqM7TjX1YcB",
      "metadataStatus": "pending",
      "name": "string",
      "status": "uploading",
      "totalSize": 1,
      "updatedAt": "2024-01-15T10:30:00Z"
    }
  ]
}
```

**404** - Configuration not found

---

## Attach traceset to configuration

<span class="api-method api-method-patch">PATCH</span> `/api/v1/configurations/{config_id}/traceset`

Sets the activeTraceset field on the configuration to reference the specified traceset. Requires at least one application with trafficType 'chakra' to exist in the configuration. Workspace-agnostic: any traceset can be attached regardless of workspace.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |
| `If-Match` | header | string | No | ETag for optimistic concurrency control |

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `tracesetId` | string | Yes | Traceset ID to attach (workspace-agnostic) |

```json
{
  "tracesetId": "ts_2RKHfGD5Z8vW9pL3NqM7TjX1YcB"
}
```

### Responses

**200** - Traceset attached successfully

```json
{
  "activeTraceset": {
    "id": "string",
    "name": "string"
  }
}
```

**400** - Invalid request (traceset not found, no Chakra application in configuration)

**404** - Configuration not found

**409** - Traceset rank count not yet resolved — metadata extraction is still in progress. Retry after extraction completes.

**412** - Precondition failed (ETag mismatch)

**422** - Traceset rank count is present but invalid (zero or negative). The traceset data is malformed and cannot be used for a Chakra simulation.

---

## Validate configuration

<span class="api-method api-method-post">POST</span> `/api/v1/configurations/{config_id}/validate`

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |

### Responses

**200** - Validation result

```json
{
  "errors": [
    {
      "code": "CONSTRAINT_VIOLATION",
      "component": null,
      "message": "string",
      "parameter": null,
      "path": "$.components[2].typedParameters.numUpLinks",
      "severity": "..."
    }
  ],
  "info": [
    {
      "code": "CONSTRAINT_VIOLATION",
      "component": null,
      "message": "string",
      "parameter": null,
      "path": "$.components[2].typedParameters.numUpLinks",
      "severity": "..."
    }
  ],
  "status": "valid",
  "valid": true,
  "warnings": [
    {
      "code": "CONSTRAINT_VIOLATION",
      "component": null,
      "message": "string",
      "parameter": null,
      "path": "$.components[2].typedParameters.numUpLinks",
      "severity": "..."
    }
  ]
}
```

---

## List configuration versions

<span class="api-method api-method-get">GET</span> `/api/v1/configurations/{config_id}/versions`

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |

### Responses

**200** - List of version summaries

```json
{
  "versions": [
    {
      "createdAt": "2024-01-15T10:30:00Z",
      "version": 1
    }
  ]
}
```

---

## Get specific configuration version

<span class="api-method api-method-get">GET</span> `/api/v1/configurations/{config_id}/versions/{version}`

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `config_id` | path | string | Yes | - |
| `version` | path | integer | Yes | - |

### Responses

**200** - Configuration version details

```json
{
  "activeTraceset": {},
  "applications": [
    {
      "model": "ChakraWorkload",
      "name": "ScalaChakraGenerator",
      "trafficType": "...",
      "type": "...",
      "typedParameters": "...",
      "version": "4.0.1"
    }
  ],
  "components": [
    {
      "allowedComponents": [
        "..."
      ],
      "components": [
        "..."
      ],
      "model": "ScalaSwitch",
      "name": "spine-switch-1",
      "trafficType": "...",
      "type": "...",
      "typedParameters": "...",
      "version": "3.2.1"
    }
  ],
  "containers": [
    {
      "allowedComponents": [
        "..."
      ],
      "components": [
        "..."
      ],
      "configuration": null,
      "model": "ScalaRack",
      "name": "roce-rack",
      "type": "...",
      "version": "4.0.0"
    }
  ],
  "createdAt": "2024-01-15T10:30:00Z",
  "globalParameters": {},
  "links": [
    {
      "allowedComponents": [
        "..."
      ],
      "components": [
        "..."
      ],
      "model": "ScalaSwitch",
      "name": "spine-switch-1",
      "trafficType": "...",
      "type": "...",
      "typedParameters": "...",
      "version": "3.2.1"
    }
  ],
  "topology": {
    "allowedComponents": [
      "string"
    ],
    "appDistribution": {
      "description": "...",
      "randomApplicationDistribution": "..."
    },
    "components": [
      "..."
    ],
    "name": null,
    "topologyTiers": 1,
    "type": "clos"
  },
  "version": 1
}
```

**404** - Version not found

---
