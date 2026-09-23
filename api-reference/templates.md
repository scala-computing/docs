---
title: "Templates"
description: "Curated example configurations"
---

Curated example configurations

{/* AUTO-GENERATED CONTENT BELOW - DO NOT EDIT MANUALLY */}
{/* Generated from OpenAPI spec by generate-api-reference.py */}

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/templates` | List templates |
| POST | `/api/v1/templates` | Create template |
| GET | `/api/v1/templates/{template_id}` | Get template details |
| POST | `/api/v1/templates/{template_id}/clone` | Clone template to workspace |

---

## List templates

<span class="api-method api-method-get">GET</span> `/api/v1/templates`

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `limit` | query | integer | No | - |
| `next` | query | string | No | Opaque cursor token to fetch results after a specific item |
| `category` | query | string | No | Filter by template category |
| `search` | query | string | No | Full-text search across name and description fields (case-insensitive) |

### Responses

**200** - List of templates

```json
{
  "templates": [
    {
      "id": "string",
      "name": "string",
      "description": "string",
      "category": "..."
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

## Create template

<span class="api-method api-method-post">POST</span> `/api/v1/templates`

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Template name |
| `description` | string | Yes | Human-readable description |
| `category` | TemplateCategory | Yes | Category of template |
| `components` | array[ComponentDeclaration] | No | Component declarations with typedParameters |
| `containers` | array[ContainerDefinition] | No | Container definitions |
| `topology` | TopologyDefinition | No | Active network topology definition |
| `globalParameters` | TypedParameters | No | Hierarchical parameters structure. Leaf nodes are ParameterValue objects, branch nodes are nested TypedParameters representing ModelComponents. |
| `applications` | array[ApplicationDefinition] | No | Traffic generator and workload applications |
| `links` | array[ComponentDeclaration] | No | Layer components defining spine plane linking configuration (type='layer') |

```json
{
  "name": "string",
  "description": "string",
  "category": "getting-started",
  "components": [
    {
      "type": "...",
      "trafficType": "...",
      "model": "ScalaSwitch",
      "name": "spine-switch-1",
      "version": "3.2.1",
      "typedParameters": "...",
      "allowedComponents": [
        "..."
      ],
      "components": [
        "..."
      ]
    }
  ],
  "containers": [
    {
      "type": "...",
      "name": "roce-rack",
      "model": "ScalaRack",
      "version": "4.0.0",
      "configuration": null,
      "allowedComponents": [
        "..."
      ],
      "components": [
        "..."
      ]
    }
  ],
  "topology": {
    "name": null,
    "type": "clos",
    "topologyTiers": 1,
    "allowedComponents": [
      "string"
    ],
    "components": [
      "..."
    ],
    "appDistribution": {
      "description": "...",
      "randomApplicationDistribution": "..."
    }
  },
  "globalParameters": {},
  "applications": [
    {
      "type": "...",
      "trafficType": "...",
      "name": "ScalaChakraGenerator",
      "model": "ChakraWorkload",
      "version": "4.0.1",
      "typedParameters": "..."
    }
  ],
  "links": [
    {
      "type": "...",
      "trafficType": "...",
      "model": "ScalaSwitch",
      "name": "spine-switch-1",
      "version": "3.2.1",
      "typedParameters": "...",
      "allowedComponents": [
        "..."
      ],
      "components": [
        "..."
      ]
    }
  ]
}
```

### Responses

**201** - Template created

```json
{
  "id": "string",
  "name": "string",
  "description": "string",
  "category": "getting-started",
  "configuration": {
    "id": "config_10045",
    "workspaceId": "string",
    "name": "string",
    "status": "draft",
    "createdAt": "2024-01-15T10:30:00Z",
    "modifiedAt": "2024-01-15T10:30:00Z",
    "version": 1,
    "componentCount": 1,
    "components": [
      "..."
    ],
    "containers": [
      "..."
    ],
    "applications": [
      "..."
    ],
    "links": [
      "..."
    ],
    "topology": {
      "name": "...",
      "type": "...",
      "topologyTiers": "...",
      "allowedComponents": "...",
      "components": "...",
      "appDistribution": "..."
    },
    "globalParameters": {},
    "activeTraceset": {
      "name": "...",
      "id": "..."
    },
    "activeMappingFile": {
      "id": "...",
      "name": "...",
      "deleted": "..."
    }
  }
}
```

---

## Get template details

<span class="api-method api-method-get">GET</span> `/api/v1/templates/{template_id}`

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `template_id` | path | string | Yes | - |

### Responses

**200** - Template details

```json
{
  "id": "string",
  "name": "string",
  "description": "string",
  "category": "getting-started",
  "configuration": {
    "id": "config_10045",
    "workspaceId": "string",
    "name": "string",
    "status": "draft",
    "createdAt": "2024-01-15T10:30:00Z",
    "modifiedAt": "2024-01-15T10:30:00Z",
    "version": 1,
    "componentCount": 1,
    "components": [
      "..."
    ],
    "containers": [
      "..."
    ],
    "applications": [
      "..."
    ],
    "links": [
      "..."
    ],
    "topology": {
      "name": "...",
      "type": "...",
      "topologyTiers": "...",
      "allowedComponents": "...",
      "components": "...",
      "appDistribution": "..."
    },
    "globalParameters": {},
    "activeTraceset": {
      "name": "...",
      "id": "..."
    },
    "activeMappingFile": {
      "id": "...",
      "name": "...",
      "deleted": "..."
    }
  }
}
```

---

## Clone template to workspace

<span class="api-method api-method-post">POST</span> `/api/v1/templates/{template_id}/clone`

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `template_id` | path | string | Yes | - |

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `workspaceId` | string | No | Target workspace identifier |
| `name` | string | Yes | Name for the new configuration |

```json
{
  "workspaceId": "string",
  "name": "string"
}
```

### Responses

**201** - Configuration created

```json
{
  "id": "config_10045",
  "workspaceId": "string",
  "name": "string",
  "status": "draft",
  "createdAt": "2024-01-15T10:30:00Z",
  "modifiedAt": "2024-01-15T10:30:00Z",
  "version": 1,
  "componentCount": 1,
  "components": [
    {
      "type": "...",
      "trafficType": "...",
      "model": "ScalaSwitch",
      "name": "spine-switch-1",
      "version": "3.2.1",
      "typedParameters": "...",
      "allowedComponents": [
        "..."
      ],
      "components": [
        "..."
      ]
    }
  ],
  "containers": [
    {
      "type": "...",
      "name": "roce-rack",
      "model": "ScalaRack",
      "version": "4.0.0",
      "configuration": null,
      "allowedComponents": [
        "..."
      ],
      "components": [
        "..."
      ]
    }
  ],
  "applications": [
    {
      "type": "...",
      "trafficType": "...",
      "name": "ScalaChakraGenerator",
      "model": "ChakraWorkload",
      "version": "4.0.1",
      "typedParameters": "..."
    }
  ],
  "links": [
    {
      "type": "...",
      "trafficType": "...",
      "model": "ScalaSwitch",
      "name": "spine-switch-1",
      "version": "3.2.1",
      "typedParameters": "...",
      "allowedComponents": [
        "..."
      ],
      "components": [
        "..."
      ]
    }
  ],
  "topology": {
    "name": null,
    "type": "clos",
    "topologyTiers": 1,
    "allowedComponents": [
      "string"
    ],
    "components": [
      "..."
    ],
    "appDistribution": {
      "description": "...",
      "randomApplicationDistribution": "..."
    }
  },
  "globalParameters": {},
  "activeTraceset": {
    "name": "string",
    "id": "string"
  },
  "activeMappingFile": {
    "id": "string",
    "name": "string",
    "deleted": true
  }
}
```

---
