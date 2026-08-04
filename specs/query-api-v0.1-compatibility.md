# Offer Query API v0.1 Compatibility

**Status:** supported only for existing integrations. New integrations MUST use
[Query API v0.2](./query-api.md) with `AON-Protocol-Version: 0.2`.

## Compatibility selection

The legacy SDK endpoint remains unchanged:

```text
POST /v1/sdk/offers/query
```

For the hosted Query endpoint, an omitted `AON-Protocol-Version` header or an
explicit `0.1` selects the v0.1 compatibility lane. The legacy response shape
and headers remain unchanged. An explicit `0.2` selects the v0.2 contract.
Unknown, repeated, or invalid version headers fail closed.

## Contract artifacts

- Request: [`offer-query-schema-v0.1.json`](../../schema/json-schema/offer-query-schema-v0.1.json)
- Offer: [`offer-schema-v0.1.json`](../../schema/json-schema/offer-schema-v0.1.json)
- Request example: [`offer-query-request.json`](../../examples/http/offer-query-request.json)
- Response example: [`offer-response.json`](../../examples/http/offer-response.json)

The v0.1 compatibility Offer may contain `bid`. It is not a valid v0.2 Offer
field and MUST NOT appear in v0.2 requests, responses, SDK entry points, or
new-integration documentation.

## Change policy

This compatibility contract receives regression fixes only. It must not gain
new protocol fields. Any retirement requires measured legacy usage, an
announced migration window, and an explicit release decision.
