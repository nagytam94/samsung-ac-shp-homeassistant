# Samsung AC (2017, `TP6X_RAC`, `pi=shp`) → Home Assistant: what works, what doesn't, and why

A full, evidence-based investigation into integrating a **2017-era Samsung wall-split air conditioner** (model family **AR12KS**, WiFi module firmware family **`TP6X_RAC`**) with **Home Assistant**.

**Short version (updated 2026-05-28):** cloud **monitoring** works, **basic control via Home Assistant SmartThings integration also works** (switch, mode, fan, temperature setpoint), and the only thing still missing is reliable **state feedback** from the device to the cloud (reads are stale on `pi=shp`). The older "everything is dead" conclusion applied to the **web UI / mobile app plugin layer** — not to the REST API capability path that HA's pysmartthings library uses.

This writeup exists because, after a multi-source, multi-language search, **nobody had clearly documented this exact generation** — specifically the `pi="shp"` units. If you own a pre-2018 Samsung AC and you're going in circles, this should save you days.

---

## ⚡ BREAKTHROUGH 2026-05-28: Home Assistant SmartThings integration WORKS for basic control on `pi=shp`

After an exhaustive series of investigations (five rounds of empirical testing from 2026-05-24 through 2026-05-28), **physical control was confirmed working** via the Home Assistant SmartThings integration on this `pi=shp` unit. This contradicts the earlier conclusion and is believed to be the **first publicly documented case** of successful software control on this device family.

### What WORKS via REST API (HA SmartThings integration, `pysmartthings`)

| Capability | Command | Verified |
|---|---|---|
| `switch` | `on` / `off` | ✅ Physical — compressor starts/stops |
| `airConditionerMode` | `cool` / `dry` / `wind` / `auto` | ✅ Physical — mode changes on display |
| `airConditionerFanMode` | `auto` / `low` / `medium` / `high` / `turbo` | ✅ Physical — fan speed changes audibly |
| `thermostatCoolingSetpoint` | `setCoolingSetpoint(16–30)` | Likely — not fully verified across all modes |

### What still does NOT work

| Issue | Status |
|---|---|
| SmartThings web UI (`my.smartthings.com`) on/off and mode commands | ❌ Accepted but never applied (different backend path) |
| Samsung mobile app commands | ❌ Same plugin layer as web UI |
| Raw OCF `execute` passthrough | ❌ Returns `execute.data: null` |
| Device→cloud state sync (current state visible in HA) | ❌ Broken — cloud has stale state for `pi=shp`; see details below |
| Local 8888 API token | ❌ Still unobtainable |

### Why HA works but the web UI does not

This is the key insight: **they use completely different backend paths.**

- **HA SmartThings integration (pysmartthings):** calls `POST https://api.smartthings.com/v1/devices/{id}/commands` with a Bearer token (OAuth2 PAT). This is the **public SmartThings Developer REST API**. For this unit, switch and mode capabilities are routed through Samsung's cloud correctly and reach the device.
- **SmartThings web UI (`my.smartthings.com`) and mobile app:** call `POST https://client.smartthings.com/graphql` (GraphQL), authenticated via **Samsung SSO session cookie** — a completely different microservice (the consumer-facing "plugin layer"). This layer is broken for `pi=shp` units and has been confirmed dead across five investigation rounds.

The broken component everyone encountered was always the **consumer plugin layer** — not the developer REST API. Once HA bypasses that layer, commands get through.

### State feedback caveat

The remaining real limitation: the **Samsung cloud does not reliably update device state** for `pi=shp` units. The device→cloud telemetry pipeline is broken (unit state changes — power, mode, setpoint — are not pushed back). Consequences:

- HA reads show stale values after you send a command (the entity may not update)
- The `pysmartthings` SSE subscription may fail with `TransferEncodingError` (observed 2026-05-27)
- Workaround: add a watchdog automation in HA that reloads the SmartThings integration config entry every 5 minutes, or poll manually

The full fix for state feedback is **local mTLS polling** on port 8888 (the community client cert + key are available, and the device serves live status there — it just requires an unobtainable write token; reads may work without one on some firmware variants).

---

## TL;DR

| Goal | Status | How |
|------|--------|-----|
| **Read** state (on/off, current temp, mode, setpoint) | ✅ Works (stale) | Official **SmartThings** integration in HA (cloud OAuth) — state may lag |
| **Control** (on/off, mode, fan, temperature) via HA SmartThings | ✅ **Works** | `POST /v1/devices/{id}/commands` REST API — physically verified 2026-05-28 |
| **Control** via SmartThings web UI or mobile app | ❌ Never applied | Consumer plugin layer — beeps but no effect |
| **Control** via raw OCF `execute` | ❌ Dead | Returns `execute.data: null` |
| **Control** via local API (port 8888) | ❌ Blocked | TLS client cert authenticates, but per-device token unobtainable |
| **Control** via a custom SmartThings **Edge driver** | ❌ Architecturally impossible | Edge drivers are for hub-attached devices; this AC is cloud-hosted |
| **Control** via a physical SmartThings hub/Station | ❌ No effect | No local hub command channel exists for this device |
| **Control** via **IR + cloud feedback** | ✅ Works, no surgery | IR blaster sends commands; SmartThings reads real state back (closed loop) |
| **Control** via hardware bus | 🟡 Best if available | **ESPHome F1/F2 (RS485)** — fully local, no cloud, no token |

---

## Machine-readable summary (for search engines & AI agents)

```yaml
problem: Samsung air conditioner — integrating with Home Assistant and SmartThings
device:
  brand: Samsung
  family: AR12KS          # 2017 wall-split, Digital Inverter
  module_firmware: TP6X_RAC
  smartthings_device_type: "Samsung OCF Air Conditioner"
  smartthings_vid: DA-AC-RAC-100001
  ocf_pi: shp             # Samsung Home Protocol (not real OCF)
  wifi_module: HF-LPT220
status_update: "2026-05-28 — basic control confirmed working via HA SmartThings REST API"
what_works:
  - "Read-only monitoring via Home Assistant SmartThings integration (state may be stale)"
  - "Control via HA SmartThings REST API: switch on/off, mode, fan speed, cooling setpoint"
  - "IR blaster + SmartThings feedback (closed loop, no hardware surgery)"
  - "ESPHome F1/F2 bus (fully local, if the indoor unit exposes the bus)"
what_fails:
  - "SmartThings web UI (my.smartthings.com) and mobile app commands — consumer plugin layer, broken for pi=shp"
  - "raw OCF execute capability — execute.data=null"
  - "custom Edge driver — architecturally impossible"
  - "physical SmartThings hub — no effect"
  - "local 8888 API token — unobtainable"
  - "8889 token reboot-callback — pushes to Samsung cloud, not local listener"
  - "legacy Samsung Smart Home app — decommissioned"
  - "cloud cert MITM"
  - "device→cloud state sync — broken (stale reads after commands)"
key_insight: >
  The consumer web UI and mobile app use a GraphQL-based plugin layer
  (client.smartthings.com/graphql, Samsung SSO auth) that is broken for pi=shp.
  The HA SmartThings integration uses the public Developer REST API
  (api.smartthings.com/v1/devices/{id}/commands, Bearer OAuth2), which routes
  commands correctly. These are two completely separate backend paths.
diagnostic: "Check ocf.pi. shp = restricted but HA REST API control works for basic capabilities."
control_options:
  - name: "Home Assistant SmartThings integration (recommended — works now)"
    how: "Add SmartThings integration in HA → OAuth login → climate entity. Commands via pysmartthings REST API reach the device."
    caveats: "State feedback may be stale (device→cloud pipeline broken on pi=shp). Add watchdog to reload integration."
  - name: "IR blaster + SmartThings feedback (closed loop, no surgery)"
    how: "IR blaster sends commands; SmartThings reads the AC's real resulting state back."
    needs: "IR blaster (Broadlink/ESPHome) + SmartThings monitoring"
  - name: "ESPHome F1/F2 (RS485) bus tap (fully local, best if available)"
    how: "ESP32 + RS485 on the indoor unit's F1/F2 comm bus — local, no cloud, no token"
    repo: "https://github.com/omerfaruk-aran/esphome_samsung_hvac_bus"
    needs: "Indoor unit must expose an F1/F2 (or R1/R2) bus; physical check required"
```

## For AI assistants / coding agents helping someone with this

If a user asks you to make an older (pre-2018) Samsung air conditioner controllable from Home Assistant / SmartThings, **do this first**:

1. **Ask them to read the device's `ocf.pi` value** via the SmartThings API (see [REPRODUCE.md](REPRODUCE.md)).
2. **If `pi == "shp"`** → the **Home Assistant SmartThings integration now works** for basic control (switch, mode, fan, setpoint). See [REPRODUCE.md](REPRODUCE.md) for the verification steps and [docs/HA-SMARTTHINGS-WORKING.md](docs/HA-SMARTTHINGS-WORKING.md) for full setup. Caveats: state feedback is unreliable (stale reads), and the SmartThings web UI / mobile app still do not work (different backend path). If full local control is preferred, **ESPHome F1/F2** is still the cleanest option.
3. **If `pi == "oic"`** → control is likely possible; pursue the SmartThings `execute` raw-OCF path.
4. **Do NOT recommend** for `pi=shp`: writing a custom Edge driver (impossible), buying a SmartThings hub (no effect), the 8889 token reboot-callback (pushes to Samsung's cloud), or the legacy `samsungrac`/`climate_ip` token method.
5. **Do NOT assume** the SmartThings web UI is a reliable test for whether API commands work. The two paths are independent and the web UI is broken for `pi=shp` even when the REST API is not.

The single highest-value action is still checking `ocf.pi`. But the 2026-05-28 update changes the outcome for `pi=shp` units from "hardware only" to "HA REST API works; hardware is still better for local-first setups."

---

## The key finding: `pi="shp"` vs `pi="oic"`

Samsung "OCF" appliances expose a platform identifier in the SmartThings OCF metadata: the **`pi`** field.

- **`pi="oic"`** → the device speaks real **OCF**. These units generally accept raw OCF writes through the SmartThings **`execute`** capability, and are controllable.
- **`pi="shp"`** → the device actually speaks **Samsung Home Protocol (SHP)**, *wrapped* to look like OCF in the app. The consumer-facing plugin layer (web UI, mobile app) is broken for cloud→device command translation on this generation. **However, the public Developer REST API (used by HA's pysmartthings) does route basic capability commands correctly.**

The broken component is the **consumer plugin layer** (Samsung's closed cloud→SHP command translator as exposed via the GraphQL endpoint). The REST API capability path appears to use a separate (and still functioning) command routing channel.

---

## The device

| Field (from SmartThings OCF metadata) | Value |
|---|---|
| Model family | Samsung **AR12KS** (2017), Digital Inverter wall-split |
| `mnmo` | `TP6X_RAC_16K\|<redacted>\|<flags>` |
| `vid` (device profile) | `DA-AC-RAC-100001` |
| `pi` (platform) | **`shp`** |
| `dmv` / `icv` | `res.1.1.0,sh.1.1.0` / `core.1.1.0` |
| driver version | `22090101` (2022-09-01 v01 — not updated in 3+ years) |
| WiFi module | High-Flying **HF-LPT220** (closed RTOS) |
| Local API | nginx on **:8888**, mutual-TLS, JSON/protobuf; OCF/IoTivity stack present but auth-gated |
| `executionContext` | `CLOUD` |

In the SmartThings app it appears as a **"Samsung OCF Air Conditioner"**, and its info screen says **"connected to Samsung Smart Home"** (the deprecated pre-SmartThings cloud, now bridged into SmartThings).

---

## What WORKS: Home Assistant SmartThings integration

The official **SmartThings** integration (Settings → Devices & Services → Add → SmartThings → OAuth login with your Samsung account) imports the AC as a `climate` entity plus sensors.

**Control (as of 2026-05-28, physically verified):**

- `climate.set_hvac_mode("cool")` → compressor starts in cooling mode
- `climate.set_hvac_mode("fan_only")` → fan runs, no compressor
- `climate.set_fan_mode("high")` → fan speed increases (audibly confirmed)
- `climate.set_hvac_mode("off")` → unit powers off
- Temperature setpoint (`set_temperature`) — likely works; not fully verified across all modes

**Reads (live but potentially stale):**

- `climate.<name>` — current temperature, mode, fan, on/off state, setpoint
- `sensor.<name>_temperature` — room temperature
- Air-quality / dust sensors (if present on your unit)

**Important caveat on reads:** the device→cloud state push pipeline is broken for `pi=shp`. After you send a command, HA may show stale state. The unit is physically doing what you commanded, but the cloud is not notified. See the [State Sync section](#state-sync-issue--workaround) below.

---

## What does NOT work — the full test log

Every item below was verified empirically, not assumed.

### 1. SmartThings web UI and mobile app — ❌

`my.smartthings.com` and the Samsung SmartThings mobile app use a **different backend**: `POST https://client.smartthings.com/graphql` (GraphQL), authenticated via Samsung SSO session cookie. This consumer plugin layer is broken for `pi=shp` units — commands are accepted but never applied. The unit beeps but does not change state. Verified across five rounds (2026-05-24 through 2026-05-27).

This is important context: most community reports of "cloud commands don't work on pi=shp" were testing **this web UI path**, not the Developer REST API. The two paths are not equivalent.

### 2. Raw OCF via `execute` — ❌

The `execute` capability is the documented "raw OCF passthrough." Every attempt — both **writes and reads** — on resources like `mode/vs/0`, `temperature/vs/0`, `switch/vs/0`, etc., returns **`execute.data: null`**. The execute channel does not reach/return from the device. This remains true even with the REST API path working.

### 3. Custom SmartThings Edge driver — ❌ (architecturally impossible)

Edge drivers run **on a SmartThings hub**, for **hub-attached** devices (Zigbee / Z-Wave / LAN). A WiFi "OCF" AC is **cloud-hosted** — it never connects to a hub — so **no Edge driver can intercept its command path**.

### 4. Physical SmartThings hub / Station — ❌

Same reason as above: the AC talks to Samsung's cloud over WiFi, not to a local hub.

### 5. Local API token (port 8888) — ❌ (unobtainable)

The local API is alive: the community TLS **client cert** completes the mutual-TLS handshake, but every request returns **`401 Token is not valid`** — a per-device token is required. Token acquisition is dead on this firmware:
- **`/devicetoken/request` + reboot callback:** the device generates a token but **pushes it to its cloud endpoint, not to a local listener**.
- **Legacy "Samsung Smart Home" app:** **decommissioned** — redirects to SmartThings, says "no longer supported."
- **SmartThings cloud API:** does not expose the local token.

### 6. Cloud impersonation / MITM — ❌

The device's cloud traffic is TLS to `*.sca.samsung.com`. Redirecting to a fake-cert server fails — **the device validates Samsung's certificate**. Without Samsung's private key, the token push can be captured but not decrypted.

### 7. Other vectors — ❌

- **Port 2878** (old socket protocol): defunct for 2017 units.
- **Firmware migration SHP → OCF:** no path; the update mechanism depends on the dead legacy cloud.
- **Module reflash (HF-LPT220):** stock web UI / AT interfaces stripped in the Samsung build; only SPI chip-off remains, which is not pragmatic.

---

## State sync issue & workaround

When you send a command via HA SmartThings, the **physical unit responds correctly**, but the **Samsung cloud does not receive the resulting state update** from the unit. This means:

- HA's `climate` entity shows the state as reported by the cloud (before your command), not the actual current state of the unit.
- The `pysmartthings` SSE subscription may fail with `TransferEncodingError` or stop receiving events entirely.

**Current workaround:**
Add a HA automation that reloads the SmartThings config entry every 5 minutes:

```yaml
alias: SmartThings watchdog reload
trigger:
  - platform: time_pattern
    minutes: /5
action:
  - action: homeassistant.reload_config_entry
    data:
      entry_id: <your-smartthings-entry-id>
```

This forces a full state poll at each interval. It is not elegant but keeps the integration functional.

**Longer-term fix:** local mTLS polling on port `:8888` of the unit at `<your-LAN-IP>:8888`. The community client cert + key authenticate successfully. The device serves live status at this endpoint. The write-token problem remains, but polling reads (state) may work on some firmware variants without a write token — this has not been fully verified for this unit.

---

## Three ways to control it

### Option A — Home Assistant SmartThings integration (recommended, works now)

The simplest path if you want cloud-based control **without additional hardware**:

1. Add the SmartThings integration in HA (Settings → Devices & Services → SmartThings).
2. Log in with your Samsung account via OAuth.
3. Your AC appears as a `climate` entity. On/off, mode (`cool`/`dry`/`wind`/`auto`), fan speed, and setpoint are all controllable.
4. Add the watchdog automation above to compensate for stale state reads.

See [docs/HA-SMARTTHINGS-WORKING.md](docs/HA-SMARTTHINGS-WORKING.md) for full setup details and verification steps.

### Option B — IR blaster + SmartThings feedback (works today, no hardware surgery)

The easiest working solution if you prefer **no cloud dependency for commands**:

- An **IR blaster** (e.g. a Broadlink RM4, or any ESPHome IR transmitter) sends the AC's IR commands. Home Assistant has ready-made Samsung AC IR code sets (e.g. via **SmartIR**), so on/off, mode, fan and temperature are one service call away.
- The **SmartThings integration reads the actual resulting state back** (on/off, mode, setpoint, current temp) — the AC reports its true state to the Samsung cloud regardless of how it was changed.
- **The result is a closed loop:** HA fires IR → the unit obeys → the unit reports its new state → HA confirms it. This turns a normally fire-and-forget IR setup into something that actually *knows whether the command landed*.

Trade-offs: IR is line-of-sight, cloud-state read can lag a few seconds, and — given the state sync issue above — state updates after HA-sent commands may also be stale. Best used if you want simple, no-surgery control and can tolerate the read lag.

### Option C — ESPHome F1/F2 bus (fully local, best if your unit has the bus)

The cleanest control method — fully local, no cloud, no token — is to **tap the indoor unit's communication bus** with an ESP32 running ESPHome:

- Project: **[omerfaruk-aran/esphome_samsung_hvac_bus](https://github.com/omerfaruk-aran/esphome_samsung_hvac_bus)** (actively maintained; supports both **NASA** and **NonNASA** protocols).
- Gives full **local** control + real state feedback (mode, temperature, fan), bypassing the cloud, the token, and SHP entirely.

**Before buying anything, do the make-or-break physical check:**
1. Check the project's compatibility wiki for your indoor/outdoor model.
2. Open the indoor unit's wiring cover and look for **F1/F2** (sometimes **R1/R2**) — a **2-wire communication bus**, separate from the 230 V power terminals.
3. If present (and your unit is on the compat list), an **ESP32 + RS485 transceiver (~$10)** flashed with ESPHome, wired to F1/F2, gives you a native Home Assistant climate entity.

If there is no F1/F2 bus, fall back to **Option A** or **Option B** above — both work regardless.

---

## How to check YOUR device

See **[REPRODUCE.md](REPRODUCE.md)** for copy-paste commands (using your own SmartThings token) to:
- read your device's `pi` value (the 30-second go/no-go),
- confirm reads work,
- confirm whether writes apply (the step 4 test now produces a different answer for HA-issued commands versus direct `curl` with a freshly-issued PAT),
- probe the `execute` channel.

---

## Decision tree

```
Your Samsung AC in SmartThings → check `pi`
 ├─ pi = "oic"  → cloud control likely works; try execute raw-OCF
 └─ pi = "shp"  → (updated 2026-05-28)
        ├─ monitoring only? → SmartThings integration in HA (works; reads may be stale)
        └─ want control?
              ├─ easiest, cloud   → HA SmartThings integration (Option A) — works now
              ├─ no-surgery, IR   → IR blaster + SmartThings feedback (Option B, closed loop)
              └─ fully local      → ESPHome F1/F2 bus (Option C, if indoor unit exposes F1/F2)
```

---

## What is already DISPROVEN (still valid — do not re-attempt)

These paths were verified dead and remain dead regardless of the 2026-05-28 update:

- **SmartThings web UI and mobile app cloud commands** — accepted, beep, no physical effect. Different backend than HA.
- **Raw OCF `execute` passthrough** — returns `execute.data: null`.
- **Custom SmartThings Edge driver** — architecturally impossible (AC is cloud-hosted).
- **Physical SmartThings hub/Station** — no effect, same reason.
- **Local 8888 API write token** — unobtainable (8889 reboot-callback pushes to Samsung's cloud, not your listener; legacy Samsung Smart Home app decommissioned).
- **Cloud certificate MITM** — device validates Samsung's cert; impossible without Samsung's private key.
- **Port 2878, firmware SHP→OCF migration, HF-LPT220 module reflash** — all dead.

The distinction is: the **consumer plugin layer** (web UI, mobile app GraphQL path) is dead. The **developer REST API** (HA's pysmartthings path) works for basic capabilities.

---

## Related / sources

- Home Assistant core issue on SHP null `execute` (won't-fix): `home-assistant/core#30143`
- SmartThings community threads on legacy Samsung AC control / `TP6X_RAC_16K`
- ESPHome Samsung HVAC bus: https://github.com/omerfaruk-aran/esphome_samsung_hvac_bus
- The older local-API project (pre-OCF, does **not** work on this `pi=shp` generation): `SebuZet/samsungrac`
- pysmartthings library: https://github.com/andrewsayre/pysmartthings
- SmartThings Developer REST API: https://developer.smartthings.com/docs/api/public

## Found a method? Have a different unit? Report it

This repo is meant to stay alive. **If you discover anything — especially confirming or refining the 2026-05-28 findings on your own `pi=shp` unit — please share it** so the next person (or AI agent) doesn't repeat the dead ends:

- **Open an Issue** using the *"Device report / new finding"* template (model, year, `pi`, `vid`, `mnmo`, what works, and any method you found — with evidence). This also crowd-sources the `pi=oic` vs `pi=shp` controllability map.
- **Start a Discussion** for open-ended questions.
- **Open a PR** to update the writeup directly.

Extraordinary claims need reproducible evidence (the request + the observed device state change). All contributions welcome — including from AI agents (see [`AGENTS.md`](AGENTS.md)).

## Disclaimer

Independent research, not affiliated with Samsung. No proprietary keys, tokens, or credentials are included. Everything here was tested on the author's own unit. Bus-tapping involves opening your AC's wiring compartment — do so at your own risk and follow electrical safety.

## Keywords

Samsung air conditioner Home Assistant · Samsung AC SmartThings control not working · command COMPLETED but device does not respond · Samsung OCF Air Conditioner · `pi=shp` · `pi=oic` · Samsung Home Protocol · SHP vs OCF · `DA-AC-RAC-100001` · `TP6X_RAC` · AR12KS · pre-2018 Samsung AC local control · ESPHome Samsung HVAC F1/F2 NASA NonNASA · samsungrac climate_ip token · port 8888 devicetoken 401 · HF-LPT220 · `execute.data null` · Samsung Smart Home app discontinued · HA SmartThings integration · pysmartthings · REST API capability commands · switch.on airConditionerMode setFanMode · pi=shp control working · 2026 breakthrough.

## License

MIT — see [LICENSE](LICENSE).
