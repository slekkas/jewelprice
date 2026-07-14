# Next features — design plan (Repairs, Reservations/Layaway, Stale Stock)

Planned overnight from Sotiris's notes. Implementation starts next session.
Nothing here is built yet. Build order suggestion at the bottom.

---

## Shared foundations (build once, reuse across all three)

- **Branded form PDF** — factor the header of `buildSharePDF` (LEKKAS JEWELRY logo,
  KARPATHOS subtitle, address, phone, date, maps-QR, gold line, Noto Sans Greek
  fonts) into a reusable `drawFormHeader(doc)` so the repair claim form and the
  reservation form share the exact same professional header. Body drawn per form.
  All forms include a **signature line** and an empty **stamp box** (Sotiris
  explicitly wants a stamped, signed, professional paper).
- **Photo capture** — reuse `renderPhotoPicker(container,{previewId,existingUrl,onCapture})`
  + `uploadInventoryPhoto(small,hi,name)` (dual-res ImgBB). Repairs need the
  customer's-item photo; reservations reuse the inventory photos already on file.
- **Sequential numbers** — one counter per type, same REST pattern as
  `getNextSeqId`: `/jewelprice_repair_counter`, `/jewelprice_reservation_counter`.
  Format e.g. `R-0042` / `H-0042` (H = "hold"). Simple increment (no gap-recycling
  needed here).
- **Customer picker** — a small reusable "pick from Clients or type a name" control
  backed by `getClients()`; free-text fallback for walk-ins. **Add PHONE to the
  client model** (currently name + email) — used by reservations, repairs, and the
  "notify when ready" action below. Add phone to the client add/edit form and to
  the customer capture in repairs/reservations.
- **Every form: Print AND Share.** All PDFs (repair/custom claim, reservation/
  layaway form, receipts) get both a **Print** button and a **Share** button
  (Web Share API — send the PDF to WhatsApp/Viber/email, save paper). Caveat: the
  physical **stamp + signature** only exist on a printed copy, so forms that need a
  signature (custom-order deposit, reservation) are printed for signing but can
  ALSO be shared as the customer's digital record. Reuse the existing
  `buildSharePDF` share path.
- **Notify customer when ready** — leverage the new phone number. A "Notify"
  action on a repair/custom order opens a **pre-filled WhatsApp message**
  (`https://wa.me/<phone>?text=<bilingual msg>`), with SMS (`sms:`) and email
  (mailto/EmailJS) fallbacks. Owner taps send — no API, no cost, no consent
  headaches. Message e.g. "Your repair at Lekkas Jewelry is ready for pickup 💍".
- **Private Firebase nodes** — `/jewelprice_repairs`, `/jewelprice_reservations`.
  NEVER published to the public showroom (contain customer names/phones/photos).
  Not added to `PUBLIC_SAFE_FIELDS`.
- **Bilingual** — every screen + printed form in EL/EN, following the existing
  i18n pattern.
- **Nav (RECOMMENDED) — 5 everyday tabs + a top "More" menu.** The bottom nav is
  reserved for the daily counter work; rarely-used admin moves to an overflow menu
  (☰/⋯) in the top-right header (next to the language flags). This makes room for
  the new features without clutter and goes one step past just hiding Sync/
  Settings/Info:
  - **Bottom nav (5, everyday):** Calc · Stock · Sales · Clients · **Desk**
  - **Desk** = a single tab with a segmented toggle **Repairs | Reservations**
    (both are "customer commitments" — keeps them first-class without adding two
    tabs).
  - **Top "More" menu (rare/admin):** Vendors · Sync · Settings · Info — all
    role-gated as today (employees see little/none here).
  - Stale Stock stays inside the Stock tab (sort + filter + report), no new tab.
  → *confirm structure + the "Desk" name (EN: Desk/Jobs · EL: Εργασίες/Πάγκος).*
- **Roles** — employees work the counter, so they can CREATE repair tickets and
  reservations and record payments (same as they already record sales). Owner/
  manager see everything and can delete/cancel. → *confirm.*

---

## Feature A — Stale Stock lens (discount-focused, NOT melt)

**Insight from Sotiris:** every jewel has its customer; a piece can sit 10 years
then sell for far more than its original price (e.g. bought €125, first retail
€300, sold €1300 years later as gold rose). So stale ≠ melt. The play is
**spotting long-sitting stock and knowing the negotiation room to discount it**.
Melting is rare (only genuinely ugly pieces) — keep it a quiet secondary option.

**What we can compute from existing data** (no schema change):
- **Time in stock** — from `dateAdded` (fallback to the `inv_<ms>` id via the
  existing `_invItemDate` helper). Express in seasons/years (store operates
  Apr 1–Nov 30).
- **Current retail** — `calcInvItemPrice(item).retail`.
- **Today's metal value (melt floor)** — `calcInvItemPrice(item).metalValue`. The
  natural "won't go below this without melting" floor.
- **Negotiation room** — `retail − metalValue` = the markup a customer can haggle
  into. This is the actionable number ("you've got €150 to play with").
- **Gold gain since added** — already computed (`initialRetail` vs live retail).

**Delivery (three parts, all read-only, owner/manager only):**
1. **Sort → "Oldest in stock"** in the Stock sort menu (dateAdded ascending).
   Trivial, immediately useful.
2. **Filter → "Time in stock"** pills: `1+ season / 2+ / 3+ / 5+` (owner/manager).
3. **"🕰️ Stale stock" report** (button in Stock, owner-only) — the hero. Lists
   in-stock, non-coin/metal pieces older than a chosen threshold (default 2
   seasons), oldest first. Each row: photo, ref code, description, **time in
   stock** ("here since 2019 · 6 seasons"), **retail**, **metal-value floor**,
   **negotiation room €N**, **gold gain %**. Per-row quick actions: **Open**
   (`invEdit` to set a discount price), **Share** (send to a customer / feature),
   **Show in showroom** (clear `hideFromShowroom` to surface it online), and
   **Melt** — discounting is the main objective but **melting is always a valid
   option** (Sotiris), so show the melt value clearly and offer a one-tap path to
   the existing Metal/scrap flow for pieces he does decide to melt.
   Exportable via the existing CSV engine.

**Notes / open points:**
- We do NOT reliably have the *historical* purchase price for old items
  (`purchaseCost` is often blank; `baseCost` uses today's gold), so the report is
  honest about "today's metal floor + negotiation room" rather than inventing a
  historical profit number. `initialRetail` (display price when added) IS stored
  and gives the appreciation story.
- Smallest of the three features → good first build.

---

## Feature B — Repairs & Custom Orders

**From Sotiris:** happens a lot; wants to open a ticket, be prompted to photograph
the customer's item, write a description, and print a claim form. Two distinct
types with different rules:
- **Repair** — **NO deposit** (the "deposit" is the jewel they leave), but an
  **approximate estimate IS required** (Sotiris: the customer needs to know the
  rough cost). Single value or a small range (e.g. €20–50).
- **Custom order** — a **deposit is MANDATORY** ("we don't want people ordering
  custom things and disappearing"), plus the estimate. Photo can be a reference/
  sketch of what to make.

**Data model** — `/jewelprice_repairs/<id>`:
```
{
  id: 'rep_<ms>',
  ticketNo: 42,                 // → "R-0042"
  type: 'repair' | 'custom',
  clientName, clientPhone,      // phone now captured (for the "notify" action)
  clientId: <optional link>,
  photoUrl, photoUrlHi,         // the customer's item / the reference for a custom piece
  description,
  estLow, estHigh,              // estimate — REQUIRED for repair (single→low==high) and custom
  deposit: number|null,         // REQUIRED for custom, absent for repair
  depositMethod: 'cash'|'card',
  dateIn: ms,
  datePromised: ms|null,        // optional "ready by"
  status: 'in_progress' | 'ready' | 'collected',
  notifiedAt: ms|null,          // when the customer was messaged
  finalPrice: number|null,      // optional, entered at collection
  createdBy, notes,
  dateReady: ms|null, dateCollected: ms|null
}
```

**Flows:**
1. **New ticket** (Desk → Repairs → "＋ New"): type (Repair/Custom), customer
   (pick/typed, incl. **phone**), photo, description, **estimate** (required),
   **deposit** (required if Custom), optional ready-by date → save (assigns
   `ticketNo`).
2. **Print / Share claim form** (A4 branded PDF via `drawFormHeader`): ticket no +
   date, customer name/phone, **the item photo**, description, estimate, deposit
   received (custom), ready-by date, short terms, **signature line + stamp box**.
   Both **Print** (the signed/stamped copy) and **Share** (customer's digital copy).
   *Optional:* a small QR encoding `ticketNo` so scanning the slip opens the ticket
   on collection (nice, not required for v1).
3. **Repairs list**: open tickets first (in-progress / ready), searchable by
   name / ticket no; photo thumbnail, customer, description, status chip. Tap →
   detail.
4. **Status**: In progress → **Ready** → **Collected**.
   - On **Ready**, a **"Notify customer"** button (WhatsApp/SMS/email pre-filled,
     bilingual) so the customer knows to come pick it up; records `notifiedAt`.
   - On **Collected**, optionally enter the final price; for custom orders the
     deposit is applied and the balance is what's collected. If a final price is
     entered it can be logged as income (a lightweight sale-type record) so it
     counts toward day revenue — default on for custom, optional for repairs.

**Reused pieces:** `renderPhotoPicker`/`uploadInventoryPhoto`, `drawFormHeader`
+ share path, client picker (with phone), `getNextSeqId`-style counter, WhatsApp
click-to-chat for the notify action.

**Open points:** claim form size (A4 vs half-page); ticket-number format
(`R-0042` vs year-based `R-26-0042`); include the lookup-QR in v1?; default
notify channel (WhatsApp proposed).

---

## Feature C — Reservations & Layaway (holds + installments)

**From Sotiris, two real workflows:**
1. **Reserve/hold** — specific in-stock pieces held for a customer (he currently
   has 2 chains + 2 crosses on hold). Often no deposit for trusted regulars, but
   frequently a deposit is taken. Wants to replace the little paper notes with a
   **printed, signed, professional form with a stamp box** listing all details.
2. **Layaway/installments** — customer wants a €1500 piece, leaves e.g. €200,
   comes back every few weeks with more until fully paid, then collects. Track
   balance + payment history. Very rarely (trusted customers only) he hands the
   piece over early and they pay it off slowly ("reverse layaway" — a receivable).

**Unified model** — a reservation holds one or more items, carries an agreed
total, and logs payments; layaway is just a reservation with payments over time.
`/jewelprice_reservations/<id>`:
```
{
  id: 'res_<ms>',
  reservationNo: 42,                    // → "H-0042"
  clientName, clientPhone, clientId?,
  itemIds: [inv ids],                   // the held pieces
  itemsSnapshot: [{ref, desc, qr, retail, photoUrl}],  // frozen for the form
  agreedTotal: number,                  // default Σ retails; EDITABLE (special/discount price)
  payments: [{amount, date, method:'cash'|'card', by, note}],  // deposit = first payment
  // balance is derived: agreedTotal − Σ payments
  status: 'active' | 'completed' | 'cancelled',
  itemGivenEarly: false,                // reverse layaway flag
  dateCreated, dateCompleted, createdBy, notes,
  holdUntil: ms|null                    // optional "held until" date on the form
}
```

**Inventory interaction:**
- On reserve, stamp each item with `reservationId` + show a **🔒 Reserved** badge
  on its Stock card; **exclude reserved items from the public showroom**
  (`publicProjection` returns null when `reservationId` set) and guard them from
  accidental sale (warn if someone tries to sell a reserved piece).
- On **complete** (fully paid, or Sotiris decides): create **sale record(s)** for
  the items (matching `confirmSale`'s shape so profit / leaderboard / day-revenue
  all capture it), dated at completion, priced at `agreedTotal` (split across
  items proportionally like the existing multi-item sale). Items → `status:'sold'`,
  reservation → `completed`. Clear `reservationId`.
- On **cancel**: clear `reservationId` on all items (back to normal in-stock),
  reservation → `cancelled`.
- **Reverse layaway**: `itemGivenEarly=true` — piece physically leaves but the
  reservation stays `active` tracking the outstanding balance as a receivable;
  revenue booked on final completion (or at handover — *confirm his preference*).

**Flows:**
1. **Create** — two entry points: (a) Stock select-mode → new **"🔒 Reserve"**
   action (alongside Group/Sell/Share) with the selected items pre-filled;
   (b) Desk → Reservations → "＋ New" then add items. Pick customer, confirm/edit
   agreed total (allow a special/discount price — ties into Stale Stock), optional
   initial deposit (amount + method) → save, mark items reserved.
2. **Print / Share reservation form** (A4 branded PDF): reservation no + date,
   customer name/phone, **item list with photo + ref + description**, agreed price,
   deposit/payments to date, **balance due**, optional "held until" date, short
   terms, **signature line + stamp box**. Print = the signed paper; Share = the
   customer's digital copy.
3. **Add payment** — open a reservation → "＋ Payment" (amount, date, method,
   note); balance updates. Optional printed payment receipt / updated form.
4. **Complete & collect** — when balance ≤ 0 (or manual): convert to sale(s) as
   above; print a final receipt.
5. **Reservations list** — active first, showing customer, items, **balance**,
   status; reserved badge visible in Stock too.
6. *(Phase 2)* client detail shows that client's reservations + repairs — a full
   relationship view.

**Reused pieces:** `drawFormHeader`, inventory photos/refs, `confirmSale` record
shape + sales push, client picker, counter.

**Open points:** reverse-layaway revenue timing (handover vs final payment);
who can complete a reservation (all roles vs owner/manager); exact terms wording
on the form; whether a reserved item still counts in Stock value totals (proposal:
yes, it's still owned stock — just flagged).

---

## Suggested build order & versioning

1. **Stale Stock lens** (smallest, pure read/analysis on existing data) — warm-up.
2. **Repairs & Custom Orders** (new data type + photo + one printed form).
3. **Reservations & Layaway** (largest — item flags + payments + printed form +
   sale conversion).

Each is independent and shippable on its own version bump; ship incrementally so
Sotiris can use each as it lands. Every step: bump `APP_VERSION`, update the inline
changelog (EL+EN), prepend to `changelog.htm`, and update `CLAUDE.md` in the same
commit (per repo rules). Push only when asked.

## Decisions — ALL SETTLED ✓
- Stale Stock: discount-first, melting always an available option. ✓
- Repairs need a required estimate; custom orders need a required deposit. ✓
- Add PHONE to the client model. ✓
- Notify the customer when a repair/order is ready (WhatsApp/SMS/email pre-filled;
  **default channel WhatsApp**). ✓
- Every PDF gets Print AND Share (stamp/signature only on the printed copy). ✓
- **Nav: 5-tab bottom** (Calc·Stock·Sales·Clients·**Desk**) **+ top "☰ More" menu**
  (Vendors·Sync·Settings·Info). Desk = combined **Repairs | Reservations** toggle. ✓
- **Reverse layaway (item handed over early): revenue books at FINAL payment**;
  outstanding shown as money owed until then. ✓

**Sensible defaults (locked unless Sotiris says otherwise while building):**
- Desk tab name: EN "Desk", EL "Εργασίες" (open to "Πάγκος").
- Claim/reservation forms: **A4** (consistent with quotes/catalog).
- Numbering: `R-0042` (repairs/custom), `H-0042` (holds/reservations) — simple.
- Lookup-QR on the printed slip: **skip in v1** (keep simple; easy to add later).
- Reserved items **still count** in Stock value totals (still owned stock, flagged).
- Roles: employees can create repairs/reservations, record payments, notify, and
  complete (they already record sales); **cancel/delete = owner/manager only**.

## Build order (each ships independently, its own version bump)
1. **Stale Stock** — lives inside Stock, no nav change. Warm-up, immediate value.
2. **Nav restructure** — 5-tab bottom + top "☰ More" menu + empty **Desk** shell.
3. **Repairs & Custom Orders** — + phone on the client model + notify action.
4. **Reservations & Layaway** — into Desk; completion → sale conversion.
