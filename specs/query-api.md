# Offer Query API v0.1

**Version**: 0.1
**Status**: Draft
**Last Updated**: 2026-03-25

## Introduction

This document defines the HTTP query interface that AI agents and SDKs use to discover offers from an AgentOffer-compatible service.

The API is organized around two protocol concepts:

- `offer query`: the structured request submitted by agents to express intent and context
- `offer response`: the query envelope that wraps one result set of `offer` objects

### Conformance Keywords

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHOULD", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Endpoint Overview

- Method: `POST`
- Path: `/v1/offers/query`
- Content-Type: `application/json`
- Purpose: Submit intent and context to discover relevant offers for recommendation and action-taking flows

## Authentication

Requests MUST include a bearer token in the `Authorization` header.

```http
Authorization: Bearer {api_key}
```

## Request Body

The query request uses the same REQUIRED/RECOMMENDED/OPTIONAL field requirement levels as the Offer Schema:

| Level | Label | Meaning |
|-------|-------|---------|
| **REQUIRED** | Required | Field MUST be present with a valid, non-empty value. |
| **RECOMMENDED** | Recommended | Field SHOULD be present and MUST follow the standard structure when present, but the value MAY be empty or null. |
| **OPTIONAL** | Optional | Field MAY be omitted entirely. When included, it SHOULD follow the specified format. |

### Top-Level Shape

| Field | Type | Level | Description |
|------|------|-------|-------------|
| `request_id` | string | REQUIRED | Unique request identifier. UUIDv7 is recommended. |
| `timestamp` | string | REQUIRED | RFC 3339 timestamp of the request. |
| `test_mode` | boolean | OPTIONAL | When `true`, the request is treated as a test and SHOULD NOT generate real tracking or billing events. Defaults to `false`. |
| `context` | object | REQUIRED | Contextual information about the requesting platform, session, and user. |
| `intent` | object | REQUIRED | The user's intent expressed as multimodal content. |
| `filter` | object | OPTIONAL | Structured filter constraints. Hard constraints applied before intent-based semantic ranking. |
| `pagination` | object | RECOMMENDED | Pagination control. Field SHOULD be present; values have defaults. |

### Context

`context` provides the environment and user information that the offer matching engine uses for personalization and targeting.

| Field | Type | Level | Description |
|------|------|-------|-------------|
| `context.platform` | object | RECOMMENDED | Information about the requesting platform or agent. |
| `context.session_id` | string | RECOMMENDED | Session identifier for grouping related queries. |
| `context.conversation_id` | string/number | OPTIONAL | Conversation or thread identifier within the session. |
| `context.user_profile` | object | REQUIRED | User profile information for intent matching and targeting. |

#### `context.platform`

| Field | Type | Level | Description |
|------|------|-------|-------------|
| `platform.name` | string | RECOMMENDED | Platform or agent name (e.g., `ChatGPT`, `Claude`, `CustomAgent`). |
| `platform.version` | string | RECOMMENDED | Platform or model version (e.g., `gpt-4o`, `claude-sonnet-4-6`). |
| `platform.channel` | string | RECOMMENDED | Integration channel (e.g., `plugin`, `sdk`, `api`, `skill`). |

#### `context.user_profile`

| Field | Type | Level | Description |
|------|------|-------|-------------|
| `user_profile.viewer_id` | string | RECOMMENDED | Pseudonymous viewer identifier for frequency capping and personalization. Not required to be a real user ID. |
| `user_profile.language` | string | RECOMMENDED | User language preference. ISO 639-1 code. |
| `user_profile.interests` | array | RECOMMENDED | User interest tags for intent matching. MAY be an empty array. |
| `user_profile.device_info` | object | RECOMMENDED | Device information for targeting and rendering. |

#### `context.user_profile.device_info`

| Field | Type | Level | Description |
|------|------|-------|-------------|
| `device_info.device_type` | string | RECOMMENDED | Device type such as `desktop`, `mobile`, `tablet`. |
| `device_info.os` | string | RECOMMENDED | Operating system such as `macOS`, `Windows`, `iOS`, `Android`. |
| `device_info.os_version` | string | RECOMMENDED | OS version string. |

### Intent

`intent` expresses what the user is looking for. It uses a multimodal content array to support text, images, and other input types.

| Field | Type | Level | Description |
|------|------|-------|-------------|
| `intent.content` | array | REQUIRED | Array of content items representing the user's intent. At least one item is REQUIRED. |

#### `intent.content[]`

Each content item has a `type` field that determines the payload:

| Field | Type | Level | Description |
|------|------|-------|-------------|
| `content[].type` | string | REQUIRED | Content type such as `input_text` or `input_image`. |
| `content[].text` | string | REQUIRED (when type=input_text) | The text content of the user's intent. |
| `content[].image_url` | string | REQUIRED (when type=input_image) | URL to the image representing the user's intent. |

Design notes:

- The multimodal content array follows a pattern similar to LLM message formats, making it natural for AI agent platforms to construct.
- `input_text` is the primary intent signal. `input_image` supports visual search scenarios (e.g., "find me a hotel like this").
- Future content types (e.g., `input_audio`, `input_file`) can be added without changing the array structure.

### Filter

`filter` provides structured constraints that narrow the result set before semantic ranking. When both `filter` and `intent` are present, `filter` acts as a hard constraint (exact match) and `intent` acts as a soft signal (semantic relevance) within the filtered results.

| Field | Type | Level | Description |
|------|------|-------|-------------|
| `filter.category_types` | array | OPTIONAL | Filter by category type. Values reference `offer_info.category.type` enum (e.g., `["software_saas", "education"]`). |
| `filter.commission_models` | array | OPTIONAL | Filter by commission model. Values reference `commission.model` enum (e.g., `["cps", "cpa"]`). |
| `filter.status` | array | OPTIONAL | Filter by offer status (e.g., `["active"]`). |
| `filter.availability` | array | OPTIONAL | Filter by availability (e.g., `["available", "limited"]`). |
| `filter.min_commission_amount` | string | OPTIONAL | Minimum commission amount. Decimal string. Requires `filter.currency`. |
| `filter.max_price_amount` | string | OPTIONAL | Maximum consumer-facing price. Decimal string. Requires `filter.currency`. |
| `filter.currency` | string | OPTIONAL | ISO 4217 currency code for `min_commission_amount` and `max_price_amount`. Applies to both fields simultaneously; offers with mismatched currencies are excluded. |
| `filter.brand` | string | OPTIONAL | Filter by brand or entity name (case-insensitive substring match). |
| `filter.country` | string | OPTIONAL | Filter by target country. ISO 3166-1 alpha-2 code. |
| `filter.tags` | array | OPTIONAL | Filter by tags (AND logic: offer must match all specified tags). |

Design notes:

- `filter` is entirely OPTIONAL. When omitted, the query relies solely on `intent` for matching.
- Array-typed filters use OR logic within the array (e.g., `category_types: ["software_saas", "education"]` matches either).
- `min_commission_amount` and `max_price_amount` require `currency` to be set; if `currency` is absent, numeric filters are ignored.
- `brand` uses case-insensitive substring matching against `entity.name`.

> **Enum Extensibility**: All enum values referenced in filter fields (`category_types`, `commission_models`, `status`, `availability`) follow the protocol's open-ended enum design. Servers SHOULD accept unknown enum values gracefully (return empty results rather than errors). New enum values may be added in future revisions without being considered a breaking change.

### Pagination

| Field | Type | Level | Description |
|------|------|-------|-------------|
| `pagination.limit` | integer | RECOMMENDED | Number of offers to return. Default: `20`. Maximum: `100`. |
| `pagination.offset` | integer | RECOMMENDED | Number of offers to skip. Default: `0`. |

## Offer Response

The query result is wrapped in an `offer response` envelope.

| Field | Type | Level | Description |
|------|------|-------|-------------|
| `trace_id` | string | REQUIRED | Query trace ID generated by AON. UUIDv7 is recommended. |
| `offers` | array | REQUIRED | List of `offer` objects matching the intent. |
| `has_more` | boolean | OPTIONAL | Indicates whether another page is available. |
| `total` | integer | OPTIONAL | Total number of matching offers (when available). |

Design notes:

- `trace_id` belongs to the response envelope, not to the offer itself.
- `offers[]` SHOULD contain full `offer` objects as defined in the Offer Schema.
- Pagination shifted from cursor-based to offset-based to align with the request structure. Cursor-based pagination may be reintroduced in a future revision if needed for large result sets.

## Request Example

```json
{
  "request_id": "0195f0a1-2b3c-4d5e-6f7a-8b9c0d1e2f3a",
  "timestamp": "2026-03-25T14:30:00Z",
  "test_mode": false,
  "context": {
    "platform": {
      "name": "ChatGPT",
      "version": "gpt-4o",
      "channel": "plugin"
    },
    "session_id": "sess_0195f0a1-2b3c-4d5e-6f7a-8b9c0d1e2f3b",
    "user_profile": {
      "viewer_id": "user_789012",
      "language": "en",
      "interests": ["travel", "luxury", "hospitality"],
      "device_info": {
        "device_type": "desktop",
        "os": "macOS",
        "os_version": "14.4"
      }
    }
  },
  "intent": {
    "content": [
      {
        "type": "input_text",
        "text": "I want to book a 5-star hotel in Manhattan New York for next month"
      }
    ]
  },
  "filter": {
    "category_types": ["travel_hospitality"],
    "availability": ["available", "limited"],
    "currency": "USD",
    "max_price_amount": "500.00"
  },
  "pagination": {
    "limit": 5,
    "offset": 0
  }
}
```

## Response Example

```json
{
  "trace_id": "0195ef98-90af-7e1f-b57d-1130cf0c57f2",
  "offers": [
    {
      "uuid": "0195ef94-f17d-7a4f-b6e0-2c52bb49e142",
      "version": "1.0",
      "offer_info": {
        "title": "The Manhattan Grand — Deluxe King Room",
        "offer_type": "offline_service",
        "category": {
          "type": "travel_hospitality",
          "attributes": {
            "property_type": "hotel",
            "destination": { "city": "New York", "country": "US" },
            "star_rating": 5
          },
          "commercial": {
            "price": { "amount": "420.00", "currency": "USD" },
            "availability": "limited"
          }
        },
        "description": "Luxury 5-star hotel in Midtown Manhattan with rooftop pool, full-service spa, and complimentary breakfast."
      },
      "material": [],
      "entity": {
        "id": "ent_manhattan_grand",
        "name": "The Manhattan Grand Hotel",
        "type": "service_provider"
      },
      "action": {
        "type": "web_redirect",
        "name": "Book now",
        "payload": {
          "target": "https://www.manhattangrand.example/book/deluxe-king"
        }
      },
      "commission": {
        "model": "cps",
        "rate": "0.10",
        "currency": "USD",
        "payout_delay_days": 14,
        "validation_window_days": 7
      },
      "conversion_rule": {
        "click_window_hours": 720,
        "attribution_model": "last_click",
        "accepted_types": ["sale"],
        "dedup_strategy": "first"
      }
    }
  ],
  "has_more": false,
  "total": 1
}
```

## Error Codes

| HTTP Status | Error Code | When It Happens |
|-------------|------------|-----------------|
| `400` | `BAD_REQUEST` | The request body is malformed or missing required fields. |
| `401` | `UNAUTHORIZED` | The request is missing an API key or the key cannot be authenticated. |
| `403` | `FORBIDDEN` | The API key is valid but suspended, blocked, or not allowed to access the resource. |
| `429` | `RATE_LIMITED` | The client exceeded the allowed request rate. |
| `500` | `INTERNAL_ERROR` | The server failed to process the request due to an unexpected error. |

## Error Response Example

```json
{
  "error": {
    "code": "BAD_REQUEST",
    "message": "intent.content must contain at least one item"
  }
}
```

## Design Decisions

- **POST with structured body**: The query moved from `GET` with query parameters to `POST` with a JSON body. The request now carries structured context, multimodal intent, and user profile — this complexity is better expressed as a JSON object than as URL parameters.
- **Multimodal intent**: `intent.content[]` uses a typed array that mirrors LLM message formats. This makes it natural for AI agent platforms to construct queries from conversation context.
- **Context separation**: `context` separates platform metadata, session tracking, and user profile into distinct sub-objects. This keeps concerns clear and allows each layer to evolve independently.
- **RECOMMENDED for context sub-fields**: Platform, session, user profile fields, and device info are RECOMMENDED — the fields SHOULD be present for consistent request parsing, but values MAY be empty when data is unavailable.
- **`viewer_id` as pseudonymous**: The protocol does not require real user IDs. A pseudonymous viewer identifier is sufficient for frequency capping and personalization, preserving user privacy.
- **Offset pagination**: The initial design uses offset-based pagination for simplicity. Cursor-based pagination can be reintroduced if performance requirements demand it.
- **Structured filter + semantic intent**: `filter` provides deterministic narrowing (SQL WHERE equivalent) while `intent` provides relevance ranking (search scoring equivalent). This dual-signal design lets agents express both hard business constraints and soft user preferences in a single query.

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1 | 2026-03-20 | Initial draft. |
| 0.1 | 2026-03-22 | Added broader compatibility-oriented filters. |
| 0.1 | 2026-03-23 | Added CTA-oriented action semantics to the response example. |
| 0.1 | 2026-03-24 | Reframed the query result as `offer response { trace_id, offers[] }`, aligned the payload with `offer`, and updated key field names. |
| 0.1 | 2026-03-24 | Added `offer-query` example guidance and clarified the boundary between query request, canonical `offer`, and `offer response`. |
| 0.1 | 2026-03-24 | Updated the response example to the `uuid + offer_info + entity + action + targeting + commission` draft shape. |
| 0.1 | 2026-03-25 | Restructured from GET parameters to POST JSON body. Introduced `context` (platform, session, user_profile), multimodal `intent.content[]`, REQUIRED/RECOMMENDED/OPTIONAL requirement levels (RFC 2119), and offset-based pagination. |
| 0.1 | 2026-03-28 | Added `filter` object for structured query constraints (category_types, commission_models, status, availability, price/commission range, brand, country, tags). Added enum extensibility note. Updated request example with filter fields and response example with commission and conversion_rule fields. |
