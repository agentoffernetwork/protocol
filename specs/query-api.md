# Offer Query API v0.1

- **Version**: 0.1
- **Status**: Draft
- **Last Updated**: 2026-05-09
- **Source of truth**: `agentoffernetwork/protocol/specs/query-api.md`

This document defines the HTTP query interface that AI agents and SDKs use to discover offers from an AgentOffer-compatible service.

## At a Glance

| Item | Value |
|------|-------|
| Method | `POST` |
| Path | `/v1/offers/query` |
| Auth | `Authorization: Bearer YOUR_API_KEY` |
| Content-Type | `application/json` |
| Request contract | `context` + `intent`, with optional `filter` and `pagination` |
| Response contract | `request_id` + `offers[]` |
| Schema companion | [`offer-query-schema-v0.1.json`](https://github.com/agentoffernetwork/schema/blob/main/json-schema/offer-query-schema-v0.1.json) |
| Example companion | [`offer-query-request.json`](https://github.com/agentoffernetwork/examples/blob/main/http/offer-query-request.json) |

Use this API when an agent has user intent and wants ranked commercial offers it can recommend or present.

## 5-Minute Integration Path

1. Copy the [minimal request](#minimal-request) and replace `YOUR_API_KEY`.
2. Fill the two required objects: [`context`](#required-fields) and [`intent`](#required-fields).
3. Add [`filter`](#optional-fields) only when you need hard constraints such as category, country, price, or bid model.
4. Read the [canonical response shape](#success-response) and use `offers[].offer_instance_id` for click/conversion attribution.
5. Validate payloads with the [schema repository](https://github.com/agentoffernetwork/schema) and compare with [examples](https://github.com/agentoffernetwork/examples).

## Conformance Keywords

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHOULD", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

| Level | Meaning |
|-------|---------|
| **REQUIRED** | Field MUST be present with a valid, non-empty value. |
| **RECOMMENDED** | Field SHOULD be present and MUST follow the standard structure when present, but the value MAY be empty or null. |
| **OPTIONAL** | Field MAY be omitted entirely. When included, it SHOULD follow the specified format. |

## Minimal Request

This is the shortest useful request shape for a first integration. Production systems SHOULD add richer platform, session, user profile, filter, and pagination fields when available.

```bash
curl -s -X POST "https://api.agentoffernetwork.com/v1/offers/query" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "context": {
      "user_profile": {}
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

## Request Body

### Required Fields

| Field | Type | Description |
|------|------|-------------|
| `context` | object | Platform, session, and user context used for matching and targeting. |
| `context.user_profile` | object | User profile container. It may be sparse, but the object must be present. |
| `intent` | object | The user's intent expressed as multimodal content. |
| `intent.content` | array | One or more content items. At least one item is REQUIRED. |
| `intent.content[].type` | string | Content type. Current values: `input_text`, `input_image`. |
| `intent.content[].text` | string | Text intent. REQUIRED when `type` is `input_text`. |
| `intent.content[].image_url` | string | Image URL. REQUIRED when `type` is `input_image`. |

### Recommended Fields

| Field | Type | Description |
|------|------|-------------|
| `context.platform` | object | Requesting platform or agent metadata. |
| `context.platform.name` | string | Platform or agent name, for example `TravelBot` or `CustomAgent`. |
| `context.platform.version` | string | Platform, agent, or model version. |
| `context.platform.channel` | string | Integration channel, for example `api`, `sdk`, `plugin`, or `skill`. |
| `context.session_id` | string | Session identifier for grouping related queries. |
| `context.user_profile.user_pseudo_id` | string | Pseudonymous viewer identifier. Do not send raw personal identifiers unless your integration is authorized to do so. |
| `context.user_profile.language` | string | User language preference. ISO 639-1 code is recommended. |
| `context.user_profile.interests` | string[] | User interest tags. May be an empty array. |
| `context.user_profile.device_info` | object | Device information used for targeting and rendering. |
| `pagination` | object | Pagination control. Defaults apply when omitted. |
| `pagination.limit` | integer | Number of offers to return. Default: `20`. Maximum: `100`. |
| `pagination.offset` | integer | Number of offers to skip. Default: `0`. |

### Optional Fields

| Field | Type | Description |
|------|------|-------------|
| `request_id` | string | Unique request identifier. UUIDv7 is recommended. When omitted, the server generates one. |
| `timestamp` | string | RFC 3339 timestamp. When omitted, the server uses the current time. |
| `test_mode` | boolean | When `true`, the request is treated as a test and SHOULD NOT generate real tracking or billing events. Defaults to `false`. |
| `context.conversation_id` | string/number | Conversation or thread identifier within the session. |
| `filter` | object | Hard constraints applied before semantic ranking. Omit it when intent-only matching is enough. |
| `filter.category_types` | string[] | Category filters. OR logic within the array. |
| `filter.bid_models` | string[] | Bid model filters. OR logic within the array. |
| `filter.status` | string[] | Offer lifecycle status filters. |
| `filter.availability` | string[] | Availability filters. |
| `filter.min_bid_amount` | string | Minimum bid amount as a decimal string. Requires `filter.currency`. |
| `filter.max_price_amount` | string | Maximum consumer-facing price as a decimal string. Requires `filter.currency`. |
| `filter.currency` | string | ISO 4217 currency code for amount filters. |
| `filter.brand` | string | Brand or entity-name substring filter. |
| `filter.country` | string | Target country. ISO 3166-1 alpha-2 code. |
| `filter.tags` | string[] | Tag filters. AND logic: the offer should match all specified tags. |

<details>
<summary>Full field notes</summary>

### Context

`context` provides environment and user information. Keep it privacy-preserving: use pseudonymous IDs and coarse preferences unless the user and integration explicitly permit more detail.

`context.user_profile.device_info` may include:

| Field | Type | Description |
|------|------|-------------|
| `device_type` | string | Device type such as `desktop`, `mobile`, or `tablet`. |
| `os` | string | Operating system such as `macOS`, `Windows`, `iOS`, or `Android`. |
| `os_version` | string | OS version string. |

### Intent

`intent.content[]` is a typed multimodal array similar to LLM message formats. `input_text` is the primary signal. `input_image` supports visual search scenarios.

Future content types can be added without changing the array structure. Clients should ignore unknown content types they do not support and preserve the payload when proxying.

### Filter

`filter` narrows the candidate set before semantic ranking:

- Array filters use OR logic within the array, for example `category_types: ["software_saas", "education"]`.
- `tags` uses AND logic: an offer should match all specified tags.
- `min_bid_amount` and `max_price_amount` require `currency`; without currency, servers may ignore amount filters.
- `brand` is intended as a case-insensitive match against provider or entity names.

### Pagination

The initial design uses offset pagination for simplicity. Cursor pagination may be introduced in a future revision if large result sets require it.

</details>

## Common Values

Enums follow the protocol's open-ended enum design. Servers SHOULD handle unknown values gracefully, usually by returning no matching offers instead of failing the entire request.

| Field | Current values |
|------|----------------|
| `intent.content[].type` | `input_text`, `input_image` |
| `filter.category_types` / `offer_info.category.type` | `software_saas`, `travel_hospitality`, `education`, `financial_service`, `electronics`, `entertainment`, `health_beauty`, `fashion`, `food_grocery`, `home_garden`, `automotive` |
| `filter.bid_models` / `bid.model` | `cpa`, `cps`, `cpl`, `cpi`, `hybrid` |
| `filter.status` / `offer_info.status` | `active`, `paused`, `pending`, `rejected`, `expired` |
| `filter.availability` / `offer_info.category.commercial.availability` | `available`, `limited`, `sold_out`, `pre_order` |
| `offer_info.offer_type` | `physical_product`, `digital_goods`, `content`, `online_service`, `offline_service` |

For category boundaries and sub-types, see [`category-taxonomy.md`](category-taxonomy.md) and [`offer-schema.md`](offer-schema.md).

## Request Example

```json
{
  "request_id": "019dd200-1234-7890-abcd-ef0123456789",
  "timestamp": "2026-04-28T03:30:00Z",
  "test_mode": false,
  "context": {
    "platform": {
      "name": "TravelBot",
      "version": "2.1.0",
      "channel": "api"
    },
    "session_id": "sess_abc123",
    "user_profile": {
      "user_pseudo_id": "viewer_xyz",
      "language": "en",
      "interests": ["travel", "hotels", "luxury"],
      "device_info": {
        "device_type": "mobile",
        "os": "ios",
        "os_version": "18.2"
      }
    }
  },
  "intent": {
    "content": [
      {
        "type": "input_text",
        "text": "Find me a luxury hotel in Tokyo under $300/night for a weekend trip"
      }
    ]
  },
  "filter": {
    "category_types": ["travel_hospitality"],
    "bid_models": ["cpa", "cps"],
    "status": ["active"],
    "min_bid_amount": "5.00",
    "max_price_amount": "300.00",
    "currency": "USD",
    "brand": "Hilton",
    "country": "JP"
  },
  "pagination": {
    "limit": 10,
    "offset": 0
  }
}
```

## Success Response

The response envelope is intentionally small and canonical:

| Field | Type | Description |
|------|------|-------------|
| `request_id` | string | Echoes the request identifier. If the client omitted it, the server-generated value is returned. |
| `offers` | array | Ranked `Offer` objects matching the request. Empty array means no eligible offer. |
| `offers[].offer_id` | string | Stable inventory-level Offer ID. |
| `offers[].offer_instance_id` | string | Per-dispatch Offer instance ID generated for this presentation. Use this for click -> conversion -> settlement attribution. |
| `offers[].offer_info` | object | Human-readable title, offer type, category, description, and commercial details. |
| `offers[].entity` | object | Provider or advertiser identity. |
| `offers[].action` | object | User action target, such as web redirect or deep link. |
| `offers[].bid` | object | Payout model and amount/rate information. Follow [`offer-schema.md`](offer-schema.md) for the current requirement level and model-specific rules. |

The canonical `offers.query` response does **not** include `query_id`, `trace_id`, `has_more`, or `total` as top-level public response fields. Historical or internal uses must be labeled as such; see [`contract-governance.md`](contract-governance.md).

## Response Example

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
          "type": "travel_hospitality",
          "attributes": {
            "sub_type": "hotel",
            "property_type": "hotel",
            "destination": {
              "city": "Tokyo",
              "country": "JP"
            },
            "star_rating": 5
          }
        },
        "description": "Luxury hotel stay in central Tokyo with weekend availability.",
        "commercial": {
          "price": {
            "amount": "289.00",
            "currency": "USD"
          },
          "availability": "limited"
        }
      },
      "entity": {
        "id": "ent_tokyo_grand",
        "name": "Tokyo Grand Hotel",
        "type": "business"
      },
      "action": {
        "type": "web_redirect",
        "name": "Book now",
        "payload": {
          "target": "https://travel.example/hotels/tokyo-grand"
        }
      },
      "bid": {
        "model": "cpa",
        "amount": "12.50",
        "currency": "USD"
      }
    }
  ]
}
```

## Troubleshooting

| Symptom | Likely cause | What to check |
|---------|--------------|---------------|
| `400 BAD_REQUEST` | Malformed JSON or missing required fields | Confirm `context`, `context.user_profile`, `intent.content[]`, and content item `type` are present. |
| `401 UNAUTHORIZED` | Missing or invalid bearer token | Use `Authorization: Bearer YOUR_API_KEY`. Do not paste real keys into examples or issues. |
| `403 FORBIDDEN` | Key is valid but not allowed for this environment or endpoint | Confirm the key is active and has Query API access. |
| `429 RATE_LIMITED` | Too many requests | Add backoff and retry later. |
| Empty `offers` | No eligible offer matched the hard filters | Remove or loosen `filter`, especially category, country, brand, price, and bid constraints. |
| Amount filters ignored | Missing `filter.currency` | Send `currency` with `min_bid_amount` or `max_price_amount`. |
| Pagination repeats results | Incorrect `offset` | Increase `offset` by the previous `limit`. |
| Attribution cannot be reconciled | Wrong identifier propagated | Use `offers[].offer_instance_id` for the dispatched offer instance; use `offer_id` only for inventory-level identity. |

## Related Files

| Need | File |
|------|------|
| Canonical offer object | [`offer-schema.md`](offer-schema.md) |
| Category boundaries | [`category-taxonomy.md`](category-taxonomy.md) |
| Field lifecycle and stale-field handling | [`contract-governance.md`](contract-governance.md) |
| JSON Schema validation | [`agentoffernetwork/schema`](https://github.com/agentoffernetwork/schema) |
| Request/response examples | [`agentoffernetwork/examples`](https://github.com/agentoffernetwork/examples) |
| Protocol changes | [`agentoffernetwork/rfcs`](https://github.com/agentoffernetwork/rfcs) |

## Design Decisions

- **POST with structured body**: intent, context, filters, and user profile are better expressed as JSON than URL parameters.
- **Multimodal intent**: `intent.content[]` mirrors LLM message formats and can evolve beyond text.
- **Context separation**: platform, session, and user profile are separate so each layer can evolve independently.
- **Pseudonymous user identifiers**: `user_pseudo_id` is sufficient for personalization and frequency capping; real user IDs are not required.
- **Structured filter + semantic intent**: `filter` provides hard constraints while `intent` provides relevance ranking.
- **Offset pagination**: simple default for v0.1; cursor pagination may be introduced later.

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1 | 2026-03-20 | Initial draft. |
| 0.1 | 2026-03-22 | Added broader compatibility-oriented filters. |
| 0.1 | 2026-03-23 | Added CTA-oriented action semantics to the response example. |
| 0.1 | 2026-03-24 | Reframed the query result as `offer response { trace_id, offers[] }`, aligned the payload with `offer`, and updated key field names. |
| 0.1 | 2026-03-24 | Added `offer-query` example guidance and clarified the boundary between query request, canonical `offer`, and `offer response`. |
| 0.1 | 2026-03-24 | Updated the response example to the `offer_id + offer_instance_id + offer_info + entity + action + targeting + bid` draft shape. |
| 0.1 | 2026-03-25 | Restructured from GET parameters to POST JSON body. Introduced `context`, multimodal `intent.content[]`, requirement levels, and offset-based pagination. |
| 0.1 | 2026-03-28 | Added `filter` object for structured query constraints and enum extensibility note. |
| 0.1 | 2026-03-31 | Changed `request_id` and `timestamp` from REQUIRED to OPTIONAL. Server generates defaults when omitted. |
| 0.1 | 2026-05-05 | Clarified that the response envelope uses `request_id` rather than `trace_id`, removes `has_more` / `total` response metadata, and aligns examples with current offer identity fields. |
| 0.1 | 2026-05-09 | Reorganized the GitHub reference for developer readability: added at-a-glance summary, minimal request, required-first field guide, common values, troubleshooting, and related files. |
