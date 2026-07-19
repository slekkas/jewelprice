# Returns, Refunds, Store Credit & Exchange — design

**Status:** built in v11.46.
**Trigger:** a real case. A mother bought her daughter a €1500 necklace. The daughter came back
days later wanting something else, found nothing she liked, and left with €1500 in credit. The
app had no way to record any of that — the only tool was "Restore to Stock (mistake / return)",
which flips the piece back to stock and **deletes** the sale record.

That old path is right for a *mistake* (a sale that should never have been recorded) and wrong
for a *return* (a real sale that really happened and was later reversed). This document designs
the return.

---

## 1. The three things, kept separate

The request bundles three ideas that must not be conflated:

| | What it is | Where it lives |
|---|---|---|
| **Return** | The piece comes back; the sale is reversed in the books | A new negative sale record |
| **Refund method** | How the money goes back: cash, card, or store credit | A field on that record |
| **Exchange** | Return one piece, take another | Return-to-credit **then** an ordinary sale paid with that credit |

The key realisation: **an exchange is not a new mechanism.** It is a return whose refund method is
store credit, immediately followed by a normal sale that spends the credit. Building "exchange"
as its own transaction type would duplicate the sale flow (VAT, profit, bonus, stock, multi-item,
grouping) for no gain. So the app builds **return** and **store credit**, and exchange falls out.

---

## 2. Money model — the load-bearing decision

### 2.1 A return is booked on the day it happens, not backdated

Three options were considered:

- **Delete the sale.** Rejected. It destroys history — the owner could no longer see that a
  €1500 sale happened at all, and any report already printed would silently disagree with the app.
- **Flag the original and exclude it from totals.** Rejected. June's revenue would drop in July.
  Past months must not move.
- **Write a negative record dated the return date.** ✅ Chosen. This is what every real POS does
  (a credit note). June keeps its €1500, July carries −€1500, the year nets to zero.

Worked through the real case:

```
June   sale     +1500   cost +900   profit +600     (necklace)
July   return   −1500   cost −900   profit −600     (credit note G-0012 issued, €1500)
Aug    sale     +1200   cost +700   profit +500     (ring, paid with the credit)
─────────────────────────────────────────────────────────────────────────
Year            +1200               +500            ✅ correct
Outstanding credit after Aug: €300
```

Nothing is double-counted: the €1500 is booked, un-booked, and the €1200 re-booked once.

### 2.2 The return mirrors the original **exactly** — it never recomputes

A return must reverse what the sale *actually recorded*, not what the same sale would record
today. VAT changed from 24% to 17% (v11.40); gold moves daily; a vendor rate may have been
corrected. Recomputing would leave a residue that silently corrupts profit.

So the return record is built by **spreading the original record and negating its stored money
fields**. Spreading (rather than listing fields) also means it can never miss a field added
later — `customer`, `vendor`, `employee`, `invId`, `seqRef`, `bulkInvId`, `salePhotoUrl` all
carry across automatically.

### 2.3 Partial refunds and the "keep the item?" question — exact, without needing the VAT rate

Two options complicate pure negation:

- **Refund less than the sale** (a damage deduction: paid 1500, refund 1400).
- **Don't restock** (the piece came back broken and is scrapped).

Both need an ex-VAT figure, and the record does **not** store the VAT rate that was in force.
Reading today's `state.vat` would reintroduce exactly the drift 2.2 avoids.

The escape: the original's ex-VAT is recoverable from its own stored numbers, because
`profit = exVat − cost`, therefore **`exVat = profit + cost`**. No VAT rate required, no drift.

```
origExVat    = orig.profit + orig.cost              (both stored on the record)
r            = refundAmt / orig.amount              (1 for a full return)
returnExVat  = −(origExVat × r)
returnCost   = restocked ? −orig.cost : null        (scrapped goods are not credited back)
returnProfit = returnExVat − (returnCost || 0)
```

Checks:

| Case | returnExVat | returnCost | returnProfit | Sanity |
|---|---|---|---|---|
| Full, restocked | −1500 | −900 | **−600** | exactly `−orig.profit` ✅ |
| Refund 1400, restocked | −1400 | −900 | **−500** | shop keeps €100, item back ✅ |
| Full, scrapped | −1500 | null | **−1500** | net profit −900 = the cost of goods lost ✅ |
| Cost unknown (`profit == null`) | — | null | **null** | unknown stays unknown ✅ |

The full-restock case — the common one — reduces to exact negation, so the default path carries
zero recomputation risk.

### 2.4 The cash / receipt regime carries through

Because `origExVat` is derived from the original's own stored profit, the regime the sale arrived
under (cash = no VAT extracted, receipt = VAT extracted) is already baked in. A cash sale reverses
as cash; a receipt sale un-declares exactly the VAT it declared.

When the refund becomes store credit, that credit carries `cashSale = orig.cashSale`, and the
existing gift-certificate machinery consumes it as `giftCardWasCash` when the credit is later
spent (v11.41). So money taken as cash in June is still treated as cash when it turns into a ring
in August, months later. **This is the single reason store credit is modelled as a gift
certificate rather than a new object** — that split-regime problem was already solved.

---

## 3. Store credit = a gift certificate with `origin: 'return'`

A store credit and a gift certificate are the same thing: a stored-value balance, redeemable
against a future sale, that expires. Reusing the object inherits, at zero cost:

- balance derivation from the redemption log (never a stored balance that can drift)
- the sale-time picker, the scan path (`gcFindByCode`), proportional spreading across a cart
- expiry sweep, the 30-day bell warning, breakage booking
- the printed document, `BACKUP_PATHS`, badges, the Desk list

**Numbering is shared** (`G-0012`, one counter). Considered a separate `C-` series and rejected:
two number spaces mean two scan resolvers and a chance of collision at the counter, to buy a
distinction the list, the detail sheet and the printed card already make obvious.

Fields added to the certificate record:

```
origin:         'return'          // absent / 'purchase' = a bought gift certificate
returnOfSaleId: <original sale id>
returnSaleId:   <the return record id>
cashSale:       <original sale's regime>     // consumed later as giftCardWasCash
buyerName/Phone/Email: the returning customer, copied from the sale
expiresAt:      +RETURN_CREDIT_TERM_DAYS (730)
```

### 3.2 A return credit runs for TWO years, not one

`RETURN_CREDIT_TERM_DAYS = 730`, separate from the bought-certificate default
(`GC_DEFAULT_TERM_DAYS = 365`, unchanged). Two reasons, both the owner's:

- **The shop is seasonal** (Apr–Nov). A customer given credit in August is realistically back
  the *following* season — a 1-year card could expire days before they walk in.
- **There is no hurry to be paid.** The money is already in hand, and the credit is a fixed
  number of euros against stock that reprices with gold. The longer it sits, the less of the
  shop it buys. Time favours the shop, so pressure on the customer buys nothing.

`gcTermDays(g)` picks the right term per card, so the Extend button offers a card its own kind.

### 3.1 The liability is one number

A bought certificate is money received in advance. A return credit is money already received —
but §2.1 just un-booked it, so it is back to being money owed in jewelry. Both are genuinely the
same liability, so the Desk "Outstanding" total sums them together, with a split shown
underneath so the owner can still see which is which.

---

## 3.3 The gift exchange card (`X-0001`) — value that doesn't exist yet

Someone buys a piece as a present. The person who unwraps it may want something else — and must
never learn what was paid for it. So they get a **κάρτα αλλαγής** printed with the gift.

**It is not a certificate and carries no value.** It identifies **the sale**. That distinction is
the whole design: a gift card with money on it would be a liability the moment it was printed,
for a piece that hasn't come back and may never come back. Instead the card is only a *way in* —
scanning it opens that sale, with the piece on screen and ↩️ Return right there. **Value comes
into existence only when the piece actually returns**, at which point the existing return flow
issues a credit card that *does* carry value.

- Number: `exchangeCardNo` on the sale record, counter at `/jewelprice_exchangecard_counter`.
  `X-0001` collides with neither a reference code (2 letters + 4 digits) nor a gift certificate
  (`G-0004`) — the letter and the hyphen keep all three apart.
- `handleScannedCode` resolves it **before** `lookupCode`, and opens `showSaleDetail(id, true)`.
- **Two entry points, both required by the owner:** a 🎁 **Gift toggle in the sale box** for when
  it's known in advance, and a 🎁 **button on the sale** for the customer who asks on the way out.
- **Idempotent:** reprinting a lost card returns the *same* number, never a second card.
- The printed card shows photo, description, metal/karat/type, ref code, purchase date and a
  validity date — and **never a price** (asserted in the tests).

## 4. Guards — the ways this could corrupt the books

| Risk | Guard |
|---|---|
| **Returning the same sale twice** | A sale is returned iff a record exists with `returnOfSaleId === s.id`. Derived, never a stored flag that can drift. The button is replaced by a "Returned on …" banner. |
| **Returning a return** | `isReturn` records offer Undo, never Return. |
| **Refunding cash the customer never paid** | A sale part-paid by a certificate can only refund that portion as **store credit**. The dialog states the split and enforces it — you cannot hand back cash for a €100 gift certificate. |
| **Deleting a return record** | Blocked. `deleteSaleRecord` would restore stock (a no-op — already restored) and orphan the credit note. Replaced by **Undo return**: re-marks the piece sold, removes the return record, and cancels the credit note — refused outright if that credit has already been partly spent, since that money is now in another sale. |
| **Restoring the wrong twin** | Reuses the exact-record logic from v10.59: prefer the stored `invId`, fall back to a QR match restricted to a `sold` item. Extracted into one shared helper so return and delete can never diverge. |
| **A return folded into the original transaction row** | `saleGroup` is dropped from the return. Returning several items of one transaction mints a *new* shared group so they collapse into one "↩️ 3 items" row of their own. |
| **Negative amounts breaking the analytics** | Every total is a plain `reduce(+amount)` / `+profit`, so negatives flow correctly. The bar chart clamps a negative month to a 0-height bar (verified, no broken layout). |

### 4.1 `_saleExVat` had to be made sign-safe

`_saleExVat` (v11.41) clamps the gift portion with `Math.min(Math.max(giftCardAmt, 0), amt)`,
which assumes a **positive** amount. Given `amount: −1500` and no certificate it returns
`Math.min(0, −1500) = −1500` — the whole amount lands in the gift portion and is then divided by
VAT under the *wrong* regime.

Rewritten to clamp by **magnitude** and force the gift portion to share the sale's sign. Proven
identical for every positive record (§6 test 7), correct for returns.

---

## 5. UI

**Return** lives on the sale, exactly as asked — a `↩️ Return` button in the sale detail sheet,
next to Edit and Delete. Multi-item transactions show it per item (open a line from the combined
view) plus "Return all" on the transaction.

**The return dialog** asks only what it must:

- refund amount (defaults to the full sale, capped at it)
- refund method — **💳 Store credit** (default) · 💵 Cash · 🏦 Card
- return date (defaults to today)
- reason (optional, free text)
- **📦 Put the piece back in stock** — on by default; off means scrapped (see §2.3)
- if the sale used a certificate: a fixed notice that €X of it returns as store credit

**After a credit return** the credit sheet opens with `🖨 Print exchange card` / `📤 Share`, and
a `🔄 Use now` that arms the credit for the next sale dialog opened. That arming is deliberately
narrow: single-use, and ignored after 30 minutes, so it can never silently attach itself to an
unrelated sale the next day. It is also visible and removable in the sale dialog.

**Elsewhere:** return rows are red with a `↩️` badge in the sales list; the original sale shows a
"Returned on …" banner; credit notes carry a `↩️` icon in the Desk gift list.

**Permissions:** the same rule the Edit button already uses — `can('sales_admin')`, or the
employee who made that sale. An employee alone in the shop has to be able to take a return. If
the owner would rather returns were owner-only, that is one condition in `_saleCanReturn`.

**The old "Restore to Stock (mistake / return)" button** keeps its job but loses the word
"return" — it is now labelled as the mistake path only, so the two are not confused.

---

## 6. Test plan

Driven against the owner's real backup (1072 items, live sales), in the browser, exercising the
real functions — not reasoning about them:

1. Full return → store credit. Item back in stock, negative record dated today, credit spendable.
2. The €1500 exchange, end to end: sell → return → credit → buy a cheaper piece with it → verify
   revenue, profit and the remaining balance.
3. Cash refund, receipt sale — VAT un-declared exactly.
4. Cash refund, cash sale — no VAT either way.
5. Return of a sale part-paid by a gift certificate — the gift portion is forced to credit.
6. Partial refund (damage deduction) and scrapped (no restock) — profit per §2.3.
7. **Regression: `_saleExVat` unchanged for every existing record** across amounts × costs × cash
   flags × VAT rates.
8. Multi-item: return one line, then all lines; grouping and totals.
9. Bulk chain (by-the-gram) return — grams back on the batch.
10. Guards: double return, undo, undo after the credit was partly spent.
11. Analytics: month/year totals, profit, leaderboard, bar chart with a negative month.

---

## 7. Deliberately not built

- **A separate "exchange" transaction type** — §1; it is return + sale.
- **Partial return of a discrete piece by value** (returning "half a ring"). A refund below the
  sale price is supported as a deduction, but the piece itself is all-or-nothing.
- **Restocking-fee automation / return windows.** A one-shop island jeweller sets these by
  judgement, not policy.
- **Marking a returned piece as "previously returned"** in stock. Considered, but it would be a
  stored field with no reader — the owner can note it on the item if it matters.
