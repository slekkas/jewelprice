# Security pass — credentials in links & owner recovery

**Status:** item 1 built (pending a Firebase rules change), item 2 designed only.
Raised 18 Jul 2026 after the roles work.

---

## Background: what was already fixed, and what wasn't

Earlier we removed the master DB secret from the **public website**. That fix holds — verified:
`showroom.htm` and `item.htm` contain **zero** secret references, read only the
`/jewelprice_public` mirror, and their sole mention of `jewelprice_inventory` is a comment
warning not to touch it. Regenerating the secret then was correct, as the old one had been
published.

But that covered the public *pages*, not the *links*. The secret still travelled in two URLs:

| Link | Goes to | Carries the secret? |
|---|---|---|
| Team invite link | employees (trusted) | **yes** — by design, it is how their app authenticates |
| **Quote link** | **customers (strangers)** | **yes** ← the problem |

The database secret bypasses all Firebase rules and grants full read **and write** to every
node — inventory, cost, margin, sales, clients, backups. Putting it in a link sent to
customers over WhatsApp/email is the same class of mistake already fixed on the showroom.

---

## 1. Quote links — remove the secret  *(BUILT)*

### Change
- `jewelprice.htm` → `shareQuote` no longer appends `&secret=…` to the viewer URL. A quote
  link is now just `quote.htm?id=…&db=…`.
- `quote.htm` already omitted `auth=` when no secret is present, so it needs no change to
  work — but its **all-quotes fallback** (`/jewelprice_quotes.json`, which reads EVERY quote)
  is now skipped unless a secret is present, i.e. only for old links. A modern link can read
  exactly one quote record and nothing else.

### Required Firebase rules change — **DO THIS FIRST**
`/jewelprice_quotes_html` must become world-readable (read-only), exactly like the showroom
mirror. In Firebase Console → Realtime Database → Rules:

```json
{
  "rules": {
    ".read": false,
    ".write": false,
    "jewelprice_public":      { ".read": true, ".write": false },
    "jewelprice_quotes_html": { ".read": true, ".write": false }
  }
}
```

**Order matters.** Update the rules *before* deploying the app change, or newly created quote
links will fail to load for customers (they would carry no credentials while the node is still
private). Old links already sent keep working either way — they still carry a secret.

### Residual risk (accepted for now)
Quote ids are `Date.now()` — a millisecond timestamp, so technically guessable. With the node
public, someone brute-forcing timestamps could surface a single quote (customer name, item,
price). ~86.4 million values per day makes this impractical at any scale, and Firebase
rate-limits, so this is **far** smaller than shipping the master key to every customer.

**Follow-up (not done):** make ids unguessable, e.g. `Date.now() + '_' + random`. Deliberately
deferred because the quote id is injected **unquoted** into inline handlers
(`onclick="deleteQuote(${e.id})"`, `reprintQuote`, `openSaleModalFromQuote`) and compared with
strict equality (`q.id === id`). A string id breaks those, so it needs its own careful pass.

### Should the secret be regenerated again?
Every quote link ever sent contains the current secret. Truly revoking that exposure means
generating a new secret — which **breaks every old quote link** and requires re-sending every
team invite link. Owner's decision; not required for the fix above to be effective going
forward.

---

## 2. Owner login & no self-service accounts  *(DESIGN ONLY)*

### What happens today
- `creds` defaults to **`role: 'owner'`**, so any fresh device starts as owner.
- The name prompt appears whenever no display name is set, and simply asks for a name.
- **But** a fresh device has no Firebase URL, so it has **no data** — it is an empty
  calculator. A stranger typing a name does not reach the shop's data.
- Therefore the owner is **not** locked out on a new phone; what he actually needs to recover
  is the **Firebase URL + DB secret**. If those are not written down somewhere off-phone,
  *that* is the real lock-out risk — independent of roles.
- A tampering gap exists: the role comes from a URL parameter, so `role=employee` can be
  edited to `role=owner`. `syncIdentityFromCloud` normally corrects this from the cloud user
  record, but the correction is skipped when the record is missing (`if (!u) return;`), so
  pairing a forged role with an unknown `user=` slug sticks.

### Proposed
1. **Stop defaulting to owner.** A device with no Firebase URL is "not set up" and shows
   *Connect to your shop*, not an account-creation prompt. No self-service accounts.
2. **Owner status requires the PIN.** The PIN already round-trips through the cloud config
   (`config.ownerPin` → `localStorage 'jpro_owner_pin'`), so on a new phone: enter URL/secret
   → the app learns the PIN → enter it → owner. Nobody else can claim owner without it.
3. **Owner recovery link.** Now that Owner is a choosable role, the owner can add *himself* as
   an Owner entry and keep that link somewhere safe (password manager). With (2), the link
   alone is not enough.
4. **Close the tamper gap:** if a device's user record is not found in the cloud, drop it to
   employee rather than trusting the role in the URL.

### The honest ceiling
None of this is real security while the secret rides in the team invite links — it is good
locks on a house whose master key is under the mat. Genuine enforcement needs Firebase Auth
with per-user accounts and rules, which is a much larger project. For a two-person family shop
the above is a sensible level; the point is to know the ceiling rather than feel protected.
