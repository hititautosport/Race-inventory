# Race Shop Inventory — Build Guide

**Owner:** Jordan Welch (Hit It Autosport / Hit Digital)
**Last updated:** 2026-10-19 (v2: standalone app on Supabase with logins)
**Purpose of this file:** Upload it to any new chat (Claude or ChatGPT) to continue work on this app without needing old conversations. It describes what exists, how the data is shaped, the rules the app must follow, and what comes next.

---

## 1. What the app is for

An inventory system for an ARCA race team bought with 15 chassis, 7 engines, 2 complete cars and a large stock of spares. There was no existing shop inventory.

- **Phase 1 (now):** Count everything in the shop fast. Every entry defaults to the **Shop** location.
- **Later:** QR labels on bins, scan to check parts out of the Shop and into the Hauler, and saved **track kits** that tell the crew what to load for each track.

---

## 2. Current build (v2)

| | |
|---|---|
| **App address** | https://hititautosport.github.io/Race-inventory/ (GitHub Pages) |
| **Code** | GitHub repo `hititautosport/Race-inventory` — one file, `index.html` (HTML + CSS + vanilla JS). Pushing to `main` updates the live app in about a minute |
| **Database** | Supabase org "18", project **Race Inventory** (ref `osbjpkhxerztakdqdyql`, region us-west-2, Free plan). URL `https://osbjpkhxerztakdqdyql.supabase.co` |
| **Libraries** | `@supabase/supabase-js@2.45.4` (jsDelivr UMD), `qrcodejs 1.0.0` (cdnjs, makes QR codes), `jsqr 1.4.0` (jsDelivr, reads QR codes from the camera), `jspdf 2.5.1` (cdnjs, label PDFs), Google Fonts: Barlow Condensed, IBM Plex Sans, IBM Plex Mono |
| **Key** | The page uses the project's public **anon** key (works with every supabase-js version). It can't read data without a signed-in user |
| **Keep-awake** | GitHub Action `.github/workflows/keep-awake.yml` pings the database and the users service Mondays and Thursdays so the free plan never pauses. Run it by hand from the repo's Actions tab. GitHub turns off scheduled jobs if the repo has no commits for 60 days; if that happens, re-enable it in the Actions tab |
| **Logins** | Supabase Auth, email + password. No public sign-up screen: the Master admin adds people from the Users screen |
| **User management** | Edge Function `manage-users` (first-admin setup, list, add, change level, reset password, remove) |
| **Photos** | Private Storage bucket `photos` (5 MB max, jpeg/png/webp). Compressed in the browser to max 1280 px, JPEG 0.7. Shown with 1-hour signed links |
| **Live updates** | Supabase Realtime on all inventory tables, so counts from one phone show up on the others |
| **Export** | CSV (opens in Excel). **Stock totals:** one row per item, a column per location, total, last entry date and who made it (initials). **Full log:** every entry with date/time, type (Count, Move), quantity, location, bin, note, and who made it (initials + name). Who-made-it is only in the exports, not shown in the app. On iPhone, files open the Share sheet (Save to Files, Print, AirDrop) |
| **Old v1** | Claude artifact https://claude.ai/artifact/7veNwWLonVmRKc2qwzbL7L — retired; its data was copied to Supabase on 2026-10-19 |

Free plan notes: the project pauses after 7 days with no use (data is kept; unpause it on supabase.com), 500 MB database, 1 GB photos, no automatic backups, so export a CSV weekly.

## 2a. User levels

| Level | Can do |
|---|---|
| **No access** | Can sign in but sees "Waiting for access". Every new account starts here unless the admin picks a level |
| **View only** | Stock tab only: search, filters, item details, QR labels, export. No Count tab, no editing |
| **Editor** | Everything on Count and Stock: count, add/rename categories, parts, sizes, locations, undo and delete count entries, save bins, photos |
| **Master admin** | Everything, plus all deletes (categories, parts, sizes, items, locations, photos) and the Users screen |

Rules enforced in the database (row level security), not just hidden in the app:
- Read inventory: View only and up. Add/change: Editor and up. Delete count entries: Editor and up. Delete anything else: Master admin.
- `profiles` table holds each user's level (`disabled`, `viewer`, `editor`, `admin`). Users can only read their own row; admins read all. Levels change only through `manage-users`.
- There must always be at least one Master admin, and an admin can't remove their own account.
- First visit with no admin in the database shows **Set up the master admin**; after that it's always **Sign in**.

## 3. Data model

Every quantity is calculated from a log of **moves**. Quantities are never edited directly. On-hand at a location = sum of `qty` of all moves for that item at that location.

### Collections / tables

**locations** — where stock lives
| field | type | notes |
|---|---|---|
| id | text | v1 seeds: `shop`, `hauler` |
| name | text | "Shop", "Hauler", later bins or specific cars |
| sort | int | display order |
| created_at | timestamp | |

**categories**
| field | type | notes |
|---|---|---|
| id | text | |
| name | text | unique |
| sort | int | |

Seeded categories: Engine, Drivetrain, Chassis, Suspension, Steering, Brakes, Body, Cooling, Fuel, Electrical & Electronics, Safety, Wheels & Tires, Hardware & Fasteners, Fluids & Consumables, Setup Tools, Pit Equipment, Shop Tools.

**parts** — the part-name list, per category (starts empty, grows with "Add new")
| field | type | notes |
|---|---|---|
| id | text | |
| category_id | text | → categories |
| name | text | singular, no size in the name ("Camber shim", not "Camber shims .030") |

**sizes** — size/spec list, per part (starts empty, grows with "Add new")
| field | type | notes |
|---|---|---|
| id | text | |
| part_id | text | → parts |
| name | text | ".030", "1°", "3/8 x 1.25" |

**items** — one stock-keeping unit = category + part + size. This is what gets a QR code.
| field | type | notes |
|---|---|---|
| id | text | |
| code | text | unique, e.g. `SUS-7K3Q` (3-letter category prefix + 4 random chars, no 0/O/1/I) |
| category_id, part_id | text | |
| size_id | text, nullable | |
| bin | text | home bin/shelf, set from the first count if blank |
| photo | text, nullable | asset id (v1) / storage path (Supabase) |
| notes | text | |
| created_at, created_by | | |

**moves** — the log. Every count, check-out, check-in, use or damage is a row.
| field | type | notes |
|---|---|---|
| id | text | |
| item_id | text | → items |
| qty | int | positive adds, negative removes |
| type | text | v1 writes only `count`. Future: `check_out`, `check_in`, `used`, `damaged`, `adjust`, `transfer` |
| location_id | text | → locations |
| bin | text | where it was counted |
| note | text | |
| photo | text, nullable | |
| by | text | viewer id |
| at | ISO timestamp | |
| ts | number | ms since epoch, for sorting |

---

## 3a. Look and logo

- **Colors:** red, white and black. Accent red `#D7232F` (white text on it), dark theme background `#101315`, light theme background `#ECEFED`. Errors and Delete also use red.
- **Logo:** "Fast bins" — two red parts bins and a checkered top bin leaning forward with speed lines, on black. Files in the repo: `icon.svg` (rounded), `icon-square.svg` (full square), `apple-touch-icon.png` (180), `icon-192.png`, `icon-512.png`, `favicon-32.png`, `manifest.webmanifest` (home-screen name "Race Inv").
- **Title:** logo + slanted two-line wordmark, "RACE SHOP" in white/black and "INVENTORY" in red.

## 4. Count screen rules (do not break these)

Speed is the top requirement. Any change must keep these behaviors:

1. Fields in order: **Category → Part → Size/spec (optional) → Quantity → ENTER**.
2. Each field is its own rounded bubble (no table look). Tapping one opens a picker **centered on screen** with the options shown as tappable bubbles, plus search. Typing a name that doesn't exist shows **Add "…"** at the top; adding saves it to the permanent list immediately and **selects it automatically** (it is added to the on-screen list before the save finishes so the selection never gets dropped).
3. Near-duplicates are caught: matching ignores case, punctuation and plural "s" ("Camber shims" matches "Camber shim").
4. Lists start empty (except categories and locations).
5. Picking a category opens the part picker. Picking a part opens the size picker if that part has sizes or was just added; otherwise it jumps to quantity.
6. Quantity uses the app's **own number pad** (1–9, 0, Clear, Back) that sits right above the Enter button. The top row of the pad shows the part being counted (left), the quantity being typed (center, large, red outline) and **Close** (right). Keys act on press for instant response. The phone keyboard never opens for quantity, so it can't cover Enter. It appears after a part/size is picked or when the quantity box is tapped. Desktop keyboards can type digits and press Enter. The pad has a **Close** button at the top, and tapping anywhere else on the page also closes it.
7. **ENTER** is a big button pinned to the bottom. It saves in the background, shows a toast with **Undo**, clears Part/Size/Quantity/Note/Photo, and reopens the part picker.
8. **Category, Location and Bin stay set** between entries (you count one shelf at a time). Location and Bin / shelf sit at the **bottom** of the Count screen, under Photo and Note.
9. Location defaults to **Shop**.
10. Same item counted twice (e.g. two bins) adds up; it never overwrites.
11. The Count screen has no history list. **Recent entries** (Menu → Counting) shows the last 25 **count** entries with time and Undo (moves don't show there). The toast after Enter also has Undo.
12. Pickers have **Edit list** to rename and delete. Delete always works: first tap shows exactly what else goes with it (e.g. "Also removes 1 stock item, 3 count entries"), second tap confirms. Deleting a category removes its parts, sizes, stock and entries; deleting a part removes its sizes, stock and entries; deleting a size removes the stock items with that size. Locations that still hold stock can't be deleted (says why), and the last location can't be deleted.
13. Save status in the header is **hidden when everything is saved**. A small grey blinking dot shows while saving. A red **Not saved** pill appears only if a save failed (tap it for help; the error message has Retry).
16. **Saves go out one at a time, in order.** A new part/item always reaches the database before the count that points at it. (Before 2026-10-19 21:10 a new item and its first count were sent together and the count could arrive first and be rejected by the database's link check.)
14. Pickers have a red outline and a dark blurred background so they stand out from the page.
17. **Layout on iPhone:** the window itself scrolls (never the body). The sticky header sits at the very top and pads itself by the status-bar height, so nothing scrolls behind it. The Enter bar sits at the bottom with only a small gap above the home bar. Don't put `overflow` on `body` or padding on `:root`/`html`; it doubles the safe-area space. The home-screen app uses status bar style **black** (not black-translucent): with black-translucent, iOS shortens the page by the status-bar height but anchors it to the top, leaving a dead strip under the Enter bar. The Menu's bottom line shows a screen report (screen size, view size, home-screen app or browser) for troubleshooting.
15. Zoom is locked: no pinch zoom, no double-tap zoom, no sideways panning. Every text box uses 16px+ text so iPhones don't auto-zoom when it's tapped.

## 5. Stock screen

- **Scan** (red button in the header, every user level): opens the camera, reads a QR label and jumps to that item. Also has a box to type a code. Closing it turns the camera off. iPhone asks for camera permission the first time.
- **Labels** button: makes one PDF of QR labels for exactly the items currently showing (search or filter first, e.g. "Suspension" or "Rack B"). Two formats: **Label printer** (one 3.5 × 1.5 in label per page) or **Letter sheets** (Avery 5163/8163, 2 × 4 in, 10 per page). The choice is remembered.
- Grouped by category, searchable by part, size, code or bin, filterable by location. Category headings are large (24px, white, total in red); part names are smaller (14px) so categories stand out while scrolling.
- Tap an item: code, QR, qty per location, photo, home bin (editable), full history with delete, **Count more**, **Move** (Editor+: from/to location with on-hand counts, how many, blocks moving more than is there; saved as two linked `transfer` rows, −qty at From and +qty at To, sharing a `transfer_id`; Undo or deleting either row removes both), **Save QR label** (PNG, 1050×450, sized for 3.5×1.5 in labels), **Delete item** (two taps; removes the item and its entries from inventory, keeps the part name in the list).
- Exports and label printing are in the **Menu** (section 5a).

## 5b. Fixing mistakes, low stock, Used/Damaged, bulk changes (v2.5)

**Rule:** none of this touches the Count screen. The only thing that can appear while counting is the duplicate prompt below, and only when a near-identical part already exists.

- **Duplicate prompt when adding a part:** before creating a new part, the app checks every category for the same or a similar name (ignores case, spaces, hyphens, plural "s", and catches short forms like "Carb" vs "Carburetor"). If found, it offers **Use Engine › Carburetor** or **Add "Carb" to Fuel anyway**.
- **Find duplicate parts** (Menu → Master admin): groups same/similar part names across all categories. Tap **Keep** on the right one, tap again to confirm, and the others merge into it.
- **Merge parts** (master admin, run from the app in order): each item of the merged part moves to the kept part. Sizes line up by name (missing sizes move over). If the kept part already has the same part+size item, the two items are merged: all counts/history move to the kept item, which fills in a blank bin, photo or low-stock number from the other, then the extra item is removed. Finally the merged part is removed.
- **Fix** (item screen, Editor+): pick Category, Part (or "＋ New part '<same name>' here") and Size (or "＋ New size"). Counts and history stay with the item. If the result is identical to an existing item, a Master admin is offered **Merge into CODE**; an Editor is told to ask a master admin.
- **Used / Damaged** (item screen, Editor+): Used or Damaged, take from a location (shows on-hand), how many (can't exceed what's there), optional note. Saved as a negative `used`/`damaged` move. Undo on the toast.
- **Keep at least** (low stock): on the item screen next to Home bin, with a single Save. Stock rows show a red **Low** tag when the total across all locations is below it; a **Low (N)** filter chip appears when anything is low. Stock export has "Keep at least" and "Low" columns.
- **Quick edit** (Stock tab → **Edit**, Editor+): every row gets Keep at least / Used / Damaged boxes; pick which location Used/Damaged come out of; changed boxes turn red; one **Save N changes** button at the bottom (or Cancel). Won't save if a Used/Damaged number is more than what's in that location. Works with search and filters.
- **Bulk edit sheet** (Menu → Bulk changes, Editor+): **Download** makes a CSV (opens in Excel) with Code, Category, Part, Size, Bin, a column per location, Total, Keep at least, Used, Damaged, Take from, Note. Fill in what you need and **Import** it (CSV or Excel .xlsx). A preview lists what will change and what's skipped (unknown codes, non-numbers, taking more than is there); nothing changes until **Apply**. A blank Bin cell clears the bin; a blank Keep at least clears it. Applied through the database function `bulk_apply` in batches of 150.

## 5a. Menu (tap your initials, top right)

- **Counting** (Editor+): Recent entries (last 25 counts with Undo; has a Menu button to go back).
- **Bulk changes** (Editor+): Download bulk edit sheet, Import bulk edit sheet.
- **Reports:** Stock totals (Excel), Full log (Excel), Print QR labels (opens Labels for whatever Stock is showing).
- **Settings on this phone:** Theme (Auto / Light / Dark), "Open the part list after Enter" (on by default; Editors+ only). Saved per phone.
- **Master admin:** Manage users, Find duplicate parts.
- **Account:** Change my password, Sign out. App version and screen report at the bottom (currently 2.5).

## 6. QR / label rules

- One QR per **item** (part + size). Small parts (shims, slugs, bolts) get the QR on the **bin**, not each piece.
- Serialized parts (chassis, engines, shocks, rear ends, transmissions, carbs) should each be their own item with their own QR so each keeps its own history. Name them with the serial or number in the Size/spec field for now (e.g. Part "Chassis", Size "#7 Hamke"); Phase 5 adds proper serialized records.
- The QR is a link: `https://hititautosport.github.io/Race-inventory/#SUS-7K3Q`. Scanning it with the phone camera opens the app (after sign-in) straight to that item. The code after `#` is `items.code`, so a future in-app scanner reads the same labels.
- Print on oil-resistant polyester thermal labels. Label PNG is 1050×450 (3.5×1.5 in at 300 dpi).

---

## 7. Roadmap

### Phase 2 — Move to Supabase ✅ done 2026-10-19
Supabase project, user levels, live updates, photos in Storage, hosted on GitHub Pages, v1 data copied (17 categories, 4 parts, 1 size, 3 items, 2 locations; there were no count entries yet).

### Phase 3 — Scan check-out / check-in (partly done)
- ✅ Camera QR scanning opens the item. ✅ Move between locations (`transfer` rows).
- Next: scan-to-load mode (scan bins one after another, each adds to a load list, one tap moves them all Shop → Hauler).
- After a race: scan back in, or mark `used` / `damaged` (negative move at Hauler). Used/damaged builds a restock list.
- Optional: low-stock minimum per item with a Low tag and "Low only" filter (reorder list).

### Phase 4 — Track kits
- `kits` (name, track_id, is_base) and `kit_lines` (kit_id, item_id, qty).
- A **Base kit** goes to every race; each track adds its own lines (gears, springs, body parts).
- Pick a track → pick list (base + track lines) with shortages shown against Shop stock → scan each bin onto the truck → check-out moves.

### Phase 5 — Serialized assets
- Chassis and engine records: serial, builder, crash/repair history, laps/hours since rebuild, dyno sheets (PDF uploads).
- Expiration tracking for safety items (belts, seats, nets, fire systems) with reminders.

---

## 8. Supabase schema (as built)

Applied as migrations `roles_and_profiles`, `inventory_tables`, `photos_bucket_and_realtime`, `lock_helper_functions`. Column names match what the app writes.

```sql
create type public.app_role as enum ('disabled','viewer','editor','admin');
create table public.profiles (id uuid primary key references auth.users on delete cascade, email text, full_name text,
  role public.app_role not null default 'disabled', created_at timestamptz default now());
-- trigger on auth.users insert -> profiles row with role 'disabled'
-- helpers (security definer): my_role(), can_read() viewer+, can_edit() editor+, is_admin()

create table public.locations  (id text pk, name text, sort int, created_at timestamptz);
create table public.categories (id text pk, name text, sort int, created_at timestamptz);
create table public.parts  (id text pk, category_id text -> categories on delete cascade, name text, created_at);
create table public.sizes  (id text pk, part_id text -> parts on delete cascade, name text, created_at);
create table public.items  (id text pk, code text unique, category_id -> categories cascade, part_id -> parts cascade,
  size_id -> sizes cascade null, bin text, photo text /* storage path */, notes text, created_at, created_by uuid default auth.uid());
create table public.moves  (id text pk, item_id -> items cascade, qty int, type text check (count|transfer|check_out|check_in|used|damaged|adjust),
  location_id -> locations, transfer_id text, bin text, note text, photo text, by uuid default auth.uid(), at timestamptz, ts bigint /* ms */);
create view public.stock with (security_invoker) as select item_id, location_id, sum(qty) qty from moves group by 1,2;

-- RLS on every table: select can_read(); insert/update can_edit(); delete is_admin() (moves: can_edit())
-- team_names(): id + name of every profile, only for viewer+; used for the "By" columns in exports
-- items.min_qty int (Keep at least / low stock)
-- bulk_apply(changes jsonb): editor+; per item: min_qty, bin, used, damaged, location_id, note -> updates items and adds used/damaged moves
-- Merges run from the app (supabase-part.js mergeItems / mergeParts) because migrations containing deletes need an approval step that doesn't reach the phone
-- storage bucket 'photos' (private): read can_read(), upload can_edit(), delete is_admin()
-- realtime publication: locations, categories, parts, sizes, items, moves
```

Phase 4 tables still to add: `tracks (id, name, track_type)`, `kits (id, name, track_id, is_base)`, `kit_lines (kit_id, item_id, qty)` with the same RLS pattern.

---

## 9. Testing

After every change, run a full tap-through test on a phone-sized touch screen with a stand-in for Supabase that enforces the same permission rules: first-admin setup, sign in (wrong and right password), every picker, Add new, Edit list rename/delete, number pad keys (fast double taps, Back, Clear, Close, tap-outside), +/−, bin, note, photo, Enter, both Undos, Stock search and filters, item sheet (QR link, bin, label, Count more, history delete, Delete item), Export, Users screen (add, duplicate email, change level, last-admin guard, reset password, remove), change own password, sign out, View only and Editor restrictions, No access screen, opening a QR link, light and dark theme, no sideways scrolling, no script errors. Also: Scan (camera finds a label, unknown code, typed code, camera stops on Close), Move (over-limit blocked, both rows written and linked, Undo and history delete remove both, not in Just counted), label PDFs in both formats for exactly what Stock shows, export columns for who and when. Also: saves arrive in order (the stand-in database enforces the same links between tables and answers in random order like a phone connection), status hidden when saved, Menu reports/theme/setting. Also: the Count screen has exactly the same controls as before (checked by id), duplicate prompt, Find duplicates merge (totals add up), Fix to another category, Fix into an existing item (merge), Used/Damaged limits and Undo, Keep at least + Low tag + Low filter, Quick edit with limit check, bulk sheet download → edit → import preview → apply, Editor and View only restrictions. The latest run passed 68/68 checks. Proven: the version before this fix lost 5 of 6 rapid new-item counts in the same test; the fixed version saved 6 of 6.

## 10. How to continue in a new chat

Upload this file and say something like:

> "This is the build guide for my race shop inventory app. Read it, then [do the next thing]. Keep every rule in sections 2a and 4. Update the guide's changelog when you're done."

The code is `index.html` in the `hititautosport/Race-inventory` GitHub repo; this guide is also saved there as `BUILD_GUIDE.md`. Database changes go to the Supabase project in section 2.

---

## 11. Changelog

- **2026-10-19** — v2.5: duplicate-part prompt, Find duplicate parts + merge, Fix item, Used/Damaged, Keep at least with Low tag/filter, Stock quick edit, bulk edit sheet export/import with preview. Count screen unchanged. 68/68 checks passed.
- **2026-10-19** — Stock list: category headings bigger and brighter, part names smaller.
- **2026-10-19** — v2.4: home-screen app status bar switched to solid black to remove the dead strip under Enter; screen report added under the version in the Menu.
- **2026-10-19** — Count screen simplified: Just counted list moved to Menu → Recent entries (last 25, with Undo); Location and Bin / shelf moved to the bottom. 57/57 checks passed.
- **2026-10-19** — Fixed iPhone layout: no dead space above the header, nothing scrolls behind it, smaller gap under Enter.
- **2026-10-19** — v2.3: fixed lost counts on brand-new items (saves now go in order); status dot hidden unless saving or failed; Export moved into a new Menu under your initials with Reports, Theme, part-list setting, users, password, sign out and version. 56/56 checks passed.
- **2026-10-19** — New red/white/black theme (red replaces yellow everywhere), new "Fast bins" logo for the home screen, browser tab and header, slanted two-tone title. 54/54 test checks passed.
- **2026-10-19** — Added Scan button (camera), Move between locations, batch label PDFs (label printer or Avery 5163 sheets), who-made-it initials and names in Excel exports (not shown in the app), readable dates in exports, and a twice-weekly keep-awake job so the free database never pauses. 54/54 test checks passed.
- **2026-10-19** — Removed the "Chassis · Engines · Spares" subtitle under the title.
- **2026-10-19** — v2: standalone app on GitHub Pages + Supabase. Email/password logins, user levels (No access, View only, Editor, Master admin) enforced in the database, Users screen, change own password, live updates between phones, photos in private Storage, QR codes are links that open the item, iPhone Share sheet for exports/labels. v1 data copied over. 48/48 test checks passed.
- **2026-10-19** — Delete now works on anything in use (with a confirm that lists what goes with it); Delete item added to the item sheet; quick taps on two different spots no longer get swallowed; full 36-check button test passed.
- **2026-10-19** — Quantity readout moved onto the number pad's top row; Back and Close fixed; adding a new part/size/category now auto-selects it.
- **2026-10-19** — Number pad gets a Close button; tapping outside it closes it too.
- **2026-10-19** — Gold outline on pickers, quantity scrolls above the number pad, zoom and sideways panning locked.
- **2026-10-19** — Quantity now uses a built-in number pad above Enter (phone keyboard no longer blocks Enter).
- **2026-10-19** — Pickers now open in the middle of the screen; options and form fields are separate bubbles instead of table rows.
- **2026-10-19** — v1 built: Count and Stock screens, category/part/size pickers with add-new, edit list, location + bin, photos, QR per item, label PNG, CSV export. Seeded 17 categories and Shop/Hauler locations.
