---
title: "Workload Metadata"
description: "workloadMetadata describes the workload a traceset was captured from. You send it when you request upload URLs (POST /api/v1/tracesets/upload-url), and GET..."
---

`workloadMetadata` describes the workload a traceset was captured from. You send it when you request upload URLs (`POST /api/v1/tracesets/upload-url`), and `GET /api/v1/tracesets/{id}` returns it as stored. This page describes it on upload, where its schema is `UploadWorkloadMetadata` in the [OpenAPI description](./api-reference/openapi.json). Clients generated from an earlier version of the OpenAPI description name this type `WorkloadMetadata`; the JSON is the same.

The schema describes the members that declare the traceset's rank count: `shape.rankCount` and the `parallelism` degrees. Every member is optional. The object may also carry descriptive members (see [Descriptive members](#descriptive-members)). Upload stores and returns the whole object as given, up to 65,536 bytes when serialized, and validates only the rank-count members.

## Declaring the rank count

An upload that contains `primary` trace files must declare its rank count in one of two ways, or both:

- **`shape.rankCount`**: the total rank count.
- **`parallelism`**: the degrees `dp`, `tp`, `sp`, `ep` and `pp`. The rank count is their product, and an absent degree counts as 1. Other degrees, such as `cp` and `fsdp`, may be given but are not factors, and upload does not check them; promotion checks `cp` and `fsdp`.

The rules:

- Each of `shape.rankCount`, `dp`, `tp`, `sp`, `ep` and `pp` that you give must be a whole number of at least 1. `0`, `-1`, `2.0` and `"4"` are refused.
- When both forms are given, `shape.rankCount` must equal the product.
- The rank count must not exceed 2,147,483,647.
- JSON `null` counts as absent, for `shape`, `parallelism` and each value inside them.
- A `parallelism` object that holds none of `dp`, `tp`, `sp`, `ep` and `pp` declares nothing, and promotion refuses such a `parallelism`.
- The key is `rankCount`. A `shape.rank_count` is not read as a declaration.

An upload of supporting files only (no `primary` file) does not need a rank count.

The traceset reports its rank count as `rankCountDerived` on `GET /api/v1/tracesets/{id}`. A rank count cannot be changed after upload.

## Members

| Member | Type | Description |
|---|---|---|
| `shape` | `WorkloadShape` or null | Workload scale (below) |
| `parallelism` | `WorkloadParallelism` or null | Parallelism degrees (below) |

### `WorkloadShape`

| Member | Type | Description |
|---|---|---|
| `rankCount` | integer, 1 to 2,147,483,647, or null | Total rank count. When parallelism degrees are also given, their product must equal it |

Other members of `shape` are stored and returned as given.

### `WorkloadParallelism`

| Member | Type | Description |
|---|---|---|
| `dp` | integer, at least 1, or null | Data parallelism degree |
| `tp` | integer, at least 1, or null | Tensor parallelism degree |
| `sp` | integer, at least 1, or null | Sequence parallelism degree |
| `ep` | integer, at least 1, or null | Expert parallelism degree (MoE) |
| `pp` | integer, at least 1, or null | Pipeline parallelism degree |

Other members of `parallelism`, such as `cp` and `fsdp`, are stored and returned as given. Upload does not check them; promotion checks `cp` and `fsdp`.

## Descriptive members

`workloadMetadata` may also describe the workload, for example its family, name, precision or accelerator. Upload does not read these members: it stores and returns them as given, whatever their spelling. The schema does not describe them yet, because the platform does not spell them consistently. Where it reads them, it reads snake_case, for example `workload_family`, `workload_name` and `shape.global_batch_size`: traceset promotion (the Python client's `promote_traceset`) and the traceset list's workload filters, which this API description does not yet document, both do. Declare the rank count itself in the camelCase spellings above: upload reads it that way, and so does promotion's guard against changing it, where a `shape.rank_count` does not count as a declaration. Promotion's member checks do read a `shape.rank_count`, and refuse one that differs from the parallelism product. If you promote a traceset with new metadata, promotion can refuse it unless it declares, through `shape.rankCount` or the parallelism degrees, the same rank count the traceset holds (none, if it holds none). If you rely on promotion or on those filters, send the descriptive members in snake_case. Promotion also reads the whole object through a typed model and applies its own validation, so a member that upload stored without checking, such as a `cp` that is not a whole number, can make promotion fail.

## Example

A 256-rank trace, declared both ways (32 × 8 × 1 = 256):

```bash
curl -X POST https://api.scalacomputing.com/api/v1/tracesets/upload-url \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "GPT training, 256 ranks",
    "files": [
      { "name": "traces.zip", "size": 1048576, "role": "primary" }
    ],
    "workloadMetadata": {
      "parallelism": { "dp": 32, "tp": 8, "pp": 1 },
      "shape": { "rankCount": 256 }
    }
  }'
```

## Errors

`POST /api/v1/tracesets/upload-url` answers `400` before issuing any upload URL when:

- the upload has `primary` files and declares no rank count: `Uploads with primary trace files must declare a rank count: set workloadMetadata.shape.rankCount, or workloadMetadata.parallelism degrees (dp, tp, sp, ep, pp)`;
- a rank-count value is not a whole number of at least 1, naming the member and the value;
- `shape.rankCount` and the parallelism product differ, naming both;
- the rank count exceeds 2,147,483,647;
- `workloadMetadata`, `shape` or `parallelism` is neither an object nor null;
- `workloadMetadata` is larger than 65,536 bytes when serialized.
