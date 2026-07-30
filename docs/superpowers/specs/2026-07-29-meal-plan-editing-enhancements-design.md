# Meal Plan Editing Enhancements — Design

Date: 2026-07-29
File under change: `index.html` (single-file app, ~1200 lines)

## Summary

Four changes to the meal-plan app:

1. Fix a bug where the searchable meal-picker dropdown closes itself when the
   user tries to scroll its own option list.
2. Change grocery-list checked items so they drop to a single "Purchased"
   pile at the bottom of the whole list, not just the bottom of their
   category section.
3. Let the user edit a recipe's ingredients and steps in place, with edits
   synced across devices like everything else in the app.
4. Let the user mark a specific night as 1.5x / 2x / 3x (e.g. for guests),
   which scales that night's contribution to the grocery list.

All four are additive changes to the existing single-file architecture —
no new files, no new build step, no schema replacement (only new,
optional keys layered next to the existing ones so old data keeps
working).

---

## 1. Meal-picker scroll bug

**Root cause:** `window.addEventListener("scroll", closeMealPicker, true)`
(current line ~872) closes the picker on *any* scroll event fired during
the capture phase — including a scroll event fired by the picker's own
`.meal-picker-list` (which has `overflow-y:auto`, `max-height:240px`).
Scroll events don't bubble, but a capture-phase listener on `window` still
sees them regardless of where they originate.

**Fix:** guard the handler so it ignores scroll events whose target is
inside the currently-open picker:

```js
window.addEventListener("scroll", function(e){
  const p = document.getElementById("mealPicker");
  if(p && p.style.display==="block" && p.contains(e.target)) return;
  closeMealPicker();
}, true);
```

Scrolling the page behind the picker still closes it (unchanged — the
picker is `position:fixed` and would otherwise drift away from its anchor
button).

No data model impact. No sync impact.

---

## 2. Grocery list: checked items sink to the bottom of the whole page

**Current behavior:** `renderGrocery(slot)` renders one `.gsection` per
category (from `effectiveGrocery`), sorting checked entries to the bottom
*within* that section. Custom ("Added by You") items are their own
section at the end, with the same within-section sink behavior. Each
section's header subtotal sums every entry in that section, checked or
not.

**New behavior:**

- Every category section renders **only its unchecked items**. If a
  category has zero unchecked items left, its section (including header)
  is omitted entirely from the top of the list.
- The category header subtotal changes to reflect only the unchecked
  items still shown in that section (not the full original total).
- All checked items — from every category and from custom items — collect
  into one new section, `✓ Purchased`, rendered after every category
  section and after "Added by You". It gets its own subtotal (sum of
  everything checked).
- Order within Purchased: same relative order the items would have
  appeared in above (category iteration order, then custom), not
  check-order. No new state is introduced to track when something was
  checked.
- Unchecking an item in Purchased sends it back to its original section
  on the next render (same `toggleCheck` → `renderGrocery` flow as
  today).

**Implementation shape:** `renderGrocery` restructures its loop to
partition each section's entries into `unchecked`/`checked`, push
`checked` onto a page-level `purchased` array as it goes, render category
sections from `unchecked` only, and render one final section from
`purchased` after the loop (plus after the custom-items block, which goes
through the same partition).

No data model impact (`checks` keeps the same shape/keys). No sync
impact — this is pure rendering.

---

## 3. Editable recipes (ingredients + steps)

### Data model

New global, non-period-keyed state, following the same pattern as
`prices`/`staplePrices` (global) rather than `custom`/`checks` (per
cycle-period):

```js
let recipeEdits = load('mp_recipeEdits_v1', {});
// shape: { [recipeKey]: { ing: [...groups], steps: [...] } }
function saveRecipeEdits(){ save('mp_recipeEdits_v1', recipeEdits); }
```

`ing`/`steps` fully replace the base recipe's fields when present — no
merge at the field level, matching how `custom` fully owns its own items
rather than patching `R`. The base `R` object is never mutated; it's the
implicit "reset to original."

A single helper resolves the effective recipe everywhere it's displayed:

```js
function recipeData(k){
  const base = R[k], edit = recipeEdits[k];
  if(!edit) return base;
  return Object.assign({}, base, {
    ing: edit.ing || base.ing,
    steps: edit.steps || base.steps
  });
}
```

`recipeCard`, `showRecipeDetail`, and anywhere else `r.ing`/`r.steps` are
read get switched from `R[k]` to `recipeData(k)`. Title, `serves`, `src`,
`url`, and `note` continue to come from base `R[k]` (not editable, per
scope decision).

### Edit UI

Lives only on the recipe detail view (`showRecipeDetail`, the "All
Recipes" tab's single-recipe page) — not duplicated onto the day-by-day
`recipeCard` view on the Cycle A/B tabs, which only ever reads the
resolved data.

- A `✏️ Edit ingredients & steps` button appears next to the existing
  "← All recipes" back button.
- Clicking it re-renders the same recipe in an edit state: the
  ingredients block and steps `<ol>` are replaced by two `<textarea>`s,
  plus **Save**, **Cancel**, and (only when `recipeEdits[k]` already
  exists) **Reset to original**.
- Only one recipe can be in edit state at a time (`let recipeEditKey =
  null;` module-level).

**Text convention** (chosen to avoid building a structured multi-field
editor for a nested group/heading structure):

- Ingredients textarea: one ingredient per line. A line starting with
  `## ` begins a new named group (e.g. `## Marinade`), mirroring the
  existing `[heading, ...items]` shape where `heading` can be `""`. Lines
  before the first `## ` belong to an unheaded initial group. Blank lines
  are ignored.
- Steps textarea: one step per line, blank lines ignored; the existing
  `<ol>` still handles numbering, so no numbers are typed in the box.
- A short inline hint under each textarea documents the `## ` convention.

```js
function parseIngredients(text){
  const lines = text.split("\n").map(l=>l.trim()).filter(Boolean);
  const groups = []; let current = null;
  lines.forEach(line=>{
    if(line.startsWith("## ")){ current=[line.slice(3).trim()]; groups.push(current); }
    else { if(!current){ current=[""]; groups.push(current); } current.push(line); }
  });
  return groups.length ? groups : [[""]];
}
function ingredientsToText(groups){
  return groups.map(g=>{
    const head = g[0] ? `## ${g[0]}\n` : "";
    return head + g.slice(1).join("\n");
  }).join("\n");
}
function parseSteps(text){
  return text.split("\n").map(s=>s.trim()).filter(Boolean);
}
```

**Save flow:** parse both boxes → `recipeEdits[k] = {ing, steps}` →
`saveRecipeEdits()` (persists locally + triggers the existing
`scheduleSyncPush`) → exit edit state → re-render the detail view and
`renderRecipes()` (so Cycle A/B tabs pick up the change too).

**Reset-to-original:** `delete recipeEdits[k]; saveRecipeEdits();`
re-render.

### Sync

Added to the shared Firebase envelope alongside `staples`/`prices`/etc.
(not alongside `meals`/`checks`/`custom`, since this isn't period-keyed):

- `syncPush`: add `recipeEdits: recipeEdits` to the `state` object being
  serialized.
- `applyRemote`: `recipeEdits = s.recipeEdits || {}; save('mp_recipeEdits_v1', recipeEdits);`
  then re-render recipes (and the detail view if one happens to be open).

This is the same category of fix as the earlier "stale artifact" schema
divergence — a genuinely new field being added additively, so any client
that predates this feature simply won't echo it back (there is now only
one client, so that's a non-issue going forward, but the shape follows
the established safe pattern regardless).

---

## 4. Per-night serving multiplier

### Data model

New per-cycle, period-keyed array, following the exact same pattern as
`mealPlan`:

```js
const SCALE_OPTIONS = [1, 1.5, 2, 3];
const ckScale = p => "mp_scale_"+p;
function normalizeScale(arr){
  const a = (arr||[]).slice(0,14);
  while(a.length<14) a.push(1);
  return a.map(v => SCALE_OPTIONS.indexOf(v)>=0 ? v : 1);
}
let mealScale = {
  A: normalizeScale(load(ckScale(CTX.A.key), null)),
  B: normalizeScale(load(ckScale(CTX.B.key), null))
};
function saveScale(slot){ save(ckScale(CTX[slot].key), mealScale[slot]); }
```

### Wiring into existing meal-plan mutators

Every function that already mutates `mealPlan[slot][i]` gets the matching
`mealScale` treatment so the multiplier "belongs" to the meal, not the
calendar slot:

- `moveMeal(slot,i,dir)`: swap `mealScale[slot][i]`/`[j]` alongside the
  meal swap; `saveScale(slot)`.
- `removeMeal(slot,i)`: reset `mealScale[slot][i] = 1`; `saveScale(slot)`.
- `sendOther(slot,i)`: carry the scale value to the destination index in
  the other cycle, then reset the source slot to `1`; `saveScale` both
  slots.
- `resetMealPlan(slot)`: reset the whole `mealScale[slot]` array to all
  `1`s; `saveScale(slot)`.
- `changeMeal(slot,i,val)` (meal picker selection, including clearing or
  picking Leftovers): reset `mealScale[slot][i] = 1` — a fresh pick
  shouldn't inherit whatever the previous meal was scaled to;
  `saveScale(slot)`.

### Edit-mode UI

In `dayCard(slot,i)`'s `editMode` branch, add a small button-group row
under the meal-picker button:

```html
<div class="scale-row">
  <span class="slab">Servings</span>
  <div class="scale-btns">
    <button class="${scale===1?'sel':''}"   onclick="setScale('${slot}',${i},1)">1x</button>
    <button class="${scale===1.5?'sel':''}" onclick="setScale('${slot}',${i},1.5)">1.5x</button>
    <button class="${scale===2?'sel':''}"   onclick="setScale('${slot}',${i},2)">2x</button>
    <button class="${scale===3?'sel':''}"   onclick="setScale('${slot}',${i},3)">3x</button>
  </div>
</div>
```

Rendered only when the day has a real recipe (`isSet && k!==LEFTOVERS`);
hidden for empty days and Leftovers, since there's nothing to scale.
`setScale(slot,i,val)` sets the value, saves, and re-renders overview +
recipes + that cycle's grocery list.

New CSS (`.scale-row`, `.slab`, `.scale-btns button`, `.scale-btns
button.sel`) follows the existing small-pill-button visual language
already used for `.ov-controls button`, with the same `cycleB` accent
override pattern used elsewhere (e.g. `.cycleB .ov .d`).

### Grocery calculation

`effectiveGrocery(slot)` currently builds a `Set` of recipe keys "present"
this cycle and `qtyOf` sums `per[k]` for each present key with an
implicit factor of 1. This changes to a list of occurrences (one per
non-empty, non-Leftovers day) carrying that day's scale, so a recipe
appearing twice in one cycle (not intended by the "no repeats" rule but
not actually prevented by the editor) is still handled correctly — each
occurrence contributes independently instead of collapsing into one Set
membership check:

```js
function effectiveGrocery(slot){
  const occurrences = (mealPlan[slot]||[])
    .map((k,i)=> (k && k!==LEFTOVERS) ? {key:k, scale:(mealScale[slot][i]||1)} : null)
    .filter(Boolean);
  const sections=[], idx={};
  function sec(name){ if(idx[name]==null){ idx[name]=sections.length; sections.push({sec:name,items:[]}); } return sections[idx[name]]; }
  GITEMS.forEach(item=>{
    const qty=qtyOf(item, occurrences);
    if(qty<=0) return;
    const used = occurrences.filter(o=>item.per && item.per[o.key]!=null)
      .map(o=>R[o.key] && R[o.key].title).filter(Boolean);
    sec(item.sec).items.push({item, qty, used});
  });
  return sections;
}
function qtyOf(item, occurrences){
  if(item.always) return 1;
  let q=0; const per=item.per||{};
  occurrences.forEach(o=>{ if(per[o.key]!=null) q += per[o.key]*o.scale; });
  if(q<=0) return 0;
  return item.divisible ? Math.ceil(q*2)/2 : Math.ceil(q-1e-9);
}
```

With every `mealScale` entry defaulting to `1`, this produces identical
totals to today for anyone who never touches the new control —
purely additive behavior.

`item.always` entries (pantry staples marked "as needed") are untouched
by scaling, by design — a 2x taco night doesn't imply "2x salt."

### Recipe-page note

`recipeCard(k, dayNo, date, scale)` and `showRecipeDetail(k, scale)` both
gain an optional `scale` parameter (default `1`). When `scale !== 1`, a
small note renders under "Serves N": `👥 2x tonight — grocery list
already adjusted.` The underlying ingredient/step text is never rewritten
(it's free-form prose pulled from `recipeData(k)`, which can't be safely
auto-scaled).

- `recipesHTML(slot)` passes `mealScale[slot][i]` into `recipeCard`
  (it already has `i` in scope).
- `dayCard`'s read-mode click handler changes from `openRecipe('${k}')`
  to `openRecipe('${k}','${slot}',${i})`.
- `openRecipe(k, slot, i)`: when `slot`/`i` are given, look up
  `mealScale[slot][i]` and pass it through to `showRecipeDetail`;
  otherwise (opened from the "All Recipes" index tab, with no day
  context) default to `1` — no note shown when just browsing the recipe
  catalog in the abstract.

### Sync

Follows the exact same period-keyed pattern as `mealPlan`/`meals`:

- `applyRemote`: add `remoteScaleAll = s.mealScale || {}`; derive
  `mealScale.A`/`mealScale.B` via `normalizeScale(remoteScaleAll[key])`;
  persist locally via `saveScale`.
- `syncPush`: add `remoteScaleAll[CTX.A.key]=mealScale.A;
  remoteScaleAll[CTX.B.key]=mealScale.B;` and include `mealScale:
  remoteScaleAll` in the serialized `state`.

---

## Testing approach

This app has no test harness (single static HTML file, no build step) and
syncs to a **live, shared Firebase database** — see the standing project
note: preview evals must stay read-only against the synced preview, since
any mutating call (`changeMeal`, `moveMeal`, `toggleCheck`, `setScale`,
saving a recipe edit, etc.) pushes to real shared data within ~400ms with
no undo.

Verification plan for implementation:

1. Serve the file locally (`python -m http.server`, per the existing
   `.claude/launch.json`-style config) and, before exercising any
   mutating path, stub out sync in the browser console
   (`syncRef = null;`) so nothing reaches the live database.
2. Exercise all four features against that stubbed instance: scroll
   within an open meal picker; check off items across categories and
   confirm the Purchased pile behavior and subtotal math; edit a
   recipe's ingredients/steps, confirm the `## ` convention parses
   correctly and Reset-to-original works; set 1.5x/2x/3x on a day, move
   it, remove it, send it to the other cycle, and confirm the grocery
   totals and the recipe-page note track correctly through each
   mutation.
3. Only after local, stubbed verification passes does real device
   testing (with live sync) make sense, and only by deliberately
   choosing to leave sync on for that one check.

## Out of scope

- Recipe title, source link, and serving-size text remain fixed
  (non-editable).
- No guest-count input — scaling is fixed-option (1x/1.5x/2x/3x), not an
  arbitrary number.
- No automatic rewriting of ingredient/step prose to reflect a scaled
  quantity.
- No per-item check-order tracking for the Purchased pile ordering.
