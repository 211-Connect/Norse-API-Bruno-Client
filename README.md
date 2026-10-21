# Norse API — Bruno Collection

A [Bruno](https://www.usebruno.com/) API client for the **Norse API**, exposed
through the Connect211 **API gateway**. It lets partners and developers explore
and test the public Norse API surface — search, resources, taxonomy,
suggestions, organizations, services, geocoding, and short URLs — using a
single API key.

> **New here?** See **[USAGE.md](./USAGE.md)** for a task-oriented walkthrough —
> how to find resources and pull their details, with copy-paste `curl` examples
> (search, taxonomy typeahead, detail lookup, batch, organizations, geo, and
> AI-assisted search).

The authoritative, always-current contract is the live OpenAPI document:
**<https://services.c211.io/norse-api/v1/swagger>** (JSON at `…/swagger/json`).
It needs no API key. This collection tracks it; when the two disagree, the
OpenAPI document wins.

## Getting started

1. Install [Bruno](https://www.usebruno.com/) and open this folder as a collection.
2. Select the **Gateway** environment (top-right).
3. Open the environment and fill in the two secrets: paste your key into
   **`SERVICES_APIKEY`** (see [Getting your API key](#getting-your-api-key-and-tenant-id))
   and your tenant UUID into **`tenantId`**.
4. Run any request — start with **`00. Health → Health Check`**, then
   **`01. Search → Search Resources (GET)`**.

## Authentication

Every request through the gateway sends four headers. The collection wires all
of them for you from environment variables:

| Header | Value | Source |
|---|---|---|
| `X-API-Key` | your API key | `SERVICES_APIKEY` (**secret** — you provide it) |
| `x-tenant-id` | tenant UUID | `tenantId` (**secret** — you provide it) |
| `x-api-version` | API version (`1`) | set per request |
| `accept-language` | `en`, `es`, … | `acceptLanguage` |

- **No key / an invalid key → `401`.**
- **Missing `x-tenant-id` → `400 tenant_target_required`.** Every route except
  `/health` needs it — including Geocoding and Short URL, whose *data* is not
  tenant-specific but which still go through the gateway's tenant check.
- **A tenant your key is not provisioned for → `403 tenant_scope_denied`.**
- **`POST` routes count as writes.** `POST /search`, `POST /resource/batch`,
  `POST /resource/titles` and `POST /short-url` need the *write* permission for
  the tenant on your key. If a `POST` returns `403 tenant_scope_denied` while
  `GET` works, ask for write access (below) or use the `GET` equivalent.

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
Say whether you need **write** access (for the `POST` routes above) — keys are
read-only unless asked.

Once you receive them, paste the key into `SERVICES_APIKEY` and the tenant UUID
into `tenantId` in the **Gateway** environment.

## Environment variables

| Variable | Meaning |
|---|---|
| `baseUrl` | Gateway root for Norse API: `https://services.c211.io/norse-api/v1`. |
| `tenantId` (secret) | Tenant UUID sent as `x-tenant-id`. **You provide this; never commit it.** |
| `acceptLanguage` | `accept-language` header (`en`, `es`, …). |
| `SERVICES_APIKEY` (secret) | Your gateway API key, sent as `X-API-Key`. **You provide this; never commit it.** |

## How the API works (essentials)

- **Base URL includes the gateway prefix** — requests are relative to
  `https://services.c211.io/norse-api/v1` (e.g. `GET …/v1/search`).
- **Versioning is header-based** via `x-api-version`. Every endpoint is
  currently `1`. (Taxonomy search used to need `2` for its slim shape; that is
  now what `1` returns, and `2` no longer exists.)
- **Tenant scoping.** Search, Resource, Taxonomy, Suggestion, Organization and
  Service return data for the tenant in `x-tenant-id` and localize by
  `accept-language`. Geocoding and Short URL are not tenant-specific but still
  require the header (see Authentication).

## Folder map

| Folder | Notes |
|---|---|
| `00. Health` | Liveness. |
| `01. Search` | Resource search (incl. `organization_id` scope and opt-in relevance cutoff) + AI predict/re-rank. |
| `02. Resource` | Single + batch resource (service-at-location) lookups. |
| `03. Taxonomy` | HSIS taxonomy search + term lookup. |
| `04. Suggestion` | Combined taxonomy + organization typeahead. |
| `05. Organization` | Organization typeahead, full record by id, and lookup by service-at-location. |
| `06. Geocoding` | Forward/reverse geocoding. |
| `07. Short URL` | Create/resolve short URLs. |
| `08. Service` | HSDS service record by `serviceId`. |

> **Scope:** this collection covers only the endpoints reachable through the
> gateway with a standard *norse-api* key. Internal/admin surfaces and
> user-authenticated features (favorites, printable directories) are not part
> of this public collection.
