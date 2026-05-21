# Card flow

What happens between "child taps a card" and "speaker plays audio".

## Table of contents

- [End-to-end sequence](#end-to-end-sequence)
- [Cache lookup](#cache-lookup)
- [Cache miss handling](#cache-miss-handling)
- [Unknown card](#unknown-card)
- [Concurrency: card swapped mid-playback](#concurrency-card-swapped-mid-playback)
- [Event reporting](#event-reporting)

---

## End-to-end sequence

```
+-------+     +--------+     +---------+     +-----------+     +-------+
| child | --> | RFID   | --> | cache   | --> | I2S       | --> | spkr  |
| taps  |     | reader |     | (SD)    |     | playback  |     |       |
+-------+     +--------+     +---------+     +-----------+     +-------+
                                  |
                                  | miss
                                  v
                              +---------+     +----------+
                              | backend | --> | object   |
                              | sync    |     | storage  |
                              +---------+     +----------+
```

1. `MFRC522` IRQ fires.
2. Device reads the UID.
3. Device looks up the UID in its cached `CardBinding` table → resolves an `audioId`.
4. Device looks up the audio file on the microSD cache.
5. **Hit**: start I2S playback immediately.
6. **Miss**: download from the presigned URL stored in the last sync, write to SD, then play.

## Cache lookup

> TODO(phase-1): The cache index lives in `/cache/index.bin` on the SD card and is hash-keyed by `audioId`. Files are stored as `/cache/<audio-id>.mp3`. Integrity is verified by comparing the SHA-256 in the index to the SHA-256 in the latest sync.

## Cache miss handling

> TODO(phase-1): If no presigned URL is cached (e.g. expired), do an opportunistic `GET /v1/device/sync` first. If still missing, flash the LED amber and post a `card_unknown` or `error` event.

## Unknown card

UID not present in the binding table:

1. LED blinks red briefly.
2. Device posts a `card_unknown` event in the next batch.
3. Dashboard surfaces this as "Unknown card seen, want to bind it?" with the UID pre-filled.

## Concurrency: card swapped mid-playback

> TODO(phase-1): If a new UID is detected while audio is playing, fade out current playback (200 ms), free the I2S DMA buffer, look up the new UID and start the new track. No queue.

## Event reporting

Each scan produces a `card_scanned` event in the firmware buffer. Events are flushed:

- After 8 events queued.
- After 30 seconds since the last flush.
- On entering LIGHT_SLEEP.
- On any `POST /v1/device/events` retry.
