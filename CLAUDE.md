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

## Key features (current — verified against jewelprice.htm at app v10.16)
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
- "Promo Image" generator for Instagram/Facebook (logo + photo + type/description, no price; Story or Post formats, live preview). Uses the high-res photo (`photoUrlHi`) when available so the large canvas stays sharp.

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
- **Dual-resolution photos (v10.10):** each uploaded inventory photo produces two ImgBB uploads — a small ~480px `photoUrl` (used everywhere in the app, keeps it light/fast) and a ~1200px/85% `photoUrlHi` (used ONLY by the public showroom, `item.htm`, and promo images). `_photoInputChanged` builds both dataURLs from the original picked file (hi only when the source is >480px, else it reuses the small one) and passes `(dataUrl, hiUrl)` to the picker's `onCapture`; the three inventory save paths (label save, scan-to-inventory, edit item) stash both in parallel pending vars and upload via `uploadInventoryPhoto()` (parallel ImgBB uploads, returns `{url,name,hiUrl}`). `photoUrlHi` is in `PUBLIC_SAFE_FIELDS` so it reaches `/jewelprice_public`; the showroom lightbox/compare views, `item.htm`, and `drawPromoCanvas` use `photoUrlHi || photoUrl` (grid thumbnails stay on the small `photoUrl` for fast browsing). Existing items only gain a hi-res version when their original photo is re-selected/re-taken (ImgBB can't upscale the already-shrunk 480px image).
- **HD upload tracking (v10.11 — TEMPORARY, remove when done):** while the owner re-uploads existing photos to gain their hi-res version, the inventory list shows a small green **"HD"** badge on the thumbnail of any item that already has `photoUrlHi`, and the inventory filter sheet gains an **"HD Photo"** group with **"HD ✓"** / **"Needs HD"** pills (filter array `_invFilters.hd`, values `'yes'`/`'no'`; only applies to items that have a `photoUrl`). This is a throwaway aid — delete the badge in `renderInvList`, the `hdGroup`/`hd` filter logic, and the `invFilterHd*` i18n keys once all photos are re-uploaded. **v10.16 fix:** `uploadInventoryPhoto` used to return `hiUrl:''` when the source was ≤480px (no distinct hi copy), leaving `photoUrlHi` empty so those items stayed permanently stuck in "Needs HD"; it now falls back to the small url in that case (`hiUrl: hi ? hi.url : small.url`) so every uploaded photo is flagged HD (showroom/promo already use `photoUrlHi||photoUrl`, so nothing else changes).
- **Stone-system review & hardening (v10.15):** three round-trip bugs fixed. (1) **Missing abbreviations** — `Akoya Pearl`, `Freshwater Pearl`, `South Sea Pearl`, `Tahitian Pearl` and `Jade` were in `STONE_TYPES` but absent from `STONE_ABBREV_MAP`, so `stoneToLabelPart` encoded them by full name (with carat → re-scanned as **Diamond**) or `type.slice(0,4)` (no carat → dropped on scan). Added them to `STONE_ABBREV_MAP` (keys `AKOY`/`FRES`/`SOUT`/`TAHI`/`JADE`, matching the historical 4-letter fallback so already-printed no-carat tags now parse) and to `COLORED_FULL`/`COLORED_FULL_EARLY`. (2) **Scan-to-inventory dropped qty/melee** — the invScan item builder hardcoded `qty: 1`, so scanning a `4xRUB`/`tZIR` tag into inventory lost the count/melee; now uses `s.qty || 1`. (3) **Label-save `stonesDesc` from lossy QR** — it rebuilt the description from `parseCodeInput(fullQR)` and merged DOM shape/price **by index**, so a no-carat Diamond (absent from the QR) was dropped and, when any stone was skipped in the code, shape/price shifted onto the wrong stone. Now builds `stonesDesc` directly from the DOM stone rows (the complete source the QR was itself built from), gated by the same stone-section-open check as `buildLabelStrings`, falling back to the QR only when there are no DOM rows. Note: 3 divergent copies of the diamond/colored price-per-carat tables still exist (calc estimate / inv-edit live price / another) — a maintainability risk, left as-is.
- **Photo-only edits don't trigger a tag reprint (v10.14):** the Edit Item reprint prompt now fires only when the **stones actually change** in that edit session, not whenever the rebuilt suffix differs from the stored code. On open, `invEdit` captures a baseline stone suffix (`_invEditStoneRowsEl.dataset.baselineSuffix = _buildStoneQRSuffix(...)`); on save it flags the tag stale only if `_newSuffix !== baseline`. This stops photo-only edits (e.g. the bulk HD-photo re-upload) from nagging for a reprint, including on legacy items whose stored code is malformed (those `+`-form codes stay in the DB and the tolerant parser handles them; editing/reprinting still auto-corrects them when the user does change stones).
- **Stone-code separator fix — skipped leading stone (v10.13):** the stone suffix on a code uses `-` before the FIRST stone and `+` before each subsequent one. The generators (`buildLabelStrings`, `_buildStoneQRSuffix`) previously chose the separator from the stone's **array index**, but a stone can be skipped (e.g. a carat-less Diamond yields no label part via `stoneToLabelPart`). So an item whose first stone was a carat-less Diamond followed by e.g. 4× Ruby produced `KAL18S1.3+4xRUB` (no `-`) instead of `KAL18S1.3-4xRUB`; with no `-`, `parseCodeInput` couldn't split base from stones and the calc showed "Code not found". Fixed by tracking the count of parts **actually emitted** (not the index) in all three generator loops. `parseCodeInput` is now also tolerant of the malformed form (no `-` but a `+` → treat the first `+` as the stone boundary) so already-printed bad tags still scan; and the edit-save + restock QR-regen paths normalise a leading `+` suffix to `-`, so editing/reprinting such an item auto-corrects its code. Existing bad codes in the DB are left as-is (the tolerant parser handles them).
- **Stone melee marking recovered from the QR code on edit (v10.12):** the printed QR code is the authoritative source for a stone's quantity/melee (`t` prefix, e.g. `-tZIR`). Older items whose stored `stonesDesc` lost the "melee" word (built before the recent qty change) used to re-parse as plain qty=1 in the Edit Item form, so simply changing a photo regenerated the code as `ZIR` and forced a spurious "print new tag" prompt. `invEdit` now overlays the qty read from `parseCodeInput(item.qrCode).stones` onto the `stonesDesc`-parsed rows (matched by index; or rebuilds rows straight from the code if `stonesDesc` yields none), so melee is restored on load and, on save, `buildStoneDescription` self-heals the description to include "melee" and the regenerated suffix matches the old one (no reprint prompt).
- About/Info page with the Lekkas Jewelry SVG logo, in-app Feature List, User Manual, and Changelog (currently at v10.16)

> Note: the in-app "About" version badge derives from the `APP_VERSION` constant (currently `'10.16'`), not the stale "2.9" hardcoded fallback in the markup. The canonical version to bump is `APP_VERSION` plus the `changelog` arrays (both `el` and `en`) in the `ABOUT` object. The green update banner also reads those same `changelog` arrays, so a normal version bump keeps everything in sync.

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
- The live app is served by **GitHub Pages** (custom domain `www.lekkasjewelry.com`, preserved by the root `CNAME` file), and deploys automatically on every push to `main`.
- Deployment runs via a custom GitHub Actions workflow: **`.github/workflows/deploy-pages.yml`**. It builds the site and **retries the Pages deploy up to 3 times** (with waits in between), so GitHub's occasional transient `Deployment failed, try again later` hiccup self-heals instead of turning the deploy red. This requires **Settings → Pages → Build and deployment → Source = "GitHub Actions"** (not "Deploy from a branch"); do not switch it back, or the retry protection is lost.
- You can also trigger a deploy by hand: **Actions** tab → "Deploy to GitHub Pages" → **Run workflow**.
- If all 3 attempts ever fail (a sustained GitHub Pages outage), just re-run the workflow later from the Actions tab.
- After a successful deploy, allow ~1–2 min for the CDN to update; the app's own green "new version" banner confirms the new `APP_VERSION` is live.

## Public website & data security
Alongside the private pricing app (`jewelprice.htm`) the repo also serves a small **public website** on the same GitHub Pages site:
- `index.html` — public landing page (hero, hours, contact, links to the showroom).
- `showroom.htm` — password-gated "Private Showroom": customers browse in-stock pieces (photo, description, type, metal, stones), select favourites, and send an inquiry via EmailJS. No prices shown.
- `item.htm` — single-item "view item" page opened from a QR/`?qr=` link (used in inquiry emails and physical tags).
- `quote.htm` — standalone quote view.

**Security model (important — do not regress):**
- These public pages must **never** carry the Firebase legacy **master DB secret** (it grants full read/write to the entire database) and must **never** read the private `/jewelprice_inventory` node (it contains cost, margin, vendor and supplier fields).
- Instead the app maintains a separate, **display-only public mirror** at Firebase node **`/jewelprice_public`**, containing only the safe fields in `PUBLIC_SAFE_FIELDS` (qrCode, descriptions, jewelryType, metal, karat, goldColor, weight, gender, handmade, stonesDesc, seqId, photoUrl, photoUrlHi, status, dateAdded). `jewelprice.htm` keeps it in sync automatically via `syncPublicItem` / `syncPublicDelete` (on every inventory save/delete) and a once-per-session self-healing `republishPublicShowroom()` (called from `renderInvList`). Only in-stock, non-bulk items are published (`publicProjection`).
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
