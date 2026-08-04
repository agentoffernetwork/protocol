# Click and Conversion Events v0.2

**Version**: 0.2
**Status**: Stable source-follow; runtime support not available
**Last Updated**: 2026-07-27

## Introduction

This document explains the event payloads used for attribution and revenue
tracking in AgentOffer Protocol v0.2. The canonical AON → Agent conversion
payload is the closed Agent Postback schema referenced below; this document
does not define a parallel payload.

The Offer Schema defines the surrounding attribution context, including
`goals[]`, `conversion_rule`, and `source.tracking_url_template`. Public v0.2
Offer payloads do not contain top-level `bid` or
`conversion_rule.accepted_types`.

### Conformance Keywords

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHOULD", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

### Field Requirement Levels

The protocol classifies fields into three requirement levels to balance standardization with flexibility:

| Level | Label | Meaning |
|-------|-------|---------|
| **REQUIRED** | Required | Field MUST be present with a valid, non-empty value. |
| **RECOMMENDED** | Recommended | Field SHOULD be present and MUST follow the standard structure when present, but the value MAY be empty or null. |
| **OPTIONAL** | Optional | Field MAY be omitted entirely. When included, it SHOULD follow the specified format. |

## Overview

AgentOffer v0.2 defines two core event types:

- `click`: emitted when a user follows a tracked recommendation link
- `conversion`: emitted when a tracked user completes a billable action

## Click Event

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `event_type` | string | Yes | Fixed value: `"click"`. |
| `aon_tracking_id` | string | Yes | Unique identifier of the tracking link instance. |
| `click_id` | string | No | AON-generated per-click event identifier. Exposed to landing URLs through `{CLICK_ID}` or the `aon_click_id` fallback query parameter. |
| `offer_id` | string | Yes | Identifier of the recommended offer. |
| `agent_id` | string | Yes | Identifier of the agent that issued the recommendation. |
| `session_id` | string | No | Optional session or conversation correlation identifier. |
| `timestamp` | string | Yes | ISO 8601 event timestamp. |
| `user_agent` | string | No | Optional raw or normalized user-agent string from the click request. |
| `sub_id` | string | No | First custom tracking parameter (typically recommendation scenario). |
| `sub_id_2` | string | No | Custom tracking parameter 2 (typically user cohort). |
| `sub_id_3` | string | No | Custom tracking parameter 3. |
| `sub_id_4` | string | No | Custom tracking parameter 4. |
| `sub_id_5` | string | No | Custom tracking parameter 5. |

### Click Event Example

```json
{
  "event_type": "click",
  "aon_tracking_id": "trk_01_click_abc",
  "click_id": "aci_019e30f2-1b35-7b58-b47d-2cc086ace710",
  "offer_id": "ao_01HX2B3C4D5E6F7G8H9J0KABCD",
  "agent_id": "agt_assistant_123",
  "session_id": "sess_chat_456",
  "timestamp": "2026-03-20T10:00:00Z",
  "user_agent": "Mozilla/5.0",
  "sub_id": "homepage_widget",
  "sub_id_2": "cohort_a"
}
```

## Conversion Event

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `event_id` | string | Yes | Durable conversion event identifier and receiver idempotency key component. |
| `event_type` | string | Yes | Fixed value: `"conversion"`. |
| `event_name` | string | Yes | Exact declared `Offer.goals[].event` identity. |
| `aon_tracking_id` | string | Yes | Dispatch-level tracking identifier previously associated with a click event. Kept as the legacy fallback attribution key. |
| `offer_id` | string | Yes | Identifier of the converted offer. |
| `agent_id` | string | Yes | Identifier of the agent that drove the conversion. |
| `amount` | number | Yes | Gross converted amount. |
| `currency` | string | Yes | ISO 4217 currency code for the converted amount. |
| `timestamp` | string | Yes | ISO 8601 event timestamp. |
| `sub_id`, `sub_id_2` through `sub_id_5` | string | No | Developer attribution slots inherited from the associated click event. Absent (not null) when not set on the original click. |

Canonical conversion identity: `event_name`.
Forbidden legacy conversion fields: `bid_amount`, `conversion_type`.

### Conversion Event Example

```json
{
  "event_id": "evt_01J0AONCONVERSION000001",
  "event_type": "conversion",
  "aon_tracking_id": "trk_01_click_abc",
  "offer_id": "ao_01HX2B3C4D5E6F7G8H9J0KABCD",
  "agent_id": "agt_assistant_123",
  "event_name": "subscription",
  "amount": 120,
  "currency": "USD",
  "sub_id": "homepage_widget",
  "sub_id_2": "cohort_a",
  "timestamp": "2026-03-21T03:10:00Z"
}
```

## Postback Notification

Postback is the mechanism for notifying Agent developers of attributed
conversion events in real time. Partner conversion intake remains the
simplified `GET|POST /v1/postback/{partner_id}` callback route. The dedicated
[`postback.md`](./postback.md) document defines the active intake boundary and
the WS-22-S3 AON → Agent webhook target contract.

> **Source of truth**: `postback.md`,
> `postback-agent-payload-v0.2.json`, and `goal-event-name-v0.2.json` own the
> conversion fields, webhook delivery, signature, retry, and receiver
> idempotency rules. This event specification is an explanatory projection.

## Offer Lifecycle Events (P2)

The following event types are reserved for a future revision of the protocol. They enable Agent developers to react to Offer state changes in real time, reducing the risk of recommending expired or paused Offers.

### offer_updated

Emitted when one or more Offer fields change (for example price, title, or description).

| Field | Type | Description |
|------|------|-------------|
| `event_type` | string | Fixed value: `"offer_updated"`. |
| `offer_id` | string | Identifier of the updated offer. |
| `updated_fields` | array | List of field paths that changed (for example `["offer_info.commercial.price.amount", "offer_info.description"]`). |
| `timestamp` | string | ISO 8601 event timestamp. |

### offer_expired

Emitted when an Offer reaches its `expire_at` date or is manually expired by the advertiser.

| Field | Type | Description |
|------|------|-------------|
| `event_type` | string | Fixed value: `"offer_expired"`. |
| `offer_id` | string | Identifier of the expired offer. |
| `reason` | string | Expiration reason: `"scheduled"` (reached expire_at) or `"manual"` (advertiser action). |
| `timestamp` | string | ISO 8601 event timestamp. |

### offer_paused

Emitted when an Offer is temporarily paused by the advertiser.

| Field | Type | Description |
|------|------|-------------|
| `event_type` | string | Fixed value: `"offer_paused"`. |
| `offer_id` | string | Identifier of the paused offer. |
| `reason` | string | Pause reason (e.g., `"budget_exhausted"`, `"inventory_low"`, `"manual"`). |
| `expected_resume_at` | string | ISO 8601 timestamp of expected resumption, or empty string if unknown. |
| `timestamp` | string | ISO 8601 event timestamp. |

> **Note**: These events share the same webhook delivery and signature mechanism defined in [`postback.md`](./postback.md). Implementation is deferred to a future revision of the protocol.

## Design Decisions

- Event payloads are intentionally flat in v0.2 so they are easy to emit from multiple systems without schema translation overhead.
- `session_id` is optional because not every surface has stable conversation state, but it is valuable when agents need recommendation traceability.
- Goal identity is `event_name`, copied exactly from the matched
  `Offer.goals[].event`; legacy `conversion_type` and `bid_amount` are not part
  of the public v0.2 conversion payload.
- `aon_tracking_id` is the dispatch-level join key across click and conversion flows. Click-level identifiers may exist in tracking internals but are not fields in the closed AON → Agent v0.2 Postback payload.
- `offer_id` examples use the same `ao_{ulid}` shape as the reference Offer Schema so event records can be joined to canonical offer documents without translation.
- `sub_id` plus `sub_id_2` through `sub_id_5` provide five fixed developer
  attribution slots. The unnumbered first slot keeps the common single-field
  case unambiguous; extending beyond five requires a protocol revision.
- `event_type` remains `"conversion"` for all conversion events, while
  `event_name` provides the declared Goal identity.
- Postback uses POST with a JSON body exclusively. GET-based postback URLs are a legacy pattern; the protocol standardizes on POST for richer payloads and consistent signature verification. Platforms that need to support GET-based endpoints MAY implement an adapter layer outside the protocol.
- Lifecycle `event_type` values are future-reserved. The closed conversion
  Postback schema must not accept undeclared fields or undeclared Goal names.

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1 | 2026-03-20 | Initial draft. |
| 0.1 | 2026-03-20 | Clarified mapping to settlement fields from the reference Offer Schema and aligned `offer_id` examples with the canonical ID format. |
| 0.1 | 2026-03-28 | PROTO-F004 revision: added Conformance Keywords, conversion classification, Postback Notification, lifecycle events, and field-level event documentation. |
| 0.1 | 2026-04-17 | PROTO-F010: Consolidated Postback Notification section into dedicated `postback.md`; replaced detailed content with overview and pointer. Lifecycle Events note updated to reference `postback.md`. |
| 0.1 | 2026-06-30 | Added click-level attribution fields: `click_id` as the canonical per-click event ID and `aon_click_id` as the landing URL fallback field. Clarified that `aon_tracking_id` remains the dispatch-level fallback key. |
| 0.2 | 2026-07-27 | Aligned the conversion projection with canonical Postback v0.2: required `event_name`, removed `bid_amount`/`conversion_type`, and made `postback.md` plus JSON Schema authoritative. |
