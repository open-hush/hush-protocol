# `hush-protocol` — plan

This is the roadmap for the Hush API specification. The spec is the source of truth for every Hush component — firmware, backend, dashboard and mobile app — so changes here are cross-cutting.

## Purpose

A single OpenAPI 3.1 document (`hush-api.yaml`) that:

- Defines every HTTP endpoint and payload shape exchanged in the Hush ecosystem.
- Is validated in CI so consumers can trust it.
- Is published as a browsable HTML doc at <https://docs.open-hush.com>.
- Powers code generation for the backend (Rust), dashboard (TypeScript) and mobile app (TypeScript).

## Versioning

- URL prefix: `/v1`.
- `info.version` follows SemVer. Additive changes bump minor; breaking changes bump major and ship under `/v2`.
- A new major is supported in parallel with the previous one for at least 6 months.

---

## Phase 1 — Base spec **[current]**

Acceptance: the spec is valid, every endpoint has at least one example, and the HTML doc is live.

- [x] OpenAPI 3.1 skeleton with `info`, `servers`, `tags`, `components/securitySchemes`.
- [x] Device endpoints: `POST /v1/device/register`, `GET /v1/device/sync`, `POST /v1/device/events`.
- [x] User endpoints: `POST /v1/users/register`, `POST /v1/users/login`, `POST /v1/users/refresh`, `GET /v1/users/me`.
- [x] Device-management endpoints (user-side): `GET /v1/devices`, `GET /v1/devices/{id}`, `POST /v1/devices/{id}/claim`.
- [x] Audio endpoints: `POST /v1/audio`, `POST /v1/audio/{id}/finalize`, `GET /v1/audio`, `GET /v1/audio/{id}`.
- [x] Card-binding endpoints: `POST /v1/devices/{id}/cards`, `DELETE /v1/devices/{id}/cards/{uid}`.
- [ ] `redocly lint` clean in CI on every PR.
- [ ] HTML docs published to GitHub Pages at <https://docs.open-hush.com>.
- [ ] At least one curl example per endpoint, committed to `examples/`.
- [ ] Long-form docs in `docs/` filled in (auth, device-lifecycle, power-states, card-flow, audio-pipeline, storage).

## Phase 2 — Refinements

Acceptance: webhooks specified, eventing model documented, AsyncAPI added if MQTT is chosen.

- [ ] Webhook schemas (server → user-defined endpoint) for device-online / audio-finalized.
- [ ] `v1.1`: extra event types (`button_pressed`, `volume_changed`, `low_battery`, `wifi_signal`).
- [ ] AsyncAPI document if MQTT is chosen as a sync transport (decision open, see below).
- [ ] Negative examples (400/401/403/404/409/422) documented per endpoint.
- [ ] Rate-limit headers (`X-RateLimit-*`) standardised.

## Phase 3 — Multi-tenant & sharing

Acceptance: a parent can share a device with another adult, and permissions are documented.

- [ ] Org / household model in the spec.
- [ ] `POST /v1/devices/{id}/share` endpoint with role enum.
- [ ] Granular permissions (read-only, can-add-audio, can-bind-cards, admin).
- [ ] User-invitation flow.

## Phase 4 — Public read-only catalog (optional)

Acceptance: a community store of free audio content can be browsed and added to a personal library.

- [ ] `/v1/catalog/*` endpoints (read-only, no auth).
- [ ] Licensing metadata on each catalog item.
- [ ] Mechanism for a user to copy a catalog item into their library.

---

## Decisions taken

- **OpenAPI 3.1** (not 3.0) — JSON Schema 2020-12 compatibility matters for code generation.
- **JSON over HTTP** for all device traffic. No MQTT in phase 1 — keeps the firmware simple. Reconsidered in phase 2.
- **HMAC-SHA256** for devices, **JWT** (15 min access + 30 day refresh) for users.
- **Presigned URLs** for all binary audio I/O. The backend never proxies bytes.
- **`camelCase`** in JSON payloads, **`snake_case`** in path parameters and query strings.
- **UTF-8** everywhere; timestamps are ISO-8601 in UTC.

## Decisions open

- **Sync transport** — long-poll vs. SSE vs. MQTT vs. plain periodic GET. Phase 1 ships periodic GET; revisit if battery numbers force a change.
- **Audio format** — MP3 128k mono is confirmed for v1. AAC under evaluation for v2.
- **Catalog model** — public catalog (phase 4) vs. user-curated only. To be discussed when there are real users.
- **Soft-delete vs. hard-delete** — for audio items, do we keep tombstones for clients to detect deletions? Likely yes.

---

## Cross-repo impact reminders

Every spec change must be followed by:

1. `hush-backend/api/src/openapi.rs` regenerated (or `utoipa` annotations updated).
2. `hush-backend/dashboard/lib/api/` regenerated with `npm run gen:api`.
3. `hush-app/lib/api/` regenerated with `npm run gen:api`.
4. `hush-device/src/proto/api.rs` hand-updated. Drift caught by CI.

Open companion PRs in those repos and link them in the protocol PR description.
