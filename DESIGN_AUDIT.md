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

### Proposed standard (apply to every modal)
1. **Close ✕ — always top-right**, one component: 34px circle, same glyph (`✕`),
   same muted style, everywhere. (Big enough to tap, always in the same spot.)
2. **Primary / safe action — bottom, full-width, gold or green.** Same spot every time.
3. **Destructive action (delete / cancel) — visually separated and de-emphasized:**
   outlined/text style (not a big filled block), set apart from the primary CTA
   (its own row with a gap, or smaller), and always behind a `showConfirm`. Never the
   same size/position as the primary tap target.
4. On detail sheets that both *complete* and *cancel* (reservations, repairs), keep
   **Complete/primary prominent, Cancel small and low-contrast** so the finger goes to
   the right one by habit.

Rollout: introduce one shared modal header (close ✕) + footer (primary + de-emphasized
destructive) and migrate modals to it a few at a time.

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
