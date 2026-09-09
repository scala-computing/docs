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
  "pagination": {
    "count": 1,
    "hasMore": true,
    "nextCursor": "string"
  },
  "templates": [
    {
      "category": "...",
      "description": "string",
      "id": "string",
      "name": "string"
    }
  ]
}
```

---

## Create template

<span class="api-method api-method-post">POST</span> `/api/v1/templates`

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `applications` | array[ApplicationDefinition] | No | Traffic generator and workload applications |
| `category` | TemplateCategory | Yes | Category of template |
| `components` | array[ComponentDeclaration] | No | Component declarations with typedParameters |
| `containers` | array[ContainerDefinition] | No | Container definitions |
| `description` | string | Yes | Human-readable description |
| `globalParameters` | TypedParameters | No | Hierarchical parameters structure. Leaf nodes are ParameterValue objects, branch nodes are nested TypedParameters representing ModelComponents. |
| `links` | array[ComponentDeclaration] | No | Layer components defining spine plane linking configuration (type='layer') |
| `name` | string | Yes | Template name |
| `topology` | TopologyDefinition | No | Active network topology definition |

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
  ],
  "category": "getting-started",
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
  "description": "string",
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
  }
}
```

### Responses

**201** - Template created

```json
{
  "category": "getting-started",
  "configuration": {
    "activeTraceset": {},
    "applications": [
      "..."
    ],
    "componentCount": 1,
    "components": [
      "..."
    ],
    "containers": [
      "..."
    ],
    "createdAt": "2024-01-15T10:30:00Z",
    "globalParameters": {},
    "id": "config_10045",
    "links": [
      "..."
    ],
    "modifiedAt": "2024-01-15T10:30:00Z",
    "name": "string",
    "status": "draft",
    "topology": {
      "allowedComponents": "...",
      "appDistribution": "...",
      "components": "...",
      "name": "...",
      "topologyTiers": "...",
      "type": "..."
    },
    "version": 1,
    "workspaceId": "string"
  },
  "description": "string",
  "id": "string",
  "name": "string"
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
  "category": "getting-started",
  "configuration": {
    "activeTraceset": {},
    "applications": [
      "..."
    ],
    "componentCount": 1,
    "components": [
      "..."
    ],
    "containers": [
      "..."
    ],
    "createdAt": "2024-01-15T10:30:00Z",
    "globalParameters": {},
    "id": "config_10045",
    "links": [
      "..."
    ],
    "modifiedAt": "2024-01-15T10:30:00Z",
    "name": "string",
    "status": "draft",
    "topology": {
      "allowedComponents": "...",
      "appDistribution": "...",
      "components": "...",
      "name": "...",
      "topologyTiers": "...",
      "type": "..."
    },
    "version": 1,
    "workspaceId": "string"
  },
  "description": "string",
  "id": "string",
  "name": "string"
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
| `name` | string | Yes | Name for the new configuration |
| `workspaceId` | string | No | Target workspace identifier |

```json
{
  "name": "string",
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
