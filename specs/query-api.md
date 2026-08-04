# Offer Query API v0.2

**Contract status:** current normative source contract

**Runtime implementation status: BL-034 delivers the hosted Query boundary;
public lifecycle promotion remains governed by BL-038 release and drift gates.**

The Agent-facing Query surface discovers canonical Offer v0.2 objects through
the unchanged hosted API shell:

```text
POST /v1/offers/query
Content-Type: application/json
AON-Protocol-Version: 0.2
Authorization: Bearer <token>
```

`/v1/` identifies the HTTP API major. `AON-Protocol-Version` selects the
protocol payload contract; these version signals are intentionally independent.

<a id="protocol-version"></a>

## Version selection

| Request header | Selected contract | Success response header |
|---|---|---|
| omitted | v0.1 compatibility | unchanged legacy response |
| `0.2` | v0.2 | `AON-Protocol-Version: 0.2` |
| `0.1` | v0.1 compatibility | unchanged legacy response |
| unknown or invalid | none; fail closed | none |

New integrations MUST send `AON-Protocol-Version: 0.2`; v0.2 has no legacy
field fallback. Omitted and explicit `0.1` requests remain a compatibility
lane for existing integrations and are not examples for new integrations.
BL-034 owns actual HTTP negotiation, response echo, and runtime support.

## Contract authority

The request structure is defined by
[`offer-query-schema-v0.2.json`](../../schema/json-schema/offer-query-schema-v0.2.json).
The protocol success payload is defined by
[`offer-query-response-v0.2.json`](../../schema/json-schema/offer-query-response-v0.2.json),
whose `offers[]` items reference
[`offer-schema-v0.2.json`](../../schema/json-schema/offer-schema-v0.2.json).

Protocol objects are closed: fields not declared by the relevant schema are
rejected. Required properties are not nullable unless a schema explicitly says
otherwise.

## Request

### Minimal request

```json
{
  "context": {
    "user_profile": {}
  },
  "intent": {
    "content": [
      {
        "type": "input_text",
        "text": "Find a secure team workspace"
      }
    ]
  }
}
```

The canonical file is
[`offer-query-request-v0.2-minimal.json`](../../examples/http/offer-query-request-v0.2-minimal.json).

### Full request

The complete canonical request is
[`offer-query-request-v0.2-full.json`](../../examples/http/offer-query-request-v0.2-full.json).

### Top-level fields

| Field | Level | Meaning |
|---|---|---|
| `request_id` | OPTIONAL | UUID correlation id. Hosted service generates one when omitted. |
| `timestamp` | OPTIONAL | RFC 3339 request time. |
| `test_mode` | OPTIONAL | Test traffic marker; default is false. |
| `placement_id` | OPTIONAL | Provider-neutral hosted placement id. |
| `context` | REQUIRED | Requester and viewer context. |
| `intent` | REQUIRED | One or more discriminated content items. |
| `constraints` | OPTIONAL | Deterministic eligibility constraints. |
| `pagination` | OPTIONAL | Offset/limit control. |

<a id="location-context"></a>

### Context

`context.user_profile` is required but may be `{}`. Its optional v0.2 fields
are:

- `user_pseudo_id`
- `language`
- `location_ids[]`
- `verified_age_over[]`
- `interests[]`
- `device_info`

`device_info` itself is optional. Its `device_type` and `os` are also optional;
omitted or `other` values both mean unknown viewer context. Known device values
are `desktop`, `mobile`, `tablet`, and `smart_tv`. Known Query OS values are
`ios`, `android`, `windows`, and `macos`; a producer maps Linux or other
unlisted systems to `other`.

The legacy `context.user_profile.country` string is not a v0.2 field. Location
context uses AON Location Registry `location_ids`.

`language` should be a BCP 47 tag. A non-empty but unparseable value remains
structurally reachable so the targeting policy can treat it as unknown rather
than silently normalize it.

### Intent

`intent.content` is non-empty. Every item is exactly one closed branch:

```json
{"type": "input_text", "text": "Find a hotel"}
```

or:

```json
{"type": "input_image", "image_url": "https://example.com/photo.png"}
```

Text items require `text` and forbid `image_url`. Image items require
`image_url` and forbid `text`.

<a id="taxonomy-constraints"></a>
<a id="pagination"></a>

### Constraints and pagination

`constraints.category_ids[]` contains AON Taxonomy v1 ids. Values are ORed and
match the selected node or any descendant against an Offer's primary and
secondary categories. Registry membership and subtree evaluation are semantic
checks owned by BL-035 and BL-034.

`pagination.limit` defaults to 20 and is inclusive `1..100`.
`pagination.offset` defaults to 0 and is non-negative.

<a id="unknown-targeting-context"></a>

## Targeting context semantics

The Query schema only carries viewer context. Matching against
`offers[].targeting[]` follows the Offer v0.2 truth table:

- rules are ORed; active dimensions inside one rule are ANDed;
- empty or omitted dimensions do not restrict;
- geo exclude wins, and active geo fails closed when location context is
  missing, unknown, or insufficient;
- missing verified age and unknown language/device/OS are compatibility-pass
  states;
- targeting is an offer-selection rule, not a legal-compliance age gate.

BL-033 records these outcomes as downstream vectors. BL-034 executes matching.

<a id="response-payload"></a>

## Success payload

The protocol payload root is exactly:

```json
{
  "request_id": "0195af51-9f4c-7e2d-b3a1-d5e6f7081234",
  "offers": []
}
```

`offers` may be empty. A non-empty entry must validate as a complete canonical
Offer v0.2.

<a id="hosted-wrapper"></a>

A hosted service may return:

```json
{
  "code": "SUCCESS",
  "message": "",
  "data": {
    "request_id": "0195af51-9f4c-7e2d-b3a1-d5e6f7081234",
    "offers": []
  },
  "extra": {}
}
```

The outer `{code,message,data,extra}` object is an HTTP service wrapper, not
part of the protocol success schema. It must not validate directly against
`offer-query-response-v0.2.json`.

`X-AON-TRACE-ID` may be used as hosted diagnostic response metadata. It is not
a JSON payload field.

## Errors and runtime boundary

Hosted authentication, rate limits, HTTP status codes, and error envelopes are
runtime transport concerns. They do not change the v0.2 Query request or
success payload schema. Runtime behavior remains unavailable until the
downstream implementation and release gates pass.

## References

- [Full Query example](../../examples/http/offer-query-request-v0.2-full.json)
- [Hosted response example](../../examples/http/offer-query-hosted-response-v0.2.json)
- [Offer v0.2](./offer-schema-v0.2.md)
- [OfferProvider v0.2](./offer-provider-api.md)
- [Contract Governance](./contract-governance.md)
