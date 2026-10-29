---
title: "Charts"
description: "Chart definitions served to the simulation UI (feature-gated: enable_chart_definitions)"
---

Chart definitions served to the simulation UI (feature-gated: enable_chart_definitions)

{/* AUTO-GENERATED CONTENT BELOW - DO NOT EDIT MANUALLY */}
{/* Generated from OpenAPI spec by generate-api-reference.py */}

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/chart-definitions` | List chart definitions |

---

## List chart definitions

<span class="api-method api-method-get">GET</span> `/api/v1/chart-definitions`

Returns every chart definition stored on this platform as a bare body, ordered by id, under one schema version. The set is bounded server-side and is therefore unpaginated: a client needs all of it atomically under a single schemaVersion. A reader names the newest chart-definition schema version it parses in the schemaVersion query parameter; each chart is served at the highest stored body version that does not exceed it, and a request without the parameter is served chart-definition/v1.

### Parameters

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `schemaVersion` | query | string | No | The newest chart-definition schema version the reader parses, as chart-definition/v<N> with N a positive integer and no leading zero. Absent, the response is chart-definition/v1. Versions compare as integers, and a version newer than any body stored for a chart is served that chart's newest stored body. A value outside that grammar, or the parameter given more than once, is refused with 400. |

### Responses

**200** - The platform's chart definitions

```json
{
  "schemaVersion": "string",
  "definitions": [
    {}
  ]
}
```

**400** - The schemaVersion query parameter is malformed or repeated

**401** - Unauthorized

**404** - Endpoint not served: this platform's release predates it, or it runs with enable_chart_definitions off. No handler is mounted on either path, so this status carries no body.

**500** - The definitions could not be read, or the set breached a served bound and was refused whole

**504** - The read exceeded the server-side statement budget and was cancelled

---
