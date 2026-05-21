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

> TODO(phase-2): A background worker picks up finalized items from a queue (Postgres `LISTEN/NOTIFY` initially, swap to Redis or SQS if needed).
>
> Encoding target: **MP3 128 kbps CBR mono 44.1 kHz**.
> Tool: `ffmpeg` invocation in `hush-backend/api/src/transcode/`.
>
> On success:
> - Transcoded file written to `audio/<audioId>.mp3` in object storage.
> - SHA-256 computed and stored in the DB.
> - `Audio.state` flips to `ready`.
>
> On failure:
> - `Audio.state` flips to `failed`.
> - User receives a notification (push + dashboard).

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
