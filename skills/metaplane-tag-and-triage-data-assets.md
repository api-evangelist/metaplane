---
name: metaplane-tag-and-triage-data-assets
description: Use Metaplane tags to label tables and monitors in bulk, then read assets back by tag — the routing dimension Metaplane uses to send the right incident to the right team. Every apply here has a matching remove.
api: Metaplane API
base_url: https://dev.api.metaplane.dev
operations:
  - fetchTagDefinitions
  - tagTables
  - tagMonitors
  - removeTableTags
  - removeMonitorTags
  - fetchTaggedObjects
  - fetchTaggedMonitors
generated: '2026-08-29'
method: generated
source: openapi/_original/metaplane-api-openapi.yml (Metaplane's own published OpenAPI, harvested 2026-08-29)
---

# Tag and triage data assets in Metaplane

Tags are how Metaplane prioritises assets and routes incident alerts. This is the one part of the API
where every write has a published inverse, so it is the safest surface for an agent to act on.

## Before you start

- Auth: `Authorization: Bearer <token>`. Base: `https://dev.api.metaplane.dev`.
- Tables are addressed as `"{database}.{schema}.{table}"`; monitors by UUID.
- If you use dbt, Metaplane already imports table tags from dbt automatically — check before adding.

## Steps

1. **Resolve the tag first.** `POST /v1/tags/batch/fetch-tags` (`fetchTagDefinitions`) returns
   `PublicTagDefinition` records (`tagId` — an int64, not a UUID — plus `name` and `createdAt`).
   POST here is for an expressive request body only; nothing mutates.
2. **Apply in bulk.**
   - Tables: `POST /v1/tags/batch/tag-tables` (`tagTables`) → `TagTablesResult`.
   - Monitors: `POST /v1/tags/batch/tag-monitors` (`tagMonitors`) → `TagMonitorsResult`.
   Read the result envelope rather than assuming the whole batch landed.
3. **Read assets back by tag.**
   - Objects: `GET /v1/tags/tagged-objects/{tag}` (`fetchTaggedObjects`). **This one is paginated** —
     pass `nextPageToken` and loop while the `TaggedObjectResponse.hasMore` flag is true, reading rows
     from `data`.
   - Monitors: `GET /v1/tags/tagged-monitors/{tag}` (`fetchTaggedMonitors`), with optional
     `includeDisabled` and `fetchGroups`. This one returns a bare unpaginated array.
   Two different pagination contracts in one resource — do not assume the monitor call pages.
4. **Route on it.** Tag-based alert routing is configured in the app
   (https://docs.metaplane.dev/docs/alert-routing), not through this API.

## Reversibility

Fully reversible, and this is the only place in the Metaplane API where that is true:

| Applied with | Reversed with |
|---|---|
| `tagTables` — `POST /v1/tags/batch/tag-tables` | `removeTableTags` — `POST /v1/tags/remove-table-tags` |
| `tagMonitors` — `POST /v1/tags/batch/tag-monitors` | `removeMonitorTags` — `POST /v1/tags/remove-monitor-tags` |

Metaplane publishes no time limit on either removal, so no window is claimed here.
There is no idempotency key: re-applying a tag is harmless, but confirm with step 3 rather than retrying blind.

## Failure modes

Only `200` is declared in the contract. `401` = missing or invalid token. Anything else is undocumented.
