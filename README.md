# Travel Database

Geocoded place database built from Malla's Instagram saved travel collection.
Generated 2026-10-16.

## Source

- Instagram account: @poojith_malla (via Meta Accounts Center)
- Collection used: **"SF Places to visit"** (id `17892071370130481`) — the only
  travel-related saved collection found. There is **no "Travelling"/"Travel"
  collection** on the account, so all places come from "SF Places to visit".
  Other collections on the account: Life Tips, English songs, Outfits
  recommendation, Movie Recommendations, Life skills, Website recommendations,
  YouTube channels.
- Posts pulled: **489 unique** (collection header reported 494; 489 unique media
  IDs retrieved across 41 pages, no rate limiting hit).
- Raw posts: `posts.jsonl` — one JSON object per line with `post_id`,
  `username`, `post_caption`, `url`. (The Instagram API returned no location
  tags or timestamps for saved posts, so `post_created_at`/`location` are null.)

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
| `region` | state ("California" for all entries) |
| `country` | "United States" for all entries |
| `category` | `sf_bay` (SF Bay Area) or `california` (rest of California) |
| `lat`, `lng` | WGS-84 coordinates |
| `description` | one-sentence description |
| `mentions` | number of saved posts referencing the place |
| `urls` | up to 3 sample Instagram post URLs |

## Stats

- 185 places: **124 sf_bay**, **61 california**
- 0 international / non-California entries — the collection is entirely
  California-focused (no `usa`/`international` categories resulted).
- 358 of 396 curated post-mentions carry coordinates (90%).

## Caveats

- Some pins resolve to street/district or city level (e.g. small towns,
  neighborhoods) rather than exact addresses — acceptable for a travel map.
- Place names/descriptions come from Instagram captions; a few business
  entries could not be verified in the geocoder and were dropped.
- Descriptions are one-sentence summaries, not verbatim captions.
