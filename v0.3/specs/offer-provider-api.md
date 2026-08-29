# OfferProvider API

> **Current stable contract**

Provider request reuses the Query request shape and requires `request_id`.
Provider success returns Partner supply Offers; AON resolves source identity,
evaluates Partner-only rules, and creates the public Query response. Provider
errors use the existing uppercase `code`, `message`, `data`, `extra` envelope (`BAD_REQUEST`,
`UNAUTHORIZED`, `FORBIDDEN`, `RATE_LIMITED`, `INTERNAL_ERROR`). HMAC, nonce and
transport metadata remain headers, not public body fields.

## Version selection

A Partner with `credentials.protocol_version: "0.3"` receives
`AON-Protocol-Version: 0.3`. The Partner response must use the same contract.
Existing explicit v0.2 profiles keep their current behavior; AON does not
silently downgrade an attempted dispatch.

## Request and authentication

The request body is exactly
[`offer-provider-request.json`](https://github.com/agentoffernetwork/schema/blob/main/v0.3/json-schema/offer-provider-request.json).
It shares all Query business fields, including current-turn intent,
bounded context, structured signals, constraints, `force_offer`, and
`response_options.thinking_mode`; `request_id` is required only for the
Provider channel.

The request uses `POST` and JSON. `AON-Protocol-Version: 0.3` is required for
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
| `AON-Protocol-Version` | Required | `0.3` |
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
[`offer-provider-response.json`](https://github.com/agentoffernetwork/schema/blob/main/v0.3/json-schema/offer-provider-response.json):

- **Success:** the Partner supply envelope with an exactly matching
  `request_id`, `protocol_version: "0.3"`, `language`, and `offers` from
  `offer-partner-schema-v0.3.json`.
- **Error:** `{code,message,data,extra}` with a closed uppercase error code.

A success response is not wrapped in the hosted API envelope. Each Partner
Offer requires stable `source_offer_id` in the identity namespace configured
for that integration. It must not contain AON-owned `offer_id`,
`offer_instance_id`, or `match_reason`. AON resolves
`owner Partner + identity namespace + source_offer_id` to canonical `offer_id`,
evaluates `targeting` and `conversion_rule`, and then creates a public Offer
with a fresh dispatch identity and any permitted match explanation.

`entity`, optional `listing_source`, and `action` retain their separate Offer
meanings. Provider-private freshness, supply lineage, affiliate, or mapping data
other than the declared source identity must not appear in the response.

Provider adapters may use private freshness, mapping and supply lineage data
internally, but those fields must not leak into the Partner supply payload or
the later public response. The public Offer projection is produced by AON and
uses the Query response shape, including the semantic separation of `entity`,
`listing_source` and `action`.

When a Provider sends `listing_source.logo`, it must be an explicit absolute
HTTPS URI no longer than 2048 characters; non-ASCII components must be
percent-encoded. Invalid, relative, HTTP, non-string,
or oversized Logo values reject the complete Provider response before adapter
processing; the adapter must not strip the value and continue. Providers must
not infer the field from entity, action, or material data.

## Implementation vectors

- [HMAC signing vectors](https://github.com/agentoffernetwork/examples/blob/main/v0.3/http/offer-provider/hmac-signing-cases.md)
- [Provider request example](https://github.com/agentoffernetwork/examples/blob/main/v0.3/http/offer-provider/request.json)
- [Provider success example](https://github.com/agentoffernetwork/examples/blob/main/v0.3/http/offer-provider/success.json)
- [Provider request schema](https://github.com/agentoffernetwork/schema/blob/main/v0.3/json-schema/offer-provider-request.json)
- [Provider response schema](https://github.com/agentoffernetwork/schema/blob/main/v0.3/json-schema/offer-provider-response.json)
