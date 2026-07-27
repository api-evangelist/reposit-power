---
name: Build and maintain a Reposit power station
description: Enumerate the nodes visible to a partner organisation and aggregate them into a STATIC or DYNAMIC power station — the unit every curtailment and dispatch command targets.
api: openapi/reposit-power-market-api-openapi.yml
generated: '2026-07-27'
method: generated
operations:
  - GET /api/nodes
  - GET /api/nodes/{nodeId}/address
  - GET /api/nodes/{nodeId}/network
  - GET /api/nodes/{nodeId}/status
  - GET /api/nodes/{nodeId}/namePlate
  - GET /api/nodes/{nodeId}/events
  - GET /api/powerstations
  - POST /api/powerstations
  - GET /api/powerstations/{powerstationId}
  - PUT /api/powerstations/{powerstationId}
  - DELETE /api/powerstations/{powerstationId}
---

# Build and maintain a Reposit power station

Base URL: `https://marketapi.repositpower.com/`

> The Reposit Power Market API declares no `operationId` on any operation, so every
> step below names the HTTP method and path exactly as they appear in
> `openapi/reposit-power-market-api-openapi.yml`.

## Before you start

- You need a Reposit Fleet account (issued to network, retailer and installer
  organisations) and an API key generated at
  `https://fleet.repositpower.com/user/settings` → API Keys → Add API Key. The key
  is shown once and is not stored by Reposit.
- Send it as `Authorization: Bearer <api key>` on every request. The key does not
  expire, may be restricted to an IP allow-list, is revocable, and **inherits the
  full permissions of the Fleet user that created it** — create a dedicated
  low-privilege Fleet account if you need a restricted key.
- Operations are tagged by audience (`Network`, `Retailer`, `Installer`,
  `Customer`). Power-station operations are `Network` and `Retailer`.

## Steps

1. **List the fleet.** `GET /api/nodes` returns `{data: [...], status: "OK"}` where
   each node carries an `id` plus URL paths for `address`, `network`, `status` and
   `namePlate`.
2. **Expand instead of N+1 fetching.** Add `?expand=address,network,status,namePlate`
   — a CSV list — to resolve those URL properties in place on the same response.
   Only fall back to the per-node reads (`GET /api/nodes/{nodeId}/address`,
   `/network`, `/status`, `/namePlate`) when you need one node.
3. **Qualify candidate nodes before including them.**
   - `GET /api/nodes/{nodeId}/status` → `operationalStatus.lastValue` and
     `ping.lastTimestamp`; a stale ping means the node is not currently reachable.
   - `GET /api/nodes/{nodeId}/namePlate` → `batteryCapacity`, `batteryPower`,
     `inverterPower`; this is what your aggregate is actually rated at.
   - `GET /api/nodes/{nodeId}/network` → the `nmi`, the National Metering
     Identifier you will need for any capability or tariff assignment.
4. **Check what already exists.** `GET /api/powerstations` lists the power stations
   belonging to your organisation. Add `?includePredictions=true` to include
   predictions.
5. **Create the power station.** `POST /api/powerstations` with `name` and `nodes`
   both required:
   - **STATIC** (the default): supply the explicit `nodes` array of node ids.
   - **DYNAMIC**: set `type: DYNAMIC`, set `nodes` to null, and supply
     `filters.postcodes` (array of strings) and/or `filters.state` (one of `ACT`,
     `NSW`, `NT`, `QLD`, `SA`, `TAS`, `VIC`, `WA`). Matching nodes are added
     automatically as they appear.
   The response returns the new `id` and the aggregated `namePlate`.
6. **Read it back.** `GET /api/powerstations/{powerstationId}` confirms membership
   and the aggregate name plate before you command anything against it.
7. **Change membership.** `PUT /api/powerstations/{powerstationId}` with `name` and
   the full `nodes` array — this is a replace, not a merge. Send the complete
   intended membership every time.
8. **Remove.** `DELETE /api/powerstations/{powerstationId}` returns `{status: "OK"}`.

## Rules and gotchas

- **Replace semantics on PUT.** Omitting a node from the `nodes` array removes it
  from the power station.
- **404 means "not yours or not there".** `{"error":"Powerstation <id> not found",
  "status":"ERROR"}` is returned for an unknown id *and* for one belonging to
  another organisation.
- **The name plate shape is inconsistent** across the spec's own examples —
  `{capacity, power}` in some responses, `{batteryCapacity, batteryPower,
  inverterPower}` in others. Handle both.
- **No idempotency key.** A retried `POST /api/powerstations` creates a second power
  station. Read back with `GET /api/powerstations` before retrying.
- **Deleting a power station removes the target of any standing curtailment
  setpoint** for it. Read `GET /api/curtailments/setpoint?powerstationId=<id>` and
  cancel anything scheduled (`GET /api/curtailments?filter=UPCOMING`) first.
- Error envelope, paging and expansion rules:
  `errors/reposit-power-problem-types.yml`,
  `conventions/reposit-power-conventions.yml`.
