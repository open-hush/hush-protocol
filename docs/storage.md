# Storage

Where bytes live, who holds the source of truth, and what the boundaries are.

## Table of contents

- [Stores](#stores)
- [Backend Postgres](#backend-postgres)
- [Object storage (S3 / R2)](#object-storage-s3--r2)
- [Device microSD](#device-microsd)
- [Device NVS (internal flash)](#device-nvs-internal-flash)
- [Presigned URLs](#presigned-urls)
- [What is the source of truth for what](#what-is-the-source-of-truth-for-what)

---

## Stores

| Store | Hosted | Authority for |
|---|---|---|
| Postgres | Backend infra | Users, devices, audio metadata, card bindings, events |
| Object storage (S3 / R2) | Cloudflare R2 (default) or S3-compatible | Raw and transcoded audio bytes |
| microSD | On-device | Cache of audio bytes only |
| NVS (internal flash) | On-device | WiFi credentials, device secret, last config snapshot |

## Backend Postgres

> TODO(phase-2): Schema lives in `hush-backend/migrations/`. Soft-delete vs hard-delete TBD; default to soft-delete with tombstones so devices can detect deletions across syncs.

## Object storage (S3 / R2)

> TODO(phase-2): Layout
>
> ```
> hush-audio-prod/
>   uploads/<audioId>.<ext>     # raw user upload, transient
>   audio/<audioId>.mp3         # transcoded output, durable
> ```
>
> Lifecycle: `uploads/*` expire after 7 days. `audio/*` persists until user deletes the item.

## Device microSD

- Filesystem: FAT32 (compatibility with manufactured cards).
- Layout:

  ```
  /cache/
    index.bin           # binary blob: array of {audioId, sha256, size, mtime}
    <audio-id>.mp3      # one file per cached item
  /logs/
    YYYY-MM-DD.log      # daily rotated text logs (size-capped, eviction LRU)
  ```

- Eviction policy: **LRU** when free space < 10% of card capacity. Last-played wins on ties.
- Integrity: every read checks the SHA-256 against the latest sync. Mismatch → evict + redownload.

## Device NVS (internal flash)

- Region: 1 MB partition allocated in `hush-device/partitions.csv`.
- Authority for: WiFi SSID/password, per-device HMAC secret, last `DeviceConfig`, outbox of unflushed events.
- Backed by `sequential-storage` + `esp-storage`. Never reformatted on factory reset of the SD card.

## Presigned URLs

- **Upload**: `PUT` presigned, valid 15 minutes. Returned by `POST /v1/audio`.
- **Download (user)**: not currently exposed; users stream from the dashboard via a public CDN path (TODO phase-2).
- **Download (device)**: `GET` presigned, valid for the next sync window (~10 min). Returned in every `DeviceSyncResponse.audio[].downloadUrl`.

> TODO(phase-1): Decide whether to use `R2` (cheaper, no egress fees) or `S3` (more mature SDK story) for v1. Default: R2.

## What is the source of truth for what

| Subject | Source of truth |
|---|---|
| Whether a card is bound to an audio | Backend Postgres |
| Whether an audio file is `ready` to play | Backend Postgres |
| The audio **bytes** | Object storage |
| The audio **cache** | microSD (rebuildable from object storage) |
| WiFi credentials | NVS (entered via BLE, not stored backend-side) |
| Device secret | NVS (mirrored backend-side as hash for verification) |
