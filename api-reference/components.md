---
title: "Components"
description: "Component schema discovery"
---

Component schema discovery

{/* AUTO-GENERATED CONTENT BELOW - DO NOT EDIT MANUALLY */}
{/* Generated from OpenAPI spec by generate-api-reference.py */}

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/components` | List component types |
| GET | `/api/v1/components/{component_id}` | Get component schema |

---

## List component types

<span class="api-method api-method-get">GET</span> `/api/v1/components`

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `limit` | query | integer | No | - |
| `next` | query | string | No | Opaque cursor token to fetch results after a specific item |
| `category` | query | string (enum) | No | Filter by category |
| `type` | query | string | No | Filter by type within category |

### Responses

**200** - List of component types

```json
{
  "components": [
    {
      "category": "model",
      "description": "string",
      "id": "model_37w6uUurBkspg7voji4julJIPSP",
      "latestVersion": "string",
      "model": "ScalaSwitch",
      "name": "string",
      "type": "string",
      "versions": [
        "..."
      ]
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

## Get component schema

<span class="api-method api-method-get">GET</span> `/api/v1/components/{component_id}`

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `component_id` | path | string | Yes | - |
| `version` | query | string | No | Specific version (defaults to latest) |
| `notes` | query | boolean | No | Include product notes in response (defaults to false) |

### Responses

**200** - Component schema

```json
{
  "category": "model",
  "containerSchema": {
    "allowed": [
      "string"
    ],
    "components": {
      "description": "...",
      "items": "...",
      "type": "..."
    },
    "name": {
      "description": "...",
      "enum": "...",
      "minimum": "...",
      "type": "..."
    },
    "type": "rack"
  },
  "createdAt": "2024-01-15T10:30:00Z",
  "description": "string",
  "id": "string",
  "latestVersion": "string",
  "name": "string",
  "notes": "string",
  "trafficType": "chakra",
  "type": "string",
  "typedParameters": {},
  "version": "string"
}
```

---
