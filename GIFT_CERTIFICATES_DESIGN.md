# Gift Certificates — Design

Status: **planning** (nothing built yet). Target version: v11.40.

A customer pays now for a certificate; someone redeems it for jewelry later.
Modelled on the existing **Reservations** feature — private Firebase node, own
counter, printed A4 form, third segment in the Desk tab.

---

## 1. The money model (decided)

A gift certificate is money in **now** for goods out **later**. The trap is
counting it twice: log €200 when the certificate is sold *and* log the €200 sale
when it is redeemed, and the books show €400 of revenue for €200 of cash.

**Decision: count at redemption.**

| Event | Sales record | Revenue | Profit |
|---|---|---|---|
| Certificate issued (€200 taken) | none | — | — |
| Redeemed against a €160 piece | normal sale, `amount: 160` | €160 | 160 − cost |
| Expires with €50 unused | breakage record | €50 | €50 (pure) |

Over the life of a certificate, logged revenue equals cash received. No leak, no
double count.

### Sales stays untouched until a sale happens

Nothing is logged the day the €200 cash comes in — **decided: the Sales tab says
nothing about a certificate until it is actually spent.** No "issued today" line,
no outstanding-liability line. Sales means sales.

The total still outstanding is naturally visible where it belongs: the Desk →
Gift Cards list sums the active balances at the top. It never enters revenue or
profit.

---

## 2. Data

Private Firebase node **`/jewelprice_giftcards`** (array, same shape as
reservations). Counter at `/jewelprice_giftcard_counter`. Local cache
`localStorage 'jpro_giftcards'`. Numbered **`G-0001`**.

Add to `BACKUP_PATHS`. **Never** published — not in `PUBLIC_SAFE_FIELDS`,
`publicProjection` never sees it. Contains customer PII.

```js
{
  id: 'gc_<ms>', cardNo: 1,            // G-0001
  amount: 200,                          // face value, never changes
  buyerName, buyerPhone, buyerEmail,    // who paid — who we contact before expiry
  toName: '',                           // optional "To:" printed on the certificate
  message: '',                          // optional short dedication
  redeemerName: '', redeemerPhone: '',  // captured at redemption, not at issue (§3)
  redeemerEmail: '',
  payMethod: 'cash'|'card'|'bank',      // how the money came in
  cashSale: true,                       // receipt or not — forced false for card (§3)
  expiresAt: 'YYYY-MM-DD',
  status: 'active' | 'redeemed' | 'expired' | 'cancelled',
  redemptions: [ { saleId, amount, date, by } ],
  breakageSaleId: null,                 // set when expired balance is booked
  notifiedAt: null,
  dateCreated, createdBy
}
```

`balance = amount − Σ redemptions.amount`. Derived, never stored — a stored
balance is one more thing that can drift out of sync.

**Buyer vs recipient:** often the buyer is not the person who redeems (that is
the point of a gift). We hold the *buyer's* contact details — they are who we
call before expiry — and print an optional `toName` on the certificate itself.

---

## 3. Issue flow

Where: **Desk → Gift Cards → "＋ New certificate"**, plus a shortcut from the
Sales tab. All roles may issue (an employee has to be able to sell one).

Form: amount · buyer name (with the existing client autocomplete) · phone ·
email · optional "To" name · optional message · expiry date · payment method ·
cash toggle.

On save: assign `cardNo`, write the record, `upsertClientFromSale`, then offer
**Print** and **Share** of the certificate.

**No sale record is written.** (§1)

### Whose details do we take? — recommendation

**Buyer: name, phone, email, all required.** They handed over the money and they
are who gets called before expiry. Non-negotiable.

**Recipient: name only, optional. No phone, no email at issue.** Three reasons:

1. The buyer usually **doesn't know** their friend's email, and asking holds up
   the counter for a field that gets left blank anyway.
2. Contacting the recipient about an expiring gift is the wrong move regardless —
   you'd be telling someone their present is about to run out, possibly before
   they've even been given it. The expiry call goes to the buyer.
3. The certificate is a **bearer instrument** — paper plus QR plus your stamp.
   Whoever walks in with it redeems it. Recipient contact details don't gate
   anything, so they'd be data collected for no purpose.

**Take the recipient's details at redemption instead.** That is the natural
moment: they are standing in your shop spending money, you are already writing a
sale, and `upsertClientFromSale` already creates the client record. They stop
being a name on a gift and become an actual customer — which is the real prize
in a gift certificate. Someone else paid to introduce you to a new client.

So: `toName` optional at issue for the printed "To:" line, and
`redeemerName`/`Phone`/`Email` filled in during the redemption sale.

### How the money came in

Two separate things, and they are worth keeping separate:

- **`payMethod`** — the tender: **cash / card / bank transfer**. Useful for
  reconciling the till and for knowing what actually happened months later.
- **`cashSale`** — with or without a receipt, the same single toggle the rest of
  the app uses (per the house rule: cash is ONE toggle, never a pair).

**Card forces a receipt.** A card payment leaves a trace at the bank whatever the
app records, so when `payMethod === 'card'` the cash toggle is set off and
disabled, with a short note saying why. Bank transfer likewise. Only `cash`
leaves the toggle free. This stops the records from claiming something the bank
statement contradicts.

### The wrinkle: which regime governs the redemption?

This falls straight out of counting at redemption (§1), and it needs deciding
before the redemption code is written.

> €100 certificate taken as cash, no receipt. Months later the customer picks a
> €160 piece and pays the €60 difference by card. The sale is logged at **€160**.

If that sale carries a single "with receipt" flag, VAT is computed on the whole
€160 — but only €60 ever went through with a receipt. Flag it the other way and
€60 that went through the card reads as cash. One flag cannot describe a sale
whose two halves arrived under different regimes, months apart.

**Decided — split the calculation by portion.**

Worked example at the store's **17%** VAT (Karpathos island rate — never 24;
code must read `state.vat`), €100 certificate taken as cash, €60 paid by card,
cost €50:

| | Amount | Regime | Ex-VAT |
|---|---|---|---|
| Certificate | €100.00 | cash, no receipt | €100.00 |
| Card today | €60.00 | with receipt | €51.28 |
| | **€160.00** | | **€151.28** |

**Profit €101.28.** VAT owed on the card portion: €8.72.

For contrast, the same sale forced onto one flag: all-receipt gives €86.75
(€14.53 of profit lost to VAT on money that never saw a receipt), all-cash gives
€110.00 (€8.72 of real VAT ignored). The two errors reconcile to the €23.25
spread between them.

```js
giftPart  = giftCardAmt                     // regime: the certificate's cashSale
cashPart  = amount - giftCardAmt            // regime: this sale's own toggle
exVat     = part(giftPart, card.cashSale) + part(cashPart, sale.cashSale)
profit    = exVat - cost
```

Each half is treated as it actually happened. Totals stay right, and the sale
still records the true €160.

Two consequences to handle:

- The stored `cashSale` boolean on the record describes only what was paid
  today; the record also carries `giftCardWasCash` so the split can be recomputed
  and shown. The sale detail should spell it out rather than showing one badge —
  *"€100 certificate (cash) + €60 card"*.
- The combined-sale cash toggle (`_toggleGroupCash`) currently flips `cashSale`
  across every record in a transaction and re-derives profit. It must not
  silently rewrite the certificate half, which was settled months ago.

**This is the only part of the plan that touches `confirmSale`** — the most
safety-critical function in the app, since it writes the sales records that
profit, leaderboards and the day totals all read. It gets built last, and it gets
verified by driving real numbers through it against a copy of the live data
before it ships, not by reasoning about it.

The simpler alternative, if you'd rather not touch the profit maths: keep one
flag per sale, default it to the certificate's regime, and warn when they differ.
Less accurate on a mixed payment, but nothing changes inside `confirmSale`.

### The printed certificate

`buildGiftCertHTML(g)` — A4 HTML, printed via `openQuoteWindow` and shared via
`shareQuoteAsPDF` (filename `gift-G-0001.pdf`), the same path the quote and
reservation forms already use.

But unlike those, this one is **handed to a customer as a gift** — it is the
object someone unwraps, so it should look like a present rather than paperwork.
The quote form is the starting point for the branding, not the layout:

- **Landscape**, centred, generous margins — a certificate, not an invoice
- A thin **gold double-rule border** framing the whole sheet
- LEKKAS JEWELRY wordmark and logo at the top, as on the quote
- The heading **GIFT CERTIFICATE / ΔΩΡΟΕΠΙΤΑΓΗ**, letter-spaced small caps
- The **amount very large** in the gold accent — the one thing the eye lands on
- **To:** and the dedication message in an italic serif, only if filled in
- Certificate number and valid-until small and quiet at the bottom
- **QR** bottom-right (§4), small
- **Signature line** bottom-left and a **dashed stamp box** bottom-right, the
  same pair the reservation form uses — a signed and stamped certificate is what
  makes it real when someone brings it back a season later
- Short bilingual terms in fine print

No emojis, per the rule for every printed document. Worth a first draft printed
on real card stock before we call it done — this one is judged by eye, not by
logic, and it is the only thing in the app a customer keeps.

---

## 4. Redemption — the important part

Your idea, with one change.

In the sale dialog (single item from Calc, or a whole cart) add a **🎁 Gift
certificate** button next to the cash toggle. Tapping it opens a sheet listing
active certificates — name, number, balance, expiry — searchable. Pick one and
the dialog shows:

```
Total          €160.00
🎁 G-0004 (Maria P.)   −€100.00
─────────────────────────────
Cash due       €60.00
```

**The change: the €160 in the amount field must not go down.** It is tempting to
deduct the €100 there, since €60 is what the customer hands over — but that
field *is* the sale record. Deducting it would log a €160 piece as a €60 sale
and destroy €100 of revenue and €100 of profit, permanently, on every redemption.

So: the amount field stays at the full price, and the deduction is shown as its
own line telling you what to actually collect. The sale record gains
`giftCardId` and `giftCardAmt: 100`; `amount` stays `160`.

This also keeps the multi-item cart correct — `confirmSale` splits `amount`
proportionally across the items, and that split must be over the true total.

On confirm:

- push `{ saleId, amount: 100, date, by }` onto the certificate's `redemptions`
- balance 0 → status `redeemed`; balance left → stays `active` and is reusable
- certificate worth **more** than the piece: it covers the whole price, cash due
  €0, the remainder stays on the card
- the sale row in Sales shows a 🎁 badge, and its detail lists which certificate
  was used

Guards: an expired or fully redeemed certificate cannot be selected. Deleting a
sale that used a certificate must **give the balance back** — same principle as
`deleteSaleRecord` restoring a sold item (v10.59).

### Also redeemable against a custom order — but not a repair

**Decided.** A custom order is a piece of jewelry being made, so a gift
certificate paying for it is the same transaction as any other sale. A repair is
labour on the customer's own item — a different kind of thing, and left out.

So the same certificate picker appears in `_repairCollectOpen` **only when
`r.type === 'custom'`**, feeding `_recordRepairIncome` the same way: the final
price stays whole, the certificate line shows what is still due in cash.

### The QR on the certificate

The printed certificate carries a QR so you can scan it at the counter instead
of hunting for a name in a list. Payload is just the certificate number,
**`G-0004`**.

`handleScannedCode` gets a gift-card branch **before** the item lookup: anything
matching `/^G-\d{4}$/` opens that certificate. The hyphen is doing real work
here — reference codes are two letters plus four digits, and `GC0510` is a
**coin**. `G-0004` cannot be mistaken for one. Keep the hyphen.

Scanning a certificate while a sale dialog is open should select it for that sale
rather than just opening it.

---

## 5. Expiry, contact, and breakage

**Expiry date is per certificate — default 1 year, freely editable at issue.**
Most customers return within days, but an end-of-season buyer with little stock
left to choose from may well come back the following season, so the date has to
be the owner's call every time, not a fixed rule.

`_gcCheckExpiry()` runs from `autoSync`, like `_resCheckExpiry`:

- **Expiring soon** (within 30 days): raises a **notification on the existing
  bell** — see below.
- **Past expiry**: status → `expired`.

### The bell (reusing the employee-message notifications)

The app already has exactly the right mechanism: per-user records at
`/jewelprice_notifications/<slug>/<notifId>`, a polled badge
(`notifCheckAndUpdateBadge`, every 20s) and a panel (`openNotifPanel`). Today it
carries item messages between employees. Gift expiry becomes a second **kind** of
notification in that same list rather than a parallel system:

```js
{ kind: 'gift_expiry', giftCardId, cardNo, buyerName, balance, expiresAt, sentAt }
```

`openNotifPanel` branches on `kind`: an item message renders as it does now; a
gift-expiry one reads *"Gift certificate G-0004 (Maria P., €100) expires in 30
days"* with a **View certificate** button → `openGiftCardDetail(id)`, where the
Contact buttons live (WhatsApp / SMS / Email, a near-verbatim reuse of
`_repairNotify`, stamping `notifiedAt`).

Sent to owner/manager slugs — whoever can actually act on it. The notification id
is deterministic (`gcexp_<cardId>`) so repeated `autoSync` runs can't stack
duplicates, and the card is stamped `expiryNotifSent` so dismissing it from the
bell doesn't make it reappear on the next poll.

### Colour-coded by kind

So a glance at the bell tells you *what* is waiting, and so later kinds of
notification never blur together:

| Kind | Colour | Badge appears on |
|---|---|---|
| Message from an employee | **red** | bell + **Stock** tab (where it already is) |
| Gift certificate expiring | **green** | bell + Desk tab + the Gift Cards segment |

`notifCheckAndUpdateBadge` today takes a single `count` and hard-codes red on the
bell (`rgba(229,57,53,0.5)` / `#e57373`). It splits into `msgCount` and
`giftCount` driving the two badges independently; the bell shows the total and
takes its colour from what is in it — green when only expiries are waiting, red
as soon as a message is, since a person waiting on a reply outranks a date. In
the panel each row is tinted by its own kind, so a mixed list still reads
clearly.

The message badge (`navNotifBadge`) **stays on Stock**, where it already is —
messages are about items and tapping one jumps to the item. Only the new green
gift badge is added, on Desk.

The bell itself (`#invNotifBtn`) is positioned for the Stock tab only and needs
to be visible wherever a warning could land.

### Booking the unused balance

An expired certificate with a balance shows **"Book €50 as income"**, which
writes a sale record via `_recordGiftBreakage` —
`{ amount: 50, profit: 50, cost: null, cashSale: true, giftBreakage: true }` —
pure profit, exactly as you asked.

**One tap rather than fully automatic**, and deliberately so: the app has been
burned once by code that wrote to records on its own (the v10.19 stone
auto-pricing, which silently changed item prices and inflated every gain figure).
A tap keeps money entering the books only when a person put it there, and lets
you extend a certificate for a good customer instead of pocketing it. Say the
word and it becomes automatic on expiry.

**Worth one line, then it's your call:** enforcing expiry on prepaid vouchers is
restricted in some EU consumer law, and I don't know Greece's rule with enough
confidence to advise. Worth asking your accountant before you rely on the
breakage income.

---

## 6. Where it lives

Desk gains a third segment: `deskShow('reservations' | 'repairs' | 'gift')`.

- **List** — active first, then expiring soon, then redeemed/expired. Search by
  name or number. Each row: number, buyer, face value, **balance**, expiry.
- **Detail** — buyer block with tappable phone, amount/balance/expiry, redemption
  log, Contact, Print, Share, and (owner/manager, `can('desk_admin')`) Cancel.

Roles: issue and redeem = all roles. Cancel, delete, and edit the amount =
owner/manager.

---

## 7. Build order

1. Data layer — node, counter, get/save/push/pull, `autoSync` pull, `BACKUP_PATHS`
2. Desk segment: list + detail
3. Issue form
4. Printed certificate (`buildGiftCertHTML`) + print/share
5. **Redemption in the sale dialog** — the piece to get right
6. Expiry check, contact buttons, breakage booking
7. Bell integration: `kind: 'gift_expiry'` in `openNotifPanel`, bell visible
   outside the Stock tab
8. i18n both languages, `APP_VERSION` → 11.40, changelog inline + `changelog.htm`,
   CLAUDE.md feature entry, `manual.htm` section

---

## 8. Settled

- Sales shows nothing until the certificate is spent (§1)
- Expiry default 1 year, editable per certificate (§5)
- 30-day warning, delivered on the notification bell (§5)
- Breakage booked by one tap, never automatically (§5)
- Redeemable against a custom order, not a repair (§4)
- QR on the certificate, payload `G-0004` (§4)
- Notifications colour-coded by kind — red messages, green expiries (§5)
- Certificate designed as a gift: landscape, gold border, signature + stamp (§3)

- Message badge stays on Stock; only the green gift badge is added, on Desk (§5)
- Buyer's details required; recipient name optional, contact taken at
  redemption (§3)
- `payMethod` (cash/card/bank) separate from the receipt toggle; card forces a
  receipt (§3)

- Mixed-regime redemption: split the VAT/profit calculation by portion (§3)

## 9. Fix first — unrelated VAT defaults

Found while checking this feature's maths. Not part of gift certificates; should
ship as its own small commit **before** this work starts.

The store's VAT is **17%** (reduced island rate). Three places assume 24:

| Line | What | Impact |
|---|---|---|
| 4059 | `vat: 24` in the `state` initializer | Applies on a fresh device until `applyCloudConfig` loads the real 17 |
| 1635 | `placeholder="24"` on the Settings VAT input | Cosmetic, but suggests the wrong rate |
| 20948 | `parseFloat(cartItems[0].result.vat) \|\| 24` in `buildMultiQuoteHTML` | Hardcoded fallback **and** `\|\|` treats a legitimate VAT of 0 as missing, silently applying 24% |

Every other VAT read (~20 sites) uses `parseFloat(state.vat) || 0`, which is
safe — absent VAT means no VAT. Pricing and profit are unaffected today.

Fix: 4059 → 17, 1635 → 17, and 20948 → fall back to `state.vat` with an
explicit undefined check rather than `||`.

## 10. Still open

Nothing — ready to build once the VAT fixes above are in.
