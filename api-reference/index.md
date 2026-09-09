---
title: "API Reference"
description: "This section provides detailed documentation for all Network Simulation API endpoints."
---

This section provides detailed documentation for all Network Simulation API endpoints.

{/* AUTO-GENERATED CONTENT - DO NOT EDIT MANUALLY */}
{/* Generated from OpenAPI spec by generate-api-reference.py */}

## Endpoints Overview

| Tag | Description | Endpoints |
|-----|-------------|-----------|
| [Billing](./billing.md) | Billing balance, costs, items, events, and rates (feature-gated: enable_billing_endpoints) | 7 |
| [Charts](./charts.md) | Chart definitions served to the simulation UI | 1 |
| [Components](./components.md) | Component schema discovery | 2 |
| [Configurations](./configurations.md) | Workload configuration management | 34 |
| [Health](./health.md) | Health and monitoring endpoints | 2 |
| [Libraries](./libraries.md) | Content-addressed trace libraries (feature-gated: enable_trace_routes) | 2 |
| [Models](./models.md) | Model catalog and discovery | 3 |
| [Simple](./simple.md) | Simple collective-operation simulation | 3 |
| [Simulations](./simulations.md) | Simulation execution and results | 9 |
| [Templates](./templates.md) | Curated example configurations | 4 |
| [Topology](./topology.md) | Real-time topology validation | 1 |
| [Tracesets](./tracesets.md) | Trace file management (feature-gated: enable_trace_routes) | 9 |
| [Workspaces](./workspaces.md) | Workspace management | 6 |

## Common Patterns

### Pagination

List endpoints return paginated results. Use the `limit` and `next` parameters:

```bash
# First page
GET /api/v1/workspaces?limit=50

# Next page
GET /api/v1/workspaces?limit=50&next=<cursor>
```

### Error Responses

All errors follow a consistent format:

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message"
  }
}
```

### Authentication

All endpoints require bearer token authentication:

```bash
curl -H "Authorization: Bearer YOUR_TOKEN" ...
```

---

> For the interactive API explorer, see [Swagger UI](../swagger-ui.md).
