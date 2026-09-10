# Anova oven app — handoff notes

Everything a fresh Claude session needs to keep working on this app. Upload this
file plus `index.html` at the start of a new chat and say "here's my Anova oven
app, I want to change X."

Written 2026-09-06. Owner runs an **Anova Precision Oven 1.0, 120V, firmware 2.1.16**.

---

## What this is

A recipe app that also drives the oven. `index.html` is the whole app - one
file, installed to an Android home screen as a PWA - and it holds recipes with
ingredients, method, photos and a log of past bakes, of which the oven programme
is one optional part. It talks to the oven through Anova's official developer
API. Deployed on GitHub Pages, served from the `main` branch at the repo root.

Recipes live in Firestore under the owner's Google account and are mirrored to
`localStorage`, which stays the local source of truth; the oven token lives only
in `localStorage` and is never synced. No server of our own.

`bossy.html` is a second, standalone page for whoever is reading the recipe out
loud - see **The helper page** below.

Supporting files: `bossy.html`, `manifest.webmanifest`, `sw.js` (network-first
shell cache), the icons, `recipes/library.json` (the shared library),
`firestore.rules`, `RECIPES.md`, `docs/anova-quick-start-guide.pdf`, and
`.claude/skills/add-recipe/`.

There used to be a `netlify.toml` that sent `Cache-Control: max-age=0,
must-revalidate` for `sw.js` and `index.html`, so the phone could never keep
running an old app shell after a deploy. GitHub Pages does not support custom
headers, so that file was inert and has been removed. Nothing replaces it: Pages
serves everything with `max-age=600`, browsers cap a service worker script at 24
hours and bypass the HTTP cache when checking it for updates, and the fetch
handler is network-first, so `recipes/library.json` is re-fetched whenever the
phone is online. Worst case a deploy takes ten minutes to reach a phone that has
the app open. If that ever becomes a problem, bump `CACHE` in `sw.js`.

**Status: working on real hardware.** Connects, reads live state, starts and
stops cooks, including multi-stage and sous vide. Confirmed 2026-09-05/06/07.
Recipe sync and photos confirmed 2026-09-07. Two things have never run against
the oven: the **mid-recipe hold** and the **probe path** - see Open items.

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

Now taken from the **Quick Start Guide**, a copy of which is in
`docs/anova-quick-start-guide.pdf`. Read it before guessing at a constraint -
several things chased experimentally today are stated plainly in it.

| Mode | Range |
|---|---|
| Dry (sous vide off) | 77–482 °F / 25–250 °C |
| Dry, bottom element only | 77–356 °F / 25–180 °C |
| Wet (sous vide on) | 77–212 °F / 25–100 °C |
| Probe | 33–212 °F / 1–100 °C |

**The floor was wrong until 2026-09-06** - the app advertised 75 °F where the
oven's is 77 °F (25 °C). The Celsius check also carried a 0.6 °C tolerance, over
a degree Fahrenheit, which let 76 °F and 483 °F through; it is 0.05 now, since
every Fahrenheit bound is an exact conversion of its Celsius one.

Other constraints the guide states outright:

- **"The convection fan will always run at high speed while the rear element is
  in use."** Anova's own words, and it is why a sous vide stage at fan 25 is
  refused - sous vide runs on the rear element.
- **Steam locks the fan to high as well.** Not in the guide, but the owner
  reports that Anova's app locks the fan whenever steam is set to anything other
  than 0%. `needsFullFan()` matches that exactly, **0% included: steam at 0% is
  not steam and locks nothing.** So a top-and-bottom stage with no steam is the
  only case where the fan is yours to choose.
- Elements: top and rear 1600 W to 482 °F, bottom 700 W to 356 °F.
- **Five tray positions**, not three. The app offered 1-3 until 2026-09-06.
- **Steam percentage changes meaning at 212 °F / 100 °C.** At or below, the
  figure is relative humidity and the boiler fires only as needed; above, steam
  is generated continuously and the oven does not measure humidity at all. So a
  `relative-humidity` setpoint above 212 °F cannot be honoured, and `validate()`
  now rejects it and points at steam percentage instead.
- The oven cannot dehumidify below ambient, so a low humidity target may be
  unreachable in a humid kitchen. Not enforced; nothing to enforce.
- Sous vide never browns. Finish with a separate non-sous-vide stage at high
  heat - which is exactly what a multi-stage recipe is for.

The oven rejects **all three heating elements on** and **all three off**. Sous
vide mode requires steam. All of this is enforced in `validate()` before sending.

---

## Code layout (`index.html`, one file)

CSS, HTML and JS inline. No frameworks, no CDN except Google Fonts (Barlow and
Barlow Condensed). Vanilla DOM.

The oven half:

- `store` — localStorage wrapper: `anova.token`, `anova.unit`, `anova.recipes`,
  `anova.deleted`, `anova.wake`, `anova.compliments`, `anova.shareId`,
  `anova.sharing`.
- `connect()` / `scheduleRetry()` — socket with exponential backoff, forced
  reconnect after 8 min of silence, reconnect on `visibilitychange`.
- `handle()` / `normalize()` — inbound routing, v1/v2 shape flattening.
- `renderCook()` / `drawScale()` / `drawStrip()` — live screen.
- `stagePair()` / `runRecipe()` / `stopCook()` — payload construction and sending.
  **Read "The multi-stage payload" before touching `stagePair()`.**
- `expect()` — command acknowledgement tracking.
- `wakeApply()` / `wakeFollow()` — screen wake lock during a cook.

The recipe half:

- `normalizeRecipe()` — **the trust boundary.** Everything arriving from outside
  (paste, file, library, sync) goes through it: gaps filled, types coerced,
  ranges clamped, ids reissued, `photo` accepted only as an `https://` URL.
- `validate()` — oven constraints; returns early for `usesOven: false`.
- `parseIngredients()` / `parseIngLine()` / `parseQty()` / `showQty()` —
  quantity, unit and name off each line. Handles `1/2` and `½`.
- `bakersPercents()` — flour at 100%, when the list contains flour.
- `scaleLine()` — used by the ×½–×3 control in the cooking view.
- `splitSteps()` — blank-line **or** numbered-line separated methods.
- `mentionedIn()` / `headingFor()` / `NOTANAME` — which ingredients a step names,
  so the amounts can be listed under it. Longest run of words wins; a lone
  descriptor ("large", "can") never matches.
- `durationIn()` — pulls a duration out of a step for the timer button.
- `parsePastedRecipe()` — prose to recipe, opened in the editor for review.
- `drawView()` — the cooking screen. `showRecipe()` / `logBake()` / `shareBossy()`.
- The Firestore module is a separate inline `<script type="module">` at the end
  of the file. It attaches `window.ovenSync`; everything degrades if it never
  loads.

Recipe object:
```js
{id, name, servings, ingredients, steps, notes, source, tags:[], photo, hasPhoto,
 usesOven, updatedAt, bakes:[{id, at, changed, result}],
 stages:[{temp, unit:'F'|'C', mode:'dry'|'wet', steam:'none'|'rh'|'pct',
   steamVal, top, bottom, rear, fan, vent, rack, timer, probe, hold}]}
```
`probe` is always stored in °F. `temp` is stored in whatever `unit` says.
`ingredients`, `steps` and `notes` are plain text, shown as written - nothing
rewrites what the cook typed.

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

## The multi-stage payload, as confirmed working

**Sous vide (`mode: "wet"`) stages get NO preheat** - just a bare `cook`. The
oven holds a wet-bulb temperature and there is nothing to preheat to; the
official plan's wet proof stage has no partner while both its dry stages do.
Emitting a preheat for a wet stage makes the oven refuse the cook, silently,
exactly like the multi-stage id bug. Found 2026-09-06 when a single-stage proof
recipe would not start.

For each **dry** user stage, in order, emit a pair:

```
preheat: id "<uuid>-preheat", type "preheat", no timer/probe fields
cook:    id "<uuid>",         type "cook",    timerAdded/probeAdded + timer{initial}
```

Both halves carry the same temperature, elements, fan, vent, rack and steam.
A proof-then-bake recipe therefore compiles to `cook, preheat, cook, preheat,
cook` - the exact shape of the captured official plan.
`cookId` is `android-<uuid>`; **stage ids are bare**. No `stageTransitionType`,
no `timer.startType`, no `timerStartOnDetect` anywhere. The last stage may set
`userActionRequired: true` when it has neither timer nor probe, which parks the
oven at temperature until someone presses its panel.

## Holding a stage for the cook

A recipe stage carries `hold`. When set, `stagePair()` puts
`userActionRequired: true` on both halves of that stage, and the oven holds at
temperature until its front panel is pressed - the app surfaces this through
`stageTransitionPendingUserAction` as "Preheated - press start on the oven".
The official Anova plan sets `userActionRequired` on every stage, so a mid-plan
hold is well supported by the protocol; **this app's use of it is not yet
confirmed on hardware.**

It exists so a single recipe can soak a baking steel, wait while the food goes
in, and then bake - see "Salt rolls" in the library.

## Recipe sync (Firestore)

**Confirmed working on hardware 2026-09-07.** Recipes live in Firestore under
`users/{uid}/recipes/{recipeId}`, one document per recipe, with Google sign-in. Rules are in `firestore.rules` and must be
pasted into the Firebase console by hand - there is no Node on this Mac, so no
firebase CLI.

- The SDK is imported from the gstatic CDN inside an **inline
  `<script type="module">`** at the end of `index.html`. The main app stays a
  plain script and runs first, so its functions exist when the module executes.
- **localStorage remains the local source of truth.** The module reads and
  writes it through `syncApply()` / `store.recipes`, so the app is fully usable
  signed out, offline, or if the SDK never loads. Every failure path there is
  swallowed deliberately.
- `persistentLocalCache` keeps Firestore working on bad kitchen wifi and flushes
  writes when the connection returns.
- Merge is last-write-wins per recipe on `updatedAt`, with local tombstones
  (`anova.deleted`) beating an older remote copy. `firstMerge()` runs once at
  sign-in; `onSnapshot` keeps it current after that.
- `syncPush()` is called after every local mutation and is debounced 600 ms,
  because the editor rewrites the whole list on each save.
- **The Anova token is deliberately not synced.** It stays in `anova.token` on
  the phone, per "things not to break".

The web config in the module is public project identification, not a secret -
Google intends it to ship in client code, and access is gated by the rules and
sign-in.

## Photos

**Firebase Storage is not used.** Google requires the paid Blaze plan to
provision a bucket, even though the free allowance would cover this. Photos go
in Firestore instead, as one document per recipe at
`users/{uid}/photos/{recipeId}` holding a base64 JPEG. No extra rules are
needed - the existing `users/{uid}/**` rule already covers it.

- A Firestore document caps at **1 MiB** and base64 adds a third, so
  `shrinkToFit()` encodes at 1200px q0.78 and steps down through 1000, 800, 640
  and 480 until the string is under `PHOTO_CAP` (700k characters). If even 480px
  will not fit, the upload is refused rather than failing at the server.
- The photo is kept **out of the recipe document** deliberately: recipes are
  rewritten on every save and pushed whole, and a base64 JPEG riding along would
  make every edit expensive.
- `photoCache` holds fetched images for the session; a Firestore read is not
  free and the card and the view both want the same image.
- `hasPhoto` on the recipe says a document exists. `photo` remains for an
  external `https://` URL and is still validated, so a recipe from a paste, a
  file or the library cannot smuggle in `javascript:` or a data URI.

## Bake log, paste import, step timers

- **Bakes** live on the recipe as `bakes[]` - `{id, at, changed, result}`, newest
  first, capped at 60. Logged from the cooking view; two questions, what changed
  and how it came out. Deliberately part of the recipe document so it syncs and
  scales with it.
- **Paste import** (`parsePastedRecipe`) splits written prose into name,
  servings, ingredients, method and notes. It looks for `Ingredients` /
  `Instructions` headings, and when there are none it infers the list from lines
  that begin with a quantity, ending it at the first sentence. The result opens
  **in the editor for review, never saved blind** - the split is a guess and the
  cook should see it.
- **Step timers** (`durationIn`) read a duration out of a step - "about 60
  minutes", "2-3 min", "20-24 minutes" - taking the longer end of a range.
  Memory only, not persisted: a timer is something you stand next to. They are
  for the proofs and rests either side of the cook; the oven times its own
  stages.

**Fixed in passing: tags were never saved.** Two editor handlers each copied the
same block of field reads, and an earlier patch used a guard that silently
skipped when its anchor matched twice - so the copy in `btnSave` never gained
the tags line. Both now call one `readEditor()`. Do not duplicate that block
again, and do not write a patch that skips silently when it does not match.

## The helper page (`bossy.html`)

A standalone page for whoever is reading the recipe aloud - "bossy mode", an
in-joke with Kat. Big type, mise en place first as a checkable list plus the
whole method, then one step at a time with the ingredients that step needs
listed under it, a timer where a step names a duration, and a rotating
compliment. No SDK, no account, nothing to install; it adds to an iPhone home
screen like an app.

- **It gets recipes two ways.** `?s=<shareId>` fetches the owner's shared list
  over the plain Firestore REST API and shows a pickable list. `?r=<payload>`
  carries a single recipe base64'd in the URL. Both are cached in
  `localStorage`, so the home-screen icon works with no signal.
- **Use the query string, never the fragment.** iOS discards the fragment when
  a page is added to the home screen, and the icon opens to nothing. `#r=` is
  still parsed so older links keep working.
- **Sharing** publishes `{owner, at, data}` to `shared/{shareId}` - one public
  document at an unguessable 32-hex address, republished on every recipe change.
  It carries ingredients and method only: no photos, no notes, no bake log,
  nothing that could drive the oven. "Stop sharing" deletes it.
- **It duplicates `mentionedIn()`, `splitSteps()`, `UNITS` and `NOTANAME`
  verbatim** from `index.html`. That is deliberate - the page loads no modules -
  but it has already caused one silent bug, where the two files disagreed about
  the shape of a parsed ingredient and no amounts ever appeared. **If you change
  one of those functions, change both, and make sure the data shapes still
  match.**

## Adding a recipe

There is a project skill at `.claude/skills/add-recipe/` covering the workflow,
the constraints the oven enforces silently, and a validation block that runs a
candidate recipe through the app's own `validate()` and stage compiler before it
is pushed. Use it rather than reasoning about the rules from memory - the oven
refuses an invalid cook without saying anything, which is expensive to diagnose.

## Getting recipes onto the phone

Three routes, in order of how little the owner has to do:

1. **Repo library.** `recipes/library.json` ships with the app; the phone lists
   it under Recipes -> Recipe library and one tap copies an entry into the
   owner's own recipes. **This is how Claude adds a recipe: edit that file,
   commit, push.** Schema and the oven's rules are in `RECIPES.md`. The library
   only offers - it never overwrites or syncs, so the owner's edits are safe.
2. **Paste import.** Recipes -> Import takes JSON pasted straight in, which is
   how a recipe travels from a chat to the phone without a file.
3. **File import/export.** Still there, and Export all is the only backup the
   app has.

All three go through `normalizeRecipe()`, which fills gaps from `blankStage()`,
coerces types, clamps ranges, caps stage count and always reissues the id. Treat
it as the trust boundary for anything hand-authored - the editor assumes every
field exists and would break on a partial recipe otherwise.

## The oven refuses plans that look fine - including Anova's own

**Owner-reported 2026-09-06: custom recipes built in the official Anova app also
often fail to start on this oven.** Not always; often. That reframes every
silent refusal in this file. Some of what was chased as an app bug may be the
oven declining configurations regardless of who sends them, and it means
"matches what Anova's app sends" is not sufficient evidence that a payload will
run.

**Owner's hypothesis was right. CONFIRMED ON HARDWARE 2026-09-06: a sous vide
stage is refused at fan 25 and runs at fan 100**, everything else identical.
Silently, like every other shape error. `stagePair()` now forces `fan.speed 100`
on any wet stage, `validate()` repairs the saved recipe so the editor stops
showing a figure that is not what gets sent, and the editor shows the fan as
locked when the mode is sous vide. Dry stages keep whatever fan they are given -
a dry stage at fan 25 runs fine.

The exact threshold is unknown; 100 is simply the one value observed to work.
Note this was raised early and dismissed on a bad reading - the library file had
been changed to fan 100 but the owner's *saved copy*, imported before that
change, still held 25. **Check what is on the phone, not what is in the repo.**

**Owner's fuller rule, now implemented:** the fan must be at 100 whenever the
**rear element** or **steam** is in use. Physically sensible - the rear element
is the convection element and steam has to be circulated. `needsFullFan(s)` is
the single source of truth, used by `stagePair()`, `validate()` and the editor,
which shows the fan locked with the reason. A stage using only the top and/or
bottom elements with no steam keeps whatever fan it is given.

**Counter-evidence, unresolved.** The very first successful cook (2026-09-06
15:54, `fixtures/state-v1-cook-preheat.json`) had the rear element on and echoed
`fan.speed 25` on its **cook** stage, and the oven accepted the plan. Two
readings: single-stage plans are validated loosely - independently true, since
that same plan carried bare uuids and a `stageTransitionType` that a multi-stage
plan will not accept - or the cook stage would have misbehaved once reached,
which was never observed because the capture is 5 s into the preheat. Forcing
the fan is applied anyway: the failure mode is a silent drop, and a faster fan
is cheap.

**Diagnostic (still shipped, Setup -> Troubleshooting -> "Send Anova's own proof
stage"): it STARTED, 2026-09-06.** The oven accepts the reference wet stage
replayed field-for-field, so the payload this app built was at fault, not the
oven. Keep the button - it is the fastest way to re-split app-vs-oven if
something else starts failing.

**What that isolated.** A wet stage this app built carried five fields the
reference does not - `stepType`, `description`, `rackPosition`, `timerAdded`,
`probeAdded` - and `userActionRequired: false` where the reference has `true`.
All of those are provably fine on a **dry** stage: timed two-stage dry recipes
run with every one of them. So a wet stage is now emitted in exactly the
reference shape and a dry stage is left alone.

**Which of the six actually mattered is unknown.** Bisecting would cost the
owner a kitchen trip per field and buys nothing while both paths work. If it
ever matters, add them back one at a time to a wet stage.

## Open items

**Unverified paths, in the order they will bite:**

1. **The mid-recipe hold.** "Salt rolls" opens with a 45 min steel soak and then
   waits for a press on the oven's panel before baking. The protocol supports it
   - Anova's own plan sets `userActionRequired` on every stage - but this app's
   use of it has never run. Worth a dry run with an empty oven and a two-minute
   soak before anyone commits dough to it.
2. **The probe path. BLOCKED** - the owner's probe broke 2026-09-06 and cannot
   be tested until it is replaced. `stagePair()` sends `temperatureProbe`; the
   SDK docs call it `probe` in one example. Do not assume a probe recipe works
   because timed ones do.
3. **A capture at the preheat -> cook transition**, to see
   `stageTransitionPendingUserAction` actually flip.
4. **Which id `processedCommandIds` echoes** - `requestId` or `cookId`. Evidence
   points at `requestId`; never confirmed.
5. **DOM wiring is generally untested by the harness.** JavaScriptCore covers
   logic, not rendering. Several screens have only been driven through the
   in-app browser by hand.

**Ideas, not commitments:** a notification when a stage ends (needs background
execution the web cannot give - that is the Capacitor conversation from
2026-09-06), per-recipe rack reminders, sub-recipes or reusable components
(the tangzhong and the honey butter are both recipes inside recipes).

**Done since this list was written:** screen wake lock, multi-stage cooks, sous
vide, error reporting, Firestore sync, photos, ingredients and method, scaling
and baker's percentages, tags and search, the bake log, paste import, step
timers, and the helper page.

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

1. **Multi-stage recipes would not start. FIXED AND CONFIRMED ON HARDWARE
   2026-09-06** - a simple two-stage recipe now runs. The cause was stage id
   pairing (see below); everything above it in this entry is the trail of three
   wrong guesses, kept because the reasoning shows what the oven does *not*
   care about. Owner-reported: adding a second stage
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

   *B - the oven was hot and venting.* **RULED OUT 2026-09-06.** A single-stage
   recipe started normally with the oven above 250 F, so cooldown state is not
   the cause. Stage count is.

   **ACTUAL CAUSE, from a real official-app cook (2026-09-06).** Owner ran a
   multi-stage recipe from the Anova app; the echoed plan is saved as
   `fixtures/state-v1-official-anova-multistage.json`. The stage ids give it away:

   ```
   "id": "0e81c84f-3a29-4033-b2f0-3199b0059ceb-preheat"   <- preheat
   "id": "0e81c84f-3a29-4033-b2f0-3199b0059ceb"           <- its cook
   ```

   **A preheat is bound to its cook stage by id: `<cookStageId>-preheat`.** This
   app generated two unrelated uuids per pair, so every preheat referred to
   nothing. One pair happened to survive that; more than one did not, and the
   oven dropped the plan silently. Fixed in `stagePair()`.

   Two corrections to the previous round while here:
   - **Stage ids must stay bare uuids.** `cookId` takes the `android-` prefix
     (confirmed: `android-68b93a6d-...`) but stage ids do not. The third-party
     reference example prefixes both; the real oven does not.
   - `userActionRequired: true` is set on **every** stage in the official plan,
     mid-plan included, so it is not the "only the last stage" rule assumed
     earlier. Harmless as we send it, but the earlier reasoning was wrong.

   Also visible: the official plan's **first stage has no preheat at all** (it is
   a wet proof stage). Preheats appear to be emitted only when the setpoint has
   to rise. This app always emits one; not known to be a problem.

   **Superseded reasoning below (kept for the audit trail).** A known-working
   3-stage payload exists at
   `https://github.com/bogd/anova-oven-api/blob/main/docs/examples/CMD_APO_START.json`,
   saved as `fixtures/reference-CMD_APO_START.json`. It is preheat/cook pairs -
   so the original layout was right and both of my layout guesses were wrong
   about which part was broken. Diffing it against
   `fixtures/sent-2stage-rejected.json` showed four field-level differences, all
   now corrected:

   | | was sent | reference |
   |---|---|---|
   | `cookId`, stage `id` | bare uuid | **`android-<uuid>`** |
   | `stageTransitionType` | `"automatic"` / `"manual"` | absent entirely |
   | `timer.startType` | `"when-preheated"` | absent |
   | `timerStartOnDetect` | `false` | absent |

   The stage field sets now match the reference exactly, asserted in the tests.
   Which of the four mattered is unknown - a single-stage cook was accepted with
   all four wrong, so the oven is stricter about multi-stage payloads than
   single-stage ones. If narrowing that down ever matters, reintroduce them one
   at a time.

   **Casualty, now resolved:** the recipe editor's "start timer when preheated /
   immediately / when I tap start" option could no longer reach the oven, since a
   working payload has no `timer.startType`. The control was removed on
   2026-09-06 rather than left inert, and the editor now states plainly that the
   timer starts once the oven reaches temperature - which is what the preheat
   stage already guarantees. `timerStart` is gone from the recipe object; old
   saved recipes simply carry an ignored extra key.

   **REMOVED: the `store.plan` layout selector.** It briefly let the owner switch
   between preheat layouts from Setup. It also created a trap that cost a full
   test cycle: the owner had selected "No preheat" while testing, that choice
   persisted in `localStorage`, and a stored value beats a changed default - so
   a later fix to the preheat pairing never ran on his phone at all. The layout
   is now a constant (preheat before every stage), the selector is gone, and boot
   deletes any `anova.plan` left behind.

   **Lesson worth keeping: do not ship an experiment as a stored preference.**
   A stored value outlives the experiment and silently disables whatever comes
   next. Hardcode the variant, ship it, and change the code to test the next one.
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
- Never store the access token in a file, and never sync it. It belongs only in
  the phone's `localStorage`. The Firebase web config in the module is *not* a
  secret and is fine in source; the oven token is the real one.
- Keep the send → acknowledge flow; don't revert to claiming success on send.
- Keep the validation guards; the oven silently rejects bad combinations, so a
  guard removed here becomes an hour of diagnosis in a kitchen.
- `localStorage` stays the local source of truth. The app must remain fully
  usable signed out, offline, or with the Firebase SDK failing to load — every
  swallowed error in the sync module is deliberate.
- Everything from outside goes through `normalizeRecipe()`. Do not add a path
  that writes a recipe without it.
- Don't overwrite the owner's `notes`, and never touch `bakes` — that is his
  record of what actually happened.
- Don't ship an experiment as a stored preference. A value saved on the phone
  outlives the experiment and silently disables the next fix; that cost a full
  test cycle on 2026-09-06.
- Don't write a patch that skips silently when its anchor doesn't match. That is
  how the editor lost its tags for a day.

## Working habits that paid off

- **The oven says nothing when it refuses a cook.** Get a capture from Anova's
  own app and diff against it rather than reasoning from docs; two rounds of
  confident wrong answers came from trusting third-party documentation over the
  device. `fixtures/state-v1-official-anova-multistage.json` is the gold copy.
- **Check what is on the phone, not what is in the repo.** The fan rule was
  dismissed once because the library file had been fixed while the owner's saved
  copy still held the old value.
- **There is no Node on this Mac.** JavaScriptCore is the test runner; extract
  the functions under test out of `index.html` with Python and drive them with
  stubs. See Testing.
- **The in-app browser can drive the deployed app**, which is the only way to
  check rendering. Clear `caches` first or the service worker serves a stale
  shell. Screenshots come out misaligned when the page is scrolled — trust the
  DOM over the picture.
