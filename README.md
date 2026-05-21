# hush-protocol

> OpenAPI 3.1 specification for the Hush ecosystem. **Source of truth** for every API consumer: firmware, backend, dashboard and mobile app.

[![spec](https://img.shields.io/badge/spec-OpenAPI%203.1-blue)](./hush-api.yaml)
[![license](https://img.shields.io/badge/license-MIT-green)](./LICENSE)

Hush is an open-source RFID-activated audio device for children — see [open-hush.com](https://open-hush.com).

This repository contains the contract that every Hush component agrees on. If you are adding a field, an endpoint or an error code, **change it here first**, then regenerate the clients/types in the consumer repos.

---

## What's inside

```
hush-protocol/
├── hush-api.yaml        # The spec — single source of truth
├── docs/                # Long-form docs about auth, lifecycle, audio pipeline, etc.
├── examples/            # Curl / payload examples per endpoint (added in phase 1)
├── PLAN.md              # Roadmap by phases
├── CLAUDE.md            # Operating context for Claude Code
└── README.md            # You are here
```

---

## Quick start

You'll need [`@redocly/cli`](https://redocly.com/docs/cli/) (it ships with `npx`, no global install needed).

```bash
# Validate the spec
npx @redocly/cli lint hush-api.yaml

# Preview the docs locally (live reload on save)
npx @redocly/cli preview-docs hush-api.yaml

# Build a static HTML doc into ./site
npx @redocly/cli build-docs hush-api.yaml -o site/index.html
```

The hosted version lives at <https://docs.open-hush.com>.

---

## How the spec is consumed

| Consumer | Mechanism | Where the generated code goes |
|---|---|---|
| `hush-backend` (Rust, axum) | `utoipa` annotations validated against this spec in CI | `hush-backend/api/src/openapi.rs` |
| `hush-backend/dashboard` (Next.js + TS) | `openapi-typescript` generator | `hush-backend/dashboard/lib/api/` |
| `hush-app` (Expo + TS) | `openapi-typescript` generator | `hush-app/lib/api/` |
| `hush-device` (Rust no_std) | Hand-written types kept in sync with the spec; drift caught by CI | `hush-device/src/proto/api.rs` |

Any change to `hush-api.yaml` is a **cross-repo change**: bump the spec version, regenerate clients, and open companion PRs in the consumer repos.

---

## Two authentication models

| Audience | Mechanism | Header |
|---|---|---|
| **Devices** (`/v1/device/*`) | HMAC-SHA256 of canonical request, with a per-device pre-shared secret | `Authorization: HMAC keyId=…,signature=…,ts=…` |
| **Users** (`/v1/users/*`, `/v1/devices/*`, `/v1/audio/*`) | JWT Bearer tokens with refresh | `Authorization: Bearer <jwt>` |

Details and canonicalization rules in [`docs/auth.md`](./docs/auth.md).

---

## Versioning

- The URL is prefixed with `/v1`.
- Additive changes (new optional fields, new endpoints) bump the `info.version` minor number.
- Breaking changes are released under `/v2` and the previous version is supported for at least 6 months.

---

## Contributing

Open a PR with:

1. The change to `hush-api.yaml`.
2. Updated docs in `docs/` if the change affects behaviour, not just shape.
3. An entry in the consumer repos' PR descriptions (or follow-up issues) listing what they need to regenerate.

The CI workflow runs `redocly lint` on every PR.

---

## License

MIT — see [`LICENSE`](./LICENSE).
