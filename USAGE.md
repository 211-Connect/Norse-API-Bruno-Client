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
| `x-tenant-id` | `$TENANT_ID` | Required on tenant-scoped routes. Missing → `400`. |
| `x-api-version` | latest per endpoint (see below) | **Versioning is by header, not URL.** |
| `accept-language` | `en`, `es`, … | Drives localized fields. |

A reusable header array for the examples:

```bash
H=(-H "X-API-Key: $SERVICES_APIKEY" -H "x-tenant-id: $TENANT_ID" \
   -H "x-api-version: 1" -H "accept-language: en")
```

### Always use the latest version an endpoint supports

Versioning is per-endpoint, selected by the `x-api-version` header. Send the
**highest version the endpoint offers** — newer versions return cleaner, more
stable shapes.

| Endpoint | Version to use |
|---|---|
| `/taxonomy` | **v2** |
| `/search`, `/resource`, `/suggestion`, `/organization`, `/geocoding` | v1 |

Today, **taxonomy uses v2**; every other endpoint is v1-only, so v1 *is* the
latest there. If a new version ships for an endpoint you use, move to it.

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
  localized `translation{}` block. Fields: `_id`, `originalId`, `displayName`,
  `displayPhoneNumber`, `website`, `organizationUrl`, `email`, `organizationName`,
  `locationName`, `location{type,coordinates}`, `addresses[]`, `phoneNumbers[]`,
  `contacts[]`, `languages[]`, `serviceArea` + `serviceAreaName`, `minimumAge` /
  `maximumAge`, `lastAssuredDate`, `attribution`, `facetsEn[]`, and
  `translation{}`. **Prefer `translation.*`** (localized) over the top-level
  fields when present.

---

## The core flow

```
(1) typeahead        (2) search              (3) detail
GET /taxonomy  ─────► GET /search   ─────►    GET /resource/{service_at_location_id}
(x-api-version: 2)    (x-api-version: 1)       (x-api-version: 1)
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
- `limit` must be **25–300** (a smaller value errors). Paginate with `page`.
- `query_type`: **`hybrid`** (recommended) · `text` · `taxonomy` · `more_like_this`.
- `sort`: `relevance` (default) · `distance` · `name` · `organization`.

For a vague natural-language need where you'd rather *classify* the intent and
possibly ask the user to clarify before searching, see the
[AI need-classification flow](#recipe-6--ai-need-classification-advanced).

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

**a) Typeahead** as the user types — use **`x-api-version: 2`** for the clean list:

```bash
curl -s -H "X-API-Key: $SERVICES_APIKEY" -H "x-tenant-id: $TENANT_ID" \
     -H "x-api-version: 2" -H "accept-language: en" \
     "$BASE/taxonomy?query=housing&page=1"
```

```jsonc
{ "total": 40, "page": 1,
  "items": [ { "id": "cccccccc-cccc-cccc-cccc-cccccccccccc", "code": "BH-3700", "name": "Housing Counseling" } ] }
```

**b) Search by the chosen code(s)** — pass `taxonomy` (comma-separated) and set
`query_type=taxonomy`:

```bash
curl -s "${H[@]}" \
  "$BASE/search?taxonomy=BH-3700&query_type=taxonomy&page=1&limit=25"
```

`GET /suggestion?query=food` is a related helper that returns
`{ taxonomies[], organizations[] }` for mixed typeahead.

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
> shape, but **only `POST` carries a JSON body**. Geographic capability is moving
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
    "place_type": ["locality"],                          // address | postcode | locality | region | country
    "bbox": [-74.042412, 40.566162, -73.833365, 40.739446], // [minLng, minLat, maxLng, maxLat] — feed to Boundary
    "postcode": "11215", "place": "New York City",
    "district": "Kings County", "region": "New York", "country": "United States"
  }
]
```

- `provider`: `mapbox` (default) or `opencage`. `limit`: 1–10.
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

## Recipe 6 — AI need-classification (advanced)

> For most "AI search" needs, prefer **`query_type=hybrid`**
> ([Recipe 1](#recipe-1--find-resources-search)) — one call, no extra steps.
> Use this flow only when you want to *classify* a vague natural-language need
> into taxonomy codes and possibly ask the user to clarify before searching.

Turn a natural-language need into taxonomy codes, then search with them.

```bash
curl -s "${H[@]}" "$BASE/search/predict?query=i%20need%20rent%20help&top_k=150"
```

```jsonc
{
  "scenario": "search",                 // or clarify_* — ask the user to narrow
  "hsis_taxonomies": ["BH-3800.4900", "BH-3500", "..."],
  "options": [ { "code": "BH-3500", "score": 0.91, "pre_selected": true, "results_count": 42 } ]
}
```

- `scenario: "search"` → feed `hsis_taxonomies` straight into
  `GET /search?taxonomy=<codes>&query_type=taxonomy`.
- `scenario: "clarify_*"` → present `options[]` and let the user pick.
- `GET /search/re-rank?need_weights=<url-encoded JSON>&top_k=150` re-orders codes
  when the user weights competing needs.

---

## Search parameter reference (`/search`, GET & POST)

| Param | Type | Default | Notes |
|---|---|---|---|
| `query` | string | `""` | Free text; or comma HSIS codes with `query_type=taxonomy`. |
| `query_type` | enum | `text` | **`hybrid`** recommended (semantic + keyword). Also `text` (exact/agency, being phased out) · `taxonomy` · `more_like_this`. |
| `page` | int ≥1 | `1` | 1-based. |
| `limit` | int | `25` | **25–300** (below 25 errors). |
| `taxonomy` | string/CSV | — | HSIS codes, e.g. `BM-1400,BM-1700`. |
| `coords` | string | — | **`lng,lat`**, e.g. `-73.9857,40.7484`. |
| `distance` | int (mi) | `0` | Only with `coords`. |
| `geo_type` | enum | — | `proximity` · `boundary` (boundary → POST body geometry). |
| `filters` | bracket obj | `{}` | `filters[<key>][<i>]=<value>`. |
| `age` | int ≥0 | — | Age eligibility filter. |
| `sort` | enum | `relevance` | `relevance` · `distance` · `name` · `organization`. |

---

## Versioning & errors

- **Version by header** (`x-api-version`), latest per endpoint: **`2`** for
  `/taxonomy`; `1` for everything else (see
  [Always use the latest version](#always-use-the-latest-version-an-endpoint-supports)).
- `401` — missing/invalid `X-API-Key`.
- `400` — missing/invalid `x-tenant-id` (on tenant-scoped routes) or a bad param
  (e.g. `limit` < 25).
- `404` — resource not found — **most often because you used `_source.id`
  instead of `service_at_location_id`** (see the gotcha above).

---

## Endpoint ↔ Bruno folder map

| Task | Endpoint | Bruno folder |
|---|---|---|
| Find (text/taxonomy/geo) | `GET`/`POST` `/search` | `01. Search` |
| Get details | `GET /resource/{id}`, `/resource/original/{id}` | `02. Resource` |
| Batch / titles | `POST /resource/batch`, `/resource/titles` | `02. Resource` |
| Typeahead codes | `GET /taxonomy` (v2), `/suggestion` | `03. Taxonomy`, `04. Suggestion` |
| Org typeahead | `GET /organization` | `05. Organization` |
| Geocode | `GET /geocoding/forward` · `/reverse` | `06. Geocoding` |
| AI need → codes | `GET /search/predict` · `/re-rank` | `01. Search` |
