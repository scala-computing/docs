---
title: "Models"
description: "Model catalog and discovery"
---

Model catalog and discovery

{/* AUTO-GENERATED CONTENT BELOW - DO NOT EDIT MANUALLY */}
{/* Generated from OpenAPI spec by generate-api-reference.py */}

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/models` | List available models |
| GET | `/api/v1/models/{model_id}` | Get model details |
| GET | `/api/v1/models/{model_id}/versions` | List model versions |

---

## List available models

<span class="api-method api-method-get">GET</span> `/api/v1/models`

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `limit` | query | integer | No | - |
| `next` | query | string | No | Opaque cursor token to fetch results after a specific item |
| `type` | query | string | No | Filter by model type (comma-separated for multiple) |
| `vendor` | query | string | No | Filter by vendor identifier |
| `search` | query | string | No | Full-text search across name and description fields (case-insensitive) |

### Responses

**200** - List of models

```json
{
  "models": [
    {
      "id": "string",
      "model": "string",
      "name": "string",
      "type": "...",
      "vendor": null,
      "latestVersion": "string",
      "compatibility": null,
      "status": "..."
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

## Get model details

<span class="api-method api-method-get">GET</span> `/api/v1/models/{model_id}`

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `model_id` | path | string | Yes | - |
| `version` | query | string | No | Specific version to retrieve (defaults to latest stable) |

### Responses

**200** - Model details in ComponentDeclaration format - copy-paste ready for configuration components array

```json
{
  "type": "switch",
  "trafficType": "chakra",
  "model": "ScalaSwitch",
  "name": "spine-switch-1",
  "version": "3.2.1",
  "typedParameters": {},
  "allowedComponents": [
    "string"
  ],
  "components": [
    {
      "name": "spine-1",
      "count": 4,
      "model": "ScalaSwitch",
      "version": "3.2.1",
      "type": "switch",
      "group": null,
      "parentApplicationName": null
    }
  ]
}
```

**404** - Model not found

---

## List model versions

<span class="api-method api-method-get">GET</span> `/api/v1/models/{model_id}/versions`

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `model_id` | path | string | Yes | - |

### Responses

**200** - List of versions

```json
{
  "id": "string",
  "model": "string",
  "versions": [
    {
      "version": "string",
      "status": "...",
      "compatibility": null
    }
  ]
}
```

---
