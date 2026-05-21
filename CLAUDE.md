# `hush-protocol` — Claude Code operating context

You are working in **`hush-protocol`**, the OpenAPI 3.1 spec repository for Hush.

## Source of truth — non-negotiable

`hush-api.yaml` is **the** contract for every Hush component. **Nobody** modifies request/response schemas in `hush-backend`, `hush-app` or `hush-device` without first updating this file. If you find code that defines a request/response shape that isn't derived from this spec, that's a bug — flag it.

## Conventions

- **OpenAPI 3.1** strict. JSON Schema 2020-12 keywords are allowed.
- **camelCase** in JSON payload field names.
- **snake_case** in path parameters and query strings.
- **UTC ISO-8601** timestamps. Always include the `Z` suffix.
- **UUIDs** as `string` with `format: uuid` for resource identifiers.
- **Errors**: a single `Error` schema with `code` (machine-readable enum), `message` (human-readable), and optional `details` map.
- **Pagination**: cursor-based with `cursor` query param and `nextCursor` in response. Never offset/limit.
- **Versioning**: URL prefix `/v1`. New major under `/v2`. `info.version` follows SemVer.

## Two auth schemes (`components.securitySchemes`)

| Name | Type | Used by |
|---|---|---|
| `deviceHmac` | `apiKey` in header (`Authorization`) | All `/v1/device/*` endpoints |
| `userJwt` | `http bearer` with `bearerFormat: JWT` | Everything else |

Public endpoints (e.g. `/v1/health`, `/v1/auth/login`) declare empty `security: []`.

## Common commands

```bash
# Validate spec — must pass before commit
npx @redocly/cli lint hush-api.yaml

# Preview docs locally with live reload
npx @redocly/cli preview-docs hush-api.yaml

# Build static HTML for GitHub Pages
npx @redocly/cli build-docs hush-api.yaml -o site/index.html

# Diff a working copy vs. main (useful before opening cross-repo PRs)
npx @redocly/cli diff main:hush-api.yaml hush-api.yaml
```

CI runs `redocly lint` on every PR; do not push if local lint fails.

## Cross-repo workflow

When you modify the spec, **always** tell the user which consumer repos need follow-up work. The mapping is:

| Spec change touches | Consumer to update |
|---|---|
| `paths./v1/device/*` or `components.schemas.Device*` | `hush-backend/api/src/routes/device.rs`, `hush-device/src/proto/api.rs` |
| `paths./v1/users/*` | `hush-backend/api/src/routes/users.rs`, `hush-backend/dashboard/lib/api/`, `hush-app/lib/api/` |
| `paths./v1/audio/*` | `hush-backend/api/src/routes/audio.rs`, `hush-backend/dashboard/lib/api/`, `hush-app/lib/api/` |
| `components.schemas.Error` or auth | All four consumers — flag it loudly |

When proposing a change, draft the spec edit **and** list the consumer PRs that will follow. Do not let the user discover the cross-repo work after the spec is merged.

## Operating principles for this repo

- **Spec before code.** No "I'll add it to the backend and update the spec later." That's how drift happens.
- **Examples are mandatory.** Every endpoint must have at least one request example and one success-response example before it lands.
- **No silent breaking changes.** If a field becomes required, that's `/v2`. If a field is renamed, that's `/v2`. Deprecate first, remove in a major.
- **Errors are part of the contract.** Document the 4xx codes a client must handle. A client that can't enumerate the errors it will see is broken.

## Where things live

| Topic | File |
|---|---|
| The spec | `hush-api.yaml` |
| Auth details | `docs/auth.md` |
| Device state machine | `docs/device-lifecycle.md` |
| Power model | `docs/power-states.md` |
| Card scan → audio playback flow | `docs/card-flow.md` |
| Audio upload + transcoding | `docs/audio-pipeline.md` |
| Object storage and presigned URLs | `docs/storage.md` |
| Curl examples | `examples/` |
