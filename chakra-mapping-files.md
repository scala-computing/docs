---
title: "Chakra Mapping Files"
description: "A Chakra mapping file tells the simulator which server runs each Chakra rank. Without one, ranks are placed in order: rank 0 on the first Chakra-capable..."
---

A Chakra mapping file tells the simulator which server runs each Chakra rank. Without one, ranks are placed in order: rank 0 on the first Chakra-capable server, rank 1 on the second, and so on. With one, you choose the placement, and you can declare scale-up groups so that traffic inside a group uses the scale-up fabric instead of the scale-out network.

You upload a mapping file once and attach it to a simulation configuration, and every simulation created from that configuration is launched with it. The file is stored byte for byte: a download returns exactly the bytes you uploaded.

## Current limitations

Uploading, listing, downloading, deleting, attaching, and detaching mapping files work today, and a simulation created from a configuration with a mapping file attached records it in `appliedMappingFile`. A simulation cannot yet run to completion with a mapping file:

- **Tracesets you upload (coming soon).** Attaching a mapping file requires the configuration's traceset to have a resolved rank count, and the platform does not yet determine the rank count of a traceset you upload. A mapping file therefore cannot be attached behind an uploaded traceset: the mapping-file attach answers `409` `CONFLICT`. The traceset attach refuses an uploaded traceset with the same `409` while its rank count is unresolved, although a configuration create or update can still name it as `activeTraceset`.
- **The platform's own Chakra tracesets (coming soon).** These have a rank count, so a mapping file attaches to a configuration that uses one. A simulation created from that configuration currently fails before the simulator starts, with the `failureReason` `Mapping file <name> (<id>) could not be delivered`, until an update to the platform's simulation hosts is in service.

The rest of this page describes the file format and the full flow as the API defines it.

## The file format

A mapping file is plain text with one required section and one optional section.

- Everything from a `#` to the end of its line is a comment. Blank lines are ignored.
- Fields are separated by spaces, commas, or any run of both. Tabs are **not** separators, and the file must use Unix (LF) line endings, not Windows (CRLF).
- A line may carry at most 127 bytes before its first comment, and at most 4,096 bytes in total.
- A comment block at the top of the file must use `#`. A line starting with `@` is reserved for Section II.

### Section I: rank placement

Section I lists one rank per line, with exactly two or exactly four fields:

```text
ChakraLogicalId, serverId [, ScaleUpGroupId, ScaleUpSubgroupId]
```

| Field | Required | Meaning |
|-------|----------|---------|
| `ChakraLogicalId` | Yes | The rank's logical id: the index encoded in the rank's trace file name. |
| `serverId` | Yes | The global id of the server that runs this rank. It must be a Chakra-capable server in the configuration's topology. |
| `ScaleUpGroupId` | No | The scale-up group the rank belongs to. Given together with `ScaleUpSubgroupId` or not at all. |
| `ScaleUpSubgroupId` | No | The scale-up subgroup the rank belongs to. |

All fields are non-negative base-10 integers that fit in 32 bits, and `4294967295` is not accepted as a `serverId`. The ids must cover the contiguous range `0` through `rankCount - 1`, each exactly once, and no two ranks may name the same `serverId`. A rank with no scale-up fields belongs to no group, and all of its traffic uses the scale-out network.

### Section II: scale-up group parameters (optional)

Section II begins at the first line that starts with `@`, after at least one Section I entry. Every line starting with `@` is a comment, so the marker line can carry a heading, and inside Section II both `#` and `@` start a comment anywhere in a line. Each remaining line declares one group, with exactly four fields:

```text
ScaleUpGroupId, ScaleUpBandwidthGbps, ScaleUpPerHopSwitchLatencyNs, ScaleUpNpuCableLatencyNs
```

| Field | Units | Accepted values |
|-------|-------|-----------------|
| `ScaleUpGroupId` | — | A group id, declared once. |
| `ScaleUpBandwidthGbps` | Gbit/s | A positive decimal number of at least 1e-9 and below 1.8446744073709552e10. |
| `ScaleUpPerHopSwitchLatencyNs` | ns | 0, or a decimal number of at least 1e-3 and below 1.8446744073709552e16. |
| `ScaleUpNpuCableLatencyNs` | ns | 0, or a decimal number of at least 1e-3 and below 1.8446744073709552e16. |

Every group a Section I entry names must have a Section II line. Hexadecimal and non-finite values are not accepted. If the file has no scale-up groups, leave Section II out.

### Examples

An 8-rank file with no scale-up groups, placing the ranks in reverse order. The comment on each row is free-form; here it records where the server sits in the topology:

```text
# ChakraID,ServerID # DataCenterId, PodId, RackId
0,7 #0, 0, 0
1,6 #0, 0, 0
2,5 #0, 0, 0
3,4 #0, 0, 0
4,3 #0, 0, 1
5,2 #0, 0, 1
6,1 #0, 0, 1
7,0 #0, 0, 1
```

An 8-rank file with two scale-up groups. Ranks 0 to 5 are group members; ranks 6 and 7 are not:

```text
# Section I: ChakraLogicalId, serverId, ScaleUpGroupId (optional), ScaleUpSubgroupId (optional)
0, 0, 0, 0
1, 1, 0, 0
2, 2, 0, 1
3, 3, 0, 1
4, 4, 1, 0
5, 5, 1, 0
6, 6
7, 7
@@@@@@@@
@ Section II: ScaleUpGroupId, ScaleUpBandwidthGbps, ScaleUpPerHopSwitchLatencyNs, ScaleUpNpuCableLatencyNs
0, 800.0, 100.0, 100.0
1, 800.0, 50.0, 50.0
```

### When each rule is checked

Every rule above is checked when the upload completes. A file that breaks one is refused with `422` `MAPPING_FILE_INVALID`, the message names the first offending line (`line <n>: <reason>`) or the first missing rank, and nothing is stored.

Two things depend on a configuration, so they are checked later:

- **Rank count.** Attach requires the file's `rankCount` to equal the rank count of the configuration's traceset.
- **Chakra-capable servers.** Whether each `serverId` names a Chakra-capable server is a fact about the topology. A file that names another server uploads and attaches, and the simulation fails when the simulator starts, with the simulator's error naming the `serverId`.

## Upload, attach, and simulate

### 1. Set up your environment

```bash
export SCALA_API_TOKEN="your-token-here"
```

See [Authentication](./authentication.md) for how to obtain a token.

### 2. Request upload URLs

Send the file's size in bytes and the workspace it belongs to. `name` is optional and defaults to `chakra-mapping.txt`. It must be 1 to 255 bytes of `A-Z`, `a-z`, `0-9`, `.`, `_` and `-`, must not be `.`, and must not contain `..` anywhere. `sizeBytes` is an upper bound of at most 64,000,000 bytes (the 64 MB per-file limit), and `description` holds at most 2,048 characters. The name is display metadata only: inside the simulator the file is always called `chakra-mapping.txt`.

```bash
curl -X POST "https://api.scalacomputing.com/api/v1/mapping-files/upload-url" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "workspaceId": "workspace_def456",
    "name": "reverse-256.txt",
    "sizeBytes": 4520,
    "description": "Reverse placement for the 256-rank study"
  }'
```

```json
{
  "uploadId": "upload_2cVQ8f0YyQ3nT1hKpR7mBz4LsXd",
  "checksumAlgorithm": "crc64nvme",
  "parts": [
    {
      "partNumber": 1,
      "url": "https://example.com/presigned-part-url"
    }
  ],
  "expiresAt": "2026-09-17T14:30:00Z"
}
```

### 3. Upload the bytes

`PUT` each part to its presigned URL and keep the `ETag` response header of each one.

```bash
curl -X PUT "$PART_URL" \
  --upload-file reverse-256.txt \
  --dump-header -
```

### 4. Complete the upload

Send the upload id and each part's number and ETag. The platform validates the file, derives `rankCount` and `scaleUpGroupCount`, stores the file, and answers `201` with the new mapping file. `sizeBytes` in the response is the stored file's actual length.

```bash
curl -X POST "https://api.scalacomputing.com/api/v1/mapping-files/upload-url/complete" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "uploadId": "upload_2cVQ8f0YyQ3nT1hKpR7mBz4LsXd",
    "parts": [
      {
        "partNumber": 1,
        "etag": "\"d41d8cd98f00b204e9800998ecf8427e\""
      }
    ]
  }'
```

```json
{
  "id": "mapfile_2cVQ8f0YyQ3nT1hKpR7mBz4LsXd",
  "name": "reverse-256.txt",
  "workspaceId": "workspace_def456",
  "sizeBytes": 4520,
  "rankCount": 256,
  "scaleUpGroupCount": 0,
  "hasScaleUp": false,
  "createdAt": "2026-09-16T14:30:00Z",
  "description": "Reverse placement for the 256-rank study",
  "hash": "crc64nvme:vK3Jd8Qm1pA",
  "etag": "\"0\"",
  "duplicateOfExisting": false
}
```

If the workspace already holds a live mapping file with the same bytes, under any name, the platform creates nothing and answers `200` with that existing file and `duplicateOfExisting` set to `true`. Uploading the bytes of a deleted file creates a new mapping file with a new id; a deleted file is never revived.

### 5. Attach it to a configuration

```bash
curl -X PATCH "https://api.scalacomputing.com/api/v1/configurations/{config_id}/mapping-file" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "If-Match: \"4\"" \
  -H "Content-Type: application/json" \
  -d '{
    "mappingFileId": "mapfile_2cVQ8f0YyQ3nT1hKpR7mBz4LsXd"
  }'
```

Attach requires, in this order of checks:

- a configuration outside the template workspace (`400` otherwise);
- at least one application with `trafficType` `chakra` (`422` `MAPPING_REQUIRES_CHAKRA_APP`);
- an attached traceset (`409` `MAPPING_REQUIRES_TRACESET`) whose rank count has been resolved: while it is unresolved, attach answers `409` `CONFLICT`, changes nothing, and is safe to retry;
- a mapping file in the configuration's own workspace that has not been deleted (`404` otherwise);
- a mapping file whose `rankCount` equals the traceset's rank count (`422` `MAPPING_RANK_COUNT_MISMATCH`, naming both counts and the missing or extra ranks).

`If-Match` is optional. The configuration's ETag is its version number in quotes, and a stale one answers `412`. Attaching writes a new configuration version whose `activeMappingFile` names the file (excerpt):

```json
{
  "id": "config_ghi789",
  "version": 5,
  "activeMappingFile": {
    "id": "mapfile_2cVQ8f0YyQ3nT1hKpR7mBz4LsXd",
    "name": "reverse-256.txt",
    "deleted": false
  }
}
```

Attaching also sets `UseMapping` to `true` and `MappingFileName` to `chakra-mapping.txt` in every Chakra application's `ChakraConfiguration` parameters. While the file is attached, the platform owns those two parameters: a configuration update, an application-parameter update, or a new Chakra application that would change them is refused with `422` `MAPPING_PARAMS_LOCKED`. A traceset with a different rank count is refused with `409` `TRACESET_CONFLICTS_WITH_MAPPING`. A configuration update whose body carries `activeMappingFile` is refused with `400`: attach and detach are the only ways to set it. To make any of these changes, detach the mapping file, make the change, and attach it again.

One mapping file can be attached to any number of configurations in its workspace, and each configuration holds its own reference.

### 6. Create a simulation

Every simulation created from the configuration is launched with the attached file (see [Current limitations](#current-limitations)).

```bash
curl -X POST "https://api.scalacomputing.com/api/v1/simulations" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "chakra-mapped-run-1",
    "workspaceId": "workspace_def456",
    "configurationId": "config_ghi789",
    "duration": "100us"
  }'
```

The simulation records the mapping file it was created with in `appliedMappingFile` from the moment it is created, and keeps naming it after the file is deleted, with `deleted` reporting that state. A simulation just created (excerpt):

```bash
curl -X GET "https://api.scalacomputing.com/api/v1/simulations/{sim_id}" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

```json
{
  "id": "sim_abc123",
  "name": "chakra-mapped-run-1",
  "configurationId": "config_ghi789",
  "status": "provisioning",
  "appliedMappingFile": {
    "id": "mapfile_2cVQ8f0YyQ3nT1hKpR7mBz4LsXd",
    "name": "reverse-256.txt",
    "deleted": false
  }
}
```

A simulation created from a configuration with no mapping file carries no `appliedMappingFile`, and its ranks are placed in order.

Creating a simulation is refused, and no simulation is created, when:

- the attached mapping file has been deleted: `409` `MAPPING_FILE_DELETED_REFERENCE`, naming the file. Detach it, or attach another, and create again;
- the attached mapping file's name contains `..` or a shell metacharacter: `400` `VALIDATION_ERROR`, naming the field. Upload rejects such names, but a file stored before that rule keeps its name; upload it again under another name and attach it in place of the old one.

A run never falls back to in-order placement: a mapping file that cannot be delivered to the simulator fails the simulation. When the platform rejects the file while staging it, for example because the stored object is missing, over the size limit, or fails its checksum, the simulation's `failureReason` reads `Mapping file <name> (<id>) could not be delivered`. On some runs a long `<name>` is shortened and ends in `…`; the id is always given in full. A failure the platform does not attribute to the mapping file, such as a storage or infrastructure fault while staging it, reports a general failure reason instead.

### 7. Check the placement the run used

When a mapped run completes, the simulator has written the placement it applied to one CSV result file whose name contains `-chakra-mapping-`. List the run's CSV results, find that file by name (page through with `next` if needed), and download it through its `downloadUrl` or the download-url endpoint:

```bash
curl -X GET "https://api.scalacomputing.com/api/v1/simulations/{sim_id}/results?type=csv&limit=100" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

The file's header is `ChakraId,Hostname,NicNodeID,ServerNodeID,ServerID,Core,IPAddress`, plus `ScaleUpGroupId,ScaleUpSubgroupId` when the run uses scale-up groups. The first column is the rank and the fifth is the `serverId` it ran on, so each row can be compared with the Section I entry you uploaded. Rows are not in rank order. See [Simulation Output Files](./simulation-output-files.md) for downloading result files.

## Browse, download, and delete

List the live mapping files in a workspace, newest first. `workspaceId` is required, and `limit` is at least 1:

```bash
curl -X GET "https://api.scalacomputing.com/api/v1/mapping-files?workspaceId=workspace_def456&limit=50" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

```json
{
  "items": [
    {
      "id": "mapfile_2cVQ8f0YyQ3nT1hKpR7mBz4LsXd",
      "name": "reverse-256.txt",
      "workspaceId": "workspace_def456",
      "sizeBytes": 4520,
      "rankCount": 256,
      "scaleUpGroupCount": 0,
      "hasScaleUp": false,
      "createdAt": "2026-09-16T14:30:00Z"
    }
  ],
  "pagination": {
    "count": 1,
    "hasMore": false
  }
}
```

When `hasMore` is `true`, pass the returned `nextCursor` as the `next` query parameter to fetch the following page.

Fetch one mapping file's details, or a presigned URL (`url`, valid until `expiresAt`) that serves its exact uploaded bytes:

```bash
curl -X GET "https://api.scalacomputing.com/api/v1/mapping-files/{mapping_file_id}" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

```bash
curl -X GET "https://api.scalacomputing.com/api/v1/mapping-files/{mapping_file_id}/download" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

Delete a mapping file. The delete answers `204` whether or not a configuration refers to the file; deleting it again answers `404`:

```bash
curl -X DELETE "https://api.scalacomputing.com/api/v1/mapping-files/{mapping_file_id}" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

After a delete:

- Simulations already created are unaffected. They keep the file they were created with, and their `appliedMappingFile` reads `deleted: true`.
- A configuration that refers to the file keeps its `activeMappingFile`, which reads `deleted: true`, and creating a simulation from it is refused until you detach the file or attach another.
- Fetching the file or its download URL answers `410` `MAPPING_FILE_DELETED`.

Detach a mapping file from a configuration to restore in-order placement. The mapping file itself is unchanged and can still be attached elsewhere:

```bash
curl -X DELETE "https://api.scalacomputing.com/api/v1/configurations/{config_id}/mapping-file" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

## Errors

| Status | Code | Operation | When |
|--------|------|-----------|------|
| `400` | `VALIDATION_ERROR` | upload-url | `name` is not a safe single segment or contains `..`; `sizeBytes` is outside 1 to 64,000,000; `workspaceId` is absent, blank, or padded; `description` is over 2,048 characters |
| `400` | `VALIDATION_ERROR` | complete | The upload session is not a mapping-file upload, or the parts do not match the session |
| `404` | `NOT_FOUND` | complete | No such upload session |
| `409` | `CONFLICT` | complete | The upload session is already being completed, or was completed or aborted |
| `409` | `MAPPING_FILE_HASH_COLLISION` | complete | A different file with the same display name is already stored under the same content checksum. Upload it under another name. A workspace holds at most two different files with one checksum; a third is refused with this code, naming the two, until you delete one |
| `410` | `GONE` | complete | The upload session expired |
| `413` | `PAYLOAD_TOO_LARGE` | complete | The uploaded object is larger than the declared `sizeBytes`, or above 64,000,000 bytes |
| `422` | `MAPPING_FILE_INVALID` | complete | The file breaks a format rule; the message names the first offending line or missing rank |
| `503` | `SERVICE_UNAVAILABLE` | complete | Another upload of the same bytes was in progress for too long. The session is closed: open a new upload session and upload again; the new upload is most often answered `200` with `duplicateOfExisting` set |
| `400` | `VALIDATION_ERROR` | list | `workspaceId` is absent or blank, `limit` is zero or does not parse, or the cursor is invalid or expired |
| `404` | `NOT_FOUND` | get, download, delete | No such mapping file; for delete, also one already deleted |
| `410` | `MAPPING_FILE_DELETED` | get, download | The mapping file has been deleted |
| `400` | `VALIDATION_ERROR` | attach, detach | The configuration is in the template workspace, or an id or the body is malformed |
| `404` | `NOT_FOUND` | attach, detach | No such configuration; for attach, a mapping file that is absent, deleted, or in another workspace; for detach, no mapping file attached |
| `409` | `MAPPING_REQUIRES_TRACESET` | attach | The configuration has no traceset attached |
| `409` | `CONFLICT` | attach | The traceset's rank count is not resolved yet; safe to retry |
| `412` | `PRECONDITION_FAILED` | attach, detach | `If-Match` does not match the configuration's current ETag |
| `422` | `MAPPING_REQUIRES_CHAKRA_APP` | attach | The configuration has no Chakra application |
| `422` | `MAPPING_RANK_COUNT_MISMATCH` | attach | The file's `rankCount` differs from the traceset's rank count |
| `422` | `MAPPING_PARAMS_LOCKED` | configuration update, application update or create | The change would alter `UseMapping` or `MappingFileName` while a mapping file is attached |
| `409` | `TRACESET_CONFLICTS_WITH_MAPPING` | traceset attach, configuration update | The traceset's rank count differs from the attached mapping file's |
| `409` | `MAPPING_FILE_DELETED_REFERENCE` | create simulation | The attached mapping file has been deleted |

All errors use the shared error shape:

```json
{
  "error": {
    "code": "MAPPING_FILE_INVALID",
    "message": "line 3: expected a non-negative base-10 serverId that fits in 32 bits, got 'x'"
  }
}
```

See the [Mapping Files API reference](./api-reference/mapping-files.md) for every field and status code of each operation.
