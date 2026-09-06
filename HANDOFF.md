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

### Active cook (`payload.state.cook`) — VERIFIED 2026-09-06

Real capture in `fixtures/state-v1-cook-preheat.json` (one user stage, 400 F dry,
no timer, no probe, 5 s into preheat). Corrections to the doc-derived shape below:

- **`originSource` and `type` are NOT present** on the cook object. Only:
  `stages, activeStageSecondsElapsed, activeStageId, cookId,
  stageTransitionPendingUserAction, secondsElapsed, activeStageIndex`.
- **The oven strips the stages it echoes back.** Sent fields `stepType`,
  `description`, `rackPosition`, `stageTransitionType`, `timerAdded`,
  `probeAdded`, `timerStartOnDetect` are all absent from the echo. Echoed stages
  carry only `id, type, title, userActionRequired, temperatureBulbs,
  heatingElements, fan, vent` (plus `timer`/`temperatureProbe` when set).
  **Do not use the echo to verify what was sent.**
- **The oven rewrites the preheat stage's fan to 100** regardless of the stage
  fan. In the capture the preheat echoes `fan.speed 100` and the cook `25`, from
  a recipe whose two stages `stagePair()` builds with identical fan.
- `timer.mode` is `"idle"` during a running cook when no timer was set.
- `temperatureProbe` is `{connected:false}` alone when unplugged — no `current`
  or `setpoint` keys. Code must not assume they exist.
- Live `dry.setpoint` reads 399.99 F for a requested 400. Round for display.
- `secondsElapsed` and `activeStageSecondsElapsed` are equal during stage 0.
- New `systemInfo` fields: `firmwareUpdatedTimestamp`, `uiHardwareVersion`
  (`"UI_ORIGINAL_2"`), `powerHertz`, `lastConnectedTimestamp`,
  `lastDisconnectedTimestamp`. `state.updatedTimestamp` sits beside `nodes`.

**`processedCommandIds` (open item 3):** the `cookId` of the running cook is NOT
in the list, which points at `requestId` being the echoed id. Evidence, not
proof — it is not yet confirmed that this cook was started from this app.

### Doc-derived shape, superseded by the above

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

**On this Mac there is no Node**, so the jsdom route below is unavailable. What
works instead is macOS's built-in JavaScriptCore at
`/System/Library/Frameworks/JavaScriptCore.framework/Versions/A/Helpers/jsc`:
extract the functions under test out of `index.html` with a short Python script,
`load()` them into jsc with stubs for `store`, `uuid`, `navigator` and
`document`, and assert on the objects they build. That covers payload
construction, validation and the wake-lock state machine — everything except DOM
wiring. Installing Node would restore the fuller jsdom option:

jsdom (`npm install jsdom`) stubs `WebSocket` to capture outbound frames and
inject recorded state messages. This catches real bugs — use it rather than
eyeballing. Ask Claude to rebuild the harness; the pattern is:

1. Load `index.html` with `runScripts:'dangerously'`, stub `WebSocket`,
   `crypto.randomUUID`, `scrollTo`, `Element.prototype.scrollIntoView`.
2. Drive the real DOM (`.click()`, set `.value` + dispatch `input`).
3. Assert on rendered text and on the captured `CMD_APO_START` JSON.

---

## Open items

1. ~~Mid-cook state capture.~~ Done 2026-09-06, see above. Still wanted: a capture
   **at the preheat -> cook transition** (does `stageTransitionPendingUserAction`
   flip to true?) and one from a genuinely multi-stage recipe (4+ API stages).
2. **Probe cook on hardware.** Confirms `temperatureProbe` vs `probe` field naming.
3. **Which id `processedCommandIds` echoes** — requestId or cookId.
4. Possible features: notification when a stage ends, cook history,
   per-recipe rack reminders. (Screen wake lock: done 2026-09-06.)

## Screen wake lock

`wakeApply()` / `wakeFollow()` next to the other lifecycle handlers. The lock is
held only while a cook is running, driven from the `cooking` flag `renderCook()`
already computes. Android silently drops the lock whenever the page is hidden and
never restores it, so `wakeApply()` is both the acquire and the re-acquire path
and is called from `visibilitychange`. Setting lives at `anova.wake`, default on,
toggled under Setup -> Screen; the buttons disable themselves when
`navigator.wakeLock` is missing. A denied request (battery saver) is swallowed.

Verified with a JavaScriptCore harness driving the extracted functions against a
stubbed `navigator.wakeLock`. **The DOM wiring - the Setup toggle - has not been
exercised in a browser.**

## Bugs found from the 2026-09-06 capture — all fixed same day

1. **Multi-stage recipes would not start.** Owner-reported: adding a second stage
   made a working recipe fail to start; deleting it fixed it. `stagePair()` set
   `userActionRequired:true` + `stageTransitionType:'manual'` on *every* cook
   stage lacking a timer and probe, so a middle stage had no exit condition -
   the oven was asked to wait forever partway through a plan. Now only the
   **final** stage may go manual (`stagePair(s, isLast)`), and `validate()`
   rejects a non-final stage with neither timer nor probe before sending.
   **This did NOT fix it.** A 2-stage recipe with a timer on every stage still
   fails to start (capture: `fixtures/state-v1-idle-after-rejected-start.json`,
   `mode:"idle"`, no `cook` object - the oven refused the plan outright). Note
   that with timers set, the old and new code build near-identical payloads, so
   this change was never going to address the timed case. The change is still
   correct on its own terms, but **the multi-stage cause is still unknown.**

   **The oven sends no error at all.** Full outbound frame captured in
   `fixtures/sent-2stage-rejected.json`: 4 well-formed stages, and the oven never
   responds - no `ERROR`, no `cook`, `mode` stays `idle`. The only thing that
   fires is the app's own 12 s acknowledgement timeout. So the frame is being
   accepted by the cloud and dropped by the oven, which rules out reading a
   reason off the wire. Progress from here has to come from controlled
   experiments, not from more logging.

   **Two live hypotheses, both untested:**

   *A - mid-plan preheat.* `stagePair()` emits a `preheat` before *every* user
   stage, so 2 stages send preheat, cook, preheat, cook. The oven may accept only
   one leading preheat followed by plain `cook` stages.

   *B - the oven was hot and venting.* **This is a real confound and was missed
   at first.** Every failed attempt (16:08, 16:13) happened while the oven was
   cooling from the 15:54 cook - 338 F then 298 F, `vent.open true`, `fan.speed
   0`, `rear.on true` at 0 watts. The one success started from a cool oven. The
   oven may simply refuse a new cook mid-cooldown, in which case stage count is
   irrelevant and nothing is wrong with the payload.

   **Discriminating test, do this before changing any code:** with the oven hot
   and venting, start a *single*-stage recipe. If it also fails, the cause is
   oven state (B) and the multi-stage theory is dead. If it starts, the cause is
   stage count (A). Then repeat the 2-stage recipe from a fully cool oven.
2. **A timerless cook parks at temperature with no hint.** Legitimate behaviour
   on the last stage, but the app never read `stageTransitionPendingUserAction`.
   The cook screen now shows "Preheated - press start on the oven" when it flips.
3. **Stage counts were API stages.** A one-stage recipe read "stage 1 of 2".
   `userStage()` maps back by counting `cook` stages, one per user stage.
4. **Errors were only ever toasted** for 2.6 s, so a silent rejection left no
   trace. `lastError` is kept and rendered under Setup -> Troubleshooting, and
   an ERROR now clears the pending-command wait instead of letting it time out.
5. Both stages of a pair shared one `temperatureBulbs` object. Now built per stage.

Regression tests for all of this run under JavaScriptCore (see Testing).

## Diagnosis

Failures used to leave no trace: a 2.6 s toast and nothing else, which is why
"it just will not start" was all anyone had. As of 2026-09-06:

- `fail(msg)` is the single failure path. Validation refusals, a send with no
  socket, an oven `ERROR`, and the 12 s acknowledgement timeout all route
  through it, so every one of them persists.
- `lastError` (message, timestamp, and the whole inbound frame) renders under
  **Setup -> Troubleshooting**.
- `lastSent` keeps the exact outbound frame.
- **Setup -> Show raw oven data** now dumps `{lastError, lastCommandSent,
  ovenState}` together. One copy of that is enough to diagnose a failed start -
  ask for it rather than reasoning about what the app probably sent.

## Things not to break

- Never show a setpoint while idle; the oven reports a stale one.
- Never store the access token in a file — it belongs only in phone storage.
- Keep the send → acknowledge flow; don't revert to claiming success on send.
- Keep the validation guards; the oven silently rejects bad combinations.
