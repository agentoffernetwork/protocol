# OfferProvider API

> **Current stable contract**

Provider request reuses the Query request shape and requires `request_id`.
Provider success returns Partner supply Offers; AON resolves source identity,
evaluates Partner-only rules, and creates the public Query response. Provider
errors use the existing uppercase `code`, `message`, `data`, `extra` envelope (`BAD_REQUEST`,
`UNAUTHORIZED`, `FORBIDDEN`, `RATE_LIMITED`, `INTERNAL_ERROR`). HMAC, nonce and
transport metadata remain headers, not public body fields.

## Version selection

A Partner with `credentials.protocol_version: "1.0"` receives
`AON-Protocol-Version: 1.0`. The Partner response must use the same contract.
Missing or different selectors fail closed; AON does not silently downgrade an
attempted dispatch.

## Request and authentication

The request body is exactly
[`offer-provider-request.json`](https://github.com/agentoffernetwork/schema/blob/main/v1.0/json-schema/offer-provider-request.json).
It shares all Query business fields, including current-turn intent,
bounded context, structured signals, constraints, `force_offer`, and
`response_options.thinking_mode`; `request_id` is required only for the
Provider channel.

The request uses `POST` and JSON. `AON-Protocol-Version: 1.0` is required for
an explicit Provider profile. HMAC key id, timestamp, nonce, signature,
and optional `X-AON-Test: true` remain transport headers. The signature covers
the exact UTF-8 JSON body sent over the wire; Providers must not reserialize
the body before verification.

### Authentication

| Header | Requirement | Meaning |
|---|---|---|
| `X-AON-Key` | Required | Partner-issued key id identifying AON |
| `X-AON-Timestamp` | Required | Unix epoch seconds as ASCII decimal |
| `X-AON-Nonce` | Required | Unique request nonce; UUIDv4 recommended |
| `X-AON-Signature` | Required | Lowercase hexadecimal HMAC-SHA256 |
| `AON-Protocol-Version` | Required | `1.0` |
| `X-AON-Test` | Optional | Authoritative test-mode signal when `true` |

The signing input is exactly:

```text
METHOD + "\n" +
PATH + "\n" +
BODY + "\n" +
TIMESTAMP + "\n" +
NONCE
```

- `METHOD` is uppercase ASCII and is `POST` for this contract.
- `PATH` is the request path without host or query.
- `BODY` is the exact UTF-8 JSON body transmitted on the wire.
- `TIMESTAMP` and `NONCE` exactly match their request headers.
- There is no trailing newline.

The Provider selects the secret identified by `X-AON-Key`, computes
`HMAC-SHA256(secret, signing_input)`, encodes lowercase hex, and compares in
constant time. It rejects timestamps outside ±300 seconds. A Provider should
retain `(key, nonce)` for five minutes and reject replay; a declared stateless
integration may omit nonce storage but must still enforce timestamp and HMAC.

`X-AON-Test: true` is authoritative when present. If it disagrees with
`body.test_mode`, the header wins. Test requests must not create production
billing or settlement effects.

## Response

The response is exactly one branch of
[`offer-provider-response.json`](https://github.com/agentoffernetwork/schema/blob/main/v1.0/json-schema/offer-provider-response.json):

- **Success:** the Partner supply envelope with an exactly matching
  `request_id`, `protocol_version: "1.0"`, `language`, and `offers` from
  `offer-partner-schema-v1.0.json`.
- **Error:** `{code,message,data,extra}` with a closed uppercase error code.

A success response is not wrapped in the hosted API envelope. Each Partner
Offer requires stable `source_offer_id` in the identity namespace configured
for that integration. It must not contain AON-owned `offer_id`,
`offer_instance_id`, `match_reason`, or
`offer_info.commercial.display_price`. The display-price field is owned by the
AON Query response projection, not the Provider supply carrier; its presence
causes the complete Provider response to fail structural validation. AON resolves
`owner Partner + identity namespace + source_offer_id` to canonical `offer_id`,
evaluates `targeting` and `conversion_rule`, and then creates a public Offer
with a fresh dispatch identity and any permitted match explanation.

`entity`, optional `listing_source`, and `action` retain their separate Offer
meanings. Provider-private freshness, supply lineage, affiliate, or mapping data
other than the declared source identity must not appear in the response.

### Registered supply profiles

The Partner Offer in a success response may carry an optional closed
`offer_info.details` envelope from the v1.0 Supply Offer Profile Registry. The
only registered names are `flight` and `hotel_rate`. Profile facts, observed
`commercial.price.tax_status`, and `commercial.quote` are accepted on this
supply carrier when they meet the Offer semantic rules. A later Generic Hosted Query or MCP response may retain these registered
profile facts, including for a non-real-time request, subject to the same
profile validation. Details alone do not establish live execution. Typed Flight Query has its
own required detail projection described below.

Flight Providers send endpoint `local_at` values in each airport's local clock
using `YYYY-MM-DDTHH:mm:ss`, without offsets, and include positive
source-provided `duration_minutes` for every segment. A Provider must not infer
airport timezone data solely to construct the payload.

When a Hotel Provider offers only a Ctrip-style one-night starting price, it
uses `hotel_rate` with `reference_starting_nightly` and omits `stay` and `room`
entirely. The `book` action remains a source jump and is not a guarantee of
dates, room type, inventory, or a total price.

Provider adapters may use private freshness, mapping and supply lineage data
internally, but those fields must not leak into the Partner supply payload or
the later public response. The public Offer projection is produced by AON and
uses the Query response shape, including the semantic separation of `entity`,
`listing_source` and `action`.

AON may derive one response-scoped `commercial.display_price` only after it has
accepted the Provider supply Offer. The Provider does not select the target
currency and does not submit the converted amount.

When a Provider sends `listing_source.logo`, it must be an explicit absolute
HTTPS URI no longer than 2048 characters; non-ASCII components must be
percent-encoded. Invalid, relative, HTTP, non-string,
or oversized Logo values reject the complete Provider response before adapter
processing; the adapter must not strip the value and continue. Providers must
not infer the field from entity, action, or material data.

## Typed Flight Provider queries

Provider requests reuse the [typed Flight Query](query-api.md#typed-flight-query)
request and its exact hard constraints. Typed Provider success MUST carry
closed `flight_search` execution metadata (`query_kind`, `status` with
`complete` or `partial`, and RFC3339 batch collection `fetched_at`) and only Flight Partner Offers with
explicit matching price_basis. Generic requests MUST NOT carry flight_search
in responses. A Provider aggregating upstreams may return partial only if at
least one source completes; a sole upstream failure requires the error envelope.
An outer Query retains Provider execution status internally; its public
success response does not expose `flight_search` or replacement metadata.
`complete` means every selected capable source completed; `partial` requires
at least one completed source and at least one failure, timeout or unusable
candidate source. `query_kind` must match the request. `fetched_at` records
batch collection completion, not quote observation or expiry.
Provider request_id is mandatory and must equal the response request_id.

The complete paired request is required for semantic validation, including
empty results; the shared city evidence, traveler composition, calendar date,
cabin, connection and nonstop rules apply. Provider never returns empty_reason,
alternative_offers, public offer_id/offer_instance_id or display_price. Supply
source_offer_id identity remains unchanged. The same uppercase Flight error
code/kind mapping and producer responsibility apply; errors are not success
payloads. See [Flight price and itinerary facts](offer-field-semantics.md#flight-price-and-itinerary-facts)
for conditional travelers/quote, durations and stop evidence. Protocol support
is not certification that any deployed Provider implements real-time Flight.

## Implementation vectors

- [HMAC signing vectors](https://github.com/agentoffernetwork/examples/blob/main/v1.0/http/offer-provider/hmac-signing-cases.md)
- [Provider request example](https://github.com/agentoffernetwork/examples/blob/main/v1.0/http/offer-provider/request.json)
- [Provider success example](https://github.com/agentoffernetwork/examples/blob/main/v1.0/http/offer-provider/success.json)
- [Provider request schema](https://github.com/agentoffernetwork/schema/blob/main/v1.0/json-schema/offer-provider-request.json)
- [Provider response schema](https://github.com/agentoffernetwork/schema/blob/main/v1.0/json-schema/offer-provider-response.json)
