# Offer Schema v0.1

**Version**: 0.1
**Status**: Draft
**Last Updated**: 2026-03-24

## Introduction

This document defines the protocol-facing `offer` object for AgentOffer Protocol v0.1.

The object is centered on:

- identity
- content
- entity
- action

The protocol now also distinguishes between:

- `offer`: the canonical unit returned to clients
- `offer response`: the envelope used by query APIs

## Offer

### Required Shape

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `id` | string | Yes | Stable offer identifier. UUIDv7 is recommended. |
| `title` | string | Yes | Human-readable offer title. |
| `description` | string | Yes | Primary offer description for agent understanding and display. |
| `offer_type` | string | No | Offer classification such as `saas`, `physical_product`, `service`, or `promotion`. |
| `source_offer_id` | string | No | Upstream source-side offer or inventory identifier. |
| `expire_at` | string | No | Expiration timestamp for the offer. |
| `entity` | object | Yes | Business entity the offer belongs to. |
| `action` | object | Yes | Primary user action exposed by the offer. |

## Object Model

### Identity and Core Content

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `id` | string | Yes | Stable offer identifier in the protocol response. UUIDv7 is recommended. |
| `title` | string | Yes | Display-ready title. |
| `description` | string | Yes | Core semantic description used by agents and clients. |
| `offer_type` | string | No | Optional offer type used for coarse classification. |
| `source_offer_id` | string | No | Upstream inventory or source-side offer ID. |
| `expire_at` | string | No | RFC 3339 timestamp indicating when the offer should no longer be surfaced. |

Design notes:

- `id` is the protocol-facing offer identity for query and tracking use cases.
- `source_offer_id` is intentionally lightweight; richer upstream reconciliation can be defined in later layers if needed.
- `expire_at` is simpler than a broader lifecycle model and is enough for early filtering and UI suppression.

### Entity

`entity` is the normalized business subject of the offer. It replaces the more specific `advertiser` wording in the protocol-facing object so the model can work for merchants, brands, service providers, platforms, and other supply-side entities.

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `entity.id` | string | Yes | Stable entity identifier. |
| `entity.name` | string | Yes | Display name of the entity. |
| `entity.type` | string | No | Optional entity classification such as `merchant`, `brand`, `creator`, `seller`, or `service_provider`. |
| `entity.description` | string | No | Optional short description of the entity. |

Design notes:

- `entity` is intentionally neutral and does not assume every offer comes from a classic affiliate advertiser.
- Future protocol layers may still define richer source, settlement, or advertiser-specific metadata outside `offer`.

### Action

`action` describes what the end user can actually do with the offer. This is now a first-class protocol concept rather than a rendering hint.

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `action.type` | string | Yes | User-facing action type such as `view_offer`, `buy`, or `start_trial`. |
| `action.name` | string | No | Short user-facing action name. |
| `action.description` | string | No | Optional explanation of the action intent or destination. |
| `action.payload` | object | Yes | Action-specific payload. Structure depends on `action.type`. |

#### `action.type`

The current draft recommends user-facing action values such as:

- `view_offer`
- `buy`
- `start_trial`
- `subscribe`
- `install`
- `book`
- `apply`

Delivery mechanics such as web redirects or app deep links should be expressed through `action.payload`, not by overloading `action.type`.

#### `action.payload`

For a web-based action flow, the payload should support:

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `action.payload.target` | string | Yes | Action target. For web delivery this should be a destination URL. |
| `action.payload.requires_auth` | boolean | No | Whether the user is likely required to sign in first. |
| `action.payload.platform` | string | No | Target platform such as `web`, `ios`, `android`, or `desktop`. |
| `action.payload.delivery_method` | string | No | Delivery mechanism such as `web_redirect`, `app_deep_link`, or `api_trigger`. |

Design notes:

- `action` answers "what the client can ask the user to do now".
- `action.payload` keeps the base schema extensible without flattening action-specific fields into top-level offer properties.
- `action.event` is intentionally not included in v0.1. It can be reconsidered in a future revision if downstream action telemetry needs protocol-level standardization.

## Validation Rules

- `id` must be a UUIDv7 string.
- `title` must be a non-empty string suitable for direct display.
- `description` must be a non-empty string suitable for semantic retrieval and end-user display.
- `expire_at`, when present, must be a valid RFC 3339 timestamp.
- `entity.id` and `entity.name` are both required whenever `entity` is present.
- `action.type` and `action.payload` are both required whenever `action` is present.
- `action.payload.target` must be a valid URI when the action is delivered through a web flow.

## Example

```json
{
  "id": "0195ef94-f17d-7a4f-b6e0-2c52bb49e13f",
  "title": "Notion Team Plan",
  "description": "Collaborative workspace for docs, wikis, and project planning.",
  "offer_type": "saas",
  "source_offer_id": "adx_inventory_298174",
  "expire_at": "2026-12-31T23:59:59Z",
  "entity": {
    "id": "ent_notion",
    "name": "Notion",
    "type": "service_provider",
    "description": "Productivity software provider."
  },
  "action": {
    "type": "start_trial",
    "name": "Start free trial",
    "description": "Redirect the user to the Notion signup flow.",
    "payload": {
      "target": "https://www.notion.so/signup",
      "requires_auth": false,
      "platform": "web",
      "delivery_method": "web_redirect"
    }
  }
}
```

## Migration Notes

This revision is a protocol simplification pass. Compared with the earlier richer object model:

- `id` is the protocol-facing offer identifier
- `entity` replaces the more specific `advertiser` wording in the client-facing shape
- `action` replaces the earlier `primary_action` plus `action_labels` approach
- `action.payload.target` replaces a web-only `url` field so action delivery can expand beyond redirect-only flows
- many settlement, reconciliation, ranking, and catalog fields are intentionally not included in `offer`

This does not mean those richer fields are permanently removed from the broader ecosystem. It means the protocol-facing object is being narrowed so teams can first agree on the minimum interoperable offer unit.

Machine-readable artifacts are intentionally not updated by this document alone. They should be aligned only after this protocol draft is confirmed.

## Design Decisions

- The protocol should first standardize the minimum client-facing offer unit before expanding into settlement-heavy or ranking-heavy shapes.
- `entity` is preferred over `advertiser` because the supply-side actor is not always a classic advertiser.
- `action` is preferred over a lighter CTA hint because agents and clients need a structured way to know what to do with an offer.
- The human-readable protocol should state required versus optional fields directly; deferred concepts such as action events should stay out of the core v0.1 shape until they are truly needed.

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1 | 2026-03-20 | Initial draft. |
| 0.1 | 2026-03-22 | Added compatibility-oriented fields and industry mapping guidance. |
| 0.1 | 2026-03-23 | Added `primary_action` and `action_labels` for CTA semantics. |
| 0.1 | 2026-03-24 | Reframed the protocol around `offer`, `entity`, and `action`, simplified field guidance to required versus optional, and aligned key names around `id`, `offer_type`, `source_offer_id`, and user-facing action semantics. |
