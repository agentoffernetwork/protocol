# Postback

> **Current canonical contract**
>
> Wire version `1.0` defines two separate trust boundaries: an Offer Provider
> reports a conversion (or reverses one) to AON over a single `GET` query-string
> binding, and AON delivers an attributed conversion event to a Developer
> Application / Agent callback. Runtime rollout is independent from contract
> adoption; see [Rollout boundary](#9-rollout-boundary).

The key words **MUST**, **MUST NOT**, **SHOULD**, and **MAY** are normative.

## 1. Scope and canonical sources

This specification defines:

- **Provider → AON:** the single Postback intake binding,
  `GET /v1/postback/{partner_id}?...`, for reporting a Goal conversion and for
  reversing a previously reported Goal conversion;
- **AON → Agent:** current signed conversion and earning-status webhook delivery to a registered
  Developer Application callback; and
- the retirement of the former JSON body lane and the rollout boundaries that
  prevent either direction from being silently interpreted as another protocol
  version.

Provider → AON parameters are defined by the authoritative query parameter
table in section 2.3; that table is the Provider-side field authority. The
machine-readable sources below define the Agent payload, the shared Goal
grammar, and the public type names. Resolve these public paths in the
[`agentoffernetwork/schema`](https://github.com/agentoffernetwork/schema)
repository at the commit recorded by the protected release manifest:

- Agent payload: `v1.0/json-schema/postback-agent-payload.json`
- shared Goal identity: `v1.0/json-schema/goal-event-name.json`
- public TypeScript contract names and response types:
  `v1.0/types/postback.types.ts`
- Provider semantic validation: `v1.0/validators/postback-semantics.mjs`
- retired Provider JSON body: `v1.0/json-schema/postback-partner-payload.json`
  (deprecated; retained only to describe the retired JSON body lane in
  section 2.4 and as the value-grammar reference for the shared field names)

If prose and a machine-readable Agent payload constraint disagree, the matching
v1.0 JSON Schema is the payload-shape authority for what AON sends; receivers
apply the tolerance rule in section 5.3. Semantic rules that depend on
the attributed Offer, pricing model, identity history, settlement state, or
transport state remain normative in this specification and the semantic
validator.

Provider intake does **not** define a Partner-to-AON HMAC, nonce, or Agent API
key. The `partner_id`, configured route scope, enablement, and IP policy form
that inbound trust boundary for both conversions and reversals. Agent callback
credentials are separate and MUST NOT be reused as Provider credentials.

## 2. Provider → AON transport

### 2.1 Single query-string binding

The canonical Provider request is an HTTP `GET` whose parameters are carried
only in the query string:

```text
GET {aon_base_url}/v1/postback/{partner_id}?aon_click_id={aon_click_id}&event_name={event_name}[&event_id=...|&order_id=...|&partner_txn_id=...][&amount=...&currency=...]
```

A reversal uses the same URL with two additional parameters (section 3.5):

```text
GET {aon_base_url}/v1/postback/{partner_id}?aon_click_id={aon_click_id}&event_name={event_name}[&order_id=...]&action=reversal&reason=refunded
```

- The request has no body and no protocol-version header. The binding itself
  is the version; there is no `version` parameter.
- Parameter values are UTF-8, percent-encoded per RFC 3986. AON decodes each
  value once before validation. Each defined parameter MUST appear at most
  once.
- `aon_click_id` is the RECOMMENDED attribution anchor. Providers SHOULD
  configure their Postback URL template with the AON click macro so that every
  conversion and reversal carries `aon_click_id`.
- Query parameters that are not defined in section 2.3 carry no meaning in this
  contract. A Provider MUST NOT rely on any undefined parameter, alias, or
  alternative method; deployment-specific compatibility tolerance is outside
  this contract.

The route's configured Partner scope, enablement, and IP policy apply equally
to conversions and reversals. They do not alter the parameter contract.

`partner_id` is the UUID assigned to the Provider route. The canonical HTTP
examples use a syntactically valid UUID so they can be sent without replacing a
non-runtime placeholder.

### 2.2 Request kinds

`action` selects the request kind. It is matched case-insensitively.

| `action` | Request kind | Defined in |
| --- | --- | --- |
| absent, empty, or `conversion` | Report a Goal conversion | sections 3.1 – 3.4 |
| `reversal` | Reverse one previously reported Goal conversion | section 3.5 |
| any other value | reject | 400, reason `invalid_action`, before any business processing |

### 2.3 Authoritative query parameter table

| Parameter | Conversion | Reversal | Value rules |
| --- | --- | --- | --- |
| `aon_click_id` | exactly one anchor REQUIRED (RECOMMENDED) | same anchor kind and value as the conversion | Opaque AON click id, typically `aci_…`; non-empty; no surrounding whitespace. |
| `aon_tracking_id` | exactly one anchor REQUIRED | same anchor kind and value as the conversion | Opaque AON tracking id; non-empty; no surrounding whitespace. |
| `offer_instance_id` | exactly one anchor REQUIRED | same anchor kind and value as the conversion | UUID. |
| `event_name` | REQUIRED | REQUIRED; the Goal of the conversion being reversed | Shared v1.0 Goal grammar `^[a-z][a-z0-9_]{0,63}$`; exact, case-sensitive Goal match (section 3.1). |
| `event_id` | OPTIONAL business identity | see section 3.5 | 1–256 UTF-8 bytes after trim; at least one non-whitespace character; case-sensitive. |
| `order_id` | OPTIONAL business identity | see section 3.5 | Same as `event_id`; separate namespace. |
| `partner_txn_id` | OPTIONAL business identity | see section 3.5 | Same as `event_id`; separate namespace. |
| `amount` | paired with `currency`; required for CPS | MUST NOT be sent | Decimal string `^(?:0\|[1-9][0-9]{0,11})(?:\.[0-9]{1,6})?$`. |
| `currency` | paired with `amount`; required for CPS | MUST NOT be sent | ISO 4217 code `^[A-Z]{3}$`. |
| `action` | OPTIONAL (`conversion`) | REQUIRED, `reversal` | Case-insensitive; see section 2.2. |
| `reason` | not used | REQUIRED | One of `refunded`, `chargeback`, `merchant_cancelled`, `invalid_order`, `fraud`, `duplicate`; case-insensitive (section 3.5). |

Exactly one of `aon_click_id`, `aon_tracking_id`, or `offer_instance_id` is
present on every request (section 3.2). `event_id`, `order_id`, and
`partner_txn_id` are the business identity aliases of section 3.4.
Because a reversal is always a whole-conversion reversal, a reversal request
MUST NOT carry `amount` or `currency`; AON never uses them to select or size a
reversal. Partial amounts are not defined in this contract.

### 2.4 Retired JSON body lane

The former Provider lane — `POST` with a JSON body, selected by the
`AON-Protocol-Version` request header (`0.2`, `0.3`, or `1.0`) — is retired.
It had no production traffic when it was removed and has no transition period.

A Provider → AON request that carries an `AON-Protocol-Version` header, with
any value, MUST be rejected with HTTP `410`, reason
`postback_json_lane_retired`, `result="rejected"`, `retryable=false`,
`event_logged=false`, and `data={}`. This check runs before query extraction
and before any durable intake, settlement, or outbound enqueue. The rejection
message SHOULD point the Provider to the `GET` binding in section 2.1.
AON MUST NOT reinterpret a retired-lane request as a `GET` binding request.

The `AON-Protocol-Version` header remains part of the AON → Agent callback
contract (section 5); this retirement applies only to Provider intake.

## 3. Provider request semantics

### 3.1 Goal identity

`event_name`, the attributed Offer's `goals[].event`, and the Agent webhook's
`event_name` all reference the same v1.0 Goal schema. Implementations MUST NOT
copy or broaden that grammar locally. A syntactically valid event is mapped by
exact, case-sensitive equality to a Goal declared by the attributed Offer.

A syntactically valid but undeclared event produces `result="unmapped"` with
`reason="UNMAPPED_EVENT"`. `result` is the stable business-outcome class;
`reason` is the diagnostic reason code for that outcome. They are separate
response dimensions, not interchangeable aliases. The event is not silently
mapped to a default Goal and does not create settlement. A value that does not
satisfy the shared grammar is an invalid request.

The current Offer contract accepts only `first` and `all` as
`dedup_strategy`. `highest` is not a v1.0 current strategy and MUST be rejected
by current schema, types, runtime validation, and rollout audits.

### 3.2 Attribution anchor

Every request carries exactly one of `aon_click_id`, `aon_tracking_id`, or
`offer_instance_id`. Zero anchors and multiple anchors are invalid. There is no
precedence rule that permits an implementation to discard one supplied anchor
in favor of another. Anchor values are non-empty and MUST NOT contain
surrounding whitespace; `offer_instance_id` is a UUID. `aon_click_id` is the
RECOMMENDED anchor for new integrations.

### 3.3 Revenue facts

`amount` and `currency` are a pair. Their value grammar is defined by the
parameter table in section 2.3.

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
identity aliases defined by the parameter table in section 2.3. Their semantic
rules are:

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

The 256-byte limit is measured in UTF-8 bytes after trimming. The semantic
validator linked in section 1 is the executable reference for this limit.

### 3.5 Reversal

A Provider reverses a previously reported Goal conversion — for example after a
refund, chargeback, or merchant cancellation — by sending the same `GET`
binding with `action=reversal` and a `reason`.

**Reason.** `reason` is REQUIRED for a reversal and MUST be one of:

| `reason` | Meaning |
| --- | --- |
| `refunded` | The customer was refunded. |
| `chargeback` | The payment was charged back. |
| `merchant_cancelled` | The merchant cancelled the order. |
| `invalid_order` | The order was invalid or never fulfilled. |
| `fraud` | The conversion was fraudulent. |
| `duplicate` | The conversion duplicated another conversion. |

`reason` is matched case-insensitively and stored in lowercase. A missing or
unknown value returns 400 `INVALID_REVERSAL_REASON`.

**Target location.** AON locates exactly one previously accepted conversion
using:

1. the Partner route (`partner_id`);
2. the same attribution anchor kind and value the conversion carried
   (section 3.2);
3. the same `event_name`, resolved to the same Goal; and
4. the business identity the conversion carried, matched by the same field
   kind and value. Identity namespaces stay separate exactly as in section 3.4:
   a conversion bound to `order_id:x` is not located by `event_id:x`.

Business identity requirements follow the Goal's `dedup_strategy`:

- `dedup_strategy=first`: business identity is OPTIONAL. Anchor plus
  `event_name` identifies the single first-wins conversion. If identity is
  supplied, it MUST match an alias bound to that conversion.
- `dedup_strategy=all`: business identity is REQUIRED, because the anchor and
  Goal alone may identify several conversions. A request without identity, or
  with identity that matches no conversion, returns 404
  `REVERSAL_TARGET_NOT_FOUND`.

If no conversion matches, AON returns 404 `REVERSAL_TARGET_NOT_FOUND`. If the
supplied facts match more than one conversion, AON returns 409
`REVERSAL_TARGET_AMBIGUOUS` and reverses nothing; the Provider SHOULD resend
with the business identity used on the original conversion. A conversion that
was recorded only as `unmapped` has no settlement and cannot be reversed.

**Scope.** A reversal always reverses the whole conversion. Partial amounts
are not defined; a reversal MUST NOT carry `amount` or `currency`, and AON
does not use them.

**Window.** A conversion is reversible until the Partner confirms the monthly
statement that carries it. Before confirmation — including when the statement
has been generated but not yet confirmed — the reversal succeeds and the
conversion is removed from the unconfirmed statement. After confirmation AON
returns 409 `REVERSAL_WINDOW_CLOSED` and changes no financial fact; the
Partner MUST dispute the confirmed statement instead of sending a reversal.

**Authorization.** A reversal is authorized exactly like a conversion: the
same Partner route scope, enablement, and IP policy apply. No additional
credential or allowlist requirement is defined for reversals.

**Idempotency.** Each conversion can be reversed at most once. A successful
reversal returns 200 with `result="reversed"` and `reason="REVERSED"`. A later
reversal that locates the same, already reversed conversion returns 200 with
`result="already_reversed"` and `reason="ALREADY_REVERSED"` and has no further
effect, regardless of the `reason` value supplied. A Provider SHOULD treat
both as final.

**Effect.** A successful reversal moves the conversion's settlement to
`reversed`, withdraws the Developer earning derived from it, removes it from
conversion counts and revenue in reporting, and suppresses any Agent
`conversion` callback or downstream export that has not been sent yet. A
Developer Application that subscribes to earning-status events receives the
withdrawal as `earning_rejected`.

## 4. Provider response contract

Provider responses do not depend on a request version header. AON MAY include
an `AON-Protocol-Version: 1.0` response header; a Provider MUST NOT require
it.

The outer response uses the hosted API envelope keys `code`, `message`,
`data`, and `extra`. The response field names and result unions are defined by
`ProviderPostbackResponseV10` and `ProviderPostbackErrorExtraV10` in the
v1.0 types source linked in section 1.

### 4.1 Success and durable results

For 2xx outcomes, `data` contains exactly the Provider response data:
`result`, `event_logged`, `reason`, `correlation_id`, and `retryable`.
Stable result classes are `accepted`, `already_recorded`, `unmapped`,
`reversed`, `already_reversed`, `rejected`, and `retry`.

| Outcome | Typical HTTP result | Client meaning |
| --- | ---: | --- |
| `accepted` | 200 | The conversion was durably accepted. |
| `already_recorded` | 200 | This is a safe replay or first-wins duplicate; do not create another business conversion. |
| `result="unmapped"` with `reason="UNMAPPED_EVENT"` | 200 | The event was recorded for audit but did not match a declared Goal and was not settled. |
| `result="reversed"` with `reason="REVERSED"` | 200 | The located conversion was reversed (section 3.5). |
| `result="already_reversed"` with `reason="ALREADY_REVERSED"` | 200 | The located conversion had already been reversed; nothing changed. |
| identity conflict | 409 | The submitted aliases or immutable business facts conflict; correct the request rather than retrying it unchanged. |

### 4.2 Rejections and recoverable failures

Rejections refused before intake (for example the retired JSON lane, 410) have
`data` exactly equal to `{}`; their diagnostic fields appear only in `extra`,
which contains `result`, `reason`, `correlation_id`, `retryable`, and
`event_logged`. Other 4xx or 5xx results of the GET binding keep the same five
diagnostic fields in `data`, with `code` set to the upper-case reason.

- Parameter, method, action, scope, and identity validation failures are
  terminal 4xx results with `retryable=false`.
- A request that carries an `AON-Protocol-Version` header returns 410 with
  reason `postback_json_lane_retired` and `event_logged=false` (section 2.4).
- A pre-durable internal/configuration/context failure returns 500 with reason
  `internal_error`, `retryable=true`, and `event_logged=false`.
- Durable intake or processing unavailability returns 503 with reason
  `postback_intake_unavailable`, `retryable=true`, and `event_logged` matching
  whether the business event was durably created.

Reversal-specific outcomes:

| `reason` | HTTP | `result` | Client meaning |
| --- | ---: | --- | --- |
| `REVERSED` | 200 | `reversed` | Reversal applied. |
| `ALREADY_REVERSED` | 200 | `already_reversed` | Idempotent replay; no change. |
| `REVERSAL_TARGET_NOT_FOUND` | 404 | `rejected` | No conversion matches the anchor, Goal, and business identity. Check that the same anchor and identity field kind were used on the conversion. |
| `REVERSAL_TARGET_AMBIGUOUS` | 409 | `rejected` | More than one conversion matches; resend with the original business identity. |
| `REVERSAL_WINDOW_CLOSED` | 409 | `rejected` | The Partner already confirmed the statement carrying this conversion; dispute the statement instead. |
| `INVALID_REVERSAL_REASON` | 400 | `rejected` | `reason` is missing or not one of the values in section 3.5. |
| `invalid_action` | 400 | `rejected` | `action` is not absent, `conversion`, or `reversal` (section 2.2). |

Every response produced by reversal processing — including
`REVERSAL_TARGET_NOT_FOUND`, `REVERSAL_TARGET_AMBIGUOUS`,
`REVERSAL_WINDOW_CLOSED`, and `INVALID_REVERSAL_REASON` — reports
`event_logged=true`, because the reversal attempt is journaled for Partner
Postback Logs and health reporting. `invalid_action` is rejected before
processing and reports `event_logged=false`. All reversal 4xx outcomes are
terminal with `retryable=false`.

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

- a `conversion` event for a conversion accepted through the Provider `GET`
  binding is stored as wire `1.0`, and every attempt carries the current
  version header.

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

The exact JSON body MUST conform to the Agent payload schema linked in
section 1. Each event branch is closed on the producer side and has no payload
version field: AON emits only members this specification defines for that
`event_type` at the revision it implements; the schema's
`additionalProperties: false` states that producer contract. Receivers follow
the receiver tolerance rule below.
The conversion branch fields are listed below; earning branches are defined in
[Earning status events](#earning-status-events).

| Field | Level | Description |
| --- | --- | --- |
| `event_id` | REQUIRED | AON-generated opaque globally unique conversion-event id. |
| `event_type` | REQUIRED | Fixed value `conversion`. |
| `aon_click_id` | OPTIONAL | Opaque AON click id of the attributed click; begins with `aci_`, for example `aci_0192f3a4b5c67d8e9f0a1b2c3d4e5f60` (older clicks may carry a hyphenated form). Receivers MUST treat it as an opaque string. Omitted when the conversion has no attributed click. |
| `offer_instance_id` | OPTIONAL | AON-issued identifier of the Offer delivery this conversion is attributed to: the `offer_instance_id` returned for that Offer by Query or MCP, and the value AON substitutes for the `aon_tracking_id` destination macro. One delivery can produce several clicks and conversions (for example a multi-goal Offer), so it is a correlation key, not an idempotency key — deduplicate by `(application_id, event_id)`. For offer walls AON generates it, so the receiver does not hold it in advance. Opaque string without a format guarantee; leading and trailing whitespace removed; omitted when absent. |
| `offer_id` | REQUIRED | Converted Offer identifier. |
| `application_id` | REQUIRED | Developer Application identifier, for example `app_demoApplication1`. |
| `placement_id` | OPTIONAL | Developer placement identifier, for example `plc_demoPlacement001`. Omitted when absent. |
| `event_name` | REQUIRED | Exact value of the matched Offer Goal. |
| `amount` | REQUIRED | Developer expected net earning in USD, as a JSON string with exactly six fractional digits; see below. |
| `currency` | REQUIRED | Always `USD`. |
| `sub_id`, `sub_id_2` … `sub_id_5` | OPTIONAL | Developer attribution slots inherited from the click. |
| `timestamp` | REQUIRED | Conversion event time. |

The frozen body serializes fields in this order, omitting each optional field
when absent: `event_id`, `event_type`, `aon_click_id`, `offer_instance_id`,
`offer_id`, `application_id`, `placement_id`, `event_name`, `amount`, `currency`, `sub_id`,
`sub_id_2` … `sub_id_5`, `timestamp`.

`amount` and `currency` are always present. `amount` is the Developer's
expected net earning for this conversion: the settlement commission minus the
platform fee, converted to USD with the settlement's frozen FX rate, rounded to
6 decimal places for this webhook representation, and rendered as a canonical signed
decimal string with exactly six fractional digits (for example `"96.000000"`,
`"12.500000"`, `"0.070000"`). The exact grammar is `^-?(?:0|[1-9][0-9]*)\.[0-9]{6}(?![\s\S])`; leading zeroes, exponent notation, and a trailing line ending are invalid. Negative values preserve their sign. The underlying
conversion amount is calculated at six decimal places; a non-billable conversion
sends `"amount":"0.000000"`. `currency` is always `USD`. A
non-billable conversion without a settlement sends that same zero string. The value is
an expectation at delivery time. Risk freezes emit no event; later reversals are
reported through the independently subscribed `earning_rejected` event.

The maximum current raw body size is 2,097,152 bytes (2 MiB). A sender MUST
reject an oversized body before HTTP delivery. A receiver MUST enforce a
transport limit no greater than 2 MiB and MUST reject an oversized complete
body before HMAC calculation or JSON parsing.

Optional `aon_click_id`, `offer_instance_id`, `placement_id`, and `sub_id`
fields are omitted when absent, never synthesized as `null`.
The payload's `timestamp` is conversion event time; it is distinct from
`X-AON-Timestamp`, which is attempt signing time. Payload event time is an
RFC3339 `date-time` normalized to UTC (`Z`, `z`, or `+00:00`); non-UTC offsets
and timezone-free values are invalid.

Receiver tolerance: each event branch is closed on the producer side only. A
receiver MUST ignore any top-level member it does not recognize for the
event's `event_type` (including a member defined only for another event type)
and MUST NOT reject the event for it. AON MAY add OPTIONAL top-level members to
any event type within protocol line 1.0 without changing
`AON-Protocol-Version`; each addition is recorded in section 12 with an RFC and
takes a fixed position in the frozen field order. Within 1.0, AON does not
remove or rename a REQUIRED member, change the type or meaning of an existing
member, or add a REQUIRED member, except by an in-place revision that an RFC
records and section 12 flags as BREAKING. A receiver that validates with the
published schema MUST NOT use it unmodified to reject events; it MUST first
reduce the body to recognized members as follows:

1. read `event_type` and select the schema branch whose `event_type` `const`
   equals it — if none matches, the event type is unknown and is handled by the
   receiver's event-type rule, not by this rule;
2. remove every top-level member that is not a key of that branch's
   `properties`; and
3. validate the reduced object against that branch.

The signature and idempotency always use the original raw bytes. New
`event_type` values are outside this rule; they are delivered only after the
receiver subscribes to them.

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
5. only after HMAC succeeds, parse JSON and validate the recognized members
   of the Agent payload for its `event_type`, ignoring unrecognized top-level
   members (section 5.3); and
6. apply the durable idempotency transaction in section 7 before any business
   side effect.

An explicitly local/test-only insecure mode MAY skip timestamp, key, and HMAC
checks, but it MUST still enforce version, size, schema, and idempotency rules.
Such a mode MUST fail closed if enabled in production.

## 7. Agent receiver idempotency

A receiver MUST retain a durable record keyed by `(application_id, event_id)` with a
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
Each environment MUST keep Agent current delivery disabled until its writer,
schema, signing, retry, idempotency, and callback evidence is complete. The
deployment owner must certify the Provider `GET` binding and parameter table,
reversal location and window rules, retired-lane `410` rejection, Agent
selector/echo behavior, producer-closed Agent payloads (section 5.3), five-attempt retry ladder,
frozen raw body, signature vectors, and replay rules before claiming v1.0
support. Until certification, the surface is `unvalidated`.

A deployment MUST NOT publish the reversal binding (section 3.5) to Providers
before it implements `action` handling; a deployment that ignores unknown
query parameters would otherwise process a reversal request as a conversion.

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

- `v1.0/http/postback/agent/basic-conversion.http` — signed Agent first success;
- `v1.0/http/postback/agent/invalid-duplicate-signature.http` — duplicate
  security-header rejection; and
- `v1.0/http/postback/agent/retry-scenario.http` — frozen-body retry sequence.

The Provider examples `v1.0/http/postback/partner/basic-conversion.http` and
`v1.0/http/postback/partner/invalid-unknown-field.http` describe the retired
JSON body lane (section 2.4). They are retained for history only and are not
current Provider inputs; such requests now receive `410
postback_json_lane_retired`.

Canonical Provider `GET` requests:

```http
### Conversion — expected 200, result=accepted, reason=null
GET https://aon-deployment.example/v1/postback/019f5f65-dbc1-7c3c-9ef9-9d6d2d8ae001?aon_click_id=aci_0192f3a4b5c67d8e9f0a1b2c3d4e5f60&event_name=subscription&order_id=ord_10001&amount=120.00&currency=USD HTTP/1.1

### Reversal — expected 200, result=reversed, reason=REVERSED
GET https://aon-deployment.example/v1/postback/019f5f65-dbc1-7c3c-9ef9-9d6d2d8ae001?aon_click_id=aci_0192f3a4b5c67d8e9f0a1b2c3d4e5f60&event_name=subscription&order_id=ord_10001&action=reversal&reason=refunded HTTP/1.1

### Same reversal replayed — expected 200, result=already_reversed, reason=ALREADY_REVERSED

### Reversal with a different identity kind (event_id instead of order_id)
### Expected 404, data={}, extra.reason=REVERSAL_TARGET_NOT_FOUND
GET https://aon-deployment.example/v1/postback/019f5f65-dbc1-7c3c-9ef9-9d6d2d8ae001?aon_click_id=aci_0192f3a4b5c67d8e9f0a1b2c3d4e5f60&event_name=subscription&event_id=ord_10001&action=reversal&reason=refunded HTTP/1.1

### Retired JSON lane — expected 410, data={}, extra.reason=postback_json_lane_retired
POST https://aon-deployment.example/v1/postback/019f5f65-dbc1-7c3c-9ef9-9d6d2d8ae001 HTTP/1.1
Content-Type: application/json
AON-Protocol-Version: 1.0

{"event_name":"subscription","aon_click_id":"aci_0192f3a4b5c67d8e9f0a1b2c3d4e5f60"}
```

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


## Earning status events

The current Agent payload schema uses five closed `oneOf` branches (producer
contract; see section 5.3), selected by `event_type`: `conversion`, `earning_confirmed`, `earning_available`,
`earning_paid_out`, and `earning_rejected`. All five branches represent `amount`
as a signed USD decimal string with exactly six fractional digits. This in-place
v1.0 extension changes the signed bytes wherever the former numeric amount was
used; receivers must validate the amount string before applying an event. All
events use the same endpoint, headers, raw-body HMAC, 2 MiB limit, durable
idempotency rule and five-attempt retry ladder.

| Event | Meaning |
| --- | --- |
| `earning_confirmed` | The associated Partner statement has been confirmed. |
| `earning_available` | The earning has become available for payout. |
| `earning_paid_out` | AON has paid out this earning in full; this does not certify receipt in the developer's bank account. This state is terminal: AON does not register further earning status events for it. |
| `earning_rejected` | The earning was rejected by the merchant, reversed, or marked refunded in a review. |

Earning bodies have this exact field order. Optional fields are omitted, never null:

```text
event_id, event_type, earning_id, sequence, conversion_event_id,
offer_instance_id?, offer_id, application_id, placement_id?, amount, currency,
reason?, sub_id?, sub_id_2?, sub_id_3?, sub_id_4?, sub_id_5?, timestamp
```

| Field | Meaning |
| --- | --- |
| `event_id` | `earning:` followed by the immutable outbox UUID; unchanged on retry. |
| `event_type` | One of the four earning event names above. |
| `earning_id` | Lowercase hyphenated settlement UUID, identical to the Earnings API id. |
| `sequence` | Positive JSON integer increasing within this earning; gaps are allowed. |
| `conversion_event_id` | The originating conversion's event id, even if its webhook was never delivered. |
| `offer_instance_id` | Optional. The originating conversion's `offer_instance_id`, rendered by the conversion-event rule; every earning event of that conversion carries the same value. Not unique. |
| `offer_id` | The Earnings API Offer identity; canonical resolution may differ from the conversion's Offer id. Join events using `conversion_event_id`. |
| `application_id` | Developer Application id. |
| `placement_id` | Optional placement identity. |
| `amount` | Net earning in USD, including review adjustments, as a signed decimal string with exactly six fractional digits; rejection carries the last observed live amount, or the original amount when none was observed. |
| `currency` | Always `USD`. |
| `reason` | Required only for `earning_rejected`; forbidden on the other earning branches. |
| `sub_id`, `sub_id_2` … `sub_id_5` | Optional. The originating conversion's attribution slots, rendered by the same rule as the conversion event (leading and trailing whitespace removed; omitted when empty, never null). Present on every earning branch, including `earning_rejected`, even when the conversion event was never delivered. |
| `timestamp` | Business time of this state, RFC3339 seconds in UTC (`...Z`); not a sorting key. |

Earning amounts retain their existing cent precision and are rendered with six fractional digits (for example `"5.000000"` or `"4.200000"`).

The `reason` closed set is `merchant_cancelled`, `duplicate`, `invalid_order`,
`fraud`, `returned`, `reversed`, or `refund`; operator free text is never sent.
Conversion-only fields (`event_name`, `aon_click_id`) are forbidden in earning bodies.

Use `(application_id, event_id)` for deduplication. Track the largest `sequence`
per `earning_id` for the latest state; delivery can be duplicated or arrive out
of order, and timestamps need not increase. Do not add the amounts of these
snapshots together. Amounts can differ between events because of review
adjustments or USD rounding (even by one cent); use the amount of the earning
event with the largest sequence. If an adjustment and rejection occur between
two observations, the rejection may carry the amount before the adjustment.
In rare cases a rejection is withdrawn and the earning continues.

Receivers that credit their own end users by `sub_id` should key credits by
`earning_id`: record the amount already credited for that earning, and on
`earning_rejected` reverse only the amount already credited. A rejection can
arrive for an earning that was never credited, or whose conversion event was
never received; then nothing is reversed. If a later event with a larger
`sequence` returns the earning to a non-rejected state, credit it again. When
the conversion carried no `sub_id`, the slots are absent; the slots are only
populated when the developer passes `sub_id` values on the click link.

Each event has an independent subscription. The first earning subscription
establishes a baseline at the configuration transaction's commit time: existing
states are not replayed. Reopening subscriptions after all four were off also
establishes a baseline; changes while all were off are not backfilled. While any
earning event is enabled, all earning status changes are observed, but only
currently subscribed events are registered. Opening another switch does not
rebuild that baseline. Disabling a switch stops new event registration; already
registered events continue delivery. Skipped registrations may consume sequence
numbers. A risk freeze emits no earning event. Available may return to confirmed;
unsubscribed intermediate states are not guaranteed to arrive.

## 12. Changelog

| Version | Date | Changes |
| --- | --- | --- |
| 1.0 | 2026-10-15 | In place (RFC-0020): conversion and earning bodies carry optional `offer_instance_id` immediately before `offer_id`; receivers MUST ignore unrecognized top-level members and AON MAY add optional members within 1.0; BREAKING for closed receivers that predate this revision (for example AON SDK ≤ 1.0.3); bodies without the value are byte-identical. |
| 1.0 | 2026-10-13 | BREAKING, in place (RFC-0017): earning bodies carry optional `sub_id` … `sub_id_5` between `reason?` and `timestamp`, copied from the originating conversion by the conversion-event rule; closed receivers must accept them; earning bodies without `sub_id` are byte-identical. |
| 1.0 | 2026-10-12 | Current v1.0 Agent conversion and earning webhook `amount` values are signed USD JSON strings with exactly six fractional digits; signed bytes change where the prior numeric amount was used. |
| 1.0 | 2026-10-08 | Provider intake revised in place (WS-54): `GET /v1/postback/{partner_id}` with query parameters is the only Provider binding, defined by the authoritative parameter table (section 2.3) with `aon_click_id` as the recommended anchor; the JSON body lane selected by `AON-Protocol-Version` (`0.2`/`0.3`/`1.0`) is retired with `410 postback_json_lane_retired` and the Provider JSON Schema is deprecated. Added Goal conversion reversal (`action=reversal` + `reason`): location by anchor + `event_name` + same-kind business identity (required for `dedup_strategy=all`), whole-conversion only, reversible until the Partner confirms the statement (`REVERSAL_WINDOW_CLOSED` after), same IP policy as conversions, once per conversion; new results `reversed`/`already_reversed` and reasons `REVERSED`, `ALREADY_REVERSED`, `REVERSAL_TARGET_NOT_FOUND`, `REVERSAL_TARGET_AMBIGUOUS`, `REVERSAL_WINDOW_CLOSED`, `INVALID_REVERSAL_REASON`, `invalid_action`. Developers learn of a reversal through the earning-status `earning_rejected` event (RFC-0014). |
| 1.0 | 2026-10-01 | Agent conversion webhook payload revised in place: `aon_tracking_id` replaced by optional `aon_click_id` (`aci_` click id), `agent_id` replaced by `application_id`, optional `placement_id` added, `amount` redefined as Developer expected net earning in USD with `currency` always `USD` and both always present, field order pinned, and receiver idempotency keyed by `(application_id, event_id)`. |
| 1.0 | 2026-08-29 | Promoted the bidirectional canonical contract with exact `1.0` selection while preserving closed Provider JSON, shared Goal/identity/revenue semantics, event-level Agent wire version, four required security/control headers, exact-byte HMAC verification, 2 MiB bound, keyring/time/idempotency rules, and the fixed retry behavior. |
