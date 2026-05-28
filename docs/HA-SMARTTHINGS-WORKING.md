# Home Assistant SmartThings integration — working on `pi=shp` units

**Status: Confirmed working — 2026-05-28**

This document covers the setup, verification, and known limitations of using the Home Assistant SmartThings integration to control a `pi=shp` Samsung air conditioner (specifically the AR12KS / `TP6X_RAC` family, but likely applicable to other `pi=shp` units sharing the same SmartThings capability profile).

This is believed to be the **first publicly documented confirmation** of software control working on this device family. Prior community research had concluded that all cloud paths were dead — that conclusion applied to the **consumer plugin layer** (web UI / mobile app), not to the Developer REST API path that HA uses.

---

## Prerequisites

- A **SmartThings account** with the AC already paired (it appears in the app as "Samsung OCF Air Conditioner").
- A running **Home Assistant** instance (any recent version; 2024.x+ recommended).
- Your AC's **`ocf.pi`** value confirmed as `shp` (see [REPRODUCE.md](../REPRODUCE.md) step 2).

---

## Setup

### 1. Add the SmartThings integration

In Home Assistant:

1. Go to **Settings → Devices & Services → Add Integration**.
2. Search for **SmartThings**.
3. When prompted, open the SmartThings OAuth page and log in with your Samsung account.
4. Authorize HA to access your SmartThings location.

HA will import all SmartThings devices, including your AC as a `climate` entity.

### 2. Find your climate entity

Your AC will appear as `climate.<room_name>` or similar. The entity will show current temperature, mode, and fan speed (though these reads may be stale — see the State Sync section below).

### 3. Test basic control from Developer Tools

In HA Developer Tools → Services:

```yaml
service: climate.set_hvac_mode
target:
  entity_id: climate.<your_ac_entity>
data:
  hvac_mode: cool
```

Watch your physical unit. It should power on and switch to cool mode within ~2 seconds.

---

## Supported capabilities

| HA service | SmartThings capability | Verified | Notes |
|---|---|---|---|
| `climate.set_hvac_mode("cool")` | `switch.on` + `airConditionerMode.setAirConditionerMode("cool")` | ✅ | Compressor starts |
| `climate.set_hvac_mode("dry")` | `switch.on` + `airConditionerMode.setAirConditionerMode("dry")` | ✅ | Dry mode |
| `climate.set_hvac_mode("fan_only")` | `switch.on` + `airConditionerMode.setAirConditionerMode("wind")` | ✅ | Fan only, no compressor |
| `climate.set_hvac_mode("heat_cool")` | `switch.on` + `airConditionerMode.setAirConditionerMode("auto")` | ✅ | Auto mode |
| `climate.set_hvac_mode("off")` | `switch.off` | ✅ | Unit powers off |
| `climate.set_fan_mode("low")` | `airConditionerFanMode.setFanMode("low")` | ✅ | Fan speed audibly changes |
| `climate.set_fan_mode("medium")` | `airConditionerFanMode.setFanMode("medium")` | ✅ | |
| `climate.set_fan_mode("high")` | `airConditionerFanMode.setFanMode("high")` | ✅ | |
| `climate.set_fan_mode("auto")` | `airConditionerFanMode.setFanMode("auto")` | ✅ | |
| `climate.set_temperature` | `thermostatCoolingSetpoint.setCoolingSetpoint(16–30)` | Likely ✅ | Not fully verified across all modes |
| `climate.set_hvac_mode("heat")` | `airConditionerMode.setAirConditionerMode("heat")` | ❌ Not available | `heat` mode not in this unit's capability profile |

---

## Network architecture: why HA works, web UI does not

The core finding is that **two completely different backend paths exist** for SmartThings commands, and only one of them works for `pi=shp`.

```
                          Samsung Cloud
                               │
         ┌─────────────────────┼──────────────────────────────┐
         │                     │                              │
         ▼                     ▼                              ▼
  Developer REST API    Consumer Plugin Layer         Device telemetry
  (api.smartthings.com) (client.smartthings.com)      (broken for pi=shp)
   /v1/devices/{id}/     /graphql                     device → cloud push
   commands               (GraphQL)                   does NOT update
         │                     │                       Samsung's state
         │ Auth: OAuth2         │ Auth: Samsung SSO     (reads are stale)
         │ Bearer PAT           │ session cookie
         │                     │
         ▼                     ▼
  ✅ Commands reach      ❌ Commands accepted
  the physical unit        but never applied
  (pi=shp, 2026-05-28      (beep, no effect)
   verified)                (pi=shp, 5 rounds
                             of testing)
         │
         ▼
  Used by:
  HA SmartThings integration (pysmartthings library)
  Direct curl with Bearer PAT
```

The SmartThings web UI (`my.smartthings.com`) and the Samsung mobile app use the **consumer plugin layer** (GraphQL). This layer has a broken cloud→SHP translator for legacy units. The Developer REST API appears to route via a different channel that does reach the device.

---

## State sync issue & workaround

### The problem

After sending a command via HA (or curl), the **physical unit obeys** — but the Samsung cloud is not updated with the new state. The device→cloud telemetry push is broken for `pi=shp`. Consequences:

- HA's `climate` entity shows stale state (the state before your command).
- The `pysmartthings` SSE subscription may stop with `TransferEncodingError`.
- `GET /v1/devices/{id}/status` after a command still returns the pre-command state.

This is a Samsung cloud limitation for this device generation, not a Home Assistant bug.

### Watchdog workaround

Add this automation to HA to force a state poll every 5 minutes:

```yaml
alias: SmartThings watchdog reload
description: >
  Workaround for pi=shp state sync broken in Samsung cloud.
  Forces periodic re-read of SmartThings integration state.
trigger:
  - platform: time_pattern
    minutes: /5
action:
  - action: homeassistant.reload_config_entry
    data:
      entry_id: <your-smartthings-config-entry-id>
mode: single
```

To find your config entry ID: Settings → Devices & Services → SmartThings → the entry's ID is visible in the URL when you click into it (`/config/integrations/integration/smartthings#config_entry_id=<id>`), or you can find it via `GET /api/config/config_entries` in HA's REST API.

### Longer-term: local mTLS polling

The unit serves live status on port `:8888` at `<your-LAN-IP>:8888`. The community TLS client cert authenticates. This provides real-time state without going through Samsung's cloud. Full local polling (reads without the write token) has not been fully verified for this firmware variant — this is a promising avenue for future investigation.

---

## Evidence summary

| Evidence | Source | Date |
|---|---|---|
| Network capture of HA SmartThings commands (POST to `api.smartthings.com/v1/devices/{id}/commands`, Bearer auth, COMPLETED response) | Local packet capture during HA operation | 2026-05-28 |
| Physical verification: `set_hvac_mode(cool)` → compressor starts | Direct observation | 2026-05-28 |
| Physical verification: `set_hvac_mode(fan_only)` → fan runs, no compressor | Direct observation | 2026-05-28 |
| Physical verification: `set_fan_mode(high)` → fan audibly speeds up | Direct observation | 2026-05-28 |
| Physical verification: `set_hvac_mode(off)` → unit powers off | Direct observation | 2026-05-28 |
| Web UI (my.smartthings.com) commands confirmed non-working | Playwright browser automation + direct observation | 2026-05-27 (5 rounds) |
| `pysmartthings` SSE subscription `TransferEncodingError` | HA logs | 2026-05-27 |
| Watchdog automation active | HA automation config | 2026-05-28 |

---

## Community verification request

This result has not yet been independently verified by other `pi=shp` unit owners. If you own a Samsung AC with `pi=shp` (especially other model families — AR09, AR18, other year variants), please test the HA SmartThings integration and share your results in a GitHub Issue.

Key things to report:
- Your unit's `mnmo`, `vid`, `pi`
- Which `hvac_mode` and `fan_mode` values worked / did not
- Whether your unit also has stale state reads
- Any difference in behavior from what's documented here

This will help build a clearer picture of which `pi=shp` units benefit from this path.
