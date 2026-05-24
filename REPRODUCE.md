# Check your own Samsung AC (copy-paste)

All commands use the **SmartThings cloud API** with a Personal Access Token (PAT) you generate yourself. No special tools needed — just `curl` + `python3`.

> ⚠️ A PAT can read/control **all** your SmartThings devices. Treat it like a password, and **revoke it when done** at https://account.smartthings.com/tokens.

## 0. Get a token

1. Go to https://account.smartthings.com/tokens → **Generate new token**.
2. Scope it to **Devices** (list / read / execute).
3. Copy the token (shown once).

```bash
export TOK="<YOUR_TOKEN>"
```

## 1. Find your AC and its device id

```bash
curl -s -H "Authorization: Bearer $TOK" https://api.smartthings.com/v1/devices \
| python3 -c "import sys,json;[print(i['deviceId'],'|',i.get('label'),'|',i.get('deviceTypeName') or i.get('type')) for i in json.load(sys.stdin)['items']]"
export DEV="<DEVICE_ID_OF_THE_AC>"
```

## 2. The 30-second go/no-go: read the `pi` value

```bash
curl -s -H "Authorization: Bearer $TOK" https://api.smartthings.com/v1/devices/$DEV/status \
| python3 -c "import sys,json;print('pi =', json.load(sys.stdin)['components']['main']['ocf']['pi']['value'])"
```

- `pi = oic` → real OCF; cloud control likely works, keep going (try step 5).
- `pi = shp` → Samsung Home Protocol; **cloud/local software control is dead** (this repo). Use the SmartThings integration for monitoring, and look at the ESPHome F1/F2 hardware path.

## 3. Confirm reads work

```bash
curl -s -H "Authorization: Bearer $TOK" https://api.smartthings.com/v1/devices/$DEV/status \
| python3 -c "
import sys,json;m=json.load(sys.stdin)['components']['main']
g=lambda c,a: m.get(c,{}).get(a,{}).get('value')
print('switch        :',g('switch','switch'))
print('mode          :',g('airConditionerMode','airConditionerMode'))
print('fan           :',g('airConditionerFanMode','fanMode'))
print('current temp  :',g('temperatureMeasurement','temperature'))
print('setpoint      :',g('thermostatCoolingSetpoint','coolingSetpoint'))
"
```

## 4. Test whether a WRITE actually applies (the real test)

Sends a reversible fan-mode change, then reads it back. If the value does **not** change (but the API said `COMPLETED` and the unit beeps), your write path is broken — the `pi=shp` symptom.

```bash
curl -s -X POST -H "Authorization: Bearer $TOK" -H "Content-Type: application/json" \
  -d '{"commands":[{"component":"main","capability":"airConditionerFanMode","command":"setFanMode","arguments":["high"]}]}' \
  https://api.smartthings.com/v1/devices/$DEV/commands
sleep 8
curl -s -H "Authorization: Bearer $TOK" https://api.smartthings.com/v1/devices/$DEV/status \
| python3 -c "import sys,json;print('fan now =', json.load(sys.stdin)['components']['main']['airConditionerFanMode']['fanMode']['value'])"
```

## 5. Probe the raw OCF `execute` channel

On `pi=oic` units this returns OCF data in `execute.data`. On `pi=shp` units it returns `null` (channel dead).

```bash
curl -s -X POST -H "Authorization: Bearer $TOK" -H "Content-Type: application/json" \
  -d '{"commands":[{"component":"main","capability":"execute","command":"execute","arguments":["mode/vs/0"]}]}' \
  https://api.smartthings.com/v1/devices/$DEV/commands
sleep 6
curl -s -H "Authorization: Bearer $TOK" https://api.smartthings.com/v1/devices/$DEV/status \
| python3 -c "import sys,json;print('execute.data =', json.load(sys.stdin)['components']['main'].get('execute',{}).get('data',{}).get('value'))"
```

## 6. Clean up

Revoke the PAT: https://account.smartthings.com/tokens

---

### Interpreting results

| `pi` | write applies (step 4) | `execute.data` (step 5) | Verdict |
|------|------------------------|--------------------------|---------|
| `oic` | yes | OCF payload | controllable via cloud |
| `shp` | no (beeps, no change) | `null` | **software control dead → hardware (ESPHome F1/F2)** |

Please share your results in an issue to grow the compatibility map.
