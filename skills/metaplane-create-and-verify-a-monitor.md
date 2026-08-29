---
name: metaplane-create-and-verify-a-monitor
description: Create a Metaplane monitor on a warehouse table or column, run it, read its status, and pull its evaluation history — including how to back the change out, since Metaplane publishes no monitor delete.
api: Metaplane API
base_url: https://dev.api.metaplane.dev
operations:
  - getMonitors
  - createMonitor
  - getMonitor
  - updateMonitor
  - runMonitors
  - getMonitorStatus2
  - getEvaluationHistory
  - getAllMonitorsForSource
  - bulkGetTableMonitors
generated: '2026-08-29'
method: generated
source: openapi/_original/metaplane-api-openapi.yml (Metaplane's own published OpenAPI, harvested 2026-08-29)
---

# Create and verify a Metaplane monitor

## Before you start

- Auth: `Authorization: Bearer <token>`. Base: `https://dev.api.metaplane.dev`.
- You need a `connectionId` first — see `metaplane-onboard-a-warehouse-connection`.
- Targets are absolute paths: `"{database}.{schema}.{table}"` or `"{database}.{schema}.{table}.{column}"`.

## Steps

1. **Check what already exists** before creating anything, because there is no delete.
   - One entity: `GET /v1/monitors/path/{connectionId}/{absolutePath}` (`getMonitors`).
   - Many tables at once: `POST /v2/monitors/bulk-fetch/tables/{connectionId}` (`bulkGetTableMonitors`) —
     **maximum 200 table paths per request**, the only hard request ceiling Metaplane publishes.
   - Everything on a connection: `GET /v1/monitors/connection/{connectionId}` (`getAllMonitorsForSource`).
2. **Create it.** `POST /v1/monitors` (`createMonitor`) with a `MonitorEgg`:
   `type`, `connectionId`, `entityType`, `absolutePathString`, optional `cronTab`, `name`,
   `description` and a `MonitorConfig`. `type` comes from the published enum — `ROW_COUNT`,
   `COLUMN_COUNT`, `CARDINALITY`, `UNIQUENESS`, `NULLNESS`, `PERCENT_ZERO`, `PERCENT_NEGATIVE`,
   `MIN`, `MAX`, `MEAN`, `STDDEV`, `FRESHNESS`, `CUSTOM`, `PUSH`, `SUM`, `DURATION`, `GENERIC_OBJECT`.
   Returns a `PublicMonitor` — keep its `id`.
   **No idempotency key exists.** If the call times out, go back to step 1 and look before you retry.
3. **Tune sensitivity if asked.** `MonitorConfig.alertRule` carries a `PublicAnomalyAlertRule` whose
   `sensitivity` defaults to 3.0; 0.3 is the tightest bounds, 6.0 the loosest. Manual thresholds go in
   `PublicManualThresholdRule` instead.
4. **Run it.** `POST /v1/monitors/run` (`runMonitors`). Success means enqueued, not finished.
5. **Read status.** `GET /v2/monitors/status/{monitorId}` (`getMonitorStatus2`).
   A `404` here means the monitor has not been run and modeled yet — go back to step 4 and wait.
   **Use v2.** The v1 operation `GET /v1/monitors/status/{monitorId}` is deprecated — Metaplane says so
   in the summary text ("Status (deprecated)") but has NOT set `deprecated: true` in the contract, so a
   generated client will happily offer it.
6. **Pull history.** `POST /v1/monitors/evaluation-history/{monitorId}` (`getEvaluationHistory`).
   Despite being a POST it mutates nothing. Page with `createdAt` (leave empty for the most recent),
   `limit` (default and maximum 500) and `sortOrder` (`DESC` by default).

## Reversibility — read this before creating anything

**Metaplane publishes no monitor delete operation.** The only way back from step 2 is
`POST /v1/monitors/{monitorId}` (`updateMonitor`) with `isEnabled: false`, which disables the monitor
but leaves it in the account. `updateMonitor` is a partial update: omitted fields are left unchanged.

No reversal window is published for any of this, so do not promise one.

`importHistoricDataForMonitor` and `ingestDataPoint` have **no** reversal at all. Before importing,
rehearse with `isPreview: true`, which validates the import without inserting data — the only dry-run
mode in this API.

## Failure modes

Only `200` is declared across all 23 operations. `401` = bad token; `404` on status = not yet modeled.
Everything else is undocumented — surface the raw body rather than interpreting it.
