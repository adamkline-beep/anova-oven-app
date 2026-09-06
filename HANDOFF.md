# Anova oven app — handoff notes

Everything a fresh Claude session needs to keep working on this app. Upload this
file plus `index.html` at the start of a new chat and say "here's my Anova oven
app, I want to change X."

Written 2026-09-06. Owner runs an **Anova Precision Oven 1.0, 120V, firmware 2.1.16**.

---

## What this is

A single-file web app (`index.html`) that installs to an Android home screen as a
PWA and controls the oven through Anova's official developer API. Deployed on
Netlify. No backend, no Home Assistant. Token and recipes live in the phone's
`localStorage`.

Supporting files: `manifest.webmanifest`, `sw.js` (network-first shell cache),
`icon-192.png`, `icon-512.png`, `icon-maskable.png`, `README.md`.

**Status: working on real hardware.** Connects, reads live state, starts and stops
cooks. Confirmed 2026-09-05/06.

---

## The API

Anova launched an official developer API in July 2025. Docs at
`developer.anovaculinary.com`. It is **cloud-mediated, not local** — the phone
talks to Anova's servers, not to the oven on the LAN.

**Auth:** a personal access token generated in the Anova Oven app under
More → Developer → Personal Access Tokens. Starts with `anova-`. Up to 10 tokens.

**Connect:**
```
wss://devices.anovaculinary.io?token=<TOKEN>&supportedAccessories=APO
```

**Confirmed:** the server does **not** check the `Origin` header, so a plain web
page can open this socket. This was the main unknown at the start; it is resolved.

**Constraints:** connections drop after ~30 min idle; rate-limited for personal
use with disconnection as enforcement; personal access tokens are restricted to
`CMD_APO_START`, `CMD_APO_STOP`, `CMD_APO_SET_TEMPERATURE_UNIT`.

### Inbound messages

- `EVENT_APO_WIFI_LIST` — `payload` is an array of `{cookerId, name, type}`.
  `type` is `oven_v1` or `oven_v2`.
- `EVENT_APO_STATE` — see the real capture below.
- `ERROR` — `payload.errorMessage`.

### v1 state shape (verified against this owner's oven)

v1 nests the real state under `payload.state`; v2 keeps it flat. The app's
`normalize()` handles both by checking whether `payload.state.nodes` exists.

```
payload.cookerId
payload.type                    "oven_v1"
payload.state.version           1
payload.state.systemInfo        {online, hardwareVersion:"120V1", powerMains:120,
                                 firmwareVersion:"2.1.16", triacsFailed, ...}
payload.state.state.mode        "idle" | "cook" | ...
payload.state.state.temperatureUnit  "F"
payload.state.state.processedCommandIds  [last ~10 command ids the oven acted on]
payload.state.nodes.temperatureBulbs
      .mode                     "dry" | "wet"
      .dry / .wet / .dryTop / .dryBottom
          .current   {celsius, fahrenheit}
          .setpoint  {celsius, fahrenheit}     (dry only; STALE when idle)
          .overheated / .dosed / .doseFailed
payload.state.nodes.timer       {mode:"idle"|"running", initial, current, startType}
payload.state.nodes.temperatureProbe  {connected, current, setpoint}
payload.state.nodes.steamGenerators
      .mode                     "idle" | "relative-humidity" | "steam-percentage"
      .relativeHumidity         {current, setpoint}   <-- current reports even when idle
      .steamPercentage          {current, setpoint}
      .evaporator / .boiler     {celsius, watts, failed, overheated, descaleRequired, dosed}
payload.state.nodes.heatingElements  {top,bottom,rear}: {on, failed, watts}
payload.state.nodes.fan         {speed, failed}
payload.state.nodes.vent        {open}
payload.state.nodes.waterTank   {empty}
payload.state.nodes.door        {closed}
payload.state.nodes.lamp        {on, failed, preference}
payload.state.nodes.userInterfaceCircuit  {communicationFailed}
payload.state.cook              present only during a cook (see below)
```

**Gotchas found in the real data:**
- `dry.setpoint` and `heatingElements.rear.on` retain stale values from the last
  cook while the oven is idle. Never render a setpoint unless a cook is running.
- `relativeHumidity.current` is live even when `steamGenerators.mode` is `"idle"`.
- `uiFirmwareVersion` is `"0.0.0"` on this oven. Normal; not a fault.

### Active cook (`payload.state.cook`) — from SDK docs, NOT yet verified on this oven

```
{cookId, originSource, type, stages:[...], activeStageId, activeStageIndex,
 activeStageSecondsElapsed, secondsElapsed, stageTransitionPendingUserAction}
```
`stages` is the full stable plan and does not shrink; `activeStageIndex` is the
authoritative pointer. **A mid-cook capture is still the top outstanding item.**

### Outbound: `CMD_APO_START`

```json
{"command":"CMD_APO_START","requestId":"<uuid>",
 "payload":{"id":"<cookerId>","type":"CMD_APO_START",
   "payload":{"cookId":"<uuid>","cookerId":"<cookerId>","cookableId":"",
     "title":"","type":"oven_v1","originSource":"api","cookableType":"manual",
     "stages":[ ...preheat, cook, preheat, cook... ]}}}
```

Each user-facing stage compiles to **two** API stages: a `preheat` then a `cook`
with identical settings. Multi-stage recipes repeat that pair. Stage fields:

```
stepType:"stage", id:<uuid>, title:"", description:"", type:"preheat"|"cook",
userActionRequired:bool,
temperatureBulbs:{mode:"dry", dry:{setpoint:{celsius,fahrenheit}}}
                 or {mode:"wet", wet:{setpoint:{...}}},
heatingElements:{top:{on},bottom:{on},rear:{on}},
fan:{speed 0-100}, vent:{open}, rackPosition:1-3,
stageTransitionType:"automatic"|"manual",
steamGenerators:{mode:"relative-humidity", relativeHumidity:{setpoint}}
             or {mode:"steam-percentage", steamPercentage:{setpoint}},
timer:{initial:<seconds>, startType:"when-preheated"|"manual"}   (cook stage only)
timerAdded / probeAdded / timerStartOnDetect
temperatureProbe:{setpoint:{celsius,fahrenheit}}                 (cook stage only)
```

Note: the docs call the probe field `probe` in one example; the app sends
`temperatureProbe`, matching the state node name and a known-working Python
implementation. **Untested on hardware.**

### Outbound: `CMD_APO_STOP`

```json
{"command":"CMD_APO_STOP","requestId":"<uuid>",
 "payload":{"id":"<cookerId>","type":"CMD_APO_STOP"}}
```

### Command acknowledgement

`state.state.processedCommandIds` lists ids the oven actually acted on. The app
sends a command, keeps its `requestId` and `cookId` in `pending`, and only
reports success once one of them appears in that list. 12-second timeout, after
which it tells the user to check the oven's panel. Which of the two ids the oven
echoes has not been pinned down, so it watches for both.

---

## v1 hardware limits enforced by the app

| Mode | Range |
|---|---|
| Dry | 75–482 °F / 25–250 °C |
| Dry, bottom element only | 75–356 °F / 25–180 °C |
| Wet (sous vide) | 75–212 °F / 25–100 °C |
| Probe | 33–212 °F / 1–100 °C |

The oven rejects **all three heating elements on** and **all three off**. Sous
vide mode requires steam. All of this is enforced in `validate()` before sending.

---

## Code layout (`index.html`, one file)

CSS, HTML and JS inline. No frameworks, no CDN except Google Fonts (Barlow and
Barlow Condensed). Vanilla DOM.

- `store` — localStorage wrapper: `anova.token`, `anova.unit`, `anova.recipes`.
- `connect()` / `scheduleRetry()` — socket with exponential backoff, forced
  reconnect after 8 min of silence, reconnect on `visibilitychange`.
- `handle()` / `normalize()` — inbound routing, v1/v2 shape flattening.
- `renderCook()` / `drawScale()` / `drawStrip()` — live screen.
- `blankStage()` / `stageForm()` / `validate()` — recipe editor.
- `stagePair()` / `runRecipe()` / `stopCook()` — payload construction and sending.
- `expect()` — command acknowledgement tracking.

Recipe object:
```js
{id, name, stages:[{temp, unit:'F'|'C', mode:'dry'|'wet', steam:'none'|'rh'|'pct',
  steamVal, top, bottom, rear, fan, vent, rack, timer, timerStart, probe}]}
```
`probe` is always stored in °F. `temp` is stored in whatever `unit` says.

### Design tokens

Dark warm charcoal (`--oven:#16120F`, `--surface:#211B16`, `--raised:#2E2621`).
Amber `#F0A039` means dry heat, steam blue `#7FB6D5` means wet or humidity, red
`#D9553F` is reserved for stop and faults. The colour split is semantic — keep it.
Type: Barlow Condensed for numerals and headings, Barlow for body.

---

## Testing

There is no browser in the Claude container, so the app is tested headlessly with
jsdom (`npm install jsdom`), stubbing `WebSocket` to capture outbound frames and
inject recorded state messages. This catches real bugs — use it rather than
eyeballing. Ask Claude to rebuild the harness; the pattern is:

1. Load `index.html` with `runScripts:'dangerously'`, stub `WebSocket`,
   `crypto.randomUUID`, `scrollTo`, `Element.prototype.scrollIntoView`.
2. Drive the real DOM (`.click()`, set `.value` + dispatch `input`).
3. Assert on rendered text and on the captured `CMD_APO_START` JSON.

---

## Open items

1. **Mid-cook state capture.** Setup → Show raw oven data while a multi-stage cook
   is running. Pins down the `cook` object, `timer.mode` values, and stage transitions.
2. **Probe cook on hardware.** Confirms `temperatureProbe` vs `probe` field naming.
3. **Which id `processedCommandIds` echoes** — requestId or cookId.
4. Possible features: keep the screen awake during a cook (Wake Lock API),
   notification when a stage ends, cook history, per-recipe rack reminders.

## Things not to break

- Never show a setpoint while idle; the oven reports a stale one.
- Never store the access token in a file — it belongs only in phone storage.
- Keep the send → acknowledge flow; don't revert to claiming success on send.
- Keep the validation guards; the oven silently rejects bad combinations.
