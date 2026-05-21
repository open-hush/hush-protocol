# Audio pipeline

How an audio file goes from "user picks a file" to "byte stream out of the speaker".

## Table of contents

- [Upload path (user → storage)](#upload-path-user--storage)
- [Transcoding](#transcoding)
- [Distribution to devices](#distribution-to-devices)
- [Decoding on device](#decoding-on-device)
- [Failure modes](#failure-modes)

---

## Upload path (user → storage)

The backend **never proxies audio bytes**. Uploads go straight to object storage via presigned URLs.

```
client                       backend                       object storage
  |                            |                              |
  |--- POST /v1/audio -------->|                              |
  |    {title, contentType}    |                              |
  |                            |                              |
  |<--- 201 {audio, upload} ---|                              |
  |                            |                              |
  |--- PUT upload.url --------------------------------------->|
  |    raw bytes               |                              |
  |                            |                              |
  |--- POST /v1/audio/{id}/    |                              |
  |    finalize -------------->|                              |
  |                            |--- enqueue transcode ------->|
  |<--- 202 {state:processing}-|                              |
```

## Transcoding

A background worker picks up finalized items from an **in-process queue** inside the API process (concurrency capped by `TRANSCODE_CONCURRENCY`, default 2). On API boot, any `audios.state='processing'` row is re-enqueued — restart recovery is automatic. Move to a separate process / Redis queue when concurrent demand exceeds the in-process cap or when horizontal scale is needed.

Encoding target: **MP3 128 kbps CBR mono 44.1 kHz**.
Tool: `ffmpeg` invocation in `hush-backend/api/src/transcode/`.

Object storage layout (same bucket, two prefixes):

- `uploads/<audioId>` — raw client upload (kept until transcode completes; lifecycle expiry 7 days catches abandoned uploads).
- `audio/<audioId>.mp3` — transcoded artifact.

On success:

- Transcoded file written to `audio/<audioId>.mp3`.
- SHA-256, size and duration written to the DB.
- `Audio.state` flips to `ready`.
- Raw upload at `uploads/<audioId>` is deleted.

On failure:

- `Audio.state` flips to `failed` with `failure_reason`.
- Raw upload retained for inspection; the user can retry by hitting `POST /v1/audio/{id}/finalize` again (phase 4 dashboard UX).

## Distribution to devices

When a device calls `GET /v1/device/sync`, every audio item the device might play is included with a **presigned GET URL** valid for the next sync window (~10 min, configurable). Devices use this URL to fetch on cache miss.

## Decoding on device

> TODO(phase-1): MP3 decoder crate selected during firmware bring-up. Candidates: Helix MP3 (proven, C bindings), `minimp3` (pure C, well-tested), `puremp3` (pure Rust, less mature). Decision in `hush-device/PLAN.md`.

## Failure modes

| Failure | Detection | User-facing behaviour |
|---|---|---|
| Upload abandoned | `state=uploading` for > 1 hour | Auto-cleanup; dashboard shows "Upload failed". |
| Transcoding errors out | ffmpeg non-zero exit | `state=failed`; user can retry from dashboard. |
| Source file unsupported | ffmpeg returns unsupported codec | Same as above, with hint message. |
| Device cache miss + sync URL expired | Device sees 403 on presigned URL | Device issues fresh `GET /v1/device/sync`. |
