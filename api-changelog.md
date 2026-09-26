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

- **Chakra mapping files**: a mapping-file resource that pins each Chakra logical rank to a server, and optionally declares scale-up groups and their fabric parameters. Upload a file once, attach it to a simulation configuration, and every simulation created from that configuration is launched with it. Without one, placement is unchanged: rank `i` runs on the `i`-th Chakra-capable server. Running a simulation to completion with a mapping file is not yet available: the rank count of an uploaded traceset is not yet determined, so a mapping file cannot be attached behind one (the mapping-file attach, like the traceset attach, answers `409` `CONFLICT` for it), and a simulation on one of the platform's own Chakra tracesets fails before the simulator starts until an update to the platform's simulation hosts is in service (see Current limitations in [Chakra Mapping Files](./chakra-mapping-files.md)). Eight new operations, all tagged `mapping-files`:

  | Method | Path | Purpose |
  |---|---|---|
  | POST | `/api/v1/mapping-files/upload-url` | Presigned multipart upload URLs for a new mapping file; `name` is optional (default `chakra-mapping.txt`) and `sizeBytes` is at most 64,000,000 |
  | POST | `/api/v1/mapping-files/upload-url/complete` | Validate and store the file: `201` with the new file, or `200` with `duplicateOfExisting: true` when the workspace already holds the same bytes; `422` `MAPPING_FILE_INVALID` names the first offending line |
  | GET | `/api/v1/mapping-files` | A workspace's live mapping files, newest first; `workspaceId` is required; cursor pagination through `next` and `nextCursor` |
  | GET | `/api/v1/mapping-files/{mapping_file_id}` | One mapping file's details; `410` `MAPPING_FILE_DELETED` once deleted |
  | GET | `/api/v1/mapping-files/{mapping_file_id}/download` | Presigned URL serving the exact uploaded bytes; `410` `MAPPING_FILE_DELETED` once deleted |
  | DELETE | `/api/v1/mapping-files/{mapping_file_id}` | Delete a mapping file, whether or not a configuration refers to it |
  | PATCH | `/api/v1/configurations/{config_id}/mapping-file` | Attach a mapping file to a configuration (`If-Match` supported) |
  | DELETE | `/api/v1/configurations/{config_id}/mapping-file` | Detach it and restore in-order placement (`If-Match` supported) |

  The stored file is the simulator's own text format, byte for byte, with a 64 MB per-file limit and no per-workspace quota. `MappingFileSummary` carries `id`, `name`, `workspaceId`, `sizeBytes`, `rankCount`, `scaleUpGroupCount`, `hasScaleUp` and `createdAt`; `MappingFileDetails` adds `description`, `hash`, `etag` and `duplicateOfExisting`. New `error.code` values: `MAPPING_FILE_INVALID`, `MAPPING_FILE_HASH_COLLISION`, `MAPPING_FILE_DELETED`, `MAPPING_REQUIRES_TRACESET`, `MAPPING_REQUIRES_CHAKRA_APP`, `MAPPING_RANK_COUNT_MISMATCH`, `MAPPING_PARAMS_LOCKED`, `TRACESET_CONFLICTS_WITH_MAPPING` and `MAPPING_FILE_DELETED_REFERENCE`, plus `MAPPING_FILE_MISSING` and `MAPPING_PARAMS_INCONSISTENT` on `500` platform faults. See [Chakra Mapping Files](./chakra-mapping-files.md) for the file format and the full flow.

- **`activeMappingFile` on `ConfigurationDetails`**: names the attached mapping file as `{ "id", "name", "deleted" }`, or is `null`. `deleted` is resolved when the configuration is read, so a configuration that still refers to a deleted file reports it as deleted. Only attach and detach set it: a configuration update whose body carries `activeMappingFile` returns `400`. Attach requires at least one Chakra application (`422` `MAPPING_REQUIRES_CHAKRA_APP`), an attached traceset (`409` `MAPPING_REQUIRES_TRACESET`) whose rank count has been resolved (`409` `CONFLICT` until then, retry-safe), a mapping file in the configuration's workspace that has not been deleted (`404` otherwise), and equal rank counts (`422` `MAPPING_RANK_COUNT_MISMATCH`, naming both counts). While a file is attached, a configuration update, an application-parameter update or an application create that would change `UseMapping` or `MappingFileName` returns `422` `MAPPING_PARAMS_LOCKED`, and a traceset with a different rank count returns `409` `TRACESET_CONFLICTS_WITH_MAPPING` on traceset attach and on configuration update.

- **`appliedMappingFile` on `SimulationDetails`**: names the mapping file the simulation was created with, as `{ "id", "name", "deleted" }`, and is absent for a configuration with no mapping file. It keeps naming the file after a delete, with `deleted` reading `true`; deleting a mapping file never changes a simulation already created. `POST /api/v1/simulations` from a configuration whose attached mapping file has been deleted returns `409` `MAPPING_FILE_DELETED_REFERENCE`, naming the file. A mapped run never falls back to in-order placement: a file that cannot be delivered to the simulator fails the simulation. When the platform rejects the file while staging it, the `failureReason` reads `Mapping file <name> (<id>) could not be delivered`, with a long `<name>` possibly shortened to end in `…` and the id always in full; a failure not attributed to the mapping file reports a general failure reason.

### Fixed

- **Simulation results endpoints**: Removed non-functional workspace ownership check that blocked all users from accessing results. `DefaultWorkspace.owner = 'system'` caused `verify_workspace_access` to reject every real user. Endpoints now correctly return results for any authenticated user. Affected: `GET /simulations/{id}/results`, `GET /simulations/{id}/results/download-url`, `GET /simulations/{id}/results/data`.

### Changed

- **BREAKING**: `GET /api/v1/simulations` now accepts only a single `field:direction` pair in `sort` (for example `created_at:desc`). A compound sort such as `status:asc,created_at:desc` returns **400**; it previously returned 200 but could not be paginated correctly past the first page. A `next` cursor replayed with a different `sort` than the one that produced it also returns **400** instead of a wrong page. To migrate: send one sort field, and when you change `sort`, restart pagination without `next`.

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
