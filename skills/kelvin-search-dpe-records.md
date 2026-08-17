---
generated: '2026-08-14'
method: generated
name: Find a dwelling's DPE and attach it to a simulation
description: Search published French DPE certificates by number or address key, then bind the right one to a kelvin simulation so the study is grounded in a filed certificate rather than a model estimate.
api: openapi/kelvin-api-openapi.yml
operations: ["GET /api/v3/dpes", "PUT /api/v3/simulations/{simulation_id}/housing", "GET /api/v3/simulations/{simulation_id}/initial-state", "GET /api/v3/simulations"]
source: >-
  Grounded in openapi/kelvin-api-openapi.yml (kelvin API v3, harvested 2026-08-14 from
  https://app.go-kelvin.com/api/docs; verbatim copy in openapi/_original/). Parameters,
  fields and status codes verified verbatim in that spec. kelvin declares NO
  operationId, so operations are identified by method + path. Provenance semantics per
  data-model/kelvin-data-model.yml.
---

# Find a dwelling's DPE and attach it to a simulation

A kelvin study is far stronger when it is anchored to a real filed DPE (Diagnostic de
Performance Énergétique) instead of a purely modelled estimate. This flow finds the
certificate and binds it.

## Auth

`Authorization: Bearer team-api-key-…`, base URL `https://app.go-kelvin.com`.

## Steps

1. **Search** — `GET /api/v3/dpes` with any combination of:
   - `dpe_id` — the certificate number, when the customer already has it.
   - `ban_id` — the Base Adresse Nationale interoperability key for the address. This
     is the same identifier used to create a simulation, so you usually already hold it.
   - `building_type`, `surface`, `energy_class`, `report_date` — narrowing filters for
     when an address resolves to several certificates (a block of flats, or a dwelling
     re-diagnosed after works).
   Returns `200`. A `400` means a malformed filter.
2. **Pick the right certificate.** Prefer the most recent `report_date`, and check the
   `surface` against the dwelling you are studying — an address key alone will not
   distinguish two flats in the same building.
3. **Attach it** — `PUT /api/v3/simulations/{simulation_id}/housing` with `epc_id` set
   to the chosen DPE number (alongside `housing_type`, `surface`, `floor_level`,
   `number_of_exterior_wall` if you know them). Returns `204`.
4. **Verify the binding took** — `GET /api/v3/simulations/{simulation_id}/initial-state`
   and confirm `epc_id` is populated and that `sources` now reports `dpe` rather than
   `ai` for the fields the certificate supplied.

## Rules an agent must follow

- **`sources` is the whole point of this flow.** After binding, the initial state's
  `sources` object tells you per field whether the value came from `ai`, `user` or
  `dpe`. Report that provenance to the human. A number sourced from a filed certificate
  and a number sourced from a model are not the same claim, and kelvin is one of the
  few APIs that hands you the difference.
- **Attach before you run.** Binding a DPE after the model has computed does not
  retroactively re-derive the study; sequence this ahead of
  `POST /api/v3/simulations/{simulation_id}/run`.
- **`epc_id` and `dpe_id` are the same identifier under two names** — `dpe_id` on the
  search endpoint, `epc_id` on the housing and initial-state payloads. Do not treat them
  as different keys.
- **This is external reference data.** DPE records are French public certificates;
  kelvin searches them, it does not mint them. Never present a search result as
  kelvin's own assessment, and never present a kelvin estimate as a DPE.
- **No idempotency, no rate limits published.** The `PUT` is naturally idempotent by
  virtue of being a PUT, but nothing else in this API is — see
  `conventions/kelvin-conventions.yml`.
- **Errors are a flat `{"error": "…"}` string**; branch on the HTTP status.
