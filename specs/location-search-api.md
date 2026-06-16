# Location Search API

## Status

This document defines the AON Location Search API as a v0.1 runtime API.
It is the lookup companion to the static registry, not a caller-side offer search filter.
Offer discovery still runs through `POST /v1/offers/query`, where viewer
location is provided as `context.user_profile.location_ids`.

The runtime routes are public lookup endpoints. Clients that need offline or
build-time lookup can still use the static registry file plus the schema helper
functions, but production interactive selectors SHOULD use the runtime API.

## Source Model

AON Location Registry v1 is generated from Google Ads Geo Target Criteria IDs.
The current public surface intentionally supports three normalized AON levels:

| Level | Meaning |
|-------|---------|
| `COUNTRY` | Country or region accepted as the broadest supported targeting level |
| `REGION` | First-level administrative area such as state, province, territory, or equivalent |
| `CITY` | City-level location from the Google source file |

The original Google `target_type` remains available for display and debugging.
AON exposes `REGION` as the public normalized level instead of copying every
provider-specific administrative label into the protocol field name.

## `GET /v1/locations/search`

Search the supported location registry by human-readable text and optional
constraints.

### Query Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `q` | string | CONDITIONALLY REQUIRED | Case-insensitive search text matched against `name`, `canonical_name`, `location_id`, country code, and supported aliases. Required unless `parent_location_id` is supplied or `levels=COUNTRY` is used for top-level browsing. |
| `parent_location_id` | string | OPTIONAL | Numeric parent id used for direct child browsing. When present, results are restricted to immediate children of that parent. |
| `country` | string | OPTIONAL | Uppercase ISO 3166-1 alpha-2 country code, such as `US`. |
| `levels` | string[] | OPTIONAL | Comma-separated subset of `COUNTRY`, `REGION`, and `CITY`. |
| `subdivision_code` | string | OPTIONAL | External first-level subdivision code used to narrow candidates, such as ISO 3166-2 `US-CA` or CLDR `USCA`. |
| `subdivision_code_type` | string | OPTIONAL | One of `AUTO`, `ISO_3166_2`, `CLDR`, or `PROVIDER_SHORT`. Default `AUTO`. |
| `limit` | integer | OPTIONAL | Maximum results to return. Default `20`, maximum `50`. |
| `locale` | string | OPTIONAL | Reserved for future localized display names. v0.1 results use registry names. |

For city search, callers SHOULD provide `country` and SHOULD provide
`subdivision_code` when available. City names are not stable identifiers; they
are lookup text used to select AON `location_id` values. The v0.1 contract
requires deterministic case-insensitive text matching and disambiguation
constraints, but does not require typo correction, distance ranking, or
localized names.

For cascade selectors, the recommended calls are:

```text
GET /v1/locations/search?levels=COUNTRY&limit=50
GET /v1/locations/search?parent_location_id=2840&levels=REGION&limit=50
GET /v1/locations/search?parent_location_id=21137&levels=CITY&q=san&limit=50
```

An empty broad search without `parent_location_id` or `levels=COUNTRY` is
invalid.

### Response Fields

The response uses the standard AON success envelope:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `code` | string | REQUIRED | `SUCCESS` on a successful lookup. |
| `message` | string | REQUIRED | Empty string on normal success. |
| `data.registry_version` | string | REQUIRED | Static registry version, currently `v1`. |
| `data.source_file_date` | string | REQUIRED | Source Google Geo Targets file date. |
| `data.locations[]` | object[] | REQUIRED | Matching locations. |
| `data.locations[].location_id` | string | REQUIRED | Numeric-string AON location id, aligned to Google Criteria ID. |
| `data.locations[].name` | string | REQUIRED | Short display name. |
| `data.locations[].canonical_name` | string | REQUIRED | Provider canonical name, commonly comma-separated from narrow to broad. |
| `data.locations[].country_code` | string | REQUIRED | Uppercase ISO 3166-1 alpha-2 country code. |
| `data.locations[].level` | string | REQUIRED | One of `COUNTRY`, `REGION`, or `CITY`. |
| `data.locations[].target_type` | string | REQUIRED | Original Google target type, such as `Country`, `State`, or `City`. |
| `data.locations[].parent_location_id` | string \| null | REQUIRED | Immediate parent location id, or null for top-level countries. |
| `data.locations[].path` | object[] | REQUIRED | Root-to-self `path` derived from parent links. |
| `data.locations[].external_codes` | object | OPTIONAL | Lookup aliases such as `iso_3166_2`, `cldr_subdivision`, or `provider_short`, when AON has a supported mapping. These are not matching keys. |
| `extra` | object | REQUIRED | Optional metadata bag for future warnings or pagination. |

`external_codes` are migration lookup aliases only. They are never canonical
Offer or Query API matching keys.

```json
{
  "code": "SUCCESS",
  "message": "",
  "data": {
    "registry_version": "v1",
    "source_file_date": "2026-05-28",
    "locations": [
      {
        "location_id": "21137",
        "name": "California",
        "canonical_name": "California,United States",
        "country_code": "US",
        "level": "REGION",
        "target_type": "State",
        "parent_location_id": "2840",
        "external_codes": {
          "iso_3166_2": "US-CA",
          "cldr_subdivision": "USCA",
          "provider_short": "CA"
        },
        "path": [
          { "location_id": "2840", "name": "United States", "level": "COUNTRY" },
          { "location_id": "21137", "name": "California", "level": "REGION" }
        ]
      }
    ]
  },
  "extra": {}
}
```

## `GET /v1/locations/resolve`

Resolve external location signals into AON `location_id` values. This endpoint
is intended for integrations that already receive ISO, CLDR, CDN, or edge
location fields and need to normalize them before writing Offer targeting or
Query API viewer context.

### Query Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `country` | string | OPTIONAL | Uppercase ISO 3166-1 alpha-2 country code, such as `US`. Required for short provider subdivision codes such as `CA`. |
| `subdivision_code` | string | OPTIONAL | External first-level subdivision code. Accepted v0.1 forms include ISO 3166-2 (`US-CA`), CLDR (`USCA`), and provider short form (`CA`) when `country=US`. |
| `subdivision_code_type` | string | OPTIONAL | One of `AUTO`, `ISO_3166_2`, `CLDR`, or `PROVIDER_SHORT`. Default `AUTO`. |
| `city` | string | OPTIONAL | Optional city text used to pick a `CITY` candidate under the resolved country or subdivision. |
| `limit` | integer | OPTIONAL | Maximum candidates to return. Default `10`, maximum `50`. |

### Response Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `data.location` | object | REQUIRED | Best resolved location. Unsupported input returns an error envelope rather than a null success. |
| `data.location_ids` | string[] | REQUIRED | Self-to-root chain suitable for Query API context. Empty when no supported mapping is found. |
| `data.candidates[]` | object[] | REQUIRED | Candidate locations after applying the normalized constraints. |

```json
{
  "code": "SUCCESS",
  "message": "",
  "data": {
    "registry_version": "v1",
    "source_file_date": "2026-05-28",
    "location": {
      "location_id": "1014221",
      "name": "San Francisco",
      "canonical_name": "San Francisco,California,United States",
      "country_code": "US",
      "level": "CITY",
      "target_type": "City",
      "parent_location_id": "21137",
      "path": [
        { "location_id": "2840", "name": "United States", "level": "COUNTRY" },
        { "location_id": "21137", "name": "California", "level": "REGION" },
        { "location_id": "1014221", "name": "San Francisco", "level": "CITY" }
      ]
    },
    "location_ids": ["1014221", "21137", "2840"],
    "candidates": []
  },
  "extra": {}
}
```

## `GET /v1/locations/{location_id}`

Look up one supported location id and return its root-to-self path plus the
self-to-root `location_ids` chain callers can place in Query API context.

### Path Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `location_id` | string | REQUIRED | Numeric-string AON Location Registry v1 id. |

### Response Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `data.location` | object | REQUIRED | A single Location Search result object. |
| `data.location_ids` | string[] | REQUIRED | Self-to-root chain, for example city, region, country. |

```json
{
  "code": "SUCCESS",
  "message": "",
  "data": {
    "registry_version": "v1",
    "source_file_date": "2026-05-28",
    "location": {
      "location_id": "1014221",
      "name": "San Francisco",
      "canonical_name": "San Francisco,California,United States",
      "country_code": "US",
      "level": "CITY",
      "target_type": "City",
      "parent_location_id": "21137",
      "path": [
        { "location_id": "2840", "name": "United States", "level": "COUNTRY" },
        { "location_id": "21137", "name": "California", "level": "REGION" },
        { "location_id": "1014221", "name": "San Francisco", "level": "CITY" }
      ]
    },
    "location_ids": ["1014221", "21137", "2840"]
  },
  "extra": {}
}
```

## Migration Helpers

The schema repository ships small helper functions so integrations can migrate
from country-only geo fields, run offline tests, or normalize headers in
environments that cannot call the runtime API:

- `countryCodeToLocationId(countryCode)` maps legacy uppercase or lowercase
  country codes to country-level location ids.
- `legacyCountryGeoToLocationGeo(countries)` converts legacy country arrays to
  structured `{ "location_id": "<id>" }` entries.
- `subdivisionCodeToLocationId(code, options)` maps supported ISO 3166-2,
  CLDR, or country-scoped provider short subdivision codes to AON REGION ids.
- `buildLocationChain(location_id)` derives a self-to-root chain from
  `parent_location_id` links.
- `resolveLocationInput(input)` combines `country`, `subdivision_code`, and
  optional `city` into a best resolved location and Query `location_ids` chain.
- `cloudflareHeadersToLocationContext(headers)` normalizes Cloudflare visitor
  location headers such as `cf-ipcountry`, `cf-region-code`, and `cf-ipcity`.
- `googleCloudHeadersToLocationContext(headers)` normalizes Google Cloud Load
  Balancing custom headers such as `client_region`,
  `client_region_subdivision`, and `client_city`.

These helpers are convenience code for client migration and tests. The static
registry remains the source of truth.

## External Code Guidance

External codes are lookup aliases. They MUST NOT be sent in Offer
`targeting[].geo.include/exclude` and MUST NOT be sent as Query matching keys.
After resolving an external code, callers write AON `location_id` values:

```json
{
  "targeting": [
    {
      "geo": {
        "include": [{ "location_id": "21137" }]
      }
    }
  ]
}
```

For Cloudflare, combine country and region code before resolving:

```text
cf-ipcountry: US
cf-region-code: CA
cf-ipcity: San Francisco
```

This resolves as `country=US`, `subdivision_code=US-CA`, and
`city=San Francisco`.

For Google Cloud Load Balancing custom headers, use the CLDR subdivision id:

```text
client_region: US
client_region_subdivision: USCA
client_city: San Francisco
```

This resolves as `country=US`, `subdivision_code=USCA`, and
`city=San Francisco`.

## Query API Boundary

`GET /v1/locations/search` helps developers select and store location ids.
It is not a caller-side offer search filter. To evaluate offer eligibility,
callers provide the resolved viewer chain in `context.user_profile.location_ids`
on `POST /v1/offers/query`; AON applies offer `targeting[].geo.include` and
`targeting[].geo.exclude` before ranking.
