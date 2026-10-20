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
| **Libraries** | `@supabase/supabase-js@2.45.4` (jsDelivr UMD), `qrcodejs 1.0.0` (cdnjs), Google Fonts: Barlow Condensed, IBM Plex Sans, IBM Plex Mono |
| **Logins** | Supabase Auth, email + password. No public sign-up screen: the Master admin adds people from the Users screen |
| **User management** | Edge Function `manage-users` (first-admin setup, list, add, change level, reset password, remove) |
| **Photos** | Private Storage bucket `photos` (5 MB max, jpeg/png/webp). Compressed in the browser to max 1280 px, JPEG 0.7. Shown with 1-hour signed links |
| **Live updates** | Supabase Realtime on all inventory tables, so counts from one phone show up on the others |
| **Export** | CSV stock totals and full count log. On iPhone, files and labels open the Share sheet (Save to Files, Print, AirDrop) |
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

## 4. Count screen rules (do not break these)

Speed is the top requirement. Any change must keep these behaviors:

1. Fields in order: **Category → Part → Size/spec (optional) → Quantity → ENTER**.
2. Each field is its own rounded bubble (no table look). Tapping one opens a picker **centered on screen** with the options shown as tappable bubbles, plus search. Typing a name that doesn't exist shows **Add "…"** at the top; adding saves it to the permanent list immediately and **selects it automatically** (it is added to the on-screen list before the save finishes so the selection never gets dropped).
3. Near-duplicates are caught: matching ignores case, punctuation and plural "s" ("Camber shims" matches "Camber shim").
4. Lists start empty (except categories and locations).
5. Picking a category opens the part picker. Picking a part opens the size picker if that part has sizes or was just added; otherwise it jumps to quantity.
6. Quantity uses the app's **own number pad** (1–9, 0, Clear, Back) that sits right above the Enter button. The top row of the pad shows the part being counted (left), the quantity being typed (center, large, gold outline) and **Close** (right). Keys act on press for instant response. The phone keyboard never opens for quantity, so it can't cover Enter. It appears after a part/size is picked or when the quantity box is tapped. Desktop keyboards can type digits and press Enter. The pad has a **Close** button at the top, and tapping anywhere else on the page also closes it.
7. **ENTER** is a big button pinned to the bottom. It saves in the background, shows a toast with **Undo**, clears Part/Size/Quantity/Note/Photo, and reopens the part picker.
8. **Category, Location and Bin stay set** between entries (you count one shelf at a time).
9. Location defaults to **Shop**.
10. Same item counted twice (e.g. two bins) adds up; it never overwrites.
11. "Just counted" list shows the last 8 entries with Undo.
12. Pickers have **Edit list** to rename and delete. Delete always works: first tap shows exactly what else goes with it (e.g. "Also removes 1 stock item, 3 count entries"), second tap confirms. Deleting a category removes its parts, sizes, stock and entries; deleting a part removes its sizes, stock and entries; deleting a size removes the stock items with that size. Locations that still hold stock can't be deleted (says why), and the last location can't be deleted.
13. Status in the header shows "All saved" / "Saving…" / "Some saves failed" with Retry.
14. Pickers have a gold outline and a dark blurred background so they stand out from the page.
15. Zoom is locked: no pinch zoom, no double-tap zoom, no sideways panning. Every text box uses 16px+ text so iPhones don't auto-zoom when it's tapped.

## 5. Stock screen

- Grouped by category, searchable by part, size, code or bin, filterable by location.
- Tap an item: code, QR, qty per location, photo, home bin (editable), full history with delete, **Count more**, **Save QR label** (PNG, 1050×450, sized for 3.5×1.5 in labels), **Delete item** (two taps; removes the item and its entries from inventory, keeps the part name in the list).
- **Export:** stock totals CSV (one row per item, a column per location) and full count log CSV.

## 6. QR / label rules

- One QR per **item** (part + size). Small parts (shims, slugs, bolts) get the QR on the **bin**, not each piece.
- Serialized parts (chassis, engines, shocks, rear ends, transmissions, carbs) should each be their own item with their own QR so each keeps its own history. Name them with the serial or number in the Size/spec field for now (e.g. Part "Chassis", Size "#7 Hamke"); Phase 5 adds proper serialized records.
- The QR is a link: `https://hititautosport.github.io/Race-inventory/#SUS-7K3Q`. Scanning it with the phone camera opens the app (after sign-in) straight to that item. The code after `#` is `items.code`, so a future in-app scanner reads the same labels.
- Print on oil-resistant polyester thermal labels. Label PNG is 1050×450 (3.5×1.5 in at 300 dpi).

---

## 7. Roadmap

### Phase 2 — Move to Supabase ✅ done 2026-10-19
Supabase project, user levels, live updates, photos in Storage, hosted on GitHub Pages, v1 data copied (17 categories, 4 parts, 1 size, 3 items, 2 locations; there were no count entries yet).

### Phase 3 — Scan check-out / check-in
- Camera QR scanning (works in a Supabase-hosted web app; the Claude artifact can't use the camera).
- Check-out = move of type `transfer` written as two rows sharing a `transfer_id`: −qty at Shop, +qty at Hauler.
- After a race: scan back in, or mark `used` / `damaged` (negative move at Hauler). Used/damaged builds a restock list.

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
-- storage bucket 'photos' (private): read can_read(), upload can_edit(), delete is_admin()
-- realtime publication: locations, categories, parts, sizes, items, moves
```

Phase 4 tables still to add: `tracks (id, name, track_type)`, `kits (id, name, track_id, is_base)`, `kit_lines (kit_id, item_id, qty)` with the same RLS pattern.

---

## 9. Testing

After every change, run a full tap-through test on a phone-sized touch screen with a stand-in for Supabase that enforces the same permission rules: first-admin setup, sign in (wrong and right password), every picker, Add new, Edit list rename/delete, number pad keys (fast double taps, Back, Clear, Close, tap-outside), +/−, bin, note, photo, Enter, both Undos, Stock search and filters, item sheet (QR link, bin, label, Count more, history delete, Delete item), Export, Users screen (add, duplicate email, change level, last-admin guard, reset password, remove), change own password, sign out, View only and Editor restrictions, No access screen, opening a QR link, light and dark theme, no sideways scrolling, no script errors. The 2026-10-19 v2 run passed 48/48 checks.

## 10. How to continue in a new chat

Upload this file and say something like:

> "This is the build guide for my race shop inventory app. Read it, then [do the next thing]. Keep every rule in sections 2a and 4. Update the guide's changelog when you're done."

The code is `index.html` in the `hititautosport/Race-inventory` GitHub repo; this guide is also saved there as `BUILD_GUIDE.md`. Database changes go to the Supabase project in section 2.

---

## 11. Changelog

- **2026-10-19** — v2: standalone app on GitHub Pages + Supabase. Email/password logins, user levels (No access, View only, Editor, Master admin) enforced in the database, Users screen, change own password, live updates between phones, photos in private Storage, QR codes are links that open the item, iPhone Share sheet for exports/labels. v1 data copied over. 48/48 test checks passed.
- **2026-10-19** — Delete now works on anything in use (with a confirm that lists what goes with it); Delete item added to the item sheet; quick taps on two different spots no longer get swallowed; full 36-check button test passed.
- **2026-10-19** — Quantity readout moved onto the number pad's top row; Back and Close fixed; adding a new part/size/category now auto-selects it.
- **2026-10-19** — Number pad gets a Close button; tapping outside it closes it too.
- **2026-10-19** — Gold outline on pickers, quantity scrolls above the number pad, zoom and sideways panning locked.
- **2026-10-19** — Quantity now uses a built-in number pad above Enter (phone keyboard no longer blocks Enter).
- **2026-10-19** — Pickers now open in the middle of the screen; options and form fields are separate bubbles instead of table rows.
- **2026-10-19** — v1 built: Count and Stock screens, category/part/size pickers with add-new, edit list, location + bin, photos, QR per item, label PNG, CSV export. Seeded 17 categories and Shop/Hauler locations.
