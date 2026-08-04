# Postback Specification v0.2

**Version**: 0.2
**Status**: Current normative v0.2 payload source; runtime follow-up required

**Last Updated**: 2026-07-27

## 1. Scope and implementation status

This specification covers the two conversion-notification directions used by
the AgentOffer Network (AON):

- **Partner conversion intake.** A Partner reports a conversion to AON through
  the existing simplified `GET|POST /v1/postback/{partner_id}` route.
- **AON → Agent conversion webhook.** AON delivers an attributed conversion to
  a Developer Application callback URL.

BL-033 owns the payload contract and Goal identity only. Partner intake runtime
must follow through BL-039. The AON → Agent webhook delivery rules in sections
3–7 remain the normative **WS-22 target contract**. This document does not
claim that either current runtime already implements the v0.2 payload.

This specification does **not** define a Partner-to-AON HMAC, nonce, refund, or
adjustment protocol. AON-issued Partner AppKey/AppSecret credentials remain
for the separate **AON → Partner Offer Fetch** signing contract in
[`offer-provider-api.md`](./offer-provider-api.md).

## 2. Partner conversion intake

### 2.1 Endpoint and trust boundary

Partners send conversion callbacks to their configured route:

```text
GET|POST {aon_base_url}/v1/postback/{partner_id}
```

The `partner_id` identifies the configured Partner route. AON applies the
route's existing Partner scope, enablement, and IP-allowlist checks before it
records a conversion. A Partner must not send `X-AON-Key`,
`X-AON-Timestamp`, `X-AON-Nonce`, or `X-AON-Signature` for this route;
they are not an inbound Partner callback contract.

#### Partner payload

The decoded GET query or POST JSON payload is defined by
[`postback-partner-payload-v0.2.json`](../../schema/json-schema/postback-partner-payload-v0.2.json).
It requires:

- `event_name`: exact non-null identity of the matched Offer
  `goals[].event`; and
- at least one attribution anchor: `offer_instance_id`, `aon_tracking_id`,
  `tracking_id`, `click_id`, or `aon_click_id`.

Optional values are `amount` as a decimal string, `currency`, `order_id`,
`partner_txn_id`, and `event_id`. The object is closed.
`conversion_type` and `bid_amount` are forbidden v0.2 public fields.

### 2.2 Result and retry handling

- A 2xx response means the conversion was recorded or was an idempotent
  duplicate. Do not send the same business conversion again.
- A 4xx response is terminal for the submitted payload. Correct the payload or
  Partner configuration before sending a new request.
- A 5xx response, including 503 intake unavailability, is retryable with
  backoff. Existing conversion identity and journal semantics prevent a retry
  from producing a second settlement.

`POST /v1/events/conversion` remains a legacy conversion endpoint. Its existing
global `x-postback-token` behavior is unchanged: a configured global token must
match, and an unset global token preserves the existing fallback. A Partner
credential configured as `header_value`, `token`, or `shared_token` continues
to validate its configured header. A historical inbound signing setting does
not activate an inbound HMAC reader; it follows the same global-token/fallback
behavior as an absent `postback_auth`.

## 3. AON → Agent conversion webhook

### 3.1 Callback registration

The callback belongs to a Developer Application, not to a Partner credential.
Production callback URLs MUST use HTTPS. Local or test environments MAY allow
an explicit loopback HTTP URL; this exception MUST NOT be enabled in
production.

A callback URL may include a query string. Userinfo (`user:password@host`) and
fragments are invalid configuration. AON sends the configured path exactly:
it MUST NOT normalize, add, remove, or collapse trailing or repeated slashes.

### 3.2 Request headers and signing string

Every delivery uses `POST` with `Content-Type: application/json` and all three
headers below:

| Header | Requirement | Meaning |
|---|---|---|
| `X-AON-Key` | REQUIRED | Opaque, version-specific callback key id. |
| `X-AON-Timestamp` | REQUIRED | Unix epoch seconds as an ASCII decimal string. |
| `X-AON-Signature` | REQUIRED | Lowercase-hex HMAC-SHA256 result. |

For each delivery, AON selects exactly one version-specific callback secret and
computes:

```text
POST\n{request-target}\n{exact-raw-body}\n{timestamp}
```

`{request-target}` is the exact HTTP origin-form request-target bytes sent on
the wire: raw path plus `?raw_query` when a query is present. It excludes
scheme, authority, and fragment. Neither sender nor receiver may percent-decode,
sort query parameters, change case, or parse and re-serialize this value.

`{exact-raw-body}` is the exact JSON byte sequence transmitted on the wire.
The receiver MUST verify it before parsing or re-serializing the body. The
signature is `HMAC-SHA256(secret, signing_string)` encoded as lowercase hex;
the receiver MUST compare it in constant time.

### 3.3 Key rotation and receiver verification

Callback keys and secrets have a lifecycle independent of Partner
AppKey/AppSecret. Each rotation creates a new opaque key id and secret. A
receiver uses `X-AON-Key` to select exactly one current or grace-period secret;
unknown or expired key ids MUST be rejected without trying other secrets.

A production receiver MUST:

1. require the three headers;
2. reject a timestamp more than 300 seconds from its clock (exactly 300 seconds
   is accepted; 301 seconds is rejected);
3. select the keyed secret and verify HMAC in constant time; and
4. validate the payload schema and durable idempotency rule in section 4.

Only an explicit local/test insecure setting may skip timestamp, key, and HMAC
verification. It MUST still validate the schema and idempotency rules. A
production configuration containing that insecure setting MUST fail at startup.
AON always sends signing headers; if its callback key or secret is missing or
empty, it MUST record an observable configuration/delivery error before HTTP
and MUST NOT send an unsigned request.

### Agent payload

The conversion webhook body is defined by
[`postback-agent-payload-v0.2.json`](https://github.com/agentoffernetwork/schema/blob/main/json-schema/postback-agent-payload-v0.2.json).

| Field | Level | Description |
|---|---|---|
| `event_id` | REQUIRED | AON-generated opaque globally unique conversion-event id. |
| `event_type` | REQUIRED | Fixed value: `conversion`. |
| `event_name` | REQUIRED | Exact value of the matched Offer `goals[].event`. |
| `aon_tracking_id` | REQUIRED | AON tracking identifier for the attributed conversion. |
| `offer_id` | REQUIRED | Converted offer identifier. |
| `agent_id` | REQUIRED | Developer Application / Agent identifier. |
| `amount` | REQUIRED | Gross converted amount. |
| `currency` | REQUIRED | ISO 4217 currency code. |
| `sub_id` | OPTIONAL | First developer attribution slot, inherited from the click. |
| `sub_id_2` … `sub_id_5` | OPTIONAL | Additional developer attribution slots, inherited from the click. |
| `timestamp` | REQUIRED | ISO 8601 conversion-event time. |

Optional attribution fields are omitted when absent, never sent as `null`.
`conversion_type` and `bid_amount` are not part of this closed payload.

<a id="goal-identity"></a>
<a id="no-bid"></a>

### Goal identity and no-bid

Both Postback schemas reference the same
[`goal-event-name-v0.2.json`](../../schema/json-schema/goal-event-name-v0.2.json)
definition used by Offer Goals. A well-known or custom slug is valid only when
it exactly matches the unique Goal declared by the attributed Offer. Schema
checks grammar and requiredness; BL-039 owns runtime declaration lookup.

### 3.5 Delivery and retry policy

AON waits at most 10 seconds for a response. Any 2xx response completes the
delivery; AON ignores the response body. A non-2xx response or timeout retries
the same raw body on this fixed schedule:

| Attempt | Delay after the prior attempt |
|---:|---|
| 1 | immediate |
| 2 | 1 minute |
| 3 | 5 minutes |
| 4 | 30 minutes |
| 5 | 2 hours |

Each retry preserves `event_id` and the exact raw body, while using a fresh
timestamp and corresponding signature. After the fifth failed attempt, AON
records a final observable delivery failure.

## 4. Receiver idempotency

A receiver MUST keep a durable record for `(agent_id, event_id)` for at least
24 hours from the first successful receipt.

- A repeated key with identical raw body MUST return a 2xx result without a
  second business side effect.
- A repeated key with a different raw body MUST return HTTP 409 and MUST NOT
  process the new body.

This is business-level idempotency. It replaces any requirement for a separate
nonce cache.

## 5. Reference examples and test vectors

The repository contains the contract fixtures that WS-22-S3 must use as its
compatibility gate:

- [`basic-conversion.http`](https://github.com/agentoffernetwork/examples/blob/main/http/postback/agent/basic-conversion.http)
- [`partner/basic-conversion-v0.2.json`](../../examples/http/postback/partner/basic-conversion-v0.2.json)
- [`retry-scenario.http`](https://github.com/agentoffernetwork/examples/blob/main/http/postback/agent/retry-scenario.http)
- [`signature-verification.md`](https://github.com/agentoffernetwork/examples/blob/main/http/postback/agent/signature-verification.md)

They cover valid and tampered raw bodies, expired timestamps, current,
previous, and unknown key ids, same-body idempotency, conflicting-body 409, and
all five retry times. They are contract fixtures, not evidence that the legacy
notifier already meets this target.

## 6. Security and privacy

- Do not log callback secrets or signature values.
- Treat `aon_tracking_id` and `sub_id` fields as potentially pseudonymous
  data. Retain only what is necessary for attribution and applicable legal
  obligations.
- Use separate callback credentials for AON → Agent delivery. Do not reuse
  Partner AppKey/AppSecret or any Partner inbound-replay storage.

## 7. Changelog

| Version | Date | Changes |
|---|---|---|
| 0.2 | 2026-07-27 | Unified both directions on required `event_name` referencing Offer `goals[].event`; removed public `conversion_type` and `bid_amount`; added Partner/Agent closed payload schemas and bound Agent signing vectors to the canonical example. |
| 0.2 | 2026-07-24 | Retired the unimplemented Partner inbound signing, replay, and reversal-event contract. Kept simplified Partner conversion intake and global-token/fallback semantics. Defined the versioned-key, HMAC/timestamp, raw request-target, durable 24-hour idempotency, and fixed retry target contract for WS-22. Standardized the five attribution fields as `sub_id` plus `sub_id_2`–`sub_id_5`. |
