# Unified vendor (metal + other categories coexist) — design

## Goal
One vendor can hold BOTH metal categories (silver/gold, karat+rate) AND other categories
(bronze/steel, multiplier). In Calc, the **metal button** decides which set the category
dropdown shows: Gold/Silver → metal categories; Other → other categories.

## Non-negotiable: zero change to existing data
Every existing vendor and every existing item must price and scan **byte-identically**.
The feature is purely additive.

## Key result that makes it safe
The discriminator **`catIsOther(cat) = cat.multiplier !== undefined`** is *exactly
equivalent* to the old `vendor.vendorType === 'other'` for ALL existing data — verified
against the live backup: 0 violations across 14 vendors (every other category has a
multiplier; no metal category has one). So each `vendorType` → `catIsOther` swap produces
identical output for current vendors.

## Data model (Design A — single array, no migration)
- A vendor keeps ONE `laborCategories` array holding BOTH kinds. Metal cats have
  karat/rate/rateType (no multiplier). Other cats have `multiplier` + karat `'—'`.
- `discount` (supplier %) applies to the other side (already exists on other vendors).
- `vendorType:'other'` is written ONLY for a *pure* other vendor (no metal cats). A mixed
  vendor has no `vendorType`. Nothing relies on `vendorType` for logic anymore.
- **catIdx stays a single-array index** → existing items' catIdx unchanged. No migration.

## Helpers (new)
- `catIsOther(c)` → `!!(c && (c.multiplier !== undefined || c.type === 'other'))`
- `vendorHasMetal(v)` → any non-deleted non-other cat
- `vendorHasOther(v)` → any non-deleted other cat

## Item pricing/creation — already metal-driven, NO change
- `calcInvItemPrice`: other item (`item.metal==='other'`) → `fixedRetail`; metal item → its
  metal category. Both key on `item.metal` (stored). **Not touched.**
- Save-from-label: `isOther = (metal === 'other')` (calc global), stores `metal:'other'` +
  `fixedRetail` + `catIdx`. Works for a mixed vendor already. **Not touched.**
- `getItemCat`: name-first then catIdx into `getVendorCategories`. **Not touched.**

## Changes (all equivalence-preserving on existing data)
1. Add helpers `catIsOther`/`vendorHasMetal`/`vendorHasOther`.
2. `catOptionLabel(cat, isOtherType)` → use `isOtherType || catIsOther(cat)`.
3. `calcOtherRetail`: get the selected vendor regardless of vendorType; discount from
   `v.discount`; multiplier from the selected cat.
4. `buildLabelStrings`: `isOtherVendor = catIsOther(cat)`.
5. `buildCodeMap`: `isOtherType = catIsOther(cat)` (base-only key still per other cat).
6. `updateWorkTypeOptions`: for a MIXED vendor, filter the dropdown by the calc `metal`
   (other cats iff metal==='other'), keeping the ORIGINAL index; label via `catIsOther(c)`.
   Pure vendors: unchanged (no filtering). Reset workType if the selection is hidden.
7. `setMetal`: refresh the category dropdown so switching metal reshows the right set.
8. `editVendor`: load BOTH sections (metal→#laborList, other→#vfOtherCatList), each row
   tagged `dataset.origIdx`; show both. Discount loaded.
9. `saveVendor`: read BOTH lists; rebuild `laborCategories` by placing each row at its
   `origIdx` (positions preserved) and appending new rows; set `discount` if other rows;
   set `vendorType:'other'` only if other-only. Freeze abbrs across the combined array.
   Remove the v12.17 destructive-type guard (no longer reachable — both sets persist).
10. `_invEditVendorOptions`: `vendorType!=='other'` → `vendorHasMetal(v)`.
11. `_invEditCatOptions`: exclude other cats (`!catIsOther(c)`) from a metal piece's move list.
12. Bulk sheet vendor filter → `vendorHasMetal(v)`; `updateBulkCatSelect` → metal cats only.
13. workType change listener: sync karat only for a metal cat (`!catIsOther(cat)`).
14. `renderVendors`: show `×mult` for other cats in the code rows.

## Order-preservation (the one subtle part)
Both directions must keep catIdx:
- pure-metal → +other: metal cats keep idx 0..k-1, new other cats append. ✓
- pure-other (Butterflyfish) → +metal: other cats keep idx 0..k-1 via `origIdx`, new metal
  cats append. A naive "metal-first" rebuild would shift them — hence the `origIdx` merge.

## Verification plan
- Equivalence: prove `catIsOther === (vendorType==='other')` per cat (done, 0 violations).
- Merge unit test: pure metal / pure other / mixed both directions → indices preserved.
- Browser smoke test: inject Amaya-as-mixed, price a silver code (identical) + a bronze
  other code (multiplier), switch metal to refilter, buildCodeMap collision check.
- No edits to calcInvItemPrice / getItemCat / catAbbrs / vendorAbbr.
