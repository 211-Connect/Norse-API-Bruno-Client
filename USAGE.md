# Norse API — Usage Guide

How to **find resources** and **pull their details** through the Connect211 API
gateway. Every example below is a copy-paste `curl`. Pair this with the Bruno
collection in this repo — the folders map 1:1 to the recipes here.

> All IDs, names, phone numbers, and addresses in this guide are **placeholders**.
> Substitute values from your own search results and tenant.

## Setup

```bash
export SERVICES_APIKEY="<your-api-key>"        # see "Getting credentials" below
export TENANT_ID="<your-tenant-uuid>"          # your tenant UUID (provisioned with the key)
export BASE="https://services.c211.io/norse-api/v1"
```

### Getting credentials

To receive your **API key** and the correct **tenant ID(s)**, email
**help@connect211.com** with the subject line:

```
Developer API Key Provisioning - {tenant name}
```

CC **david@connect211.com** as well — especially for questions. Sending to
**help@connect211.com** is what matters, though: it opens a ticket so a human is
guaranteed to prioritize a follow-up to every request (even when David is busy).

Every request sends four headers:

| Header | Value | Notes |
|---|---|---|
| `X-API-Key` | `$SERVICES_APIKEY` | Missing/invalid → `401`. |
| `x-tenant-id` | `$TENANT_ID` | Required on every route except `/health` (even geocoding / short-url). Missing → `400 tenant_target_required`; a tenant your key lacks → `403 tenant_scope_denied`. |
| `x-api-version` | `1` | **Required.** Versioning is by header, not URL. Omit it and the route does not resolve: `404 Cannot GET /search`. |
| `accept-language` | `en`, `es`, … | Drives localized fields. |

> **`POST` needs write access.** The gateway treats every `POST` as a write and
> requires the *write* permission for the tenant on your key. Keys are
> read-only unless you ask for write when provisioning. A `403` on
> `POST /search`, `/resource/batch`, `/resource/titles` or `/short-url` while
> `GET` works is that, not a bug.

A reusable header array for the examples:

```bash
H=(-H "X-API-Key: $SERVICES_APIKEY" -H "x-tenant-id: $TENANT_ID" \
   -H "x-api-version: 1" -H "accept-language: en")
```

### Versions

Versioning is per-endpoint, selected by the `x-api-version` header. Today
**every endpoint is `1`** — send it on every request. (`/taxonomy` used to need
`2` for its slim `{ id, code, name }` shape; that is now what `1` returns and
`2` no longer exists.) If a new version ships for an endpoint you use, the
live OpenAPI document at
<https://services.c211.io/norse-api/v1/swagger> is where it will appear first.

---

## ⚠️ Read this first: the resource ID gotcha

A search hit contains several IDs. **The one you use to fetch details, favorite,
or batch-load is `service_at_location_id`** — *not* the hit's `id` / `_id`.

```
_source.service_at_location_id   ✅  GET /resource/{this}      → 200
_source.id                       ❌  GET /resource/{this}      → 404
_source.service_id               ❌                            → 404
```

Treat `service_at_location_id` as *the* resource identifier throughout your
integration. (`GET /resource/original/{originalId}` is the one alternate entry —
see [Recipe 3](#recipe-3--get-full-details).)

---

## The two resource shapes

The API returns a resource in **two different shapes** depending on the endpoint:

- **Search hit** (`GET /search` → `_source`): Elasticsearch-flavored, `snake_case`.
  Good for result cards. Fields: `id`, `service_at_location_id`, `service_id`,
  `name`, `description`, `summary`, `phone`, `url`, `email`, `schedule`,
  `minimum_age` / `maximum_age`, `taxonomies[]`, `taxonomy_names`,
  `attribute_values`, nested `service` / `location` / `organization`.

- **Resource detail** (`GET /resource/{id}`): richer, `camelCase`, with a nested
  localized `translation{}` block. Fields: `_id`, `serviceAtLocationId`,
  `originalId`, `displayName`, `displayPhoneNumber`, `website`,
  `organizationUrl`, `email`, `organizationName`, `organizationAlternateName`,
  `locationName`, `location{type,coordinates}`, `addresses[]`, `phoneNumbers[]`,
  `languages[]`, `serviceArea` + `serviceAreaName`, `lastAssuredDate`,
  `attribution`, `facetsEn[]`, and `translation{}` (which carries the localized
  copy plus `contacts[]`, `phoneNumbers[]`, `taxonomies[]`, `facets[]`,
  `eligibilities`, `fees`, `hours`, …). **Prefer `translation.*`** (localized)
  over the top-level fields when present.

Two more record shapes exist for the *provider* side — the HSDS
**organization** (`GET /organization/{id}`) and **service**
(`GET /service/{id}`) records — see [Recipe 7](#recipe-7--organizations--service-records).

---

## The core flow

```
(1) typeahead        (2) search              (3) detail
GET /taxonomy  ─────► GET /search   ─────►    GET /resource/{service_at_location_id}
(x-api-version: 1)    (x-api-version: 1)       (x-api-version: 1)
   pick a code           hits[]._source           full localized record
```

---

## Recipe 1 — Find resources (search)

**Use `query_type=hybrid`.** It blends semantic (vector) and keyword matching in a
**single `/search` call** — no extra endpoints, no classification step — and
handles both natural language ("I need help paying rent") and plain keywords
("food"). This is the right default for any search box:

```bash
curl -s "${H[@]}" \
  "$BASE/search?query=i%20need%20help%20paying%20rent&query_type=hybrid&page=1&limit=25&sort=relevance"
```

Response (trimmed) — note the **double `hits`** nesting and the id fields:

```jsonc
{
  "search": {
    "hits": {
      "total": { "value": 1852, "relation": "eq" },
      "hits": [
        {
          "_id": "aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa",
          "_source": {
            "id": "bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb",              // NOT the detail id
            "service_at_location_id": "aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa", // ✅ use THIS for details
            "name": "Example Community Service",
            "phone": "555-0100",
            "url": "https://www.example.org",
            "email": "info@example.org",
            "location": { "physical_address": { "city": "Anytown", "state": "CA", "...": "" } }
          }
        }
      ]
    }
  },
  "facets": []
}
```

- Total matches: `search.hits.total.value`. Result array: `search.hits.hits[]`.
- `limit` must be **25–300** (a smaller value errors). Paginate with `page`;
  `page * limit` may not exceed **10,000** (with `limit=25`, the last page is 400).
- `query_type`: **`hybrid`** (recommended) · `text` · `taxonomy` · `more_like_this`.
- `sort`: `relevance` (default) · `distance` · `name` · `organization`.

### Too many results? `relevance_cutoff=on` (opt-in)

A broad query can match well over a thousand resources, which is more than
anyone will review. `relevance_cutoff=on` trims results that score far below
the best match (hybrid search only):

```bash
curl -s "${H[@]}" \
  "$BASE/search?query=food&query_type=hybrid&relevance_cutoff=on&limit=25"
```

```jsonc
{
  "search": { "hits": { "total": { "value": 18 }, "hits": [ /* ... */ ] } },
  "relevance_cutoff": {
    "applied": true,
    "reason": null,
    "kept": 18,
    "matched_before_cutoff": 1234,
    "cutoff_score": 81.5,
    "candidates_examined": 18
  }
}
```

**This changes what `hits.total.value` means, but only when you ask for it.**
With a cutoff applied it is the *kept* count (18 above); the number the query
actually matched moves to `relevance_cutoff.matched_before_cutoff` (1,234), so
you can still show "18 of 1,234". Omit the param (or send `off`) and everything
is exactly as documented above — same totals, and no `relevance_cutoff` key in
the response.

How `on` decides: it keeps results scoring at least 20% of the top score
(tightened stepwise up to 50% when more than 1,000 results would survive) and
never removes the top 20. `applied: false` is a normal outcome, not an error,
and `reason` says why:

| `reason` | Meaning |
|---|---|
| `no_elbow` | The scores are flat — nothing scored meaningfully below the threshold, so returning everything is the honest answer. |
| `below_min_keep` | The matched set is already small. |
| `cut_too_large` | A cut point was found but would keep more than 1,000 results, which the API will not enumerate. Nothing is trimmed. |

`cutoff_score` is on a different scale from `hits[]._score` (it excludes the
distance and pinning terms), so compare it across responses, never against a
hit's `_score`. The cut is computed on semantic/lexical relevance only.
Proximity still filters and ranks, but never decides relevance: how far
someone will travel is their choice, not a property of the resource.

For a vague natural-language need where you'd rather *classify* the intent and
possibly ask the user to clarify before searching, see the
[AI need-classification flow](#recipe-6--ai-need-classification--the-clarification-workflow-advanced).

> **Building something new?** `GET` and `POST /search` are equivalent for basic
> search, but new geographic and custom-filtering features arrive through the
> `POST` **request body** — so prefer `POST` for new integrations. See
> [Recipe 5](#recipe-5--geographic-search).

### Exact / agency lookups: `query_type=text`

Use `query_type=text` for literal keyword matching — e.g. pulling services for a
specific agency by name:

```bash
curl -s "${H[@]}" \
  "$BASE/search?query=Example%20Nonprofit&query_type=text&page=1&limit=25"
```

> **Heads up:** `text` is being gradually decommissioned in favor of the
> continuously improving `hybrid` semantic search which will eventually 
> incorporate NLP techniques which will far outpace the performance of `text` search. 
> Prefer `hybrid` for new integrations; keep `text` only for exact-match 
> / known-agency lookups.

### Filters & age

```bash
# filters is bracket-notation: filters[<facetKey>][<i>]=<value>
# NOTE: pass -g so curl doesn't try to glob the [ ] in the URL.
curl -g -s "${H[@]}" \
  "$BASE/search?query=food&limit=25&filters[language][0]=Spanish&age=30"
```

---

## Recipe 2 — Typeahead & category (taxonomy) search

Norse is a taxonomy-driven directory (HSIS codes like `BH-3700`). Two steps:

**a) Typeahead** as the user types:

```bash
curl -s "${H[@]}" "$BASE/taxonomy?query=housing&page=1"
```

```jsonc
{ "total": 40, "page": 1,
  "items": [ { "id": "cccccccc-cccc-cccc-cccc-cccccccccccc", "code": "BH-3700", "name": "Housing Counseling" } ] }
```

**b) Search by the chosen code(s)** — put the code(s) in **`query`** (comma-separated)
with **`query_type=taxonomy`**:

```bash
curl -s "${H[@]}" \
  "$BASE/search?query=BH-3700&query_type=taxonomy&page=1&limit=25"
```

> The codes go in **`query`**, not the `taxonomy=` param. `query_type=taxonomy`
> tells the API to treat `query` as HSIS codes. (The separate `taxonomy=` param
> is a *scope filter* layered on a text/hybrid query — see the
> [parameter reference](#search-parameter-reference-search-get--post).)

`GET /suggestion?query=food` is a related helper that returns
`{ taxonomies[], organizations[] }` for mixed typeahead — the same taxonomy
items plus `{ organization_id, name, city, state }` organization matches
(only for tenants with organization search enabled; otherwise `[]`). See
[Recipe 7](#recipe-7--organizations--service-records) for what to do with an
`organization_id`.

---

## Recipe 3 — Get full details

**By `service_at_location_id`** (the id from a search hit):

```bash
SAL="aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"   # a service_at_location_id from a search hit
curl -s "${H[@]}" "$BASE/resource/$SAL?locale=en&tenant_id=$TENANT_ID"
```

```jsonc
{
  "_id": "aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa",   // == service_at_location_id
  "originalId": "1000123",
  "displayName": "Example Community Service",
  "displayPhoneNumber": "555-0100",
  "website": "https://www.example.org",
  "email": "info@example.org",
  "organizationName": "Example Nonprofit",
  "location": { "type": "Point", "coordinates": [-73.9857, 40.7484] },
  "addresses": [ { "address_1": "100 Example Ave", "address_2": "Suite 200",
                   "city": "Anytown", "stateProvince": "CA", "postalCode": "90000",
                   "rank": 1, "type": "physical" } ],
  "phoneNumbers": [ { "number": "555-0100", "type": "voice", "rank": 0, "description": "Main" } ],
  "lastAssuredDate": "2025-01-15T00:00:00Z",
  "translation": { "displayName": "...", "serviceName": "...", "serviceDescription": "...",
                   "hours": "...", "fees": "...", "eligibilities": "...", "taxonomies": [ ] }
}
```

**By original (source) ID** — for deep links from an external system:

```bash
curl -s "${H[@]}" "$BASE/resource/original/1000123?locale=en&tenant_id=$TENANT_ID"
```

Resolves to the same record (its `_id` is the canonical `service_at_location_id`).

- `location.coordinates` is **`[longitude, latitude]`**.
- `phoneNumbers[].type` is `voice` / `fax`; the primary is usually `rank: 0/1`.
- Localized copy lives under `translation.*`; fall back to the top-level fields.

---

## Recipe 4 — Batch load & titles

For hydrating saved/favorited IDs or building lists. Both are `POST`, accept
`{ "ids": [...] }` (**1–100** `service_at_location_id`s), and are keyed by that id.

**Full records (partial success):**

```bash
curl -s "${H[@]}" -H "Content-Type: application/json" -X POST \
  -d '{"ids":["aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa","00000000-0000-0000-0000-000000000000"]}' \
  "$BASE/resource/batch?tenant_id=$TENANT_ID&locale=en"
```

```jsonc
{
  "data":   { "aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa": { "_id": "aaaaaaaa-...", "displayName": "..." } },
  "errors": [ { "id": "00000000-0000-0000-0000-000000000000", "reason": "Resource not found", "statusCode": 404 } ],
  "meta":   { "requested": 2, "successful": 1, "failed": 1 }
}
```

**Display titles only** (lightweight, e.g. list headers):

```bash
curl -s "${H[@]}" -H "Content-Type: application/json" -X POST \
  -d '{"ids":["aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"]}' \
  "$BASE/resource/titles?tenant_id=$TENANT_ID&locale=en"
# → [ { "id": "aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa", "displayName": "Example Community Service" } ]
```

---

## Recipe 5 — Geographic search

Norse can scope results to *where* someone is or *what area* they care about.
Two search shapes cover almost everything:

| You have… | Use | How |
|---|---|---|
| A **point + radius** ("within 5 mi of me") | **Proximity** | `coords` + `distance`, `sort=distance` |
| An **area / region** (a city, county, ZIP, or a map viewport / drawn shape) | **Boundary** | `geo_type=boundary` + a GeoJSON `Polygon` in the POST body |
| No location | Plain search | omit all geo params |

> ### 🔭 New integrations: prefer `POST /search`
> `GET` and `POST /search` accept the same query params and return the same
> shape, but **only `POST` carries a JSON body**. (`POST` needs the *write*
> permission on your key — see [Setup](#setup).) Geographic capability is moving
> into that body: today it takes a GeoJSON `geometry` for boundary search, and it
> will increasingly accept **richer GeoJSON for custom area filtering and other
> features that can't fit in a query string or header**. Build new integrations
> on `POST` so you inherit those features without re-plumbing. Keep the geo params
> (`coords`, `distance`, `geo_type`) on the URL; put geometry in the body.

### Step 0 — Geocoding (turn an address/place into coordinates)

You almost never have coordinates to start with — you have a typed address, city,
or ZIP. **Geocode first, then search.**

**Forward** (address/place → coordinates + bounding box):

```bash
curl -s "${H[@]}" \
  "$BASE/geocoding/forward?address=Brooklyn%2C%20New%20York&provider=mapbox&limit=1"
```

```jsonc
[
  {
    "type": "coordinates",
    "address": "Brooklyn, New York, United States",
    "coordinates": [-73.9497, 40.6526],                 // [lng, lat] — feed to `coords`
    "place_type": ["locality"],                          // country | region | postcode | district | place | locality | neighborhood | address | poi
    "bbox": [-74.042412, 40.566162, -73.833365, 40.739446], // [minLng, minLat, maxLng, maxLat] — feed to Boundary
    "postcode": "11215", "place": "New York City",
    "district": "Kings County", "region": "New York", "country": "United States"
  }
]
```

- `provider`: `mapbox` (default) or `opencage`. `limit`: 1–10.
- Mapbox only: `types=address,poi` restricts feature types (`country`, `region`,
  `postcode`, `district`, `place`, `locality`, `neighborhood`, `address`, `poi`;
  a `400` with OpenCage), and `proximity=lng,lat` biases results toward a point
  (ignored by OpenCage).
- `coordinates` is **`[lng, lat]`** — use it directly for proximity `coords`.
- `place_type` tells you the granularity. A **point-like** result
  (`address`/`postcode`) → use **proximity**. An **area-like** result
  (`locality`/`region`/`country`) → use **boundary** with its `bbox`.

**Reverse** (coordinates → nearest address) — for "search near my current GPS
location," to show the user a human-readable place, or to normalize a dropped map
pin:

```bash
curl -s "${H[@]}" \
  "$BASE/geocoding/reverse?coordinates=-73.9857%2C40.7484&provider=mapbox"
```

### Proximity — point + radius

When you have a specific point (device GPS, a geocoded street address, a map pin)
and want the nearest services. Pass `coords` as **`longitude,latitude`** and
`distance` in **miles**; pair with `sort=distance`:

```bash
# Recommended: POST (body reserved for upcoming geo features), coords on the URL
curl -s "${H[@]}" -H "Content-Type: application/json" -X POST \
  "$BASE/search?query=food&coords=-73.9857,40.7484&distance=5&sort=distance&page=1&limit=25&tenant_id=$TENANT_ID&locale=en" \
  -d '{}'

# Equivalent via GET (fine for quick/read-only use)
curl -s "${H[@]}" \
  "$BASE/search?query=food&coords=-73.9857,40.7484&distance=5&sort=distance&limit=25"
```

> ⚠️ `coords` order is **lng,lat** (GeoJSON convention). Passing `lat,lng`
> silently returns **zero results** — a common integration bug. `geocoding`
> output is already `[lng, lat]`, so pass it straight through.

> **`sort=distance` drops relevance entirely.** Hits come back nearest-first
> with `_score: null`, so a broad query can lead with barely-related records
> that happen to be close. Under `sort=relevance` (hybrid) distance is only a
> small part of the score, so the nearest provider can sit deep in the list.
> If you want *relevant, then nearest*: with `query_type=hybrid`, pair
> `relevance_cutoff=on` with `sort=distance` — the cutoff is applied as a
> relevance floor first, and the survivors are then ordered by distance.

### Boundary — search within an area (polygon)

When results should be constrained to a region rather than a radius — a city /
county / ZIP, a map viewport, or a user-drawn shape. Send `geo_type=boundary` on
the URL and a GeoJSON **`Polygon`** in the POST body (a closed ring — first point
repeated as the last):

```bash
curl -s "${H[@]}" -H "Content-Type: application/json" -X POST \
  "$BASE/search?query=food&geo_type=boundary&page=1&limit=25&tenant_id=$TENANT_ID&locale=en" \
  -d '{"geometry":{"type":"Polygon","coordinates":[[
        [-74.05,40.68],[-73.90,40.68],[-73.90,40.82],[-74.05,40.82],[-74.05,40.68]]]}}'
```

**From a geocoded area:** convert the `bbox` (`[minLng, minLat, maxLng, maxLat]`)
into a rectangle polygon:

```
bbox = [minLng, minLat, maxLng, maxLat]
Polygon = [[ [minLng,minLat], [maxLng,minLat], [maxLng,maxLat], [minLng,maxLat], [minLng,minLat] ]]
```

For a real municipal boundary or a hand-drawn map selection, pass that shape's
GeoJSON `Polygon` directly instead of a bbox rectangle.

---

## Recipe 6 — AI need-classification & the clarification workflow (advanced)

> For most "AI search" needs, prefer **`query_type=hybrid`**
> ([Recipe 1](#recipe-1--find-resources-search)) — one call, no follow-up.
> Use this flow when you want the ML broker to *classify* a natural-language
> query into needs and, **when the input is ambiguous, ask the user a follow-up
> question before searching** — the behavior Connect211's own app implements.

### Step 1 — Classify the query

```bash
curl -s "${H[@]}" "$BASE/search/predict?query=i%20need%20help%20with%20rent%20and%20food&top_k=150"
```

```jsonc
{
  "scenario": "clarify_multiple_labels",
  "hsis_taxonomies": ["BH-8400.3000", "YV-6500", "..."],   // best-guess HSIS codes (may be empty)
  "options": [                                             // candidate NEEDS, sorted by score desc
    { "code": "HO-300", "score": 0.9589, "pre_selected": true,  "results_count": 959  },
    { "code": "EM-180", "score": 0.7108, "pre_selected": true,  "results_count": 508  },
    { "code": "FO-200", "score": 0.4762, "pre_selected": false, "results_count": 1088 },
    { "code": "IC-330", "score": 0.1896, "pre_selected": false, "results_count": 318  }
  ]
}
```

### Step 2 — Branch on `scenario`

The `scenario` tells you **whether to ask a follow-up question or just search**:

| `scenario` | Meaning | What to do |
|---|---|---|
| `search` | Confident, single interpretation. | **Search now** with `hsis_taxonomies`. No follow-up. |
| `search_and_notify_low_confidence` | Searched, but the model isn't sure. | **Search now**; show a non-blocking "we weren't sure — refine?" notice. |
| `search_and_notify_low_info` | Query too sparse to classify well (`hsis_taxonomies` often empty). | **Search now** (broad / keyword); invite the user to add detail. |
| `clarify_low_info` | Too little info to search confidently. | **Ask a follow-up** — show `options[]`, let the user pick a need. |
| `clarify_multiple_labels` | **More than one distinct need detected** (e.g. rent *and* food). | **Ask a follow-up** — let the user pick which need(s) they mean. |

So the rule is simple: **`clarify_*` → ask the user; everything else → search immediately** (the `*_notify_*` variants just add an advisory message).

### Step 3 — Ask the follow-up question (`clarify_*` only)

Present `options[]` as the choices. Each option is a **candidate need**:

| Field | Use in your follow-up UI |
|---|---|
| `code` | The need identifier (a top-level need code, e.g. `HO-300`). **The API returns no display label** — you supply it (see note below). |
| `score` | Model confidence `0–1`. Options come sorted high→low. Connect211's app shows those with `score > 0.1` (top 4, with a "show more"). |
| `pre_selected` | **Check this option by default.** |
| `results_count` | Resources available for that need (or `null`). Show as "(959 results)"; consider hiding `0`-result needs. |

> **Labeling the options:** option `code`s are *need categories*, not HSIS terms —
> they don't resolve through `/taxonomy/term`, and `predict` returns only the code.
> Maintain your own `code → label` map for the follow-up prompt (Connect211's app
> keeps a "needs" label table). **Ask help@connect211.com for the current need-code
> list** for your tenant. If you have no label, fall back to showing the code.

Let the user confirm/adjust the selection (start from the `pre_selected` ones).

### Step 4 — Re-rank on the selection, then search

Turn the chosen needs into `need_weights` and call `re-rank` to get the ordered
HSIS codes to search with. Connect211's app weights them like this:

| Selection state | Weight |
|---|---|
| Selected **and** was `pre_selected` | its `score` |
| Selected **and** not `pre_selected` | `0.6` |
| Not selected but was `pre_selected` | `0.1` (kept, down-weighted) |

```bash
# user kept Housing (was pre_selected) + added Food; Employment left unpicked
curl -s "${H[@]}" \
  "$BASE/search/re-rank?need_weights=%7B%22HO-300%22%3A0.9589%2C%22FO-200%22%3A0.6%2C%22EM-180%22%3A0.1%7D&top_k=150"
# → { "hsis_taxonomies": ["BH-1800.1500-330", "BH-1800", "..."] }
```

Then search with those codes — codes go in **`query`** with `query_type=taxonomy`:

```bash
curl -s "${H[@]}" \
  "$BASE/search?query=BH-1800.1500-330,BH-1800&query_type=taxonomy&page=1&limit=25"
```

For the **auto-search** scenarios (Step 2), skip Steps 3–4 and go straight to
`GET /search?query=<hsis_taxonomies>&query_type=taxonomy`.

### Prefer to skip the follow-up entirely? (simpler integrations)

You don't have to build the clarify UI. From simplest to most tailored:

1. **Skip `predict` altogether — use `query_type=hybrid`.** One call, natural
   language in, ranked results out ([Recipe 1](#recipe-1--find-resources-search)).
   This is the recommended default and needs no follow-up logic.

2. **Call `predict`, then just use its `hsis_taxonomies`** — the model's overall
   best guess across every need it detected — and ignore `scenario`/`options`:

   ```bash
   # one predict + one search, no follow-up, even for a clarify_* result
   curl -s "${H[@]}" \
     "$BASE/search?query=BH-8400.3000,YV-6500,BH-0500.7000&query_type=taxonomy&limit=25"
   ```

3. **Auto-select the highest-likelihood need** instead of asking. Take the top
   option (`options[0]`, or the first `pre_selected`), re-rank on just that one,
   and search — committing to a single need rather than the blended guess:

   ```bash
   # re-rank on the single top need, then search the returned codes
   curl -s "${H[@]}" \
     "$BASE/search/re-rank?need_weights=%7B%22HO-300%22%3A1%7D&top_k=150"
   # → { "hsis_taxonomies": [...] }  → GET /search?query=<those codes>&query_type=taxonomy
   ```

The follow-up UI exists to *disambiguate* when the model is unsure or sees several
needs; auto-picking trades that precision for a one-shot flow. If you only want
simplicity, option 1 (`hybrid`) is almost always the right call.

### The whole workflow at a glance

```
GET /search/predict
        │
        ├─ scenario = search / search_and_notify_*  ─────────────► GET /search?query=<hsis_taxonomies>&query_type=taxonomy
        │                                                          (notify_* → also show an advisory)
        │
        └─ scenario = clarify_low_info / clarify_multiple_labels
                 │  show options[] (pre-check pre_selected)
                 ▼  user answers the follow-up
           GET /search/re-rank?need_weights={code:weight}  ─────► GET /search?query=<hsis_taxonomies>&query_type=taxonomy

  (Simpler: skip predict and use query_type=hybrid; or use predict's
   hsis_taxonomies directly without the follow-up — see below.)
```

---

## Recipe 7 — Organizations & service records

Search hits are *service-at-location* records. The provider side has its own
endpoints when you need the whole organization or the whole service.

**a) Find an organization** (prefix typeahead, every org in the tenant):

```bash
curl -s "${H[@]}" "$BASE/organization?query=food%20bank&page=1&limit=10"
```

```jsonc
{ "total": 3, "page": 1, "limit": 10,
  "hits": [ { "_id": "…", "_score": 12.1,
              "_source": { "organization_id": "dddddddd-dddd-dddd-dddd-dddddddddddd",
                           "name": "Example Food Bank", "alternate_name": null,
                           "email": "info@example.org", "website": "https://example.org",
                           "phone": "555-0100",
                           "location": { "address_1": "100 Example Ave", "city": "Anytown",
                                         "state": "CA", "postal_code": "90000" } } } ] }
```

**b) List or search that organization's resources** — `organization_id` is a
scope on `/search` that composes with any `query_type` (an empty `query` lists
everything):

```bash
ORG="dddddddd-dddd-dddd-dddd-dddddddddddd"
curl -s "${H[@]}" "$BASE/search?query=&organization_id=$ORG&limit=25"
curl -s "${H[@]}" "$BASE/search?query=pantry&query_type=hybrid&organization_id=$ORG&limit=25"
```

**c) The full organization record** (HSDS-shaped, UPPER_CASE nested keys:
`phones[]`, `contacts[]`, `locations[]`, `services[]` each with
`SERVICE_AT_LOCATIONS[]`):

```bash
curl -s "${H[@]}" "$BASE/organization/$ORG"
```

`services[].SERVICE_AT_LOCATIONS[].ID` is a `service_at_location_id` — feed it
to `GET /resource/{id}` or `POST /resource/batch` for the search-facing document.

**d) From a resource back to its organization** — when all you have is a
`service_at_location_id`:

```bash
curl -s "${H[@]}" "$BASE/organization/service-at-location/$SAL"
# 404 → no org in this tenant carries it · 409 → more than one does (never a guess)
```

**e) The HSDS service record** — one service across *all* its locations, by
`service_id` (from a search hit's `_source.service_id`; not the
service-at-location id):

```bash
SERVICE_ID="bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb"   # _source.service_id from a search hit
curl -s "${H[@]}" "$BASE/service/$SERVICE_ID"
```

Returns `serviceId`, `name`, `description`, `eligibilityDescription`,
`applicationProcess`, `minimumAge`/`maximumAge`, `organization{}`,
`locations[]`, `phones[]`, `schedules[]`, `languages[]`, `serviceAreas[]`,
`costOptions[]`, `requiredDocuments[]`, `taxonomyCodes[]`, `taxonomyPath[]`.
Nested `TRANSLATIONS` arrays on both records are filtered to `accept-language`,
falling back to English, then the canonical locale.

---

## Search parameter reference (`/search`, GET & POST)

| Param | Type | Default | Notes |
|---|---|---|---|
| `query` | string | `""` | Free text; **or comma-separated HSIS codes when `query_type=taxonomy`** (that's how you search *by* codes). |
| `query_type` | enum | `text` | **`hybrid`** recommended (semantic + keyword). Also `text` (exact/agency, being phased out) · `taxonomy` (treats `query` as HSIS codes) · `more_like_this`. |
| `page` | int ≥1 | `1` | 1-based. `page * limit` ≤ 10,000. |
| `limit` | int | `25` | **25–300** (below 25 errors). |
| `taxonomy` | string/CSV | — | Optional **scope filter** to narrow a text/hybrid search to these HSIS codes (e.g. `BM-1400,BM-1700`). To search purely *by* codes, use `query`+`query_type=taxonomy` instead. |
| `organization_id` | string | — | Scope results to one organization by its stable id (from `/organization` or `/suggestion`). Composes with any `query_type` and every other param. |
| `coords` | string | — | **`lng,lat`**, e.g. `-73.9857,40.7484`. |
| `distance` | int (mi) | `0` | Only with `coords`. |
| `geo_type` | enum | — | `proximity` · `boundary` (boundary → POST body geometry). |
| `filters` | bracket obj | `{}` | `filters[<key>][<i>]=<value>`. |
| `age` | int ≥0 | — | Age eligibility filter. |
| `sort` | enum | `relevance` | `relevance` · `distance` · `name` · `organization`. |
| `relevance_cutoff` | enum | `off` | **Opt-in**, hybrid only. `on` keeps results scoring ≥ 20% of the top score (never trimming the top 20) and returns everything when the scores are flat. Adds a `relevance_cutoff` block to the response and makes `hits.total` the *kept* count. |

---

## Versioning & errors

- **Version by header** (`x-api-version`): `1` for every endpoint (see
  [Versions](#versions)).
- `401` — missing/invalid `X-API-Key`.
- `400` — missing/invalid `x-tenant-id` (`tenant_target_required` /
  `tenant_target_invalid`), or a bad param (e.g. `limit` < 25, `page * limit`
  > 10,000, `geo_type=boundary` on `GET`).
- `403 tenant_scope_denied` — your key is not provisioned for that tenant, or
  it is read-only and you sent a `POST`.
- `404 Cannot GET /search` (or any route) — you omitted `x-api-version`. The
  header is required; without it the route does not resolve at all.
- `404` on `/resource/{id}` — resource not found — **most often because you
  used `_source.id` instead of `service_at_location_id`** (see the gotcha
  above).
- `409` — `GET /organization/service-at-location/{id}` found more than one
  organization carrying that id in the tenant.

---

## Endpoint ↔ Bruno folder map

| Task | Endpoint | Bruno folder |
|---|---|---|
| Find (text/taxonomy/geo) | `GET`/`POST` `/search` | `01. Search` |
| Get details | `GET /resource/{id}`, `/resource/original/{id}` | `02. Resource` |
| Batch / titles | `POST /resource/batch`, `/resource/titles` | `02. Resource` |
| Typeahead codes | `GET /taxonomy`, `/suggestion` | `03. Taxonomy`, `04. Suggestion` |
| Org typeahead / record | `GET /organization`, `/organization/{id}`, `/organization/service-at-location/{salId}` | `05. Organization` |
| Service record | `GET /service/{id}` | `08. Service` |
| Geocode | `GET /geocoding/forward` · `/reverse` | `06. Geocoding` |
| AI need → codes | `GET /search/predict` · `/re-rank` | `01. Search` |
