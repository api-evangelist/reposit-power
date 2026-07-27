---
name: Dispatch a Reposit power station for grid support
description: Issue a dispatch/support request instructing a power station of household batteries to deliver real power (optionally at a requested power factor), then monitor acceptance, prediction and completion.
api: openapi/reposit-power-market-api-openapi.yml
generated: '2026-07-27'
method: generated
consequence: safety-critical
operations:
  - GET /api/powerstations
  - GET /api/powerstations/{powerstationId}/data/latest
  - GET /api/dispatches
  - POST /api/dispatches
  - GET /api/dispatches/{dispatchId}
  - GET /api/nodes/{nodeId}/events
---

# Dispatch a Reposit power station for grid support

Base URL: `https://marketapi.repositpower.com/`

> **This skill commands real hardware in real Australian homes.** A dispatch
> discharges household batteries into the grid. Treat every write as
> safety-critical: human authorisation, full logging, no blind retries. See
> `agentic-access/reposit-power-agentic-access.yml`.

> The Market API declares no `operationId`, so steps name method and path exactly
> as they appear in `openapi/reposit-power-market-api-openapi.yml`.

## Before you start

- Authenticate with `Authorization: Bearer <api key>`. Dispatch operations are
  tagged `Network` and `Retailer`.
- `powerFactor` and `powerFactorLeadLag` are accepted **only if your account has
  been enabled for power factor / voltage control**. Do not send them speculatively.

## Steps

1. **Pick the target and check its aggregate rating.** `GET /api/powerstations`
   returns each power station's `namePlate` (battery capacity, battery power,
   inverter power). Do not request more real power than the aggregate can deliver.
2. **Check current headroom before committing.**
   `GET /api/powerstations/{powerstationId}/data/latest?metrics=remainingCharge,batteryPower,meterPower`
   returns the most recent value per metric, bounded by `maxAge` (default 30
   seconds). `remainingCharge` is the summed remaining charge of the batteries in
   the aggregate — the single most useful pre-dispatch check.
3. **Check what is already scheduled.** `GET /api/dispatches?filter=UPCOMING`
   (also `INPROGRESS`, `COMPLETED`), paged with `offset` and `limit` (defaults 0
   and 100).
4. **Create the dispatch.** `POST /api/dispatches` with the four required fields:
   - `powerstation` — the power station id to dispatch
   - `startTime` — UNIX timestamp
   - `duration` — seconds
   - `realPowerP` — real power to be dispatched, in kW
   - optional `powerFactor` (number) and `powerFactorLeadLag` (`LEADING` or
     `LAGGING`), both subject to the account enablement above.
   The response returns `{data: {id: "<uuid>", status: "OK"}}`.
5. **Monitor it.** `GET /api/dispatches/{dispatchId}` returns
   `request.deploymentsRequested` versus
   `currentResponse.deploymentsAccepted` / `deploymentsResponded`, the delivered
   and `rejectedRealPowerP`, the achieved `powerFactor` / `powerFactorLeadLag`, a
   `prediction` block (`data[]`, `deltaSeconds`, `startTime`), the enlisted
   `nodes[]` and the `state`.
6. **Reconcile per household afterwards.**
   `GET /api/nodes/{nodeId}/events?limit=10` shows `eventType: DISPATCH` entries
   with the dispatch `id`, `ts` and `duration` — the per-node record that the
   command reached that home.

## Rules and gotchas

- **Acceptance is partial by design.** `deploymentsAccepted` is routinely below
  `deploymentsRequested`, and `rejectedRealPowerP` records the shortfall. Settle
  against the response, never against the request.
- **No idempotency key exists.** A retried `POST /api/dispatches` creates a second
  dispatch. On timeout, re-read `GET /api/dispatches?filter=UPCOMING` and
  reconcile before resending.
- **Only 200 and 401 are declared.** There is no documented error shape for an
  over-capacity request, an overlapping dispatch, or a malformed power factor.
  Fail closed on anything unexpected.
- **Dispatch is not curtailment.** Dispatch asks for delivery (positive real power
  in the examples); curtailment limits export (`realPowerP` must be negative). See
  `skills/reposit-power-run-export-curtailment.md`.
- **Telemetry windows are bounded** if you follow up with `/data` or `/data/raw`:
  `format=nodes` is limited to 6 hours, `format=powerstation` to 1 week. See
  `conventions/reposit-power-conventions.yml`.
- **All timestamps are UNIX seconds.**
