---
name: add-recipe
description: Add or edit a recipe in the Anova oven app's shared library (recipes/library.json). Use whenever Adam supplies a recipe — pasted prose, a link, a photo of one, or a description — and wants it in the app, or asks to change an existing one. Also use when adjusting a recipe's oven stages, since the oven silently refuses combinations that look fine.
---

# Adding a recipe to the oven app

`recipes/library.json` ships with the app. Anything in it appears on Adam's
phone under **Recipes → Recipe library**, where one tap copies it into his own
recipes. Adding a recipe is: edit that file, commit, push, and tell him to pull
it down.

## The workflow

1. Read `RECIPES.md` for the full schema. This file covers the parts that are
   easy to get wrong.
2. Edit `recipes/library.json`. Use the same name as an existing entry to
   replace it — his app offers **Update mine**, which keeps his local copy's id
   and any photo he added.
3. Validate before committing (below). A recipe the oven refuses is worse than
   no recipe, because it fails silently.
4. Commit and push. GitHub Pages redeploys in under a minute.
5. **Tell him to pull it down.** Nothing reaches his phone on its own — the
   library offers, it never syncs. This step is always his.

## Validate before you push

Do not trust a recipe by reading it. Run it through the app's own validator:

```bash
cd ~/Developer/anova-oven-app
python3 - <<'PY'
import io
s = io.open('index.html', encoding='utf-8').read()
def blk(a, b):
    i = s.index(a); return s[i:s.index(b, i)]
io.open('/tmp/build.js', 'w', encoding='utf-8').write(
  "const clamp=(v,a,b)=>Math.min(b,Math.max(a,v));\nvar store={unit:'F'};\n"
  "const c2f=c=>c*9/5+32, f2c=f=>(f-32)*5/9;\n"
  "const toF=s=>s.unit==='F'?s.temp:c2f(s.temp);\nconst toC=s=>s.unit==='F'?f2c(s.temp):s.temp;\n"
  "const onlyBottom=s=>s.bottom&&!s.top&&!s.rear;\n"
  + blk('// Straight from the Quick Start Guide', 'const bounds')
  + "const bounds = lim => store.unit==='F'?lim.f:lim.c;\n"
  + blk('function blankStage(){', 'function normalizeRecipe')
  + blk('function normalizeRecipe(r){', 'function recipesFrom')
  + blk('function validate(r){', '/* =====')
  + blk('// A full fan is required', 'function runRecipe'))
PY
cat > /tmp/check.js <<'EOF'
var n=0; function uuid(){return 'u'+(++n);} function cookUuid(){return 'android-u'+(++n);}
load('/tmp/build.js');
var lib = JSON.parse(readFile('recipes/library.json'));
var bad = 0;
lib.recipes.forEach(function(raw){
  var r = normalizeRecipe(raw), e = validate(r);
  var out = [];
  r.stages.forEach(function(s, i){
    out.push.apply(out, stagePair(s, i === r.stages.length - 1, s.mode !== 'wet'));
  });
  print((e.length ? 'FAIL ' : 'ok   ') + r.name + '  -> ' + out.length + ' API stages');
  e.forEach(function(x){ print('       ' + x); bad++; });
  if (out.length > 8){ print('       more than 8 API stages: the oven will refuse this'); bad++; }
});
print(bad ? 'FIX THESE BEFORE PUSHING' : 'all recipes valid');
EOF
/System/Library/Frameworks/JavaScriptCore.framework/Versions/A/Helpers/jsc /tmp/check.js
```

There is no Node on this Mac; JavaScriptCore is the test runner. See HANDOFF.

## Rules the oven enforces, silently

An invalid cook is **dropped without any error** — no `ERROR` frame, nothing in
the state, just the app's 12-second timeout. Hours went into learning these.

- **Never all three heating elements on, and never all three off.**
- **The fan must be 100 whenever the rear element or steam above 0% is in use**,
  sous vide included. The app forces this, so write `"fan": 100` on those stages
  so the file reads truthfully. Only a top-and/or-bottom stage with no steam has
  a fan you can choose.
- **Every stage but the last needs a `timer` or a `probe`.** Without one the
  oven has no way to know when to move on and refuses the whole cook.
  A last stage with neither parks the oven at temperature until someone presses
  its panel — legitimate, and the app says so on screen.
- **Temperatures:** 77–482 °F with sous vide off, 77–212 °F with it on,
  77–356 °F when the **bottom element is the only one on**. The floor is 77, not
  75.
- **Steam changes meaning at 212 °F.** At or below, the figure is relative
  humidity (`"steam": "rh"`). Above, steam runs continuously and the oven does
  not measure humidity at all, so use `"steam": "pct"`. `validate()` rejects the
  wrong one.
- **Eight API stages maximum.** Each dry user stage compiles to a preheat plus a
  cook, so **four dry stages is the ceiling**. A sous vide stage compiles to one.
- The probe path has **never run on this oven** and his probe is broken. Avoid
  `probe` unless he asks.

## Writing stages well

- One user stage becomes a preheat and a cook. To bake with steam only at the
  start — the usual bread technique — write two stages at the same temperature,
  the first with steam and a short timer, the second with `"steam": "none"`.
- `"hold": true` makes the oven finish the previous stage, hold at temperature,
  and wait for a press on its panel before starting that one. It is how a recipe
  can preheat a baking steel and then pause while the food goes in. Ignored on
  the first stage. **Not yet confirmed on hardware.**
- The rear element is convection and browns tops hard. For a crisp base and a
  gentle top, use the bottom element for the long middle and finish with a short
  top-and-bottom stage for colour.
- Sous vide never browns. Finish with a separate non-sous-vide stage.

## Writing the food

`ingredients`, `steps` and `notes` are plain text, shown as written.

- **Ingredients:** one per line, `330g bread flour`. A line in CAPITALS or
  ending in a colon becomes a heading. Put the quantity first — the app splits
  it off, scales it, computes baker's percentages when the list contains flour,
  and shows the amounts beside the steps that mention them.
- **Steps:** one per paragraph, separated by blank lines. Numbered lines with
  single newlines also work. A step naming a duration ("about 60 minutes",
  "2-3 min") gets a timer button, so phrase times inside the step text.
- **`usesOven: false`** for anything the oven has no part in — a compound butter,
  a dressing. It skips oven validation entirely and offers no Send button.
- **`notes`** is where the record of what to change next time lives. Preserve
  what is already there; do not overwrite his notes with a tidier version.

## Things not to do

- Do not invent quantities or times he did not give. Ask, or mark the gap in
  `notes`.
- Do not rewrite a recipe he has already tuned. Change what he asked to change.
- Do not touch `bakes` — that is his record of what actually happened.
- Do not add a photo; photos are uploaded from his phone.
