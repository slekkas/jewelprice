# Similar Items (Grouped Inventory) — Design Spec

**Status:** Approved for implementation. Discussion-only until this point; no code has been written yet.
**Author:** Design worked out with Sotiris over chat, July 2026.
**Scope:** `jewelprice.htm` (the pricing app) and the public showroom (`showroom.htm` / `item.htm`).

---

## 1. The problem

Sometimes a batch of near-identical pieces comes in — e.g. 12 silver-925 gold-plated
necklaces, all the same design, same vendor cost, same retail, differing only slightly
in weight. Today each piece would need its own inventory entry, its own reference code,
its own card. That's 12 near-duplicate cards to scroll past, and it will happen more and
more often.

We want **one card that represents many physical pieces of the same design**, with a live
count ("12 in stock" → sell one → "11 in stock"), while keeping every piece's exact
weight / cost / retail / profit intact underneath.

## 2. Guiding principle (reuse what already works)

The app **already** solved this shape for sales: a cart checkout stores **one real record
per item** (so profit, leaderboards, and totals stay exact) but **collapses them into a
single row** in the UI via `_groupSalesForDisplay` (the "🛍 N items" badge). We apply the
**same philosophy** to inventory:

> **Keep N exact item records underneath. Collapse them into one card at display time.**

We do **not** build a single record that holds many weights inside it — that would fight
the sale flow, QR resolution, gain %, the showroom mirror, and restore-on-delete, all of
which key off individual `inv_...` item ids.

## 3. Data model

### 3.1 The grouping key: `groupId` (NOT the ref code)

Each piece in a group carries a new hidden field:

```
groupId: "grp_" + Date.now() + "_" + <rand>   // shared by all pieces in the group
```

**Why a dedicated `groupId` and not the shared reference code:**
- Existing items we retro-group already have **different** ref codes and **already-printed
  tags** — we must be able to group them without reprinting.
- Color variants (§8) may want distinct ref codes but still belong to one group.

Ref-code equality is the *common* case (a fresh batch prints the same code on every tag),
but `groupId` is the real key. An item with **no** `groupId` renders exactly as it does
today — the feature is purely additive and legacy items are untouched.

### 3.2 New / reused fields on an inventory item

| Field          | Type              | Meaning                                                        |
|----------------|-------------------|---------------------------------------------------------------|
| `groupId`      | string (optional) | Shared id for all pieces of one design. Absent = ungrouped.   |
| `color`        | string (optional) | **General field on ALL items** (single or grouped). Replaces manually-typed color in the description; filterable/searchable; doubles as the grouping variant that splits colored groups in the showroom (§8). |
| `seqId`        | existing          | Per-piece sequence. Pieces in a batch may share one, or not.  |
| `weight`       | existing          | Per-piece, exact. Drives QR and gain.                         |
| `qrCode`       | existing          | Per-piece, **unique** (encodes weight → naturally differs).   |

Nothing about the sale record, QR format, or `PUBLIC_SAFE_FIELDS` needs to change for
Phase 1 **except** adding `groupId` and `color` to `PUBLIC_SAFE_FIELDS` (so the showroom
can group by `groupId` and display/split by `color`).

**`color` is a first-class field independent of grouping.** It is added to all three item
forms (label creation, scan-to-inventory, edit) in Phase 1 and applies to single items too
— it replaces the color the owner currently types by hand into the description and makes
color filterable/searchable in the app (and shown in the showroom). Grouped colored items
(§8) simply reuse this same field as the split dimension.

**Grouping never changes the item count.** It is display-only — no record is deleted,
merged, or combined. 407 items grouped in any way remain 407 items; only the number of
*cards* shown changes. (Same principle as multi-item sales: 3 items sold together are still
3 records, shown as one row.)

### 3.3 The canonical / representative piece

For any display that needs "the group as one thing" (card header, showroom card) we pick a
**representative**: the first in-stock member by `dateAdded` (stable, deterministic). It
supplies the representative photo, description, type, metal, and the ref code shown on the
collapsed card.

### 3.4 Metal- and type-agnostic

Grouping and the `color` field apply **identically to gold, silver, and Other-type items** —
nothing here is gold-specific. The color field, grouping, count, price (single or range),
and showroom split all work the same for a silver bracelet or an "Other" fixed-price piece
as for a gold necklace. A **blank** color labels its bucket by the item's **metal**
(Gold / Silver); Other-type items with no metal simply show no color word.

## 4. Pricing across the three methods

Grouping is **pricing-method-agnostic** because prices live on the individual items, not on
the group. The only thing that adapts is the **card header price**:

- **Per-item vendor method / fixed price:** every piece has the *same* price →
  header shows one number: `€90 · 🔗 12 in stock`.
- **Labor-per-gram:** price depends on weight, so pieces differ slightly →
  header shows a **range**: `€176–€182 · 🔗 3 in stock`.

Rule: **show a single price when all in-stock members are equal, a range (`min–max`) when
they differ.** No averaging, no fudging. The expanded view always lists each piece with its
own weight *and* its own price.

## 5. The grouped Stock card (Phase 1 core)

### 5.1 Collapsed card

Rendered once per group, in place of the N individual cards. Shows:

- Representative photo + description + type/metal.
- Reference code of the representative (expanded view shows each piece's own code).
- Price (single or range per §4).
- **Count badge:** `🔗 N in stock` (N = number of **in-stock** members).
- Gain line (owner/manager only), computed on the representative or as a range, consistent
  with §4.

### 5.2 Expanded view (tap to open)

Lists every in-stock piece in the group:

- Weight, individual price, individual ref code, individual QR.
- Each row opens the normal single-item edit (`invEdit`) so nothing is lost.
- Actions available here: print one tag, edit one piece, sell one piece.

### 5.3 Grouping is display-time

`renderInvList` groups the sorted rows by `groupId` **after** sorting/filtering, emitting one
collapsed card per group and normal cards for ungrouped items. Windowing (v10.23,
`_invSortedRows` / `_invRenderNextPage`) counts a collapsed group as **one row** for paging.
Sort/filter still operate on the representative (e.g. gain filter uses the representative's
gain; a group matches a gender filter if the representative does).

## 6. Creation & restock flows

**Grouping is always an explicit owner action — the app never auto-detects that two items
are the same design.** There are two entry points: mark a batch **at add-time** (you know
up front), or **"Group these"** afterwards (you found out later). Both end at one grouped
card.

### 6.1 New batch — "Add similar items" (the real convenience win)

A batch-add mode driven off the normal label/calc flow:

Triggered by a **"This is part of a batch of similar items"** option shown in the normal
add/label flow. Once ticked:

1. Enter the shared attributes **once**: type, metal, karat, gold color, vendor + labor
   category (or fixed price), retail, cost, gender/alsoGenders, description, photo.
   (Phase 2: also set a **color** per piece as you go, for mixed-color batches.)
2. Then repeat, fast: **weigh → tap → weigh → tap …** Each tap:
   - Mints one `inv_...` record with the shared attributes + this piece's weight.
   - Stamps the shared `groupId`.
   - Prints that piece's tag (same ref code, unique QR).
3. Finish → one grouped card with N pieces.

This is what turns "12 tedious entries" into a few minutes of weigh-tap.

### 6.2 Restock into an existing group ("＋ Add more")

On the group card: **＋ Add more** → same weigh-tap-weigh-tap loop, each new piece stamped
with the **existing** `groupId` and printing a tag. Example: 12 → sell 6 → add 8 → card
shows **14**. (Sold pieces stay as their own sale records; the count is in-stock members
only.)

**Inherited vs. entered.** ＋ Add more copies **all shared attributes** from the group so
they are NOT re-entered: type, metal, karat, gold color, **stones**, vendor + labor category
(or fixed price), retail, cost, description, gender/alsoGenders, and photo. The owner only
provides:
- **Weight** — the per-piece value (drives price on a per-gram group; no effect on
  fixed/per-item, where price stays identical).
- **Color** — pre-filled with the group's color; untouched for a single-color batch, flipped
  per piece for a mixed group.

So the interaction is **weigh → tap** (single color) or **weigh → set color → tap** (mixed).
Photo is inherited; swap it only when a color needs its own showroom photo (§8). The same
"inherited vs. entered" rule applies to the initial "Add similar items" batch (§6.1) after
the first piece establishes the shared attributes.

### 6.3 Group existing inventory items ("Group these")

For pieces already entered separately (e.g. the 3 identical pendants), **including the
common case of adding one today and discovering another days later** — just add the later
one normally, then group them:

- Multi-select their cards (existing Stock select-mode) → **"Group these"** → assign a
  shared new `groupId`.
- **No reprinting, no code changes required** — each keeps its own tag/ref code; scanning any
  of them still resolves; the collapsed card shows the representative's code.
- Reprinting to unify the printed code is **optional**, offered but not forced.

**Colors and grouping order are independent.** Setting a piece's color can happen **before or
after** grouping — order doesn't matter. To save opening three separate edits, "Group these"
should let the owner **assign each piece's color inline** in the grouping step. Plain pieces
stay **blank** (labeled by metal). *Real example:* the 3 cross pendants already in stock —
CR0245 (plain → blank), CR0244 (→ blue), CR0243 (→ green) — different ref codes, grouped by
`groupId` with no reprint; the description's hand-typed "Blue enamel / Green enamel" text can
then be cleared since the color field replaces it.

### 6.4 Adding one more to an existing group (the "third piece later" case)

Once a group exists, a new piece can join it **three ways** — all end with the group's count
+1, no limit on group size:

1. **At tag time:** Calc → enter values → Tag screen shows an **"Add to existing group?"**
   selector → pick the group. Best when you're pricing it anyway.
2. **"＋ Add more" on the group card:** open the group, weigh-tap the new piece straight in
   (this is §6.2 restock). Fastest when the group is in front of you.
3. **"Group these" again:** add it as a normal single item, then multi-select it **plus the
   group card** → Group these folds it in.

## 7. Selling

### 7.1 Sell one piece

Scan the piece's unique QR (or open the expanded view and sell that row) → resolves to that
exact necklace with its exact weight/cost/price → normal single-item sale. Count drops by 1.
**No change to the sale flow.**

### 7.2 Sell several identical at once (customer buys 3)

The group card gets a **quantity picker**: choose 3 → the app pulls 3 in-stock members into
the cart → checkout tags them with a shared `saleGroup` → the Sales list collapses them into
one **"🛍 3 items"** row (existing v10.0 behaviour). Stock drops by 3, profit exact per piece.
For a flat-priced group it doesn't matter which 3 are chosen (choose lowest-weight first, or
arbitrary); for per-gram, each carries its own price so the total is exact regardless.

### 7.3 How a sold piece appears in Sales

Because members are separate records, selling one produces an **ordinary single sale row**
(its own ref code, weight, price, profit) — identical to any single-item sale today. Three
sold together collapse via `saleGroup` as above. Nothing group-specific is needed in the
sold card.

### 7.4 Group reaches 0

When the last in-stock member sells, the collapsed card **vanishes** from Stock, exactly
like a sold single item. The sold pieces remain in Sales. (Optional future nicety: a
"sold out — reorder?" filter; not required for launch.)

### 7.5 Delete a sale (restore)

Unchanged. Deleting a member's sale restores that one piece to in-stock (existing logic),
so the group's count goes back up by 1.

## 8. Color grouping (Phase 2 — builds on the Phase 1 `color` field)

The pink-cross vs blue-cross baby bracelets are the **same design, different color**: grouped
for the owner, but **both colors must show in the showroom**. This reuses the `color` field
that Phase 1 already added to every item (§3.2):

- The `color` field is set per piece (e.g. `pink` / `blue`). Values ideally chosen from a
  small pick-list for clean showroom labels.
- **App (owner) card:** still **one card** for the design, with a breakdown —
  `Baby ID bracelet · 🔗 5 in stock (3 pink · 2 blue)`.
- **Showroom:** groups by **`groupId` + `color`**, so it shows **one public card per color**,
  each with its own photo and its own count. This is the "both show up" requirement.
- **Photos per color:** the first piece of each color supplies that color's showroom photo —
  you don't photograph every physical piece, just one per color.

**A group can hold any number of color buckets, including a "plain" one.** The color field
doesn't care how many distinct values a group contains, and "plain / no enamel" is just
another color value.

*Worked example (real case):* a design comes in as **4 pieces — 2 plain gold, 1 green,
1 blue**. Three color buckets:
- **App card:** one card — `… · 🔗 4 in stock (2 gold · 1 green · 1 blue)`.
- **Showroom:** three cards — Gold (2 in stock), Green (1), Blue (1) — each with its own
  photo, each counting down independently.
- **Photos:** three (plain, green, blue); a bucket with >1 piece (the 2 plain) shares one.

**Labeling "plain":** leave the color **blank** for plain / no-enamel pieces. The card and
showroom already display the **metal** (Gold / Silver), so a blank-color bucket is simply
**labeled by its metal** — no separate "Plain" word needed. A color value is labeled by that
color. So: blank → "Gold"/"Silver"; `green` → "Green"; etc.

Grouping by `groupId` (not ref code) is what lets one group span multiple showroom cards.

## 9. Showroom (public site)

- Publish **all** in-stock members (as today), plus `groupId` and `color` in
  `PUBLIC_SAFE_FIELDS`.
- `showroom.htm` **groups the public mirror** by `groupId` (Phase 2: `groupId + color`) and
  renders **one card per group/color** with a **live "N in stock"** count = number of
  in-stock members in that bucket.
- Count decrements automatically as pieces sell (the mirror already re-syncs on every sale).
- No prices are shown publicly, so choosing a representative photo/description per bucket is
  the only decision, and it's trivial (first in-stock member).
- `item.htm` (single `?qr=` link) is unchanged — a QR always points to one specific piece.

## 10. Edge cases & rules

- **Legacy items (no `groupId`)** render exactly as today. Feature is additive.
- **Windowing:** a collapsed group counts as one row for `INV_PAGE_SIZE` paging; "jump to
  card" (`_invEnsureRendered`) must page to a group whose member is the target.
- **Select-all / bulk actions:** operate on groups sensibly (selecting a collapsed card
  selects the group; define whether bulk delete removes all members — recommend: confirm
  "delete all N pieces in this group?").
- **Gain sort/filter (owner):** use the representative's gain; note a group can hide
  per-piece gain spread — acceptable, expanded view shows each.
- **Reference-code freeze (v10.20):** unchanged per piece. A group's collapsed card shows the
  representative's frozen ref code via `itemRefCode`.
- **Bulk-chain items (`isBulk`)** are a separate concept (sold by the gram) and are **not**
  grouped by this feature.

## 11. Bilingual strings (must add both EL + EN)

New user-visible text needs entries in both language dictionaries, following the existing
`data-i18n` pattern. At minimum:
- "N in stock" / count badge, "Add similar items", "＋ Add more", "Group these",
  "Sell quantity", "Add to existing group?", the **Color** field label + color values,
  and per-color breakdown labels.

## 12. Versioning & docs (per CLAUDE.md)

- Bump `APP_VERSION` and add a new top entry to **both** `ABOUT.el.changelog` and
  `ABOUT.en.changelog`; delete the oldest entry from each to keep 10.
- Update the "Key features" section of `CLAUDE.md` in the **same commit** as the code.
- Showroom-only changes do **not** bump `APP_VERSION`; app changes do.

## 13. Suggested build order

**Phase 1 — identical grouping (most of the daily value, low risk):**
0. ✅ **DONE (v10.27)** — `color` field on all three item forms (single items too) + badge +
   filter + `PUBLIC_SAFE_FIELDS`. Independent of grouping; shipped first on its own.
1. ✅ **DONE (v10.28)** — `groupId` field + `PUBLIC_SAFE_FIELDS` addition.
2. ✅ **DONE (v10.28)** — Display-time grouping in `renderInvList` → collapsed card (count +
   price/range + colour breakdown) + group detail sheet (`openInvGroup`). Render-only.
3. ✅ **DONE (v10.28)** — "Group these" for existing items (🔗 in select bar, retro-group,
   no reprint) + Ungroup (`invUngroup`).
4. ✅ **DONE (v10.29)** — "Add similar items" weigh-tap create (`invAddSimilar`, shared ref
   code + groupId, per-piece unique QR + tag print). Entry point: "🔗 Add similar items" in
   the Edit Item form (starts a batch from the first piece).
5. ✅ **DONE (v10.29)** — "＋ Add more" restock into a group (same `invAddSimilar`, from the
   group detail sheet).
6. ✅ **DONE (v10.30)** — "Sell several" quantity picker in the group sheet
   (`invSellFromGroup`, routes through `openMultiSaleModal` → shared `saleGroup`).
7. ⏳ TODO — Showroom: group by `groupId`, show live "N in stock".

**Phase 2 — color variants (builds on the Phase 1 `color` field):**
8. Per-color breakdown on the grouped app card ("N in stock — 1 pink · 1 blue").
9. Showroom split by `groupId + color`, per-color photo + count.

Phase 1 fully covers the necklaces and the 3 pendants. Phase 2 covers the pink/blue
bracelets once Phase 1 is proven in the store.

## 14. Open decisions (confirm before/while building)

- Bulk-delete of a group: delete all members with one confirm? (Recommended: yes, explicit
  confirm showing the count.)
- "Sold out — reorder?" visibility after a group hits 0: hide (like sold) for launch;
  optional filter later. (Current decision: **hide**.)
- Variant vocabulary for colors: free text vs a fixed pick-list (**decided: fixed
  bilingual pick-list**, see Appendix A).

## Appendix A — Color pick-list (bilingual)

**Storage rule:** store a **language-neutral key** (e.g. `light_blue`) on the item, render
the EL/EN label from a `COLORS` dictionary — same i18n pattern as the rest of the app. This
keeps color grouping/labels language-independent (switching the UI language flips every color
label; showroom grouping-by-color is unaffected). **Blank = plain**, labeled by the item's
metal (Gold/Silver), so it is intentionally the first/default option.

| Key         | English     | Ελληνικά        |
|-------------|-------------|-----------------|
| *(blank)*   | *(plain — shows metal)* | *(απλό — δείχνει μέταλλο)* |
| white       | White       | Λευκό           |
| black       | Black       | Μαύρο           |
| red         | Red         | Κόκκινο         |
| bordeaux    | Bordeaux    | Μπορντό         |
| pink        | Pink        | Ροζ             |
| fuchsia     | Fuchsia     | Φούξια          |
| light_blue  | Light Blue  | Γαλάζιο         |
| blue        | Blue        | Μπλε            |
| navy        | Navy        | Σκούρο Μπλε     |
| turquoise   | Turquoise   | Τιρκουάζ        |
| green       | Green       | Πράσινο         |
| light_green | Light Green | Ανοιχτό Πράσινο |
| yellow      | Yellow      | Κίτρινο         |
| orange      | Orange      | Πορτοκαλί       |
| purple      | Purple      | Μωβ             |
| lilac       | Lilac       | Λιλά            |
| brown       | Brown       | Καφέ            |
| grey        | Grey        | Γκρι            |
| silver      | Silver      | Ασημί           |
| multicolor  | Multicolor  | Πολύχρωμο       |

*This list is the starting set (common enamel colors + owner-named light blue / blue /
bordeaux); confirm/extend before building the pick-list. Adding a color later = one row in
the `COLORS` dictionary, no data migration.*
