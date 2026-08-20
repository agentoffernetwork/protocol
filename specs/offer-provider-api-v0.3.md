# OfferProvider API v0.3

> **Canonical v0.3 contract**

Provider request reuses `offer-query/v0.3` and requires `request_id`. Provider
success reuses `offer-query-response/v0.3`; Provider errors use the existing
uppercase `code`, `message`, `data`, `extra` envelope (`BAD_REQUEST`,
`UNAUTHORIZED`, `FORBIDDEN`, `RATE_LIMITED`, `INTERNAL_ERROR`). HMAC, nonce and
transport metadata remain headers, not public body fields.

## Version selection

A Partner with `credentials.protocol_version: "0.3"` receives
`AON-Protocol-Version: 0.3`. The Partner response must use the same v0.3
contract. Existing explicit v0.2 profiles stay on v0.2; unconfigured Partner
profiles keep their existing legacy behavior. AON does not silently downgrade
an attempted v0.3 dispatch.

## Request and authentication

The request body is exactly
[`offer-provider-request-v0.3.json`](https://github.com/agentoffernetwork/schema/blob/main/json-schema/offer-provider-request-v0.3.json).
It shares all Query v0.3 business fields, including current-turn intent,
bounded context, structured signals, constraints, `force_offer`, and
`response_options.thinking_mode`; `request_id` is required only for the
Provider channel.

The request uses `POST` and JSON. `AON-Protocol-Version: 0.3` is required for
an explicit v0.3 Provider profile. HMAC key id, timestamp, nonce, signature,
and optional `X-AON-Test: true` remain transport headers. The signature covers
the exact UTF-8 JSON body sent over the wire; Providers must not reserialize
the body before verification.

## Response

The response is exactly one branch of
[`offer-provider-response-v0.3.json`](https://github.com/agentoffernetwork/schema/blob/main/json-schema/offer-provider-response-v0.3.json):

- **Success:** the raw Query v0.3 response with an exactly matching
  `request_id`, `protocol_version: "0.3"`, `language`, and `offers`.
- **Error:** `{code,message,data,extra}` with a closed uppercase error code.

A success response is not wrapped in the hosted API envelope. `entity`,
optional `listing_source`, and `action` retain their separate Offer v0.3
meanings. Provider-private eligibility, freshness, supply lineage, affiliate,
or mapping data must not appear in the response.

Provider adapters may use private eligibility, freshness, mapping and supply
lineage data internally, but those fields must not leak into the public response.
The public Offer projection is the same shape as Query, including the semantic
separation of `entity`, `listing_source` and `action`.
