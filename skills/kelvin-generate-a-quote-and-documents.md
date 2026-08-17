---
generated: '2026-08-14'
method: generated
name: Generate a quote and the renovation paperwork
description: Build a custom renovation plan against the team's own price book, then generate the quote and the CEE paperwork as PDFs and collect their download URLs.
api: openapi/kelvin-api-openapi.yml
operations: ["GET /api/v3/catalog/enabled/services", "GET /api/v3/catalog/enabled/gestures", "GET /api/v3/catalog/enabled/references", "POST /api/v3/simulations/{simulation_id}/projected-state/renovation-plans", "PATCH /api/v3/simulations/{simulation_id}/projected-state/renovation-plans/{renovation_plan_id}", "POST /api/v3/simulations/{simulation_id}/documents/commercial-offer", "POST /api/v3/simulations/{simulation_id}/documents/report", "POST /api/v3/simulations/{simulation_id}/documents/contribution-framework", "POST /api/v3/simulations/{simulation_id}/documents/dimensioning-note", "POST /api/v3/simulations/{simulation_id}/documents/sworn-statement", "GET /api/v3/simulations/{simulation_id}/documents", "GET /api/v3/simulations/{simulation_id}/documents/{id}"]
source: >-
  Grounded in openapi/kelvin-api-openapi.yml (kelvin API v3, harvested 2026-08-14 from
  https://app.go-kelvin.com/api/docs; verbatim copy in openapi/_original/). Every path,
  method, enum and status code below is verified verbatim in that spec. kelvin declares
  NO operationId, so operations are identified by method + path. Scopes per
  scopes/kelvin-scopes.yml, errors per errors/kelvin-problem-types.yml, async polling
  per conventions/kelvin-conventions.yml.
---

# Generate a quote and the renovation paperwork

Once a simulation has run (see `kelvin-run-an-energy-simulation.md`), this flow turns a
scenario into the documents a French renovation company actually hands to a homeowner
and to a CEE delegatee.

## Auth

- `Authorization: Bearer team-api-key-…`, base URL `https://app.go-kelvin.com`.
- The catalogue endpoints additionally require the `catalog:read` scope — the one scope
  kelvin names publicly. Document generation requires further scopes kelvin does **not**
  name; see `scopes/kelvin-scopes.yml`.

## Steps

1. **Load the team's price book** — the three catalogue endpoints return only what is
   enabled for this team, which is what makes a generated quote carry the company's own
   prices rather than a market average:
   - `GET /api/v3/catalog/enabled/services` — optionally filter with
     `service_technical_ids[]`.
   - `GET /api/v3/catalog/enabled/gestures` — optionally filter with
     `gesture_technical_ids[]`.
   - `GET /api/v3/catalog/enabled/references` — **requires at least one** of
     `service_technical_ids[]`, `gestures_technical_ids[]` or `reference_ids[]`, and
     returns `422 Aucun filtre fourni` if you send none. Call this after the other two
     so you have identifiers to filter on.
2. **Create the custom plan** —
   `POST /api/v3/simulations/{simulation_id}/projected-state/renovation-plans` with
   `renovation_plan` (required) and an optional `name`. Categories available in v3:
   `heating`, `hot_water`, `ventilation`, `walls`, `doors_windows`, `low_floor`,
   `high_floor`, `renewable_energy`, `summer_comfort`, `winter_comfort`,
   `thermal_bridge_treatment`, `lighting`, `sobriety`. Each category is an **array of
   items**, and each item is addressed by `technical_id` and `name` — the v2 shape
   (single object, `id`/`simplified_id`/`label`) no longer exists.
   Returns `200` with the plan, including its recomputed `financial_support` and `kpi`.
3. **Adjust it** *(optional)* — `PATCH /api/v3/simulations/{simulation_id}/projected-state/renovation-plans/{renovation_plan_id}`.
   Read it back any time with the matching `GET`.
4. **Generate the quote** —
   `POST /api/v3/simulations/{simulation_id}/documents/commercial-offer` with
   `renovation_plan_id` (required). Returns `202 Accepted`: the devis is created and PDF
   generation has started. A `422` here means the plan id is unknown for this
   simulation.
5. **Generate the supporting paperwork** — each returns `202` and takes an optional
   `scenario_ids` array:
   - `POST /api/v3/simulations/{simulation_id}/documents/report` — the full study.
   - `POST /api/v3/simulations/{simulation_id}/documents/contribution-framework` — the
     CEE *cadre de contribution*.
   - `POST /api/v3/simulations/{simulation_id}/documents/dimensioning-note` — the
     equipment *note de dimensionnement*.
   - `POST /api/v3/simulations/{simulation_id}/documents/sworn-statement` — the
     *attestation sur l'honneur*.
6. **Collect the files** — poll
   `GET /api/v3/simulations/{simulation_id}/documents/{id}` for one generation, or list
   with `GET /api/v3/simulations/{simulation_id}/documents` filtered by `type`
   (`report`, `commercial_offer`, `contribution_framework`, `dimensioning_note`,
   `sworn_statement`), `begin`/`end`, `full`, `page` and `per_page`. A finished document
   carries `generated_at`, a `metadata` object (`format`, `scenario_ids`, `type`,
   `report_template`) and a `download_url`.

## Rules an agent must follow

- **Every generator is asynchronous and none is idempotent.** `202` means "started",
  not "done", and there is no request key. Firing the same POST twice produces two
  documents — and in the case of `commercial-offer`, two quotes against the same plan.
  Before retrying a call whose response you lost, `GET .../documents?type=…` and check
  whether one already exists.
- **`409` means the simulation has not finished.** Every document endpoint declares
  `409 Conflict - La simulation n'est pas encore terminée.` Finish the run flow first.
  No `Retry-After` is sent.
- **A `403` here is ambiguous by design.** `Forbidden - Scope manquant ou document
  désactivé pour l'équipe.` covers *both* a missing scope on the key *and* the document
  type being switched off for the team, with nothing in the body to distinguish them.
  Do not retry; surface it to a human who can ask kelvin which of the two it is.
- **`report_template` is `null` for API-generated reports.** The spec says the template
  selector (`current_state` / `full`) applies to generations made in the UI; documents
  created through the API carry `null`. Do not treat that as an error.
- **Two report paths exist.** The legacy `POST/GET /api/v3/simulations/{id}/report`
  (204 / status poll) still works and predates the `documents/*` family. Prefer
  `documents/report`, which returns a listable, downloadable document record.
- **The quote reflects the team's price book, so a stale catalogue means a wrong
  price.** Reload step 1 rather than caching `technical_id` sets across sessions.
- **Errors are a flat `{"error": "…"}` string** with no code — branch on status. And no
  `429` or `5xx` is declared anywhere in the spec, so handle unexpected statuses
  defensively.
