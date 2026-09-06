# Adding recipes to the library

`recipes/library.json` is served with the app. Anything in it appears on the
phone under **Recipes -> Recipe library**, where one tap copies it into the
owner's own recipes. Add an entry, commit, push; Netlify redeploys and the phone
picks it up on the next open. Nothing is overwritten on the phone - the library
offers, it never syncs.

## Shape

```json
{ "recipes": [ { "name": "...", "stages": [ { ...stage... } ] } ] }
```

Omit `id`; the app always issues a fresh one on import.

## A stage

| field | type | meaning |
|---|---|---|
| `temp` | number | temperature, in whatever `unit` says |
| `unit` | `"F"` \| `"C"` | unit `temp` is written in |
| `mode` | `"dry"` \| `"wet"` | `wet` is sous vide, and requires steam |
| `steam` | `"none"` \| `"rh"` \| `"pct"` | relative humidity, or steam percentage |
| `steamVal` | 0-100 | the humidity or steam figure |
| `top`, `bottom`, `rear` | bool | heating elements |
| `fan` | 0-100 | fan speed |
| `vent` | bool | vent open |
| `rack` | 1-3 | rack position |
| `timer` | seconds | 0 means no timer |
| `probe` | °F | probe target; 0 means none. **Untested on hardware** |

Every field is optional - `normalizeRecipe()` fills gaps from `blankStage()`,
coerces types and clamps ranges, so a malformed entry degrades rather than
breaking the editor. Write them all anyway; the defaults are not obvious.

## Rules the oven enforces

- Never all three elements on, never all three off.
- **The fan must be 100 whenever the rear element or steam above 0% is in use**,
  sous vide included. Steam at 0% does not count. Confirmed on hardware for sous vide: the same stage at fan 25
  is refused without a word. The app forces it, so `fan` is ignored on those
  stages - but write 100 anyway so the file reads truthfully. Only a stage using
  the top and/or bottom elements with no steam has a fan you can choose.
- `wet` mode requires steam, and caps at 212 F / 100 C.
- Dry: 77-482 F. Dry with **only** the bottom element: 77-356 F. The floor is
  77 F / 25 C, not 75.
- **Above 212 F a humidity target is meaningless** - the oven generates steam
  continuously and stops measuring humidity. Use `"steam": "pct"` there and
  `"rh"` only at or below 212 F.
- Rack positions run **1-5**.
- **Every stage but the last needs a `timer` or a `probe`**, or the oven has no
  way to know when to move on and refuses the whole cook.
- A last stage with neither parks the oven at temperature until someone presses
  its front panel. That is legitimate, and the app says so on screen.

## Notes on writing good stages

One user stage compiles to a preheat plus a cook. To bake with steam only at
the start - the usual bread technique - write two stages at the same
temperature, the first with steam and a short timer, the second with
`"steam": "none"`.
