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
      "id": "model_37w6uUurBkspg7voji4julJIPSP",
      "model": "ScalaSwitch",
      "name": "string",
      "category": "model",
      "type": "string",
      "description": "string",
      "latestVersion": "string",
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
  "id": "string",
  "name": "string",
  "category": "model",
  "type": "string",
  "trafficType": "chakra",
  "description": "string",
  "version": "string",
  "latestVersion": "string",
  "createdAt": "2024-01-15T10:30:00Z",
  "typedParameters": {},
  "containerSchema": {
    "type": "rack",
    "name": {
      "type": "...",
      "description": "...",
      "enum": "...",
      "minimum": "..."
    },
    "allowed": [
      "string"
    ],
    "components": {
      "type": "...",
      "description": "...",
      "items": "..."
    }
  },
  "notes": "string"
}
```

---
