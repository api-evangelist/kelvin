---
generated: '2026-08-14'
method: generated
name: Run an energy-renovation simulation from an address
description: Create a kelvin simulation from coordinates, record the occupant profile, run the model, and read back the dwelling's current energy performance and the costed renovation scenarios.
api: openapi/kelvin-api-openapi.yml
operations: ["POST /api/v3/simulations", "PUT /api/v3/simulations/{simulation_id}/housing", "PUT /api/v3/simulations/{simulation_id}/qualification", "POST /api/v3/simulations/{simulation_id}/run", "GET /api/v3/simulations/{simulation_id}/initial-state", "GET /api/v3/simulations/{simulation_id}/projected-state"]
source: >-
  Grounded in openapi/kelvin-api-openapi.yml (kelvin API v3, harvested 2026-08-14 from
  https://app.go-kelvin.com/api/docs; verbatim copy in openapi/_original/). Every path,
  method, field name, status code and enum below is verified verbatim in that spec.
  kelvin declares NO operationId on any operation, so operations are identified by
  method + path. Auth per authentication/kelvin-authentication.yml, errors per
  errors/kelvin-problem-types.yml, retry and polling semantics per
  conventions/kelvin-conventions.yml.
---

# Run an energy-renovation simulation from an address

This is the core kelvin flow: from a point on the map, produce a dwelling's current DPE
performance and a set of costed renovation scenarios with French public aid applied.

## Auth

- `Authorization: Bearer team-api-key-…` — one long-lived key per team. There is no
  OAuth flow and no self-serve key endpoint; kelvin issues the key.
- Base URL: `https://app.go-kelvin.com`.
- The key carries the tenant. Every resource you touch belongs to that team's
  `team_id`; you never pass it.

## Steps

1. **Create the simulation** — `POST /api/v3/simulations` with `latitude` and
   `longitude` (both required) and optionally `ban_id`, the Base Adresse Nationale
   interoperability key. Returns `201` with `{"simulation_id": "…"}` — a ten-character
   slug such as `hjjcm1qp28`.
   To obtain the coordinates and `ban_id` from a human, kelvin's own documentation
   points at its polygon-selection map iframe
   (`https://app.go-kelvin.com/docs/simulator-map-iframe`); note that page is behind
   HTTP Basic auth, so ask kelvin for access rather than assuming you can read it.
2. **Declare what you already know about the dwelling** *(optional)* —
   `PUT /api/v3/simulations/{simulation_id}/housing` with `epc_id`, `housing_type`,
   `surface`, `floor_level`, `number_of_exterior_wall`. Returns `204`. Skip this and
   kelvin infers everything from open data and its model.
3. **Record the occupant profile** — `PUT /api/v3/simulations/{simulation_id}/qualification`.
   `profile`, `household_size` and `income_range` are required; `primary_residence`,
   `tax_residence_department` and `project_maturity` are optional. `profile` is one of
   `owner_resident`, `owner_non_resident`, `renter`, `lessor`. Returns `204`.
   **Do this before step 4** — the specification states this call must precede `/run`.
   It is also what makes the aid amounts correct: MaPrimeRénov' is means-tested, so a
   missing or wrong `income_range` yields a plan whose `financial_support.mpr` is
   wrong, and the customer will quote it to a homeowner.
   A `400` here specifically means an invalid French fiscal department code.
4. **Run the model** — `POST /api/v3/simulations/{simulation_id}/run`. Returns `201`
   with no body. This is asynchronous.
5. **Poll the initial state** — `GET /api/v3/simulations/{simulation_id}/initial-state`.
   While the model is still computing this returns `409 Conflit`. When it returns `200`
   you get `overall_rating`, `energy_rating`, `energy_consumption` (kWh/m²/an),
   `carbon_rating`, `carbon_emissions` (kg CO₂/m²/an), `confidence_score`,
   `energy_loss_percentage` broken out by walls/openings/low_floor/high_floor, the
   insulation objects (`walls_insulation`, `windows_insulation`,
   `high_floor_insulation`, `low_floor_insulation`, each `{level, u_value}`), the
   heating and hot-water systems, and `sources`.
6. **Read the scenarios** — `GET /api/v3/simulations/{simulation_id}/projected-state`.
   Also `409` until ready. Returns `renovation_plans[]`, each with `energy_rating` and
   `carbon_rating` after works, `budget`, `yearly_energy_savings`, `yearly_energy_cost`,
   a `financial_support` object (`mpr`, `cee`, `ecoptz`, and a `local` map of named
   regional aids), a `kpi` block with `property_value_increase`, and the itemised
   `renovation_plan` by category.

## Rules an agent must follow

- **Read `sources` before you trust a number.** The initial state carries a `sources`
  object reporting, per field, whether the value came from `ai`, `user` or `dpe`. An
  `ai`-sourced `energy_rating` is a model estimate; a `dpe`-sourced one comes from a
  filed certificate. Do not present the two with equal confidence. `confidence_score`
  qualifies the DPE letter specifically.
- **Correct the model rather than argue with it.** If a field is wrong, write it back
  with `PUT /api/v3/simulations/{simulation_id}/initial-state` and re-read. That
  endpoint accepts the full physical description — `wall_material`, `construction_year`,
  `generator_type`, `generator_energy`, `hot_water_type`, `vents_type`,
  `collective_heating`, `high_floor_kinds`, `low_floor_kinds` and the rest — and every
  enum was revised in v3 to match the DPE/ADEME reference system, so send the exact
  enum token, not a French label.
- **Nothing here is idempotent.** kelvin publishes no `Idempotency-Key` and no
  client-supplied request id (`conventions/kelvin-conventions.yml`). A timed-out
  `POST /api/v3/simulations` that you blind-retry produces a *second* simulation for
  the same building. Re-list with `GET /api/v3/simulations` and match on
  `created_at` / `tracking_context` before creating again.
- **`409` is the retry signal, and it carries no `Retry-After`.** Choose your own
  backoff — a few seconds between polls is sensible. Do not treat `409` as a failure;
  it means "still computing".
- **`403` is a product boundary, not a bug.** The Qualification-tagged endpoints belong
  to the "offre Qualification"; if the team bought only the Simulateur offer they will
  never succeed. The body will not tell you which — see
  `errors/kelvin-problem-types.yml`.
- **Errors are a flat `{"error": "…"}` string.** No RFC 9457, no error codes, no
  field-level validation array. Branch on the HTTP status; the message is unstable and
  its casing differs between v2 and v3.
- **Capture `x-request-id`** from every response — it is the only correlation handle
  kelvin support has. Also read `x-api-version` to confirm you are on `3.0.0`.
- **This payload is personal data.** The simulation carries a named French occupant's
  email, phone, household size and income band. Handle under GDPR; do not log the
  `client` block.
- **The output is a commercial decision aid, not a regulatory diagnosis.** kelvin says
  so itself in its `llms.txt`: it replaces neither a DPE, nor a regulatory energy audit,
  nor a thermal auditor. Never present a `projected_state` rating as a certified DPE.
