---
name: metaplane-onboard-a-warehouse-connection
description: Inventory Metaplane's warehouse/BI connections, check that a connection is synced, and trigger a re-sync — the first thing to do before any monitor work, because every monitor hangs off a connectionId.
api: Metaplane API
base_url: https://dev.api.metaplane.dev
operations:
  - getAllConnections
  - getConnectionStatus
  - syncConnection
  - updatePrivateKey
generated: '2026-08-29'
method: generated
source: openapi/_original/metaplane-api-openapi.yml (Metaplane's own published OpenAPI, harvested 2026-08-29)
---

# Onboard and verify a Metaplane connection

Every monitor in Metaplane belongs to a connection — a warehouse, a BI tool or a dbt project.
Nothing else in this API works until you have a `connectionId`.

## Before you start

- Auth: `Authorization: Bearer <YOUR_SECRET_API_TOKEN>` on every request.
  Tokens are created at https://app.metaplane.dev/account/manage-tokens and shown once.
- Base: `https://dev.api.metaplane.dev` — that "dev." prefix is the production host. `api.metaplane.dev` 404s.
- There is no idempotency key on this API. Do not blind-retry a POST.

## Steps

1. **List the connections.** `GET /v1/connections` (`getAllConnections`). Returns an array of
   `Connection` — `id`, `name`, `type`, `isEnabled`, `createdAt`, `updatedAt`, `status`.
   This endpoint is unpaginated: you get the whole set in one response.
2. **Pick the connection** whose `name`/`type` matches the warehouse you were asked about and
   hold its `id`. Ids are bare UUIDs with no type prefix — carry them labelled, because a
   `connectionId` and a `monitorId` are indistinguishable by inspection.
3. **Check freshness before trusting anything downstream.**
   `GET /v1/connections/{connectionId}/sync/status` (`getConnectionStatus`) returns a
   `ConnectionSyncStatusResult`. If the connection has not synced recently, monitor and lineage
   answers will be stale.
4. **Trigger a re-sync only if step 3 says it is needed.**
   `POST /v1/connections/{connectionId}/sync` (`syncConnection`).
   A 200 here does NOT mean the sync finished — Metaplane's own doc says so. It means the task
   was enqueued. Poll step 3 until the status reports completion; do not fire `syncConnection` again
   while one is in flight, because nothing deduplicates it.

## Reversibility

`syncConnection` has no cancel operation. It is safe (read-side refresh) but it cannot be called back.

`POST /v1/connections/{connectionId}/update-private-key` (`updatePrivateKey`) **overwrites** the
connection's credential. The API never returns the previous value, so the only way back is to post the
old key again — which means you must already hold it. Do not call this operation on an agent's own
initiative; escalate to a human.

## Failure modes

The published contract declares only `200` responses, so treat anything else as undocumented:
- `401` with an empty body — missing or invalid token.
- Assume 4xx/5xx bodies are unstructured. There is no RFC 9457 problem+json here.
