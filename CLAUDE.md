# JewelPrice Pro — Project Context

## What this is
JewelPrice Pro is a bilingual (Greek/English) jewelry pricing Progressive Web App for Lekkas Jewelry, a seasonal jewelry store in Karpathos, Greece. It is used daily in the store by the owner (Sotiris) and an employee.

## Critical technical constraints — never violate these
- The entire app lives in a **single file: `jewelprice.htm`**. Do NOT split it into multiple files, and do NOT create or edit `index.htm` (the file was renamed from index.htm; old references to that name are outdated).
- Hosted on **GitHub Pages**. The app is accessed via its full URL ending in `/jewelprice.htm` — never break or rename this path, as users have it bookmarked/installed.
- Backend: **Firebase Realtime Database** (migrated from JSONBin and jsonstorage.net — do not reintroduce those).
- All UI text must exist in **both Greek and English**, following the app's existing i18n pattern.
- Greek characters CANNOT be encoded in QR codes generated for BarTender UltraLite (SHIFT-JIS limitation). Label codes therefore use parallel Greek/Latin codes.
- The QR camera scan button uses BarcodeDetector and must remain **hidden on Safari/iOS** (unsupported there).
- Users are on mobile: employee uses a Xiaomi Redmi (Android/Chrome), another user is on iPhone (Safari). Test logic must account for both.

## Key features (current — verified against jewelprice.htm at app v9.99)
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
- QR camera scanning via BarcodeDetector with automatic calculation (Android/Chrome only; hidden on Safari/iOS)
- Duplicate-QR scan tool; label history; printable code reference card

### Quotes, sales & inventory
- Quote cart (multi-item quotes) with rounding; single/multi A4 print quotes with optional jewelry photo
- Quote history (searchable) and certificate generation
- Inventory management with type/stone filters; sold flow and sale modals
- Bulk chain ("by the gram") sale flow: sells part of a chain batch by weight, deducting from the remaining grams; a rounding-target field sets the exact final price (the entered amount is honoured exactly, discount % is derived from it for display only)
- Sales tracking: sales dashboard, sales search, and a sales leaderboard by employee (monthly drill-down, medal ranking)
- Sales bonus % per team member (live preview, stored on each sale)
- Purchase-cost tracking for profit calculation; value-since-added ("gain") display
- "Promo Image" generator for Instagram/Facebook (logo + photo + type/description, no price; Story or Post formats, live preview)

### Clients, team & sync
- Clients panel (client records/modals)
- Cloud sync via Firebase Realtime Database across devices; cloud identity (change phone, reopen link, everything restored)
- PIN-protected Owner mode; team management (add employees/managers with personalised install links); employee setup via URL parameter
- Optional Telegram notifications (bot token stored in Sync settings)
- Web Share API sharing / one-tap app share

### Platform & misc
- Installable PWA (inline manifest) on Android & iOS; bilingual Greek/English throughout (`data-i18n` + `ABOUT`/i18n dictionaries)
- Green "new version" banner on load after an update: `checkForUpdate()` reads the changelog entry for the current `APP_VERSION` straight from `ABOUT[lang].changelog`, so it stays in sync automatically (there is no separate list to maintain)
- Bilingual item names shown on quotes per active language
- JSON backup export and a data-recovery/force-restore tool
- Configurable VAT %, default margin %, currency symbol
- About/Info page with the Lekkas Jewelry SVG logo, in-app Feature List, User Manual, and Changelog (currently at v9.99)

> Note: the in-app "About" version badge derives from the `APP_VERSION` constant (currently `'9.99'`), not the stale "2.9" hardcoded fallback in the markup. The canonical version to bump is `APP_VERSION` plus the `changelog` arrays (both `el` and `en`) in the `ABOUT` object. The green update banner also reads those same `changelog` arrays, so a normal version bump keeps everything in sync.

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

## First-session task
On the first session with this file, read `jewelprice.htm` fully and update the "Key features" section above to match the actual current state of the app, since this document may be behind the code.
