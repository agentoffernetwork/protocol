# Click and Conversion Events v0.1

**Version**: 0.1
**Status**: Draft
**Last Updated**: 2026-03-28

## Introduction

This document defines the event payloads used for attribution and revenue tracking in AgentOffer Protocol. These events are shared between the recommendation layer, the tracking service, and downstream settlement systems.

The Offer Schema defines the surrounding settlement context that these events operate within, including:

- `commission` object (model, amount, currency, rate, tier, cap, payout_delay_days, validation_window_days)
- `conversion_rule` object (click_window_hours, view_window_hours, attribution_model, accepted_types, dedup_strategy, minimum_amount)
- `source.postback_url_template`
- `source.tracking_url_template`

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

AgentOffer v0.1 defines two core event types:

- `click`: emitted when a user follows a tracked recommendation link
- `conversion`: emitted when a tracked user completes a billable action

## Click Event

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `event_type` | string | Yes | Fixed value: `"click"`. |
| `tracking_id` | string | Yes | Unique identifier of the tracking link instance. |
| `offer_id` | string | Yes | Identifier of the recommended offer. |
| `agent_id` | string | Yes | Identifier of the agent that issued the recommendation. |
| `session_id` | string | No | Optional session or conversation correlation identifier. |
| `timestamp` | string | Yes | ISO 8601 event timestamp. |
| `user_agent` | string | No | Optional raw or normalized user-agent string from the click request. |
| `sub_id_1` | string | No | Custom tracking parameter 1 (typically recommendation scenario). |
| `sub_id_2` | string | No | Custom tracking parameter 2 (typically user cohort). |
| `sub_id_3` | string | No | Custom tracking parameter 3. |
| `sub_id_4` | string | No | Custom tracking parameter 4. |
| `sub_id_5` | string | No | Custom tracking parameter 5. |

### Click Event Example

```json
{
  "event_type": "click",
  "tracking_id": "trk_01_click_abc",
  "offer_id": "ao_01HX2B3C4D5E6F7G8H9J0KABCD",
  "agent_id": "agt_assistant_123",
  "session_id": "sess_chat_456",
  "timestamp": "2026-03-20T10:00:00Z",
  "user_agent": "Mozilla/5.0",
  "sub_id_1": "homepage_widget",
  "sub_id_2": "cohort_a"
}
```

## Conversion Event

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `event_type` | string | Yes | Fixed value: `"conversion"`. |
| `tracking_id` | string | Yes | Tracking identifier previously associated with a click event. |
| `offer_id` | string | Yes | Identifier of the converted offer. |
| `agent_id` | string | Yes | Identifier of the agent that drove the conversion. |
| `order_id` | string | Yes | Advertiser-side order or action identifier. |
| `amount` | number | Yes | Gross converted amount. |
| `currency` | string | Yes | ISO 4217 currency code for the converted amount. |
| `timestamp` | string | Yes | ISO 8601 event timestamp. |
| `commission_amount` | number | Yes | Commission amount computed by the platform. |
| `conversion_type` | string | Yes | Conversion classification: `sale`, `lead`, `install`, `subscription`, `trial`, or `custom`. |
| `sub_id_1` through `sub_id_5` | string | No | Inherited from the associated click event. Absent (not null) when not set on the original click. |

### Conversion Event Example

```json
{
  "event_type": "conversion",
  "tracking_id": "trk_01_click_abc",
  "offer_id": "ao_01HX2B3C4D5E6F7G8H9J0KABCD",
  "agent_id": "agt_assistant_123",
  "order_id": "ord_987654",
  "amount": 120,
  "currency": "USD",
  "timestamp": "2026-03-21T03:10:00Z",
  "commission_amount": 24,
  "conversion_type": "sale",
  "sub_id_1": "homepage_widget",
  "sub_id_2": "cohort_a"
}
```

## Postback Notification

Postback is the mechanism for notifying Agent developers of conversion events in real time. When a conversion is attributed to an Agent's click, the platform sends an HTTP request to the Agent developer's registered endpoint with the full conversion details.

### Registration

Agent developers provide a `postback_url_template` during onboarding. This template is stored in the Offer Schema's `source.postback_url_template` field and supports variable substitution for dynamic routing.

### Request Method

The protocol defines **POST** as the sole request method. The request body is a JSON payload containing the full conversion event details. GET compatibility is a platform adapter concern and is not part of this specification.

### Request Body

| Field | Type | Description |
|------|------|-------------|
| `event_id` | string | Unique identifier for this notification, used for idempotency. |
| `event_type` | string | Fixed value: `"conversion"`. |
| `tracking_id` | string | Tracking identifier from the original click event. |
| `offer_id` | string | Identifier of the converted offer. |
| `agent_id` | string | Identifier of the agent that drove the conversion. |
| `conversion_type` | string | Conversion classification: `sale`, `lead`, `install`, `subscription`, `trial`, or `custom`. |
| `amount` | number | Gross converted amount. |
| `currency` | string | ISO 4217 currency code. |
| `commission_amount` | number | Commission amount computed by the platform. |
| `sub_id_1` | string | Custom tracking parameter 1 (absent when not set on the original click). |
| `sub_id_2` | string | Custom tracking parameter 2 (absent when not set on the original click). |
| `sub_id_3` | string | Custom tracking parameter 3 (absent when not set on the original click). |
| `sub_id_4` | string | Custom tracking parameter 4 (absent when not set on the original click). |
| `sub_id_5` | string | Custom tracking parameter 5 (absent when not set on the original click). |
| `timestamp` | string | ISO 8601 event timestamp. |

### URL Template Variables

The `postback_url_template` supports the following substitution variables:

| Variable | Description |
|----------|-------------|
| `{tracking_id}` | Tracking identifier from the original click. |
| `{offer_id}` | Identifier of the converted offer. |
| `{conversion_type}` | Conversion classification value. |
| `{amount}` | Gross converted amount. |
| `{commission_amount}` | Computed commission amount. |
| `{currency}` | ISO 4217 currency code. |
| `{sub_id_1}` | Custom tracking parameter 1 (empty string when not set). |
| `{sub_id_2}` | Custom tracking parameter 2 (empty string when not set). |
| `{sub_id_3}` | Custom tracking parameter 3 (empty string when not set). |
| `{sub_id_4}` | Custom tracking parameter 4 (empty string when not set). |
| `{sub_id_5}` | Custom tracking parameter 5 (empty string when not set). |
| `{timestamp}` | ISO 8601 event timestamp. |

### Signature Verification

Each postback request includes an `X-AON-Signature` HTTP header for authenticity verification. The signature is computed as:

```
HMAC-SHA256(secret, request_body_json)
```

Where `secret` is the shared secret established during Agent developer onboarding and `request_body_json` is the raw JSON string of the request body. Agent developers MUST verify the signature before processing the postback payload.

### Retry Policy

If the Agent developer's endpoint does not return an HTTP 2xx response within **10 seconds**, the platform retries delivery using the following schedule:

| Attempt | Delay after previous attempt |
|---------|------------------------------|
| 1 | Immediate (first delivery) |
| 2 | 1 minute |
| 3 | 5 minutes |
| 4 | 30 minutes |
| 5 | 2 hours |

After 5 failed attempts the postback is marked as permanently failed. The maximum total retry window is approximately 24 hours from the initial attempt.

### Idempotency

Each postback contains a unique `event_id`. The platform MAY deliver the same postback more than once due to retry logic or internal recovery. Agent developers MUST handle duplicate deliveries gracefully by deduplicating on `event_id`.

### Postback Example

```json
{
  "event_id": "evt_pb_01HX9A2B3C4D5E6F",
  "event_type": "conversion",
  "tracking_id": "trk_01_click_abc",
  "offer_id": "ao_01HX2B3C4D5E6F7G8H9J0KABCD",
  "agent_id": "agt_assistant_123",
  "conversion_type": "sale",
  "amount": 120,
  "currency": "USD",
  "commission_amount": 24,
  "sub_id_1": "homepage_widget",
  "sub_id_2": "cohort_a",
  "timestamp": "2026-03-21T03:10:00Z"
}
```

> **Platform implementation note**: The webhook infrastructure, retry queues, dead-letter handling, and subscription management are platform concerns delivered separately in SVC-CORE. This specification defines only the protocol-level contract between the platform and Agent developer endpoints.

## Offer Lifecycle Events (P2)

The following event types are reserved for a future revision of the protocol. They enable Agent developers to react to Offer state changes in real time, reducing the risk of recommending expired or paused Offers.

### offer_updated

Emitted when one or more Offer fields change (e.g., price, commission, description).

| Field | Type | Description |
|------|------|-------------|
| `event_type` | string | Fixed value: `"offer_updated"`. |
| `offer_id` | string | Identifier of the updated offer. |
| `updated_fields` | array | List of field paths that changed (e.g., `["commission.amount", "offer_info.description"]`). |
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

> **Note**: These events share the same webhook delivery and signature mechanism defined in the Postback Notification section. Implementation is deferred to a future revision of the protocol.

## Design Decisions

- Event payloads are intentionally flat in v0.1 so they are easy to emit from multiple systems without schema translation overhead.
- `session_id` is optional because not every surface has stable conversation state, but it is valuable when agents need recommendation traceability.
- `commission_amount` lives on the conversion event so downstream settlement systems can consume a single normalized record.
- `tracking_id` is the shared join key across click and conversion flows in v0.1.
- `offer_id` examples use the same `ao_{ulid}` shape as the reference Offer Schema so event records can be joined to canonical offer documents without translation.
- `sub_id_1` through `sub_id_5` follow the numbered naming convention used by industry affiliate networks (e.g., Impact, CJ Affiliate) to maximize data import compatibility. Five sub-IDs is the protocol ceiling; extending beyond five requires a protocol revision.
- `conversion_type` is a separate field from `event_type` for backward compatibility. `event_type` remains `"conversion"` for all conversion events, while `conversion_type` provides the fine-grained classification (sale, lead, install, etc.). This avoids breaking consumers that switch on `event_type`.
- Postback uses POST with a JSON body exclusively. GET-based postback URLs are a legacy pattern; the protocol standardizes on POST for richer payloads and consistent signature verification. Platforms that need to support GET-based endpoints MAY implement an adapter layer outside the protocol.
- Enum extensibility: all enumeration values in this specification (`conversion_type`, lifecycle `event_type` values) follow an open-ended design. Consumers SHOULD gracefully handle unknown values (ignore or pass-through) rather than returning errors. The protocol reserves the right to add new enumeration values in future revisions without treating such additions as breaking changes.

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1 | 2026-03-20 | Initial draft. |
| 0.1 | 2026-03-20 | Clarified mapping to settlement fields from the reference Offer Schema and aligned `offer_id` examples with the canonical ID format. |
| 0.1 | 2026-03-28 | PROTO-F004 revision: added Conformance Keywords section; added `sub_id_1`–`sub_id_5` to Click Event; added `conversion_type` and `sub_id_1`–`sub_id_5` to Conversion Event; added Postback Notification chapter; added Offer Lifecycle Events (P2) chapter; updated Design Decisions with sub-ID naming rationale, conversion_type separation rationale, POST-only postback design, and enum extensibility policy; removed placeholder status and forward-looking language. |
