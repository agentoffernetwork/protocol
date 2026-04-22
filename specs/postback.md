# Postback Specification v0.1

**Version**: 0.1
**Status**: Draft
**Last Updated**: 2026-04-17

## 1. Overview

This document defines the **Postback** protocol for the AgentOffer Network
(AON) -- the asynchronous S2S callback mechanism used for conversion
attribution, settlement, and refund/adjustment reporting.

The specification covers two directions in a single document:

- **Part A (AON -> Agent)**: AON notifies Agent developers of attributed
  conversion events via webhook. The Agent developer registers a
  `postback_url_template` (see [offer-schema.md `source.postback_url_template`](./offer-schema.md))
  and receives POST requests with the full conversion payload.

- **Part B (Partner -> AON)**: Partners report conversions, refunds, and
  adjustments back to AON by calling `POST {aon_base_url}/v1/postback`
  with a signed JSON payload.

### Relationship to Other Specifications

| Spec | Relationship |
|------|-------------|
| [`offer-provider-api.md`](./offer-provider-api.md) | F009 -- Partner-side synchronous query API. Postback shares the same HMAC-SHA256 signing mechanics, header conventions, and error envelope. |
| [`events.md`](./events.md) | Defines click and conversion event payloads. Part A Postback delivers conversion events to Agent endpoints. The original Postback section in `events.md` has been consolidated into this document. |
| [`offer-schema.md`](./offer-schema.md) | Defines `source.postback_url_template` -- the URL template that Part A uses for Agent notification. |

## 2. Conformance

### 2.1 Conformance Keywords

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHOULD",
"RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

### 2.2 Agent Conformance Levels (Part A Receiver)

| Level | Label | Obligation | Requirements |
|-------|-------|-----------|-------------|
| **L1** | Baseline | MUST | Expose an HTTPS endpoint accepting POST + JSON; verify HMAC signature (S3); check timestamp within +/-5 min; deduplicate on `event_id`; return 2xx on success |
| **L2** | Recommended | SHOULD | Verify nonce uniqueness (5-min window); log `event_id` for audit; handle all `conversion_type` enum values gracefully |
| **L3** | Enhanced | MAY | Expose `/health` for AON liveness probes; echo structured error bodies on verification failure; support idempotent reprocessing beyond 5-min dedup window |

### 2.3 Partner Conformance Levels (Part B Sender)

| Level | Label | Obligation | Requirements |
|-------|-------|-----------|-------------|
| **L1** | Baseline | MUST | Sign every request per S3; generate unique `event_id` per business event (`evt_pb_{ulid}`); reuse same `event_id` on retry; send REQUIRED fields for the chosen `event_type`; retry at least once on non-2xx |
| **L2** | Recommended | SHOULD | Retry >= 3 times with exponential backoff; send `X-AON-Attempt-Count` header; handle `extra.deduplicated` in responses; send `refund_reason` / `adjustment_reason` when applicable |
| **L3** | Enhanced | MAY | Send all 5 `sub_id_*` fields; use `coupon_code` for coupon-based attribution; implement circuit-breaker on repeated 5xx from AON |

## 3. Common Mechanics

Both Part A and Part B share the following authentication and integrity
mechanisms. These are identical to the signing rules defined in
[`offer-provider-api.md` S4](./offer-provider-api.md#4-authentication).

> Shares signature mechanics with [offer-provider-api.md S4](./offer-provider-api.md#4-authentication).

### 3.1 HMAC-SHA256 Signature

The sender authenticates each request via HMAC-SHA256 over a canonical
signing string. The sender holds a `key_id` and a matching `secret`:

- **Part A**: AON signs with the KeyId/secret assigned to each Agent
  developer during onboarding.
- **Part B**: Partner signs with their own `appkey`/`secret` issued during
  Partner onboarding.

### 3.2 Required Headers

| Header | Value | Obligation |
|--------|-------|:----------:|
| `X-AON-Key` | Sender's `key_id` or `appkey` | MUST |
| `X-AON-Timestamp` | Unix epoch seconds (integer, ASCII decimal) | MUST |
| `X-AON-Nonce` | Unique-per-request random string; UUIDv4 RECOMMENDED | MUST |
| `X-AON-Signature` | Lowercase hex-encoded HMAC-SHA256 (see S3.3) | MUST |
| `X-AON-Attempt-Count` | Integer >= 1, retry attempt number (Part B only) | OPTIONAL |

### 3.3 Signing String

```
METHOD        + "\n"
PATH          + "\n"
CANONICAL_BODY + "\n"
TIMESTAMP     + "\n"
NONCE
```

- `METHOD` -- uppercase ASCII (`POST`).
- `PATH` -- request path only, excluding host and query string.
  - Part A: the resolved Agent endpoint path (e.g. `/webhook/aon/postback`).
  - Part B: `/v1/postback`.
- `CANONICAL_BODY` -- the exact UTF-8 body bytes transmitted on the wire.
  The sender MUST NOT re-serialize, re-order keys, or normalize whitespace
  after signing. The verifier MUST use the received raw body for
  verification.
- `TIMESTAMP` -- identical ASCII decimal string sent in `X-AON-Timestamp`.
- `NONCE` -- identical string sent in `X-AON-Nonce`.

No trailing newline. Components are joined by a single LF (`U+000A`).

Compute the signature as:

```
signature_bytes = HMAC_SHA256(secret, signing_string)
X-AON-Signature = hex_lowercase(signature_bytes)
```

The verifier MUST compare signatures in **constant time** to prevent
timing side-channels.

### 3.4 Timestamp Skew (MUST)

The verifier MUST reject the request when
`|server_now - X-AON-Timestamp| > 300` seconds, returning `401
UNAUTHORIZED` with message `"timestamp outside allowed skew"`. This check
MUST happen **before** signature verification.

### 3.5 Nonce Anti-Replay

For Part B (AON as verifier), nonce enforcement is MUST (AON controls its
own infrastructure). For Part A (Agent as verifier), nonce enforcement is
RECOMMENDED -- Agent developers in stateless environments MAY skip nonce
enforcement but MUST still enforce timestamp skew.

The verifier SHOULD maintain a short-TTL set of `(key_id, nonce)` pairs
for the last 5 minutes and reject duplicates with `401 UNAUTHORIZED` and
message `"nonce already used"`.

### 3.6 Error Envelope (ApiResponse)

All error responses follow the AON `ApiResponse` contract:

```json
{
  "code": "UNAUTHORIZED",
  "message": "invalid signature",
  "data": {},
  "extra": {}
}
```

This is consistent with `CLAUDE.md` S"API Response Contract" and
[`offer-provider-api.md` S9](./offer-provider-api.md#9-error-codes).

### 3.7 Event ID

Every postback event carries a unique `event_id` in format
`evt_pb_{ulid}` (regex: `^evt_pb_[0-9A-HJKMNP-TV-Z]{26}$`).

The `event_id` serves as the **business-level idempotency key**, distinct
from the HTTP-level `X-AON-Nonce` which provides replay protection:

| Purpose | Field | Scope | Regenerated on retry? |
|---------|-------|-------|----------------------|
| Business idempotency | `event_id` | Payload body | No -- same across all retries |
| HTTP replay protection | `X-AON-Nonce` | Header | Yes -- fresh per attempt |

## 4. Part A -- AON -> Agent Postback

### 4.1 Registration

Agent developers provide a `postback_url_template` during onboarding. This
template is stored in the Offer Schema's `source.postback_url_template`
field (see [`offer-schema.md`](./offer-schema.md)) and supports variable
substitution for dynamic routing.

### 4.2 Endpoint

The Agent developer owns the endpoint. AON resolves the
`postback_url_template` by substituting variables (see S9) and sends a
POST request to the resulting URL.

### 4.3 Request Method & Headers

- **Method**: `POST`
- **Content-Type**: `application/json`
- **Authentication headers**: `X-AON-Key`, `X-AON-Timestamp`, `X-AON-Nonce`,
  `X-AON-Signature` per S3.

### 4.4 Request Payload

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `event_id` | string | REQUIRED | Unique identifier (`evt_pb_{ulid}`). Idempotency key; all retries share this value. |
| `event_type` | string | REQUIRED | Fixed value: `"conversion"`. |
| `tracking_id` | string | REQUIRED | Tracking identifier from the original click event. |
| `offer_id` | string | REQUIRED | Identifier of the converted offer. |
| `agent_id` | string | REQUIRED | Identifier of the agent that drove the conversion. |
| `conversion_type` | string | REQUIRED | Conversion classification: `sale`, `lead`, `install`, `subscription`, `trial`, or `custom`. |
| `amount` | number | REQUIRED | Gross converted amount. |
| `currency` | string | REQUIRED | ISO 4217 currency code. |
| `bid_amount` | number | REQUIRED | Bid amount computed by the platform. |
| `sub_id_1` | string | OPTIONAL | Custom tracking parameter 1. Inherited from click; absent (not null) when not set. |
| `sub_id_2` | string | OPTIONAL | Custom tracking parameter 2. |
| `sub_id_3` | string | OPTIONAL | Custom tracking parameter 3. |
| `sub_id_4` | string | OPTIONAL | Custom tracking parameter 4. |
| `sub_id_5` | string | OPTIONAL | Custom tracking parameter 5. |
| `timestamp` | string | REQUIRED | ISO 8601 event timestamp. |

JSON Schema: [`postback-agent-payload-v0.1.json`](../../schema/json-schema/postback-agent-payload-v0.1.json)

### 4.5 URL Template Variables

The `postback_url_template` supports the following 12 substitution
variables. This is a **closed set** in v0.1; no additional variables MAY be
introduced without a spec revision.

| Variable | Source field | Description |
|----------|-------------|-------------|
| `{tracking_id}` | `tracking_id` | Tracking identifier from the original click. |
| `{offer_id}` | `offer_id` | Identifier of the converted offer. |
| `{conversion_type}` | `conversion_type` | Conversion classification value. |
| `{amount}` | `amount` | Gross converted amount. |
| `{bid_amount}` | `bid_amount` | Computed bid amount. |
| `{currency}` | `currency` | ISO 4217 currency code. |
| `{sub_id_1}` | `sub_id_1` | Custom tracking parameter 1 (empty string when not set). |
| `{sub_id_2}` | `sub_id_2` | Custom tracking parameter 2 (empty string when not set). |
| `{sub_id_3}` | `sub_id_3` | Custom tracking parameter 3 (empty string when not set). |
| `{sub_id_4}` | `sub_id_4` | Custom tracking parameter 4 (empty string when not set). |
| `{sub_id_5}` | `sub_id_5` | Custom tracking parameter 5 (empty string when not set). |
| `{timestamp}` | `timestamp` | ISO 8601 event timestamp. |

Substitution rules are detailed in S9.

### 4.6 Retry Policy

If the Agent developer's endpoint does not return an HTTP 2xx response
within **10 seconds**, AON MUST retry delivery using the following
schedule:

| Attempt | Delay after previous attempt |
|---------|------------------------------|
| 1 | Immediate (first delivery) |
| 2 | 1 minute |
| 3 | 5 minutes |
| 4 | 30 minutes |
| 5 | 2 hours |

After 5 failed attempts the postback is marked as **permanently failed**.
The maximum total retry window is approximately 24 hours from the initial
attempt.

Each retry MUST use the same `event_id` and payload body but MUST
regenerate `X-AON-Nonce` and `X-AON-Timestamp` (see S6).

### 4.7 Response Semantics

Any HTTP 2xx status code is treated as successful delivery. AON ignores
the response body entirely; Agent developers are **not** required to
return an `ApiResponse` envelope.

Any non-2xx response or timeout (> 10 seconds) triggers retry per S4.6.

### 4.8 Example

See [`examples/http/postback/agent/basic-conversion.http`](../../examples/http/postback/agent/basic-conversion.http)
and [`examples/http/postback/agent/retry-scenario.http`](../../examples/http/postback/agent/retry-scenario.http).

HMAC test vectors: [`examples/http/postback/agent/signature-verification.md`](../../examples/http/postback/agent/signature-verification.md).

## 5. Part B -- Partner -> AON Postback

### 5.1 Endpoint

| Field | Value |
|-------|-------|
| URL | `POST {aon_base_url}/v1/postback` |
| Production `aon_base_url` | `https://api.agentoffernetwork.com` |
| Content-Type | `application/json` |

### 5.2 Request Method & Headers

- **Method**: `POST`
- **Content-Type**: `application/json`
- **Authentication headers**: `X-AON-Key`, `X-AON-Timestamp`, `X-AON-Nonce`,
  `X-AON-Signature` per S3.
- **Optional**: `X-AON-Attempt-Count: {int}` -- Partner MAY include the
  retry attempt number (starting from 1). AON uses this for retry behavior
  analysis and anomaly detection; it is not a business field.

### 5.3 Request Payload

Partner sends a JSON body with a shared envelope and event-type-specific
fields. The `event_type` field discriminates the variant.

#### 5.3.1 Shared Envelope (all event types)

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `event_id` | string | REQUIRED | Partner-generated unique identifier (`evt_pb_{ulid}`). AON deduplication key. |
| `event_type` | string | REQUIRED | `"conversion"`, `"refund"`, or `"adjustment"`. |
| `tracking_id` | string | REQUIRED | AON tracking identifier from the original click (attribution key). |
| `offer_id` | string | REQUIRED | Identifier of the associated offer. |
| `conversion_id` | string | REQUIRED | Partner-side unique identifier for the conversion or transaction. |
| `timestamp` | string | REQUIRED | ISO 8601 event timestamp. |

#### 5.3.2 Conversion Fields (`event_type = "conversion"`)

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `conversion_type` | string | REQUIRED | `sale`, `lead`, `install`, `subscription`, `trial`, or `custom`. |
| `amount` | number | REQUIRED | Order amount. |
| `bid` | number | REQUIRED | Commission for this conversion (Partner's payout to AON, net of Partner's margin). |
| `currency` | string | REQUIRED | ISO 4217 currency code. |
| `sub_id_1` .. `sub_id_5` | string | OPTIONAL | Custom tracking parameters inherited from click. |
| `coupon_code` | string | OPTIONAL | Coupon code for coupon-based attribution scenarios. |

#### 5.3.3 Refund Fields (`event_type = "refund"`)

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `original_event_id` | string | REQUIRED | `event_id` of the original conversion postback being refunded. |
| `refund_amount` | number | REQUIRED | Refund amount (positive number). |
| `currency` | string | REQUIRED | ISO 4217 currency code. |
| `refund_reason` | string | OPTIONAL | Human-readable refund reason (free text). |

#### 5.3.4 Adjustment Fields (`event_type = "adjustment"`)

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `original_event_id` | string | REQUIRED | `event_id` of the original conversion postback being adjusted. |
| `adjustment_amount` | number | REQUIRED | Adjustment amount. Positive = increase, negative = decrease. |
| `currency` | string | REQUIRED | ISO 4217 currency code. |
| `adjustment_reason` | string | OPTIONAL | Human-readable adjustment reason. |

JSON Schema: [`postback-partner-payload-v0.1.json`](../../schema/json-schema/postback-partner-payload-v0.1.json)

### 5.4 Retry Policy

Partners SHOULD retry at least 3 times on non-2xx responses with
exponential backoff. Recommended schedule (non-mandatory):

| Attempt | Delay after previous attempt |
|---------|------------------------------|
| 1 | Immediate (first delivery) |
| 2 | 1 minute |
| 3 | 10 minutes |
| 4 | 1 hour |

Each retry MUST use the same `event_id` but MUST regenerate `X-AON-Nonce`
and `X-AON-Timestamp` (see S6).

Partners SHOULD send `X-AON-Attempt-Count` with the current attempt number.

AON SHOULD respond within 3 seconds. If AON responds with `429
RATE_LIMITED`, Partners SHOULD respect the `Retry-After` header.

### 5.5 Response Semantics

AON returns a standard `ApiResponse` envelope.

**Success (first receipt):**

```json
{
  "code": "SUCCESS",
  "message": "",
  "data": {},
  "extra": { "event_id": "evt_pb_NWS66E195DM66YEK30HJKSVVYN", "deduplicated": false }
}
```

**Success (deduplication hit):**

```json
{
  "code": "SUCCESS",
  "message": "",
  "data": {},
  "extra": { "event_id": "evt_pb_NWS66E195DM66YEK30HJKSVVYN", "deduplicated": true }
}
```

Both cases return HTTP 200. Partners SHOULD treat any `code: "SUCCESS"`
response as successful delivery regardless of the `deduplicated` flag.

### 5.6 Example

See [`examples/http/postback/partner/conversion.http`](../../examples/http/postback/partner/conversion.http),
[`examples/http/postback/partner/refund.http`](../../examples/http/postback/partner/refund.http),
and [`examples/http/postback/partner/error-unauthorized.http`](../../examples/http/postback/partner/error-unauthorized.http).

HMAC test vectors: [`examples/http/postback/partner/signature-verification.md`](../../examples/http/postback/partner/signature-verification.md).

## 6. Idempotency

### 6.1 Event ID Generation

Every postback event MUST carry an `event_id` in the format
`evt_pb_{ulid}` where `{ulid}` is a 26-character [ULID](https://github.com/ulid/spec)
(Crockford Base32, monotonic within millisecond).

Regex: `^evt_pb_[0-9A-HJKMNP-TV-Z]{26}$`

- **Part A**: AON generates the `event_id` when a conversion is attributed.
- **Part B**: Partner generates the `event_id` when the business event
  (conversion/refund/adjustment) occurs.

### 6.2 Deduplication Window

The receiver MUST deduplicate on `event_id` with a minimum 5-minute
window. Within this window, if the same `event_id` arrives again, the
receiver MUST return success without reprocessing:

- **Part A (Agent)**: Return HTTP 2xx; body ignored by AON.
- **Part B (AON)**: Return `200 SUCCESS` with `extra.deduplicated: true`.

### 6.3 Retry and Deduplication Interaction

When retrying a postback:

1. The sender MUST reuse the **same** `event_id` and the **same** payload body.
2. The sender MUST regenerate `X-AON-Nonce` (fresh UUIDv4) and
   `X-AON-Timestamp` (current Unix time).
3. The sender MUST recompute `X-AON-Signature` over the new signing string.

This ensures that:
- Business-level deduplication works (same `event_id`).
- HTTP-level replay protection works (different nonce + timestamp pass
  anti-replay checks).
- Signature verification works (fresh signature matches fresh headers).

## 7. Signature Verification

### 7.1 Verification Algorithm (Pseudocode)

```python
def verify_postback(request, secret):
    # Step 1: Extract headers
    key_id    = request.headers["X-AON-Key"]
    timestamp = request.headers["X-AON-Timestamp"]
    nonce     = request.headers["X-AON-Nonce"]
    signature = request.headers["X-AON-Signature"]

    # Step 2: Timestamp check (MUST before signature)
    if abs(server_now() - int(timestamp)) > 300:
        return error(401, "timestamp outside allowed skew")

    # Step 3: Nonce check (SHOULD / MUST depending on role)
    if nonce_store.exists(key_id, nonce):
        return error(401, "nonce already used")
    nonce_store.set(key_id, nonce, ttl=300)

    # Step 4: Reconstruct signing string
    raw_body = request.raw_body()  # exact received bytes, no re-serialization
    signing_string = (
        request.method + "\n" +
        request.path   + "\n" +
        raw_body       + "\n" +
        timestamp      + "\n" +
        nonce
    )

    # Step 5: Compute expected signature
    expected = hmac_sha256(secret, signing_string).hex_lowercase()

    # Step 6: Constant-time comparison
    if not constant_time_equal(expected, signature):
        return error(401, "invalid signature")

    # Step 7: Proceed to business logic
    return process(request)
```

### 7.2 Canonical Body Rules

- The canonical body is the **raw UTF-8 bytes** as received on the wire.
- The verifier MUST NOT re-serialize, re-order keys, or normalize
  whitespace before hashing.
- This matches the convention used in
  [`offer-provider-api.md` S4.2](./offer-provider-api.md#42-signing-string).

### 7.3 Timestamp and Nonce Window

- Timestamp: `|server_now - X-AON-Timestamp| > 300` seconds MUST reject.
- Nonce: `(key_id, nonce)` pair must not have been seen in the last
  5 minutes.
- Both windows are 5 minutes to keep implementation simple and symmetric.

## 8. Error Codes

Error responses follow the AON `ApiResponse` contract, consistent with
[`offer-provider-api.md` S9](./offer-provider-api.md#9-error-codes).

| HTTP | `code` | When it happens |
|-----:|--------|-----------------|
| 200 | `SUCCESS` | Part B: event accepted or deduplicated. |
| 400 | `BAD_REQUEST` | Payload format error, missing REQUIRED fields, invalid `event_type`. |
| 401 | `UNAUTHORIZED` | Missing auth header, invalid signature, expired timestamp, or replayed nonce. |
| 403 | `FORBIDDEN` | Valid `appkey` but suspended or not permitted. |
| 404 | `NOT_FOUND` | `tracking_id` or `original_event_id` not found. |
| 422 | `UNPROCESSABLE_ENTITY` | Fields are valid but semantically conflicting (e.g. `refund_amount` > original `amount`). |
| 429 | `RATE_LIMITED` | Frequency cap exceeded. Response SHOULD include a `Retry-After` header (integer seconds). |
| 500 | `INTERNAL_ERROR` | AON internal failure. |

Note: Part A (AON -> Agent) does not require Agent to return `ApiResponse`
envelopes; any 2xx is treated as success. The error codes above apply
primarily to Part B (Partner -> AON).

## 9. URL Template Substitution Rules

This section applies to Part A only.

### 9.1 Variable Encoding

After substitution, each variable value is **percent-encoded** per
[RFC 3986 S2.1](https://datatracker.ietf.org/doc/html/rfc3986#section-2.1)
when it appears in a URL path or query parameter position. Values
containing characters outside the unreserved set (`A-Z a-z 0-9 - . _ ~`)
MUST be percent-encoded.

### 9.2 Empty Values

When a variable's source value is absent (e.g. `sub_id_3` was not set on
the original click), the variable MUST be replaced with an **empty string**.

Example: `https://agent.example.com/pb?sid3={sub_id_3}` becomes
`https://agent.example.com/pb?sid3=`

### 9.3 Unknown Variables

AON MUST NOT substitute unknown variables (any `{variable_name}` not in
the 12-variable closed set). If the template contains an unknown variable,
AON MUST treat this as a configuration error and fail fast (do not deliver
the postback; log an error). Templates MUST be validated at registration
time.

### 9.4 Closed Set

The 12 variables listed in S4.5 form a **closed set** in v0.1. Adding new
variables requires a spec revision to v0.2 or later.

## 10. Versioning

- **URL path**: Part B endpoint uses `/v1/` in the path
  (`POST /v1/postback`). Backward-incompatible changes MUST be introduced
  under a new `/v{N+1}/` path.
- **Schema file naming**: JSON Schema files use `v0.1` suffix
  (`postback-agent-payload-v0.1.json`, `postback-partner-payload-v0.1.json`).
- **Backward-compatible additions** (new OPTIONAL fields, new enum values)
  MAY land within `/v1/` in a minor semantic version (e.g. `v0.2`).
  Consumers SHOULD ignore unknown fields and handle unknown enum values
  gracefully.
- **Backward-incompatible changes** (removing fields, renaming REQUIRED
  fields, tightening validation) require `/v2/` and a deprecation plan.

## 11. Security Considerations

### 11.1 Transport Security

All postback communication MUST use HTTPS (TLS 1.2 or later). Plaintext
HTTP endpoints MUST NOT be registered.

### 11.2 Secret Management

- Secrets SHOULD be rotated at least every 90 days.
- During rotation, both old and new secrets SHOULD be accepted for a grace
  period (RECOMMENDED: 24 hours).
- Secrets MUST NOT be logged, included in error messages, or transmitted in
  URL query parameters.

### 11.3 PII Considerations

Postback payloads MAY contain PII-adjacent fields:

| Field | PII risk | Mitigation |
|-------|---------|------------|
| `tracking_id` | Pseudonymous identifier | Short TTL; rotated per click |
| `sub_id_1` .. `sub_id_5` | May contain user-supplied data | Agent developers MUST NOT store raw sub_id values beyond the attribution window without consent |
| `coupon_code` | Low risk | No PII in standard usage |

Partners and Agent developers MUST comply with applicable data protection
regulations (GDPR, CCPA, etc.) when processing postback data.

## 12. Design Decisions

1. **Single file, dual Part.** Postback is a single protocol concern with
   two directions. A single document avoids cross-file inconsistency and
   makes it easy to maintain shared mechanics (signing, idempotency) in
   one place.

2. **`event_id` + nonce dual protection.** `event_id` provides
   business-level idempotency (same conversion across retries), while
   `X-AON-Nonce` provides HTTP-level replay protection (each attempt is
   cryptographically unique). Collapsing them into one field would force
   a choice between idempotency and replay protection.

3. **Agent free from ApiResponse.** Part A receivers (Agent developers)
   only need to return HTTP 2xx. Mandating `ApiResponse` for Agent
   endpoints would raise the integration bar for minimal benefit -- AON
   only needs a success signal, not structured data back.

4. **Part B retry SHOULD vs Part A retry MUST.** AON controls its own
   retry infrastructure (MUST 5-step schedule). Partner retry is advisory
   (SHOULD >= 3 times) because AON cannot enforce Partner-side behavior,
   and Partners vary in infrastructure maturity.

5. **5-minute deduplication window.** Matches the nonce/timestamp anti-replay
   window for implementation simplicity. Events older than 5 minutes that
   arrive with the same `event_id` are treated as new (AON handles late
   duplicates via settlement reconciliation, outside the scope of this
   spec).

6. **`X-AON-Attempt-Count` as OPTIONAL.** This header is useful for AON
   operational analytics (e.g. Partner anomaly detection) but is not a
   business field. Making it OPTIONAL keeps the L1 bar low for Partners.

7. **`tracking_id` replaces `aon_id`.** The legacy `aon_id` field in the
   Partner onboarding manual has been renamed to `tracking_id` for
   consistency with the `events.md` click/conversion event schema. Since
   the protocol is in v0.1 Draft with no GA consumers, this is a
   non-breaking rename.

8. **12-variable closed set for URL templates.** Limiting to a fixed set
   prevents template injection and keeps AON's substitution engine simple.
   New variables require a spec revision, ensuring they are deliberately
   designed.

## 13. Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1 | 2026-04-17 | Initial draft. Consolidated from `events.md` Postback section; added Part B (Partner -> AON) with conversion/refund/adjustment event types; added `event_id` (ULID-based) idempotency; aligned signing with F009 `offer-provider-api.md`; added JSON Schema, HTTP examples, and HMAC test vectors. |
