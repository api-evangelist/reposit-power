---
name: Run an export curtailment against a Reposit power station
description: Check the curtailment envelope, issue a time-bounded export limit against a power station of real households, monitor its acceptance, and cancel or fall back safely — including the heartbeat dead-man's-switch on standing setpoints.
api: openapi/reposit-power-market-api-openapi.yml
generated: '2026-07-27'
method: generated
consequence: safety-critical
operations:
  - GET /api/constraints/curtailments
  - GET /api/curtailments
  - POST /api/curtailments
  - GET /api/curtailments/{curtailmentId}
  - POST /api/curtailments/{curtailmentId}/cancel
  - GET /api/curtailments/setpoint
  - POST /api/curtailments/setpoint
  - POST /api/curtailments/heartbeat
---

# Run an export curtailment against a Reposit power station

Base URL: `https://marketapi.repositpower.com/`

> **This skill commands real hardware in real Australian homes.** A curtailment
> limits the real power those households may export. Treat every write below as
> safety-critical: require human authorisation, log the request and the response,
> and never retry blindly. See
> `agentic-access/reposit-power-agentic-access.yml`.

> The Market API declares no `operationId`, so steps name method and path exactly
> as they appear in `openapi/reposit-power-market-api-openapi.yml`.

## Before you start

- Authenticate with `Authorization: Bearer <api key>` (Reposit Fleet → User
  Settings → API Keys). Curtailment operations are tagged `Network` and `Retailer`.
- Know your target `powerstationId` — see
  `skills/reposit-power-build-power-station.md`.

## Steps

1. **Ask what the fleet can actually do.**
   `GET /api/constraints/curtailments?powerstation=<id>&duration=<seconds>` returns
   the envelope for that power station and duration, e.g. `{data: {realPowerP: 55}}`.
   The spec is explicit that this route "should be used before executing a
   curtailment in order to determine the capabilities of that particular Virtual
   Power Plant". Do not skip it.
2. **Check what is already scheduled.**
   `GET /api/curtailments?filter=UPCOMING` (also `INPROGRESS`, `COMPLETED`; the
   description mentions `CANCELLED` although it is missing from the enum). Page
   with `offset` and `limit` (defaults 0 and 100). Overlapping curtailments on the
   same power station are your responsibility to avoid.
3. **Create the curtailment.** `POST /api/curtailments` with all four required
   fields:
   - `powerstation` — the power station id being curtailed
   - `realPowerP` — the real power limit in kW, **which must always be negative**
   - `startTime` — a UNIX timestamp
   - `duration` — seconds
   - optional `component` — `grid` or `solar`
4. **Confirm the fan-out.** `GET /api/curtailments/{curtailmentId}` returns
   `state` (`UPCOMING` / `INPROGRESS` / `COMPLETED` / `CANCELLED`), `createdAt`,
   `request.deploymentsRequested`, `currentResponse.deploymentsAccepted` and the
   `nodes[]` actually enlisted. Accepted is usually less than requested — check it
   rather than assuming full participation.
5. **Cancel when the need passes.**
   `POST /api/curtailments/{curtailmentId}/cancel` returns `{data: {status: "OK"}}`.
6. **Standing behaviour is a different surface.** A scheduled curtailment is
   time-bounded; the *default* behaviour is a setpoint:
   - `GET /api/curtailments/setpoint?powerstationId=<id>` returns
     `meterRealPowerP` and `inverterRealPowerP`. HTTP 404 with
     `{"status":"NOT FOUND"}` means the power station has never received a valid
     setpoint — that is a normal first-run state, not an error to retry.
   - `POST /api/curtailments/setpoint` with `powerstation`, `meterRealPowerP` and
     `inverterRealPowerP` (all required) changes it. Null removes a constraint on
     that component; any other value must be negative. **This persists until
     changed again.**
7. **Hold the dead-man's-switch.** `POST /api/curtailments/heartbeat` with
   `{"powerstation": "<id>"}`. Nodes receiving a heartbeat return to their default
   curtailment behaviour if the heartbeat is lost. If you are holding a
   non-default state, you must heartbeat continuously — and if your process dies,
   the fleet failing safe is the designed outcome, not a bug.

## Rules and gotchas

- **`realPowerP` must be negative.** A positive value is a request to import, not
  to curtail export, and is outside the documented contract.
- **No idempotency key exists.** A retried `POST /api/curtailments` creates a
  *second* curtailment against real homes. On a timeout, re-read
  `GET /api/curtailments?filter=UPCOMING` and reconcile before resending.
- **Only 200 and 401 are declared.** No 400/403/409/422/429/5xx shape is
  documented for any curtailment write, so validation failures have no contract.
  Fail closed on anything unexpected.
- **All timestamps are UNIX seconds**, and all durations are seconds.
- **Curtailment is not dispatch.** Curtailment limits export; dispatch requests
  delivery. See `skills/reposit-power-dispatch-fleet-support.md`.
- **Ask before you act.** The households behind these node ids did not consent to
  your agent specifically; the Fleet contract with Reposit is what authorises the
  command. Keep a human in the loop.
