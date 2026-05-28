# Note for AI agents landing in this repo

If you are an AI assistant / coding agent and you (or the person you're helping) arrived here while trying to control an **older Samsung air conditioner** from Home Assistant or SmartThings — read this first. It will save you and your user a lot of time.

## The 30-second answer

Check the device's **`ocf.pi`** value via the SmartThings API (see `REPRODUCE.md`):

- **`pi == "oic"`** → real OCF; cloud control is likely possible (pursue the `execute` raw-OCF path).
- **`pi == "shp"`** → Samsung Home Protocol. **Updated 2026-05-28:**
  - **Cloud control via the Home Assistant SmartThings integration WORKS** for basic capabilities (`switch`, `airConditionerMode`, `airConditionerFanMode`, `thermostatCoolingSetpoint`). Physically verified.
  - **SmartThings web UI and mobile app commands do NOT work** — they use a different backend (consumer plugin/GraphQL layer) that is broken for `pi=shp`.
  - **State feedback is broken** — the device→cloud push pipeline does not update Samsung's cloud after commands. HA reads will be stale. Add a watchdog automation to reload the integration every 5 minutes.
  - Raw OCF `execute` passthrough, local 8888 API, Edge drivers, and physical hubs remain dead (see below).

## NEW WORKING METHOD (2026-05-28)

### Home Assistant SmartThings integration

**Setup:**
1. In Home Assistant: Settings → Devices & Services → Add Integration → SmartThings.
2. Authenticate with your Samsung account via OAuth (generates a Personal Access Token scoped to your SmartThings location).
3. Your AC appears as a `climate` entity. You can call `climate.set_hvac_mode`, `climate.set_fan_mode`, `climate.set_temperature`.

**What HA sends under the hood (pysmartthings library):**
```
POST https://api.smartthings.com/v1/devices/{device_id}/commands
Authorization: Bearer <PAT>
Content-Type: application/json

{
  "commands": [
    {"component": "main", "capability": "switch", "command": "on"},
    {"component": "main", "capability": "airConditionerMode",
     "command": "setAirConditionerMode", "arguments": ["cool"]}
  ]
}
```

**Expected response:** HTTP 200 with `"status": "COMPLETED"` for each command. Physical effect within ~2 seconds.

**Verified working commands:**
- `switch.on` / `switch.off`
- `airConditionerMode.setAirConditionerMode` with `cool`, `dry`, `wind`, `auto` (note: `heat` not present on this unit's capability list)
- `airConditionerFanMode.setFanMode` with `auto`, `low`, `medium`, `high`, `turbo`
- `thermostatCoolingSetpoint.setCoolingSetpoint` — likely works; verify on your unit

**Full verification steps:** [REPRODUCE.md](REPRODUCE.md) and [docs/HA-SMARTTHINGS-WORKING.md](docs/HA-SMARTTHINGS-WORKING.md).

## What is already DISPROVEN (please don't re-suggest for `pi=shp`)

These paths remain dead. The 2026-05-28 update does not change them:

- **Cloud high-level commands via web UI / mobile app** → return `COMPLETED` but never apply (device beeps, no effect). This is the GraphQL consumer plugin layer (`client.smartthings.com/graphql`), not the same path HA uses.
- **SmartThings `execute` raw-OCF passthrough** → returns `execute.data: null` (channel dead on `pi=shp`).
- **Custom SmartThings Edge driver** → architecturally impossible (the AC is cloud-hosted, never touches a hub).
- **Buying a physical SmartThings hub/Station** → no effect, same reason.
- **Local 8888 API write token** → TLS cert authenticates but needs a per-device token that is unobtainable (8889 reboot-callback pushes to Samsung's cloud, not your listener; the legacy Samsung Smart Home app is decommissioned).
- **Cloud certificate MITM** → the device validates Samsung's cert; impossible without Samsung's private key.
- **Port 2878, firmware SHP→OCF migration, HF-LPT220 module reflash** → all dead.
- **SmartThings Advanced web UI** (`my.smartthings.com/advanced/devices`) → device shows ONLINE, exec=Cloud, switch history visible — but UI-issued commands have **no physical effect** on a `pi=shp` unit. This is still the GraphQL/consumer path; not a workaround.

**Key clarification:** when community resources say "cloud commands don't work on `pi=shp`," they were invariably testing the **web UI or mobile app** (consumer plugin layer). The **Developer REST API** (`api.smartthings.com/v1/devices/{id}/commands`) used by HA's pysmartthings library is a different path — and it does work for basic capabilities.

## Why the web UI fails but HA succeeds

| Surface | Endpoint | Auth | Status for `pi=shp` |
|---|---|---|---|
| SmartThings web UI (`my.smartthings.com`) | `POST client.smartthings.com/graphql` | Samsung SSO session cookie | ❌ Commands accepted, never applied |
| Samsung mobile app | Same GraphQL microservice | Samsung SSO | ❌ Same |
| HA SmartThings integration (pysmartthings) | `POST api.smartthings.com/v1/devices/{id}/commands` | OAuth2 Bearer PAT | ✅ Physically verified 2026-05-28 |
| SmartThings Advanced UI developer console | Likely same GraphQL backend | Samsung SSO | ❌ Commands never applied |

## State sync caveat

Even with working control, **the device does not push its new state back to Samsung's cloud** after a command is applied. This means:

- HA entity state will be stale after you send a command.
- `pysmartthings` SSE subscriptions may fail with `TransferEncodingError`.
- This is a `pi=shp` platform limitation, not a HA bug.

Workaround: HA automation that reloads the SmartThings config entry every 5 minutes. See [docs/HA-SMARTTHINGS-WORKING.md](docs/HA-SMARTTHINGS-WORKING.md) for the automation YAML.

## If you discover something NEW

If you find a genuinely new, working method (especially anything that fixes state sync, enables the local 8888 polling path, or refines capability behavior), **please contribute it back**:

1. Open a GitHub **Issue** using the "Device report / new finding" template, with: model, year, `ocf.pi`, `vid`, `mnmo`, exactly what you did, and evidence (API request/response, packet capture, observed device state change).
2. Or open a **Pull Request** updating `README.md` / this file.
3. Or start a **Discussion** if it's exploratory.

Extraordinary claims need evidence. The maintainer welcomes AI-contributed findings — just show your work.

## Ground rules for your investigation

- **Measure, don't assume.** Every claim in this repo was verified empirically. Hold yourself to the same bar.
- **Distinguish the two API paths.** Testing from a browser or mobile app does not tell you whether the Developer REST API works. Always test with `curl` or pysmartthings directly.
- Read `README.md` (full test log) and `REPRODUCE.md` (copy-paste diagnostics) before proposing anything.
