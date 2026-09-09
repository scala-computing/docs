---
title: "API Changelog"
description: "All notable API changes will be documented in this file."
---

All notable API changes will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [Unreleased] - 2026-03-06

### Added

- **Simple simulation endpoint**: `POST /api/v1/simple/simulations` — submit a collective-operation workload (AllReduce, Broadcast, etc.) with a single request. The platform handles topology derivation, traceset synthesis, and simulation launch automatically.

- **Topology constraints**: `topologyConstraints` object on `SimpleSimulationRequest` with `switchRadix` (enum: `64x800G`, `128x400G`, `256x200G`) and `subscriptionRatio` (enum: `1:1`, `2:1`, `3:1`, `4:1`). Enables arbitrary rank counts by deriving a valid 2-tier CLOS topology from hardware constraints.

### Fixed

- **Simulation results endpoints**: Removed non-functional workspace ownership check that blocked all users from accessing results. `DefaultWorkspace.owner = 'system'` caused `verify_workspace_access` to reject every real user. Endpoints now correctly return results for any authenticated user. Affected: `GET /simulations/{id}/results`, `GET /simulations/{id}/results/download-url`, `GET /simulations/{id}/results/data`.

### Changed

- **Presigned URL expiry reduced**: Simulation result download URLs (`GET /simulations/{id}/results` and `GET /simulations/{id}/results/download-url`) now expire after **15 minutes (900 s)** instead of the previous 60 minutes (3600 s). This is a defense-in-depth hardening to limit exposure if a URL leaks. Clients that fetch a presigned URL and consume it after a delay greater than 15 minutes will receive an HTTP 403 from S3. The `expiresAt` field in the `download-url` response reflects the actual expiry. No known consumers cache presigned URLs beyond this window.

- **BREAKING**: `SimpleWorkload.ranks` changed from integer enum (`[64, 128, 256, 512, 1024, 4096]`) to `integer` with `minimum: 1`. Without `topologyConstraints`, the server still restricts to the 6 reference values. With `topologyConstraints`, any positive value satisfying the constraint solver is accepted.

- **BREAKING**: `TracesetFile` renamed to `FileResponse` in the Python client. Adds required field `metadata_status`. Update imports: `from python_client.models import FileResponse` (was `TracesetFile`).

- **BREAKING**: `ContainerDefinition.type_` and `AllowedTypesResponse.container_type` now use `ContainerType` instead of `ComponentType` in the Python client. Both are `str` enums with identical string values, so equality comparisons by value still work. Update `isinstance` checks if used.

- **BREAKING**: `DlrmMetadata`, `TraceMetadata`, and `TraceMetadataWorkloadType` removed from the Python client. These types are no longer part of the API surface. Remove any imports that reference them.

- **Traceset listing scope widened**: `GET /api/v1/tracesets?workspaceId=X` now returns all non-deleted tracesets visible to the workspace, including promoted tracesets with scope `tenant` or `public`. Previously, only `workspace`-scoped tracesets were returned. This ensures promoted tracesets remain visible after promotion.

- **TrafficType output consolidation**: API responses now return `"chakra"` where `"coordinated"` was previously emitted for the `trafficType` field. Input is backward-compatible: both `"chakra"` and `"coordinated"` are accepted via serde alias. This reflects the A5 schema migration consolidating Coordinated into Chakra.

- **BREAKING**: `PATCH /api/v1/configurations/{config_id}/traceset` now requires the traceset's metadata extraction to be complete before attaching. The endpoint previously accepted any valid traceset regardless of library state; it now enforces that `rank_count` is resolved and positive (required to populate `NumOfChakraFiles` in the Chakra workload model).

  Two new error responses are returned when the traceset metadata is incomplete or invalid:

  | Condition | Status | Error code |
  |---|---|---|
  | `rank_count` is `NULL` — metadata extraction still in progress | `409 Conflict` | `CONFLICT` |
  | `rank_count` is present but ≤ 0 — data invariant violation | `422 Unprocessable Entity` | `UNPROCESSABLE_ENTITY` |

  **Migration**: If you call this endpoint immediately after uploading a traceset, poll until the traceset's `metadataStatus` is `complete` before attaching. A `409` response indicates the traceset is not yet ready; retry after a short backoff. A `422` response indicates a permanent problem with the traceset metadata and will not resolve on retry.

## [Previous Unreleased] - 2024-12-23

### Changed

- **BREAKING**: `Pagination.limit` renamed to `Pagination.count` in API responses
  - Affected: All paginated list endpoints
  - Before: `{"limit": 10, "hasMore": true, "cursor": "..."}`
  - After: `{"count": 10, "hasMore": true, "cursor": "..."}`
  - Semantic change: `count` represents the actual number of items returned (may be less than requested limit)
  - Note: Query parameter `?limit=N` remains unchanged; only the response field name changed
  - Migration: Update client code to read `count` instead of `limit` from pagination responses

- **BREAKING**: `ParameterDefinition.default` type changed from JSON value to string
  - Affected: `GET /api/v1/models/{id}`, `GET /api/v1/components/{id}`
  - Before: `{"default": 100}` or `{"default": true}`
  - After: `{"default": "100"}` or `{"default": "true"}`
  - Migration: Parse the string value according to the parameter's `type` field

- **BREAKING**: `ComponentDeclaration.model` format changed to use standardized model reference
  - Affected: `GET /api/v1/templates/{id}`, `GET /api/v1/configurations/{id}`, `GET /api/v1/configurations/{id}/versions/{v}`, `POST /api/v1/templates/{id}/clone`
  - Before: `{"model": "ScalaSwitch:3.2.1", "name": "spine-1"}` (model in internal format, name user-defined)
  - After: `{"model": "scala-switch-v3.2.1", "name": "spine-1"}` (model standardized to API format, name remains user-manageable)
  - Format: Model field is standardized to API format (`scala-switch-v{version}`)
  - Name field: Remains user-manageable and unchanged from input
  - Benefit: Model field can be parsed to extract model reference for use with `GET /api/v1/models/{model_id}`
  - Migration: Update any code that parses or displays component model fields

- **BREAKING**: Only referenced components are returned in configuration responses
  - Affected: `GET /api/v1/templates/{id}`, `GET /api/v1/configurations/{id}`, `POST /api/v1/templates/{id}/clone`
  - Before: All components declared in the configuration were returned, including unused ones
  - After: Only components referenced in `topology.components`, `topology.containers`, or `containers[].components` are returned
  - Rationale: Reduces response size and focuses on components actually used in the topology
  - Migration: If your code relies on seeing all declared components, use the stored configuration data directly

- **Template cloning preserves original data format**
  - Affected: `POST /api/v1/templates/{id}/clone`
  - Change: Cloned configurations now preserve the original template's data format (legacy or new format)
  - Before: Template data was transformed to new format during cloning
  - After: Template data format is preserved, only metadata fields (version, workspaceId, name, status, parent_config_id) are updated
  - Benefit: Maintains compatibility with existing templates and allows format-specific handling

### Removed

- **BREAKING**: Removed fields from `ParameterDefinition`:
  - `componentType` - was used for nested component type references
  - `model` - was used for model references within parameters
  - `parameters` - was used for nested parameter definitions
  - Migration: These fields were not widely used. If you relied on nested parameter structures, contact the API team for migration guidance.

### Added

- **Model Metadata Enrichment**: Templates API now fetches model metadata from `model_version`
  - Affected: `GET /api/v1/templates/{id}`
  - Components are enriched with parameter schemas and type information from the database
  - When a component references a model (e.g., `ScalaSwitch:4.0.0`), the API looks up the model in `model_version` and includes the full parameter schema
  - Benefit: Clients receive complete model metadata without needing separate API calls to `/api/v1/models/{model_id}`

- **Parameter Extraction**: Extended parameter extraction to include coordinator-level parameters
  - Application models now expose parameters from `Coordinators`, `ModelProfiles`, and `TrafficPattern` sections
  - Parameters include group information (e.g., `Delay.CoordinatorAttributes`)

---

## Changelog Guidelines

See [API Changelog Template](../../templates/api-changelog.md) for formatting guidelines.

