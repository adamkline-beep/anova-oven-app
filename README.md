# Oven — Anova Precision Oven v1 controller

A small web app that installs to your Android home screen like a normal app, stores
your recipes on the phone, and sends them to the oven over Anova's official
developer API.

No account, no server of mine, no Home Assistant. Your token and recipes never
leave your phone.

---

## What you get

- **Cook** — live dry/wet bulb temperature against the setpoint, humidity, probe,
  fan, timer, plus door-open, empty-tank and failed-element warnings. One big
  stop button while a cook is running.
- **Recipes** — multi-stage cooks. Each stage has a temperature, dry or sous vide
  mode, which heating elements run, humidity or steam percentage, fan speed, vent,
  rack position, and how the stage ends (timer, probe target, or hold until you say so).
- **Setup** — paste your token, switch °F/°C (which also changes the oven's own
  display), export and import recipes as a JSON file.

The app refuses to send anything the v1 oven will reject: all three heating elements
on at once, a sous vide setpoint above 212 °F, 400 °F on the bottom element alone,
and so on.

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
