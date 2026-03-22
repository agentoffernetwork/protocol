# Click and Conversion Events v0.1

**Version**: 0.1
**Status**: Draft
**Last Updated**: 2026-03-20

## Introduction

This document defines the event payloads used for attribution and revenue tracking in AgentOffer Protocol v0.1. These events are shared between the recommendation layer, the tracking service, and downstream settlement systems.

The canonical Offer Schema does not define standalone click or conversion event objects, but it does define the surrounding settlement context that these events must support, including:

- `commission.payout_level`
- `commission.validation_window_days`
- `commission.conversion_criteria`
- `source.postback_url_template`
- `source.tracking_url_template`

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

### Click Event Example

```json
{
  "event_type": "click",
  "tracking_id": "trk_01_click_abc",
  "offer_id": "ao_01HX2B3C4D5E6F7G8H9J0KABCD",
  "agent_id": "agt_assistant_123",
  "session_id": "sess_chat_456",
  "timestamp": "2026-03-20T10:00:00Z",
  "user_agent": "Mozilla/5.0"
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
  "commission_amount": 24
}
```

## Design Decisions

- Event payloads are intentionally flat in v0.1 so they are easy to emit from multiple systems without schema translation overhead.
- `session_id` is optional because not every surface has stable conversation state, but it is valuable when agents need recommendation traceability.
- `commission_amount` lives on the conversion event so downstream settlement systems can consume a single normalized record.
- `tracking_id` is the shared join key across click and conversion flows in v0.1.
- `offer_id` examples use the same `ao_{ulid}` shape as the reference Offer Schema so event records can be joined to canonical offer documents without translation.

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1 | 2026-03-20 | Initial draft. |
| 0.1 | 2026-03-20 | Clarified mapping to settlement fields from the reference Offer Schema and aligned `offer_id` examples with the canonical ID format. |
