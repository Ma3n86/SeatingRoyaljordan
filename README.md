# Odeon Theatre — Registration Desk Dashboard

A single-page check-in dashboard for the registration desk. Search guests, see their
seating classification, check them in, and show them where to go on a 2D map of the
amphitheatre. It runs in **real-time multi-agent mode** (Supabase) with an automatic
**offline preview fallback**.

## Real-time mode (Supabase) — for 4 agents at once

1. In Supabase, run the SQL in the big comment block at the top of the `<script>` in
   `index.html` (creates the `guests` + `checkins` tables, enables Realtime on
   `checkins`, and adds demo RLS policies).
2. Import your guest list into the `guests` table (Supabase Table editor → *Import CSV*).
   Expected columns: `zone, sector, org, title, name_ar, name_en` (the app also accepts
   the original CSV headers and a few aliases, see `normalizeRow()`).
3. Paste your project **URL** and **anon key** into `SUPABASE_URL` / `SUPABASE_ANON_KEY`
   at the top of the `<script>`.
4. Deploy to Vercel (static — just serve `index.html`, `odeonimage.jpeg`, `vendor/`).
   All agents now share check-ins live: a check-in on one device appears on the others
   within a second via the `checkins` realtime channel. The header pill shows
   **● Live sync** when the realtime channel is connected.

> **Why `checkins.guest_id` is the primary key:** Postgres only includes primary-key
> columns in realtime `DELETE` payloads (unless `REPLICA IDENTITY FULL`). Making
> `guest_id` the PK means un-checking a guest broadcasts their id correctly to every
> agent, and check-in becomes a simple `upsert`.

## Offline preview fallback

If the credentials are left as placeholders (or Supabase can't be reached), the app
automatically loads the embedded `guests.js` and stores check-ins in `localStorage`,
showing an amber notice and a **● Offline (local)** pill. This lets you preview the UI
with no backend — but check-ins do **not** sync between devices in this mode.

> For best results use Chrome or Edge.

## File structure

```
index.html            ← the whole app (open this)
guests.js             ← your guest list, embedded (auto-generated from FINALRSVP.csv)
odeonimage.jpeg       ← the seating chart used as the 2D map background
vendor/fuse.min.js    ← fuzzy-search engine (local copy, offline)
vendor/tailwind.js    ← styling engine (local copy, offline)
FINALRSVP.csv         ← the source spreadsheet
```
(`vendor/three.min.js` is no longer used — the 3D view was replaced by the 2D image map — and can be deleted.)

## Group categorization

Each guest is auto-categorized by scanning their `Zone`/`Sector`/`الجهة`/`المسمى الوظيفي`
for keywords (first match wins, in this priority order):

| Group | Keyword(s) | Colour |
|---|---|---|
| Lifetime Awardee | `lifetime`, `awardee` | Magenta `#FF00FF` |
| Europa Nostra | `europa`, `nostra` | Purple `#800080` |
| Diplomat | `diplomat` | Green `#00FF00` |
| Officials | `official` | Black `#000000` |
| JKB Guests | `jkb` | Maroon `#800000` |
| Shortlisted & Guests | `shortlist` | Teal `#008080` |
| PNT BoD & GA | `pnt` | Blue `#0000FF` |
| Zone A | `zone a` | Gold `#FFD700` |
| Free Seating | *(none — default)* | Light grey |

The **escort-card warning** ("CARD ON CHAIR") fires when the group is a Zone-A subgroup
(Lifetime / Europa / Zone A) **or** the row's metadata literally contains `Zone A` — so a
Zone-A Diplomat keeps its green "Diplomat" label *and* still shows the warning.

To re-tune, edit the single `GROUPS` array near the top of the `<script>` in `index.html`.

## Search engine

- **Fuse.js** powers the fuzzy matching, so typos and spelling variants still hit
  (e.g. typing **"Seif"** finds **"Saif"**, "mohamad" finds "Mohammad").
- **Strict Arabic normalisation** is applied identically to the data *and* your query
  before matching, so typing **`احمد`** instantly matches **`أحمد`**. It:
  unifies all Alef forms (أ إ آ ٱ → ا), Taa Marbuta (ة → ه) and Alef Maksura (ى → ي),
  and strips all Tashkeel/diacritics and Tatweel.
- Multi-word queries are AND-matched (every word must match), best results first.

## How to update the guest list

You have **two options**:

### Option A — Load a CSV at the desk (no coding)
Click the **“Load CSV”** button in the top-right and pick any updated `.csv` file.
It replaces the list instantly for the current session. Use this for last-minute edits.

The CSV must keep the same 6 columns, in this order:

| Column | Header | Meaning |
|---|---|---|
| 1 | *(blank)* | **Zone** — put `Zone A` for Zone A guests; leave **empty** for free seating |
| 2 | `Sector` | group, e.g. `Diplomat`, `JKB guests`, `Officials`, `Shortlisted` |
| 3 | `الجهة` | Organisation |
| 4 | `المسمى الوظيفي` | Job title |
| 5 | `Name (AR)` | Arabic name |
| 6 | `Name (EN)` | English name |

A row is treated as a guest only if the **Sector** column is filled, so blank spacer
rows and the totals block at the bottom of the sheet are ignored automatically.

### Option B — Bake a new list permanently into the app
If you want the new list to load automatically on double-click, regenerate `guests.js`.
With Python installed, run this in the project folder:

```bash
python -c "import json; open('guests.js','w',encoding='utf-8').write('window.RSVP_CSV = '+json.dumps(open('FINALRSVP.csv','r',encoding='utf-8-sig').read(),ensure_ascii=False)+';')"
```

(Replace `FINALRSVP.csv` with your file name if different.)

## Seating rules (as implemented)

- **Zone A** (`Zone` column = `Zone A`): a big red, pulsing banner appears —
  *“ZONE A: {Name} is on the chair. HAND THEM A ‘ZONE A’ ESCORT CARD.”*
  (Escort cards are in `escortcards.pdf`.)
- **Free seating** (`Zone` column empty): a blue banner —
  *“FREE SEATING: Direct guest to the unassigned White Areas.”*
  If their group has a designated coloured block on the map (Diplomats, JKB, Officials,
  Shortlisted, PNT BoD), the banner also points to it and the 3D map highlights it.

## Using it at the desk

- The search box auto-focuses. **Just start typing** — it searches English **and**
  Arabic names, organisations and titles at once, with fuzzy matching (typos OK).
- Click a guest card to open the big **status banner** and highlight their section on
  the 3D theatre map (the camera pans to it).
- Tap **Check in** on the card or banner — it turns green and records the time. Counts
  update in the header (Guests / Checked In / Zone A).
- Check-ins are saved in the browser (survive refreshes). **Reset** clears them all.
- The 3D map: **drag** to rotate, **scroll** to zoom, **Reset view** to recentre.

### Keyboard shortcuts
- `/` — focus the search box
- `Enter` — open the top result
- `Esc` — clear the search

## Notes & assumptions

- The source data is messy (names sometimes sit in the Organisation column, English in
  the Arabic field, multi-line cells, etc.). Search covers **every** text field so no
  one is unfindable, and the card shows the best available name.
- Check-in state is stored per-browser in `localStorage`. If you check people in on the
  desk laptop, keep using that same laptop/browser.
- The venue map is the real `odeonimage.jpeg` seating chart with an invisible `<svg>`
  overlay of polygon regions (one per group). Clicking a guest "lights up" their region
  (their group colour at 60% opacity) so the agent can point to the exact area. The
  polygons are in the image's native 1170×796 pixel space and the container uses
  `background-size: contain`, so overlay and image stay aligned. To nudge a region, edit
  its `points` in the `POLYS` array in `index.html`. It is for orientation, not exact seats.
