# Structured Ingredient Migration Implementation Plan (Part A of 2)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **This is Part A of a two-plan sequence.** Part B (`2026-08-07-self-serve-recipe-builder.md`) adds the actual self-serve recipe/ingredient-builder UI on top of the data model this plan produces. **Do not push to origin after Part A alone** — this plan removes the old free-text recipe-edit UI (it's incompatible with the new data shape) and Part B doesn't land its replacement until later, so recipe editing is intentionally non-functional in the gap between the two. Run both plans back-to-back in one session before pushing.

**Goal:** Replace the two-structure ingredient model (free-text `R[k].ing` + `GITEMS[i].per[recipeKey]`) with one structure per recipe (`{text, item, qty}` rows), migrating all 21 built-in recipes — including the three you've already hand-edited in production (Spicy Thai Noodles, Pad Thai, Ground Beef Taco Bowls) — with zero change to any grocery total except the ones that are supposed to change.

**Architecture:** `GITEMS` loses its `per` maps entirely and becomes a pure item catalog. Every recipe's `ing` becomes `[{heading, rows:[{text,item,qty}]}]`. `effectiveGrocery` inverts from "ask each catalog item which recipes use it" to "ask each planned recipe what it needs, sum by item name."

**Tech Stack:** Vanilla JS, no build step — same as the rest of `index.html`.

**Spec:** `docs/superpowers/specs/2026-08-07-self-serve-recipes-and-structured-ingredients-design.md`

---

## Note on testing in this codebase

No automated test suite exists. Verification is manual, against a locally-served copy, with `syncRef=null` set immediately after load before any interaction — this app syncs to a live, shared Firebase Realtime Database. The one exception is Task 1 below, which is a **read-only** query against the live production data (needed to capture the real "before" baseline) — do not set `syncRef=null` for that one read, but do not call any mutating function either.

## Note on line numbers

Line numbers below reflect `index.html` as of commit `3066b52` (current `main`, after the Walmart store addition). Locate each edit by its exact code content, not by line number, once earlier tasks in this plan have landed.

---

### Task 1: Snapshot current grocery totals (pre-migration baseline)

**Files:** none (read-only verification)

- [ ] **Step 1: Capture the live baseline**

Open `https://mg-aretestays.github.io/dinner-meal-plan/` in a browser. Do **not** set `syncRef=null` for this step — you need the real synced state. Do not call any function that mutates state (no `changeMeal`, `toggleCheck`, `setScale`, etc.) — this is a pure read.

Run in the console:

```js
JSON.stringify({A: effectiveGrocery('A'), B: effectiveGrocery('B')}, (k,v)=> k==='item' ? undefined : v, 1)
```

(The replacer drops the bulky `item` object from each entry so the output stays focused on `{qty, used}` per item name — you'll cross-reference names against the `sec`/`n` fields separately if needed.)

Save the full output somewhere durable (a local scratch file, a note) — this is Task 6's baseline for the diff. Also separately record, for cross-reference:

```js
JSON.stringify(Object.keys(recipeEdits))
```

Expected right now: `["thainoodles","tacobowls","padthai"]` — if this differs, STOP and report BLOCKED, since the migration in Task 3 is written against exactly these three edited recipes; a fourth edit would need to be incorporated too.

- [ ] **Step 2: No commit** — this step produces no code change, just the saved baseline text for Task 6.

---

### Task 2: Restructure `GITEMS` into a pure catalog (no `per`), add the missing Pad Thai Sauce item

**Files:**
- Modify: `index.html` (the `GITEMS` array, currently lines 561–662; function `buildCatalog`, currently lines 737–746)

**Context:** Your Pad Thai edit replaced five homemade-sauce ingredients (fish sauce, light soy sauce, light brown sugar, rice vinegar, sriracha, creamy peanut butter) with one bottled "Pad Thai Sauce" that doesn't exist in the catalog yet. This task adds it. (Light soy sauce and Sriracha stay in the catalog — other recipes still use them; Fish sauce, Rice vinegar, and Creamy peanut butter stay too, they just end up with zero recipe links after this migration, which is expected and harmless — see Task 6.)

- [ ] **Step 1: Replace the entire `GITEMS` array**

Current code (lines 561–662) is the full array with `per:{...}` on every non-`always` entry. Replace the **entire array** with:

```js
const GITEMS = [
  // 🥩 Meat & Seafood
  {n:"Chicken breast", sec:"🥩 Meat & Seafood", staple:"chicken_breast", unit:"lb", divisible:true},
  {n:"Ground beef", sec:"🥩 Meat & Seafood", staple:"ground_beef", unit:"lb", divisible:true},
  {n:"Pork tenderloin", sec:"🥩 Meat & Seafood", staple:"pork_tenderloin", unit:"lb", divisible:true},
  {n:"Pork loin roast (boneless)", sec:"🥩 Meat & Seafood", price:7.49, unit:"roast"},
  {n:"Ground spicy Italian chicken sausage", sec:"🥩 Meat & Seafood", price:3.99, unit:"pack"},
  {n:"Bacon", sec:"🥩 Meat & Seafood", staple:"bacon", unit:"pack"},
  {n:"Breakfast sausage", sec:"🥩 Meat & Seafood", price:3.99, unit:"lb"},
  {n:"Breaded chicken strips (frozen)", sec:"🥩 Meat & Seafood", price:5.99, unit:"box"},
  {n:"Frozen popcorn chicken / nuggets", sec:"🥩 Meat & Seafood", price:5.99, unit:"bag"},
  {n:"Frozen meatballs", sec:"🥩 Meat & Seafood", price:5.49, unit:"bag"},
  {n:"Pepperoni", sec:"🥩 Meat & Seafood", price:3.49, unit:"pkg"},
  // 🥦 Produce
  {n:"Yellow onions", sec:"🥦 Produce", price:0.50, unit:"onion"},
  {n:"Red onion", sec:"🥦 Produce", price:0.79, unit:"onion"},
  {n:"Bell peppers", sec:"🥦 Produce", price:1.00, unit:"bell pepper"},
  {n:"Broccoli florets", sec:"🥦 Produce", price:1.25, unit:"crown"},
  {n:"Garlic", sec:"🥦 Produce", price:0.50, unit:"head"},
  {n:"Fresh ginger", sec:"🥦 Produce", price:0.79, unit:"knob"},
  {n:"Carrots", sec:"🥦 Produce", price:1.49, unit:"bag"},
  {n:"Iceberg lettuce", sec:"🥦 Produce", price:1.99, unit:"head"},
  {n:"Tomatoes", sec:"🥦 Produce", price:0.50, unit:"tomato"},
  {n:"Green onions", sec:"🥦 Produce", price:0.79, unit:"bunch"},
  {n:"Fresh cilantro", sec:"🥦 Produce", price:0.79, unit:"bunch"},
  {n:"Celery", sec:"🥦 Produce", price:1.79, unit:"bunch"},
  {n:"Yukon gold / russet potatoes", sec:"🥦 Produce", price:2.99, unit:"bag"},
  {n:"Kale (Tuscan or curly)", sec:"🥦 Produce", price:1.99, unit:"bunch"},
  {n:"Limes", sec:"🥦 Produce", price:0.25, unit:"lime"},
  {n:"Lemon", sec:"🥦 Produce", price:0.50, unit:"lemon"},
  {n:"Fresh parsley", sec:"🥦 Produce", price:0.79, unit:"bunch"},
  {n:"Bean sprouts", sec:"🥦 Produce", price:1.49, unit:"bag"},
  // 🧀 Dairy & Refrigerated
  {n:"Butter", sec:"🧀 Dairy & Refrigerated", staple:"butter", unit:"box"},
  {n:"Milk", sec:"🧀 Dairy & Refrigerated", staple:"milk", unit:"gallon"},
  {n:"Heavy cream", sec:"🧀 Dairy & Refrigerated", price:2.49, unit:"pint"},
  {n:"Monterey Jack cheese", sec:"🧀 Dairy & Refrigerated", price:2.99, unit:"block"},
  {n:"Sharp cheddar cheese", sec:"🧀 Dairy & Refrigerated", price:2.49, unit:"block"},
  {n:"Shredded cheddar", sec:"🧀 Dairy & Refrigerated", staple:"shred_cheese", unit:"bag"},
  {n:"Shredded mozzarella", sec:"🧀 Dairy & Refrigerated", staple:"shred_cheese", unit:"bag"},
  {n:"Mexican-blend shredded cheese", sec:"🧀 Dairy & Refrigerated", staple:"shred_cheese", unit:"bag"},
  {n:"Grated Parmesan", sec:"🧀 Dairy & Refrigerated", price:3.49, unit:"container"},
  {n:"American cheese singles", sec:"🧀 Dairy & Refrigerated", price:2.49, unit:"pack"},
  {n:"Sour cream", sec:"🧀 Dairy & Refrigerated", price:1.99, unit:"tub"},
  {n:"Eggs", sec:"🧀 Dairy & Refrigerated", staple:"eggs", unit:"dozen"},
  // 🍝 Pasta, Rice, Sauces & Canned
  {n:"Spaghetti", sec:"🍝 Pasta, Rice, Sauces & Canned", staple:"pasta", unit:"box"},
  {n:"Fettuccine", sec:"🍝 Pasta, Rice, Sauces & Canned", staple:"pasta", unit:"box"},
  {n:"Linguine", sec:"🍝 Pasta, Rice, Sauces & Canned", staple:"pasta", unit:"box"},
  {n:"Elbow macaroni", sec:"🍝 Pasta, Rice, Sauces & Canned", staple:"pasta", unit:"box"},
  {n:"Cavatappi/macaroni", sec:"🍝 Pasta, Rice, Sauces & Canned", price:1.25, unit:"box"},
  {n:"Flat rice noodles", sec:"🍝 Pasta, Rice, Sauces & Canned", price:2.49, unit:"pack"},
  {n:"Soba/rice noodles", sec:"🍝 Pasta, Rice, Sauces & Canned", price:1.99, unit:"pack"},
  {n:"Long-grain white rice", sec:"🍝 Pasta, Rice, Sauces & Canned", price:1.99, unit:"bag"},
  {n:"Pasta sauce", sec:"🍝 Pasta, Rice, Sauces & Canned", price:1.49, unit:"jar"},
  {n:"Alfredo sauce", sec:"🍝 Pasta, Rice, Sauces & Canned", price:2.49, unit:"jar"},
  {n:"Pizza tomato sauce", sec:"🍝 Pasta, Rice, Sauces & Canned", price:1.00, unit:"can"},
  {n:"Salsa", sec:"🍝 Pasta, Rice, Sauces & Canned", price:1.99, unit:"jar"},
  {n:"Diced green chilies", sec:"🍝 Pasta, Rice, Sauces & Canned", price:0.99, unit:"can"},
  {n:"Black beans", sec:"🍝 Pasta, Rice, Sauces & Canned", price:0.99, unit:"can"},
  {n:"Kroger queso dip", sec:"🍝 Pasta, Rice, Sauces & Canned", price:2.99, unit:"jar"},
  {n:"Sweet corn", sec:"🍝 Pasta, Rice, Sauces & Canned", price:1.00, unit:"can"},
  {n:"Basil pesto", sec:"🍝 Pasta, Rice, Sauces & Canned", price:3.49, unit:"jar"},
  {n:"Heinz classic chicken gravy", sec:"🍝 Pasta, Rice, Sauces & Canned", price:1.99, unit:"jar"},
  {n:"Instant mashed potatoes", sec:"🍝 Pasta, Rice, Sauces & Canned", price:1.99, unit:"box"},
  {n:"Tomato paste", sec:"🍝 Pasta, Rice, Sauces & Canned", price:0.89, unit:"can"},
  {n:"Chicken broth (low-sodium)", sec:"🍝 Pasta, Rice, Sauces & Canned", price:1.49, unit:"carton"},
  {n:"Beef broth", sec:"🍝 Pasta, Rice, Sauces & Canned", price:1.49, unit:"carton"},
  {n:"Pad Thai Sauce", sec:"🍝 Pasta, Rice, Sauces & Canned", price:3.49, unit:"bottle"},
  // 🫙 Asian Pantry & Condiments
  {n:"Light soy sauce", sec:"🫙 Asian Pantry & Condiments", price:2.49, unit:"bottle"},
  {n:"Dark soy sauce", sec:"🫙 Asian Pantry & Condiments", price:2.99, unit:"bottle"},
  {n:"Shaoxing wine (or dry sherry)", sec:"🫙 Asian Pantry & Condiments", price:3.49, unit:"bottle"},
  {n:"Cornstarch", sec:"🫙 Asian Pantry & Condiments", price:1.29, unit:"box"},
  {n:"Hoisin sauce", sec:"🫙 Asian Pantry & Condiments", price:2.49, unit:"jar"},
  {n:"Toasted sesame oil", sec:"🫙 Asian Pantry & Condiments", price:3.49, unit:"bottle"},
  {n:"Chili paste (sambal)", sec:"🫙 Asian Pantry & Condiments", price:2.99, unit:"jar"},
  {n:"Sriracha", sec:"🫙 Asian Pantry & Condiments", price:2.99, unit:"bottle"},
  {n:"Dry roasted peanuts", sec:"🫙 Asian Pantry & Condiments", price:2.49, unit:"jar"},
  {n:"Sesame seeds", sec:"🫙 Asian Pantry & Condiments", price:2.00, unit:"jar"},
  {n:"Honey", sec:"🫙 Asian Pantry & Condiments", price:3.49, unit:"bottle"},
  {n:"Worcestershire sauce", sec:"🫙 Asian Pantry & Condiments", price:2.99, unit:"bottle"},
  {n:"Fish sauce", sec:"🫙 Asian Pantry & Condiments", price:2.99, unit:"bottle"},
  {n:"Rice vinegar", sec:"🫙 Asian Pantry & Condiments", price:2.29, unit:"bottle"},
  {n:"Creamy peanut butter", sec:"🫙 Asian Pantry & Condiments", price:2.49, unit:"jar"},
  {n:"Apple cider vinegar", sec:"🫙 Asian Pantry & Condiments", price:1.99, unit:"bottle"},
  {n:"Vegetable/peanut oil", sec:"🫙 Asian Pantry & Condiments", price:2.99, unit:"bottle"},
  {n:"Olive oil", sec:"🫙 Asian Pantry & Condiments", price:4.49, unit:"bottle"},
  {n:"Buffalo sauce", sec:"🫙 Asian Pantry & Condiments", price:2.49, unit:"bottle"},
  {n:"Ranch dressing", sec:"🫙 Asian Pantry & Condiments", price:2.49, unit:"bottle"},
  // 🌮 Bread, Tortillas & Mix
  {n:"Bread/all-purpose flour", sec:"🌮 Bread, Tortillas & Mix", staple:"flour", unit:"bag"},
  {n:"Instant yeast", sec:"🌮 Bread, Tortillas & Mix", price:0.99, unit:"packet"},
  {n:"Large burrito tortillas", sec:"🌮 Bread, Tortillas & Mix", staple:"tortillas", unit:"pack"},
  {n:"Small flour tortillas", sec:"🌮 Bread, Tortillas & Mix", staple:"tortillas", unit:"pack"},
  {n:"Corn tortillas / tostadas", sec:"🌮 Bread, Tortillas & Mix", price:1.99, unit:"pack"},
  {n:"Hamburger buns", sec:"🌮 Bread, Tortillas & Mix", price:1.99, unit:"pack"},
  {n:"Pancake mix", sec:"🌮 Bread, Tortillas & Mix", price:2.49, unit:"box"},
  {n:"Pancake/maple syrup", sec:"🌮 Bread, Tortillas & Mix", price:2.99, unit:"bottle"},
  {n:"Taco seasoning", sec:"🌮 Bread, Tortillas & Mix", price:1.00, unit:"packet"},
  // 🧂 Spice Rack
  {n:"Spice-rack restock", sec:"🧂 Spice Rack (likely on hand)", price:3.00, unit:"trip", always:true, note:"Buy what you're out of — pepper, paprika, garlic/onion powder, thyme, cumin, cayenne, sugar, Italian seasoning, red pepper flakes"}
];
```

This is byte-identical to the current array except: every `per:{...}` field is deleted, and one new entry ("Pad Thai Sauce") is added in the Pasta/Rice/Sauces section. No item was removed, renamed, reordered, re-sectioned, re-priced, or re-unit'd.

- [ ] **Step 2: Add the unified-catalog helper**

Insert directly after the `GITEMS` array (before `function datasetForPeriod`):

```js
let customItems = load('mp_customItems_v1', []);   // [{n, sec, unit, divisible, always, price}] — user-created catalog items
function saveCustomItems(){ save('mp_customItems_v1', customItems); }
function allItems(){ return GITEMS.concat(customItems); }
```

- [ ] **Step 3: Switch `buildCatalog` to scan `allItems()`**

Current code:

```js
function buildCatalog(){
  const seen=new Set(), secMap=new Map();
  GITEMS.forEach(it=>{
    if(it.staple) return;          // staples are priced per-unit on their own table
    if(seen.has(it.n)) return; seen.add(it.n);
    if(!secMap.has(it.sec)) secMap.set(it.sec,[]);
    secMap.get(it.sec).push({n:it.n, def:it.price||0});
  });
  CATALOG = Array.from(secMap.entries()).map(([sec,items])=>({sec,items}));
}
```

Replace with:

```js
function buildCatalog(){
  const seen=new Set(), secMap=new Map();
  allItems().forEach(it=>{
    if(it.staple) return;          // staples are priced per-unit on their own table
    if(seen.has(it.n)) return; seen.add(it.n);
    if(!secMap.has(it.sec)) secMap.set(it.sec,[]);
    secMap.get(it.sec).push({n:it.n, def:it.price||0});
  });
  CATALOG = Array.from(secMap.entries()).map(([sec,items])=>({sec,items}));
}
```

(`customItems` is empty until Part B ships the UI to create one, so this is currently a no-op change in observed behavior — it just makes `buildCatalog` future-proof for when `customItems` has entries.)

- [ ] **Step 4: Manual verification**

Serve locally, console: `syncRef=null;`.

1. Go to the **Prices** tab. Expected: every item that was there before is still there, same sections, same prices — including the brand-new **"Pad Thai Sauce"** row in the Pasta/Rice/Sauces section, priced at $3.49, with the full Costco/Kroger/Aldi/Publix/Walmart comparison dropdown.
2. Run `GITEMS.some(x=>x.per)` in the console → expect `false` (confirms no `per` field survived anywhere).
3. Run `allItems().length === GITEMS.length` → expect `true` (confirms `customItems` is empty and `allItems()` degrades to `GITEMS` cleanly).

At this point the grocery lists will be **broken** (Cycle A/B Grocery tabs will show empty or wrong totals) — `effectiveGrocery`/`qtyOf` still reference the now-removed `per` maps and haven't been rewritten yet (Task 5). That's expected and fine; don't try to fix it in this task.

- [ ] **Step 5: Commit**

```bash
cd "C:/Users/Matt/GitHub/dinner-meal-plan"
git add index.html
git commit -m "Strip per-recipe amounts from GITEMS; add unified item catalog"
```

---

### Task 3: Migrate all 21 built-in recipes to the structured ingredient shape

**Files:**
- Modify: `index.html` (the entire `R` object, currently lines 392–549)

**Context — read this before touching code:** Three of these 21 recipes have live edits in production right now (confirmed via Task 1): `thainoodles`, `tacobowls`, `padthai`. Per the design spec, migration uses each recipe's *current effective text* as the source, not the original hardcoded text — so this task's `ing` for those three is built from the live edit, not from what's currently sitting in the `R` object in this file. The exact live text (pulled read-only from production) is reproduced faithfully below for each of those three, with the known intentional changes (chicken breast + rice noodles for Thai Noodles; sauce swap for Pad Thai; the missing sour cream/oil lines for Taco Bowls) folded in as structured links.

Every ingredient line's original wording is preserved as `text` — nothing about what you read on the recipe page changes, except: two lines (in `stirfry` and `thainoodles`) that named multiple garnish ingredients on one line are split into one line per ingredient, since this data model links one catalog item per row; three lines are newly appended (`breakfast`'s maple syrup, `tacobowls`'s sour cream and oil, `zuppa`'s oil, `porkloin`'s oil) because the original recipe text never explicitly named an ingredient that the old `GITEMS.per` map nonetheless charged the recipe for — these are flagged individually below so you can sanity-check the wording reads naturally.

- [ ] **Step 1: Replace the entire `R` object**

Current code is the full `R` object (lines 392–549), using the old `ing:[[heading, ...strings]]` shape. Replace the **entire object** with:

```js
const R = {
  bpc:{title:"Black Pepper Chicken", serves:"4", src:"omnivorescookbook.com", url:"https://omnivorescookbook.com/black-pepper-chicken/",
    ing:[
      {heading:"Chicken", rows:[
        {text:"1 lb chicken breasts (or thighs), sliced against the grain into 1/4\" thick pieces", item:"Chicken breast", qty:1}
      ]},
      {heading:"Marinade", rows:[
        {text:"1 Tbsp light soy sauce", item:null, qty:0},
        {text:"1 Tbsp Shaoxing wine (or dry sherry)", item:null, qty:0},
        {text:"1 Tbsp cornstarch", item:null, qty:0}
      ]},
      {heading:"Sauce", rows:[
        {text:"1/2 cup chicken broth", item:"Chicken broth (low-sodium)", qty:0.2},
        {text:"2 Tbsp light soy sauce", item:"Light soy sauce", qty:0.1},
        {text:"2 Tbsp Shaoxing wine", item:"Shaoxing wine (or dry sherry)", qty:0.1},
        {text:"2 tsp dark soy sauce", item:"Dark soy sauce", qty:0.05},
        {text:"1 Tbsp cornstarch", item:"Cornstarch", qty:0.2},
        {text:"1 1/2 Tbsp sugar", item:null, qty:0},
        {text:"2 tsp coarsely ground black pepper", item:null, qty:0},
        {text:"1/8 tsp salt", item:null, qty:0}
      ]},
      {heading:"Stir fry", rows:[
        {text:"2 Tbsp peanut oil (or vegetable oil)", item:"Vegetable/peanut oil", qty:0.1},
        {text:"1 Tbsp minced ginger", item:"Fresh ginger", qty:0.5},
        {text:"2 cloves garlic, minced", item:"Garlic", qty:0.15},
        {text:"1/2 white onion, chopped", item:"Yellow onions", qty:0.5},
        {text:"2 bell peppers, chopped", item:"Bell peppers", qty:2}
      ]}
    ],
    steps:["Combine chicken, soy sauce, Shaoxing wine, and cornstarch in a medium bowl. Gently mix by hand until the chicken is coated in a thin layer. Marinate 10–15 minutes.",
      "Combine all the sauce ingredients in a small bowl. Mix well and set aside.",
      "Heat 1 Tbsp oil in a large skillet over medium-high until hot. Add the chicken and spread into a single layer. Sear ~1 minute until lightly browned, flip, cook 30 sec–1 min until both sides are browned but still a bit pink inside. Transfer to a plate.",
      "Add the remaining 1 Tbsp oil. Add ginger and garlic; stir until fragrant. Add onion and peppers and stir-cook 20 seconds.",
      "Stir the sauce until the cornstarch is fully dissolved, pour into the skillet, and stir immediately until it thickens enough to coat a spoon. Add the chicken back and toss to coat. Turn off heat and transfer everything to a plate so it stops cooking.",
      "Serve hot over rice."]},
  tacos:{title:"Slow-Cooker Shredded Chicken Tacos", serves:"6", src:"slowcookermeals.com", url:"https://slowcookermeals.com/slow-cooker-shredded-chicken-tacos/",
    ing:[
      {heading:"", rows:[
        {text:"2 lbs boneless, skinless chicken breasts (about 4 medium)", item:"Chicken breast", qty:2},
        {text:"1 (16 oz) jar of your favorite salsa", item:"Salsa", qty:1},
        {text:"1 (1.5 oz) packet taco seasoning", item:"Taco seasoning", qty:1},
        {text:"1 tsp black pepper", item:null, qty:0},
        {text:"1 (4 oz) can diced green chilies", item:"Diced green chilies", qty:1},
        {text:"Tortillas, to serve", item:"Small flour tortillas", qty:1}
      ]},
      {heading:"Toppings", rows:[
        {text:"Cilantro", item:"Fresh cilantro", qty:0.3},
        {text:"Diced tomatoes", item:null, qty:0},
        {text:"Monterey Jack & cheddar cheese", item:null, qty:0},
        {text:"Kroger queso", item:"Kroger queso dip", qty:0.5},
        {text:"Red onion", item:"Red onion", qty:1}
      ]}
    ],
    steps:["Place chicken breasts in the slow cooker. Cover with salsa and sprinkle taco seasoning, black pepper, and green chilies over the top.",
      "Cover and cook on low 7–8 hours or high 2½–3 hours. When done, shred the chicken with two forks and stir into the sauce.",
      "Serve on tortillas with your favorite toppings."]},
  padthai:{title:"Chicken Pad Thai", serves:"4", src:"tastesbetterfromscratch.com", url:"https://tastesbetterfromscratch.com/pad-thai/",
    ing:[
      {heading:"", rows:[
        {text:"8 oz flat rice noodles", item:"Flat rice noodles", qty:1},
        {text:"3 Tbsp oil", item:"Vegetable/peanut oil", qty:0.1},
        {text:"3 cloves garlic, minced", item:"Garlic", qty:0.2},
        {text:"8 oz chicken (or shrimp / extra-firm tofu), cut into small pieces", item:"Chicken breast", qty:0.5},
        {text:"2 eggs", item:"Eggs", qty:0.2},
        {text:"1 cup fresh bean sprouts", item:"Bean sprouts", qty:1},
        {text:"1 red bell pepper, thinly sliced", item:"Bell peppers", qty:1},
        {text:"3 green onions, chopped", item:"Green onions", qty:0.5},
        {text:"1/2 cup dry roasted peanuts", item:"Dry roasted peanuts", qty:0.3},
        {text:"2 limes", item:"Limes", qty:2},
        {text:"1/2 cup fresh cilantro, chopped", item:"Fresh cilantro", qty:0.4},
        {text:"6 oz Pad Thai Sauce", item:"Pad Thai Sauce", qty:1}
      ]}
    ],
    steps:["Cook noodles per package, just until tender. Rinse under cold water.",
      "Stir-fry: Heat 1½ Tbsp oil in a wok over medium-high. Add chicken, garlic, and bell pepper. Cook chicken until just cooked through, ~3–4 minutes, flipping once.",
      "Push everything to the side. Add a little more oil and the beaten eggs. Scramble, breaking into small pieces as they cook.",
      "Add noodles, sauce, bean sprouts, and peanuts (reserve some peanuts for topping). Toss to combine.",
      "Garnish with green onions, extra peanuts, cilantro, and lime wedges. Serve immediately."]},
  stirfry:{title:"Chicken Stir Fry Noodles", serves:"~4", src:"eatwell101.com", url:"https://www.eatwell101.com/chicken-pasta-skillet-recipe",
    ing:[
      {heading:"", rows:[
        {text:"1 lb skinless, boneless chicken breast, cut into strips", item:"Chicken breast", qty:1},
        {text:"1 Tbsp olive oil", item:"Olive oil", qty:0.05},
        {text:"3 garlic cloves, finely minced", item:"Garlic", qty:0.2},
        {text:"1 medium carrot, julienned or shredded", item:"Carrots", qty:0.3},
        {text:"8 oz spaghetti noodles", item:"Spaghetti", qty:0.5},
        {text:"3 cups broccoli florets", item:"Broccoli florets", qty:1},
        {text:"Chopped scallion, to garnish", item:"Green onions", qty:0.3},
        {text:"Toasted sesame seeds, to garnish", item:"Sesame seeds", qty:0.1}
      ]},
      {heading:"Sauce", rows:[
        {text:"1/2 tsp grated or ground ginger", item:"Fresh ginger", qty:0.5},
        {text:"1 Tbsp brown sugar (or honey)", item:null, qty:0},
        {text:"1/4 cup low-sodium soy sauce", item:"Light soy sauce", qty:0.1},
        {text:"2 Tbsp hoisin sauce", item:"Hoisin sauce", qty:0.2},
        {text:"2 tsp sesame oil", item:"Toasted sesame oil", qty:0.1},
        {text:"1/4 tsp crushed red pepper flakes (optional)", item:null, qty:0},
        {text:"Fresh cracked black pepper, to taste", item:null, qty:0}
      ]}
    ],
    steps:["In a large pot of boiling salted water, cook the spaghetti per package directions. Add the broccoli florets for the last 5 minutes and cook until tender. Drain noodles and broccoli and set aside.",
      "While the pasta cooks, whisk together the brown sugar, soy sauce, hoisin, sesame oil, ginger, red pepper, and black pepper in a small bowl.",
      "Season chicken strips with salt and pepper. Add olive oil to a skillet and cook chicken in an even layer over medium heat, turning occasionally, 2–3 minutes until golden. Stir in garlic and carrots and cook a minute more until cooked through.",
      "Transfer the cooked spaghetti and broccoli to the skillet, pour the sauce on top, and toss until fully incorporated. Reheat a couple minutes and serve garnished with scallion and toasted sesame seeds."]},
  thainoodles:{title:"Spicy Thai Noodles", serves:"6", src:"thechunkychef.com", url:"https://www.thechunkychef.com/spicy-thai-noodles/",
    ing:[
      {heading:"", rows:[
        {text:"1 lb chicken breast", item:"Chicken breast", qty:1},
        {text:"1 lb rice noodles", item:"Flat rice noodles", qty:2},
        {text:"1/2 – 1 1/2 Tbsp red pepper flakes (to taste)", item:null, qty:0},
        {text:"2 Tbsp vegetable oil", item:"Vegetable/peanut oil", qty:0.1},
        {text:"1/3 – 1/2 cup toasted sesame oil", item:"Toasted sesame oil", qty:0.4},
        {text:"2 tsp chili paste", item:"Chili paste (sambal)", qty:0.1},
        {text:"6 Tbsp soy sauce", item:"Light soy sauce", qty:0.3},
        {text:"6 Tbsp honey", item:"Honey", qty:0.3},
        {text:"Scallions, to garnish", item:"Green onions", qty:0.4},
        {text:"Carrots, to garnish", item:"Carrots", qty:0.2},
        {text:"Peanuts, to garnish", item:"Dry roasted peanuts", qty:0.3},
        {text:"Cilantro, to garnish", item:"Fresh cilantro", qty:0.3},
        {text:"Sriracha, to garnish", item:"Sriracha", qty:0.05},
        {text:"Sesame seeds, to garnish", item:"Sesame seeds", qty:0.1}
      ]}
    ],
    steps:["Chop garnishes and set aside.","Boil pasta, then drain.",
      "While the pasta boils, heat the oils in a large skillet with the red pepper flakes.",
      "Once the oil is hot, strain out the pepper flakes, reserving the oil in a bowl. Add the reserved oil back to the skillet and add the chili paste. Whisk in the soy sauce and honey.",
      "Toss the pasta in the skillet with the sauce.",
      "Serve hot, room temperature, or cold. Top with garnishes and enjoy."]}
};
```

This is intentionally only 5 of the 21 recipes so far (`bpc`, `tacos`, `padthai`, `stirfry`, `thainoodles`) — do not run anything yet, continue with Step 2.

- [ ] **Step 2: Add the remaining 16 recipes**

In the code you just pasted, find the closing of `thainoodles`:

```js
      "Serve hot, room temperature, or cold. Top with garnishes and enjoy."]}
};
```

Delete the `};` on its own line, add a comma after the `]}` that closes `thainoodles`, and insert all of the following before the final `};`:

```js
  snackwraps:{title:"Buffalo Ranch Snack Wraps", serves:"4", src:"Your recipe (inline)", url:"",
    ing:[
      {heading:"", rows:[
        {text:"Breaded chicken strips/patties (frozen)", item:"Breaded chicken strips (frozen)", qty:1},
        {text:"Shredded lettuce", item:"Iceberg lettuce", qty:0.4},
        {text:"Mexican-blend shredded cheese", item:"Mexican-blend shredded cheese", qty:1},
        {text:"Buffalo sauce", item:"Buffalo sauce", qty:0.3},
        {text:"Ranch dressing", item:"Ranch dressing", qty:0.3},
        {text:"Small flour tortillas", item:"Small flour tortillas", qty:1}
      ]}
    ],
    steps:["Cook the breaded chicken per package directions.","Toss or drizzle the cooked chicken with buffalo sauce.",
      "Lay chicken on a tortilla with shredded lettuce, Mexican cheese, and a drizzle of ranch.","Fold up and serve."]},
  crunchwraps:{title:"Beef Crunch Wraps", serves:"4–6", src:"Your recipe (inline)", url:"",
    ing:[
      {heading:"", rows:[
        {text:"1 lb ground beef", item:"Ground beef", qty:1},
        {text:"Kroger queso", item:"Kroger queso dip", qty:0.6},
        {text:"Shredded lettuce", item:"Iceberg lettuce", qty:0.5},
        {text:"Mexican-blend shredded cheese", item:"Mexican-blend shredded cheese", qty:1},
        {text:"Diced tomatoes", item:"Tomatoes", qty:3},
        {text:"Large burrito flour tortillas", item:"Large burrito tortillas", qty:1},
        {text:"Corn tortillas / tostadas (the crunch layer)", item:"Corn tortillas / tostadas", qty:1},
        {text:"Taco seasoning", item:"Taco seasoning", qty:1}
      ]}
    ],
    steps:["Brown the ground beef and drain. Add taco seasoning + a splash of water and simmer until thickened.",
      "On a large burrito tortilla, layer seasoned beef and queso in the center, then set a corn tortilla on top.",
      "Add lettuce, diced tomatoes, and Mexican cheese on top of the corn tortilla.",
      "Fold the edges of the large tortilla up over the center, pleating as you go.",
      "Crisp seam-side-down in a hot skillet until golden, then flip and crisp the other side."]},
  garlicbroccoli:{title:"Spicy Garlic Chicken & Broccoli Noodle Bowl", serves:"4", src:"chefnovarecipes.com", url:"https://www.chefnovarecipes.com/spicy-garlic-chicken-and-broccoli-noodle-bowl/",
    ing:[
      {heading:"", rows:[
        {text:"1 lb chicken breast, sliced", item:"Chicken breast", qty:1},
        {text:"8 oz noodles (soba, rice, or your choice)", item:"Soba/rice noodles", qty:1},
        {text:"2 cups broccoli florets", item:"Broccoli florets", qty:1},
        {text:"4 cloves garlic, minced", item:"Garlic", qty:0.3},
        {text:"2 Tbsp soy sauce", item:"Light soy sauce", qty:0.05},
        {text:"1 Tbsp chili paste", item:"Chili paste (sambal)", qty:0.1},
        {text:"2 Tbsp olive oil", item:"Olive oil", qty:0.05},
        {text:"Salt and pepper, to taste", item:null, qty:0}
      ]}
    ],
    steps:["Cook the noodles per package until al dente, then drain and set aside.",
      "In a large skillet, heat the olive oil over medium until it shimmers.",
      "Add the minced garlic and sauté until golden and fragrant, about 1 minute.",
      "Add the sliced chicken and cook until browned and no longer pink, about 5–7 minutes.",
      "Once the chicken is nearly cooked, add the broccoli florets and stir-fry 3–4 minutes.",
      "Pour in the soy sauce and chili paste, stirring to coat.",
      "Add the cooked noodles and toss everything together.",
      "Season with salt and pepper to taste before serving hot."]},
  alfredo:{title:"Chicken Alfredo", serves:"4", src:"Your recipe (inline)", url:"",
    ing:[
      {heading:"", rows:[
        {text:"1 lb chicken", item:"Chicken breast", qty:1},
        {text:"1 lb noodles (fettuccine)", item:"Fettuccine", qty:1},
        {text:"1 jar/can Alfredo sauce", item:"Alfredo sauce", qty:1}
      ]}
    ],
    steps:["Boil the noodles per package; drain.","Season and cook the chicken, then slice.",
      "Warm the Alfredo sauce, then toss with the noodles and chicken. Serve."]},
  spaghetti:{title:"Spaghetti & Meatballs", serves:"4–6", src:"Your recipe (inline)", url:"",
    ing:[
      {heading:"", rows:[
        {text:"1 lb noodles (spaghetti)", item:"Spaghetti", qty:1},
        {text:"1 lb meatballs", item:"Frozen meatballs", qty:1},
        {text:"1 jar/can pasta sauce", item:"Pasta sauce", qty:1},
        {text:"Italian seasoning", item:null, qty:0}
      ]}
    ],
    steps:["Boil the spaghetti per package; drain.",
      "Heat the pasta sauce with the meatballs and a shake of Italian seasoning until the meatballs are heated through.",
      "Serve the sauce and meatballs over the spaghetti."]},
  garlicpasta:{title:"Roasted Garlic Spaghetti", serves:"4–6", src:"motherthyme.com", url:"https://www.motherthyme.com/2017/01/roasted-garlic-spaghetti.html",
    ing:[
      {heading:"", rows:[
        {text:"2 whole garlic heads", item:"Garlic", qty:2},
        {text:"1/2 tsp olive oil", item:"Olive oil", qty:0.05},
        {text:"1 lb spaghetti", item:"Spaghetti", qty:1},
        {text:"1/2 cup reserved pasta water", item:null, qty:0},
        {text:"1 stick (1/2 cup) butter", item:"Butter", qty:0.25},
        {text:"2 Tbsp freshly grated Parmesan", item:"Grated Parmesan", qty:0.2},
        {text:"2–3 Tbsp chopped parsley (or basil)", item:"Fresh parsley", qty:1},
        {text:"1/2 tsp salt, plus more to taste", item:null, qty:0},
        {text:"1/4 tsp black pepper, plus more to taste", item:null, qty:0},
        {text:"Shaved Parmesan, to garnish (optional)", item:null, qty:0},
        {text:"Pinch of red pepper flakes (optional)", item:null, qty:0}
      ]}
    ],
    steps:["Preheat oven to 400°F.",
      "Cut the tops off the garlic heads to expose the cloves. Drizzle with olive oil and a pinch of salt.",
      "Wrap garlic in foil (or set each head in a muffin tin) and roast 35–40 minutes until soft. Cool to the touch.",
      "Squeeze the garlic out of the cloves and mash with a fork.",
      "Bring a large pot of water to a boil, add spaghetti, and cook per package directions. Drain, reserving 1/2 cup pasta water.",
      "Melt the butter in the same pot or a large deep skillet over medium heat.",
      "Add the roasted garlic, then the cooked spaghetti and reserved pasta water; stir until coated and heated through.",
      "Off heat, stir in the cheese, herbs, and red pepper flakes if using. Season with salt and pepper and garnish with shaved Parmesan."]},
  pork:{title:"Honey Garlic Pork Tenderloin", serves:"6", src:"downshiftology.com", url:"https://downshiftology.com/recipes/honey-garlic-pork-tenderloin/",
    ing:[
      {heading:"", rows:[
        {text:"2 (1 lb each) pork tenderloins", item:"Pork tenderloin", qty:2},
        {text:"1 tsp garlic powder", item:null, qty:0},
        {text:"1 tsp sweet paprika", item:null, qty:0},
        {text:"1 tsp onion powder", item:null, qty:0},
        {text:"1 tsp dried thyme", item:null, qty:0},
        {text:"1 tsp kosher salt", item:null, qty:0},
        {text:"1/2 tsp freshly ground black pepper", item:null, qty:0},
        {text:"2 Tbsp extra-virgin olive oil", item:"Olive oil", qty:0.05},
        {text:"1/2 cup honey", item:"Honey", qty:0.4},
        {text:"1/4 cup tamari / soy sauce", item:"Light soy sauce", qty:0.1},
        {text:"2 Tbsp apple cider vinegar", item:"Apple cider vinegar", qty:0.1},
        {text:"6 garlic cloves, minced", item:"Garlic", qty:0.5}
      ]}
    ],
    steps:["Preheat the oven to 375°F. In a small bowl, stir together the garlic powder, paprika, onion powder, thyme, salt, and pepper. Set aside.",
      "In a medium bowl, whisk together the honey, soy sauce, vinegar, and garlic. Set aside.",
      "Sprinkle the dry spice blend all over the pork and rub it in with your hands.",
      "In a large oven-safe skillet, heat the oil over medium-high. Sear the pork, turning occasionally, until browned on all sides, about 4 minutes total.",
      "Turn off the stove, add the sauce to the pan, and turn the pork to coat. Transfer to the oven and roast 15–20 minutes, until an instant-read thermometer reads 140°F.",
      "Return the skillet to the stove; remove the pork to a plate and cover with foil to rest 5–10 minutes. Simmer the remaining sauce 1–2 minutes until slightly reduced and thickened.",
      "Slice the pork into 1/2-inch slices, arrange on a platter, drizzle with the sauce, and serve."]},
  breakfast:{title:"Breakfast for Dinner", serves:"4", src:"Your recipe (inline)", url:"",
    ing:[
      {heading:"", rows:[
        {text:"Pancakes (mix + as needed)", item:"Pancake mix", qty:1},
        {text:"Maple syrup", item:"Pancake/maple syrup", qty:1},
        {text:"Bacon", item:"Bacon", qty:1},
        {text:"Sausage", item:"Breakfast sausage", qty:1},
        {text:"Eggs", item:"Eggs", qty:0.5}
      ]}
    ],
    steps:["Cook the bacon and sausage.","Scramble or fry the eggs.","Griddle the pancakes.","Serve everything together."]},
  helper:{title:"One-Pot Cheeseburger Macaroni (Hamburger Helper)", serves:"6", src:"jawnsicooked.com", url:"https://jawnsicooked.com/dinner/one-pot-homemade-cheeseburger-macaroni-hamburger-helper/",
    ing:[
      {heading:"", rows:[
        {text:"2 Tbsp butter", item:"Butter", qty:0.06},
        {text:"1 small onion, diced", item:"Yellow onions", qty:1},
        {text:"1 lb lean ground beef", item:"Ground beef", qty:1},
        {text:"1 Tbsp tomato paste", item:"Tomato paste", qty:1},
        {text:"2 tsp smoked paprika", item:null, qty:0},
        {text:"1 1/2 tsp garlic powder", item:null, qty:0},
        {text:"1 tsp onion powder", item:null, qty:0},
        {text:"1 1/2 tsp seasoned salt (or table salt)", item:null, qty:0},
        {text:"1 tsp sugar", item:null, qty:0},
        {text:"1/2 tsp freshly ground black pepper", item:null, qty:0},
        {text:"2 Tbsp all-purpose flour", item:"Bread/all-purpose flour", qty:0.05},
        {text:"2 cups beef broth", item:"Beef broth", qty:0.5},
        {text:"1 1/2 cups milk (not skim)", item:"Milk", qty:0.1},
        {text:"2 cups (1/2 lb) dry cavatappi or macaroni", item:"Cavatappi/macaroni", qty:1},
        {text:"2 cups (8 oz block) grated yellow sharp cheddar", item:"Sharp cheddar cheese", qty:1}
      ]}
    ],
    steps:["In a large deep sauté pan or cast iron skillet, melt the butter over medium heat.",
      "Add the diced onion and cook, stirring occasionally, until translucent, about 5 minutes.",
      "Add the ground beef and cook, breaking it up, until browned, 4–5 minutes.",
      "Add the tomato paste, stir into the meat, and cook 2–3 minutes until slightly deepened in color.",
      "Add the smoked paprika, garlic powder, onion powder, salt, sugar, pepper, and flour; stir until combined and cook 1 minute.",
      "Slowly whisk in the beef broth, then add the milk and stir well.",
      "Add the pasta, raise heat to high, and bring to a boil.",
      "Reduce heat to medium-low and simmer until the pasta is cooked through, stirring occasionally so it doesn't stick, 10–12 minutes.",
      "Remove from heat and stir in the cheese."]},
  burgers:{title:"Burgers (with homemade seasoning)", serves:"4", src:"Your recipe + simplejoy.com seasoning", url:"https://www.simplejoy.com/hamburger-seasoning/",
    ing:[
      {heading:"Burgers", rows:[
        {text:"1 lb ground beef (80/20)", item:"Ground beef", qty:1},
        {text:"American cheese", item:"American cheese singles", qty:1},
        {text:"Hamburger buns", item:"Hamburger buns", qty:1},
        {text:"1 Tbsp Hamburger Seasoning (below)", item:null, qty:0}
      ]},
      {heading:"Hamburger Seasoning (makes 9 Tbsp)", rows:[
        {text:"2 Tbsp paprika", item:null, qty:0},
        {text:"2 Tbsp brown sugar", item:null, qty:0},
        {text:"1 Tbsp onion powder", item:null, qty:0},
        {text:"2 tsp table salt", item:null, qty:0},
        {text:"2 tsp black pepper", item:null, qty:0},
        {text:"2 tsp garlic powder", item:null, qty:0},
        {text:"2 tsp cumin", item:null, qty:0},
        {text:"1/2 tsp cayenne pepper", item:null, qty:0}
      ]}
    ],
    steps:["Make the seasoning: mix all seasoning ingredients in a small bowl (makes 9 Tbsp / enough for 9 lbs of meat; store the rest airtight up to 6 months).",
      "Divide 1 lb of 80/20 ground beef into four equal patties, indenting the middle of each for even cooking.",
      "Use 1 Tbsp of the seasoning total to rub into both sides of all four patties (or mix 1 Tbsp into the pound of beef).",
      "Grill or skillet the patties over medium heat about 3 1/2 minutes per side for medium; add American cheese near the end to melt.",
      "Serve on buns with your favorite toppings."]},
  mac:{title:"Stovetop Mac & Cheese", serves:"6", src:"Your recipe (photo)", url:"",
    ing:[
      {heading:"", rows:[
        {text:"1 stick butter", item:"Butter", qty:0.25},
        {text:"2 Tbsp flour", item:"Bread/all-purpose flour", qty:0.05},
        {text:"3 cups milk", item:"Milk", qty:0.2},
        {text:"4 oz Monterey Jack cheese", item:"Monterey Jack cheese", qty:0.5},
        {text:"4 oz sharp cheddar cheese", item:"Sharp cheddar cheese", qty:0.5},
        {text:"16 oz elbow macaroni", item:"Elbow macaroni", qty:1},
        {text:"Salt, pepper, paprika", item:null, qty:0}
      ]}
    ],
    steps:["Boil the macaroni noodles.","Melt 1 stick of butter in a large skillet over medium heat.",
      "Put in the flour; mix.","Pour in the milk; add salt, pepper, and paprika.",
      "Once it thickens, add the Monterey Jack and sharp cheddar; let melt.",
      "Strain the noodles and add to the cheese sauce.","Broil the mac & cheese for 3–5 minutes."]},
  tacobowls:{title:"Ground Beef Taco Bowls", serves:"4", src:"spicedbites.com", url:"https://spicedbites.com/ground-beef-taco-bowls",
    ing:[
      {heading:"", rows:[
        {text:"1 lb ground beef (85/15 or 90/10)", item:"Ground beef", qty:1},
        {text:"1 cup long-grain white rice, rinsed", item:"Long-grain white rice", qty:0.3},
        {text:"2 cups water (for the rice)", item:null, qty:0},
        {text:"2 Tbsp taco seasoning", item:"Taco seasoning", qty:1},
        {text:"1 can (15 oz) black beans, rinsed & drained", item:"Black beans", qty:1},
        {text:"1 cup sweet corn (canned or frozen)", item:"Sweet corn", qty:1},
        {text:"2 medium tomatoes, finely diced", item:"Tomatoes", qty:2},
        {text:"2 cups iceberg lettuce, shredded", item:"Iceberg lettuce", qty:0.5},
        {text:"Sour cream", item:"Sour cream", qty:1},
        {text:"Olive oil, for the skillet", item:"Olive oil", qty:0.05},
        {text:"1/4 cup fresh cilantro, chopped", item:"Fresh cilantro", qty:0.3}
      ]}
    ],
    steps:["Cook the rice: rinse under cold water, combine with 2 cups water in a saucepan, and bring to a boil.",
      "Reduce heat to low, cover, and simmer 18 minutes. Let stand 5 minutes, then fluff.",
      "Prepare the beef: heat oil in a skillet over medium-high and brown the beef until crumbled and fully cooked, stirring in the taco seasoning.",
      "Assemble: place rice in bowls and arrange the beef, beans, corn, tomatoes, and lettuce in sections.",
      "Add a dollop of sour cream to the center and sprinkle with fresh cilantro."]},
  kfc:{title:"Homemade KFC Famous Bowl (Copycat)", serves:"4", src:"imhungryforthat.com", url:"https://imhungryforthat.com/kfc-famous-bowl-copycat-recipe/",
    ing:[
      {heading:"", rows:[
        {text:"2 1/2 cups mashed potatoes", item:"Instant mashed potatoes", qty:1},
        {text:"2 cups corn kernels (frozen)", item:"Sweet corn", qty:1},
        {text:"10 oz popcorn chicken or chicken nuggets", item:"Frozen popcorn chicken / nuggets", qty:1},
        {text:"1 cup shredded cheddar cheese", item:"Shredded cheddar", qty:1},
        {text:"1/2 cup Heinz classic chicken gravy", item:"Heinz classic chicken gravy", qty:1}
      ]}
    ],
    steps:["Preheat the oven to 350°F.",
      "Cook the popcorn chicken/nuggets in an air fryer at 390°F for 10 minutes (or bake per package directions).",
      "Prepare the mashed potatoes per package.",
      "Layer the casserole: mashed potatoes on the bottom, then corn, then chicken, topped with shredded cheese.",
      "Bake 10–15 minutes, until the cheese melts.",
      "Transfer to bowls and top with gravy."]},
  pizza:{title:"Pepperoni Pizza (homemade dough)", serves:"4 (dough makes 2 crusts)", src:"foodnetwork.com + thekitchn.com", url:"https://www.thekitchn.com/pepperoni-pizza-22931782",
    ing:[
      {heading:"Pizza Dough (Bobby Flay — makes 2 crusts)", rows:[
        {text:"3 1/2 – 4 cups bread flour, plus more for rolling", item:"Bread/all-purpose flour", qty:0.2},
        {text:"1 tsp sugar", item:null, qty:0},
        {text:"1 envelope instant dry yeast", item:"Instant yeast", qty:1},
        {text:"2 tsp kosher salt", item:null, qty:0},
        {text:"1 1/2 cups water (110°F)", item:null, qty:0},
        {text:"2 Tbsp olive oil, plus 2 tsp", item:null, qty:0}
      ]},
      {heading:"Pepperoni Pizza (The Kitchn)", rows:[
        {text:"1 lb pizza dough, at room temperature", item:null, qty:0},
        {text:"2 Tbsp olive oil", item:null, qty:0},
        {text:"1/2 – 1 cup tomato sauce", item:"Pizza tomato sauce", qty:1},
        {text:"1 cup shredded low-moisture mozzarella", item:"Shredded mozzarella", qty:1},
        {text:"1/2 cup grated Parmesan", item:"Grated Parmesan", qty:0.3},
        {text:"2 oz pepperoni slices (about 1/2 cup)", item:"Pepperoni", qty:1},
        {text:"Red pepper flakes, to serve (optional)", item:null, qty:0}
      ]}
    ],
    steps:["DOUGH: Combine the bread flour, sugar, yeast, and salt in a stand mixer. With the mixer running, add the water and 2 Tbsp oil and beat until the dough forms a ball (add flour 1 Tbsp at a time if sticky, or water if dry). Knead on a floured surface into a smooth, firm ball.",
      "DOUGH: Grease a large bowl with the remaining 2 tsp oil, add the dough, cover with plastic wrap, and let it double in a warm spot, about 1 hour. Divide into 2 pieces, cover, and rest 10 minutes.",
      "PIZZA: Bring 1 lb dough to room temperature. Set a rack in the middle of the oven and heat to 450°F.",
      "Coat a rimmed baking sheet with 2 Tbsp olive oil. Place the dough on it and press out toward the edges into an even crust (don't tear it).",
      "Spread 1/2 cup tomato sauce in a thin layer leaving a border (add more if desired). Sprinkle on the mozzarella and Parmesan, then lay the pepperoni in a single layer.",
      "Bake 15–20 minutes, until the edges are golden-brown and crispy and the cheese is melted. Rotate the pan halfway if your oven bakes unevenly. Cool 5 minutes, slice, and serve with red pepper flakes if desired."]},
  zuppa:{title:"Instant Pot Pesto Zuppa Toscana", serves:"6", src:"halfbakedharvest.com", url:"https://www.halfbakedharvest.com/instant-pot-pesto-zuppa-toscana/",
    ing:[
      {heading:"", rows:[
        {text:"4 slices thick-cut bacon, chopped", item:"Bacon", qty:0.5},
        {text:"3/4 lb ground spicy Italian chicken sausage", item:"Ground spicy Italian chicken sausage", qty:1},
        {text:"1 yellow onion, chopped", item:"Yellow onions", qty:1},
        {text:"4 cloves garlic, minced or grated", item:"Garlic", qty:0.3},
        {text:"2 ribs celery, chopped", item:"Celery", qty:1},
        {text:"4 small Yukon gold or russet potatoes, peeled & chopped", item:"Yukon gold / russet potatoes", qty:1},
        {text:"6 cups low-sodium chicken broth", item:"Chicken broth (low-sodium)", qty:1.5},
        {text:"1/3 cup basil pesto", item:"Basil pesto", qty:1},
        {text:"Juice of 1 lemon", item:"Lemon", qty:1},
        {text:"1 pinch crushed red pepper flakes", item:null, qty:0},
        {text:"Kosher salt & black pepper", item:null, qty:0},
        {text:"Olive oil, for sautéing", item:"Olive oil", qty:0.05},
        {text:"1 bunch Tuscan or curly kale, roughly chopped", item:"Kale (Tuscan or curly)", qty:1},
        {text:"3/4 cup heavy cream (or whole milk)", item:"Heavy cream", qty:1},
        {text:"1/2 cup grated Parmesan or Asiago", item:"Grated Parmesan", qty:0.2},
        {text:"Fresh thyme, to serve (optional)", item:null, qty:0}
      ]}
    ],
    steps:["Set the Instant Pot to sauté. Add the bacon and cook until crisp, about 5 minutes. Remove the bacon. If there's excess grease, drain all but 1 Tbsp. Add the chicken sausage and onions and brown all over, 5–8 minutes. Add the garlic, celery, and potatoes and cook 2 minutes. Turn the Instant Pot off.",
      "Add the broth, pesto, lemon juice, red pepper flakes, and season with salt and pepper. Cover and cook on high pressure 8 minutes.",
      "Use natural or quick release. Set to sauté and stir in the kale, cream, and Parmesan. Cook until the kale is wilted, about 10 minutes. Turn off and stir in the reserved bacon.",
      "Serve topped with additional Parmesan and fresh thyme, if desired."]},
  porkloin:{title:"Best Damn Oven-Roasted Pork Loin", serves:"4–6", src:"recipeteacher.com", url:"https://recipeteacher.com/best-damn-oven-roasted-pork-loin/",
    ing:[
      {heading:"", rows:[
        {text:"Pork loin roast, 2-3 lbs, boneless", item:"Pork loin roast (boneless)", qty:1},
        {text:"1 Tbsp kosher salt", item:null, qty:0},
        {text:"1 Tbsp Worcestershire sauce", item:"Worcestershire sauce", qty:0.1},
        {text:"1 tsp paprika", item:null, qty:0},
        {text:"1 tsp onion powder", item:null, qty:0},
        {text:"1 tsp garlic powder", item:null, qty:0},
        {text:"1 tsp rosemary, dried and crushed", item:null, qty:0},
        {text:"1 tsp black pepper, ground", item:null, qty:0},
        {text:"Olive oil (for the pan)", item:"Olive oil", qty:0.05}
      ]}
    ],
    steps:["Preheat oven to 425°F. Line a sheet pan with aluminum foil, spray with a little non-stick cooking spray, and set aside.",
      "Mix all dry ingredients in a small bowl and set aside.",
      "Trim any excess fat from the top of the pork loin roast. Place the roast on a large plate and coat with Worcestershire sauce, then with the dry seasonings. Rub liberally on all sides.",
      "Place the seasoned pork roast fat-side-up on the foil-lined sheet pan and roast at 425°F for 15 minutes.",
      "After 15 minutes, reduce heat to 375°F and continue roasting about 45 minutes, or until the internal temperature at the center reaches 145°F.",
      "Remove from the oven and tent loosely with foil for 5–10 minutes before serving."]}
```

After pasting, the object should have exactly 21 keys, ending with `porkloin`'s closing `]}\n};`. Double-check the brace/comma structure — a syntax error here breaks the entire app.

- [ ] **Step 3: One-time `recipeEdits` cleanup for the three now-superseded overrides**

Find where `recipeEdits` is loaded (in the RECIPES section):

```js
let recipeEdits = load('mp_recipeEdits_v1', {});   // { [recipeKey]: {ing, steps} } — user overrides, layered on top of R
```

Add immediately after it:

```js
// One-time migration: any recipeEdits saved under the old free-text ing shape
// (array of [heading, ...strings]) are superseded now that the corresponding
// built-in recipe was rewritten directly with the new structured shape above.
// An old-shape override would otherwise mask the new base data and break
// rendering (rows are objects now, not strings).
Object.keys(recipeEdits).forEach(k=>{
  const groups = recipeEdits[k] && recipeEdits[k].ing;
  if(groups && groups.length && Array.isArray(groups[0])) delete recipeEdits[k];
});
```

- [ ] **Step 4: Manual verification (this task alone, before Task 4/5 land — grocery tabs will still be broken, that's expected)**

Serve locally, console: `syncRef=null;`.

1. Open the **Recipes** tab (All Recipes index) — expect it to render EMPTY or throw a console error. This is expected: `renderAllRecipes`/`recipeCard`/`ingHTML` still read the OLD `ing` shape (array of strings) and haven't been updated for the new row-object shape yet (Task 4). Confirm the error is specifically about `.map is not a function` or similar on ingredient rendering, not a syntax error in `R` itself — a syntax error means Step 2's brace/comma structure is wrong and must be fixed before continuing.
2. Run `Object.keys(R).length` → expect `21`.
3. Run `R.thainoodles.ing[0].rows.some(r=>r.item==="Chicken breast")` → expect `true` (confirms the chicken-breast fix landed).
4. Run `R.padthai.ing[0].rows.some(r=>r.item==="Pad Thai Sauce")` → expect `true`, and `R.padthai.ing[0].rows.some(r=>r.item==="Fish sauce")` → expect `false` (confirms the sauce swap landed, old homemade-sauce links are gone).
5. Reload the page and run `Object.keys(recipeEdits)` → expect `[]` (confirms the three stale overrides were purged and that purge itself synced/saved — check `localStorage.getItem('mp_recipeEdits_v1')` reads `"{}"`).

- [ ] **Step 5: Commit**

```bash
cd "C:/Users/Matt/GitHub/dinner-meal-plan"
git add index.html
git commit -m "Migrate all 21 recipes to structured {text,item,qty} ingredient rows"
```

---

### Task 4: Update rendering for the new row shape; remove the now-incompatible free-text edit UI

**Files:**
- Modify: `index.html` (RECIPES section, currently lines 922–1002 and 1174–1241)

**Context:** The old textarea-based recipe editor (`editRecipe`/`saveRecipeEdit`/etc., and the `## Heading` text convention it used) reads/writes `ing` as an array of plain-string arrays. That shape no longer exists anywhere after Task 3 — every recipe's `ing` is now `[{heading, rows:[{text,item,qty}]}]`. This task removes that now-broken editor entirely (Part B replaces it with a proper structured-row builder) and updates every function that reads `ing` to the new shape. Until Part B lands, recipes can be viewed and planned but not edited — that's the expected, temporary state described at the top of this plan.

Leave the CSS classes `.recipe-edit-box`, `.edit-hint`, `.recipe-edit-actions` in place even though nothing uses them after this task — Part B's builder UI reuses these same classes for its own Save/Cancel row and hint text, so removing and re-adding them would be pure churn.

- [ ] **Step 1: Remove the free-text edit functions**

Current code (lines 930–968, between `recipeData` and `escHTML`):

```js
let recipeEditKey = null;   // recipe key currently in edit mode on the detail page, or null
let recipeDetailScale = 1;   // last-rendered scale for the recipe currently open in read mode
function ingredientsToText(groups){
  return groups.map(g=>{
    const head = g[0] ? `## ${g[0]}\n` : "";
    return head + g.slice(1).join("\n");
  }).join("\n\n");
}
function parseIngredients(text){
  const lines = text.split("\n").map(l=>l.trim()).filter(Boolean);
  const groups = []; let current = null;
  lines.forEach(line=>{
    if(line.startsWith("## ")){ current=[line.slice(3).trim()]; groups.push(current); }
    else { if(!current){ current=[""]; groups.push(current); } current.push(line); }
  });
  return groups.length ? groups : [[""]];
}
function parseSteps(text){
  return text.split("\n").map(s=>s.trim()).filter(Boolean);
}
function editRecipe(k){ recipeEditKey=k; showRecipeDetail(k, recipeDetailScale); }
function cancelRecipeEdit(k){ recipeEditKey=null; showRecipeDetail(k, recipeDetailScale); }
function saveRecipeEdit(k){
  const ingBox=document.getElementById('recipeEditIng'), stepBox=document.getElementById('recipeEditSteps');
  recipeEdits[k] = { ing: parseIngredients(ingBox.value), steps: parseSteps(stepBox.value) };
  saveRecipeEdits();
  recipeEditKey=null;
  showRecipeDetail(k, recipeDetailScale);
  renderRecipes();
  toast("Recipe saved");
}
function resetRecipeEdit(k){
  delete recipeEdits[k];
  saveRecipeEdits();
  recipeEditKey=null;
  showRecipeDetail(k, recipeDetailScale);
  renderRecipes();
  toast("Reset to original");
}
```

Delete this whole block. Nothing replaces it in this task — `recipeData`, right above where this block was, and `escHTML`, right below it, stay exactly as they are and now sit next to each other.

- [ ] **Step 2: Rewrite `ingHTML` for the new row shape**

Current code:

```js
function ingHTML(groups){
  return groups.map(g=>{
    const head = g[0] ? `<div class="sub-h">${escHTML(g[0])}</div>` : "";
    return head+`<ul class="ing-list">${g.slice(1).map(x=>`<li>${escHTML(x)}</li>`).join("")}</ul>`;
  }).join("");
}
```

Replace with:

```js
function ingHTML(groups){
  return groups.map(g=>{
    const head = g.heading ? `<div class="sub-h">${escHTML(g.heading)}</div>` : "";
    return head+`<ul class="ing-list">${g.rows.map(r=>`<li>${escHTML(r.text)}</li>`).join("")}</ul>`;
  }).join("");
}
```

- [ ] **Step 3: Recipe rendering (`recipeCard`, `recipesHTML`, `renderRecipes`) needs no shape change**

`recipeCard` calls `ingHTML(rd.ing)` and never touches `rd.ing`'s internal structure directly — Step 2's fix is sufficient, `recipeCard` itself is unchanged. Confirm this by re-reading it, but do not edit it in this task.

- [ ] **Step 4: Collapse `showRecipeDetail` to a single read-only render**

Current code (lines 1181–1228):

```js
function showRecipeDetail(k,scale){
  scale = scale||1;
  const r=R[k];
  const link = r.url ? `<a href="${r.url}" target="_blank" rel="noopener">${r.src} ↗</a>` : r.src;
  const note = r.note ? `<div class="note-box">💡 ${r.note}</div>` : "";
  const scaleNote = (scale!==1) ? `<div class="note-box">👥 ${scale}x tonight — grocery list already adjusted.</div>` : "";
  if(recipeEditKey===k){
    const rd=recipeData(k);
    document.getElementById("recipeDetail").innerHTML =
      `<button class="btn-reset" style="margin-bottom:12px" onclick="cancelRecipeEdit('${k}')">← Cancel</button>
       <article class="recipe">
         <h3>${r.title}</h3>
         <div class="meta">Serves ${r.serves} · Source: ${link}</div>
         ${scaleNote}
         <div class="cols">
           <div><h4>Ingredients</h4>
             <textarea id="recipeEditIng" class="recipe-edit-box" rows="12">${escHTML(ingredientsToText(rd.ing))}</textarea>
             <div class="edit-hint">Start a line with "## " for a section heading (e.g. "## Marinade").</div>
           </div>
           <div><h4>Instructions</h4>
             <textarea id="recipeEditSteps" class="recipe-edit-box" rows="12">${escHTML(rd.steps.join("\n"))}</textarea>
             <div class="edit-hint">One step per line.</div>
           </div>
         </div>
         <div class="recipe-edit-actions">
           <button class="btn-edit" onclick="saveRecipeEdit('${k}')">💾 Save</button>
           <button class="btn-reset" onclick="cancelRecipeEdit('${k}')">Cancel</button>
           ${recipeEdits[k] ? `<button class="btn-reset" onclick="resetRecipeEdit('${k}')">↺ Reset to original</button>` : ""}
         </div>
       </article>`;
  } else {
    recipeDetailScale = scale;
    const rd=recipeData(k);
    document.getElementById("recipeDetail").innerHTML =
      `<button class="btn-reset" style="margin-bottom:12px" onclick="hideRecipeDetail()">← All recipes</button>
       <article class="recipe">
         <h3>${r.title}</h3>
         <div class="meta">Serves ${r.serves} · Source: ${link}</div>
         ${scaleNote}
         <button class="btn-edit" style="margin:8px 0" onclick="editRecipe('${k}')">✏️ Edit ingredients & steps</button>
         <div class="cols"><div><h4>Ingredients</h4>${ingHTML(rd.ing)}</div>
         <div><h4>Instructions</h4><ol class="steps">${rd.steps.map(s=>`<li>${escHTML(s)}</li>`).join("")}</ol>${note}</div></div>
       </article>`;
  }
  document.getElementById("recipeIndex").style.display="none";
  document.getElementById("recipeDetail").style.display="block";
  window.scrollTo({top:0,behavior:"smooth"});
}
```

Replace with:

```js
function showRecipeDetail(k,scale){
  scale = scale||1;
  const r=R[k];
  const link = r.url ? `<a href="${r.url}" target="_blank" rel="noopener">${r.src} ↗</a>` : r.src;
  const note = r.note ? `<div class="note-box">💡 ${r.note}</div>` : "";
  const scaleNote = (scale!==1) ? `<div class="note-box">👥 ${scale}x tonight — grocery list already adjusted.</div>` : "";
  const rd=recipeData(k);
  document.getElementById("recipeDetail").innerHTML =
    `<button class="btn-reset" style="margin-bottom:12px" onclick="hideRecipeDetail()">← All recipes</button>
     <article class="recipe">
       <h3>${r.title}</h3>
       <div class="meta">Serves ${r.serves} · Source: ${link}</div>
       ${scaleNote}
       <div class="cols"><div><h4>Ingredients</h4>${ingHTML(rd.ing)}</div>
       <div><h4>Instructions</h4><ol class="steps">${rd.steps.map(s=>`<li>${escHTML(s)}</li>`).join("")}</ol>${note}</div></div>
     </article>`;
  document.getElementById("recipeIndex").style.display="none";
  document.getElementById("recipeDetail").style.display="block";
  window.scrollTo({top:0,behavior:"smooth"});
}
```

- [ ] **Step 5: Drop the now-undefined `recipeEditKey` reference in `openRecipe`**

Current code:

```js
function openRecipe(k,slot,i){
  document.querySelectorAll("#tabs button").forEach(x=>x.classList.remove("active"));
  var tb=document.querySelector('#tabs button[data-t="recipes"]'); if(tb) tb.classList.add("active");
  document.querySelectorAll("section.panel").forEach(p=>p.classList.remove("active"));
  document.getElementById("recipes").classList.add("active");
  recipeEditKey=null;
  const scale = (slot!=null && i!=null) ? mealScale[slot][i] : 1;
  showRecipeDetail(k, scale);
}
```

Replace with:

```js
function openRecipe(k,slot,i){
  document.querySelectorAll("#tabs button").forEach(x=>x.classList.remove("active"));
  var tb=document.querySelector('#tabs button[data-t="recipes"]'); if(tb) tb.classList.add("active");
  document.querySelectorAll("section.panel").forEach(p=>p.classList.remove("active"));
  document.getElementById("recipes").classList.add("active");
  const scale = (slot!=null && i!=null) ? mealScale[slot][i] : 1;
  showRecipeDetail(k, scale);
}
```

- [ ] **Step 6: Manual verification**

Serve locally, console: `syncRef=null;`.

1. Open the **Recipes** tab. Expected: the index renders (no console error), and every one of the 21 recipes opens correctly showing its ingredients and steps as before — text reads identically to production except the handful of intentionally-reworded/split lines from Task 3 (padthai's sauce line, thainoodles' rice-noodles/chicken-breast/garnish lines, stirfry's garnish split, tacobowls' appended sour-cream/oil lines).
2. Confirm there is **no** "✏️ Edit ingredients & steps" button anywhere — that's expected, not a bug, until Part B ships.
3. Open **Cycle A Recipes** and **Cycle B Recipes** tabs — same check, every planned recipe renders correctly.
4. Set a night to 2x in Edit Plan mode and confirm the "👥 2x tonight" note still renders correctly on both the day-card recipe view and the detail page (this logic was untouched by this task, just confirming nothing else broke).
5. Run `typeof recipeEditKey` and `typeof editRecipe` in the console → expect `"undefined"` for both (confirms full removal, no leftover references anywhere else in the file that would throw).

Grocery tabs are still broken at this point (Task 5 fixes that) — don't worry about them in this task's verification.

- [ ] **Step 7: Commit**

```bash
cd "C:/Users/Matt/GitHub/dinner-meal-plan"
git add index.html
git commit -m "Update recipe rendering for structured ingredients; remove incompatible free-text editor"
```

---

### Task 5: Rewrite the grocery calculation around recipe-owned ingredient links

**Files:**
- Modify: `index.html` (functions `qtyOf` and `effectiveGrocery`, currently lines 1011–1046; leave `unitLabel`/`fmtQty`/`qtyLabel` between them untouched)

**Context:** `qtyOf` is only ever called from `effectiveGrocery` (confirmed — no other call site in the file), so this task removes `qtyOf` as a standalone function and folds its rounding logic directly into the rewritten `effectiveGrocery`, which now sums contributions from each planned recipe's own linked ingredient rows instead of asking each catalog item for its `per` map (which no longer exists, per Task 2).

- [ ] **Step 1: Replace `qtyOf` and `effectiveGrocery`**

Current code:

```js
function qtyOf(item,occurrences){
  if(item.always) return 1;
  let q=0; const per=item.per||{};
  occurrences.forEach(o=>{ if(per[o.key]!=null) q+=per[o.key]*o.scale; });
  if(q<=0) return 0;
  return item.divisible ? Math.ceil(q*2)/2 : Math.ceil(q-1e-9);
}
function unitLabel(unit,q){
  if(unit==="lb") return "lb";
  if(q===1) return unit;
  if(/o$/.test(unit)) return unit+"es";            // tomato -> tomatoes
  if(/(ch|sh|s|x|z)$/.test(unit)) return unit+"es"; // bunch -> bunches, box -> boxes
  return unit+"s";
}
function fmtQty(q){ return (Math.round(q)===q) ? String(q) : q.toFixed(1); }
function qtyLabel(e){
  if(e.item.always) return e.item.note||"as needed";
  const used = (e.used&&e.used.length) ? " — "+e.used.join(", ") : "";
  return `${fmtQty(e.qty)} ${unitLabel(e.item.unit,e.qty)}${used}`;
}
// Build the cycle's grocery list from its planned meals, honoring each night's serving scale.
// Each occurrence of a recipe counts separately, even if the same recipe appears on two different nights.
function effectiveGrocery(slot){
  const occurrences=(mealPlan[slot]||[])
    .map((k,i)=> (k && k!==LEFTOVERS) ? {key:k, scale:(mealScale[slot][i]||1)} : null)
    .filter(Boolean);
  const sections=[], idx={};
  function sec(name){ if(idx[name]==null){ idx[name]=sections.length; sections.push({sec:name,items:[]}); } return sections[idx[name]]; }
  GITEMS.forEach(item=>{
    const qty=qtyOf(item,occurrences);
    if(qty<=0) return;
    const used=[...new Set(occurrences.filter(o=>item.per && item.per[o.key]!=null).map(o=>o.key))].map(k=>R[k]&&R[k].title).filter(Boolean);
    sec(item.sec).items.push({item, qty, used});
  });
  return sections;
}
```

Replace `qtyOf` and `effectiveGrocery` (leave `unitLabel`/`fmtQty`/`qtyLabel` exactly as they are, in between) with:

```js
function flattenRows(ing){ return ing.flatMap(g=>g.rows); }
// Build the cycle's grocery list: sum each planned night's linked ingredient
// amounts (scaled by that night's serving multiplier) by catalog item name,
// then round each item's total up to whole purchase units.
function effectiveGrocery(slot){
  const occurrences=(mealPlan[slot]||[])
    .map((k,i)=> (k && k!==LEFTOVERS) ? {key:k, scale:(mealScale[slot][i]||1)} : null)
    .filter(Boolean);
  const contrib={}, usedBy={};
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
    const used = item.always ? [] : [...(usedBy[item.n]||[])].map(k=>R[k]&&R[k].title).filter(Boolean);
    sec(item.sec).items.push({item, qty, used});
  });
  return sections;
}
```

- [ ] **Step 2: Manual verification**

Serve locally, console: `syncRef=null;`.

1. Go to **Cycle A Grocery** and **Cycle B Grocery**. Expected: both render fully populated lists again (no longer broken/empty), with sections, quantities, and prices.
2. Run `allItems().find(i=>i.n==="Fish sauce")` → still exists in the catalog, but confirm `effectiveGrocery('A')`/`effectiveGrocery('B')` never include it (it has zero recipe links now that Pad Thai's sauce was swapped) — this is expected, not a bug.
3. Confirm "Spice-rack restock" still appears on both grocery lists regardless of what's planned (it's `always:true`, independent of any recipe link).
4. In Edit Plan mode, set a night to 2x and confirm the linked ingredients for that recipe roughly double in the grocery list (same behavior as before this migration, just now driven by the recipe's own rows instead of `GITEMS.per`).
5. Do not yet compare against Task 1's baseline in detail — that's Task 6. This step is just confirming the grocery tabs render and respond to scale changes again.

- [ ] **Step 3: Commit**

```bash
cd "C:/Users/Matt/GitHub/dinner-meal-plan"
git add index.html
git commit -m "Rewrite grocery calculation to sum recipe-owned ingredient links"
```

---

### Task 6: Verify the migration against the pre-migration baseline

**Files:** none (verification only)

- [ ] **Step 1: Re-snapshot and diff**

Serve locally, console: `syncRef=null;`.

Run the same command from Task 1:

```js
JSON.stringify({A: effectiveGrocery('A'), B: effectiveGrocery('B')}, (k,v)=> k==='item' ? undefined : v, 1)
```

Compare against the baseline saved in Task 1. For every item name that does **not** appear in any linked row of `thainoodles`, `tacobowls`, or `padthai`, `qty` and `used` must be byte-identical between the two snapshots — the migration is only allowed to change grocery totals for items connected to those three recipes. If anything else differs, STOP and report BLOCKED with the specific item and both values — that means a matching error happened during Task 3's recipe-by-recipe migration.

- [ ] **Step 2: Directly verify the three intentionally-changed recipes**

The current 2-week plan may or may not currently include `thainoodles`/`tacobowls`/`padthai` — if none of them are planned right now, Step 1's diff won't show any difference at all (correctly — nothing changed for the currently-planned recipes), so this step exercises them directly regardless of what's currently planned.

Run in the console (this only reads/computes, but assigns to `mealPlan` temporarily — make sure `syncRef` is `null` before running this):

```js
const before = JSON.parse(JSON.stringify(mealPlan.A));
mealPlan.A = ['thainoodles','tacobowls','padthai', ...new Array(11).fill(null)];
mealScale.A = new Array(14).fill(1);
const groc = effectiveGrocery('A');
const flat = {};
groc.forEach(s=>s.items.forEach(e=>{ flat[e.item.n] = {qty:e.qty, used:e.used}; }));
mealPlan.A = before;  // restore immediately, don't leave the plan in this test state
JSON.stringify(flat, null, 1);
```

Confirm by hand:

1. **Chicken breast** is present (from Thai Noodles' new chicken-breast link) — before this migration, planning only these three recipes would never have included chicken breast at all for Thai Noodles.
2. **Flat rice noodles** is present with `qty: 2` (Thai Noodles' 1 lb ≈ two 8 oz packs) and, separately, that quantity also reflects Pad Thai's own 1-pack need if you look at `used` — both recipes should be listed under `used` for this item.
3. **Linguine** is **absent** (Thai Noodles no longer needs it).
4. **Pad Thai Sauce** is present with `qty: 1`.
5. **Fish sauce**, **Rice vinegar**, **Creamy peanut butter** are all **absent** (Pad Thai no longer needs any of them, and no other recipe ever did).
6. **Sour cream** is present (Taco Bowls' appended row) and **Olive oil** is present (Taco Bowls' appended row, plus whatever baseline amount other recipes already contributed).

- [ ] **Step 3: Spot-check rounding/pricing on one item**

Pick any item that appears in both snapshots with an unchanged `qty` (e.g. one of the staples) and confirm its price on the **Prices** tab still shows the same value it did before this migration — pricing data (`prices`/`staplePrices`) was never touched by this plan, only the recipe→item association was, so this should be trivially unchanged. This is a sanity check that nothing in Task 2's `GITEMS` rewrite accidentally altered a price, unit, or section along with removing `per`.

- [ ] **Step 4: No commit** — this task is verification-only. If Steps 1–3 all pass, Part A is complete and ready for Part B. Do not push to origin yet (per the note at the top of this plan) — Part B still needs to land first.

---

## Spec coverage check (Part A's share of the design)

- Unified `{text,item,qty}` ingredient row shape → Task 3.
- `GITEMS` loses `per`, becomes a pure catalog → Task 2.
- Migration of all 21 built-in recipes, using each recipe's live effective text (not stale file text) → Task 3, informed by the read-only production pull described in the chat history for this plan.
- Known Spicy Thai Noodles fix (chicken breast + rice noodles) folded into the migration → Task 3.
- Pad Thai Sauce catalog item added to cover the user's sauce-swap edit → Task 2.
- Grocery calculation inverted to sum from recipe-owned links → Task 5.
- Migration verified against exact pre-migration totals → Tasks 1 and 6.
- `customItems`/`allItems()` plumbing (used by Part B, harmless no-op until then) → Task 2.

Not covered here (Part B's share): `customRecipes`, the ingredient builder UI, recipe creation/deletion, new-catalog-item creation UI, and Firebase sync for the two new stores.

