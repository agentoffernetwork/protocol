# Postback

> **Current canonical contract**
>
> Wire version `1.0` defines two separate trust boundaries: an Offer Provider
> reports a conversion to AON, and AON delivers an attributed conversion to a
> Developer Application / Agent callback. Runtime rollout is independent from
> contract adoption; see [Rollout boundary](#9-rollout-boundary).

The key words **MUST**, **MUST NOT**, **SHOULD**, and **MAY** are normative.

## 1. Scope and canonical sources

This specification defines:

- **Provider → AON:** current JSON Postback intake at
  `POST /v1/postback/{partner_id}`;
- **AON → Agent:** current signed conversion webhook delivery to a registered
  Developer Application callback; and
- the fail-closed selection and rollout boundaries that prevent either
  direction from being silently interpreted as another protocol version.

The machine-readable sources define payload fields, requiredness, data types,
patterns, paired fields, exactly-one constraints, and closed-object behavior.
Resolve these public paths in the
[`agentoffernetwork/schema`](https://github.com/agentoffernetwork/schema)
repository at the commit recorded by the protected release manifest:

- Provider payload: `v1.0/json-schema/postback-partner-payload.json`
- Agent payload: `v1.0/json-schema/postback-agent-payload.json`
- shared Goal identity: `v1.0/json-schema/goal-event-name.json`
- public TypeScript contract names and response types:
  `v1.0/types/postback.types.ts`
- Provider semantic validation: `v1.0/validators/postback-semantics.mjs`

This document does not duplicate those field definitions. If prose and a
machine-readable payload constraint disagree, the matching v1.0 JSON Schema is
the payload-shape authority. Semantic rules that depend on the attributed
Offer, pricing model, identity history, or transport state remain normative in
this specification and the semantic validator.

Provider intake does **not** define a Partner-to-AON HMAC, nonce, or Agent API
key. The `partner_id`, configured route scope, enablement, and IP policy form
that inbound trust boundary. Agent callback credentials are separate and MUST
NOT be reused as Provider credentials.

## 2. Provider → AON version and transport selection

### 2.1 Header-first selector

Version selection MUST run before query extraction, query/body merge, form
decoding, or conversion-event processing.

| `AON-Protocol-Version` state | Selected lane | Required behavior |
| --- | --- | --- |
| Exactly one value equal to `1.0` | v1.0 canonical | Lock the request to this specification. |
| Header absent | reject | Return `unsupported_protocol_version` before durable intake or financial processing. |
| Empty, duplicated, malformed, invalidly encoded, historical, approximate, range, or any other value | reject | Return `unsupported_protocol_version` before durable intake or financial processing. |

Header uniqueness is evaluated before comma joining or choosing a first/last
value. A duplicate header is invalid even when both values are `1.0`.
Because no current lane was selected, selector failures MUST NOT include an
`AON-Protocol-Version: 1.0` response header. Responses include that header only
after the unique exact `1.0` request value has selected the current lane.

After the exact `1.0` value selects the current lane, the deployment-level
current-intake gate is evaluated. When that gate is closed, AON MUST return
`503` with reason `postback_v10_unavailable`, `retryable=true`, and `data={}`.
It MUST NOT invoke durable Postback intake, settlement, or outbound enqueue.

### 2.2 Endpoint and media type

The canonical Provider request is:

```text
POST {aon_base_url}/v1/postback/{partner_id}
Content-Type: application/json
AON-Protocol-Version: 1.0
```

The current lane accepts only `POST`, a JSON media type, no query string, and
one JSON object body conforming to the Provider payload schema. `GET`, form
data, query fields, `event_params_base64`, and non-JSON content are not current
v1.0 inputs. The payload MUST NOT contain a `version` field; the HTTP header is
the single version selector. Unknown fields are rejected by the closed schema.

The route's configured Partner scope, enablement, and IP policy still apply.
They do not alter the payload contract and cannot redirect a selected v1.0
request into another version lane.

`partner_id` is the UUID assigned to the Provider route. The canonical HTTP
examples use a syntactically valid UUID so they can be sent without replacing a
non-runtime placeholder.

## 3. Provider payload semantics

### 3.1 Goal identity

`event_name`, the attributed Offer's `goals[].event`, and the Agent webhook's
`event_name` all reference the same v1.0 Goal schema. Implementations MUST NOT
copy or broaden that grammar locally. A syntactically valid event is mapped by
exact, case-sensitive equality to a Goal declared by the attributed Offer.

A syntactically valid but undeclared event produces the stable `unmapped` /
`unmapped_event` result. It is not silently mapped to a default Goal and does
not create settlement. A value that does not satisfy the shared grammar is an
invalid payload.

The current Offer contract accepts only `first` and `all` as
`dedup_strategy`. `highest` is not a v1.0 current strategy and MUST be rejected
by current schema, types, runtime validation, and rollout audits.

### 3.2 Attribution anchor

The Provider schema requires exactly one of `aon_click_id`,
`aon_tracking_id`, or `offer_instance_id`. Zero anchors and multiple anchors
are invalid. There is no precedence rule that permits an implementation to
discard one supplied anchor in favor of another. Anchor values are non-empty
and MUST NOT contain surrounding whitespace; `offer_instance_id` is a UUID.

### 3.3 Revenue facts

`amount` and `currency` are a pair. Their wire representation and validation
are defined only by the Provider schema.

- For a CPA Goal, the pair MAY be omitted. If either value is present, both are
  required.
- For a CPS Goal, both values are required after Goal resolution. Missing
  revenue returns terminal 400 semantics using `REVENUE_REQUIRED` or
  `REVENUE_CURRENCY_REQUIRED` as applicable.

Revenue parsing MUST preserve decimal precision. AON compares normalized
decimal and currency facts for replay/conflict decisions; it does not use a
binary floating-point rendering of the Provider's decimal string.

### 3.4 Business identity and replay

`event_id`, `order_id`, and `partner_txn_id` are optional typed business
identity aliases defined by the Provider schema. Their semantic rules are:

- values are trimmed for validation, remain case-sensitive, and MUST be no
  longer than 256 UTF-8 bytes after normalization;
- each field has its own namespace, so `event_id:x` and `order_id:x` are not
  the same alias;
- when multiple aliases arrive together, all are bound atomically to one
  claim; event, order, then Partner transaction is only the canonical display
  priority and does not discard the other aliases;
- a retry MUST reuse the original identity kind and value. A later, previously
  unbound alias is not equivalent merely because its text matches another
  namespace; and
- aliases that already resolve to different claims produce
  `request_identity_conflict`.

For `dedup_strategy=all`, at least one explicit business identity is required.
Within one scope lock and transaction, AON first resolves existing aliases and
compares immutable normalized revenue facts. Identical facts are an
`already_recorded` replay; different facts are a conflict. AON MUST perform
that comparison before binding a newly supplied alias, so a conflicting
request leaves the alias set and all financial facts unchanged.

For `dedup_strategy=first`, an explicit business identity is optional. The
claim scope is the Partner, attributed chain, and frozen Goal identity/version;
the first conversion wins and later matching requests are duplicates rather
than updates.

Request-level replay identity and business conversion identity are distinct.
A durable request hash may safely replay an intake attempt, but it MUST NOT
replace the typed business alias rules above. Concurrent requests MUST converge
on one claim and one financial outcome.

The Provider JSON Schema marks the byte limit with
`x-aon-maxUtf8Bytes: 256`; the canonical schema CI registers that keyword as an
executable string validator. Implementations that do not support extension
keywords MUST run the linked semantic validator in addition to ordinary JSON
Schema validation.

## 4. Provider response contract

Every response to a request that successfully selected the current lane MUST
echo:

```text
AON-Protocol-Version: 1.0
```

The outer response uses the hosted API envelope keys `code`, `message`,
`data`, and `extra`. The response field names and result unions are defined by
`ProviderPostbackResponseV10` and `ProviderPostbackErrorExtraV10` in the
v1.0 types source linked in section 1.

### 4.1 Success and durable results

For 2xx outcomes, `data` contains exactly the Provider response data:
`result`, `event_logged`, `reason`, `correlation_id`, and `retryable`.
Stable result classes are `accepted`, `already_recorded`, `unmapped`,
`rejected`, and `retry`.

| Outcome | Typical HTTP result | Client meaning |
| --- | ---: | --- |
| `accepted` | 200 | The conversion was durably accepted. |
| `already_recorded` | 200 | This is a safe replay or first-wins duplicate; do not create another business conversion. |
| `unmapped` / `unmapped_event` | 200 | The event was recorded for audit but did not match a declared Goal and was not settled. |
| identity conflict | 409 | The submitted aliases or immutable business facts conflict; correct the request rather than retrying it unchanged. |

### 4.2 Rejections and recoverable failures

Every 4xx or 5xx response has `data` exactly equal to `{}`. Diagnostic fields
appear only in `extra`, which contains `result`, `reason`, `correlation_id`,
`retryable`, and `event_logged`.

- Schema, method, media-type, query, version, scope, and identity validation
  failures are terminal 4xx results with `retryable=false`.
- A pre-durable internal/configuration/context failure returns 500 with reason
  `internal_error`, `retryable=true`, and `event_logged=false`.
- Durable intake or processing unavailability returns 503 with reason
  `postback_intake_unavailable`, `retryable=true`, and `event_logged` matching
  whether the business event was durably created.
- A closed current-intake gate uses the distinct 503 reason
  `postback_v10_unavailable` and `event_logged=false`.

`correlation_id` is the stable support key shared by the response, request
journal metadata, and health/audit context. Audit storage legs are independent
best-effort operations: an audit failure does not change the original HTTP
result. `event_logged` reports whether a business Event exists; it does not
claim that every audit leg persisted successfully.

A Partner SHOULD stop retrying after any 2xx or terminal 4xx result. It MAY
retry 500/503 results with backoff while preserving the original business
identity and facts.

## 5. AON → Agent callback transport

### 5.1 Callback URL and frozen wire version

The callback belongs to a Developer Application, not a Partner credential.
Production callback URLs MUST use HTTPS. An explicitly configured loopback HTTP
URL MAY be used in local/test environments only. Userinfo and fragments are
invalid. AON MUST preserve the configured path, repeated/trailing slashes,
percent-encoded bytes, raw query order, and query spelling when it constructs
the origin-form request-target.

AON freezes a wire version and exact raw body when it enqueues an event:

- an event accepted through the exact v1.0 Provider intake lane is stored as
  wire `1.0` and every attempt carries the current version header.

The worker MUST use the stored wire version. Goal grammar validates the payload;
it MUST NOT select a wire version. Neither the enqueue path nor worker may parse
the event name or frozen body to guess, upgrade, downgrade, or repair the
version.

### 5.2 Request headers

Every current delivery uses `POST`, `Content-Type: application/json`, and
exactly one value for each header below:

| Header | Requirement |
| --- | --- |
| `AON-Protocol-Version` | Exact value `1.0`; selects the closed current payload. |
| `X-AON-Key` | Opaque callback key id selecting one current or grace secret. |
| `X-AON-Timestamp` | Canonical Unix epoch seconds used in the signing string. |
| `X-AON-Signature` | Lowercase-hex HMAC-SHA256 over the exact wire inputs. |

A receiver MUST reject a missing, empty, or duplicate value for any of these
four headers before HMAC verification or JSON parsing. It MUST NOT choose a
first/last value from duplicated security headers.

If AON lacks a callback URL, key id, secret, frozen body, or other required
wire fact, it MUST record an observable pre-HTTP delivery error and MUST NOT
send an unsigned or partial request.

### 5.3 Agent payload and size bound

The exact JSON body MUST conform to the Agent payload schema linked in section
1. The object is closed and has no payload version field. `amount` and
`currency` are either both present or both absent. A normal CPA conversion
without revenue therefore remains a valid current delivery; CPS and other
revenue-bearing conversions include the pair when those facts exist.

The maximum current raw body size is 2,097,152 bytes (2 MiB). A sender MUST
reject an oversized body before HTTP delivery. A receiver MUST enforce a
transport limit no greater than 2 MiB and MUST reject an oversized complete
body before HMAC calculation or JSON parsing.

Optional `sub_id` fields are omitted when absent, never synthesized as `null`.
The payload's `timestamp` is conversion event time; it is distinct from
`X-AON-Timestamp`, which is attempt signing time. Payload event time is an
RFC3339 `date-time` normalized to UTC (`Z`, `z`, or `+00:00`); non-UTC offsets
and timezone-free values are invalid.

## 6. Signature and receiver verification

### 6.1 Signing string

For every attempt, AON computes the signing string from the exact bytes sent:

```text
POST\n{request-target}\n{exact-raw-body}\n{timestamp}
```

`{request-target}` is the exact origin-form path plus `?raw_query` when a query
is present. It excludes scheme, authority, userinfo, and fragment. Neither side
may percent-decode, normalize slashes, reorder query parameters, alter percent
encoding case, or otherwise reconstruct this value.

`{exact-raw-body}` is the frozen byte sequence transmitted on the wire.
Receivers MUST verify the signature over those bytes before parsing or
re-serializing JSON. `{timestamp}` is the exact `X-AON-Timestamp` value. The
signature is `HMAC-SHA256(secret, signing_string)` encoded as lowercase hex and
MUST be compared in constant time.

The version header is required but is not an additional component of this
signing string. Its uniqueness and exact `1.0` value are checked before HMAC.
Changing the request-target, raw body, or timestamp invalidates the signature.

### 6.2 Timestamp and keyring

`X-AON-Timestamp` MUST be unsigned ASCII epoch seconds with no sign,
whitespace, or leading zeroes except the canonical value `0`. A receiver
accepts a timestamp at most 300 seconds before or after its current clock;
exactly 300 seconds is accepted and 301 seconds is rejected.

Callback keys have a lifecycle independent from Provider credentials. A
receiver keyring contains one current secret and MAY contain one explicitly
configured grace-generation secret. `X-AON-Key` selects exactly one of those
generations. Unknown or expired key ids MUST be rejected without trying every
secret or falling back to a different generation.

### 6.3 Verification order

A production receiver MUST apply this order:

1. enforce the transport body limit and require each of the four headers
   exactly once;
2. require `AON-Protocol-Version: 1.0` and a canonical timestamp within the
   allowed clock skew;
3. select exactly one current or grace secret from `X-AON-Key`;
4. verify lowercase-hex HMAC in constant time over the exact request-target,
   raw body, and timestamp;
5. only after HMAC succeeds, parse JSON and validate the closed Agent schema;
   and
6. apply the durable idempotency transaction in section 7 before any business
   side effect.

An explicitly local/test-only insecure mode MAY skip timestamp, key, and HMAC
checks, but it MUST still enforce version, size, schema, and idempotency rules.
Such a mode MUST fail closed if enabled in production.

## 7. Agent receiver idempotency

A receiver MUST retain a durable record keyed by `(agent_id, event_id)` with a
SHA-256 digest of the exact raw body for at least 24 hours after first
successful receipt.

The first insert of key and digest, the decision to apply the business side
effect, and the receiver's durable state transition MUST be atomic:

- a first valid key/body applies the business effect once and records the
  digest;
- the same key and same digest is a replay and MUST return 2xx without another
  business effect; and
- the same key with a different digest MUST return 409 and MUST NOT process the
  new body.

An SDK verifier may return the key, raw bytes, digest, protocol version, key id,
and signed-at timestamp, but it does not provide durable storage or an
exactly-once guarantee. That transaction remains the receiver's responsibility.

## 8. Delivery, retries, and version isolation

### 8.1 Current delivery policy

Any 2xx response completes an Agent delivery; AON ignores its body. Redirects
MUST NOT be followed. A 3xx, other non-2xx response, or timeout follows the
fixed five-attempt ladder:

| Attempt | Delay after the prior attempt |
| ---: | --- |
| 1 | immediate |
| 2 | 1 minute |
| 3 | 5 minutes |
| 4 | 30 minutes |
| 5 | 2 hours |

AON waits at most 10 seconds for an attempt response. Across retries,
`event_id` and the exact raw body remain unchanged. Each attempt uses a fresh
canonical timestamp and matching signature. Delivery stops immediately after
the first 2xx; no sixth attempt is allowed. A slow or failing callback MUST NOT
prevent unrelated applications from making progress.

### 8.2 Version isolation

A selected current request or stored current callback MUST NOT be downgraded,
relabeled, rewritten, or stripped of its version header after an error. Missing,
historical, malformed, or unsupported selectors do not enter this current
contract and MUST NOT be used as fallback for unavailable v1.0 traffic.

## 9. Rollout boundary

Publishing this specification does not prove that a deployment accepts v1.0.
Each environment MUST keep current intake and Agent current delivery disabled
until its writer, schema, signing, retry, idempotency, and callback evidence is
complete. The deployment owner must certify the exact selector/echo behavior,
closed payloads, five-attempt retry ladder, frozen raw body, signature vectors,
and replay rules before claiming v1.0 support. Until certification, the surface
is `unvalidated`; an exact Provider `1.0` request fails closed as unavailable.

When v1.0 delivery is disabled or certification fails, every queued v1.0 event
MUST remain preserved with its stored wire version and frozen raw body. It MUST
NOT be reclassified, rewritten, deleted as a fallback action, or sent by a
worker that has not been certified for this contract. Terminal outbox retention
may scrub target URL, raw body, and last error, but it MUST preserve the
conversion id, stored wire version, terminal status, attempt count, and last
status-code tombstone. A fallback scan MUST treat that tombstone as already
registered and MUST NOT enqueue a sixth or post-retention delivery.

The public repository map, release manifest, digest, and publication state are
updated only by protected publication work with Release Owner approval and
repository controls. Runtime rollout evidence is maintained independently; a
published protocol does not assert that an implementation has completed this
rollout.

## 10. Reference vectors and examples

Normative contract vectors in the
[`agentoffernetwork/schema`](https://github.com/agentoffernetwork/schema)
repository are:

- `v1.0/fixtures/postback-agent-webhook.json` — current/grace signatures,
  300/301-second boundaries, retry attempts, and receiver idempotency; and
- `v1.0/fixtures/protocol-contract-vectors.json` — schema/source bindings and
  positive/negative cases.

Canonical requests in the
[`agentoffernetwork/examples`](https://github.com/agentoffernetwork/examples)
repository are:

- `v1.0/http/postback/partner/basic-conversion.http` — Provider first success;
- `v1.0/http/postback/partner/invalid-unknown-field.http` — closed Provider
  payload rejection;
- `v1.0/http/postback/agent/basic-conversion.http` — signed Agent first success;
- `v1.0/http/postback/agent/invalid-duplicate-signature.http` — duplicate
  security-header rejection; and
- `v1.0/http/postback/agent/retry-scenario.http` — frozen-body retry sequence.

Canonical conformance gates MUST fail when schema, Goal grammar, types, vectors,
examples, signatures, or source bindings drift from this contract. Runtime,
OpenAPI, and SDK certification remains deployment- or owner-specific.

## 11. Security and privacy

- Do not log callback secrets, complete signature values, callback URLs, or
  raw bodies containing sensitive conversion data.
- Treat tracking, identity, and `sub_id` values as potentially pseudonymous.
  Retain only what attribution, dispute handling, and applicable law require.
- Keep Provider route/IP authorization, Agent callback credentials, and Agent
  receiver idempotency records in separate trust domains.
- Do not follow redirects with signed headers or a signed raw body.
- Do not claim SDK verification alone provides durable or exactly-once
  processing.

## 12. Changelog

| Version | Date | Changes |
| --- | --- | --- |
| 1.0 | 2026-08-29 | Promoted the bidirectional canonical contract with exact `1.0` selection while preserving closed Provider JSON, shared Goal/identity/revenue semantics, event-level Agent wire version, four required security/control headers, exact-byte HMAC verification, 2 MiB bound, keyring/time/idempotency rules, and the fixed retry behavior. |
