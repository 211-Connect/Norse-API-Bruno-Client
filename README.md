# Norse API — Bruno Collection

A [Bruno](https://www.usebruno.com/) API client for the **Norse API**, exposed
through the Connect211 **API gateway**. It lets partners and developers explore
and test the public Norse API surface — search, resources, taxonomy,
suggestions, organizations, geocoding, and short URLs — using a single API key.

> **New here?** See **[USAGE.md](./USAGE.md)** for a task-oriented walkthrough —
> how to find resources and pull their details, with copy-paste `curl` examples
> (search, taxonomy typeahead, detail lookup, batch, geo, and AI-assisted search).

## Getting started

1. Install [Bruno](https://www.usebruno.com/) and open this folder as a collection.
2. Select the **Gateway** environment (top-right).
3. Open the environment and fill in the two secrets: paste your key into
   **`SERVICES_APIKEY`** (see [Getting an API key](#getting-an-api-key)) and
   your tenant UUID into **`tenantId`**.
4. Run any request — start with **`00. Health → Health Check`**, then
   **`01. Search → Search Resources (GET)`**.

## Authentication

Every request through the gateway sends four headers. The collection wires all
of them for you from environment variables:

| Header | Value | Source |
|---|---|---|
| `X-API-Key` | your API key | `SERVICES_APIKEY` (**secret** — you provide it) |
| `x-tenant-id` | tenant UUID | `tenantId` (**secret** — you provide it) |
| `x-api-version` | API version (e.g. `1`) | set per request |
| `accept-language` | `en`, `es`, … | `acceptLanguage` |

- **No key / an invalid key → `401`.**
- **A valid key but a missing/invalid tenant → `400`.**

`SERVICES_APIKEY` is declared as an **empty secret** and is **never** committed to
this repo — each user pastes their own key locally in Bruno.

### Getting your API key and tenant ID

To receive your **API key** and the correct **tenant ID(s)**, email
**help@connect211.com** with the subject line:

```
Developer API Key Provisioning - {tenant name}
```

CC **david@connect211.com** as well — especially for questions. Sending to
**help@connect211.com** is what matters, though: it opens a ticket so a human is
guaranteed to prioritize a follow-up to every request (even when David is busy).

Once you receive them, paste the key into `SERVICES_APIKEY` and the tenant UUID
into `tenantId` in the **Gateway** environment.

## Environment variables

| Variable | Meaning |
|---|---|
| `baseUrl` | Gateway root for Norse API: `https://services.c211.io/norse-api/v1`. |
| `tenantId` (secret) | Tenant UUID sent as `x-tenant-id`. Required on tenant-scoped routes. **You provide this; never commit it.** |
| `acceptLanguage` | `accept-language` header (`en`, `es`, …). |
| `SERVICES_APIKEY` (secret) | Your gateway API key, sent as `X-API-Key`. **You provide this; never commit it.** |

## How the API works (essentials)

- **Base URL includes the gateway prefix** — requests are relative to
  `https://services.c211.io/norse-api/v1` (e.g. `GET …/v1/search`).
- **Versioning is header-based** via `x-api-version`. Taxonomy search uses
  `x-api-version: 2` (returns a slimmer `{ id, name, code }`); each request sends
  the right version for you.
- **Tenant scoping.** Search, Resource, Taxonomy, Suggestion, and Organization
  require `x-tenant-id` (a UUID) and `accept-language`. Geocoding, Short URL, and
  Health are not tenant-scoped, but still require `X-API-Key`.

## Folder map

| Folder | Notes |
|---|---|
| `00. Health` | Liveness. |
| `01. Search` | Resource search + AI predict/re-rank. |
| `02. Resource` | Single + batch resource lookups. |
| `03. Taxonomy` | HSIS taxonomy search + term lookup. |
| `04. Suggestion` | Typeahead suggestions. |
| `05. Organization` | Organization typeahead. |
| `06. Geocoding` | Forward/reverse geocoding. |
| `07. Short URL` | Create/resolve short URLs. |

> **Scope:** this collection covers only the endpoints reachable through the
> gateway with a standard *norse-api* key. Internal/admin surfaces (analytics,
> config, scorecard) and user-authenticated features (favorites, printable
> directories) are not part of this public collection.
