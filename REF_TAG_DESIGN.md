# Ref-code tags — design

**Status:** proposed (not built). Owner's idea, 17 Jul 2026.
**One line:** print the QR of the **reference code** (`RG0110`) instead of the descriptive
code (`ALO14D2.6-ZIR`), so the tag becomes a pure identity pointer and everything about a
piece becomes editable without a reprint.

---

## 1. Why this is right (and why it's smaller than it looks)

The app **already treats the ref code as the item's identity** everywhere except the QR:

| Path | Resolves a ref code today? |
|---|---|
| `lookupCode` (typed / scanned) | ✅ `/^[A-Za-z]{2}\d{4}$/` → finds item → feeds its stored code into the normal path |
| OCR text-scan (`refCodeExists`) | ✅ the whole point of it — reads the printed `PE0142` |
| Stocktake (`stkResolve`) | ✅ explicitly falls back to `itemRefCode(it)` |
| **Printed tag TEXT** | ✅ `buildTSPL`: `data.refCode ? [data.refCode] : …` — already prints ONLY the ref code |
| **Printed tag QR** | ❌ still the descriptive code |

So the tag already says `RG0110` to a human and `ALO14D2.6-ZIR` to a scanner. This change
makes the QR agree with the text. It is **finishing a migration the app already started**,
not a new architecture.

`lookupCode`'s ref path already **translates** ref → item → `item.qrCode` → normal lookup,
so every downstream consumer (weight, karat, vendor, category, secondary metal, stones,
photo, price) keeps working untouched. The stored `item.qrCode` does **not** change.

---

## 2. What it unlocks

1. **Edit anything without reprinting.** Weight, karat, stones, colour — the tag doesn't
   encode them, so it can't go stale. Today a corrected weight means delete → re-add →
   reprint. *(Owner's example: quote €500 for a 4 g ring, customer weighs it at 3 g.)*
2. **Unlimited stones.** The owner has been **under-recording stones** — pieces with 6
   stones were entered without them because the QR grew. That cost is invisible and ongoing.
3. **30% error correction, same size.** `RG0110` (6 chars) fits QR version 1 (21 modules)
   even at ECC **H**. Today's `ALO14D2.6-ZIR` needs version 2 at H. A short fixed payload
   means level H **and** a bigger cell size in the same 12 mm label → far fewer unreadable
   tags, which is exactly what the OCR fallback exists to paper over.
4. **The Greek / SHIFT-JIS problem evaporates for the QR.** Ref codes are ASCII. (The
   parallel Greek/Latin codes stay relevant only for legacy tags.)
5. **Variant ambiguity disappears** for new pieces (with §3.1) — the v11.16/11.17 scan
   chooser becomes a legacy fallback rather than the everyday path.
6. **The duplicate-QR guard becomes a duplicate-REF guard** — simpler and more correct.
7. **Renames become structurally irrelevant.** v11.10–v11.14 froze the *derivation*; this
   removes the derivation from the tag entirely.

---

## 3. Prerequisites (both required — the QR change alone is not safe)

### 3.1 One ref code per *distinguishable* piece
Today `invAddSimilar` deliberately copies `seqId` → **every group member shares one ref**
(RG0110 on both of the owner's rings). A ref QR on a shared ref is as ambiguous as today.

**Rule (owner's):** copy the ref **only when nothing differs**. If **weight**, **colour**
or **gold colour** differs → mint a new `seqId`.

- Identical twins keep one ref → one tag → interchangeable → correct (matches coins).
- Variants get their own ref → own tag → **scan is unambiguous, no chooser**.

### 3.2 Track which tag a piece carries
We must know whether an item's physical tag is a ref-QR or a descriptive-QR, or we can't
know whether editing its weight/stones breaks it.

- New field **`tagRef: true`**, stamped when a tag is printed with a ref QR.
- Absent → legacy descriptive tag → keep today's reprint prompts.

---

## 4. The change itself

```js
// buildTSPL — payload + robustness follow the payload length
const qrPayload = data.refCode || data.fullQR;          // ref when we have one
const short     = qrPayload.length <= 12;
const ecc       = short ? 'H' : 'M';                    // 30% vs 15% damage tolerance
const qrCell    = short ? 3 : 2;                        // bigger modules when we can afford them
cmds.push(`QRCODE ${qrX},${qrY},${ecc},${qrCell},A,0,M2,S7,"${qrPayload}"`);
```

Label geometry check (65×12 mm @ 203 dpi ≈ 96 dots tall, `qrY=20` → ~76 dots free):

| Payload | ECC | Version | Modules × cell | Height | Fits? |
|---|---|---|---|---|---|
| `RG0110` (6) | **H** | 1 | 21 × 3 = 63 | ~7.9 mm | ✅ |
| `ALO14D2.6-ZIR` (13) | M | 1 | 21 × 2 = 42 | ~5.3 mm | ✅ (today) |
| 6-stone code (~57) | M | 3 | 29 × 2 = 58 | ~7.3 mm | marginal → why stones got skipped |

---

## 5. Ripple list (the "little details")

| # | Area | Change |
|---|---|---|
| 1 | `buildTSPL` | ref payload + ECC H + cell 3 (§4) |
| 2 | `saveQrAsImage` | same payload |
| 3 | `invAddSimilar` | mint a new `seqId` when weight/colour/goldColor differs (§3.1) |
| 4 | save + reprint paths | stamp `tagRef: true` |
| 5 | `invEdit` | **add a weight field** (currently none) + karat; free for `tagRef`, prompts a reprint for legacy |
| 6 | `_promptStoneTagReprint` | skip entirely when `tagRef` |
| 7 | dup guard | check the **ref**, not the QR, for `tagRef` items |
| 8 | scan chooser (v11.17) | keep as the legacy fallback; unnecessary for new unique refs |
| 9 | `getNextSeqId` | recycling — see §6 |
| 10 | Stone limit | remove any implicit "keep the code short" pressure from the UI |

---

## 6. Risks — assessed

| Risk | Verdict |
|---|---|
| **Ref recycling** points an old tag at a new piece | **No new risk.** The ref code is *already* printed as text and *already* resolves via OCR/typing — recycling is equally exposed today. Owner: numbers are freed only by deleting a mistake, which is re-added immediately. **Keep recycling.** |
| Tag stops being self-describing | **Not a real problem.** The ref path already resolves to the record and fills every field. And the printed *text* is already only the ref code — the descriptive data is not on the tag today either. |
| Two tag generations forever | Accepted. Both resolve; the plumbing already exists and is exercised daily by the OCR path. |
| `refType` freezing becomes load-bearing | **Already done** (v10.20) — the ref prefix is frozen, so the ref code is stable across a type change. |
| Item must exist in inventory to price | True of the ref path today. Every printed tag corresponds to a saved item (v11.09). |

---

## 7. Migration

- **Old tags** (~600 pieces): untouched, keep scanning forever via the descriptive path.
- **New / reprinted tags**: ref QR + `tagRef: true`.
- No mass reprint. The estate converges naturally as pieces are reprinted.
- The two existing RG0110 rings keep sharing a ref until one is reprinted.

---

## 8. Open decisions

1. **Legacy items — editing weight/stones.** Prompt a reprint (today's behaviour), or offer
   "reprint as a ref tag" which fixes it permanently in one step? *(Recommend the latter —
   it converts the estate as a side effect of normal work.)*
2. **The two RG0110 rings.** Leave them sharing a ref (chooser handles it), or reprint one
   to give it its own?
3. **Cell size 3 vs 2.** Bigger, more robust — but is 7.9 mm of a 12 mm tag acceptable
   visually? Owner should eyeball one print.
