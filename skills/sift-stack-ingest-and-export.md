---
name: sift-ingest-and-export
description: Get telemetry into Sift and back out again — detect a file's config, import it against an Asset and Run, watch the resulting job, then export channel data for external analysis. Use when an agent must load test/flight/manufacturing data into Sift or pull a dataset out of it.
api: Sift API
base_url: https://api.siftstack.com
generated: '2026-08-27'
method: generated
source: openapi/sift-stack-openapi.json (every operationId below verified present in the spec); semantics from https://docs.siftstack.com/documentation/manage/set-up-api-access and https://docs.siftstack.com/documentation/reference/supported-file-formats
operations:
  - PingService_Ping
  - MeService_GetMeV2
  - AssetService_CreateAsset
  - RunService_CreateRun
  - RunService_StopRun
  - IngestionConfigService_CreateIngestionConfigV2
  - IngestionConfigService_CreateIngestionConfigFlowsV2
  - DataImportService_DetectConfigV2
  - DataImportService_CreateDataImportFromUploadV2
  - DataImportService_CreateDataImportFromUrlV2
  - DataImportService_GetDataImportV2
  - DataImportService_RetryDataImportV2
  - JobService_ListJobs
  - JobService_RetryJob
  - JobService_CancelJob
  - ChannelService_ListChannels
  - DataService_GetDataV2
  - ExportService_ExportData
  - ExportService_GetDownloadUrl
  - DlqErrorsService_ListDlqErrorsV2
---

# Ingest into Sift and export back out

## 0. Prove the credential first

`PingService_Ping` (`GET /api/v1/ping`) confirms connectivity and the base URL.
`MeService_GetMeV2` (`GET /api/v2/me`) resolves the identity behind the API key — do this before any
"my runs" style filter, because the key is user-associated and inherits that user's permissions.

## 1. Establish the Asset and the Run

`AssetService_CreateAsset` (`POST /api/v1/assets`) then `RunService_CreateRun`
(`POST /api/v2/runs`). A streaming Run is closed with `RunService_StopRun`
(`PATCH /api/v2/runs:stop`).

Names are constrained — read
<https://docs.siftstack.com/documentation/reference/naming-rules> before generating one. Uniqueness
constraints apply only to non-archived entities, so a name freed by archiving can be re-taken.

## 2. Two ways in

**File import** (CSV, Parquet, TDMS, HDF5, ULog, ROS bags, and other formats listed at
`/documentation/reference/supported-file-formats`):

1. `DataImportService_DetectConfigV2` — let Sift infer the channel layout rather than hand-writing it.
2. `DataImportService_CreateDataImportFromUploadV2` (direct upload) or
   `DataImportService_CreateDataImportFromUrlV2` (fetch from a URL).
3. Poll `DataImportService_GetDataImportV2`; on failure use `DataImportService_RetryDataImportV2`.

**Streaming ingest** — define the shape once with `IngestionConfigService_CreateIngestionConfigV2` and
`IngestionConfigService_CreateIngestionConfigFlowsV2`, then stream over gRPC. Streaming ingest is a gRPC
service (`sift.ingest.v1`) and is not fully expressible over the REST transcoding; use the Python or Rust
client for it.

## 3. Watch the work

Imports and exports run as jobs. `JobService_ListJobs` (`GET /api/v1/jobs`) with a CEL filter,
`JobService_RetryJob`, `JobService_CancelJob`. Cancelling a job is the reversal for anything still in
flight.

Rows Sift could not parse land in a dead-letter queue — read them with
`DlqErrorsService_ListDlqErrorsV2`. Silence in the DLQ is the only real confirmation that an import was
clean; a job reporting success does not by itself mean every row landed.

## 4. Get data back out

- `ChannelService_ListChannels` (`GET /api/v2/channels`) to discover signals.
- `DataService_GetDataV2` for direct reads.
- `ExportService_ExportData` (`POST /api/v1/export`) for a bulk export, then
  `ExportService_GetDownloadUrl` (`GET /api/v1/export/{jobId}/download-url`) once the job completes.

## Pagination, everywhere

`pageSize` / `pageToken` / `filter` / `orderBy` are the house convention across all 77 list operations.
`filter` is a **CEL** expression (google/cel-spec), not a query DSL of Sift's own. `pageSize` defaults to
50 and is capped at 1000. Every non-token parameter must be identical across a paginated sequence.

## Reversibility

Import is additive: it creates Channels and populates a Run. There is no single "undo import" operation.
The reversal path is to archive what the import created — `ChannelService_BatchArchiveChannels`,
`AssetService_ArchiveAsset` — or, where the resource supports it, delete it (`RunService_DeleteRun`).
Sift publishes **no restore window** for archived entities, so plan the import into a throwaway Asset when
you are unsure rather than relying on being able to unwind it later.

## Errors

`rpcStatus` bodies (`code` / `message` / `details[]`), gRPC status semantics. `429` (REST) and
`RESOURCE_EXHAUSTED` (gRPC) mean the request was not processed and is safe to retry with backoff. Sift
publishes no per-endpoint limit values; batch rather than looping single-record calls, and spread large
backfills over time.
