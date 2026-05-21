# Device lifecycle

How a device goes from factory to a user's living room.

## Table of contents

- [States](#states)
- [State diagram](#state-diagram)
- [Transitions](#transitions)
  - [Factory → provisioned](#factory--provisioned)
  - [Provisioned → online](#provisioned--online)
  - [Online → claimed](#online--claimed)
  - [Claimed → retired](#claimed--retired)
- [Reset semantics](#reset-semantics)

---

## States

| State | Meaning |
|---|---|
| `factory` | Hardware exists. No firmware, no secrets baked in. |
| `provisioned` | Factory image flashed: per-device secret in NVS, serial assigned. |
| `online` | WiFi credentials present, device has called `/v1/device/register`. |
| `claimed` | A user owns the device. `ownerId` is non-null. |
| `retired` | Soft-deleted; device may not sync anymore. |

`factory` and `provisioned` are firmware-internal; the API only ever exposes `unclaimed`, `claimed` and `retired` (see the `Device.state` schema).

## State diagram

```
factory --flash--> provisioned --boot+wifi--> online (unclaimed)
                                                |
                                                | claim
                                                v
                                              claimed --retire--> retired
```

## Transitions

### Factory → provisioned

> TODO(phase-1): The pre-provisioning flow. A small script in `hush-backend/scripts/preprovision.sh` mints a serial + secret, the factory image picks them up.

### Provisioned → online

> TODO(phase-1): BLE pairing (Improv-WiFi), first `POST /v1/device/register` succeeds.

### Online → claimed

> TODO(phase-1): User scans the QR code on the box, dashboard calls `POST /v1/devices/{id}/claim` with the short claim code returned at registration.

### Claimed → retired

> TODO(phase-2): User clicks "retire" in the dashboard. Device receives a `retired: true` flag on the next sync and stops working. Cannot be undone from the device side (must contact support).

## Reset semantics

> TODO(phase-1): Three reset levels.
>
> - **Soft reset** (short press of reset button): reboot.
> - **WiFi reset** (long press of pairing button): clears WiFi creds, keeps secret + serial, returns to BLE pairing.
> - **Factory reset** (10s press of reset + pairing): clears NVS except secret. Re-runs registration.
