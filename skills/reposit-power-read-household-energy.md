---
name: Read a Reposit household's own energy data
description: Authenticate as a Reposit customer, list the battery systems on the account, and read one household's solar generation, battery state of charge, house consumption, grid-connection power and earned GridCredits.
api: openapi/reposit-power-customer-api-openapi.yml
generated: '2026-07-27'
method: generated
operations:
  - POST /v2/auth/login/
  - POST /v2/auth/refresh/
  - GET /v2/userkeys/
  - GET /v2/deployments/{userkey}/components
  - GET /v2/deployments/{userkey}/battery/capacity
  - GET /v2/deployments/{userkey}/battery/historical/soc
  - GET /v2/deployments/{userkey}/generation/historical/p
  - GET /v2/deployments/{userkey}/inverter/historical/p
  - GET /v2/deployments/{userkey}/house/historical
  - GET /v2/deployments/{userkey}/meter/historical/p
  - GET /v2/deployments/{userkey}/gridcredits/historical
---

# Read a Reposit household's own energy data

Base URL: `https://api.repositpower.com/`

> The Reposit Customer API declares no `operationId` on any operation, so every step
> below names the HTTP method and path exactly as they appear in
> `openapi/reposit-power-customer-api-openapi.yml`.

## Before you start

- You must hold the Reposit account credentials of a household with an installed
  Reposit Controller. There is no signup, no sandbox, no free tier and no
  key-issuing console. Anonymous calls return HTTP 401.
- Decide which auth mode you want and use it consistently — the `Reposit-Auth`
  header must be sent on the login call **and on every subsequent request**.
  - `Reposit-Auth: Stateless` — short-lived `access_token`, full re-login on expiry.
  - `Reposit-Auth: API` — also returns a `refresh_token` for `POST /v2/auth/refresh/`.

## Steps

1. **Log in.** `POST /v2/auth/login/` with header `Reposit-Auth: Stateless` (or `API`)
   and a JSON body `{"username": "...", "password": "..."}` (schema `LoginRequest`).
   The response (`LoginResponse`) carries `access_token` and `expires_at`; under
   `Reposit-Auth: API` it also carries `refresh_token`.
2. **Send the token on everything after this.** Add
   `Authorization: Bearer <access_token>` plus the same `Reposit-Auth` header to
   every later request. The declared scheme is `bearerAuth`
   (`type: http`, `scheme: bearer`, `bearerFormat: JWT`).
3. **List the account's systems.** `GET /v2/userkeys/` returns `{"userKeys": [...]}`.
   Each entry is a battery-system identifier and is the `{userkey}` path parameter
   for every remaining call. Nothing else on this API works without it.
4. **Establish what hardware exists before reading metrics.**
   `GET /v2/deployments/{userkey}/components` returns booleans for `battery`,
   `inverter`, `solar_meter`, `load_meter`, `ac_dc`, `dc_coupled`,
   `frequency_trigger`, plus `load_phases`, `solar_phases` and the
   `solar_streams_ac` / `solar_streams_dc` maps. Do not request a solar series for
   a deployment whose `solar_meter` is false.
5. **Read the point-in-time battery value.**
   `GET /v2/deployments/{userkey}/battery/capacity` returns `batteryCapacity`, a
   number from 0.0 to 1.0 (0% to 100%).
6. **Read the time series you need.** All of these take the same three optional
   query parameters — `start` (UNIX timestamp), `end` (UNIX timestamp) and
   `delta_t` (seconds between data points, default 300):
   - `GET /v2/deployments/{userkey}/generation/historical/p` → `solarP`, solar
     generation expressed as **negative** real power in kW.
   - `GET /v2/deployments/{userkey}/inverter/historical/p` → `inverterP`, battery
     inverter real power in kW.
   - `GET /v2/deployments/{userkey}/house/historical` → `houseP`, house
     consumption in kW (also echoes `components`).
   - `GET /v2/deployments/{userkey}/battery/historical/soc` → `batterySOC` in kWh
     plus `batteryCapacity`.
   - `GET /v2/deployments/{userkey}/meter/historical/p` → `meterP`, real power at
     the grid connection in kW.
   - `GET /v2/deployments/{userkey}/gridcredits/historical` → `gridcredits`,
     earnings in Australian dollars.
   Each series comes back as an array of `[timestamp, value]` pairs.
7. **Refresh rather than re-login when using `Reposit-Auth: API`.** Before
   `expires_at`, `POST /v2/auth/refresh/` with `{"refresh_token": "..."}`
   (schema `RefreshRequest`) to get a new `access_token`.

## Rules and gotchas

- **Sign convention matters.** Solar generation is returned as negative real power.
  Do not take absolute values without saying so downstream.
- **No pagination, no envelope.** Responses are bare typed objects; there is no
  `{data, status}` wrapper here (that is the Market API's convention) and no paging
  parameters. Bound your series with `start`/`end` instead.
- **Errors are undeclared.** The specification documents only `200` responses. In
  practice an unauthenticated or failed call returns HTTP 401
  `{"error":"unauthorized"}` (probed 2026-07-27). Treat any non-200 as terminal and
  re-authenticate on 401. See `errors/reposit-power-problem-types.yml`.
- **No idempotency, no request id, no documented rate limit.** See
  `conventions/reposit-power-conventions.yml`.
- **`CostHistorical` is an orphan schema** — a retail-cost series is declared in
  `components.schemas` but no path exposes it. Do not attempt to call for it.
- **This is another person's household data.** The only delegation mechanism is
  credential sharing — there is no OAuth consent flow and no scope model. Handle
  the credentials and the data accordingly.
