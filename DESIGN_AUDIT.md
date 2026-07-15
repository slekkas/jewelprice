# JewelPrice Pro — Design & UX Audit (kick-off)

Read-only audit. Nothing is built until Sotiris approves each item.
Priorities set by Sotiris: (1) destructive/close-button safety, (2) consistency
so daily employees build muscle memory, (3) a light theme if it's cheap.

---

## A. Destructive-action safety — confirm before any data loss

Scanned every delete / cancel / ungroup / clear / reset function and its callers.

### ✅ Already safe (show a confirm dialog first)
Inventory: delete item, delete group, ungroup. Sales: delete sale, delete whole
sale (group). Clients: delete client. Vendors: delete vendor. Team: delete user.
Reservations: cancel, delete. Repairs: cancel, delete. Coins: delete custom coin.
Settings: delete labor row, clear quote history, reset credentials. Stocktake: cancel.
→ **All the high-stakes daily actions (sales, stock, clients, reservations, repairs)
are protected.**

### ✅ DONE (v11.03) — added `showConfirm` guards
| # | Action | Function | Status |
|---|--------|----------|--------|
| 1 | Delete a saved quote | `deleteQuote` | ✅ confirms |
| 2 | Clear the whole stone-price cache | `clearStonePriceCache` (button) | ✅ confirms |
| 3 | Reset an item's initial-price baseline | `invEditResetInitialPrice` | ✅ confirms |
| 4 | Clear all notifications | `notifClearAll` | ✅ confirms |
| 5 | Delete a label-history entry | `deleteLabelHistoryEntry` | ✅ confirms |

**Deliberately left friction-free:** single **stone-cache price** deletes
(`deleteCachedStonePrice`, and the per-row delete in the settings cache manager).
These are trivial, re-creatable cache values tapped mid-estimate; a confirm on each
would slow the daily pricing flow for no real safety gain. (Change if you disagree.)

---

## B. Consistent Close (✕) & destructive-button placement — muscle memory

### Findings (data-backed)
- **Close glyph is inconsistent:** 21 modals use `✕`, 10 use `×` (`&times;`).
- **Close button has ≥3 style variants** (different size/shape circles).
- **8 destructive actions are full-width bottom buttons** (e.g. "Cancel reservation")
  — big, in the easy-to-tap zone, sometimes right under the primary green CTA.
  This is exactly the accidental-press risk Sotiris flagged.

### ✅ DONE (v11.04) — two shared classes, applied across the modals
- **`.modal-x`** — the Close ✕: one 34px circle, glyph normalized to `✕` (was ✕/×/&times;),
  same muted style, sits top-right in the header. Applied to the reservation/repair/group
  detail sheets, add-vendor, cart, quote-history, and every circular header close (12 modals).
- **`.modal-danger`** — destructive action (Cancel/Delete): outlined red, never a filled
  block, consistent gap above, and every one is now behind a `showConfirm` (from item A).
  Applied to reservation Cancel/Delete, repair Cancel/Delete, group Delete-all.
- Glyph fully normalized: 0 `&times;` buttons remain.

**Deliberately left non-standard (context demands it):** the small `top:6px;right:6px`
popover closes (a 34px circle would dwarf them), the fullscreen photo/price overlay's big
white ✕, and the panel "Exit" button. Forcing the header style there would look worse.

The rule going forward for any NEW modal: **Close ✕ = `class="modal-x"` top-right; primary
action bottom; destructive = `class="modal-danger"` + a `showConfirm`.**

---

## C. Light theme in Settings — FEASIBLE and cheap ✅

- The palette is **centralized in `:root` CSS variables** (`--dark`, `--dark-2`,
  `--card`, `--text`, `--text-muted`, `--gold`, …), and ~238 inline styles already use
  `var(--dark…)`. So a light theme is mostly an **alternate variable block**:
  `:root[data-theme="light"] { --dark:…; --text:…; … }` + a toggle. **~1–2 KB, no
  meaningful size increase** — meets Sotiris's "only if it's just colors" condition.
- **Caveats to handle for readability:**
  - ~10 hardcoded dark hex colors in inline styles won't flip → convert those to
    variables (small list, e.g. `#1a1408`, `#1a1710`, `#0a0800`).
  - White backgrounds (`#fff`/`#fafafa`) are for print/PDF and product photos → keep white.
  - Check contrast (WCAG AA) so gold-on-cream text stays readable.
- **Palette:** elegant **ivory/cream background + charcoal text + gold accents**,
  matching the printed quote/reservation forms (clean, professional). Can align to the
  showroom's look if preferred.
- **Mechanics:** Settings toggle Light/Dark → `localStorage 'jpro_theme'` → set
  `data-theme` on `<html>` at boot (before first paint to avoid a flash).

### ✅ DONE (v11.05)
- Settings → **Appearance** toggle (🌙 Dark / ☀️ Light), persisted, no-flash boot script.
- `:root[data-theme="light"]` overrides all colour variables (ivory bg / charcoal text /
  deep gold). The 216 translucent-white surfaces were routed through `--waNN` variables
  that flip white→warm-dark; **dark theme values are byte-identical, so the dark look is
  unchanged.** New `--ink` constant = text-on-gold that stays dark in both themes.
- Bottom nav + price bar get a light skin so their text stays readable.
- Kept intentionally dark in both themes (elegant / floating / transient): the fullscreen
  customer price display, the floating select bar, the update banner, toasts.
- **Verified:** contrast scan across all panels in light mode = **0 low-contrast elements**;
  dark unchanged; toggle persists; no console errors. **Real size added: ~1.9 KB** (colours only).

---
**All of A, B, C are done.** Remaining: D (broader aesthetic/productivity) — ongoing, tackle
as separate small passes when you want.

---

## D. Broader aesthetic / productivity (initial — expand later)
- Keep extending the "one pattern everywhere" work (cash toggle already unified) to
  every repeated control (pickers, date fields, phone fields).
- Card typography/badge spacing consistency between single / grouped / coin / metal cards.
- Reduce taps in the busiest flows (add item, sell, quote) — measure taps-to-done.
- (Fuller pass to follow once A–C direction is set.)

---

## Suggested order
1. **A — destructive-safety confirms** (quick, pure safety win).
2. **B — close/destructive placement standard** (muscle memory; biggest daily payoff).
3. **C — light theme** (self-contained, cheap).
4. **D — broader polish** (ongoing).
