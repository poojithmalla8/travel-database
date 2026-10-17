# Travel Database

Geocoded place database built from Malla's Instagram saved travel collections.
Generated 2026-10-16, extended same day with a second collection.

## Sources

- Instagram account: @poojith_malla (via Meta Accounts Center)
- Collection 1: **"SF Places to visit"** (id `17892071370130481`).
  Posts pulled: **489 unique** (collection header reported 494; 489 unique
  media IDs retrieved across 41 pages, no rate limiting hit).
  Raw posts: `posts.jsonl` — one JSON object per line with `post_id`,
  `username`, `post_caption`, `url`. (The Instagram API returned no location
  tags or timestamps for saved posts, so `post_created_at`/`location` are
  null.)
- Collection 2: **"Traveling"** (id `17934681652812254`) — found via a direct
  link from the user; it did not appear in the first page of the collection
  listing (it is on page 2). Posts pulled: **350 unique** (collection header
  reported 376; 350 unique media IDs across 30 pages, no rate limiting hit).
  Raw posts: `traveling_posts.jsonl` (same schema as `posts.jsonl`).
- Pre-merge backup of the v1 database: `travel-db.backup-2026-10-16.json`
  (185 places, "SF Places to visit" only).

## Pipeline

1. `pull_posts.py` — paginated `saved-posts` for the collection, cursor-safe,
   wrote `posts.jsonl`.
2. `extract_candidates.py` — extracted 📍/📌 pin lines from captions
   (260 posts had them) → 444 raw candidates.
3. `merge_candidates.py` — junk/CTA filtering + fuzzy dedupe → 350 merged
   candidates.
4. `curate.py` + `build_db.py` — manual curation into named places
   (merges, drops of vague/CTA/event entries) → 211 unique places.
5. Geocoding via `map.geocode` (4 parallel workers + manual retry pass for
   failures; never guessed coordinates) → **185 places with coordinates**.
   26 candidates dropped for lack of a confident geocode match.

## `travel-db.json`

JSON array, one object per place, sorted by `mentions` descending:

| field | meaning |
|---|---|
| `name` | curated place name |
| `city` | city (best effort) |
| `region` | state / province |
| `country` | country |
| `category` | `sf_bay` (SF Bay Area), `california` (rest of California), `usa` (other US states), `international` (other countries) |
| `lat`, `lng` | WGS-84 coordinates |
| `description` | one-sentence description |
| `mentions` | number of saved posts referencing the place (both collections combined) |
| `urls` | up to 3 sample Instagram post URLs |
| `source` | `sf_places` (only in "SF Places to visit"), `traveling` (only in "Traveling"), `both` (in both) |

## Second-collection merge ("Traveling", 2026-10-16)

1. `pull_traveling.py` — paginated `saved-posts` for collection
   `17934681652812254` → `traveling_posts.jsonl` (350 unique posts).
2. `extract_traveling.py` — 📍/📌 pin-line extraction → 335 raw candidates
   (`traveling_candidates.json`); `merge_traveling.py` — junk filter + fuzzy
   dedupe → 302 merged candidates (`traveling_merged.json`).
3. Name dedupe vs v1 DB → **47 candidates overlapped** existing entries;
   their 57 mentions + post URLs were merged into the existing entries
   (`traveling_overlaps.json`), which got `source: "both"`.
4. The 255 non-overlapping candidates were curated (`curate_traveling.py`;
   decision table kept in the script, junk drops in `dropped_candidates.json`):
   87 dropped as junk (CTAs, addresses, hike-stat lines, vague entries,
   promos, "Taco Bell, Weed" joke), 158 kept and geocoded via `map.geocode`
   (raw results in `geocoded.jsonl`); 14 dropped for no confident match
   (Ashley Gorge, Lake Aloha, Mars Desert Research Station, Coyote Creek Cave,
   Portuguese Beach, Donnerkogel, Tianmen Mountain, Astoria–Megler Bridge,
   Fremont Lookout, Arakurayama Sengen Park, Miradouro Água d'Alto,
   Mooney Falls, Lone Rock Beach, Los Portales de García).
   → `traveling_new_places.json` (144 geocoded new places).
5. Coordinate dedupe: 7 new places fell within 1 km of an existing entry and
   were merged into it instead (e.g. Marymere Falls + Mt. Storm King →
   Lake Crescent; Eagle Falls → Vikingsholm; Vernal Fall → Mist Trail).
   When in doubt, mentions were merged rather than duplicating.
6. 137 genuinely new entries appended with `source: "traveling"`;
   all v1-only entries got `source: "sf_places"`; array re-sorted by
   mentions descending.

## Stats

- **322 places** (was 185): **125 sf_bay**, **93 california**, **88 usa**,
  **16 international** — the "Traveling" collection added the first
  non-California coverage (Pacific Northwest cluster, Utah/Arizona/Colorado,
  plus Switzerland, Madeira, Iceland, Laos, France, Canada, Australia,
  Scotland).
- Sources: 146 `sf_places`, 136 `traveling`, 40 `both`.
- 604 total post-mentions (was 358).

## Caveats

- Some pins resolve to street/district or city level (e.g. small towns,
  neighborhoods) rather than exact addresses — acceptable for a travel map.
- Place names/descriptions come from Instagram captions; a few business
  entries could not be verified in the geocoder and were dropped.
- Descriptions are one-sentence summaries, not verbatim captions.
