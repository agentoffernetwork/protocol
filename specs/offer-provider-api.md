# OfferProvider API v0.2

**Contract status:** current normative source contract

**Runtime support:** `not_available`; BL-034 owns dispatch implementation

OfferProvider is the AON-to-Partner discovery channel. Each Partner registers a
base URL, and AON sends:

```text
POST {base_url}/v1/offers/query
Content-Type: application/json
AON-Protocol-Version: 0.2
```

The HTTP `/v1/` shell is unchanged. A Partner configured with the v0.2
OfferProvider profile receives `AON-Protocol-Version: 0.2` and must return the
canonical v0.2 response body with its matching `request_id`. Existing Partner
adapters without that explicit profile remain in the legacy compatibility lane;
AON does not probe v0.2 and silently retry legacy. New Partner integrations
MUST use the v0.2 profile. Unknown or invalid configured versions fail closed
before dispatch.

## Shared Query core

The Provider request schema is
[`offer-provider-request-v0.2.json`](https://github.com/agentoffernetwork/schema/blob/main/json-schema/offer-provider-request-v0.2.json).
It composes
[`offer-query-schema-v0.2.json`](https://github.com/agentoffernetwork/schema/blob/main/json-schema/offer-query-schema-v0.2.json)
by reference and only raises `request_id` from optional to required. It does
not copy or redefine the Query fields.

| Concern | Agent Query | OfferProvider |
|---|---|---|
| Shared body fields | `timestamp`, `test_mode`, `placement_id`, `context`, `intent`, `constraints`, `pagination` | identical |
| `request_id` | optional | required |
| Success payload | `{request_id,offers[]}`, possibly inside hosted `data` | raw body `{request_id,offers[]}` |
| Auth/test transport | hosted Bearer/service rules | HMAC headers and `X-AON-Test` |
| Errors | hosted transport contract | Provider error envelope |

No other Provider-only body field is permitted.

## Authentication

AON authenticates to the Partner with HMAC-SHA256.

| Header | Requirement | Meaning |
|---|---|---|
| `X-AON-Key` | REQUIRED | Partner-issued key id identifying AON. |
| `X-AON-Timestamp` | REQUIRED | Unix epoch seconds as ASCII decimal. |
| `X-AON-Nonce` | REQUIRED | Unique request nonce; UUIDv4 recommended. |
| `X-AON-Signature` | REQUIRED | Lowercase hexadecimal HMAC-SHA256. |
| `AON-Protocol-Version` | REQUIRED for explicit dispatch | `0.2`. |
| `X-AON-Test` | OPTIONAL | Authoritative test-mode signal when `true`. |

The signing input is:

```text
METHOD + "\n" +
PATH + "\n" +
CANONICAL_BODY + "\n" +
TIMESTAMP + "\n" +
NONCE
```

- `METHOD` is uppercase ASCII.
- `PATH` is the request path without host or query.
- For JSON POST, `CANONICAL_BODY` is the exact UTF-8 body transmitted. The
  receiver uses raw bytes and must not reserialize or reorder JSON.
- `TIMESTAMP` and `NONCE` exactly match their headers.
- There is no trailing newline.

The Partner selects the secret identified by `X-AON-Key`, computes
`HMAC-SHA256(secret, signing_input)`, encodes lowercase hex, and compares in
constant time. It rejects timestamps outside ±300 seconds. A Partner should
retain `(key, nonce)` for five minutes and reject replay; a declared stateless
integration may omit nonce storage but must still enforce timestamp and HMAC.

## Test mode

`X-AON-Test: true` is authoritative. `body.test_mode: true` provides shared
body parity. If the header and body disagree, the header wins without changing
the shared body schema. Test requests must not create production billing or
settlement effects.

<a id="request-id"></a>

## Request

The minimal body is the Query minimal body plus required `request_id`:

```json
{
  "request_id": "0195af51-8b2c-7d3e-a1b2-c3d4e5f60718",
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

All value domains, object closure, intent discrimination, taxonomy constraints,
and pagination rules are inherited from Query v0.2.

## Response

[`offer-provider-response-v0.2.json`](https://github.com/agentoffernetwork/schema/blob/main/json-schema/offer-provider-response-v0.2.json)
defines exactly two closed branches.

### Success

```json
{
  "request_id": "0195af51-8b2c-7d3e-a1b2-c3d4e5f60718",
  "offers": []
}
```

This is the same protocol success payload as Query. Each entry in `offers[]`
references `offer-schema-v0.2.json`. The raw Provider body must not use the
hosted `{code,message,data,extra}` success wrapper.

### Error

```json
{
  "code": "BAD_REQUEST",
  "message": "request payload is invalid",
  "data": {},
  "extra": {}
}
```

Error codes are `BAD_REQUEST`, `UNAUTHORIZED`, `FORBIDDEN`, `RATE_LIMITED`, or
`INTERNAL_ERROR`. Errors are a channel-specific branch and are never mixed
with the success payload.

## Correlation and transport

The required body `request_id` is the supply-chain correlation identity.
`X-AON-Request-Id` may mirror it for infrastructure logging, but it is not a
body field. Provider health endpoints, onboarding procedures, timeout policy,
and adapter transformation DSLs are transport/runtime concerns and do not
extend the shared request core.

## References

- [`basic-query-v0.2.json`](https://github.com/agentoffernetwork/examples/blob/main/http/offer-provider/basic-query-v0.2.json)
- [`full-query-v0.2.json`](https://github.com/agentoffernetwork/examples/blob/main/http/offer-provider/full-query-v0.2.json)
- [`success-v0.2.json`](https://github.com/agentoffernetwork/examples/blob/main/http/offer-provider/success-v0.2.json)
- [`error-bad-request-v0.2.json`](https://github.com/agentoffernetwork/examples/blob/main/http/offer-provider/error-bad-request-v0.2.json)
- [Query API v0.2](./query-api.md)
- [Offer Schema v0.2](./offer-schema-v0.2.md)
