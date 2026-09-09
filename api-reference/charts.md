---
title: "Charts"
description: "Chart definitions served to the simulation UI"
---

Chart definitions served to the simulation UI

{/* AUTO-GENERATED CONTENT BELOW - DO NOT EDIT MANUALLY */}
{/* Generated from OpenAPI spec by generate-api-reference.py */}

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/chart-definitions` | List chart definitions |

---

## List chart definitions

<span class="api-method api-method-get">GET</span> `/api/v1/chart-definitions`

Returns every chart definition stored on this platform as a bare body, ordered by id, under one schema version. The set is bounded server-side and is therefore unpaginated: a client needs all of it atomically under a single schemaVersion.

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

**401** - Unauthorized

**404** - Endpoint not served: this platform's release predates it. No handler is mounted, so this status carries no body.

**500** - The definitions could not be read, or the set breached a served bound and was refused whole

**504** - The read exceeded the server-side statement budget and was cancelled

---
