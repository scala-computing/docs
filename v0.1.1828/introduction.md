---
title: "Network Simulation API"
description: "Welcome to the Scala Network Simulation API documentation. This API provides programmatic access to Scala's network simulation platform, enabling you to..."
---

Welcome to the Scala Network Simulation API documentation. This API provides programmatic access to Scala's network simulation platform, enabling you to create, configure, and run network simulations at scale.

## What You Can Do

With the Network Simulation API, you can:

- **Create Workspaces** - Organize your simulations into logical workspaces
- **Configure Simulations** - Define network topologies, components, and parameters
- **Run Simulations** - Execute simulations and monitor their progress
- **Retrieve Results** - Access simulation outputs, logs, and metrics
- **Manage Resources** - Upload traces, manage templates, and explore available models

## API Overview

The API follows RESTful conventions and uses JSON for request and response bodies. All endpoints require bearer token authentication.

### Base URL

```
https://api.scalacomputing.com/api/v1
```

### Content Type

All requests and responses use `application/json` content type.

### Versioning

The API is versioned via the URL path (`/api/v1/`). Breaking changes will be introduced in new major versions.

## Quick Example

Here's a simple example of listing your workspaces:

```bash
curl -X GET "https://api.scalacomputing.com/api/v1/workspaces" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "Content-Type: application/json"
```

Response:

```json
{
  "workspaces": [
    {
      "id": "ws_abc123",
      "name": "My First Workspace",
      "description": "Testing network configurations",
      "owner": "user@example.com",
      "created": "2024-01-15T10:30:00Z",
      "modified": "2024-01-15T10:30:00Z"
    }
  ],
  "pagination": {
    "limit": 100,
    "hasMore": false
  }
}
```

## Next Steps

- Review the [Authentication](./authentication.md) documentation to obtain an API token
- Follow the [Quickstart guide](./examples/quickstart.md) to run your first simulation
- Explore the [API Reference](./api-reference/index.md) for detailed endpoint documentation
