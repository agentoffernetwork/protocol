# Location Search API

## Status

This document defines the AON Location Search runtime API. Its default
`LEGACY` catalog remains the v0.1-compatible lookup companion to the static
registry. Protocol V1.0 adds explicit `catalog=FULL` search, lookup, and
resolve modes; it does not introduce a new protocol selector or change the
default LEGACY request or response shape.

Location Search is not a caller-side offer search filter. Offer discovery
still runs through `POST /v1/offers/query`, where viewer location is provided
as `context.user_profile.location_ids`.

The runtime routes are public lookup endpoints. Clients that need offline or
build-time lookup can still use the static registry file plus the schema helper
functions, but production interactive selectors SHOULD use the runtime API.

## LEGACY Source Model

The default `catalog=LEGACY` (or omitted `catalog`) uses AON Location Registry
v1, generated from Google Ads Geo Target Criteria IDs. This compatibility
surface intentionally supports three normalized AON levels:

| Level | Meaning |
|-------|---------|
| `COUNTRY` | Country or region accepted as the broadest supported targeting level |
| `REGION` | First-level administrative area such as state, province, territory, or equivalent |
| `CITY` | City-level location from the Google source file |

The original Google `target_type` remains available for display and debugging.
AON exposes `REGION` as the public normalized level instead of copying every
provider-specific administrative label into the protocol field name.

## Protocol V1.0 Catalog Selection

`catalog` selects the representation for every Location Search route:

| `catalog` value | Request and result contract |
|-------|-----------------------------|
| omitted or `LEGACY` | The existing v0.1-compatible registry search. `levels` is limited to `COUNTRY`, `REGION`, and `CITY`; the response remains the existing `registry_version` envelope and result shape. |
| `FULL` | The Protocol V1.0 full catalog. Search covers every `ACTIVE` Google Geo Target source row; lookup and resolve return any one active ID with the same raw type and verified hierarchy metadata. |

`catalog=FULL` is opt-in. Services MUST continue to interpret an omitted
`catalog` exactly as `LEGACY`, including its ordering, fields, and three-level
normalization. The V1.0 extension does not change the Offer/Query protocol
selector or the legacy static registry used by Partner.

### `catalog=FULL` Request

The FULL request accepts these query parameters. It does not accept the
LEGACY-only `levels`, `subdivision_code`, or `subdivision_code_type` filters.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `catalog` | string | REQUIRED | Must be `FULL`. |
| `q` | string | CONDITIONALLY REQUIRED | Case-insensitive search text against `name`, `canonical_name`, `location_id`, and country code. Required unless `parent_location_id` is supplied for direct-child browsing. |
| `parent_location_id` | string | OPTIONAL | Resolved canonical parent id. Results are restricted to direct children with that exact resolved `parent_location_id`. |
| `country` | string | OPTIONAL | Uppercase ISO 3166-1 alpha-2 country code, such as `US`. |
| `target_types` | string[] | OPTIONAL | Comma-separated exact raw Google Geo Target types, such as `Postal Code`, `Neighborhood`, `Ward`, or `City`. It is not a normalized AON level filter. |
| `limit` | integer | OPTIONAL | Maximum results to return. Default `20`, maximum `50`. |
| `cursor` | string | OPTIONAL | Opaque continuation cursor from `data.next_cursor`. It is valid only with the same FULL query constraints. |
| `locale` | string | OPTIONAL | Reserved for localized display names. Current results use catalog names. |

### `catalog=FULL` Response

An explicit FULL request returns the standard success envelope with the
following `data` fields. It is intentionally distinct from the unchanged
LEGACY envelope so clients can discriminate the result shape by
`data.catalog`.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `data.catalog` | string | REQUIRED | Always `FULL`. |
| `data.catalog_version` | string | REQUIRED | Full catalog version, currently `v1`. |
| `data.source_file_date` | string | REQUIRED | Pinned Google Geo Targets source file date. |
| `data.locations[]` | object[] | REQUIRED | Matching `ACTIVE` catalog records. |
| `data.next_cursor` | string | OPTIONAL | Opaque continuation cursor, omitted on the final page. |
| `data.locations[].location_id` | string | REQUIRED | Numeric-string Google Criteria ID used as the Offer targeting key. |
| `data.locations[].target_type` | string | REQUIRED | Exact raw Google Geo Target type. It MUST NOT be normalized to `CITY`, `REGION`, or `COUNTRY`. |
| `data.locations[].source_parent_location_id` | string \| null | REQUIRED | Raw `Parent ID` from the source CSV. |
| `data.locations[].parent_location_id` | string \| null | REQUIRED | Resolved parent based only on verified ACTIVE source-parent edges. It is null when no parent relation can be verified. |
| `data.locations[].path` | object[] | REQUIRED | Root-to-self canonical path containing only verified parent relations. Each path node exposes `location_id`, `name`, `canonical_name`, `country_code`, `target_type`, and `hierarchy_precision`. |
| `data.locations[].hierarchy_precision` | integer | REQUIRED | One-based verified hierarchy depth, at least `1`. It is the precision used for safe targeting evaluation. |
| `data.locations[].legacy_level` | string | OPTIONAL | `COUNTRY`, `REGION`, or `CITY` only when the record can be represented by the legacy registry without relabeling. FULL results never contain a `level` field. |
| `data.locations[].chain_status` | string | REQUIRED | `COMPLETE` or `UNRESOLVED_SOURCE_PARENT`. |
| `data.locations[].unresolved_source_parent_location_id` | string | CONDITIONAL | REQUIRED when `chain_status` is `UNRESOLVED_SOURCE_PARENT`; equals the raw parent id that could not be resolved. |

Every ACTIVE source record is searchable in FULL mode. If a raw source parent
cannot be found, the service MUST return that record with
`chain_status=UNRESOLVED_SOURCE_PARENT`, preserve its
`source_parent_location_id`, and report the missing id in
`unresolved_source_parent_location_id`. The service MUST NOT invent a parent,
path segment, or precision from `country_code`, `canonical_name`, or another
geographic source. Relationship-based Offer matching and viewer-context
derivation involving a non-`COMPLETE` chain MUST fail closed; exact-id
selection remains available.

For example, a Postal Code remains a Postal Code rather than being relabeled
as a city:

```json
{
  "code": "SUCCESS",
  "message": "",
  "data": {
    "catalog": "FULL",
    "catalog_version": "v1",
    "source_file_date": "2026-05-28",
    "locations": [
      {
        "location_id": "9304101",
        "name": "522410",
        "canonical_name": "522410,Andhra Pradesh,India",
        "country_code": "IN",
        "target_type": "Postal Code",
        "source_parent_location_id": "20453",
        "parent_location_id": "20453",
        "hierarchy_precision": 3,
        "chain_status": "COMPLETE",
        "path": [
          {
            "location_id": "2356",
            "name": "India",
            "canonical_name": "India",
            "country_code": "IN",
            "target_type": "Country",
            "hierarchy_precision": 1
          },
          {
            "location_id": "20453",
            "name": "Andhra Pradesh",
            "canonical_name": "Andhra Pradesh,India",
            "country_code": "IN",
            "target_type": "State",
            "hierarchy_precision": 2
          },
          {
            "location_id": "9304101",
            "name": "522410",
            "canonical_name": "522410,Andhra Pradesh,India",
            "country_code": "IN",
            "target_type": "Postal Code",
            "hierarchy_precision": 3
          }
        ]
      }
    ]
  },
  "extra": {}
}
```

### `catalog=FULL` Lookup and Resolve

`GET /v1/locations/{location_id}?catalog=FULL` looks up any numeric active
catalog ID. `GET /v1/locations/resolve?catalog=FULL&location_id={location_id}`
resolves that same direct ID into a Query API context. FULL resolve accepts no
legacy `country`, `subdivision_code`, `subdivision_code_type`, `city`, or
`limit` parameters; callers select text candidates with FULL search first.

Both routes return `data.catalog="FULL"`, `data.catalog_version`, and
`data.source_file_date`, then `data.location`, root-to-self `data.path`, and
self-to-root `data.location_ids`. Resolve also returns `data.candidates: []`.
`data.location` is the FULL result object described above, including its raw
parent fields, verified path, precision, and chain status. The separate
`data.path` is identical to `data.location.path`. Unknown, inactive, and
non-numeric IDs use the same `NOT_FOUND` or `BAD_REQUEST` errors as FULL
search; unresolved source parents are preserved in the location metadata and
never added to either path or `location_ids`.

## `GET /v1/locations/search`

With omitted `catalog` or `catalog=LEGACY`, search the supported legacy
location registry by human-readable text and optional constraints.

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
| `catalog` | string | OPTIONAL | Omitted or `LEGACY` preserves this legacy resolver. `FULL` switches to direct full-catalog resolution. |
| `location_id` | string | CONDITIONAL | Required with `catalog=FULL`; numeric active full-catalog ID. Ignored by the legacy resolver. |
| `country` | string | OPTIONAL | Uppercase ISO 3166-1 alpha-2 country code, such as `US`. Required for short provider subdivision codes such as `CA`. |
| `subdivision_code` | string | OPTIONAL | External first-level subdivision code. Accepted v0.1 forms include ISO 3166-2 (`US-CA`), CLDR (`USCA`), and provider short form (`CA`) when `country=US`. |
| `subdivision_code_type` | string | OPTIONAL | One of `AUTO`, `ISO_3166_2`, `CLDR`, or `PROVIDER_SHORT`. Default `AUTO`. |
| `city` | string | OPTIONAL | Optional city text used to pick a `CITY` candidate under the resolved country or subdivision. |
| `limit` | integer | OPTIONAL | Maximum candidates to return. Default `10`, maximum `50`. |

### Response Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `data.catalog` | string | CONDITIONAL | `FULL` for the full-catalog response; absent for LEGACY. |
| `data.catalog_version` | string | CONDITIONAL | Full catalog version when `data.catalog=FULL`; absent for LEGACY. |
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
| `location_id` | string | REQUIRED | Numeric-string location id. `LEGACY` accepts Registry v1 IDs; `FULL` accepts any active full-catalog ID. |

### Query Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `catalog` | string | OPTIONAL | Omitted or `LEGACY` uses the existing registry. `FULL` looks up any numeric active full-catalog ID. |

### Response Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `data.catalog` | string | CONDITIONAL | `FULL` for the full-catalog response; absent for LEGACY. |
| `data.catalog_version` | string | CONDITIONAL | Full catalog version when `data.catalog=FULL`; absent for LEGACY. |
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
