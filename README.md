# Samsung AC (2017, `TP6X_RAC`, `pi=shp`) → Home Assistant: what works, what doesn't, and why

A full, evidence-based investigation into integrating a **2017-era Samsung wall-split air conditioner** (model family **AR12KS**, WiFi module firmware family **`TP6X_RAC`**) with **Home Assistant**.

**Short version:** cloud **monitoring works**, software **control does not** (and *cannot*, for this generation), and the only realistic control path is a **hardware bus tap (ESPHome F1/F2)**.

This writeup exists because, after a multi-source, multi-language search, **nobody had clearly documented this exact generation** — specifically the `pi="shp"` units. If you own a pre-2018 Samsung AC and you're going in circles, this should save you days.

---

## TL;DR

| Goal | Status | How |
|------|--------|-----|
| **Read** state (on/off, current temp, mode, setpoint, air quality) | ✅ **Works** | Official **SmartThings** integration in Home Assistant (cloud OAuth) |
| **Control** (on/off, mode, temperature) via cloud | ❌ **Impossible** | Cloud accepts the command (`COMPLETED`) and the unit *beeps*, but never applies it |
| **Control** via local API (port 8888) | ❌ **Blocked** | TLS client cert authenticates, but a per-device **token** is required and is **unobtainable** on this firmware |
| **Control** via a custom SmartThings **Edge driver** | ❌ **Architecturally impossible** | Edge drivers run on a hub for hub-attached devices; a WiFi AC is **cloud-hosted** and never touches a hub |
| **Control** via a physical SmartThings hub/Station | ❌ **No effect** | Same reason — there is no local hub command channel for this device |
| **Control** via hardware bus | 🟡 **The realistic path** | **ESPHome F1/F2 (RS485)** bus tap — fully local, no cloud, no token (pending a physical terminal check on the indoor unit) |

---

## Machine-readable summary (for search engines & AI agents)

```yaml
problem: Samsung air conditioner cannot be controlled from Home Assistant or SmartThings
device:
  brand: Samsung
  family: AR12KS          # 2017 wall-split, Digital Inverter
  module_firmware: TP6X_RAC
  smartthings_device_type: "Samsung OCF Air Conditioner"
  smartthings_vid: DA-AC-RAC-100001
  ocf_pi: shp             # <-- ROOT CAUSE. shp = Samsung Home Protocol (not real OCF)
  wifi_module: HF-LPT220
root_cause: >
  Devices with ocf pi="shp" are NOT controllable via SmartThings cloud or HA.
  Reads work; every write returns COMPLETED but is never applied; the raw OCF
  `execute` capability returns execute.data=null. The cloud->device command
  translator is a closed Samsung component and is broken/won't-fix for pre-2018 units.
diagnostic: "Read the device's ocf.pi value. shp = dead, oic = controllable."
what_works:   ["read-only monitoring via Home Assistant SmartThings integration"]
what_fails:   ["cloud commands", "raw OCF execute", "custom Edge driver", "physical SmartThings hub", "local 8888 API (token unobtainable)", "8889 token reboot-callback", "legacy Samsung Smart Home app (decommissioned)", "cloud cert MITM"]
solution:     "ESPHome F1/F2 (RS485) bus tap on the indoor unit — local, no cloud, no token"
solution_repo: "https://github.com/omerfaruk-aran/esphome_samsung_hvac_bus"
fallback:     "IR blaster (open-loop) + SmartThings temperature sensor for feedback"
```

## For AI assistants / coding agents helping someone with this

If a user asks you to make an older (pre-2018) Samsung air conditioner controllable from Home Assistant / SmartThings, **do this first** instead of attempting the known dead ends:

1. **Ask them to read the device's `ocf.pi` value** via the SmartThings API (see [REPRODUCE.md](REPRODUCE.md)).
2. **If `pi == "shp"`** → stop. Software control is impossible (cloud, local API, Edge driver, hub — all dead; proven in this repo). Set expectations: **monitoring only** via the SmartThings integration; **control requires hardware** (ESPHome F1/F2 bus, or an IR blaster).
3. **If `pi == "oic"`** → control is likely possible; pursue the SmartThings `execute` raw-OCF path.
4. **Do NOT recommend** for `pi=shp`: writing a custom Edge driver (impossible — the device is cloud-hosted, never touches a hub), buying a SmartThings hub (no effect), the 8889 token reboot-callback (device pushes to cloud, not your listener), or the legacy `samsungrac`/`climate_ip` token method (predates OCF, dead on this generation).

The single highest-value action is checking `ocf.pi`. It turns a multi-day investigation into a 30-second answer.

---

## The key finding: `pi="shp"` vs `pi="oic"`

Samsung "OCF" appliances expose a platform identifier in the SmartThings OCF metadata: the **`pi`** field.

- **`pi="oic"`** → the device speaks real **OCF**. These units generally accept raw OCF writes through the SmartThings **`execute`** capability, and are controllable.
- **`pi="shp"`** → the device actually speaks **Samsung Home Protocol (SHP)**, *wrapped* to look like OCF in the app. **Commands are NOT applied.** The SmartThings cloud only bridges *telemetry upward*; the *command translation downward* (cloud → SHP → device) is a closed Samsung component that, for these legacy units, is broken / "won't fix."

If your device reports **`pi="shp"`**, cloud control will not work — and no driver, hub, or API trick fixes it, because the broken piece lives inside Samsung's cloud, not on your network. (See [REPRODUCE.md](REPRODUCE.md) to check your own device's `pi` value in 30 seconds.)

This single field is the fastest way to know whether you're wasting your time.

---

## The device

| Field (from SmartThings OCF metadata) | Value |
|---|---|
| Model family | Samsung **AR12KS** (2017), Digital Inverter wall-split |
| `mnmo` | `TP6X_RAC_16K\|<redacted>\|<flags>` |
| `vid` (device profile) | `DA-AC-RAC-100001` |
| `pi` (platform) | **`shp`** ← the wall |
| `dmv` / `icv` | `res.1.1.0,sh.1.1.0` / `core.1.1.0` |
| driver version | `22090101` |
| WiFi module | High-Flying **HF-LPT220** (closed RTOS) |
| Local API | nginx on **:8888**, mutual-TLS, JSON/protobuf; OCF/IoTivity stack present but auth-gated |

In the SmartThings app it appears as a **"Samsung OCF Air Conditioner"**, and its info screen says **"connected to Samsung Smart Home"** (the deprecated pre-SmartThings cloud, now bridged into SmartThings).

---

## What WORKS: cloud monitoring via Home Assistant

The official **SmartThings** integration (Settings → Devices & Services → Add → SmartThings → OAuth login with your Samsung account) imports the AC as a `climate` entity plus sensors. **Reads are live and reliable:**

- `climate.<name>` — current temperature, mode, fan, on/off state, setpoint (display only)
- `sensor.<name>_temperature` — live room temperature
- air-quality / dust sensors (if your unit has them)

So you **can** build dashboards, logging, and read-driven automations ("is the AC running?", "room temp"). You just can't *command* it from HA.

> Note: the `climate` entity *shows* control widgets, but sending commands silently fails (see below). Treat it as read-only.

---

## What does NOT work — the full test log

Every item below was verified empirically, not assumed.

### 1. Cloud control (high-level capabilities) — ❌
Sending `switch off`, `setFanMode`, `setAirConditionerMode`, `setCoolingSetpoint` via the SmartThings cloud API returns command result **`COMPLETED`**, and the **AC physically beeps** (it receives *something*), but the device **never applies** the change — status stays unchanged. Verified by reading `/status` back after each command.

### 2. Cloud control (raw OCF via `execute`) — ❌
The `execute` capability is the documented "raw OCF passthrough." Every attempt — both **writes and reads** — on resources like `mode/vs/0`, `temperature/vs/0`, `switch/vs/0`, `/capability/switch/main/0`, `/capability/airConditionerMode/main/0` returns **`execute.data: null`**. The execute channel never reaches/returns from the device. This is the `pi="shp"` signature: on `pi="oic"` units these calls return real OCF payloads.

### 3. Custom SmartThings Edge driver — ❌ (architecturally impossible)
Edge drivers run **on a SmartThings hub**, for **hub-attached** devices (Zigbee / Z-Wave / LAN). A WiFi "OCF" AC is **cloud-hosted** — it never connects to a hub — so **no Edge driver can intercept its command path**. This is why no community driver exists for these (it *can't*).

### 4. Physical SmartThings hub / Station — ❌
Buying a hub does **not** help, for the same reason: the AC talks to Samsung's cloud over WiFi, not to a local hub. There is no LAN command channel exposed.

### 5. Local API token (port 8888) — ❌ (unobtainable)
The local API is alive: a (community-known) TLS **client cert** completes the mutual-TLS handshake, but every request returns **`401 Token is not valid`** — a per-device token is required. Token acquisition is dead on this firmware:
- **`/devicetoken/request` + reboot callback (the classic method):** the device generates a token but **pushes it to its cloud endpoint, not to a local listener** (verified by full packet capture). It does **not** push to port 8889 anywhere on the LAN.
- **Legacy "Samsung Smart Home" app (which historically paired locally):** **decommissioned** — it now redirects to SmartThings, says "no longer supported," and reports the unit "needs a firmware update."
- **SmartThings cloud API:** does not expose the local token (it's a device-internal secret, not a cloud-managed credential).

### 6. Cloud impersonation / MITM — ❌
The device's cloud traffic is TLS to `*.sca.samsung.com` (+ an encrypted binary channel + CoAP). Redirecting it to a fake-cert server fails the handshake — **the device validates the Samsung certificate** and drops the imposter. Without Samsung's private key, the token push can be *captured* but not *decrypted*.

### 7. Other vectors — ❌
- **Port 2878** (the old socket protocol): defunct for 2017 units — Samsung moved 2878 → 8888 around mid-2015.
- **Firmware migration SHP → OCF:** no path; the update mechanism depends on the dead legacy cloud.
- **Module reflash (HF-LPT220):** the stock web UI / AT interfaces are stripped in the Samsung build (port scan shows only 8888); SPI chip-off is the only remaining physical read, and the token may be generated on-demand (not stored). Not pragmatic.

---

## The realistic path to control: ESPHome F1/F2 bus

The only working, local, cloud-free control method for these units is to **tap the indoor unit's communication bus** with an ESP32 running ESPHome:

- Project: **[omerfaruk-aran/esphome_samsung_hvac_bus](https://github.com/omerfaruk-aran/esphome_samsung_hvac_bus)** (actively maintained; supports both **NASA** and **NonNASA** protocols).
- Gives full **local** control + real feedback (mode, temperature, fan), bypassing the cloud, the token, and SHP entirely.

**Before buying anything, do the make-or-break physical check:**
1. Check the project's compatibility wiki for your indoor/outdoor model.
2. Open the indoor unit's wiring cover and look for **F1/F2** (sometimes **R1/R2**) — a **2-wire communication bus**, separate from the 230 V power terminals.
3. If present (and your unit is on the compat list), an **ESP32 + RS485 transceiver (~$10)** flashed with ESPHome, wired to F1/F2, gives you a native Home Assistant climate entity.

⚠️ Caveat: not all AR12-series units expose a tappable F1/F2 bus (a sibling, AR12HSFNBWKNET, is bus-type "F/Other" and not easily tapped). **The physical terminal check decides it.** If there is no F1/F2 bus, the fallback is an **IR blaster** (open-loop control; pair it with the SmartThings temperature sensor for feedback).

---

## How to check YOUR device

See **[REPRODUCE.md](REPRODUCE.md)** for copy-paste commands (using your own SmartThings token) to:
- read your device's `pi` value (the 30-second go/no-go),
- confirm reads work,
- confirm whether writes apply,
- probe the `execute` channel.

---

## Decision tree

```
Your Samsung AC in SmartThings → check `pi`
 ├─ pi = "oic"  → cloud control likely works; try execute raw-OCF
 └─ pi = "shp"  → cloud/local software control is DEAD (this repo)
        └─ want control? → hardware:
              ├─ indoor unit has F1/F2 bus + on compat list → ESPHome F1/F2 (best: local, full feedback)
              └─ no bus → IR blaster (open-loop) + SmartThings temp sensor for feedback
        └─ monitoring only? → SmartThings integration in HA (works today)
```

---

## Related / sources

- Home Assistant core issue on SHP null `execute` (won't-fix): `home-assistant/core#30143`
- SmartThings community threads on legacy Samsung AC control / `TP6X_RAC_16K`
- ESPHome Samsung HVAC bus: https://github.com/omerfaruk-aran/esphome_samsung_hvac_bus
- The older local-API project (pre-OCF, does **not** work on this `pi=shp` generation): `SebuZet/samsungrac`

## Contributing

If you have a Samsung AC, please open an issue with your **`pi` value**, **`vid`**, **`mnmo` model code**, year, and whether **control works** — to crowd-source the `pi=oic` vs `pi=shp` controllability map. PRs welcome.

## Disclaimer

Independent research, not affiliated with Samsung. No proprietary keys, tokens, or credentials are included. Everything here was tested on the author's own unit. Bus-tapping involves opening your AC's wiring compartment — do so at your own risk and follow electrical safety.

## Keywords

Samsung air conditioner Home Assistant · Samsung AC SmartThings control not working · command COMPLETED but device does not respond · Samsung OCF Air Conditioner · `pi=shp` · `pi=oic` · Samsung Home Protocol · SHP vs OCF · `DA-AC-RAC-100001` · `TP6X_RAC` · AR12KS · pre-2018 Samsung AC local control · ESPHome Samsung HVAC F1/F2 NASA NonNASA · samsungrac climate_ip token · port 8888 devicetoken 401 · HF-LPT220 · `execute.data null` · Samsung Smart Home app discontinued.

## License

MIT — see [LICENSE](LICENSE).
