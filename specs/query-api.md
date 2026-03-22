# Offer Query API v0.1

**Version**: 0.1
**Status**: Draft
**Last Updated**: 2026-03-22

## Introduction

This document defines the HTTP query interface that AI agents and SDKs use to discover offers from an AgentOffer-compatible service. The goal of v0.1 is to provide a simple, predictable read API that can be implemented early by platform teams and consumed directly by SDKs.

## Endpoint Overview

- Method: `GET`
- Path: `/v1/offers`
- Purpose: Search and filter available offers for recommendation use cases

## Authentication

Requests must include a bearer token in the `Authorization` header.

```http
Authorization: Bearer {api_key}
```

## Request Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `q` | string | No | — | Free-text keyword search over offer title, description, and tags. |
| `category_primary` | string | No | — | Filter by `category.primary` in the Offer Schema. |
| `brand` | string | No | — | Filter by `attributes.brand` for physical or branded products. |
| `availability` | enum | No | — | Filter by availability state such as `in_stock`, `out_of_stock`, `preorder`, `limited`, or `always_available`. |
| `country` | string | No | — | Filter by country availability using ISO 3166-1 alpha-2 code, evaluated against geo restrictions. |
| `source_type` | string | No | — | Filter by `source.type` such as `impact`, `cj`, `partnerstack`, or `direct`. |
| `sku` | string | No | — | Filter by canonical merchant-facing SKU. |
| `variant_group_id` | string | No | — | Filter by variant family or product group identifier. |
| `min_price` | number | No | — | Minimum offer price amount. |
| `max_price` | number | No | — | Maximum offer price amount. |
| `payout_level` | enum | No | — | Filter by `commission.payout_level`: `item`, `order`, `click`, `lead`, or `install`. |
| `limit` | integer | No | `20` | Number of offers to return. Maximum value is `100`. |
| `cursor` | string | No | — | Opaque cursor used for pagination. |

## Response Structure

| Field | Type | Description |
|------|------|-------------|
| `data` | array | List of Offer objects matching the query. |
| `has_more` | boolean | Indicates whether another page is available. |
| `next_cursor` | string or null | Cursor for the next page. Present when `has_more` is `true`; otherwise omitted or `null`. |

### Successful Response Example

```json
{
  "data": [
    {
      "id": "ao_01HX2B3C4D5E6F7G8H9J0KABCD",
      "sku": "notion-team-monthly-usd",
      "variant_group_id": "notion-team-plan",
      "title": "Notion Team Plan",
      "description": "Notion is an all-in-one workspace that combines notes, documents, knowledge bases, project management, and collaboration tools.",
      "url": "https://www.notion.so/product",
      "price": {
        "amount": 10,
        "currency": "USD"
      },
      "currency": "USD",
      "commission": {
        "type": "percentage",
        "value": 0.3,
        "payout_level": "order"
      },
      "advertiser": {
        "id": "adv_notion_001",
        "name": "Notion Labs, Inc."
      },
      "category": {
        "primary": "saas_tools",
        "secondary": "project_management"
      },
      "attributes": {
        "brand": "Notion"
      },
      "status": "active"
    }
  ],
  "has_more": true,
  "next_cursor": "eyJvZmZzZXQiOjIwfQ=="
}
```

## Pagination

- Cursor pagination is used instead of page-number pagination.
- Clients should treat `cursor` and `next_cursor` as opaque values.
- When `has_more` is `false`, `next_cursor` should be omitted or returned as `null`.

## Error Codes

| HTTP Status | Error Code | When It Happens |
|-------------|------------|-----------------|
| `400` | `BAD_REQUEST` | One or more query parameters are malformed or invalid. |
| `401` | `UNAUTHORIZED` | The request is missing an API key or the key cannot be authenticated. |
| `403` | `FORBIDDEN` | The API key is valid but suspended, blocked, or not allowed to access the resource. |
| `429` | `RATE_LIMITED` | The client exceeded the allowed request rate. |
| `500` | `INTERNAL_ERROR` | The server failed to process the request due to an unexpected error. |

### Error Response Example

```json
{
  "error": {
    "code": "BAD_REQUEST",
    "message": "limit must be an integer between 1 and 100"
  }
}
```

## Examples

### Example Request

```http
GET /v1/offers?q=notion&category_primary=saas_tools&brand=Notion&availability=always_available&payout_level=order&limit=20 HTTP/1.1
Host: api.agentoffernetwork.com
Authorization: Bearer sk_live_example
Accept: application/json
```

### Example Forbidden Response

```json
{
  "error": {
    "code": "FORBIDDEN",
    "message": "This API key is not permitted to access offers."
  }
}
```

## Design Decisions

- `GET /v1/offers` is intentionally narrow so SDK teams can build against a stable discovery endpoint before the broader platform matures.
- Cursor pagination is preferred over total-count pagination because it scales better and avoids expensive count queries.
- Authorization is header-based to keep the API compatible with typical server-side and agent-side HTTP clients.
- The response returns full Offer objects that conform to the Offer Schema, including nested `commission`, `advertiser`, and `category` objects.
- The request surface includes compatibility-oriented filters such as `brand`, `availability`, `country`, `sku`, and `variant_group_id` because implementers often need to bridge AgentOffer offers with merchant feeds, structured-data adapters, and catalog tooling.

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1 | 2026-03-20 | Initial draft. |
| 0.1 | 2026-03-22 | Added compatibility-oriented filters for brand, availability, geography, source type, SKU, and variant grouping. |
