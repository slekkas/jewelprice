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
| `variant`      | string (optional) | Variant attribute, Phase 2 — currently **color** (e.g. `pink`).|
| `seqId`        | existing          | Per-piece sequence. Pieces in a batch may share one, or not.  |
| `weight`       | existing          | Per-piece, exact. Drives QR and gain.                         |
| `qrCode`       | existing          | Per-piece, **unique** (encodes weight → naturally differs).   |

Nothing about the sale record, QR format, or `PUBLIC_SAFE_FIELDS` needs to change for
Phase 1 **except** adding `groupId` (and later `variant`) to `PUBLIC_SAFE_FIELDS` so the
showroom can group by it.

### 3.3 The canonical / representative piece

For any display that needs "the group as one thing" (card header, showroom card) we pick a
**representative**: the first in-stock member by `dateAdded` (stable, deterministic). It
supplies the representative photo, description, type, metal, and the ref code shown on the
collapsed card.

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

### 6.3 Group existing inventory items ("Group these")

For pieces already entered separately (e.g. the 3 identical pendants), **including the
common case of adding one today and discovering another days later** — just add the later
one normally, then group them:

- Multi-select their cards (existing Stock select-mode) → **"Group these"** → assign a
  shared new `groupId`.
- **No reprinting, no code changes required** — each keeps its own tag/ref code; scanning any
  of them still resolves; the collapsed card shows the representative's code.
- Reprinting to unify the printed code is **optional**, offered but not forced.

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

## 8. Color / variant support (Phase 2)

The pink-cross vs blue-cross baby bracelets are the **same design, different color**: grouped
for the owner, but **both colors must show in the showroom**. Modeled as a **variant**:

- Optional per-piece **`variant`** field (Phase 2 = color, e.g. `pink` / `blue`). Values are
  free but ideally chosen from a small list for clean showroom labels.
- **App (owner) card:** still **one card** for the design, with a breakdown —
  `Baby ID bracelet · 🔗 5 in stock (3 pink · 2 blue)`.
- **Showroom:** groups by **`groupId` + `variant`**, so it shows **one public card per color**,
  each with its own photo and its own count. This is the "both show up" requirement.
- **Photos per color:** the first piece of each color supplies that color's showroom photo —
  you don't photograph every physical piece, just one per color.

Grouping by `groupId` (not ref code) is what lets one group span multiple showroom cards.

## 9. Showroom (public site)

- Publish **all** in-stock members (as today), plus `groupId` (and Phase 2 `variant`) in
  `PUBLIC_SAFE_FIELDS`.
- `showroom.htm` **groups the public mirror** by `groupId` (Phase 2: `groupId + variant`) and
  renders **one card per group/variant** with a **live "N in stock"** count = number of
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
  "Sell quantity", per-color breakdown labels, and (Phase 2) color/variant labels.

## 12. Versioning & docs (per CLAUDE.md)

- Bump `APP_VERSION` and add a new top entry to **both** `ABOUT.el.changelog` and
  `ABOUT.en.changelog`; delete the oldest entry from each to keep 10.
- Update the "Key features" section of `CLAUDE.md` in the **same commit** as the code.
- Showroom-only changes do **not** bump `APP_VERSION`; app changes do.

## 13. Suggested build order

**Phase 1 — identical grouping (most of the daily value, low risk):**
1. `groupId` field + `PUBLIC_SAFE_FIELDS` addition.
2. Display-time grouping in `renderInvList` → collapsed card (count + price/range) +
   expanded view. (Read/render only; no data-write behaviour change.)
3. "Group these" for existing items (retro-group, no reprint).
4. "Add similar items" batch-create (weigh-tap loop, shared groupId, per-piece tags).
5. "＋ Add more" restock into a group.
6. "Sell quantity" from the group card (reuses cart + `saleGroup`).
7. Showroom: group by `groupId`, show live "N in stock".

**Phase 2 — color variants:**
8. `variant` field + per-color breakdown on the app card.
9. Showroom split by `groupId + variant`, per-color photo + count.

Phase 1 fully covers the necklaces and the 3 pendants. Phase 2 covers the pink/blue
bracelets once Phase 1 is proven in the store.

## 14. Open decisions (confirm before/while building)

- Bulk-delete of a group: delete all members with one confirm? (Recommended: yes, explicit
  confirm showing the count.)
- "Sold out — reorder?" visibility after a group hits 0: hide (like sold) for launch;
  optional filter later. (Current decision: **hide**.)
- Variant vocabulary for colors: free text vs a fixed pick-list (recommended: small
  pick-list for clean showroom labels).
