# Meal Plan Editing Enhancements Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix the meal-picker scroll bug, move checked grocery items into a single bottom "Purchased" pile, let recipes' ingredients/steps be edited in place, and add a per-night 1x/1.5x/2x/3x serving multiplier that feeds the grocery calculation.

**Architecture:** All four changes land in the single existing file, `index.html` — no new files, no build step. New state (`recipeEdits`, `mealScale`) is added additively alongside the existing `checks`/`custom`/`mealPlan` state, following the exact same load/save/sync pattern already used for those, and is synced through the same shared Firebase envelope. Nothing already stored changes shape.

**Tech Stack:** Vanilla JS, inline `<style>`, Firebase Realtime Database (existing `firebase-database-compat.js` v10.12.0), no framework, no bundler.

**Spec:** `docs/superpowers/specs/2026-07-29-meal-plan-editing-enhancements-design.md`

---

## Note on testing in this codebase

This app has **no automated test suite** — it's a single static HTML file with
no build step, and adding a test framework now would be unrelated scope
creep. Per the approved spec's "Testing approach" section, verification is
manual, in a browser, against a local server — **never against the live
preview with sync live**, since `index.html` connects to a real, shared
Firebase Realtime Database on load and any mutating action pushes to it
within ~400ms with no undo (this cost the user real data once before).

So every task below replaces the usual "write failing test → make it pass"
steps with:

1. Implement the change.
2. Serve the file locally and verify manually in the browser **with sync
   stubbed out** (open the console and run `syncRef=null;` immediately after
   the page loads, before touching anything else — this disables the
   `scheduleSyncPush()` write path for the rest of that browser session
   while leaving everything else, including the initial read, functional).
3. Commit.

A local server config already exists at `aretestays-artifacts/.claude/launch.json`
(python http.server) — for this repo, run the equivalent directly:

```bash
cd "C:/Users/Matt/GitHub/dinner-meal-plan" && python -m http.server 8765
```

Then open `http://localhost:8765/` in a browser, open the developer console,
and run `syncRef=null;` before doing anything else in that tab.

## Note on line numbers

Line numbers cited below reflect `index.html` as it exists at the start of
this plan (before Task 1). Because every task edits the same file, line
numbers drift after each earlier task lands. Locate each edit by the exact
code shown (the `old_string`/current-code block), not by line number, once
you're past Task 1.

---

## File Structure

Everything below modifies the single existing file:

- Modify: `C:\Users\Matt\GitHub\dinner-meal-plan\index.html`
  - `<style>` block (CSS) — new rules for the Purchased grocery section, the
    recipe edit textareas, and the per-day scale buttons.
  - Storage/state section (~line 673-699) — new `ckScale`, `mealScale`,
    `recipeEdits` globals and their save helpers.
  - Grocery section (~line 907-1007) — `qtyOf`/`effectiveGrocery` rewritten
    to use per-night scale; `renderGrocery` rewritten for the Purchased pile.
  - Meal-plan mutators (~line 772-831) — `dayCard`, `changeMeal`,
    `moveMeal`, `removeMeal`, `sendOther`, `resetMealPlan`, plus new
    `setScale`.
  - Meal picker (~line 833-872) — scroll-close bug fix only.
  - Recipes section (~line 874-905, 1061-1094) — `recipeData` helper,
    `recipeCard`, `recipesHTML`, `showRecipeDetail`, `openRecipe`, plus new
    edit-mode functions.
  - Firebase sync (~line 1107-1178) — `applyRemote`/`syncPush` extended for
    `recipeEdits` and `mealScale`.

No other files are created or touched.

---

### Task 1: Fix meal-picker scroll-close bug

**Files:**
- Modify: `index.html` (function attached via `window.addEventListener("scroll", ...)`, currently line 872)

- [ ] **Step 1: Replace the scroll listener**

Current code (line 872):

```js
window.addEventListener("scroll",closeMealPicker,true);
```

Replace with:

```js
window.addEventListener("scroll",function(e){
  const p=document.getElementById("mealPicker");
  if(p && p.style.display==="block" && p.contains(e.target)) return;
  closeMealPicker();
},true);
```

- [ ] **Step 2: Manual verification**

```bash
cd "C:/Users/Matt/GitHub/dinner-meal-plan" && python -m http.server 8765
```

Open `http://localhost:8765/`, console: `syncRef=null;`. Then:

1. Click **✏️ Edit plan**.
2. Click any day's meal dropdown to open the searchable picker.
3. Scroll inside the picker's option list (mouse wheel or trackpad, over the
   list itself, not the page).

Expected: the list scrolls and the picker **stays open**. Then click
somewhere outside the picker — expected: it closes. Then open it again and
scroll the page behind it (e.g. drag the outer scrollbar) — expected: the
picker closes, same as before this fix.

- [ ] **Step 3: Commit**

```bash
cd "C:/Users/Matt/GitHub/dinner-meal-plan"
git add index.html
git commit -m "Fix meal picker closing when scrolling its own option list"
```

---

### Task 2: Grocery list — checked items collapse into a bottom "Purchased" pile

**Files:**
- Modify: `index.html` (CSS in `<style>`, ~line 148-166; function `renderGrocery`, currently lines 946-986)

- [ ] **Step 1: Add CSS for the Purchased section**

Insert after the existing `.cust-price-in{...}` rule (~line 166), before
`.staple-card{...}`:

```css
.gsection.purchased > h4{background:#f1f0ec}
```

- [ ] **Step 2: Replace `renderGrocery` with a version that partitions checked items out**

Current code (lines 946-986):

```js
function renderGrocery(slot){
  const groc = effectiveGrocery(slot);
  const container = document.getElementById("groceryList"+slot);
  let html="";
  groc.forEach((section)=>{
    if(!section.items.length) return;
    let subtotal=0;
    const entries = section.items.map((e)=>{ const id=gid(section.sec,e.item.n), p=priceOf(e.item,e.qty); subtotal+=p; return {item:e.item, qty:e.qty, used:e.used, id, p, checked:!!checks[slot][id]}; });
    entries.sort((a,b)=>(a.checked?1:0)-(b.checked?1:0));   // unchecked rise, checked sink
    const rows = entries.map(function(e){
      const ck = e.checked?"checked":"", li = e.checked?"checked":"";
      const flag = e.item.staple?`<span class="staple-flag">STAPLE</span>`:"";
      const pc = e.item.staple?"price staple":"price";
      const st = storeOf(e.item); const stLabel = st?` · 🏬 ${st}`:"";
      return `<li class="${li}" data-id="${e.id}">
        <input type="checkbox" ${ck} onchange="toggleCheck('${slot}','${e.id}',this)">
        <label><span class="nm">${e.item.n}${flag}</span><span class="qty">${qtyLabel(e)}${stLabel}</span></label>
        <span class="${pc}">${money(e.p)}</span></li>`;
    }).join("");
    html+=`<div class="gsection"><h4>${section.sec}<span class="sub">${money(subtotal)}</span></h4><ul class="glist">${rows}</ul></div>`;
  });
  // custom items
  if(custom[slot].length){
    let csub=0;
    const centries = custom[slot].map(item=>{ const id="c-"+item.id, p=item.price||0; csub+=p; return {item,id,checked:!!checks[slot][id]}; });
    centries.sort((a,b)=>(a.checked?1:0)-(b.checked?1:0));
    const rows = centries.map(function(e){
      const item=e.item, id=e.id;
      const ck = e.checked?"checked":"", li = e.checked?"checked":"";
      return `<li class="${li}" data-id="${id}">
        <input type="checkbox" ${ck} onchange="toggleCheck('${slot}','${id}',this)">
        <label><span class="nm">${item.n}</span><span class="qty">${item.q||"added by you"}</span></label>
        <input class="cust-price-in" type="number" step="0.01" min="0" value="${(item.price||0).toFixed(2)}"
          onchange="editCustomPrice('${slot}','${item.id}',this.value)">
        <button class="remove-x" title="Remove" onclick="removeCustom('${slot}','${item.id}')">✕</button></li>`;
    }).join("");
    html+=`<div class="gsection custom"><h4>🛒 Added by You<span class="sub">${money(csub)}</span></h4><ul class="glist">${rows}</ul></div>`;
  }
  container.innerHTML=html;
  updateTotals(slot);
}
```

Replace the whole function (and add the two small row-builder helpers it now
uses) with:

```js
function groceryRowHTML(slot, e){
  const ck = e.checked?"checked":"", li = e.checked?"checked":"";
  const flag = e.item.staple?`<span class="staple-flag">STAPLE</span>`:"";
  const pc = e.item.staple?"price staple":"price";
  const st = storeOf(e.item); const stLabel = st?` · 🏬 ${st}`:"";
  return `<li class="${li}" data-id="${e.id}">
    <input type="checkbox" ${ck} onchange="toggleCheck('${slot}','${e.id}',this)">
    <label><span class="nm">${e.item.n}${flag}</span><span class="qty">${qtyLabel(e)}${stLabel}</span></label>
    <span class="${pc}">${money(e.p)}</span></li>`;
}
function customRowHTML(slot, e){
  const item=e.item, id=e.id;
  const ck = e.checked?"checked":"", li = e.checked?"checked":"";
  return `<li class="${li}" data-id="${id}">
    <input type="checkbox" ${ck} onchange="toggleCheck('${slot}','${id}',this)">
    <label><span class="nm">${item.n}</span><span class="qty">${item.q||"added by you"}</span></label>
    <input class="cust-price-in" type="number" step="0.01" min="0" value="${(item.price||0).toFixed(2)}"
      onchange="editCustomPrice('${slot}','${item.id}',this.value)">
    <button class="remove-x" title="Remove" onclick="removeCustom('${slot}','${item.id}')">✕</button></li>`;
}
function renderGrocery(slot){
  const groc = effectiveGrocery(slot);
  const container = document.getElementById("groceryList"+slot);
  let html="";
  const purchased=[]; // {row, p} for every checked item, page-wide
  groc.forEach((section)=>{
    if(!section.items.length) return;
    const entries = section.items.map((e)=>{ const id=gid(section.sec,e.item.n), p=priceOf(e.item,e.qty); return {item:e.item, qty:e.qty, used:e.used, id, p, checked:!!checks[slot][id]}; });
    const unchecked = entries.filter(e=>!e.checked);
    entries.filter(e=>e.checked).forEach(e=>purchased.push({row:groceryRowHTML(slot,e), p:e.p}));
    if(!unchecked.length) return;
    const subtotal = unchecked.reduce((sum,e)=>sum+e.p,0);
    const rows = unchecked.map(e=>groceryRowHTML(slot,e)).join("");
    html+=`<div class="gsection"><h4>${section.sec}<span class="sub">${money(subtotal)}</span></h4><ul class="glist">${rows}</ul></div>`;
  });
  // custom items
  if(custom[slot].length){
    const centries = custom[slot].map(item=>{ const id="c-"+item.id, p=item.price||0; return {item,id,p,checked:!!checks[slot][id]}; });
    const unchecked = centries.filter(e=>!e.checked);
    centries.filter(e=>e.checked).forEach(e=>purchased.push({row:customRowHTML(slot,e), p:e.p}));
    if(unchecked.length){
      const csub = unchecked.reduce((sum,e)=>sum+e.p,0);
      const rows = unchecked.map(e=>customRowHTML(slot,e)).join("");
      html+=`<div class="gsection custom"><h4>🛒 Added by You<span class="sub">${money(csub)}</span></h4><ul class="glist">${rows}</ul></div>`;
    }
  }
  if(purchased.length){
    const psub = purchased.reduce((sum,e)=>sum+e.p,0);
    html+=`<div class="gsection purchased"><h4>✓ Purchased<span class="sub">${money(psub)}</span></h4><ul class="glist">${purchased.map(e=>e.row).join("")}</ul></div>`;
  }
  container.innerHTML=html;
  updateTotals(slot);
}
```

- [ ] **Step 3: Manual verification**

Serve locally (as in Task 1), console: `syncRef=null;`. Go to **Cycle A
Grocery**:

1. Check one item in "🥩 Meat & Seafood" and one in "🥦 Produce".
2. Expected: both items disappear from their category and reappear at the
   bottom of the page under a new **✓ Purchased** section; each category's
   header total drops to reflect only what's left unchecked in it; the
   Purchased header shows the combined total of both checked items.
3. Check every remaining item in one whole category until it's empty.
   Expected: that category's section (header included) disappears entirely
   from the top.
4. Uncheck an item from the Purchased pile. Expected: it moves back to its
   original category section, in that section's normal (unsorted) order.
5. Check a custom "Added by You" item. Expected: it also lands in Purchased.
6. Confirm the "Full List" / "Still to Buy" / "Checked" totals at the top of
   the page are unchanged in behavior (they read from `updateTotals`, not
   touched by this change).

- [ ] **Step 4: Commit**

```bash
cd "C:/Users/Matt/GitHub/dinner-meal-plan"
git add index.html
git commit -m "Move checked grocery items into one bottom Purchased section"
```

---

### Task 3: Recipe data model + wire Cycle A/B recipe cards to it

**Files:**
- Modify: `index.html` (insert new state near line 874; modify function `recipeCard`, currently lines 881-892)

- [ ] **Step 1: Add the `recipeEdits` state and `recipeData` helper**

Insert immediately after the `/* ============================ RECIPES ============================ */`
comment (line 874), before `function ingHTML(groups){`:

```js
let recipeEdits = load('mp_recipeEdits_v1', {});   // { [recipeKey]: {ing, steps} } — user overrides, layered on top of R
function saveRecipeEdits(){ save('mp_recipeEdits_v1', recipeEdits); }
function recipeData(k){
  const base=R[k], edit=recipeEdits[k];
  if(!edit) return base;
  return Object.assign({}, base, { ing: edit.ing || base.ing, steps: edit.steps || base.steps });
}
```

- [ ] **Step 2: Switch `recipeCard` to render via `recipeData`**

Current code (lines 881-892):

```js
function recipeCard(k,dayNo,date){
  const r=R[k];
  const link = r.url ? `<a href="${r.url}" target="_blank" rel="noopener">${r.src} ↗</a>` : r.src;
  const note = r.note ? `<div class="note-box">💡 ${r.note}</div>` : "";
  return `<article class="recipe">
    <span class="day-tag">Day ${dayNo} · ${fmt(date)}</span>
    <h3>${r.title}</h3>
    <div class="meta">Serves ${r.serves} · Source: ${link}</div>
    <div class="cols">
      <div><h4>Ingredients</h4>${ingHTML(r.ing)}</div>
      <div><h4>Instructions</h4><ol class="steps">${r.steps.map(s=>`<li>${s}</li>`).join("")}</ol>${note}</div>
    </div></article>`;
}
```

Replace with:

```js
function recipeCard(k,dayNo,date){
  const r=R[k], rd=recipeData(k);
  const link = r.url ? `<a href="${r.url}" target="_blank" rel="noopener">${r.src} ↗</a>` : r.src;
  const note = r.note ? `<div class="note-box">💡 ${r.note}</div>` : "";
  return `<article class="recipe">
    <span class="day-tag">Day ${dayNo} · ${fmt(date)}</span>
    <h3>${r.title}</h3>
    <div class="meta">Serves ${r.serves} · Source: ${link}</div>
    <div class="cols">
      <div><h4>Ingredients</h4>${ingHTML(rd.ing)}</div>
      <div><h4>Instructions</h4><ol class="steps">${rd.steps.map(s=>`<li>${s}</li>`).join("")}</ol>${note}</div>
    </div></article>`;
}
```

(No behavior change yet — `recipeEdits` is always `{}` at this point, so
`recipeData(k)` returns `R[k]` unchanged. This just wires the plumbing.)

- [ ] **Step 3: Manual verification**

Serve locally, console: `syncRef=null;`. Open **Cycle A Recipes** and **Cycle
B Recipes** tabs. Expected: every recipe card renders exactly as before
(ingredients and steps unchanged) — this step only proves the refactor is a
no-op so far.

- [ ] **Step 4: Commit**

```bash
cd "C:/Users/Matt/GitHub/dinner-meal-plan"
git add index.html
git commit -m "Add recipeEdits data layer and route recipe cards through it"
```

---

### Task 4: Recipe edit UI (ingredients + steps) on the recipe detail page

**Files:**
- Modify: `index.html` (CSS in `<style>`, ~line 124; function `showRecipeDetail`, currently lines 1068-1083; new functions near it)

- [ ] **Step 1: Add CSS for the edit textareas and action row**

Insert after the existing `.note-box{...}` rule (~line 124), before
`.grocery-intro{...}`:

```css
.recipe-edit-box{width:100%;box-sizing:border-box;border:1px solid var(--line);border-radius:8px;
  padding:10px;font-family:inherit;font-size:.88rem;line-height:1.5;resize:vertical;color:var(--ink)}
.edit-hint{font-size:.75rem;color:var(--muted);margin:4px 0 12px}
.recipe-edit-actions{display:flex;gap:8px;margin-top:14px;flex-wrap:wrap}
```

- [ ] **Step 2: Add parse/serialize helpers and edit-mode state**

Insert directly after the `recipeData` function added in Task 3:

```js
let recipeEditKey = null;   // recipe key currently in edit mode on the detail page, or null
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
function editRecipe(k){ recipeEditKey=k; showRecipeDetail(k); }
function cancelRecipeEdit(k){ recipeEditKey=null; showRecipeDetail(k); }
function saveRecipeEdit(k){
  const ingBox=document.getElementById('recipeEditIng'), stepBox=document.getElementById('recipeEditSteps');
  recipeEdits[k] = { ing: parseIngredients(ingBox.value), steps: parseSteps(stepBox.value) };
  saveRecipeEdits();
  recipeEditKey=null;
  showRecipeDetail(k);
  renderRecipes();
  toast("Recipe saved");
}
function resetRecipeEdit(k){
  delete recipeEdits[k];
  saveRecipeEdits();
  recipeEditKey=null;
  showRecipeDetail(k);
  renderRecipes();
  toast("Reset to original");
}
```

- [ ] **Step 3: Rewrite `showRecipeDetail` to support edit mode**

Current code (lines 1068-1083):

```js
function showRecipeDetail(k){
  const r=R[k];
  const link = r.url ? `<a href="${r.url}" target="_blank" rel="noopener">${r.src} ↗</a>` : r.src;
  const note = r.note ? `<div class="note-box">💡 ${r.note}</div>` : "";
  document.getElementById("recipeDetail").innerHTML =
    `<button class="btn-reset" style="margin-bottom:12px" onclick="hideRecipeDetail()">← All recipes</button>
     <article class="recipe">
       <h3>${r.title}</h3>
       <div class="meta">Serves ${r.serves} · Source: ${link}</div>
       <div class="cols"><div><h4>Ingredients</h4>${ingHTML(r.ing)}</div>
       <div><h4>Instructions</h4><ol class="steps">${r.steps.map(s=>`<li>${s}</li>`).join("")}</ol>${note}</div></div>
     </article>`;
  document.getElementById("recipeIndex").style.display="none";
  document.getElementById("recipeDetail").style.display="block";
  window.scrollTo({top:0,behavior:"smooth"});
}
```

Replace with:

```js
function showRecipeDetail(k){
  const r=R[k];
  const link = r.url ? `<a href="${r.url}" target="_blank" rel="noopener">${r.src} ↗</a>` : r.src;
  const note = r.note ? `<div class="note-box">💡 ${r.note}</div>` : "";
  if(recipeEditKey===k){
    const rd=recipeData(k);
    document.getElementById("recipeDetail").innerHTML =
      `<button class="btn-reset" style="margin-bottom:12px" onclick="cancelRecipeEdit('${k}')">← Cancel</button>
       <article class="recipe">
         <h3>${r.title}</h3>
         <div class="meta">Serves ${r.serves} · Source: ${link}</div>
         <div class="cols">
           <div><h4>Ingredients</h4>
             <textarea id="recipeEditIng" class="recipe-edit-box" rows="12">${ingredientsToText(rd.ing)}</textarea>
             <div class="edit-hint">Start a line with "## " for a section heading (e.g. "## Marinade").</div>
           </div>
           <div><h4>Instructions</h4>
             <textarea id="recipeEditSteps" class="recipe-edit-box" rows="12">${rd.steps.join("\n")}</textarea>
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
    const rd=recipeData(k);
    document.getElementById("recipeDetail").innerHTML =
      `<button class="btn-reset" style="margin-bottom:12px" onclick="hideRecipeDetail()">← All recipes</button>
       <article class="recipe">
         <h3>${r.title}</h3>
         <div class="meta">Serves ${r.serves} · Source: ${link}</div>
         <button class="btn-edit" style="margin:8px 0" onclick="editRecipe('${k}')">✏️ Edit ingredients & steps</button>
         <div class="cols"><div><h4>Ingredients</h4>${ingHTML(rd.ing)}</div>
         <div><h4>Instructions</h4><ol class="steps">${rd.steps.map(s=>`<li>${s}</li>`).join("")}</ol>${note}</div></div>
       </article>`;
  }
  document.getElementById("recipeIndex").style.display="none";
  document.getElementById("recipeDetail").style.display="block";
  window.scrollTo({top:0,behavior:"smooth"});
}
```

- [ ] **Step 4: Manual verification**

Serve locally, console: `syncRef=null;`. Go to **Recipes** tab (the "All
Recipes" index) and open any recipe:

1. Click **✏️ Edit ingredients & steps**. Expected: ingredients and steps
   become editable textareas, pre-filled with the current content, headings
   prefixed with `## `.
2. Change an ingredient line, add a new one, add a new `## Heading` group,
   edit a step. Click **Save**. Expected: toast "Recipe saved", view returns
   to read mode showing your edits.
3. Go to that recipe's day in **Cycle A Recipes** (or B). Expected: the same
   edited ingredients/steps show there too.
4. Reopen the recipe detail, click Edit again — expected: an additional
   **↺ Reset to original** button now appears (it didn't before you'd ever
   edited this recipe). Click it. Expected: content reverts to the
   hardcoded original, button disappears again.
5. Edit again, click **Cancel** instead of Save. Expected: your in-progress
   textarea changes are discarded, original (or previously-saved-edit)
   content is shown unchanged.
6. Reload the page (still with `syncRef=null;` run again after reload).
   Expected: any saved edit persisted (confirms `localStorage` round-trip).

- [ ] **Step 5: Commit**

```bash
cd "C:/Users/Matt/GitHub/dinner-meal-plan"
git add index.html
git commit -m "Add inline recipe ingredient/step editing"
```

---

### Task 5: Sync recipe edits through Firebase

**Files:**
- Modify: `index.html` (function `applyRemote`, currently lines 1130-1155; function `syncPush`, currently lines 1157-1165)

- [ ] **Step 1: Add `recipeEdits` to `applyRemote`**

Current code (lines 1130-1155) — only the relevant lines shown, locate by
the `remoteMealsAll  = s.meals||{};` line as your anchor:

```js
    remoteChecksAll = s.checks||{};
    remoteCustomAll = s.custom||{};
    remoteMealsAll  = s.meals||{};
    checks.A = remoteChecksAll[CTX.A.key]||{};  checks.B = remoteChecksAll[CTX.B.key]||{};
```

Replace with:

```js
    remoteChecksAll = s.checks||{};
    remoteCustomAll = s.custom||{};
    remoteMealsAll  = s.meals||{};
    recipeEdits = s.recipeEdits||{};
    checks.A = remoteChecksAll[CTX.A.key]||{};  checks.B = remoteChecksAll[CTX.B.key]||{};
```

Then, in the same function, current code:

```js
    save(ckMeals(CTX.A.key),mealPlan.A); save(ckMeals(CTX.B.key),mealPlan.B);
    renderStapleRows(); renderCatalog(); renderOverview(); renderRecipes(); renderGrocery("A"); renderGrocery("B");
```

Replace with:

```js
    save(ckMeals(CTX.A.key),mealPlan.A); save(ckMeals(CTX.B.key),mealPlan.B);
    saveRecipeEdits();
    renderStapleRows(); renderCatalog(); renderOverview(); renderRecipes(); renderGrocery("A"); renderGrocery("B");
```

- [ ] **Step 2: Add `recipeEdits` to `syncPush`**

Current code (lines 1157-1165):

```js
function syncPush(){
  if(!syncRef) return;
  remoteChecksAll[CTX.A.key]=checks.A; remoteChecksAll[CTX.B.key]=checks.B;
  remoteCustomAll[CTX.A.key]=custom.A; remoteCustomAll[CTX.B.key]=custom.B;
  remoteMealsAll[CTX.A.key]=mealPlan.A; remoteMealsAll[CTX.B.key]=mealPlan.B;
  myRev = DEVICE+"-"+(++seq);
  const state = JSON.stringify({staples:staplePrices, prices:prices, store:storeSel, unit:unitSel, checks:remoteChecksAll, custom:remoteCustomAll, meals:remoteMealsAll});
  syncRef.set({rev:myRev, updatedAt:Date.now(), state:state}).catch(function(e){ console.warn('sync push failed',e); });
}
```

Replace with:

```js
function syncPush(){
  if(!syncRef) return;
  remoteChecksAll[CTX.A.key]=checks.A; remoteChecksAll[CTX.B.key]=checks.B;
  remoteCustomAll[CTX.A.key]=custom.A; remoteCustomAll[CTX.B.key]=custom.B;
  remoteMealsAll[CTX.A.key]=mealPlan.A; remoteMealsAll[CTX.B.key]=mealPlan.B;
  myRev = DEVICE+"-"+(++seq);
  const state = JSON.stringify({staples:staplePrices, prices:prices, store:storeSel, unit:unitSel, checks:remoteChecksAll, custom:remoteCustomAll, meals:remoteMealsAll, recipeEdits:recipeEdits});
  syncRef.set({rev:myRev, updatedAt:Date.now(), state:state}).catch(function(e){ console.warn('sync push failed',e); });
}
```

- [ ] **Step 2: Manual verification (read-only against the real sync this one time, per the project's live-data rule)**

This is the one step in this whole plan where sync must stay **on**, since
the thing being verified is the sync path itself. Keep it read-only:

1. Serve locally, open the app **without** running `syncRef=null;`.
2. Open the browser console and run:
   `JSON.parse((await firebase.database().ref('mealplan/shared').once('value')).val().state).recipeEdits`
3. Expected: this returns `undefined` or `{}` right now (no edits saved
   yet through this path) — confirms the read side works and that this
   check itself does not mutate anything.

Do **not** use this session to actually save a recipe edit — that would
push real data. Local-storage-and-back verification (Task 4, Step 4.6) is
the safe way to confirm persistence; this step only confirms the sync
payload shape reads back cleanly.

- [ ] **Step 3: Commit**

```bash
cd "C:/Users/Matt/GitHub/dinner-meal-plan"
git add index.html
git commit -m "Sync recipe edits through the shared Firebase state"
```

---

### Task 6: Serving-scale data model + wire into meal-plan mutators

**Files:**
- Modify: `index.html` (line 677 `ckMeals` constant; lines 683-699 meal-plan section; lines 804-830 mutators)

- [ ] **Step 1: Add the `ckScale` storage key**

Current code (line 677):

```js
const ckMeals  = p => "mp_meals_"+p;
```

Replace with:

```js
const ckMeals  = p => "mp_meals_"+p;
const ckScale  = p => "mp_scale_"+p;
```

- [ ] **Step 2: Add `mealScale` state after `saveMeals`**

Current code (line 699, end of the EDITABLE MEAL PLAN section):

```js
function saveMeals(slot){ save(ckMeals(CTX[slot].key), mealPlan[slot]); }
```

Replace with:

```js
function saveMeals(slot){ save(ckMeals(CTX[slot].key), mealPlan[slot]); }

const SCALE_OPTIONS = [1, 1.5, 2, 3];
function normalizeScale(arr){
  const a=(arr||[]).slice(0,14);
  while(a.length<14) a.push(1);
  return a.map(v => SCALE_OPTIONS.indexOf(v)>=0 ? v : 1);
}
let mealScale = {
  A: normalizeScale(load(ckScale(CTX.A.key), null)),
  B: normalizeScale(load(ckScale(CTX.B.key), null))
};
function saveScale(slot){ save(ckScale(CTX[slot].key), mealScale[slot]); }
function setScale(slot,i,val){
  if(SCALE_OPTIONS.indexOf(val)<0) return;
  mealScale[slot][i]=val;
  saveScale(slot);
  renderOverview(); renderRecipes(); renderGrocery(slot);
  toast(val===1 ? "Servings reset to 1x" : "Scaled to "+val+"x");
}
```

- [ ] **Step 3: Wire scale reset/carry into the existing mutators**

Current code (lines 804-830):

```js
function changeMeal(slot,i,val){
  mealPlan[slot][i] = (val===LEFTOVERS) ? LEFTOVERS : (val && R[val]) ? val : null;
  saveMeals(slot); renderOverview(); renderRecipes(); renderGrocery(slot);
  toast(val===LEFTOVERS ? "Set to leftovers" : val ? "Meal set" : "Meal cleared");
}
function moveMeal(slot,i,dir){
  const j=i+dir; if(j<0||j>13) return;
  const a=mealPlan[slot], t=a[i]; a[i]=a[j]; a[j]=t;
  saveMeals(slot); renderOverview(); renderRecipes(); toast("Moved");
}
function removeMeal(slot,i){
  if(!mealPlan[slot][i]) return;
  mealPlan[slot][i]=null;
  saveMeals(slot); renderOverview(); renderRecipes(); renderGrocery(slot); toast("Meal removed");
}
function sendOther(slot,i){
  const k=mealPlan[slot][i]; if(!k) return;
  const other=slot==='A'?'B':'A', dest=mealPlan[other].indexOf(null);
  if(dest<0){ toast("Cycle "+other+" has no empty day — remove one there first"); return; }
  mealPlan[other][dest]=k; mealPlan[slot][i]=null;
  saveMeals(slot); saveMeals(other);
  renderOverview(); renderRecipes(); renderGrocery('A'); renderGrocery('B');
  toast("Moved to Cycle "+other+", Day "+(dest+1));
}
function resetMealPlan(slot){
  mealPlan[slot]=defaultMenu(CTX[slot].key);
  saveMeals(slot); renderOverview(); renderRecipes(); renderGrocery(slot); toast("Cycle "+slot+" reset");
}
```

Replace with:

```js
function changeMeal(slot,i,val){
  mealPlan[slot][i] = (val===LEFTOVERS) ? LEFTOVERS : (val && R[val]) ? val : null;
  mealScale[slot][i] = 1; saveScale(slot);
  saveMeals(slot); renderOverview(); renderRecipes(); renderGrocery(slot);
  toast(val===LEFTOVERS ? "Set to leftovers" : val ? "Meal set" : "Meal cleared");
}
function moveMeal(slot,i,dir){
  const j=i+dir; if(j<0||j>13) return;
  const a=mealPlan[slot], t=a[i]; a[i]=a[j]; a[j]=t;
  const sc=mealScale[slot], ts=sc[i]; sc[i]=sc[j]; sc[j]=ts;
  saveMeals(slot); saveScale(slot); renderOverview(); renderRecipes(); toast("Moved");
}
function removeMeal(slot,i){
  if(!mealPlan[slot][i]) return;
  mealPlan[slot][i]=null;
  mealScale[slot][i]=1; saveScale(slot);
  saveMeals(slot); renderOverview(); renderRecipes(); renderGrocery(slot); toast("Meal removed");
}
function sendOther(slot,i){
  const k=mealPlan[slot][i]; if(!k) return;
  const other=slot==='A'?'B':'A', dest=mealPlan[other].indexOf(null);
  if(dest<0){ toast("Cycle "+other+" has no empty day — remove one there first"); return; }
  mealPlan[other][dest]=k; mealPlan[slot][i]=null;
  mealScale[other][dest]=mealScale[slot][i]; mealScale[slot][i]=1;
  saveMeals(slot); saveMeals(other); saveScale(slot); saveScale(other);
  renderOverview(); renderRecipes(); renderGrocery('A'); renderGrocery('B');
  toast("Moved to Cycle "+other+", Day "+(dest+1));
}
function resetMealPlan(slot){
  mealPlan[slot]=defaultMenu(CTX[slot].key);
  mealScale[slot]=new Array(14).fill(1);
  saveMeals(slot); saveScale(slot); renderOverview(); renderRecipes(); renderGrocery(slot); toast("Cycle "+slot+" reset");
}
```

- [ ] **Step 4: Manual verification**

Serve locally, console: `syncRef=null;`. There's no UI to set a scale value
yet (that's Task 7), so verify the wiring directly from the console:

1. Run `mealScale.A[0] = 2; console.log(mealScale.A[0]);` → expect `2`.
2. Click **Edit plan**, use ▲/▼ to move Day 1's meal to Day 2. Run
   `console.log(mealScale.A[0], mealScale.A[1]);` → expect the `2` to have
   moved to index 1 (i.e. `1, 2`, not `2, 1`).
3. Click ✕ to remove that meal. Run `console.log(mealScale.A[1]);` → expect
   `1` (reset).
4. Set `mealScale.A[2] = 3;` in the console, then use the meal-picker
   dropdown to change Day 3's meal to something else. Run
   `console.log(mealScale.A[2]);` → expect `1` (reset on repick).
5. Click **↺ Reset Cycle A**. Run
   `console.log(mealScale.A);` → expect `[1,1,1,1,1,1,1,1,1,1,1,1,1,1]`.

- [ ] **Step 5: Commit**

```bash
cd "C:/Users/Matt/GitHub/dinner-meal-plan"
git add index.html
git commit -m "Add per-night serving-scale data model wired into meal-plan mutators"
```

---

### Task 7: Serving-scale control in Edit Plan mode

**Files:**
- Modify: `index.html` (CSS ~line 103; function `dayCard`, currently lines 772-791)

- [ ] **Step 1: Add CSS for the scale button row**

Insert after the existing `.ov-controls button:disabled{...}` rule (~line
103), before `.recipe{...}`:

```css
.scale-row{display:flex;align-items:center;gap:6px;margin-top:8px;flex-wrap:wrap}
.scale-row .slab{font-size:.72rem;color:var(--muted);font-weight:600}
.scale-btns{display:flex;gap:4px}
.scale-btns button{border:1px solid var(--line);background:#fff;color:var(--ink);
  border-radius:6px;padding:4px 9px;font-size:.75rem;font-weight:700;cursor:pointer;font-family:inherit;transition:.15s}
.scale-btns button:hover{border-color:var(--accentA)}
.scale-btns button.sel{background:var(--accentA);color:#fff;border-color:var(--accentA)}
.cycleB .scale-btns button.sel{background:var(--accentB);border-color:var(--accentB)}
```

- [ ] **Step 2: Add the scale row to `dayCard`'s edit-mode branch**

Current code (lines 772-791):

```js
function dayCard(slot,i){
  const k=mealPlan[slot][i], d=addDays(CTX[slot].start,i), other=slot==='A'?'B':'A';
  if(editMode){
    const isSet = !!k;
    return `<div class="ov editing">
      <div class="d">Day ${i+1} · ${fmtFull(d)}</div>
      <button type="button" class="meal-sel" onclick="openMealPicker('${slot}',${i},this)">
        <span class="meal-sel-label ${isSet?'':'ph'}">${mealLabel(k)}</span><span class="meal-sel-caret">▾</span>
      </button>
      <div class="ov-controls">
        <button title="Move to earlier day" onclick="moveMeal('${slot}',${i},-1)" ${i===0?'disabled':''}>▲</button>
        <button title="Move to later day" onclick="moveMeal('${slot}',${i},1)" ${i===13?'disabled':''}>▼</button>
        <button title="Move to Cycle ${other}" onclick="sendOther('${slot}',${i})" ${isSet?'':'disabled'}>⇄ ${other}</button>
        <button title="Remove meal" class="rm" onclick="removeMeal('${slot}',${i})" ${isSet?'':'disabled'}>✕</button>
      </div></div>`;
  }
  if(k===LEFTOVERS) return `<div class="ov empty"><div class="d">Day ${i+1} · ${fmtFull(d)}</div><div class="m">🍲 Leftovers</div></div>`;
  if(!k) return `<div class="ov empty"><div class="d">Day ${i+1} · ${fmtFull(d)}</div><div class="m muted">— no meal —</div></div>`;
  return `<div class="ov" role="button" tabindex="0" onclick="openRecipe('${k}')" onkeypress="if(event.key==='Enter')openRecipe('${k}')"><div class="d">Day ${i+1} · ${fmtFull(d)}</div><div class="m">${R[k].title}</div><div class="open-hint">View recipe →</div></div>`;
}
```

Replace the `editMode` branch with:

```js
function dayCard(slot,i){
  const k=mealPlan[slot][i], d=addDays(CTX[slot].start,i), other=slot==='A'?'B':'A';
  if(editMode){
    const isSet = !!k;
    const canScale = isSet && k!==LEFTOVERS;
    const scale = mealScale[slot][i];
    const scaleRow = canScale ? `<div class="scale-row"><span class="slab">Servings</span><div class="scale-btns">${
      SCALE_OPTIONS.map(v=>`<button class="${scale===v?'sel':''}" onclick="setScale('${slot}',${i},${v})">${v}x</button>`).join("")
    }</div></div>` : "";
    return `<div class="ov editing">
      <div class="d">Day ${i+1} · ${fmtFull(d)}</div>
      <button type="button" class="meal-sel" onclick="openMealPicker('${slot}',${i},this)">
        <span class="meal-sel-label ${isSet?'':'ph'}">${mealLabel(k)}</span><span class="meal-sel-caret">▾</span>
      </button>
      <div class="ov-controls">
        <button title="Move to earlier day" onclick="moveMeal('${slot}',${i},-1)" ${i===0?'disabled':''}>▲</button>
        <button title="Move to later day" onclick="moveMeal('${slot}',${i},1)" ${i===13?'disabled':''}>▼</button>
        <button title="Move to Cycle ${other}" onclick="sendOther('${slot}',${i})" ${isSet?'':'disabled'}>⇄ ${other}</button>
        <button title="Remove meal" class="rm" onclick="removeMeal('${slot}',${i})" ${isSet?'':'disabled'}>✕</button>
      </div>${scaleRow}</div>`;
  }
  if(k===LEFTOVERS) return `<div class="ov empty"><div class="d">Day ${i+1} · ${fmtFull(d)}</div><div class="m">🍲 Leftovers</div></div>`;
  if(!k) return `<div class="ov empty"><div class="d">Day ${i+1} · ${fmtFull(d)}</div><div class="m muted">— no meal —</div></div>`;
  return `<div class="ov" role="button" tabindex="0" onclick="openRecipe('${k}')" onkeypress="if(event.key==='Enter')openRecipe('${k}')"><div class="d">Day ${i+1} · ${fmtFull(d)}</div><div class="m">${R[k].title}</div><div class="open-hint">View recipe →</div></div>`;
}
```

(The read-mode `onclick="openRecipe('${k}')"` lines are updated in Task 9 —
leave them as-is here.)

- [ ] **Step 3: Manual verification**

Serve locally, console: `syncRef=null;`. Click **Edit plan**:

1. Any day with a real meal planned shows a **Servings** row with **1x /
   1.5x / 2x / 3x** buttons, `1x` highlighted by default.
2. Click **2x**. Expected: toast "Scaled to 2x", the `2x` button becomes
   highlighted instead of `1x`.
3. A day showing "— no meal —" or "🍲 Leftovers" shows **no** Servings row.
4. Move that 2x day to another slot with ▲/▼ — expected: the highlighted
   button (2x) follows the meal to its new day card.
5. Click ✕ to remove it — expected: gone (no card to check, meal cleared).

- [ ] **Step 4: Commit**

```bash
cd "C:/Users/Matt/GitHub/dinner-meal-plan"
git add index.html
git commit -m "Add 1x/1.5x/2x/3x serving control to Edit Plan day cards"
```

---

### Task 8: Grocery calculation uses per-night scale

**Files:**
- Modify: `index.html` (functions `qtyOf` and `effectiveGrocery`, currently lines 913-945)

- [ ] **Step 1: Rewrite `qtyOf` and `effectiveGrocery` around occurrences instead of a presence Set**

Current code (lines 913-945):

```js
function qtyOf(item,present){
  if(item.always) return 1;
  let q=0; const per=item.per||{};
  for(const k in per){ if(present.has(k)) q+=per[k]; }
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
// Build the cycle's grocery list straight from its planned meals.
function effectiveGrocery(slot){
  const present=new Set((mealPlan[slot]||[]).filter(Boolean));
  const sections=[], idx={};
  function sec(name){ if(idx[name]==null){ idx[name]=sections.length; sections.push({sec:name,items:[]}); } return sections[idx[name]]; }
  GITEMS.forEach(item=>{
    const qty=qtyOf(item,present);
    if(qty<=0) return;
    const used=Object.keys(item.per||{}).filter(k=>present.has(k)).map(k=>R[k]&&R[k].title).filter(Boolean);
    sec(item.sec).items.push({item, qty, used});
  });
  return sections;
}
```

Replace `qtyOf` and `effectiveGrocery` (leave `unitLabel`/`fmtQty`/`qtyLabel`
untouched, in between them) with:

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
function effectiveGrocery(slot){
  const occurrences=(mealPlan[slot]||[])
    .map((k,i)=> (k && k!==LEFTOVERS) ? {key:k, scale:(mealScale[slot][i]||1)} : null)
    .filter(Boolean);
  const sections=[], idx={};
  function sec(name){ if(idx[name]==null){ idx[name]=sections.length; sections.push({sec:name,items:[]}); } return sections[idx[name]]; }
  GITEMS.forEach(item=>{
    const qty=qtyOf(item,occurrences);
    if(qty<=0) return;
    const used=occurrences.filter(o=>item.per && item.per[o.key]!=null).map(o=>R[o.key]&&R[o.key].title).filter(Boolean);
    sec(item.sec).items.push({item, qty, used});
  });
  return sections;
}
```

- [ ] **Step 2: Manual verification**

Serve locally, console: `syncRef=null;`. Go to **Cycle A Grocery**, note the
current quantity shown for an ingredient used by one specific recipe (e.g.
"Chicken breast" if a chicken recipe is planned that cycle). Then:

1. Click **Edit plan**, set that recipe's night to **2x**.
2. Return to **Cycle A Grocery**. Expected: the quantity for ingredients
   tied to that recipe roughly doubles (rounded up per the existing
   packaging rules — e.g. `1 lb` per-recipe amount at 2x becomes `2 lb`,
   not `1 lb` still).
3. Set it back to **1x**. Expected: quantity returns to the original value.
4. Set it to **3x**. Expected: quantity scales to 3x the per-recipe amount
   (again subject to rounding/packaging).
5. Confirm an `item.always` pantry item (e.g. an "as needed" spice, if any
   exist in the catalog) is unaffected by the scale change — its line
   still just says "as needed".

- [ ] **Step 3: Commit**

```bash
cd "C:/Users/Matt/GitHub/dinner-meal-plan"
git add index.html
git commit -m "Scale grocery quantities by each night's serving multiplier"
```

---

### Task 9: Show the scale note on the recipe page

**Files:**
- Modify: `index.html` (functions `recipeCard`, `recipesHTML`, `dayCard` read-mode branch, `openRecipe`, `showRecipeDetail`)

- [ ] **Step 1: Add a `scale` parameter and note to `recipeCard`**

Current code (from Task 3):

```js
function recipeCard(k,dayNo,date){
  const r=R[k], rd=recipeData(k);
  const link = r.url ? `<a href="${r.url}" target="_blank" rel="noopener">${r.src} ↗</a>` : r.src;
  const note = r.note ? `<div class="note-box">💡 ${r.note}</div>` : "";
  return `<article class="recipe">
    <span class="day-tag">Day ${dayNo} · ${fmt(date)}</span>
    <h3>${r.title}</h3>
    <div class="meta">Serves ${r.serves} · Source: ${link}</div>
    <div class="cols">
      <div><h4>Ingredients</h4>${ingHTML(rd.ing)}</div>
      <div><h4>Instructions</h4><ol class="steps">${rd.steps.map(s=>`<li>${s}</li>`).join("")}</ol>${note}</div>
    </div></article>`;
}
```

Replace with:

```js
function recipeCard(k,dayNo,date,scale){
  const r=R[k], rd=recipeData(k);
  const link = r.url ? `<a href="${r.url}" target="_blank" rel="noopener">${r.src} ↗</a>` : r.src;
  const note = r.note ? `<div class="note-box">💡 ${r.note}</div>` : "";
  const scaleNote = (scale && scale!==1) ? `<div class="note-box">👥 ${scale}x tonight — grocery list already adjusted.</div>` : "";
  return `<article class="recipe">
    <span class="day-tag">Day ${dayNo} · ${fmt(date)}</span>
    <h3>${r.title}</h3>
    <div class="meta">Serves ${r.serves} · Source: ${link}</div>
    ${scaleNote}
    <div class="cols">
      <div><h4>Ingredients</h4>${ingHTML(rd.ing)}</div>
      <div><h4>Instructions</h4><ol class="steps">${rd.steps.map(s=>`<li>${s}</li>`).join("")}</ol>${note}</div>
    </div></article>`;
}
```

- [ ] **Step 2: Pass the scale value from `recipesHTML`**

Current code (lines 894-901):

```js
function recipesHTML(slot){
  return mealPlan[slot].map((k,i)=>{
    const date=addDays(CTX[slot].start,i);
    if(k===LEFTOVERS) return `<article class="recipe empty"><span class="day-tag">Day ${i+1} · ${fmt(date)}</span><h3 class="muted">🍲 Leftovers</h3></article>`;
    if(!k) return `<article class="recipe empty"><span class="day-tag">Day ${i+1} · ${fmt(date)}</span><h3 class="muted">No dinner planned</h3></article>`;
    return recipeCard(k,i+1,date);
  }).join("");
}
```

Replace with:

```js
function recipesHTML(slot){
  return mealPlan[slot].map((k,i)=>{
    const date=addDays(CTX[slot].start,i);
    if(k===LEFTOVERS) return `<article class="recipe empty"><span class="day-tag">Day ${i+1} · ${fmt(date)}</span><h3 class="muted">🍲 Leftovers</h3></article>`;
    if(!k) return `<article class="recipe empty"><span class="day-tag">Day ${i+1} · ${fmt(date)}</span><h3 class="muted">No dinner planned</h3></article>`;
    return recipeCard(k,i+1,date,mealScale[slot][i]);
  }).join("");
}
```

- [ ] **Step 3: Pass day context through the overview click-through**

Current code (in `dayCard`'s read-mode return, currently the last line of
the function after Task 7's edit):

```js
  return `<div class="ov" role="button" tabindex="0" onclick="openRecipe('${k}')" onkeypress="if(event.key==='Enter')openRecipe('${k}')"><div class="d">Day ${i+1} · ${fmtFull(d)}</div><div class="m">${R[k].title}</div><div class="open-hint">View recipe →</div></div>`;
```

Replace with:

```js
  return `<div class="ov" role="button" tabindex="0" onclick="openRecipe('${k}','${slot}',${i})" onkeypress="if(event.key==='Enter')openRecipe('${k}','${slot}',${i})"><div class="d">Day ${i+1} · ${fmtFull(d)}</div><div class="m">${R[k].title}</div><div class="open-hint">View recipe →</div></div>`;
```

- [ ] **Step 4: Update `openRecipe` to resolve and forward the scale**

Current code (lines 1088-1094):

```js
function openRecipe(k){
  document.querySelectorAll("#tabs button").forEach(x=>x.classList.remove("active"));
  var tb=document.querySelector('#tabs button[data-t="recipes"]'); if(tb) tb.classList.add("active");
  document.querySelectorAll("section.panel").forEach(p=>p.classList.remove("active"));
  document.getElementById("recipes").classList.add("active");
  showRecipeDetail(k);
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

- [ ] **Step 5: Add the `scale` parameter and note to `showRecipeDetail`**

Current code (from Task 4):

```js
function showRecipeDetail(k){
  const r=R[k];
  const link = r.url ? `<a href="${r.url}" target="_blank" rel="noopener">${r.src} ↗</a>` : r.src;
  const note = r.note ? `<div class="note-box">💡 ${r.note}</div>` : "";
  if(recipeEditKey===k){
```

Replace with:

```js
function showRecipeDetail(k,scale){
  scale = scale||1;
  const r=R[k];
  const link = r.url ? `<a href="${r.url}" target="_blank" rel="noopener">${r.src} ↗</a>` : r.src;
  const note = r.note ? `<div class="note-box">💡 ${r.note}</div>` : "";
  const scaleNote = (scale!==1) ? `<div class="note-box">👥 ${scale}x tonight — grocery list already adjusted.</div>` : "";
  if(recipeEditKey===k){
```

Then, still in `showRecipeDetail`, current code (the read-mode `else`
branch, from Task 4):

```js
  } else {
    const rd=recipeData(k);
    document.getElementById("recipeDetail").innerHTML =
      `<button class="btn-reset" style="margin-bottom:12px" onclick="hideRecipeDetail()">← All recipes</button>
       <article class="recipe">
         <h3>${r.title}</h3>
         <div class="meta">Serves ${r.serves} · Source: ${link}</div>
         <button class="btn-edit" style="margin:8px 0" onclick="editRecipe('${k}')">✏️ Edit ingredients & steps</button>
         <div class="cols"><div><h4>Ingredients</h4>${ingHTML(rd.ing)}</div>
         <div><h4>Instructions</h4><ol class="steps">${rd.steps.map(s=>`<li>${s}</li>`).join("")}</ol>${note}</div></div>
       </article>`;
  }
```

Replace with:

```js
  } else {
    const rd=recipeData(k);
    document.getElementById("recipeDetail").innerHTML =
      `<button class="btn-reset" style="margin-bottom:12px" onclick="hideRecipeDetail()">← All recipes</button>
       <article class="recipe">
         <h3>${r.title}</h3>
         <div class="meta">Serves ${r.serves} · Source: ${link}</div>
         ${scaleNote}
         <button class="btn-edit" style="margin:8px 0" onclick="editRecipe('${k}')">✏️ Edit ingredients & steps</button>
         <div class="cols"><div><h4>Ingredients</h4>${ingHTML(rd.ing)}</div>
         <div><h4>Instructions</h4><ol class="steps">${rd.steps.map(s=>`<li>${s}</li>`).join("")}</ol>${note}</div></div>
       </article>`;
  }
```

(Calls to `showRecipeDetail('${k}')` from `renderAllRecipes`'s recipe-index
buttons, and from `editRecipe`/`cancelRecipeEdit`/`saveRecipeEdit`/
`resetRecipeEdit`, all continue to work unchanged — `scale` defaults to `1`
when omitted, so no note shows when browsing generically.)

- [ ] **Step 6: Manual verification**

Serve locally, console: `syncRef=null;`.

1. Click **Edit plan**, set a day's meal to **2x**, click "✓ Done editing".
2. On the **Overview** tab, click that day's card. Expected: it opens the
   recipe detail (via the "All Recipes" tab) showing **"👥 2x tonight —
   grocery list already adjusted."** under "Serves N".
3. Go to that recipe's **Cycle A/B Recipes** tab entry directly. Expected:
   same note shown there too.
4. Go to the **Recipes** tab (All Recipes index) and open the *same*
   recipe from there instead (not via the day click). Expected: **no**
   scale note shown (this view has no day context).
5. Set that day back to 1x and re-check both views — expected: note is
   gone in both places.

- [ ] **Step 7: Commit**

```bash
cd "C:/Users/Matt/GitHub/dinner-meal-plan"
git add index.html
git commit -m "Show a serving-scale note on the recipe page for scaled nights"
```

---

### Task 10: Sync serving-scale through Firebase

**Files:**
- Modify: `index.html` (line 1118 remote-state globals; function `applyRemote`; function `syncPush`)

- [ ] **Step 1: Add the `remoteScaleAll` global**

Current code (line 1118):

```js
let remoteChecksAll={}, remoteCustomAll={}, remoteMealsAll={};
```

Replace with:

```js
let remoteChecksAll={}, remoteCustomAll={}, remoteMealsAll={}, remoteScaleAll={};
```

- [ ] **Step 2: Read `mealScale` in `applyRemote`**

Current code (from Task 5's version) — anchor on the `remoteMealsAll  =
s.meals||{};` line:

```js
    remoteChecksAll = s.checks||{};
    remoteCustomAll = s.custom||{};
    remoteMealsAll  = s.meals||{};
    recipeEdits = s.recipeEdits||{};
    checks.A = remoteChecksAll[CTX.A.key]||{};  checks.B = remoteChecksAll[CTX.B.key]||{};
    custom.A = remoteCustomAll[CTX.A.key]||[];  custom.B = remoteCustomAll[CTX.B.key]||[];
    mealPlan.A = normalizeMenu(remoteMealsAll[CTX.A.key] || defaultMenu(CTX.A.key));
    mealPlan.B = normalizeMenu(remoteMealsAll[CTX.B.key] || defaultMenu(CTX.B.key));
    save(LS.staples,staplePrices); save(LS.prices,prices); save(LS.store,storeSel); save(LS.unit,unitSel);
    save(ckChecks(CTX.A.key),checks.A); save(ckChecks(CTX.B.key),checks.B);
    save(ckCustom(CTX.A.key),custom.A); save(ckCustom(CTX.B.key),custom.B);
    save(ckMeals(CTX.A.key),mealPlan.A); save(ckMeals(CTX.B.key),mealPlan.B);
    saveRecipeEdits();
```

Replace with:

```js
    remoteChecksAll = s.checks||{};
    remoteCustomAll = s.custom||{};
    remoteMealsAll  = s.meals||{};
    remoteScaleAll  = s.mealScale||{};
    recipeEdits = s.recipeEdits||{};
    checks.A = remoteChecksAll[CTX.A.key]||{};  checks.B = remoteChecksAll[CTX.B.key]||{};
    custom.A = remoteCustomAll[CTX.A.key]||[];  custom.B = remoteCustomAll[CTX.B.key]||[];
    mealPlan.A = normalizeMenu(remoteMealsAll[CTX.A.key] || defaultMenu(CTX.A.key));
    mealPlan.B = normalizeMenu(remoteMealsAll[CTX.B.key] || defaultMenu(CTX.B.key));
    mealScale.A = normalizeScale(remoteScaleAll[CTX.A.key]);
    mealScale.B = normalizeScale(remoteScaleAll[CTX.B.key]);
    save(LS.staples,staplePrices); save(LS.prices,prices); save(LS.store,storeSel); save(LS.unit,unitSel);
    save(ckChecks(CTX.A.key),checks.A); save(ckChecks(CTX.B.key),checks.B);
    save(ckCustom(CTX.A.key),custom.A); save(ckCustom(CTX.B.key),custom.B);
    save(ckMeals(CTX.A.key),mealPlan.A); save(ckMeals(CTX.B.key),mealPlan.B);
    save(ckScale(CTX.A.key),mealScale.A); save(ckScale(CTX.B.key),mealScale.B);
    saveRecipeEdits();
```

- [ ] **Step 3: Write `mealScale` in `syncPush`**

Current code (from Task 5's version):

```js
function syncPush(){
  if(!syncRef) return;
  remoteChecksAll[CTX.A.key]=checks.A; remoteChecksAll[CTX.B.key]=checks.B;
  remoteCustomAll[CTX.A.key]=custom.A; remoteCustomAll[CTX.B.key]=custom.B;
  remoteMealsAll[CTX.A.key]=mealPlan.A; remoteMealsAll[CTX.B.key]=mealPlan.B;
  myRev = DEVICE+"-"+(++seq);
  const state = JSON.stringify({staples:staplePrices, prices:prices, store:storeSel, unit:unitSel, checks:remoteChecksAll, custom:remoteCustomAll, meals:remoteMealsAll, recipeEdits:recipeEdits});
  syncRef.set({rev:myRev, updatedAt:Date.now(), state:state}).catch(function(e){ console.warn('sync push failed',e); });
}
```

Replace with:

```js
function syncPush(){
  if(!syncRef) return;
  remoteChecksAll[CTX.A.key]=checks.A; remoteChecksAll[CTX.B.key]=checks.B;
  remoteCustomAll[CTX.A.key]=custom.A; remoteCustomAll[CTX.B.key]=custom.B;
  remoteMealsAll[CTX.A.key]=mealPlan.A; remoteMealsAll[CTX.B.key]=mealPlan.B;
  remoteScaleAll[CTX.A.key]=mealScale.A; remoteScaleAll[CTX.B.key]=mealScale.B;
  myRev = DEVICE+"-"+(++seq);
  const state = JSON.stringify({staples:staplePrices, prices:prices, store:storeSel, unit:unitSel, checks:remoteChecksAll, custom:remoteCustomAll, meals:remoteMealsAll, mealScale:remoteScaleAll, recipeEdits:recipeEdits});
  syncRef.set({rev:myRev, updatedAt:Date.now(), state:state}).catch(function(e){ console.warn('sync push failed',e); });
}
```

- [ ] **Step 4: Manual verification (read-only against the real sync, same rule as Task 5)**

Serve locally **without** stubbing sync this one time:

1. Console: `JSON.parse((await firebase.database().ref('mealplan/shared').once('value')).val().state).mealScale`
2. Expected: returns `undefined` (field doesn't exist yet in the live data)
   or an object shaped like `{ "<period>": [1,1,...] }` if a previous local
   test happened to already push it — either way, confirms the read path
   and JSON shape are sound without you performing any write here.

Do not set a scale value in this un-stubbed session — that would push real
data to the shared plan.

- [ ] **Step 5: Commit**

```bash
cd "C:/Users/Matt/GitHub/dinner-meal-plan"
git add index.html
git commit -m "Sync per-night serving scale through the shared Firebase state"
```

---

### Task 11: Full integrated manual verification pass

**Files:** none (verification only)

- [ ] **Step 1: Serve and stub sync**

```bash
cd "C:/Users/Matt/GitHub/dinner-meal-plan" && python -m http.server 8765
```

Open `http://localhost:8765/`, console: `syncRef=null;`.

- [ ] **Step 2: Walk through all four features together**

1. **Scroll bug:** open the meal picker on any day, scroll its list — stays
   open.
2. **Purchased pile:** on Cycle A Grocery, check a few items across
   different categories and a custom item — all land in one Purchased
   section at the bottom; category totals reflect only what's left.
3. **Recipe editing:** open a recipe, edit an ingredient and a step, save;
   confirm it shows on both the Cycle A/B tab and the All Recipes detail
   view; reset it back to original.
4. **Serving scale, end to end:** in Edit Plan mode, set a night to 2x;
   confirm the grocery list for that cycle reflects the doubled quantity;
   open that night's recipe from the Overview and confirm the "👥 2x
   tonight" note appears; open the same recipe from the All Recipes tab
   and confirm no note appears there; move the 2x night to a different day
   and confirm the scale follows it; remove the meal and confirm the scale
   resets.
5. Reload the page (re-run `syncRef=null;` after reload) and spot-check
   that the recipe edit and the 2x setting both persisted through
   `localStorage`.

- [ ] **Step 3: Confirm nothing is left uncommitted**

```bash
cd "C:/Users/Matt/GitHub/dinner-meal-plan"
git status
```

Expected: `nothing to commit, working tree clean` (every prior task already
committed its own change).

---

## Spec coverage check

- Meal-picker scroll bug → Task 1.
- Grocery checked items → bottom-of-page Purchased pile → Task 2.
- Editable recipes (ingredients + steps, inline on detail page, synced) →
  Tasks 3, 4, 5.
- Per-night serving multiplier (data model, edit-mode UI, grocery-math
  integration, recipe-page note, sync) → Tasks 6, 7, 8, 9, 10.
- End-to-end confirmation → Task 11.
