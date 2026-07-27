---
name: Read Reposit fleet and power station telemetry
description: Query downsampled, raw and latest metrics across the whole fleet, one power station, or per node — including phase splitting, fill policy, sign conventions and the documented request-window limits.
api: openapi/reposit-power-market-api-openapi.yml
generated: '2026-07-27'
method: generated
operations:
  - GET /api/data
  - GET /api/powerstations/{powerstationId}/data
  - GET /api/powerstations/{powerstationId}/data/raw
  - GET /api/powerstations/{powerstationId}/data/latest
  - GET /api/powerstations
---

# Read Reposit fleet and power station telemetry

Base URL: `https://marketapi.repositpower.com/`

> The Market API declares no `operationId`, so steps name method and path exactly
> as they appear in `openapi/reposit-power-market-api-openapi.yml`.

## Before you start

Authenticate with `Authorization: Bearer <api key>`. The fleet-wide route
`GET /api/data` is tagged `Network`, `Retailer`, `Installer` and `Data`; the
power-station routes are `Network`, `Retailer` and `Data`.

## The metric vocabulary

`metrics` is a **required** CSV parameter on all four operations. Documented values:

| metric | meaning | unit |
|---|---|---|
| `meterVoltage` | average voltage across phases at the meter/grid | V |
| `meterFrequence` | average frequency across phases at the meter/grid | Hz |
| `meterPower` | total real power across phases at the meter/grid | kW |
| `meterReactivePower` | total reactive power at the meter/grid | var |
| `solarPower` | total real power across solar phases from the PV array | kW |
| `solarReactivePower` | total reactive power across solar phases | var |
| `remainingCharge` | total remaining charge of all attached batteries | W |
| `batteryPower` | total real power measured at the batteries | kW |
| `inverterReactivePower` | total reactive power at the inverter | var |
| `inverterApparentPower` | total apparent power at the inverter | kW |
| `meterCurrent` | total current at the meter/grid | A |

Note `meterFrequence` is spelled that way in the request vocabulary while the
phase-suffix rule in the same description spells it `meterFrequency`. Send what the
metric list documents.

## Steps

1. **Choose the scope.**
   - Whole fleet: `GET /api/data`
   - One power station: `GET /api/powerstations/{powerstationId}/data`
   - Get ids first from `GET /api/powerstations`.
2. **Choose the resolution.**
   - **Downsampled** (`/data`) — pass `interval` in seconds. Timestamps refer to
     the **start of the period**.
   - **Raw** (`/data/raw`) — no downsampling. Timestamps are sample times and
     **alignment across metrics and phases is not guaranteed**.
   - **Latest** (`/data/latest`) — one value per metric, no timestamps, no older
     than `maxAge` seconds (default 30).
3. **Bound the window.** Pass `start` and `end` as UNIX timestamps on `/data` and
   `/data/raw`. Respect the documented limits on the power-station routes:
   `format=nodes` is limited to a **6 hour** window, `format=powerstation` to a
   **1 week** window.
4. **Choose the shape.** `format=powerstation` (default) aggregates across the
   power station; `format=nodes` breaks the result down per node id.
5. **Split by phase when you need it.** Only when `format=nodes`, append a phase
   selector to the metric name: `meterVoltage{a}`, `meterVoltage{a|b}`, or
   `meterVoltage{*}` (equivalent to `{a|b|c}`). Permitted on `meterVoltage`,
   `meterFrequency`, `meterPower`, `meterReactivePower`, `solarPower` and
   `solarReactivePower`. With a phase selector the average/sum aggregation is not
   applied — you get that phase's own value.
6. **Decide the gap policy.** By default missing chunks are omitted. Pass
   `fill=null` to receive every timestamp with `null` where data is missing.
   `null` is the only supported fill value.

## Rules and gotchas

- **Sign conventions are fixed and asymmetric:** all real power follows a **sink**
  convention, all reactive power follows a **source** convention, and apparent
  power is always positive. Get this wrong and your import/export analysis inverts.
- **Phase letters are arbitrary.** Reposit states it cannot determine which phase
  is red/white/blue; single-phase installations always report on phase `a`.
- **Response shape is a list of single-key objects**, e.g.
  `data: [{"meterPower{a}": {"<ts>": <value>}}, {"meterVoltage": {...}}]` — not a
  map. Under `format=nodes` there is an extra node-id level between the metric and
  the timestamps.
- **`remainingCharge` is in watts** while the power metrics are in kW. Do not sum
  them without converting.
- **Only 200 and 401 are declared.** Exceeding a window limit or naming an unknown
  metric has no documented error shape.
- **No rate limit is documented.** Be conservative on `/data/raw` and on
  fleet-wide queries.
