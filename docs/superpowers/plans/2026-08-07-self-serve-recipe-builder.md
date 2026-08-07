# Self-Serve Recipe Builder Implementation Plan (Part B of 2)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **This is Part B of a two-plan sequence — Part A (`2026-08-07-structured-ingredient-migration.md`) must be fully complete and verified first.** This plan's "current code" references assume Part A's changes have already landed (in particular: `GITEMS` has no `per` fields, every recipe's `ing` is `[{heading,rows:[{text,item,qty}]}]`, `effectiveGrocery`/`qtyOf` are Part A's rewritten versions, `showRecipeDetail` is Part A's single-read-only-branch version, and the old free-text editor functions no longer exist). Do not start this plan against pre-Part-A code — none of the "current code" blocks below will match.

**Goal:** Let the user create brand-new recipes, edit any recipe (built-in or custom) through a structured ingredient builder instead of free text, create new grocery-catalog items inline when needed, and delete recipes they created — all fully integrated with the existing meal picker, Cycle A/B tabs, grocery math, and Prices tab (not a separate "custom" experience).

**Architecture:** A single shared ingredient-builder component (row-based: text + optional catalog-item link + quantity) is used both for editing an existing recipe and for creating a new one. New recipes live in a `customRecipes` store; new catalog items live in `customItems` (added in Part A, unused until now). A `baseRecipe(k)` helper unifies "is this key built-in or custom" everywhere the code currently assumes `R[k]`.

**Tech Stack:** Vanilla JS, no build step — same as the rest of `index.html`.

**Spec:** `docs/superpowers/specs/2026-08-07-self-serve-recipes-and-structured-ingredients-design.md`

---

## Note on testing in this codebase

No automated test suite exists. Verification is manual, against a locally-served copy, with `syncRef=null` set immediately after load before any interaction — this app syncs to a live, shared Firebase Realtime Database.

## Note on line numbers

Line numbers below reflect `index.html` as it will exist immediately after Part A lands (not the current `main` branch state, which still has the old free-text editor and the old `GITEMS.per` model). Locate each edit by its exact code content, not by line number.

---

### Task 7: `customRecipes` data layer — unify built-in and custom recipes everywhere

**Files:**
- Modify: `index.html` (RECIPES section: `recipeData`, `normalizeMenu`, `mealLabel`, `dayCard`, `mealPickerOptions`, `renderAllRecipes`, `recipeCard`, `showRecipeDetail`, and Part A's `effectiveGrocery`)

**Context:** Right now, every one of these functions assumes a recipe key always exists in `R`. This task adds `customRecipes` and a `baseRecipe(k)` helper, then updates every one of those call sites to go through it instead of `R[k]` directly, so a custom recipe behaves identically to a built-in one everywhere. No UI is added yet — this is purely the data-layer foundation Tasks 8–10 build on.

- [ ] **Step 1: Add the `customRecipes` store and `baseRecipe`/`allRecipeKeys` helpers**

Insert directly after `saveRecipeEdits` (which now sits right next to `recipeData`, since Part A removed everything that used to be between them):

```js
let customRecipes = load('mp_customRecipes_v1', {});   // { [key]: {title, serves, ing, steps} } — user-created recipes
function saveCustomRecipes(){ save('mp_customRecipes_v1', customRecipes); }
function baseRecipe(k){ return R[k] || customRecipes[k]; }
function allRecipeKeys(){
  return Object.keys(R).concat(Object.keys(customRecipes)).sort((a,b)=>baseRecipe(a).title.localeCompare(baseRecipe(b).title));
}
```

- [ ] **Step 2: `recipeData` checks `customRecipes` first**

Current code:

```js
function recipeData(k){
  const base=R[k], edit=recipeEdits[k];
  if(!edit) return base;
  return Object.assign({}, base, { ing: edit.ing || base.ing, steps: edit.steps || base.steps });
}
```

Replace with:

```js
function recipeData(k){
  if(customRecipes[k]) return customRecipes[k];
  const base=R[k], edit=recipeEdits[k];
  if(!edit) return base;
  return Object.assign({}, base, { ing: edit.ing || base.ing, steps: edit.steps || base.steps });
}
```

(A custom recipe has no "original" to layer `recipeEdits` on top of — editing one overwrites `customRecipes[k]` directly, per Task 9. `recipeEdits` continues to mean "override for a built-in recipe" exactly as before.)

- [ ] **Step 3: `normalizeMenu` accepts custom recipe keys**

Current code:

```js
function normalizeMenu(arr){
  const a=(arr||[]).slice(0,14);
  while(a.length<14) a.push(null);
  return a.map(k => (k===LEFTOVERS || (k && R[k])) ? k : null);
}
```

Replace with:

```js
function normalizeMenu(arr){
  const a=(arr||[]).slice(0,14);
  while(a.length<14) a.push(null);
  return a.map(k => (k===LEFTOVERS || (k && baseRecipe(k))) ? k : null);
}
```

- [ ] **Step 4: `mealLabel` resolves through `baseRecipe`**

Current code:

```js
function mealLabel(k){
  if(k===LEFTOVERS) return "Leftovers";
  return (k && R[k]) ? R[k].title : "— no meal —";
}
```

Replace with:

```js
function mealLabel(k){
  if(k===LEFTOVERS) return "Leftovers";
  return (k && baseRecipe(k)) ? escHTML(baseRecipe(k).title) : "— no meal —";
}
```

(A custom recipe's title is now user-authored text flowing into `innerHTML` via the day card and meal-sel button — `escHTML` it here, at the point it's resolved, so every caller gets a safe value automatically.)

- [ ] **Step 5: `dayCard`'s read-mode title**

Current code (the last line of `dayCard`, its read-mode return):

```js
  return `<div class="ov" role="button" tabindex="0" onclick="openRecipe('${k}','${slot}',${i})" onkeypress="if(event.key==='Enter')openRecipe('${k}','${slot}',${i})"><div class="d">Day ${i+1} · ${fmtFull(d)}</div><div class="m">${R[k].title}</div><div class="open-hint">View recipe →</div></div>`;
```

Replace with:

```js
  return `<div class="ov" role="button" tabindex="0" onclick="openRecipe('${k}','${slot}',${i})" onkeypress="if(event.key==='Enter')openRecipe('${k}','${slot}',${i})"><div class="d">Day ${i+1} · ${fmtFull(d)}</div><div class="m">${escHTML(baseRecipe(k).title)}</div><div class="open-hint">View recipe →</div></div>`;
```

- [ ] **Step 6: `mealPickerOptions` includes custom recipes**

Current code:

```js
function mealPickerOptions(){
  return [{v:"",label:"— No Meal —"},{v:LEFTOVERS,label:"Leftovers"}]
    .concat(ALL_RECIPE_KEYS.map(k=>({v:k,label:R[k].title})));
}
```

Replace with:

```js
function mealPickerOptions(){
  return [{v:"",label:"— No Meal —"},{v:LEFTOVERS,label:"Leftovers"}]
    .concat(allRecipeKeys().map(k=>({v:k,label:baseRecipe(k).title})));
}
```

- [ ] **Step 7: `renderAllRecipes` includes custom recipes**

Current code:

```js
const ALL_RECIPE_KEYS = Object.keys(R).slice().sort((a,b)=>R[a].title.localeCompare(R[b].title));
function renderAllRecipes(){
  document.getElementById("recipeIndex").innerHTML =
    `<div class="recipe-index">`+ALL_RECIPE_KEYS.map(k=>
      `<button class="ri-card" onclick="showRecipeDetail('${k}')"><span class="ri-title">${R[k].title}</span><span class="ri-meta">Serves ${R[k].serves} · ${R[k].src}</span></button>`
    ).join("")+`</div>`;
}
```

Replace with (drop the `ALL_RECIPE_KEYS` constant entirely — `allRecipeKeys()` from Step 1 replaces it, computed fresh each call so a newly-created or deleted custom recipe shows up immediately):

```js
function renderAllRecipes(){
  document.getElementById("recipeIndex").innerHTML =
    `<div class="recipe-index">`+allRecipeKeys().map(k=>{
      const r=baseRecipe(k);
      return `<button class="ri-card" onclick="showRecipeDetail('${k}')"><span class="ri-title">${r.title}</span><span class="ri-meta">Serves ${r.serves}${r.src?" · "+r.src:""}</span></button>`;
    }).join("")+`</div>`;
}
```

(`r.src` is now conditionally shown — custom recipes have no `src` field, so the " · Source" separator only appears when there is one, rather than rendering "Serves 4 · " with nothing after it.)

- [ ] **Step 8: `recipeCard` resolves fixed fields through `baseRecipe`**

Current code:

```js
function recipeCard(k,dayNo,date,scale){
  const r=R[k], rd=recipeData(k);
  const link = r.url ? `<a href="${r.url}" target="_blank" rel="noopener">${r.src} ↗</a>` : r.src;
```

Replace the first line with:

```js
function recipeCard(k,dayNo,date,scale){
  const r=baseRecipe(k), rd=recipeData(k);
  const link = r.url ? `<a href="${r.url}" target="_blank" rel="noopener">${r.src} ↗</a>` : (r.src||"your own recipe");
```

(A custom recipe has neither `url` nor `src` — the fallback avoids rendering "Source: " with nothing after it. Everything else in `recipeCard` is unchanged.)

- [ ] **Step 9: `showRecipeDetail` resolves through `baseRecipe`**

Current code (Part A's version):

```js
function showRecipeDetail(k,scale){
  scale = scale||1;
  const r=R[k];
  const link = r.url ? `<a href="${r.url}" target="_blank" rel="noopener">${r.src} ↗</a>` : r.src;
```

Replace the relevant lines with:

```js
function showRecipeDetail(k,scale){
  scale = scale||1;
  const r=baseRecipe(k);
  const link = r.url ? `<a href="${r.url}" target="_blank" rel="noopener">${r.src} ↗</a>` : (r.src||"your own recipe");
```

- [ ] **Step 10: `effectiveGrocery`'s "used by" labels resolve through `baseRecipe`**

Current code (Part A's version, inside `effectiveGrocery`):

```js
    const used = item.always ? [] : [...(usedBy[item.n]||[])].map(k=>R[k]&&R[k].title).filter(Boolean);
```

Replace with:

```js
    const used = item.always ? [] : [...(usedBy[item.n]||[])].map(k=>baseRecipe(k)&&baseRecipe(k).title).filter(Boolean);
```

- [ ] **Step 11: Escape the meal-picker list's rendering of recipe labels**

`mealPickerOptions()` (Step 6) now produces labels that can be a custom recipe's user-authored title, not just built-in titles. Find where those labels render:

```js
function renderMealPickerList(filter){
  const ul=document.getElementById("mealPickerList"); if(!ul) return;
  const cur = mealPickerCtx ? (mealPlan[mealPickerCtx.slot][mealPickerCtx.i]||"") : "";
  const f=(filter||"").trim().toLowerCase();
  const opts=mealPickerOptions().filter(o=>!f || o.label.toLowerCase().includes(f));
  if(!opts.length){ ul.innerHTML=`<li class="mp-empty">No matches</li>`; return; }
  ul.innerHTML=opts.map(o=>`<li class="mp-opt ${o.v===cur?'sel':''}" onmousedown="pickMeal('${o.v}')">${o.label}</li>`).join("");
}
```

Replace the last line with:

```js
  ul.innerHTML=opts.map(o=>`<li class="mp-opt ${o.v===cur?'sel':''}" onmousedown="pickMeal('${esc(o.v)}')">${escHTML(o.label)}</li>`).join("");
```

(`esc()` — already defined elsewhere in the file for escaping a value going into a single-quoted `onclick`/`onmousedown` JS-string context — protects the recipe key inside the attribute; `escHTML()` protects the visible label text. Custom recipe keys are generated by `genRecipeKey` in Task 10 from a sanitized slug, so this is defense-in-depth rather than a known-exploitable gap, but it costs nothing and matches how every other user-authored string in this file is treated.)

- [ ] **Step 12: Escape recipe titles/serves everywhere else they render**

Three more places interpolate a recipe's `title`/`serves` (now potentially user-authored) without escaping. Fix each:

In `recipeCard` (find `const r=baseRecipe(k), rd=recipeData(k);` from Step 8), the return statement currently has:

```js
    <h3>${r.title}</h3>
    <div class="meta">Serves ${r.serves} · Source: ${link}</div>
```

Replace with:

```js
    <h3>${escHTML(r.title)}</h3>
    <div class="meta">Serves ${escHTML(r.serves)} · Source: ${link}</div>
```

In `showRecipeDetail` (find `const r=baseRecipe(k);` from Step 9), the same two lines appear inside its template — apply the identical `escHTML(r.title)` / `escHTML(r.serves)` change there too.

In `renderAllRecipes` (Step 7's replacement), currently:

```js
      return `<button class="ri-card" onclick="showRecipeDetail('${k}')"><span class="ri-title">${r.title}</span><span class="ri-meta">Serves ${r.serves}${r.src?" · "+r.src:""}</span></button>`;
```

Replace with:

```js
      return `<button class="ri-card" onclick="showRecipeDetail('${k}')"><span class="ri-title">${escHTML(r.title)}</span><span class="ri-meta">Serves ${escHTML(r.serves)}${r.src?" · "+escHTML(r.src):""}</span></button>`;
```

(`r.src`/`r.url` stay unescaped only where they're developer-authored built-in data with no editing path — but since `renderAllRecipes` is generic over both built-in and custom keys and `r.src` could theoretically be attacker-adjacent if a future task ever lets it be user-set, escaping it too here costs nothing and removes the need to remember this distinction later.)

Finally, in Part A's rewritten `effectiveGrocery` (find `const used = item.always ? [] : [...(usedBy[item.n]||[])].map(k=>baseRecipe(k)&&baseRecipe(k).title).filter(Boolean);` from Step 10), these titles flow into the grocery list's "used by" text (via `qtyLabel`/`groceryRowHTML`, unescaped there). Replace with:

```js
    const used = item.always ? [] : [...(usedBy[item.n]||[])].map(k=>baseRecipe(k)&&escHTML(baseRecipe(k).title)).filter(Boolean);
```

- [ ] **Step 13: Manual verification**

Serve locally, console: `syncRef=null;`.

1. Run `customRecipes` in the console → expect `{}` (empty, no UI creates one yet).
2. Run `allRecipeKeys().length === 21` → expect `true`.
3. Open the **Recipes** tab, Cycle A/B tabs, and the meal picker — everything should render exactly as it did before this task (this is a pure refactor of how existing built-in recipes are looked up, no behavior change yet since `customRecipes` is always empty).
4. As a direct test of the new plumbing (since no UI exists yet to create a custom recipe through), run in the console:
   ```js
   customRecipes['test-recipe'] = {title:"Test Recipe", serves:"2", ing:[{heading:"",rows:[{text:"1 test item",item:null,qty:0}]}], steps:["Do the thing."]};
   renderAllRecipes();
   ```
   Confirm a "Test Recipe" card now appears in the Recipes tab, opens correctly showing "Serves 2" and no "Source:" line (or "your own recipe" per Step 8/9's fallback — check this reads sensibly), and appears in the meal picker's dropdown list. Then clean up: `delete customRecipes['test-recipe']; renderAllRecipes();`.
5. Repeat with an XSS-shaped title to confirm Steps 11/12's escaping actually works end to end:
   ```js
   window.__x=false;
   customRecipes['xss-test'] = {title:'<img src=x onerror="window.__x=true">', serves:"2", ing:[{heading:"",rows:[{text:"1 test item",item:null,qty:0}]}], steps:["Step one."]};
   renderAllRecipes();
   ```
   Confirm `window.__x` stays `false`, the Recipes tab shows the literal escaped text (not a broken image icon), open it via `showRecipeDetail('xss-test')` and confirm the detail page also shows literal text, and check the meal picker's dropdown list (open it on any day) shows the literal text too, not a broken tag. Clean up: `delete customRecipes['xss-test']; renderAllRecipes();`.

- [ ] **Step 14: Commit**

```bash
cd "C:/Users/Matt/GitHub/dinner-meal-plan"
git add index.html
git commit -m "Add customRecipes data layer; unify built-in/custom recipe lookups"
```

---

### Task 8: Build the ingredient builder component (rows, groups, inline new-item creation)

**Files:**
- Modify: `index.html` (CSS in `<style>`, after the existing `.recipe-edit-actions{...}` rule; new functions in the RECIPES section, after `escHTML`)

**Context:** This task builds the shared UI component Tasks 9 and 10 wire into "edit an existing recipe" and "create a new recipe" respectively. It's self-contained and directly testable via the console even before either of those land — nothing in this task depends on Tasks 9/10, and nothing renders on-screen automatically until something sets `recipeDraft` and calls `renderIngredientBuilder()` into a container element (Step 3's verification does this manually).

Every text/quantity/heading input uses `onchange` (fires on blur), not `oninput`, and its handler does **not** call `renderIngredientBuilder()` — only structural changes (add/remove row or group, picking a different linked item, creating a new item) trigger a re-render. This matches the existing `cust-price-in` pattern elsewhere in the file and avoids the input losing focus/cursor position on every keystroke, which a full-re-render-on-every-change approach would cause.

- [ ] **Step 1: CSS**

Insert after the existing `.recipe-edit-actions{display:flex;gap:8px;margin-top:14px;flex-wrap:wrap}` rule, before the blank line and `.grocery-intro{...}`:

```css
.ing-group-editor{border:1px solid var(--line);border-radius:10px;padding:10px;margin:0 0 10px}
.ing-heading-in{width:100%;box-sizing:border-box;border:none;border-bottom:1px dashed var(--line);
  padding:4px 2px 8px;margin-bottom:8px;font-weight:700;font-size:.84rem;font-family:inherit;background:transparent;color:var(--ink)}
.ing-heading-in::placeholder{font-weight:400;color:var(--muted)}
.ing-row-editor{display:flex;flex-wrap:wrap;gap:6px;align-items:center;margin:0 0 6px}
.ing-text-in{flex:2 1 220px;border:1px solid var(--line);border-radius:7px;padding:6px 8px;font-size:.87rem;font-family:inherit}
.ing-link-sel{flex:1 1 160px;border:1px solid var(--line);border-radius:7px;padding:6px 6px;font-size:.82rem;font-family:inherit;background:#fff;color:var(--ink)}
.ing-qty-in{flex:0 0 68px;border:1px solid var(--line);border-radius:7px;padding:6px 6px;font-size:.87rem;text-align:right}
.ing-unit-lbl{font-size:.76rem;color:var(--muted);flex:0 0 auto}
.new-item-form{display:flex;flex-wrap:wrap;gap:8px;align-items:center;background:var(--chip);border:1px dashed var(--line);
  border-radius:12px;padding:12px 14px;margin:8px 0}
.new-item-form input, .new-item-form select{border:1px solid var(--line);border-radius:8px;padding:8px 10px;font-size:.88rem;background:#fff;font-family:inherit}
.new-item-form label{display:flex;align-items:center;gap:5px;font-size:.82rem;color:var(--muted)}
```

- [ ] **Step 2: Builder state and rendering functions**

Insert directly after the `escHTML` function (`function escHTML(s){ return String(s).replace(/[&<>"']/g, c => ({"&":"&amp;","<":"&lt;",">":"&gt;",'"':"&quot;","'":"&#39;"}[c])); }`):

```js
/* ---- Structured ingredient builder (shared by recipe edit + create) ---- */
let recipeDraft = null;   // {key, isNew, title, serves, ing, steps} while creating/editing a recipe; null otherwise
let newItemDraft = null;  // {gi,ri} while the inline "create new item" form is open for that row; null otherwise

function newIngRow(){ return {text:"", item:null, qty:0}; }
function newIngGroup(heading){ return {heading:heading||"", rows:[newIngRow()]}; }
function cloneIng(ing){ return ing.map(g=>({heading:g.heading, rows:g.rows.map(r=>({...r}))})); }
function distinctSections(){ return [...new Set(allItems().map(i=>i.sec))]; }

function draftAddRow(gi){ recipeDraft.ing[gi].rows.push(newIngRow()); renderIngredientBuilder(); }
function draftRemoveRow(gi,ri){
  recipeDraft.ing[gi].rows.splice(ri,1);
  if(!recipeDraft.ing[gi].rows.length) recipeDraft.ing[gi].rows.push(newIngRow());
  renderIngredientBuilder();
}
function draftAddGroup(){ recipeDraft.ing.push(newIngGroup("")); renderIngredientBuilder(); }
function draftSetHeading(gi,val){ recipeDraft.ing[gi].heading=val; }
function draftSetText(gi,ri,val){ recipeDraft.ing[gi].rows[ri].text=val; }
function draftSetQty(gi,ri,val){ const n=parseFloat(val); recipeDraft.ing[gi].rows[ri].qty=(isNaN(n)||n<0)?0:n; }
function draftSetItem(gi,ri,val){
  if(val==="__new__"){ newItemDraft={gi,ri}; renderIngredientBuilder(); return; }
  recipeDraft.ing[gi].rows[ri].item = val||null;
  if(!val) recipeDraft.ing[gi].rows[ri].qty=0;
  renderIngredientBuilder();
}
function draftCancelNewItem(){ newItemDraft=null; renderIngredientBuilder(); }
function draftSaveNewItem(){
  const n=(document.getElementById('newItemName').value||"").trim();
  if(!n){ toast("Item name required"); return; }
  if(allItems().some(i=>i.n.toLowerCase()===n.toLowerCase())){ toast("An item with that name already exists"); return; }
  const secSel=document.getElementById('newItemSec').value;
  const secOther=(document.getElementById('newItemSecOther').value||"").trim();
  const sec = secSel==="__other__" ? (secOther||"Other") : secSel;
  const unit=(document.getElementById('newItemUnit').value||"item").trim();
  const divisible=document.getElementById('newItemDivisible').checked;
  const always=document.getElementById('newItemAlways').checked;
  const price=parseFloat(document.getElementById('newItemPrice').value)||0;
  customItems.push({n, sec, unit, divisible, always, price});
  saveCustomItems();
  buildCatalog(); initPrices();
  const {gi,ri}=newItemDraft;
  recipeDraft.ing[gi].rows[ri].item=n;
  newItemDraft=null;
  renderIngredientBuilder();
  toast("Item added");
}
function renderIngRowEditor(gi,ri,r){
  const opts = allItems().map(i=>`<option value="${escHTML(i.n)}" ${r.item===i.n?'selected':''}>${escHTML(i.n)}</option>`).join("");
  const linkedItem = r.item ? allItems().find(i=>i.n===r.item) : null;
  const qtyField = linkedItem ? `<input type="number" step="0.01" min="0" class="ing-qty-in" value="${r.qty}" onchange="draftSetQty(${gi},${ri},this.value)" placeholder="qty">
    <span class="ing-unit-lbl">${escHTML(linkedItem.unit||"")}</span>` : "";
  return `<div class="ing-row-editor">
    <input type="text" class="ing-text-in" placeholder="e.g. 2 bell peppers, chopped" value="${escHTML(r.text)}" onchange="draftSetText(${gi},${ri},this.value)">
    <select class="ing-link-sel" onchange="draftSetItem(${gi},${ri},this.value)">
      <option value="" ${!r.item?'selected':''}>— no grocery link —</option>
      ${opts}
      <option value="__new__">+ Create new item…</option>
    </select>
    ${qtyField}
    <button type="button" class="remove-x" title="Remove line" onclick="draftRemoveRow(${gi},${ri})">✕</button>
  </div>`;
}
function renderNewItemForm(){
  if(!newItemDraft) return "";
  const secs = distinctSections();
  return `<div class="new-item-form">
    <div class="ai-title">➕ New grocery item</div>
    <input type="text" id="newItemName" placeholder="Item name">
    <select id="newItemSec" onchange="document.getElementById('newItemSecOther').style.display=(this.value==='__other__'?'inline-block':'none')">
      ${secs.map(s=>`<option value="${escHTML(s)}">${escHTML(s)}</option>`).join("")}
      <option value="__other__">Other…</option>
    </select>
    <input type="text" id="newItemSecOther" placeholder="New section name" style="display:none">
    <input type="text" id="newItemUnit" placeholder="Unit (e.g. jar, bag, lb)">
    <label><input type="checkbox" id="newItemDivisible"> fractional amounts allowed</label>
    <label><input type="checkbox" id="newItemAlways"> always needed, not recipe-specific</label>
    <input type="number" step="0.01" min="0" id="newItemPrice" placeholder="Starting price">
    <button type="button" class="btn-edit" onclick="draftSaveNewItem()">Save item</button>
    <button type="button" class="btn-reset" onclick="draftCancelNewItem()">Cancel</button>
  </div>`;
}
function renderIngredientBuilder(){
  const el = document.getElementById('ingredientBuilder');
  if(!el || !recipeDraft) return;
  el.innerHTML = recipeDraft.ing.map((g,gi)=>`
    <div class="ing-group-editor">
      <input type="text" class="ing-heading-in" placeholder="Section heading (optional)" value="${escHTML(g.heading)}" onchange="draftSetHeading(${gi},this.value)">
      ${g.rows.map((r,ri)=>renderIngRowEditor(gi,ri,r)).join("")}
      <button type="button" class="btn-reset" onclick="draftAddRow(${gi})">+ Add ingredient</button>
    </div>`).join("")
    + `<button type="button" class="btn-reset" onclick="draftAddGroup()">+ Add section heading</button>`
    + renderNewItemForm();
}
```

- [ ] **Step 3: Manual verification (via console — no entry-point UI exists until Tasks 9/10 land)**

Serve locally, console: `syncRef=null;`.

1. Set up a throwaway container and draft, then render:
   ```js
   document.getElementById('recipeDetail').innerHTML = '<div id="ingredientBuilder"></div>';
   document.getElementById('recipeDetail').style.display = 'block';
   recipeDraft = {key:null, isNew:true, title:"", serves:"", ing:[newIngGroup("")], steps:[]};
   renderIngredientBuilder();
   ```
   Confirm one empty group renders with one empty row (text input, "— no grocery link —" select, no qty field yet, a ✕ button) and a "+ Add section heading" button below it.
2. Click into the row's text field, type something, click elsewhere (blur) — confirm `recipeDraft.ing[0].rows[0].text` reflects it in the console, and the field did **not** lose focus mid-typing (open devtools, type multiple characters continuously — cursor should stay put throughout, since text inputs don't re-render on every keystroke).
3. Change the row's select to an existing item (e.g. "Bell peppers") — confirm a quantity input + unit label ("bell pepper") appears immediately next to it, and `recipeDraft.ing[0].rows[0].item==="Bell peppers"`.
4. Select "+ Create new item…" on a row — confirm the inline new-item form appears with a section dropdown pre-populated from the real catalog's sections plus "Other…". Fill in name "Test Sauce", pick an existing section, unit "jar", leave checkboxes off, price 2.99, click "Save item". Confirm: the form disappears, that row's link is now "Test Sauce", `customItems` contains it, and — go check the **Prices** tab — "Test Sauce" appears there with full Costco/Kroger/Aldi/Publix/Walmart pricing already seeded to $2.99 for every store (confirms `buildCatalog()`/`initPrices()` picked it up immediately). Then clean up: `customItems = customItems.filter(i=>i.n!=="Test Sauce"); saveCustomItems(); buildCatalog(); initPrices();`
5. Click "+ Add ingredient" — confirm a second empty row appears in the same group. Click "+ Add section heading" — confirm a second, empty group with its own row appears. Click a row's ✕ — confirm it's removed (and if it was the last row in a group, an empty row takes its place rather than leaving a group with zero rows).
6. Clean up the throwaway state: `recipeDraft=null; newItemDraft=null; document.getElementById('recipeDetail').style.display='none';` and reload the page to confirm nothing you did in this step persisted anywhere it shouldn't (the Recipes tab should look completely normal on reload).

- [ ] **Step 4: Commit**

```bash
cd "C:/Users/Matt/GitHub/dinner-meal-plan"
git add index.html
git commit -m "Add structured ingredient builder component (rows, groups, inline item creation)"
```

---

### Task 9: Wire the builder into editing (existing recipes, built-in or custom)

**Files:**
- Modify: `index.html` (CSS in `<style>`; new functions after the builder from Task 8; `showRecipeDetail`; `openRecipe`)

**Context:** This re-introduces an "Edit" entry point on the recipe detail page (removed in Part A along with the old textarea editor) using the Task 8 builder instead of free text. `saveRecipeForm`/`renderRecipeForm` are written here to handle both "editing an existing recipe" and "creating a new one" in one shared function — Task 10 adds only the entry point that populates a blank draft and calls into this same machinery, so there's exactly one save/render path for both cases, not two parallel ones.

- [ ] **Step 1: CSS for the title/serves inputs**

Insert after the `.new-item-form label{...}` rule from Task 8:

```css
.recipe-title-in{width:100%;box-sizing:border-box;border:1px solid var(--line);border-radius:8px;padding:8px 10px;font-size:1.1rem;font-weight:700;font-family:inherit;margin-bottom:8px;color:var(--ink)}
.recipe-serves-in{width:160px;border:1px solid var(--line);border-radius:8px;padding:6px 8px;font-size:.88rem;font-family:inherit;margin-bottom:10px;color:var(--ink)}
```

- [ ] **Step 2: Re-add `recipeEditKey`/`recipeDetailScale` and add the form-level draft setters**

Insert directly after the builder functions from Task 8 (after `renderIngredientBuilder`):

```js
let recipeEditKey = null;      // recipe key currently being edited on the detail page, or "the" key while creating; null otherwise
let recipeDetailScale = 1;     // last-rendered scale for the recipe currently open in read mode — survives the edit round-trip

function draftSetTitle(val){ recipeDraft.title=val; }
function draftSetServes(val){ recipeDraft.serves=val; }
function draftSetSteps(val){ recipeDraft.steps = val.split("\n").map(s=>s.trim()).filter(Boolean); }

function genRecipeKey(title){
  const base = slug(title) || "recipe";
  let key = "custom-"+base, n=2;
  while(R[key] || customRecipes[key]){ key = "custom-"+base+"-"+n; n++; }
  return key;
}

function renderRecipeForm(){
  const d = recipeDraft;
  document.getElementById("recipeDetail").innerHTML = `
    <button class="btn-reset" style="margin-bottom:12px" onclick="cancelRecipeForm()">← Cancel</button>
    <article class="recipe">
      <input type="text" class="recipe-title-in" placeholder="Recipe title" value="${escHTML(d.title)}" onchange="draftSetTitle(this.value)">
      <input type="text" class="recipe-serves-in" placeholder="Serves (e.g. 4 or 4–6)" value="${escHTML(d.serves)}" onchange="draftSetServes(this.value)">
      <div class="cols">
        <div><h4>Ingredients</h4><div id="ingredientBuilder"></div></div>
        <div><h4>Instructions</h4>
          <textarea id="recipeStepsIn" class="recipe-edit-box" rows="10" onchange="draftSetSteps(this.value)">${escHTML(d.steps.join("\n"))}</textarea>
          <div class="edit-hint">One step per line.</div>
        </div>
      </div>
      <div class="recipe-edit-actions">
        <button class="btn-edit" onclick="saveRecipeForm()">💾 Save</button>
        <button class="btn-reset" onclick="cancelRecipeForm()">Cancel</button>
        ${(!d.isNew && customRecipes[d.key]) ? `<button class="btn-reset" onclick="deleteRecipe('${d.key}')">🗑️ Delete recipe</button>` : ""}
      </div>
    </article>`;
  renderIngredientBuilder();
  document.getElementById("recipeIndex").style.display="none";
  document.getElementById("recipeDetail").style.display="block";
}

function editRecipe(k){
  const rd = recipeData(k), base = baseRecipe(k);
  recipeDraft = {key:k, isNew:false, title:base.title, serves:base.serves, ing:cloneIng(rd.ing), steps:rd.steps.slice()};
  recipeEditKey = k;
  renderRecipeForm();
}

function cancelRecipeForm(){
  const wasNew = recipeDraft && recipeDraft.isNew;
  const k = recipeDraft && recipeDraft.key;
  recipeDraft=null; newItemDraft=null; recipeEditKey=null;
  if(wasNew) hideRecipeDetail();
  else showRecipeDetail(k, recipeDetailScale);
}

function saveRecipeForm(){
  const d = recipeDraft;
  if(!d.title.trim()){ toast("Title required"); return; }
  const cleanIng = d.ing.map(g=>({heading:g.heading, rows:g.rows.filter(r=>r.text.trim())})).filter(g=>g.rows.length);
  if(!cleanIng.length){ toast("Add at least one ingredient"); return; }
  if(d.isNew){
    const key = genRecipeKey(d.title);
    customRecipes[key] = {title:d.title.trim(), serves:d.serves.trim()||"4", ing:cleanIng, steps:d.steps};
    saveCustomRecipes();
    recipeDraft=null; newItemDraft=null; recipeEditKey=null;
    renderAllRecipes(); renderRecipes();
    showRecipeDetail(key);
    toast("Recipe added");
  } else {
    const k = d.key;
    if(customRecipes[k]){
      customRecipes[k] = {title:d.title.trim(), serves:d.serves.trim()||"4", ing:cleanIng, steps:d.steps};
      saveCustomRecipes();
    } else {
      recipeEdits[k] = {ing:cleanIng, steps:d.steps};
      saveRecipeEdits();
    }
    recipeDraft=null; newItemDraft=null; recipeEditKey=null;
    renderAllRecipes(); renderRecipes();
    showRecipeDetail(k, recipeDetailScale);
    toast("Recipe saved");
  }
}
```

Note: `d.key`'s "Delete recipe" button only renders when `customRecipes[d.key]` exists — i.e. never for a built-in recipe, and never while `isNew` (there's nothing to delete yet). `deleteRecipe` itself is written in Task 11 — the button calls it, but it doesn't exist as a function until that task lands; that's fine, this task's own verification (Step 5 below) never clicks it.

- [ ] **Step 3: Re-add the Edit button and edit-mode delegation to `showRecipeDetail`**

Current code (Part A's version, further updated by Task 7 Steps 9/12 — confirm your file matches this before editing, since two earlier tasks in this plan already touched this function):

```js
function showRecipeDetail(k,scale){
  scale = scale||1;
  const r=baseRecipe(k);
  const link = r.url ? `<a href="${r.url}" target="_blank" rel="noopener">${r.src} ↗</a>` : (r.src||"your own recipe");
  const note = r.note ? `<div class="note-box">💡 ${r.note}</div>` : "";
  const scaleNote = (scale!==1) ? `<div class="note-box">👥 ${scale}x tonight — grocery list already adjusted.</div>` : "";
  const rd=recipeData(k);
  document.getElementById("recipeDetail").innerHTML =
    `<button class="btn-reset" style="margin-bottom:12px" onclick="hideRecipeDetail()">← All recipes</button>
     <article class="recipe">
       <h3>${escHTML(r.title)}</h3>
       <div class="meta">Serves ${escHTML(r.serves)} · Source: ${link}</div>
       ${scaleNote}
       <div class="cols"><div><h4>Ingredients</h4>${ingHTML(rd.ing)}</div>
       <div><h4>Instructions</h4><ol class="steps">${rd.steps.map(s=>`<li>${escHTML(s)}</li>`).join("")}</ol>${note}</div></div>
     </article>`;
  document.getElementById("recipeIndex").style.display="none";
  document.getElementById("recipeDetail").style.display="block";
  window.scrollTo({top:0,behavior:"smooth"});
}
```

Replace with:

```js
function showRecipeDetail(k,scale){
  if(recipeEditKey===k){ renderRecipeForm(); return; }
  scale = scale||1;
  const r=baseRecipe(k);
  const link = r.url ? `<a href="${r.url}" target="_blank" rel="noopener">${r.src} ↗</a>` : (r.src||"your own recipe");
  const note = r.note ? `<div class="note-box">💡 ${r.note}</div>` : "";
  const scaleNote = (scale!==1) ? `<div class="note-box">👥 ${scale}x tonight — grocery list already adjusted.</div>` : "";
  recipeDetailScale = scale;
  const rd=recipeData(k);
  document.getElementById("recipeDetail").innerHTML =
    `<button class="btn-reset" style="margin-bottom:12px" onclick="hideRecipeDetail()">← All recipes</button>
     <article class="recipe">
       <h3>${escHTML(r.title)}</h3>
       <div class="meta">Serves ${escHTML(r.serves)} · Source: ${link}</div>
       ${scaleNote}
       <button class="btn-edit" style="margin:8px 0" onclick="editRecipe('${k}')">✏️ Edit</button>
       <div class="cols"><div><h4>Ingredients</h4>${ingHTML(rd.ing)}</div>
       <div><h4>Instructions</h4><ol class="steps">${rd.steps.map(s=>`<li>${escHTML(s)}</li>`).join("")}</ol>${note}</div></div>
     </article>`;
  document.getElementById("recipeIndex").style.display="none";
  document.getElementById("recipeDetail").style.display="block";
  window.scrollTo({top:0,behavior:"smooth"});
}
```

- [ ] **Step 4: Re-add the stale-edit-state guard in `openRecipe`**

Current code:

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

Replace with:

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

(This is the exact same fix from the prior session's recipe-editing work, re-applied because Part A removed `recipeEditKey` and this task re-adds it — without this line, navigating to a recipe from the Overview while a different recipe was mid-edit would incorrectly land back in edit mode.)

- [ ] **Step 5: Manual verification**

Serve locally, console: `syncRef=null;`.

1. Open any built-in recipe (e.g. Black Pepper Chicken) from the **Recipes** tab. Confirm an **"✏️ Edit"** button now appears (it didn't since Part A).
2. Click it. Confirm the builder renders: title/serves inputs pre-filled, each ingredient group/row pre-filled with its actual text and (where linked) its actual item + quantity — spot check 2–3 rows against what you know that recipe's grocery links are.
3. Change one row's text, change another row's linked item, add a new ingredient row and link it to an existing catalog item with a quantity, then click **Save**. Confirm: toast "Recipe saved", the read view shows your edits, and the same recipe's card on its Cycle A/B tab (if planned) shows the same edits.
4. Go to **Cycle A/B Grocery** and confirm the newly-linked ingredient's quantity is reflected in the shopping list.
5. Reopen the recipe, click Edit, click **Cancel** without saving — confirm your in-progress (unsaved) changes are discarded and the previously-saved version is shown.
6. Confirm the stale-edit-state fix still works: open a recipe, click Edit, then click a *different* day's recipe card from the Overview tab without saving/canceling first — confirm it opens that other recipe in **read** mode, not stuck in edit mode.
7. Confirm the scale-note-persists-through-edit fix still works (this logic is unchanged from before Part A, just re-verifying after all the surrounding rewrites): set a night to 2x, open its recipe, confirm the "👥 2x tonight" note shows, click Edit, click Save — confirm the note is still showing immediately after, without navigating away.

- [ ] **Step 6: Commit**

```bash
cd "C:/Users/Matt/GitHub/dinner-meal-plan"
git add index.html
git commit -m "Wire structured ingredient builder into recipe editing"
```

---

### Task 10: Add the recipe-creation entry point

**Files:**
- Modify: `index.html` (the `#recipes` section's HTML; new `addRecipeClicked` function)

**Context:** Task 9 already built the full save/render machinery to handle `recipeDraft.isNew` — this task only adds the button that starts a blank draft and the one function behind it.

- [ ] **Step 1: Add the button**

Current code:

```html
  <section class="panel" id="recipes">
    <h2 class="section-title">All Recipes</h2>
    <p class="lead">Every recipe in the rotation. Tap a title to open its full ingredients and instructions.</p>
    <div id="recipeIndex"></div>
    <div id="recipeDetail" style="display:none"></div>
  </section>
```

Replace with:

```html
  <section class="panel" id="recipes">
    <h2 class="section-title">All Recipes</h2>
    <p class="lead">Every recipe in the rotation. Tap a title to open its full ingredients and instructions.</p>
    <button class="btn-edit" style="margin-bottom:14px" onclick="addRecipeClicked()">➕ Add a recipe</button>
    <div id="recipeIndex"></div>
    <div id="recipeDetail" style="display:none"></div>
  </section>
```

- [ ] **Step 2: Add `addRecipeClicked`**

Insert directly after `genRecipeKey` (added in Task 9):

```js
function addRecipeClicked(){
  recipeDraft = {key:null, isNew:true, title:"", serves:"", ing:[newIngGroup("")], steps:[]};
  renderRecipeForm();
}
```

- [ ] **Step 3: Manual verification**

Serve locally, console: `syncRef=null;`.

1. Go to the **Recipes** tab. Confirm a **"➕ Add a recipe"** button appears above the recipe index.
2. Click it. Confirm the same builder form from Task 9 renders, but empty: blank title/serves, one empty ingredient group with one empty row, empty steps box. Confirm there is **no** "Delete recipe" button (nothing to delete yet).
3. Fill in a title ("Test Tacos"), serves ("4"), one ingredient line linked to an existing catalog item (e.g. "Salsa", qty 1), one unlinked line ("Tortillas, to serve"), and two steps. Click **Save**.
4. Confirm: toast "Recipe added", the detail page now shows "Test Tacos" in read mode with an Edit button, and it appears in the **Recipes** tab index, the **meal picker** dropdown, and — once you plan it on a day via Edit Plan mode — the **Cycle A/B Recipes** tab and the **grocery list** (with "Salsa" correctly showing up at qty 1, attributed to "Test Tacos" in its "used by" text).
5. Click Cancel on a *fresh, unsaved* "Add a recipe" session instead (start over, click Add, type something, click Cancel without saving) — confirm it returns to the recipe index with nothing created (`Object.keys(customRecipes)` should not include a half-finished entry).
6. Clean up: delete "Test Tacos" via the Edit view's "Delete recipe" button if Task 11 has landed already, or manually via `delete customRecipes[<its key>]; saveCustomRecipes(); renderAllRecipes();` if not — also clear it from any day you planned it on (`mealPlan.A`/`.B`) and re-render.

- [ ] **Step 4: Commit**

```bash
cd "C:/Users/Matt/GitHub/dinner-meal-plan"
git add index.html
git commit -m "Add recipe-creation entry point"
```

---

### Task 11: Recipe deletion (custom recipes only)

**Files:**
- Modify: `index.html` (new `deleteRecipe` function, called from the button `renderRecipeForm` already renders per Task 9)

**Context:** Task 9's `renderRecipeForm` already renders a "🗑️ Delete recipe" button when editing a custom recipe, calling `deleteRecipe('${d.key}')` — that function didn't exist yet. This task adds it. Deleting a recipe that's currently planned on any day in either cycle must clear that day back to "no meal" (matching `removeMeal`'s existing behavior) rather than leaving a dangling key in `mealPlan` that every other function would then have to guard against.

- [ ] **Step 1: Add `deleteRecipe`**

Insert directly after `saveRecipeForm` (from Task 9):

```js
function deleteRecipe(k){
  if(!customRecipes[k]) return;
  if(!confirm("Delete this recipe? This can't be undone.")) return;
  delete customRecipes[k];
  saveCustomRecipes();
  ['A','B'].forEach(slot=>{
    let touched=false;
    mealPlan[slot].forEach((mk,i)=>{ if(mk===k){ mealPlan[slot][i]=null; mealScale[slot][i]=1; touched=true; } });
    if(touched){ saveMeals(slot); saveScale(slot); }
  });
  recipeDraft=null; recipeEditKey=null;
  hideRecipeDetail();
  renderAllRecipes(); renderOverview(); renderRecipes(); renderGrocery('A'); renderGrocery('B');
  toast("Recipe deleted");
}
```

- [ ] **Step 2: Manual verification**

Serve locally, console: `syncRef=null;`.

1. Create a throwaway custom recipe (via "➕ Add a recipe", per Task 10) and plan it on one day in Cycle A via Edit Plan mode.
2. Open it, click Edit, click **"🗑️ Delete recipe"** — confirm a browser confirm dialog appears. Click Cancel on that dialog — confirm nothing happened (`customRecipes` still has it, the day still shows it planned).
3. Repeat and confirm the dialog this time. Confirm: toast "Recipe deleted", you're returned to the Recipes index, the recipe no longer appears there or in the meal picker, and — check the **Overview** tab — the day it was planned on now shows "— no meal —" (not a broken reference).
4. Confirm the built-in recipes have **no** "Delete recipe" button at all (open any of the 21, click Edit — only Save/Cancel should appear).
5. Run `Object.keys(customRecipes).length === 0` → expect `true` (fully cleaned up).

- [ ] **Step 3: Commit**

```bash
cd "C:/Users/Matt/GitHub/dinner-meal-plan"
git add index.html
git commit -m "Add custom recipe deletion with meal-plan cleanup"
```

---

### Task 12: Sync `customItems` and `customRecipes` through Firebase

**Files:**
- Modify: `index.html` (function `applyRemote`; function `syncPush`)

**Context:** Both stores are global, not period-keyed — same category as `recipeEdits`, added the same way in the prior session's Task 5. `customItems` needs `buildCatalog()` re-run *before* `ensureDefaults()` fires (which already runs later in `applyRemote`), so a custom item synced from another device gets folded into `CATALOG` and has its price seeded in the same pass, instead of only taking effect on the next full reload.

- [ ] **Step 1: Read `customItems`/`customRecipes` in `applyRemote`, rebuild the catalog before `ensureDefaults()` runs**

Current code:

```js
function applyRemote(data){
  if(!data || !data.state){ syncPush(); return; }       // empty remote → seed it from local
  if(data.rev && data.rev===myRev) return;               // ignore our own echo
  let s; try{ s=JSON.parse(data.state); }catch(e){ return; }
  applyingRemote=true;
  try{
    if(s.staples) staplePrices=s.staples;
    if(s.prices)  prices=s.prices;
    if(s.store)   storeSel=s.store;
    if(s.unit)    unitSel=s.unit;
    ensureDefaults();   // fill in any items the remote payload predates (e.g. newly added)
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
    renderStapleRows(); renderCatalog(); renderOverview(); renderRecipes(); renderGrocery("A"); renderGrocery("B");
    setSyncStatus("🟢 synced");
  } finally { applyingRemote=false; }
}
```

Replace with:

```js
function applyRemote(data){
  if(!data || !data.state){ syncPush(); return; }       // empty remote → seed it from local
  if(data.rev && data.rev===myRev) return;               // ignore our own echo
  let s; try{ s=JSON.parse(data.state); }catch(e){ return; }
  applyingRemote=true;
  try{
    if(s.staples) staplePrices=s.staples;
    if(s.prices)  prices=s.prices;
    if(s.store)   storeSel=s.store;
    if(s.unit)    unitSel=s.unit;
    customItems = s.customItems||[];
    customRecipes = s.customRecipes||{};
    buildCatalog();     // fold any synced custom items into CATALOG before seeding their prices below
    ensureDefaults();   // fill in any items the remote payload predates (e.g. newly added)
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
    saveCustomItems(); saveCustomRecipes();
    save(ckChecks(CTX.A.key),checks.A); save(ckChecks(CTX.B.key),checks.B);
    save(ckCustom(CTX.A.key),custom.A); save(ckCustom(CTX.B.key),custom.B);
    save(ckMeals(CTX.A.key),mealPlan.A); save(ckMeals(CTX.B.key),mealPlan.B);
    save(ckScale(CTX.A.key),mealScale.A); save(ckScale(CTX.B.key),mealScale.B);
    saveRecipeEdits();
    renderStapleRows(); renderCatalog(); renderAllRecipes(); renderOverview(); renderRecipes(); renderGrocery("A"); renderGrocery("B");
    setSyncStatus("🟢 synced");
  } finally { applyingRemote=false; }
}
```

(`renderAllRecipes()` is added to the final render batch — previously nothing needed to re-render the Recipes index on a remote update, since `R` never changed at runtime; now a remote-synced custom recipe needs the index to pick it up immediately, not just on next reload.)

- [ ] **Step 2: Write `customItems`/`customRecipes` in `syncPush`**

Current code:

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

Replace with:

```js
function syncPush(){
  if(!syncRef) return;
  remoteChecksAll[CTX.A.key]=checks.A; remoteChecksAll[CTX.B.key]=checks.B;
  remoteCustomAll[CTX.A.key]=custom.A; remoteCustomAll[CTX.B.key]=custom.B;
  remoteMealsAll[CTX.A.key]=mealPlan.A; remoteMealsAll[CTX.B.key]=mealPlan.B;
  remoteScaleAll[CTX.A.key]=mealScale.A; remoteScaleAll[CTX.B.key]=mealScale.B;
  myRev = DEVICE+"-"+(++seq);
  const state = JSON.stringify({staples:staplePrices, prices:prices, store:storeSel, unit:unitSel, checks:remoteChecksAll, custom:remoteCustomAll, meals:remoteMealsAll, mealScale:remoteScaleAll, recipeEdits:recipeEdits, customItems:customItems, customRecipes:customRecipes});
  syncRef.set({rev:myRev, updatedAt:Date.now(), state:state}).catch(function(e){ console.warn('sync push failed',e); });
}
```

- [ ] **Step 3: Manual verification (read-only against the real sync, same rule used for every prior sync task)**

This is the one step where sync must stay **on**, since the thing being verified is the sync path itself. Keep it strictly read-only — do not call any mutating function in this session.

1. Serve locally, open the app **without** running `syncRef=null;`.
2. Console: `JSON.parse((await firebase.database().ref('mealplan/shared').once('value')).val().state).customItems` and the same for `customRecipes` → expect `undefined` or empty (nothing has pushed through this path yet).
3. Do not create a test recipe or item in this session — that would push real data to the shared database.

- [ ] **Step 4: Commit**

```bash
cd "C:/Users/Matt/GitHub/dinner-meal-plan"
git add index.html
git commit -m "Sync custom recipes and catalog items through the shared Firebase state"
```

---

### Task 13: Full integrated verification pass (both plans together)

**Files:** none (verification only)

**Context:** Every individual task across both Part A and Part B has already been verified on its own. This task checks the whole system together, end to end, in one continuous session — the same kind of capstone check the prior session's meal-plan work ended on, extended to cover everything this two-plan sequence touched.

- [ ] **Step 1: Serve and stub sync**

```bash
cd "C:/Users/Matt/GitHub/dinner-meal-plan" && python -m http.server 8765
```

Open `http://localhost:8765/`, console: `syncRef=null;`.

- [ ] **Step 2: Walk through the full feature set**

1. **Migration integrity**: re-run Part A Task 6's snapshot diff one more time against its original baseline, on this fully-built final state — confirm it still shows no unexpected differences.
2. **Edit an existing recipe**: change an ingredient's linked item and quantity, save, confirm the grocery list updates.
3. **Create a new recipe**: full flow from Task 10, including creating a brand-new catalog item inline for one of its ingredients (Task 8), plan it, confirm it prices out correctly on the grocery list including under a 2x/3x serving scale (Part A/prior-session scale logic still applies unchanged, since it operates on `recipeData(k).ing` regardless of whether `k` is built-in or custom).
4. **Delete** that recipe (Task 11) — confirm full cleanup (index, meal picker, planned day, no dangling grocery contribution).
5. **Cross-feature interaction**: create a recipe, plan it at 2x, edit it (change an ingredient), confirm the grocery list still reflects both the edit and the 2x scale correctly together.
6. **Purchased pile interaction**: check off an item whose quantity came from a custom recipe — confirm it moves into the Purchased pile with the correct (custom-recipe-driven) quantity, same as any built-in-recipe-driven item.
7. **XSS re-check**: repeat Task 7 Step 13's escaped-title test one more time in this final state, end to end through the actual UI (type `<img src=x onerror="window.__x=true">` as a recipe title via the real "Add a recipe" form, not via console injection) — confirm `window.__x` stays `false` everywhere that title renders (index, detail page, meal picker, Cycle A/B tab if planned). Delete it afterward.
8. Reload the page (re-run `syncRef=null;` after reload) and confirm anything you created/edited and didn't clean up in this session persisted correctly through `localStorage` (or clean up everything before reloading, to leave the app in its original state — recommended, since this is meant to be a smoke test, not a place to accumulate test data).

- [ ] **Step 3: Confirm nothing is left uncommitted**

```bash
cd "C:/Users/Matt/GitHub/dinner-meal-plan"
git status
```

Expected: `nothing to commit, working tree clean`.

- [ ] **Step 4: Push**

Only after every check above passes:

```bash
cd "C:/Users/Matt/GitHub/dinner-meal-plan"
git push
```

This is the first push since Part A started — everything up to this point (both plans) has been local-only commits, per the note at the top of Part A.

---

## Spec coverage check (Part B's share of the design)

- `customRecipes` store, full CRUD, unified lookups via `baseRecipe` → Task 7, Task 11.
- Structured ingredient builder (rows, groups, inline new-item creation) → Task 8.
- Editing existing recipes through the builder → Task 9.
- Recipe creation → Task 10.
- Recipe deletion with meal-plan cleanup → Task 11.
- New catalog items get full per-store pricing automatically (via `allItems()`/`buildCatalog()`, established in Part A and exercised for the first time here) → Task 8, verified in Task 8 Step 3.4.
- Escaping extended to every new user-authored-text render path (custom recipe titles/serves, catalog item names via the picker) → Task 7 Steps 11–12, re-verified end to end in Task 13.
- Sync for both new stores → Task 12.
- "One connected system" principle (Prices/Recipes/Grocery tabs) → exercised throughout; no separate "custom" code path exists anywhere except the data stores themselves.

Out of scope, per the design spec: deleting/renaming built-in or custom catalog items, reordering ingredient rows/groups, wiring custom items into the shared staple-pricing table (moot — they already get full per-store pricing without it, per the spec revision), and anything to do with step entry/display beyond what already existed.
