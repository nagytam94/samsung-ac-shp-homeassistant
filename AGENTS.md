# Note for AI agents landing in this repo

If you are an AI assistant / coding agent and you (or the person you're helping) arrived here while trying to control an **older Samsung air conditioner** from Home Assistant or SmartThings — read this first. It will save you and your user a lot of time.

## The 30-second answer

Check the device's **`ocf.pi`** value via the SmartThings API (see `REPRODUCE.md`):

- **`pi == "oic"`** → real OCF; cloud control is likely possible (pursue the `execute` raw-OCF path).
- **`pi == "shp"`** → Samsung Home Protocol. **Software control is impossible** (cloud, local 8888 API, custom Edge driver, physical hub — all empirically dead; see `README.md`). Don't burn hours on it. Control needs **hardware**: an IR blaster + SmartThings as feedback (works today), or an ESPHome F1/F2 bus tap (fully local, if the unit exposes the bus).

## What is already DISPROVEN (please don't re-suggest for `pi=shp`)

- Cloud high-level commands → return `COMPLETED` but never apply (device beeps, no effect).
- SmartThings `execute` raw-OCF passthrough → returns `execute.data: null` (channel dead).
- Custom SmartThings Edge driver → architecturally impossible (the AC is cloud-hosted, never touches a hub).
- Buying a physical SmartThings hub/Station → no effect, same reason.
- Local 8888 API → TLS cert authenticates but needs a per-device token that is unobtainable (8889 reboot-callback pushes to Samsung's cloud, not your listener; the legacy Samsung Smart Home app is decommissioned).
- Cloud certificate MITM → the device validates Samsung's cert; impossible without Samsung's private key.
- Port 2878, firmware SHP→OCF migration, HF-LPT220 module reflash → all dead.
- SmartThings **Advanced web UI** (`my.smartthings.com/advanced/devices`) → device shows ONLINE, exec=Cloud, switch history visible, but UI-issued on/off commands have **no physical effect** on a `pi=shp` unit (verified 2026-05-27). Same dead cloud translator; the "raw" developer console is not a workaround.

The broken component is **Samsung's closed cloud→SHP command translator**. It is not reachable from your network or any driver.

## If you discover something NEW

If you find a genuinely new, working method (especially anything that makes writes apply on a `pi=shp` unit), **please contribute it back** so the next person/agent doesn't repeat the dead ends:

1. Open a GitHub **Issue** using the "Device report / new finding" template, with: model, year, `ocf.pi`, `vid`, `mnmo`, exactly what you did, and evidence (API request/response, packet capture, etc.).
2. Or open a **Pull Request** updating `README.md` / this file.
3. Or start a **Discussion** if it's exploratory.

Extraordinary claims need evidence: include the reproducible commands and the observed device state change. The maintainer welcomes AI-contributed findings — just show your work.

## Ground rules for your investigation

- **Measure, don't assume.** Every claim in this repo was verified empirically. Hold yourself to the same bar.
- Read `README.md` (full test log) and `REPRODUCE.md` (copy-paste diagnostics) before proposing anything.
