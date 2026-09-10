# Oven — a recipe app that drives an Anova Precision Oven v1

A small web app that installs to your Android home screen like a normal app. It
holds your recipes — ingredients, method, photos, notes, and a log of how each
bake actually went — and for the ones that use the oven, it sends the cook over
Anova's official developer API.

No server of mine and no Home Assistant. **Your oven token stays on the phone and
is never synced.** Recipes are kept on the phone and, if you sign in with Google,
mirrored to your own Firebase project so they survive the phone and appear on any
device you sign in from.

---

## What you get

- **Cook** — live dry/wet bulb temperature against the setpoint, humidity, probe,
  fan, timer, plus door-open, empty-tank and failed-element warnings. One big
  stop button while a cook is running, and the screen stays awake while it runs.
- **Recipes** — ingredients and method alongside the oven programme, or no oven
  programme at all for the things you cook on the hob. Scale a recipe ×½ to ×3,
  see baker's percentages when there's flour in it, tick ingredients and steps off
  as you go, and see the amounts a step needs listed under that step. A step that
  names a duration gets a timer.
- **Multi-stage cooks** — each stage has a temperature, dry or sous vide mode,
  which heating elements run, humidity or steam percentage, fan speed, vent, rack
  position, and how the stage ends: a timer, a probe target, or holding until you
  press start on the oven itself.
- **Bake log** — what you changed and how it came out, kept with the recipe.
- **Bossy mode** — a read-aloud companion page for whoever is helping. One link,
  no account, works on an iPhone, and it hands out compliments.
- **Setup** — paste your token, switch °F/°C (which also changes the oven's own
  display), sign in to sync, share your recipe list, export and import.

The app refuses to send anything the v1 oven will reject: all three heating elements
on at once, a sous vide setpoint above 212 °F, 400 °F on the bottom element alone,
a stage in the middle with no way to end, and so on. The oven refuses such cooks
silently, so the app catches them first.

---

## 1. Get your token

In the **Anova Oven** app on your phone: **More → Developer → Personal Access
Tokens → Create**. Name it something like "phone app". You'll get a string starting
with `anova-`. Copy it.

You can have up to 10 of these, and you can revoke this one any time without
affecting the Anova app itself.

## 2. Put these files on the web

The app has to be served over `https://` for Android to offer to install it. Two
free ways, pick either:

**GitHub Pages**
1. Make a new public repo.
2. Upload `index.html`, `manifest.webmanifest`, `sw.js`, and the three `icon-*.png`
   files to the root.
3. Settings → Pages → Source: *Deploy from a branch*, branch `main`, folder `/root`.
4. Wait a minute, then open the URL it gives you.

**Netlify Drop** — go to `app.netlify.com/drop` and drag the whole folder onto the
page. You get a URL immediately. No account needed to start.

This repo uses the first one: GitHub Pages, branch `main`, folder `/root`.

## 3. Install it on your phone

Open that URL in Chrome on Android. Chrome will either show an "Install app"
prompt or you can use **⋮ menu → Add to Home screen**. It'll get an icon and open
full-screen with no browser chrome.

## 4. Connect

Open the app → **Setup** → paste your token → **Connect**. Within a few seconds the
header should show your oven's name and the dot should go green.

---

## If the dot never goes green

The most likely cause is that Anova's server rejects WebSocket connections coming
from a web page rather than a native app. I could not test this from here, so treat
it as the one real unknown.

Tell me what you see and we'll switch approaches — the same recipe logic can be
wrapped so the connection comes from somewhere the server is happy with.

Other things to check first:
- The token was pasted whole, including the `anova-` prefix.
- The oven is powered on and shows as connected in the Anova app.
- Setup → **Show raw oven data**. If there's JSON in there, the connection is fine
  and something else is wrong. Copy it and send it to me.

## Other notes

- Connections drop after 30 minutes of silence; the app reconnects on its own and
  again whenever you bring it back to the foreground.
- Anova rate-limits this API for personal use. If you hammer it, the socket closes.
- The token is stored in this browser's local storage for this site only. Clearing
  Chrome's site data for the URL removes it. Use **Forget token** to remove it yourself.
- Anything you host publicly is public — but the token is never in the files, only
  in your phone's storage, so the URL alone gives nobody access to your oven.

## Safety

This starts a 480 °F appliance from a phone. Don't run a cook you can't see,
don't leave the oven loaded with something flammable, and check the oven's own
panel agrees with the app before you walk away.
