# Offer Query API v0.1

- **Status**: Draft
- **Last Updated**: 2026-05-15
- **Source**: `agentoffernetwork/protocol/specs/query-api.md`

Use this API when an agent has user intent and needs ranked commercial offers to recommend, compare, or present.

## Endpoint

| Item | Value |
|------|-------|
| Method | `POST` |
| Path | `/v1/offers/query` |
| Auth | `Authorization: Bearer YOUR_API_KEY` |
| Content-Type | `application/json` |
| Protocol-Version | `AON-Protocol-Version: 0.1` recommended. SDKs send it by default. |
| Request | `context` + `intent`, optional `constraints` and `pagination` |
| Response | `request_id` + `offers[]` |
| Response Header | `X-AON-TRACE-ID` for diagnostics; not part of the JSON payload. |

## Minimal Request

```bash
curl -s -X POST "https://api.aon.pro/v1/offers/query" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -H "AON-Protocol-Version: 0.1" \
  -d '{
    "context": {
      "user_profile": {
        "device_info": {
          "device_type": "other",
          "os": "other"
        }
      }
    },
    "intent": {
      "content": [
        {
          "type": "input_text",
          "text": "Find project management software for a 20-person design team"
        }
      ]
    }
  }'
```

## Request Shape

| Field | Level | Notes |
|------|-------|-------|
| `context` | REQUIRED | Requesting platform, session, and user context. |
| `context.user_profile` | REQUIRED | User profile container. It may be sparse, but `device_info.device_type` and `device_info.os` must be present; use `other` when unknown. |
| `intent` | REQUIRED | User intent expressed as multimodal content. |
| `intent.content[]` | REQUIRED | At least one content item. Current `type` values: `input_text`, `input_image`. |
| `constraints` | OPTIONAL | Deterministic eligibility constraints applied before semantic ranking. |
| `pagination` | RECOMMENDED | Defaults: `limit=20`, `offset=0`. Maximum `limit=100`. |
| `request_id` | OPTIONAL | UUIDv7 recommended. Server generates one when omitted. |
| `timestamp` | OPTIONAL | RFC 3339 timestamp. Server uses current time when omitted. |
| `test_mode` | OPTIONAL | `true` marks the request as a test and should avoid real billing/tracking effects. |

### Common Request Fields

| Field | Type | Example |
|------|------|---------|
| `context.platform.name` | string | `TravelBot` |
| `context.platform.channel` | string | `api`, `sdk`, `plugin`, `skill` |
| `context.session_id` | string | `sess_abc123` |
| `context.user_profile.user_pseudo_id` | string | `viewer_xyz` |
| `context.user_profile.language` | string | `en` |
| `context.user_profile.country` | string | `US` |
| `context.user_profile.location_ids` | string[] | `["1014221", "21137", "2840"]` |
| `context.user_profile.verified_age_over` | integer[] | `[18]` |
| `context.user_profile.interests` | string[] | `["travel", "hotels"]` |
| `context.user_profile.device_info` | object | `{ "device_type": "mobile", "os": "ios" }` |
| `context.user_profile.device_info.device_type` | string | `mobile` |
| `context.user_profile.device_info.os` | string | `ios` |
| `context.user_profile.device_info.os_version` | string | `18.2` |
| `context.user_profile.device_info.user_agent` | string | `Mozilla/5.0` |
| `intent.content[].text` | string | `Find me a luxury hotel in Tokyo` |
| `constraints.category_ids` | string[] | `["travel_tourism"]` |
| `pagination.limit` | integer | `10` |
| `pagination.offset` | integer | `0` |

### Common Values

| Field | Common values |
|------|---------------|
| `intent.content[].type` | `input_text`, `input_image` |
| `constraints.category_ids` | AON Taxonomy v1 ids such as `travel_tourism`, `finance.credit_lending`, `others`, `arts_entertainment.igaming` |
| `context.user_profile.device_info.device_type` | `desktop`, `mobile`, `tablet`, `smart_tv`, `other` |
| `context.user_profile.device_info.os` | `ios`, `android`, `windows`, `macos`, `other` |
| `offer_info.offer_type` | `physical_product`, `digital_goods`, `content`, `online_service`, `offline_service` |

See [`category-taxonomy.md`](category-taxonomy.md) and [`offer-schema.md`](offer-schema.md) for the full category, offer, bid, and lifecycle definitions.

### Constraints Semantics

`constraints` is a small set of deterministic eligibility constraints, not a
search DSL and not a lifecycle control surface. Use `intent.content[]` for
semantic matching and ranking signals; use `constraints` only when a result
would be invalid unless the constraint is satisfied.

The first public `constraints` surface intentionally exposes only
`constraints.category_ids`. Bid model, lifecycle status, currency, price,
brand, and location/country constraints are not part of the agent-facing Query API
contract in this version.

Note: `context.user_profile.location_ids` and legacy
`context.user_profile.country` are user-profile attributes (not `constraints`
entries). They carry the viewer's resolved location for offer geo targeting and
are used before ranking. They are not caller-specified location search filters.

`context.user_profile.location_ids` contains AON Location Registry v1 location
IDs, sourced from Google Ads Geo Target Criteria IDs. Callers SHOULD send the
most specific known location first, followed by broader known ancestors, for
example San Francisco city, California as a region-level location, then United States country:
`["1014221", "21137", "2840"]`. The first registry release supports
`COUNTRY`, `REGION`, and `CITY` levels. Ancestor chains are derived from the
registry's `parent_location_id` links; the public registry does not publish a
separate ancestor cache.

`REGION` is AON's normalized public level for a first-level geographic region
under a country, such as a state, province, prefecture, region, or equivalent.
The original Google `target_type` is preserved separately in the location
registry.

Location recall is strict and fail-closed. Offers with structured
`targeting[].geo.include` match when at least one included `location_id` is in
the viewer's self-or-ancestor `location_ids`; `geo.exclude` wins over include.
Unknown location IDs fail closed. Missing `targeting`, empty targeting, omitted
`geo`, omitted `geo.include`, or empty `geo.include` mean the location dimension
is unconstrained by this contract; they do not create a caller-facing location
search filter. When `geo.exclude` is present, the platform must be able to
resolve viewer location to prove the viewer is not excluded. Legacy
`context.user_profile.country` remains migration-compatible and maps to the
country-level registry location when structured `location_ids` are absent.
Non-location dimensions remain tolerant of unknown context:
`device_info.device_type` and `device_info.os` are required in the canonical
public Query contract; callers that cannot determine device context MUST send
`device_type: "other"` and `os: "other"`.

`context.user_profile.verified_age_over` carries non-PII verified threshold
claims such as `[18]`. It is used to satisfy `targeting[].eligibility.min_age`;
the Query API does not accept date of birth or exact age.

### Device, OS, Location, and Age Context

`context.user_profile.device_info` carries REQUIRED viewer context for targeting
and rendering decisions. It is not a semantic ranking prompt and it is not a
hard query constraint.

Required fields:

| Field | Level | Notes |
|-------|-------|-------|
| `context.user_profile.device_info` | REQUIRED | Every Query request must carry a device context object. |
| `context.user_profile.device_info.device_type` | REQUIRED | Use `other` when the form factor is unknown. |
| `context.user_profile.device_info.os` | REQUIRED | Use `other` when the OS is unknown or not in the public enum. |
| `context.user_profile.device_info.os_version` | OPTIONAL | Free-form string; no public version grammar. |
| `context.user_profile.device_info.user_agent` | OPTIONAL | Raw or normalized user-agent string for diagnostics and compatibility. Do not use as a stable viewer identifier. |

Canonical public `device_type` values:

| Value | Meaning |
|-------|---------|
| `desktop` | Desktop or laptop-class browser/app environment. |
| `mobile` | Phone-class mobile device. |
| `tablet` | Tablet-class device. |
| `smart_tv` | TV or connected-TV environment. |
| `other` | Known device that does not fit the current public enum. |

Canonical public `os` values:

| Value | Meaning |
|-------|---------|
| `ios` | Apple iOS family, including iPadOS for public Query context. |
| `android` | Android-based phone/tablet devices. |
| `windows` | Microsoft Windows. |
| `macos` | Apple macOS. |
| `other` | Known OS outside the current public enum. |

Clients SHOULD send lower_snake_case canonical values.

Public Query context intentionally does not expose `ipados`, `chromeos`, or
`linux` as OS enum values:

| Source value | Public Query handling |
|--------------|-----------------------|
| iPadOS | Send `os: "ios"` and `device_type: "tablet"` when distinguishable. |
| ChromeOS | Send `os: "other"` for now. |
| Linux | Send `os: "other"` for now. |
| Unknown/headless/server-side | Send `os: "other"` and `device_type: "other"`. |

Services MAY normalize common source aliases such as `iOS` to `ios`, `macOS`
to `macos`, `Chrome OS` to `other`, and `phone` to `mobile`, but those aliases
are compatibility inputs and not public canonical values.

`context.user_profile.country` is an OPTIONAL uppercase ISO 3166-1 alpha-2
country code such as `US`, `SG`, or `JP`. It is a legacy viewer context
attribute, not `constraints.country`.

`context.user_profile.location_ids` is an OPTIONAL array of numeric-string AON
Location Registry v1 IDs, such as `["1014221", "21137", "2840"]`. It is the
canonical location targeting input.

`context.user_profile.verified_age_over` is an OPTIONAL array of integer
thresholds from 13 through 120, such as `[18]`. It communicates verified
eligibility thresholds only, not exact age.

`context.user_profile.device_info.os_version` remains an OPTIONAL free-form
string because OS version formats differ by platform and vendor.

`context.user_profile.device_info.user_agent` is OPTIONAL. Callers MAY include a
raw HTTP `User-Agent` value or a platform-normalized equivalent when it helps
debug rendering or compatibility issues. Services MUST NOT treat it as a stable
identity key, and implementations SHOULD truncate or ignore unusually long
values.

### Protocol Versioning

`/v1/offers/query` identifies the hosted Query API major version. The request
SHOULD also include `AON-Protocol-Version: 0.1` to pin the AgentOffer Protocol
payload contract. SDKs send this header by default.

Servers MAY treat an omitted `AON-Protocol-Version` header as the current
default protocol version and SHOULD echo the resolved version in the response
header. Unsupported protocol versions SHOULD return a clear client error that
lists supported versions.

Do not put the protocol version in the JSON request body. The body remains the
portable protocol payload, while the HTTP header carries transport-level
contract negotiation.

### Eligibility and Lifecycle Status

The Query API returns active, eligible offers by default. Client requests SHOULD
NOT include lifecycle status constraints such as `active`, `paused`, `pending`,
`rejected`, or `expired`.

`offer_info.status` remains part of the Offer Schema for inventory lifecycle
representation. Lifecycle filtering belongs to supply-side or internal
selection logic, not the agent-facing Query API public constraints surface.

The legacy root `filter` field is no longer part of the canonical
Query API request. Provider-facing OfferProvider API payloads also use root
`constraints` so AON-to-Partner dispatches and agent-facing Query requests share
the same field name.

## Response Shape

| Field | Notes |
|------|-------|
| `request_id` | Echoes the request identifier. If the client omitted it, the server-generated value is returned. |
| `offers[]` | Ranked offer objects. Empty array means no eligible offer matched the request. |
| `offers[].offer_id` | Stable inventory-level Offer ID. |
| `offers[].offer_instance_id` | Per-dispatch Offer instance ID. Use this for click -> conversion -> settlement attribution. |
| `offers[].offer_info` | Title, offer type, category, description, and commercial details. |
| `offers[].entity` | Provider or advertiser identity. |
| `offers[].action` | User action target, such as web redirect or deep link. |
| `offers[].bid` | Payout model and amount/rate information. Follow [`offer-schema.md`](offer-schema.md). |

The canonical `offers.query` response does **not** include `query_id`, `trace_id`, `has_more`, or `total` as top-level public response fields. Historical or internal uses must be labeled as such; see [`contract-governance.md`](contract-governance.md).

### Response Headers

| Header | Notes |
|--------|-------|
| `X-AON-TRACE-ID` | Diagnostic trace identifier generated by the hosted AON API for support and cross-service observability. This is an HTTP response header only. It MUST NOT be sent in the JSON request body and MUST NOT appear as `trace_id` or `aon_trace_id` in the canonical JSON response payload. |

### Empty Results and HTTP Status

An empty result is a successful Query API outcome: the request was processed,
but no active eligible offer matched the intent and context. The canonical
response is HTTP `200 OK` with an empty `offers` array:

```json
{
  "request_id": "019dd200-1234-7890-abcd-ef0123456789",
  "offers": []
}
```

Do not use HTTP `204 No Content` as the canonical empty-offer response for
`/v1/offers/query`. `204` cannot carry `request_id`, the hosted platform
envelope, diagnostics, or SDK-visible empty-state metadata. OpenRTB's
`204 No Content` no-bid convention is an auction transport optimization; AON
Query empty results are processed recommendation results, not bid refusals.

### Protocol Payload vs Platform Envelope

This document defines the protocol payload: `request_id` + `offers[]`.

The hosted platform API may also add HTTP transport headers such as
`X-AON-TRACE-ID`. Those headers are useful for diagnostics, but they do not
change the protocol JSON payload shape.

The hosted platform API may wrap this payload in a service envelope such as
`code`, `message`, `data`, and `extra`. In that case, the protocol payload lives
inside `data`. Do not treat the service envelope as part of the canonical
AgentOffer Protocol response shape.

For field-level platform API tables, examples, and onboarding guidance, use the
[AON API Reference](https://docs.aon.pro/api/offer-query).

```json
{
  "request_id": "019dd200-1234-7890-abcd-ef0123456789",
  "offers": [
    {
      "offer_id": "0195ef94-f17d-7a4f-b6e0-2c52bb49e13f",
      "offer_instance_id": "019dd208-27d2-7673-b16f-2c52bb49e13f",
      "version": "1.0",
      "offer_info": {
        "title": "The Tokyo Grand Weekend Stay",
        "offer_type": "offline_service",
        "category": {
          "id": "travel_tourism.accommodations_hotels"
        },
        "description": "Luxury hotel stay in central Tokyo with weekend availability."
      },
      "entity": { "id": "ent_tokyo_grand", "name": "Tokyo Grand Hotel" },
      "action": {
        "type": "web_redirect",
        "payload": { "target": "https://travel.example/hotels/tokyo-grand" }
      },
      "bid": { "model": "cpa", "amount": "12.50", "currency": "USD" }
    }
  ]
}
```

For complete request and response payloads, see [`agentoffernetwork/examples`](https://github.com/agentoffernetwork/examples).

## Errors

| Symptom | Likely cause | What to check |
|---------|--------------|---------------|
| `400 BAD_REQUEST` | Malformed JSON or missing required fields | Confirm `context`, `context.user_profile`, `intent.content[]`, and content item `type`. |
| `401 UNAUTHORIZED` | Missing or invalid bearer token | Use `Authorization: Bearer YOUR_API_KEY`. Never paste real keys into examples or issues. |
| `403 FORBIDDEN` | Key is valid but not allowed for this endpoint | Confirm the key is active and has Query API access. |
| `429 RATE_LIMITED` | Too many requests | Add backoff and retry later. |
| Empty `offers` | No eligible offer matched the hard constraints | Loosen `constraints.category_ids` or rely on `intent.content[]` for semantic matching. |
| Attribution mismatch | Wrong identifier propagated | Use `offers[].offer_instance_id` for the dispatched instance; use `offer_id` only for inventory identity. |

## References

| Need | File |
|------|------|
| Validate query requests | [`offer-query-schema-v0.1.json`](https://github.com/agentoffernetwork/schema/blob/main/json-schema/offer-query-schema-v0.1.json) |
| Inspect complete examples | [`agentoffernetwork/examples`](https://github.com/agentoffernetwork/examples) |
| Read platform API field tables | [AON API Reference](https://docs.aon.pro/api/offer-query) |
| Run a guided first request | [Docs Quick Start](https://docs.aon.pro/quickstart/first-api-call) |
| Understand returned offers | [`offer-schema.md`](offer-schema.md) |
| Choose categories | [`category-taxonomy.md`](category-taxonomy.md) |
| Check field lifecycle | [`contract-governance.md`](contract-governance.md) |
| Propose protocol changes | [`agentoffernetwork/rfcs`](https://github.com/agentoffernetwork/rfcs) |

## Appendix

<details>
<summary>Conformance keywords</summary>

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHOULD", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

| Level | Meaning |
|-------|---------|
| **REQUIRED** | Field MUST be present with a valid, non-empty value. |
| **RECOMMENDED** | Field SHOULD be present and MUST follow the standard structure when present, but the value MAY be empty or null. |
| **OPTIONAL** | Field MAY be omitted entirely. When included, it SHOULD follow the specified format. |

</details>

<details>
<summary>Design decisions</summary>

- **POST with structured body**: intent, context, constraints, and user profile are better expressed as JSON than URL parameters.
- **Multimodal intent**: `intent.content[]` mirrors LLM message formats and can evolve beyond text.
- **Context separation**: platform, session, and user profile are separate so each layer can evolve independently.
- **Pseudonymous user identifiers**: `user_pseudo_id` is sufficient for personalization and frequency capping; real user IDs are not required.
- **Structured constraints + semantic intent**: `constraints` provides hard eligibility constraints while `intent` provides relevance ranking.
- **Lifecycle status boundary**: agent-facing Query API requests return active eligible offers by default; lifecycle status remains an offer/provider concern, not a public query constraint.
- **Protocol version header**: `/v1` identifies the hosted API major version; `AON-Protocol-Version` pins the protocol payload contract without adding version fields to JSON bodies.
- **Offset pagination**: simple default for v0.1; cursor pagination may be introduced later.

</details>

<details>
<summary>Changelog</summary>

| Version | Date | Changes |
|---------|------|---------|
| 0.1 | 2026-03-20 | Initial draft. |
| 0.1 | 2026-03-22 | Added broader compatibility-oriented narrowing fields. |
| 0.1 | 2026-03-23 | Added CTA-oriented action semantics to the response example. |
| 0.1 | 2026-03-24 | Reframed the query result as `offer response { trace_id, offers[] }`, aligned the payload with `offer`, and updated key field names. |
| 0.1 | 2026-03-25 | Restructured from GET parameters to POST JSON body. Introduced `context`, multimodal `intent.content[]`, requirement levels, and offset-based pagination. |
| 0.1 | 2026-03-28 | Added a structured query narrowing object and enum extensibility note. |
| 0.1 | 2026-03-31 | Changed `request_id` and `timestamp` from REQUIRED to OPTIONAL. Server generates defaults when omitted. |
| 0.1 | 2026-05-05 | Clarified that the response envelope uses `request_id` rather than `trace_id`, removes `has_more` / `total` response metadata, and aligns examples with current offer identity fields. |
| 0.1 | 2026-05-09 | Reorganized the GitHub reference for developer readability and density. |
| 0.1 | 2026-05-14 | Removed agent-facing `filter.status`, clarified active eligible offer defaults, and added `AON-Protocol-Version` header guidance. |
| 0.1 | 2026-05-15 | Renamed agent-facing root `filter` to `constraints` and limited the first public constraints surface to `category_types`. |
| 0.1 | 2026-05-27 | Upgraded category constraints to AON Taxonomy v1 `category_ids` with subtree matching semantics. |
| 0.1 | 2026-05-19 | Added canonical Query device/OS context values, uppercase country format, and clarified empty results use `200 OK` with `offers: []` rather than `204 No Content`. |
| 0.1 | 2026-05-22 | Documented `X-AON-TRACE-ID` as a hosted Query API response header and clarified that trace identifiers are not JSON body fields. |
| 0.1 | 2026-05-22 | Added OPTIONAL `context.user_profile.device_info.user_agent` for diagnostics and compatibility. |
| 0.1 | 2026-06-03 | Aligned examples and field guidance with Taxonomy v1 `constraints.category_ids`, required `context.user_profile.device_info`, and current category examples. |
| 0.1 | 2026-06-09 | Added canonical location targeting via AON Location Registry v1 `location_id` values and non-PII age threshold targeting via `verified_age_over` / `eligibility.min_age`. |

</details>
