# Offer Query API v0.1

- **Status**: Draft
- **Last Updated**: 2026-05-09
- **Source**: `agentoffernetwork/protocol/specs/query-api.md`

Use this API when an agent has user intent and needs ranked commercial offers to recommend, compare, or present.

## Endpoint

| Item | Value |
|------|-------|
| Method | `POST` |
| Path | `/v1/offers/query` |
| Auth | `Authorization: Bearer YOUR_API_KEY` |
| Content-Type | `application/json` |
| Request | `context` + `intent`, optional `filter` and `pagination` |
| Response | `request_id` + `offers[]` |

## Minimal Request

```bash
curl -s -X POST "https://api.agentoffernetwork.com/v1/offers/query" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "context": { "user_profile": {} },
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
| `context.user_profile` | REQUIRED | User profile container. It may be sparse, but the object must be present. |
| `intent` | REQUIRED | User intent expressed as multimodal content. |
| `intent.content[]` | REQUIRED | At least one content item. Current `type` values: `input_text`, `input_image`. |
| `filter` | OPTIONAL | Hard constraints applied before semantic ranking. |
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
| `context.user_profile.interests` | string[] | `["travel", "hotels"]` |
| `intent.content[].text` | string | `Find me a luxury hotel in Tokyo` |
| `filter.category_types` | string[] | `["travel_hospitality"]` |
| `filter.bid_models` | string[] | `["cpa", "cps"]` |
| `filter.status` | string[] | `["active"]` |
| `filter.currency` | string | `USD` |
| `filter.max_price_amount` | string | `"300.00"` |
| `pagination.limit` | integer | `10` |
| `pagination.offset` | integer | `0` |

### Common Values

| Field | Common values |
|------|---------------|
| `intent.content[].type` | `input_text`, `input_image` |
| `filter.category_types` | `software_saas`, `travel_hospitality`, `education`, `financial_service`, `electronics` |
| `filter.bid_models` | `cpa`, `cps`, `cpl`, `cpi`, `hybrid` |
| `filter.status` | `active`, `paused`, `pending`, `rejected`, `expired` |
| `offer_info.offer_type` | `physical_product`, `digital_goods`, `content`, `online_service`, `offline_service` |

See [`category-taxonomy.md`](category-taxonomy.md) and [`offer-schema.md`](offer-schema.md) for the full category, offer, bid, and status definitions.

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

```json
{
  "request_id": "019dd200-1234-7890-abcd-ef0123456789",
  "offers": [
    {
      "offer_id": "0195ef94-f17d-7a4f-b6e0-2c52bb49e13f",
      "offer_instance_id": "019dd208-27d2-7673-b16f-2c52bb49e13f",
      "offer_info": {
        "title": "The Tokyo Grand Weekend Stay",
        "offer_type": "offline_service",
        "category": { "type": "travel_hospitality" },
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
| Empty `offers` | No eligible offer matched the hard filters | Loosen `filter`, especially category, country, brand, price, and bid constraints. |
| Amount filters ignored | Missing `filter.currency` | Send `currency` with `min_bid_amount` or `max_price_amount`. |
| Attribution mismatch | Wrong identifier propagated | Use `offers[].offer_instance_id` for the dispatched instance; use `offer_id` only for inventory identity. |

## References

| Need | File |
|------|------|
| Validate query requests | [`offer-query-schema-v0.1.json`](https://github.com/agentoffernetwork/schema/blob/main/json-schema/offer-query-schema-v0.1.json) |
| Inspect complete examples | [`agentoffernetwork/examples`](https://github.com/agentoffernetwork/examples) |
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

- **POST with structured body**: intent, context, filters, and user profile are better expressed as JSON than URL parameters.
- **Multimodal intent**: `intent.content[]` mirrors LLM message formats and can evolve beyond text.
- **Context separation**: platform, session, and user profile are separate so each layer can evolve independently.
- **Pseudonymous user identifiers**: `user_pseudo_id` is sufficient for personalization and frequency capping; real user IDs are not required.
- **Structured filter + semantic intent**: `filter` provides hard constraints while `intent` provides relevance ranking.
- **Offset pagination**: simple default for v0.1; cursor pagination may be introduced later.

</details>

<details>
<summary>Changelog</summary>

| Version | Date | Changes |
|---------|------|---------|
| 0.1 | 2026-03-20 | Initial draft. |
| 0.1 | 2026-03-22 | Added broader compatibility-oriented filters. |
| 0.1 | 2026-03-23 | Added CTA-oriented action semantics to the response example. |
| 0.1 | 2026-03-24 | Reframed the query result as `offer response { trace_id, offers[] }`, aligned the payload with `offer`, and updated key field names. |
| 0.1 | 2026-03-25 | Restructured from GET parameters to POST JSON body. Introduced `context`, multimodal `intent.content[]`, requirement levels, and offset-based pagination. |
| 0.1 | 2026-03-28 | Added `filter` object for structured query constraints and enum extensibility note. |
| 0.1 | 2026-03-31 | Changed `request_id` and `timestamp` from REQUIRED to OPTIONAL. Server generates defaults when omitted. |
| 0.1 | 2026-05-05 | Clarified that the response envelope uses `request_id` rather than `trace_id`, removes `has_more` / `total` response metadata, and aligns examples with current offer identity fields. |
| 0.1 | 2026-05-09 | Reorganized the GitHub reference for developer readability and density. |

</details>
