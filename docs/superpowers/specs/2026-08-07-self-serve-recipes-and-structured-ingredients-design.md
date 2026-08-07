# Self-Serve Recipes + Structured Ingredient/Grocery Model — Design

Date: 2026-08-07
File under change: `index.html` (single-file app)
Supersedes/extends: the recipe-editing portion of
`docs/superpowers/specs/2026-07-29-meal-plan-editing-enhancements-design.md`
(Tasks 3–5 in `docs/superpowers/plans/2026-07-30-meal-plan-editing-enhancements.md`).
That work added a free-text ingredient/step editor; this design replaces its
data model with a structured one and adds the ability to create whole new
recipes.

## Summary

Today, a recipe's displayed ingredients (prose, `R[k].ing`) and its
contribution to the grocery list (`GITEMS[i].per[recipeKey] = amount`) are
two independent, hand-maintained structures. Editing a recipe's ingredient
text (as the current editor allows) never changes what the grocery list
buys — confirmed directly: a real edit to Spicy Thai Noodles this past week
(added chicken breast, swapped linguine for rice noodles) changed the
recipe page but the grocery list still asks for linguine and never adds
chicken breast.

This design:

1. Unifies the two structures. Each ingredient line becomes
   `{text, item, qty}` — the item name it's linked to (or `null`) and how
   much of it this recipe needs. `GITEMS` stops carrying per-recipe amounts
   entirely; a recipe's own ingredient list is the only source of truth for
   its grocery contribution.
2. Migrates all 21 built-in recipes to this shape in one pass, preserving
   every current grocery quantity exactly (verified by snapshot diff), and
   fixes the Spicy Thai Noodles gap as part of that migration.
3. Replaces the free-text ingredient editor with a structured builder (used
   identically for editing an existing recipe or creating a new one), which
   can create brand-new catalog items inline when an ingredient isn't in
   the catalog yet.
4. Adds full recipe creation: a "➕ Add a recipe" flow producing a recipe
   that behaves identically to the 21 built-in ones everywhere (meal
   picker, Cycle A/B tabs, grocery math), with edit and delete.

Guiding principle throughout: the Recipes, Grocery, and Prices tabs are one
system over shared data, not separate features that happen to sit in the
same file. Anything a built-in recipe or catalog item can do, a
user-created one can do too — full per-store pricing, full participation
in grocery math, no second-class "custom" variant of a capability that
already exists.

---

## 1. Data model

### 1.1 Ingredient row shape

Every ingredient group keeps its existing `{heading, rows}` shape (heading
may be `""`), but each row becomes an object instead of a plain string:

```js
{ text: "1 lb chicken breast, sliced", item: "Chicken breast", qty: 1 }
{ text: "Salt and pepper, to taste",   item: null,             qty: 0 }
```

- `text` is exactly what renders on the recipe page — nothing about the
  reading experience changes.
- `item` is a catalog item name (matches `GITEMS`/`customItems` entries by
  their `.n` field, the same identity scheme the catalog already uses
  everywhere — `storeSel[item.n]`, `prices[item.n]`, etc.) or `null` if this
  line isn't tracked for shopping.
- `qty` is "how much of one purchase-unit of `item` does *one* 1x-scale
  occurrence of this recipe need" — exactly the same semantic `per[key]`
  amounts already carry today, just relocated.

`R[k].ing` and `recipeEdits[k].ing` (and, once this ships, `customRecipes`)
all use this one shape. There is no longer a "free-text-only, no link"
shape coexisting with a "structured" shape — every recipe, built-in or
custom, edited or not, uses the same rows.

### 1.2 `GITEMS` loses `per`

```js
{n:"Chicken breast", sec:"🥩 Meat & Seafood", staple:"chicken_breast", unit:"lb", divisible:true}
```

`always` items (the single "Spice-rack restock" catch-all) are unaffected —
they already return `qty:1` unconditionally, with no dependency on which
recipes are planned, so they need no ingredient-row links at all.

### 1.3 Unified catalog

```js
let customItems = load('mp_customItems_v1', []);  // [{n, sec, unit, divisible, always, price}]
function saveCustomItems(){ save('mp_customItems_v1', customItems); }
function allItems(){ return GITEMS.concat(customItems); }
```

Every place that currently iterates `GITEMS` directly (`buildCatalog`,
`effectiveGrocery`, the Prices-tab catalog table) switches to `allItems()`.

There's no "flat price" concept anywhere in this app to fall back on —
every non-staple catalog item already gets full 4-store price comparison
today (`prices[it.n][store]` for all of Costco/Kroger/Aldi/Publix, plus a
selected store and unit, via `buildCatalog`/`initPrices`/`ensureDefaults`).
A custom item is just another non-staple entry in that same catalog — once
`buildCatalog` folds `customItems` into what it scans (the one-line switch
to `allItems()` above), `initPrices`/`ensureDefaults` pick it up
automatically and it gets the identical per-store comparison table as
"Bell peppers" or "Pork loin roast" do. No separate pricing path, no
second-class treatment — a custom item shows up on the Prices tab exactly
like a built-in one, immediately editable per store. (`STAPLE_DEFAULTS` —
the *separate* mechanism where several catalog rows like the three
shredded-cheese variants share one priced key — stays built-in-only; a
custom item doesn't need to join an existing staple group to get per-store
pricing, since that's already universal.)

A new item's `n` must be unique against `allItems()` at creation time.

v1 has no deletion for catalog items (custom or built-in) — only addition
and price edits, matching today's Prices-tab capability. Deleting an item
that some recipe still links to would need to decide what happens to that
link; punting on this avoids designing dangling-reference handling for a
feature nobody asked for yet.

### 1.4 Custom recipes

```js
let customRecipes = load('mp_customRecipes_v1', {});
// { [key]: {title, serves, ing, steps, note?} } — same shape as R[k], no src/url required
function saveCustomRecipes(){ save('mp_customRecipes_v1', customRecipes); }
```

Keys are generated once at creation (`custom-<slug-of-title>-<seq>`,
mirroring the existing `custSeq` pattern for custom grocery items) and
never change.

A single helper resolves "does this key exist, and where":

```js
function baseRecipe(k){ return R[k] || customRecipes[k]; }
```

Everywhere the code currently does `const r=R[k]` for a recipe's fixed
fields (title/serves/src/url/note), it switches to `baseRecipe(k)`. `R[k]`
keeps meaning "one of the 21 built-in recipes" — custom recipes never get
merged into `R` itself, keeping "is this one of mine to delete" a simple
`k in customRecipes` check.

`recipeEdits[k]` continues to mean "an override layered on top of a
built-in recipe" (Task 3/4's existing mechanism, unchanged in spirit — only
its `ing` shape changes per §1.1). A custom recipe has no "original" to
layer on top of, so editing one **overwrites `customRecipes[k]` directly**
— no separate override object for custom recipes. `recipeData(k)` becomes:

```js
function recipeData(k){
  if(customRecipes[k]) return customRecipes[k];
  const base=R[k], edit=recipeEdits[k];
  return edit ? Object.assign({}, base, {ing: edit.ing||base.ing, steps: edit.steps||base.steps}) : base;
}
```

`ALL_RECIPE_KEYS` (currently a `const` computed once from `Object.keys(R)`)
becomes a function, `allRecipeKeys()`, recomputed each time it's needed
(cheap — recipe counts are small) so newly-created/deleted custom recipes
show up immediately in the meal picker and the All Recipes index without a
page reload.

---

## 2. Migrating the 21 built-in recipes

This is a one-time, one-way data transformation of `R` itself (not an
overlay) — after this ships, `R[k].ing` is already in the new row shape for
every built-in recipe, and `GITEMS` no longer has any `per` fields.

**Algorithm**, applied per recipe key `k`:

1. Take `k`'s current *effective* ingredient text — `recipeData(k)` as it
   exists today, not necessarily the original hardcoded `R[k]` — so any
   edit already made (e.g. Spicy Thai Noodles) is preserved as the
   migration's starting point, not silently reverted.
2. Collect every `GITEMS` entry whose `per[k]` is defined (skipping
   `always` items, which never need a link). This is the exact set of
   `{itemName, qty}` pairs `k` currently pulls into the grocery list.
3. For each pair, look for exactly one ingredient text line that plausibly
   refers to that item (case-insensitive containment on the item's name,
   normalizing obvious parentheticals — e.g. "Chicken broth (low-sodium)"
   still matches a line containing "chicken broth"). If found, that line
   becomes `{text: <original line>, item: itemName, qty}`. Every other line
   becomes `{text: <original line>, item: null, qty: 0}` — unchanged
   display, no link. If more than one line plausibly matches the same
   item, don't guess — pick based on direct manual read-through of that
   specific recipe (there are only 21; each can be eyeballed) rather than
   an automated tie-break rule.
4. Any pair that doesn't match a line (rare — happens when the recipe's
   prose never explicitly names the ingredient the way the catalog entry
   is named) gets appended as a new row, using the item's own name as the
   line text, so the grocery contribution isn't silently dropped. Flag
   each such case in the implementation plan for a manual sanity check
   rather than assuming the auto-generated text reads naturally.
5. **Known fix folded into this pass**: for `thainoodles`, step 1's
   "effective text" already includes the user's real edit (chicken breast
   added, linguine → rice noodles). Link the rice-noodle line to the
   existing "Flat rice noodles" catalog item (qty 1, matching its current
   `pack` unit — same item `padthai` already uses) instead of "Linguine",
   and link the chicken-breast line to "Chicken breast" at qty 1 (lb). The
   stale `Linguine → thainoodles:1` association is dropped as part of
   removing all `per` maps.

**Verification**: before touching any code, snapshot
`effectiveGrocery('A')` and `effectiveGrocery('B')` (item names + qty +
price, for the plan currently loaded) under the *old* engine. After the
migration lands, take the same snapshot under the *new* engine and diff.
Every recipe except `thainoodles` must produce byte-identical quantities
and prices — `thainoodles` is expected to change (that's the intended fix,
not a regression) and should be checked by hand against the specific new
lines (chicken breast now appears; rice noodles/"Flat rice noodles"
replaces linguine; nothing else on that recipe's line moves).

---

## 3. Grocery calculation rewrite

`effectiveGrocery`/`qtyOf` currently iterate `GITEMS` and ask each item "how
much do the planned recipes need of you" via its own `per` map. That
inverts to: iterate the *planned occurrences*, read each one's own linked
rows, and accumulate into a per-item-name total.

```js
function effectiveGrocery(slot){
  const occurrences=(mealPlan[slot]||[])
    .map((k,i)=> (k && k!==LEFTOVERS) ? {key:k, scale:(mealScale[slot][i]||1)} : null)
    .filter(Boolean);
  const contrib={}, usedBy={};   // itemName -> qty sum, itemName -> Set of recipe titles
  occurrences.forEach(o=>{
    flattenRows(recipeData(o.key).ing).forEach(row=>{
      if(!row.item) return;
      contrib[row.item] = (contrib[row.item]||0) + row.qty*o.scale;
      (usedBy[row.item] || (usedBy[row.item]=new Set())).add(o.key);
    });
  });
  const sections=[], idx={};
  function sec(name){ if(idx[name]==null){ idx[name]=sections.length; sections.push({sec:name,items:[]}); } return sections[idx[name]]; }
  allItems().forEach(item=>{
    const rawQty = item.always ? 1 : (contrib[item.n]||0);
    if(rawQty<=0) return;
    const qty = item.always ? 1 : (item.divisible ? Math.ceil(rawQty*2)/2 : Math.ceil(rawQty-1e-9));
    const used = item.always ? [] : [...(usedBy[item.n]||[])].map(k=>baseRecipe(k)?.title).filter(Boolean);
    sec(item.sec).items.push({item, qty, used});
  });
  return sections;
}
```

`flattenRows(ing)` is a small helper that walks the `{heading, rows}` (or
however groups end up shaped post-migration — see the implementation plan
for the exact literal structure) and returns a flat array of `{item,qty}`
rows, so `effectiveGrocery` doesn't need to know about heading grouping.

This is the *only* grocery-computation path once migration lands — there
is no fallback to a legacy per-map anywhere, per your explicit call to
migrate everything at once rather than keep two systems running.

---

## 4. Ingredient builder UI

One component, used for editing an existing recipe (built-in or custom)
and for creating a new one.

- **Rows**: a text input (the display line) + a linked-item control. The
  linked-item control defaults to "— no grocery link —"; picking a
  different value reveals a quantity number input (pre-labeled with that
  item's unit, e.g. "qty (lb)"). Selecting **"+ Create new item…"** from
  the same control opens an inline mini-form: name, section (a `<select>`
  populated from the distinct `sec` values already in `allItems()`, plus
  an "Other…" option that reveals a free-text field), unit, two checkboxes
  ("fractional amounts allowed" / "always needed, not recipe-specific"),
  and a starting price. Saving it adds to `customItems` immediately and
  selects it for the current row.
- **Groups**: "+ Add ingredient" appends a row to the current group; "+ Add
  section heading" starts a new group. A row's own "✕" removes it.
- **Steps**: unchanged — still the plain-text, one-line-per-step textarea
  from the existing editor. Only ingredients change shape.
- **Escaping carries over unchanged.** The prior session's work went
  through two rounds of hardening to get free-text recipe content safely
  escaped (`escHTML`) everywhere it renders, including the textarea
  pre-fill breakout case. Every new render path this design adds — row
  `text` in the builder and in the read view, a new item's `n` when it's
  echoed back in the picker — must go through the same `escHTML` helper.
  New catalog item names in particular are user-authored text flowing into
  `innerHTML` for the first time in a *new* place (the item picker), so
  treat them with the same suspicion as recipe text, not as trusted data.
- **First edit of a built-in recipe** (post-migration, this only applies to
  the rare recipe not touched by the migration's auto-linking — i.e. one
  of the "appended as unlinked" cases from §2 step 4): the row already
  carries whatever the migration produced, so there's no separate
  "first-time" UX to design — migration already put every recipe into the
  same shape this builder edits.

---

## 5. New recipe creation

"➕ Add a recipe" (Recipes tab) opens: title, serves (free text, matching
the existing `serves` field's loose format like "4–6"), then the same
ingredient builder from §4 (starting with one empty group), then a steps
textarea. Saving generates a key, stores the recipe in `customRecipes`,
and it's immediately selectable everywhere a recipe key is used.

## 6. Editing and deleting custom recipes

Same detail page, same "✏️ Edit" entry point as built-in recipes. Since
there's no "original" for a custom recipe, there's no "Reset to original"
button — instead, a **"🗑️ Delete recipe"** action (with a confirm step,
since it's destructive and, unlike most of this app's edits, has no
correcting action once other devices sync the deletion). Deleting a recipe
that's currently planned on any day in either cycle clears that day back
to "no meal" (same handling `removeMeal` already does) rather than leaving
a dangling key in `mealPlan`.

---

## 7. Storage & sync

`customItems` and `customRecipes` are global, not period-keyed (like
`recipeEdits`/`prices`/`staplePrices`) — recipes and catalog items aren't
tied to a specific two-week cycle. Both follow the exact `load`/`save`
pattern already established, and both get added to the `applyRemote`/
`syncPush` pair alongside `recipeEdits`, following that same precedent
(Task 5 in the prior plan).

`recipeEdits[k].ing` changes shape (per §1.1) for any recipe a user has
*already* edited under the old free-text system — currently just
`thainoodles`. The migration (§2) reads that override as its starting text
(step 1) and writes the resulting structured rows straight into
`R.thainoodles.ing` itself. Once that's done, `recipeEdits.thainoodles`
must be deleted (not left in place) — otherwise `recipeData` would keep
layering the old, now-superseded free-text override on top of the freshly
migrated base, undoing the fix. No other recipe has a pre-existing
`recipeEdits` entry, so this cleanup is a one-line, one-key operation, not
a general migration path.

---

## 8. Testing approach

Same standing rule as before: no test framework, live Firebase sync, so
verification is manual against a locally-served copy with `syncRef=null`
set immediately after load, before any interaction.

Specific to this change:

- **Migration correctness**: the snapshot-diff described in §2 is the
  primary safety net — it must be run and pass before anything else is
  considered done, since a mistake here would silently change everyone's
  grocery totals.
- **New-item creation**: verify a newly created item appears immediately
  in the ingredient builder's picker, in the Prices tab, and (once linked
  to a planned recipe) in the grocery list with the entered price.
- **Custom recipe lifecycle**: create, plan it on a day, confirm its
  grocery contribution appears correctly (including under a 2x/3x serving
  scale), edit it, delete it, confirm the day it occupied reverts to "no
  meal."

## Out of scope

- Deleting or renaming existing catalog items (built-in or custom).
- Reordering ingredient rows/groups (add/remove only).
- Any change to how steps are entered or displayed.
- Any change to serving-scale, the Purchased-pile grocery UI, or the
  meal-picker — this design only touches the recipe/ingredient/catalog
  data model and the screens that create or edit that data.
