# Team, roles & sync — redesign

**Status:** proposal for discussion (not built). Owner's brainstorm, 17 Jul 2026.
**Goal:** make the roles / team / sync area feel solid and professional, with roles
that actually *mean* something, while keeping the invite-link system that already works well.

---

## 1. What exists today (honest audit)

### Roles
Three role names exist — `owner`, `manager`, `employee` — but only **two tiers** are real:

- Almost every gate in the app is `isOwnerOrManager()` (owner **or** manager) vs
  `creds.role === 'employee'`. So **a manager currently has full owner powers**:
  profit/cost, vendors, sales dashboard, delete, backups, exports, settings, team.
- The **only** place manager differs from owner is a single PIN check. So today
  "manager" really means *"an owner who has to type the PIN"* — not a restricted admin.

**Employee** is the only genuinely restricted role. It hides: profit & cost, the sales
dashboard, vendors, settings, sync, backups, exports, stocktake, gain/appreciation lines,
inventory delete/group/coin/metal buttons, the stale-stock report, and more.

### Where it lives
- **Sync panel** (☰ More → Sync, owner/manager only) contains, stacked together:
  1. A **"👑 Owner / 👤 Employee" device toggle** — really "what is this device," not a role
     picker. Manager isn't an option here. Tapping "Owner" asks for the PIN.
  2. The **owner setup** (Firebase URL, DB secret, PIN, Telegram) OR the **employee connect**
     field, depending on the toggle.
  3. A **"👥 Team"** box with **"+ Add Person"** and the member list.

### Add Person / invite links
- "Add Person" opens a modal: name, username, bonus %, Telegram id, and a
  **Manager / Employee** toggle (owner can't be invited from here).
- Saving stores the member in the cloud `users` node. **"Share link"** builds a URL with
  `?firebase=…&role=…&user=…&secret=…` — opening it auto-connects the device in that role.
- The member list has **Edit / Share link / Deactivate (⏸)**. There is deliberately no
  delete (deactivate keeps their sales history intact).

### The security reality (important)
The invite link appends the **database secret for every role, employees included**
(`shareUserLink`: `if (creds.dbSecret) url += secret`). The secret bypasses Firebase rules,
so **every restriction in the app is UI-only** — it hides controls, it does not stop anyone
at the database. For a small, trusted family shop this is acceptable and is how it already
works. But it means: *"restrictions" = "a cleaner, safer, simpler screen for that person,"*
not *"a security boundary."* Real enforcement would need Firebase Auth + per-user rules — a
much larger change, out of scope unless the shop ever hires someone it doesn't trust.

---

## 2. What feels non-professional (the real problems)

1. **Manager is a fake tier.** It looks like a middle role but behaves like the owner.
2. **Two unrelated jobs share one screen.** "Set up *this* device" and "manage *other*
   people" are mixed in Sync, so neither reads clearly.
3. **The device toggle looks like a role switch** but isn't, and omits Manager.
4. **Roles are hardcoded**, so there's no way to say "this one employee may also see cost."
5. **Team is buried** in a settings-ish panel instead of being its own thing.

---

## 3. Proposed design

### 3.1 Separate the two concepts
- **Sync** stays, but becomes *only* "this device & its connection": Firebase URL, DB secret,
  PIN, Telegram, Save & Push, Share app. Remove the Team box from here.
- **New "Team" screen** in the ☰ More menu (owner/manager only) — this is the user's
  "separate users tab" idea. It holds the member list + Add Person + (Phase 2) permissions.
  This alone removes most of the clutter that reads as unprofessional.

### 3.2 Make the three roles real
Give each role a **defined default set of powers**, so Manager finally means something:

| Area | Owner | Manager | Employee |
|---|---|---|---|
| Calc, scan, sell, reserve, look up items, add clients | ✅ | ✅ | ✅ |
| See profit & cost, gain %, sales dashboard | ✅ | ✅ | ❌ |
| Add / edit / delete inventory, groups, coins, metal | ✅ | ✅ | ❌ |
| Vendors & labor rates | ✅ | ✅ | ❌ |
| Reservations & repairs (manage) | ✅ | ✅ | ❌ |
| Stocktake, CSV export, stale-stock report | ✅ | ✅ | ❌ |
| VAT / margin / currency / global settings | ✅ | ❌ | ❌ |
| Firebase / DB secret / PIN (Sync setup) | ✅ | ❌ | ❌ |
| Manage team (add/edit people) | ✅ | ➖ employees only | ❌ |
| Backups: create / restore | ✅ | create only | ❌ |

The point: **Manager = trusted day-to-day admin** (runs the shop, handles stock, sees the
numbers) but **can't change the money rules, the sync/secret, or promote people.** That's the
line most small shops actually want.

### 3.3 Add Person → pick a role, clearly
Replace the Manager/Employee toggle with **three labelled choices**, each with one line:
- **👑 Owner** — full access, no limits. *(rare — only another true co-owner)*
- **🔑 Manager** — runs the shop day-to-day; can't change money rules, sync, or the PIN.
- **👤 Employee** — everyday counter work; no prices/cost, no admin.

Then generate the invite link exactly as now (keep the mechanism — it's the good part).

### 3.4 Optional per-person tweaks (Phase 2, the "checkmarks")
On each member, after the role sets the defaults, the owner can flip a **small, meaningful**
set of overrides (≈8, not 50):
- See profit & cost
- Edit / delete inventory
- Manage vendors & rates
- Manage reservations & repairs
- Run stocktake / export data
- Manage backups
- Add other team members
- Change global settings (VAT / margin)

Stored as a `permissions` object on the user; the UI gates read
`hasPerm('see_cost')` instead of `isOwnerOrManager()`. Missing `permissions` → fall back to
the role default (so nothing breaks for existing members). This gives the flexibility the
owner described **without** exploding into dozens of micro-toggles.

---

## 4. Recommendation

Do it in two clean phases:

- **Phase 1 (solid + low-risk):** move Team into its own ☰ More screen; make Manager a real
  middle tier by splitting the handful of *owner-only* gates (settings, sync/secret/PIN, team
  management, backup restore) out of `isOwnerOrManager()`; give Add Person the 3-way role
  choice with plain descriptions. This fixes the "unprofessional" feel and makes roles honest.
- **Phase 2 (flexibility, only if wanted):** add the per-member permission checkmarks on top.

Keep the invite-link system throughout — it's the strongest part and shouldn't change.

I recommend shipping **Phase 1 first** and living with it for a bit. Three well-defined roles
cover almost everything a 2–3 person shop needs; the checkmarks are a nice-to-have you can
grow into rather than commit to up front.

---

## 4b. DECISIONS MADE (18 Jul 2026)

1. **Manager's line** — approved as proposed in §3.2.
2. **Per-person checkmarks** — **dropped.** Research (below) shows permissions belong on the
   *role*, not the person. Phase 1 ships three fixed roles; the permission table is built so
   editable *roles* remain a cheap add later if ever wanted.
3. **Managers may add employees only** (not other managers/owners). **Backups: create and
   restore are owner-only.**

### Why not per-person permissions — what comparable systems do

| System | Model |
|---|---|
| **Loyverse** (closest to this shop's size — free, small retail) | 4 preset roles: Owner / Administrator / Manager / Cashier. **Owner's rights cannot be changed.** All other roles editable via checkboxes **on the role**. |
| **Shopify POS** | Roles only — *"You can't assign individual permissions to POS staff, you need to assign a role."* |
| **Square** | 3 preset templates (Standard / Enhanced / Full); custom permission **sets** are a paid upgrade, free tier gets one. |

RBAC guidance agrees: attach permissions to roles, not users; per-user grants cause
"role explosion" and roles should describe responsibilities, not exceptions.

**Decisive point for this shop:** with **one employee**, "edit the Employee role" and "edit
that person" are the same action — so per-person toggles buy nothing until there are two
employees who must differ. Editable roles start paying off at ~5+ staff with genuinely
different jobs.

### Implementation consequence (important)
Phase 1 must put the role→permission mapping in **one table** and gate the UI on
`can('see_cost')` rather than today's scattered `isOwnerOrManager()` / `role === 'employee'`
checks (~60 sites). Then "make roles editable" later is a small settings screen that edits
that table, not a refactor. Owner stays fixed and unrestricted permanently.

```
ROLE_PERMISSIONS = {
  owner:    ALL,
  manager:  { see_cost:1, edit_inventory:1, vendors:1, reservations:1, stocktake:1,
              export:1, backup_create:1,  settings:0, sync:0, backup_restore:0,
              team:'employees_only' },
  employee: { /* everyday counter work only */ }
}
```

## 5. Still open

1. **The device toggle** — replace the Sync panel's "👑 Owner / 👤 Employee" toggle with a
   clearer "Set up this device" flow, or leave it? (Employees almost never touch it — the
   invite link sets their role automatically, so it's really a first-run owner control.)

## 6. Build order for Phase 1

1. **Permission table + `can()` helper**, and convert the ~60 existing gates
   (`isOwnerOrManager()`, `role === 'employee'`) to `can('…')`. Behaviour-neutral for owner
   and employee; this step is what makes Manager real.
2. **Split the owner-only gates** out of the old owner-or-manager lump: global settings
   (VAT/margin/currency), the whole Sync panel (Firebase URL / DB secret / PIN), backup
   restore, and promoting people.
3. **Move Team to its own screen** in the ☰ More menu; Sync keeps only device/connection setup.
4. **Add Person → 3-way role choice** with one-line descriptions; managers see only
   "Employee". Invite-link mechanism unchanged.
5. Verify against the real backup: owner sees everything as before, employee sees exactly what
   they see today (no regression), manager sits correctly in between.

*(No code has been changed. This is a plan only.)*
