# JewelPrice Pro — Project Context

## What this is
JewelPrice Pro is a bilingual (Greek/English) jewelry pricing Progressive Web App for Lekkas Jewelry, a seasonal jewelry store in Karpathos, Greece. It is used daily in the store by the owner (Sotiris) and an employee.

## Critical technical constraints — never violate these
- The entire app lives in a **single file: `jewelprice.htm`**. Do NOT split it into multiple files, and do NOT create or edit `index.htm` (the file was renamed from index.htm; old references to that name are outdated).
- Hosted on **GitHub Pages**. The app is accessed via its full URL ending in `/jewelprice.htm` — never break or rename this path, as users have it bookmarked/installed.
- Backend: **Firebase Realtime Database** (migrated from JSONBin and jsonstorage.net — do not reintroduce those).
- All UI text must exist in **both Greek and English**, following the app's existing i18n pattern.
- Greek characters CANNOT be encoded in QR codes generated for BarTender UltraLite (SHIFT-JIS limitation). Label codes therefore use parallel Greek/Latin codes.
- QR camera scanning prefers the browser's native **BarcodeDetector** and falls back to an **inlined jsQR** decoder (in a `<script id="jsqr-lib">` block) for desktop browsers that lack it (Chrome on Windows/Linux). `canDecodeQr()` is true where QR decoding works and false on Safari/iOS.
- **Text (OCR) scanning** (v10.6): a "Scan code text" toggle inside the scanner overlay (`toggleScanMode`) reads the **printed reference code** (e.g. `PE0142`, from `formatSeqId`) instead of the QR square — the fallback for sun-faded/scratched tags that killed the old attempt (which tried to OCR the free-form Greek code `ΚΑΛ18Σ4.5`). Uses **Tesseract.js**, lazy-loaded from CDN on first use (`loadTesseract`) and cached in IndexedDB (needs internet the first time only). The engine runs with a narrow char whitelist (`BCEGHKLNOPRTWX0123456789`) + single-line PSM; `extractRefCode` pulls a `[A-Z]{2}\d{4}` token, applies position-aware confusion fixes (O↔0, B↔8, etc.) and validates the 2-letter prefix against `REF_PREFIXES`. The live path crops a **wide, short strip** across the centre whose height scales with the frame **width** (not height) so it stays a tight one-line strip in any orientation — a fixed fraction of frame *height* produced a tall, sparse crop on portrait phones that single-line OCR couldn't read — and it OCRs that strip at a few normalised heights, taking the first valid read (text size varies with how close the tag is held); gallery/photo OCRs the whole image. The live loop accepts a code as soon as one frame reads a valid token that **matches a real inventory item** (`refCodeExists`) — the existence check rejects misreads into non-existent codes, and `lookupCode` then shows the item so a wrong read is caught by eye. (An earlier two-frame agreement gate was removed in v10.7: it caused a ~2s delay and could get stuck showing "Reading…" forever if the user moved the tag away before the second confirming frame.) Text mode shows a dashed **read-zone guide band** (`.qr-textguide`) marking where to place the code, and the frame turns **green** (`.qr-video-wrap.aligned`) the moment a valid code is readable — a live "aligned, hold still" signal driven by OCR success itself (no tilt sensor; readability *is* alignment). Result is fed to `handleScannedCode` → `lookupCode`, which already resolves ref codes to items. Because the Calc scanner now always has a working path (QR where possible, else OCR), the Calc scan icon **shows on all devices including iOS/Safari** — where it opens straight into text mode. **Inventory scanning stays QR-only** (its confirm flow parses QR, not ref codes), so the text toggle is hidden there.
- Users are on mobile: employee uses a Xiaomi Redmi (Android/Chrome), another user is on iPhone (Safari). Test logic must account for both.

## Key features (current — verified against jewelprice.htm at app v10.9)
The app has grown well past the v2.2 list. It is organized into navigable tabs/panels:
**Calc, Inventory, Vendors, Clients, Sales, Sync, Settings, About** (some hidden until enabled/owner mode).

### Pricing & calculation (Calc panel)
- Live gold & silver spot prices in € (auto-refresh ~every 5 min) with a mini price chart; warning + manual entry fallback when the live price can't be fetched; works offline with last saved prices
- Price calculation from metal / weight / karat / vendor + labor category
- Weight-based margin tiers (up to 5) and per-labor-category margin overrides
- Handmade toggle (indicator shown ONLY when active)
- Secondary metal support and a generic "other cost" pass-through
- Stone/gemstone rows with AI price estimation (dealer / wholesale / retail values), a stone-price cache (with a settings-panel manager), and support for many stone types (e.g. Labradorite as of v9.90)
- Discount % per transaction; price rounding (works for gold/silver and Other-type items)
- Customer-facing final price bar (no cost details); breakdown view for owner

### Labels, codes & scanning
- Label code system with parallel Greek/Latin codes for BarTender UltraLite (SHIFT-JIS constraint)
- QR camera scanning with automatic calculation — native BarcodeDetector on Android/Chrome, with an inlined jsQR fallback so it also works on desktop Chrome (Windows/Linux); can also scan a photo of a tag.
- Text (OCR) scanning of the printed reference code (e.g. `PE0142`) as a fallback for faded/damaged QR squares (Tesseract.js, toggle in the scanner). Works on all devices — including iPhone/Safari, which now gets a working camera scan path for the first time.
- Duplicate-QR scan tool; label history; printable code reference card

### Quotes, sales & inventory
- Quote cart (multi-item quotes) with rounding; single/multi A4 print quotes with optional jewelry photo
- Quote history (searchable) and certificate generation
- Inventory management with type/stone filters; sold flow and sale modals
- Bulk chain ("by the gram") sale flow: sells part of a chain batch by weight, deducting from the remaining grams; a rounding-target field sets the exact final price (the entered amount is honoured exactly, discount % is derived from it for display only)
- Sales tracking: sales dashboard, sales search, and a sales leaderboard by employee (monthly drill-down, medal ranking)
- Multi-item sales (whole-cart checkout) are tagged with a shared `saleGroup` id — one record per item is still stored (so all totals/profit/leaderboards stay exact), but the sales list, hero counts, and search collapse them into ONE transaction row ("🛍 N items" badge). Tapping opens a combined detail listing each item (each openable on its own via `showSaleDetail(id, true)`); deleting removes the whole group. Grouping is done at display time by `_groupSalesForDisplay()` (v10.0). The combined view has a whole-transaction "💵 CASH" toggle next to the total (`_toggleGroupCash` → `sdmGroupCashBtn`) that flips `cashSale` on EVERY member record at once and re-derives each item's profit, so cash can be set/cleared for the whole sale in one tap instead of editing items individually (v10.4)
- Sales bonus % per team member (live preview, stored on each sale)
- Purchase-cost tracking for profit calculation; value-since-added ("gain") display
- "Promo Image" generator for Instagram/Facebook (logo + photo + type/description, no price; Story or Post formats, live preview)

### Clients, team & sync
- Clients panel (client records/modals); a client's purchase-history popup collapses multi-item (cart) sales into one row (`🛍 N items` badge, via `_groupSalesForDisplay`) and each row opens the SALE detail (`showSaleDetail`) — from which the "View Item" button jumps to the sold inventory item (v10.4). The list has a sort menu (name A→Z / Z→A, newest/oldest) and a "date added" range filter with one-tap **Season** chips (v10.9): each client's date-added is derived from its id (`String(Date.now())`) via `getClientDate`; a season chip picks that year's operating window (Apr 1 – Nov 30) into the from/to date inputs, and every client card now shows its date added
- Cloud sync via Firebase Realtime Database across devices; cloud identity (change phone, reopen link, everything restored)
- PIN-protected Owner mode; team management (add employees/managers with personalised install links); employee setup via URL parameter
- Optional Telegram notifications (bot token stored in Sync settings)
- Web Share API sharing / one-tap app share

### Platform & misc
- Installable PWA (inline manifest) on Android & iOS; bilingual Greek/English throughout (`data-i18n` + `ABOUT`/i18n dictionaries)
- Green "new version" banner on load after an update: `checkForUpdate()` reads the changelog entry for the current `APP_VERSION` straight from `ABOUT[lang].changelog`, so it stays in sync automatically (there is no separate list to maintain)
- Bilingual item names shown on quotes per active language
- JSON backup export and a data-recovery/force-restore tool
- **Automatic cloud backups (v10.9):** a once-per-calendar-day rolling snapshot of ALL data nodes (the 9 in `BACKUP_PATHS`: config, quotes, clients, sales, users, stone_cache, labels, qr_images, inventory), stored **privately** at Firebase node `/jewelprice_backups/<ms>` (written with the DB secret, so it stays private — NOT in the public mirror or repo). `autoBackupIfDue()` runs from `autoSync()` on app open, gated to owner/manager devices (those holding `creds.dbSecret`) and to one snapshot per day via `localStorage 'jpro_lastAutoBackup'`; keeps the newest `BACKUP_KEEP` (30) via `pruneCloudBackups()`. Settings → Data Backup adds "☁️ Back Up Now" (`backupNowToCloud`) and "↩️ Restore" (`openCloudBackups` → per-snapshot `restoreCloudBackup`), which takes an automatic `pre-restore` snapshot, PUTs each non-null node back, then reloads. `collectBackupData()` is shared by `exportBackup()` and the cloud backups. Snapshot keys are `String(Date.now())` (once-a-day cadence makes collisions a non-issue)
- Configurable VAT %, default margin %, currency symbol
- About/Info page with the Lekkas Jewelry SVG logo, in-app Feature List, User Manual, and Changelog (currently at v10.9)

> Note: the in-app "About" version badge derives from the `APP_VERSION` constant (currently `'10.8'`), not the stale "2.9" hardcoded fallback in the markup. The canonical version to bump is `APP_VERSION` plus the `changelog` arrays (both `el` and `en`) in the `ABOUT` object. The green update banner also reads those same `changelog` arrays, so a normal version bump keeps everything in sync.

## Owner preferences
- Clean, professional UI. **No emojis in printed documents** (quotes, labels).
- Keep changes minimal and targeted — this is a production app used in a live store.
- Increment the version number and update the in-app changelog with every user-visible change.
- Explain changes in plain language; the owner is not a professional developer.

## Workflow rules
- At the start of every session, if working in a local folder, run `git pull` to get the latest from GitHub before reading or changing anything.
- After completing a change, summarize what was modified before committing.
- Use clear, descriptive commit messages.
- Push to GitHub only when asked (GitHub Pages deploys automatically from main).
- **Update this CLAUDE.md as part of EVERY change**, in the same commit as the code. Whenever `jewelprice.htm` changes in a user-visible way, also: (1) bump the version reference(s) here to match the new `APP_VERSION`, and (2) if the change adds, removes, or meaningfully alters a feature, update the "Key features" section so it stays accurate. Do not treat CLAUDE.md as a separate follow-up task — it ships with the change.

## Deployment notes
- The live app is served by **GitHub Pages** (custom domain `www.lekkasjewelry.com`), which deploys automatically on every push to `main` via the built-in "pages build and deployment" workflow.
- GitHub Pages occasionally returns a transient `Deployment failed, try again later` error in the deploy step even though the build (artifact) succeeded — this is a GitHub-side hiccup, not a code problem. When it happens, the fix is to re-run the deploy: open the repo **Actions** tab → the failed "pages build and deployment" run → **Re-run all jobs**, or push a trivial new commit to `main` to trigger a fresh run.
- After a successful deploy, allow ~1–2 min for the CDN to update; the app's own green "new version" banner confirms the new `APP_VERSION` is live.

## Public website & data security
Alongside the private pricing app (`jewelprice.htm`) the repo also serves a small **public website** on the same GitHub Pages site:
- `index.html` — public landing page (hero, hours, contact, links to the showroom).
- `showroom.htm` — password-gated "Private Showroom": customers browse in-stock pieces (photo, description, type, metal, stones), select favourites, and send an inquiry via EmailJS. No prices shown.
- `item.htm` — single-item "view item" page opened from a QR/`?qr=` link (used in inquiry emails and physical tags).
- `quote.htm` — standalone quote view.

**Security model (important — do not regress):**
- These public pages must **never** carry the Firebase legacy **master DB secret** (it grants full read/write to the entire database) and must **never** read the private `/jewelprice_inventory` node (it contains cost, margin, vendor and supplier fields).
- Instead the app maintains a separate, **display-only public mirror** at Firebase node **`/jewelprice_public`**, containing only the safe fields in `PUBLIC_SAFE_FIELDS` (qrCode, descriptions, jewelryType, metal, karat, goldColor, weight, gender, handmade, stonesDesc, seqId, photoUrl, status, dateAdded). `jewelprice.htm` keeps it in sync automatically via `syncPublicItem` / `syncPublicDelete` (on every inventory save/delete) and a once-per-session self-healing `republishPublicShowroom()` (called from `renderInvList`). Only in-stock, non-bulk items are published (`publicProjection`).
- **Firebase Realtime Database rules** must keep everything private by default and expose **only** `/jewelprice_public` as world-readable, read-only:
  ```json
  { "rules": { ".read": false, ".write": false,
      "jewelprice_public": { ".read": true, ".write": false } } }
  ```
  (The app itself still reads/writes everything because it authenticates with the DB secret, which bypasses rules.)
- `showroom.htm` and `item.htm` read `/jewelprice_public` **without any auth token**. If either ever needs the secret again, that's a security regression.
- Changes to the public website files do **not** bump `APP_VERSION`/changelog — that constant governs the pricing app (`jewelprice.htm`) and its in-app version banner only.

## First-session task
On the first session with this file, read `jewelprice.htm` fully and update the "Key features" section above to match the actual current state of the app, since this document may be behind the code.
