# Power states

Battery life is the user's most-felt quality metric. The device aggressively transitions to lower-power modes.

## Table of contents

- [Power modes](#power-modes)
- [Transitions](#transitions)
- [Wake sources](#wake-sources)
- [Sync behaviour per state](#sync-behaviour-per-state)
- [Configurability](#configurability)

---

## Power modes

| Mode | Trigger | Power | RAM | Wake latency |
|---|---|---|---|---|
| `ACTIVE` | Playing or interacting | full draw | preserved | — |
| `LIGHT_SLEEP` | > `lightSleepAfterSec` (default 30s) idle | ~1 mA | preserved | < 50 ms |
| `DEEP_SLEEP` | > `deepSleepAfterSec` (default 300s) idle | ~14 µA | lost (RTC only) | ~500 ms reboot |

## Transitions

```
ACTIVE --idle 30s--> LIGHT_SLEEP --idle 5 min--> DEEP_SLEEP
   ^                       |                          |
   |                       v                          |
   +------- GPIO wake -----+                          |
   ^                                                  |
   +-------------- GPIO wake (cold-ish boot) ---------+
```

## Wake sources

Configured on every transition into a sleep mode.

- RFID card present (`MFRC522 IRQ` on `GPIO 2`).
- Encoder rotation (`KY-040 CLK` on `GPIO 17`).
- Encoder press / pairing button.
- WiFi disconnect interrupt (LIGHT_SLEEP only).
- RTC timer (for periodic `GET /v1/device/sync`).

## Sync behaviour per state

| State | Sync frequency |
|---|---|
| `ACTIVE` | On demand + every 10 minutes |
| `LIGHT_SLEEP` | Every 10 minutes (wake → sync → sleep) |
| `DEEP_SLEEP` | On wake only |

## Configurability

`DeviceSyncResponse.config.lightSleepAfterSec` and `deepSleepAfterSec` are dashboard-configurable. Users with anxious kids may want very fast wake (low threshold), users worried about battery may want long thresholds.

> TODO(phase-1): Validate that LIGHT_SLEEP keeps WiFi modem-sleep correctly. Measure on bench.
