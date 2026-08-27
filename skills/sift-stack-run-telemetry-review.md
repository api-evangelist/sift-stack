---
name: sift-run-telemetry-review
description: Review a hardware test Run in Sift end to end — locate the Asset and Run, evaluate rules against it, read the resulting Report, and record findings as Annotations. Use when an agent is asked to investigate a test, flight, or manufacturing run and produce reviewable findings.
api: Sift API
base_url: https://api.siftstack.com
generated: '2026-08-27'
method: generated
source: openapi/sift-stack-openapi.json (operationIds verified against the spec); semantics from https://docs.siftstack.com/documentation/reference/reports-reference and https://docs.siftstack.com/documentation/reference/rule-settings
operations:
  - AssetService_ListAssets
  - RunService_ListRuns
  - RunService_GetRun
  - RuleService_ListRules
  - RuleEvaluationService_EvaluateRulesPreview
  - RuleEvaluationService_EvaluateRules
  - ReportService_GetReport
  - ReportService_ListReportRuleSummaries
  - AnnotationService_CreateAnnotation
  - AnnotationService_ArchiveAnnotation
  - AnnotationService_UnarchiveAnnotation
  - ReportService_CancelReport
---

# Review a Run in Sift

Sift organises telemetry as **Assets** (the system under test), **Channels** (individual signals),
and **Runs** (a captured session). **Rules** are CEL expressions that flag conditions; evaluating them
against a Run produces a **Report**, and firing rules produce **Annotations**.

## Authenticate

Every request carries the API key as a bearer token. There is no OAuth flow and no scopes — a key
inherits the permissions of the user it is attached to.

```
authorization: Bearer <SIFT_API_KEY>
```

For gRPC, pass the same `authorization` header as call metadata. Base URL is
`https://api.siftstack.com` (or `https://gov.api.siftstack.com` for the AWS GovCloud environment); read
the exact base for your environment from **Manage → API Keys** in the app.

## 1. Find the Asset and the Run

`AssetService_ListAssets` (`GET /api/v1/assets`) and `RunService_ListRuns`
(`GET /api/v2/runs`) both take a CEL `filter`, an `orderBy`, and `pageSize`/`pageToken`.

- `pageSize` defaults to 50 and is **coerced down to 1000** above that maximum.
- When paginating, every other parameter must match the call that produced the `pageToken`.
- `orderBy` defaults to `created_date` descending.
- Archived records are excluded unless you pass `includeArchived=true`.

Confirm the Run with `RunService_GetRun` (`GET /api/v2/runs/{runId}`) before acting on it.

## 2. Rehearse before you evaluate

Sift gives you a real dry run. Call `RuleEvaluationService_EvaluateRulesPreview`
(`POST /api/v1/rules/evaluate-rules:preview`) first and read the annotations it *would* produce
(`v1DryRunAnnotation`). Only then call `RuleEvaluationService_EvaluateRules`
(`POST /api/v1/rules/evaluate-rules`). Prefer the preview whenever you are choosing between rule sets.

## 3. Read the Report

Evaluation produces a Report. Poll `ReportService_GetReport` (`GET /api/v1/reports/{reportId}`) and
read per-rule outcomes with `ReportService_ListReportRuleSummaries`
(`GET /api/v1/reports/{reportId}/rule-summaries`).

A Report that is still running can be stopped with `ReportService_CancelReport`
(`POST /api/v1/reports/{reportId}:cancel`).

## 4. Record findings

Create an Annotation over the time range you identified with `AnnotationService_CreateAnnotation`
(`POST /api/v1/annotations`).

## Reversibility — read this before you write

- Annotations, Rules, Assets, Channels, Metadata and Policies are **archived, not deleted**, and every one
  of those has a matching unarchive: `AnnotationService_UnarchiveAnnotation`,
  `RuleService_UnarchiveRule`, `ChannelService_BatchUnarchiveChannels`, and their `Batch*` siblings.
  Reports specifically **can be archived but not deleted** — archiving removes them from the default view
  without destroying the data.
- Sift does **not publish a retention window** for restoring an archived entity, so do not promise the user
  one. Treat archive as reversible-in-principle and confirm before relying on it.
- After archiving, a name becomes reusable by a new entity — so unarchiving into a namespace where the name
  was re-taken is not guaranteed to be clean.
- `DELETE` operations exist (24 of them, e.g. `RunService_DeleteRun`, `AssetService_DeleteAsset`) and are
  **not** covered by an unarchive. Rules are the exception — `RuleService_BatchUndeleteRules` exists.
  Treat every other delete as terminal and ask for explicit confirmation.

## Errors and retries

Errors come back as gRPC-status shaped bodies (`rpcStatus`: `code`, `message`, `details[]`), not RFC 9457
problem+json. On `429 Too Many Requests` (REST) or `RESOURCE_EXHAUSTED` (gRPC) the request was **not
processed** — it is safe to retry after the indicated delay. Back off; do not tight-loop.

## Do not

- Do not invent per-endpoint rate-limit numbers. Sift deliberately does not publish them.
- Do not assume a request header named `Idempotency-Key` exists. It does not. `client_key` is an immutable
  client-supplied identifier on some resources, not a replay-safe idempotency token.
