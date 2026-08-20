---
title: Partner quickstart v0.3
audience: partner
canonical: true
contract_version: "0.3"
---

# Partner quickstart v0.3

Use this document as the canonical starting point for a new Offer Provider
integration. It describes the public Protocol v0.3 source contract; deployment
approval and runtime availability remain owned by the Provider and AON runtime.

## Machine-readable contract

```json
{
  "schema": "aon-partner-quickstart-v1",
  "contract_version": "0.3",
  "runtime_availability": "deployment_owned",
  "access_eligibility": "not_publicly_specified",
  "version_selection": {
    "configured_0_3": { "action": "select", "contract_version": "0.3" },
    "configured_0_2": { "action": "select", "contract_version": "0.2", "mode": "compatibility" },
    "omitted": { "action": "retain_existing_legacy_profile" },
    "unknown": { "action": "reject", "mode": "fail_closed" }
  },
  "references": {
    "provider_api": "https://github.com/agentoffernetwork/protocol/blob/main/specs/offer-provider-api-v0.3.md",
    "query_api": "https://github.com/agentoffernetwork/protocol/blob/main/specs/query-api-v0.3.md",
    "offer_schema": "https://github.com/agentoffernetwork/protocol/blob/main/specs/offer-schema-v0.3.md",
    "request_schema": "https://github.com/agentoffernetwork/schema/blob/main/json-schema/offer-provider-request-v0.3.json",
    "response_schema": "https://github.com/agentoffernetwork/schema/blob/main/json-schema/offer-provider-response-v0.3.json",
    "examples": "https://github.com/agentoffernetwork/examples/blob/main/http/offer-query-v0.3.json",
    "v0_2_compatibility": "https://github.com/agentoffernetwork/protocol/blob/main/PARTNER-QUICKSTART-v0.2.md"
  }
}
```

## New integration path

1. Register the Partner with `credentials.protocol_version: "0.3"`. AON then
   dispatches the canonical v0.3 Provider request and sends
   `AON-Protocol-Version: 0.3`.
2. Accept an HTTP `POST` with the shared v0.3 Query body. Provider dispatch
   additionally requires `request_id`; do not use a request adapter that
   changes the public body shape.
3. Verify transport authentication before processing. HMAC key id, timestamp,
   nonce, signature, and test-mode headers remain transport metadata, not body
   fields. Sign the exact transmitted request body.
4. Return either a raw v0.3 success response whose `request_id` exactly echoes
   the request, or the closed Provider error envelope. Do not wrap a success
   body in the hosted `{code,message,data,extra}` envelope.

## Minimal request

```http
POST {partner_endpoint}
Content-Type: application/json
AON-Protocol-Version: 0.3
X-AON-Key: partner-issued-key-id
X-AON-Timestamp: 1787184000
X-AON-Nonce: 7f22b4df-7f1f-42c0-8579-6bf7997f4a8e
X-AON-Signature: <lowercase-hex-hmac-sha256>
```

```jsonc
{
  "request_id": "0195af51-8b2c-7d3e-a1b2-c3d4e5f60718",
  "context": {"platform": {"name": "travel-agent", "channel": "api"}},
  "intent": {
    "content": [{"type": "input_text", "text": "quiet hotel in Kyoto"}],
    "provenance": "user_expressed",
    "signals": {"budget": {"max": 250, "currency": "USD"}}
  }
}
```

`intent.origin[]`, `constraints`, `force_offer`, and
`response_options.thinking_mode` use the same bounded semantics as Query v0.3.
The Provider must not require raw conversation history or a long-term user
profile to serve the request.

## Response

### Success

```jsonc
{
  "request_id": "0195af51-8b2c-7d3e-a1b2-c3d4e5f60718",
  "protocol_version": "0.3",
  "language": "en",
  "offers": []
}
```

Each `offers[]` item follows Offer v0.3. Keep `entity`, optional
`listing_source`, and `action` separate. Include `match_reason` only when it
is safe for end users; AON removes it when the original request disables
`thinking_mode`.

### Error

```jsonc
{
  "code": "BAD_REQUEST",
  "message": "request payload is invalid",
  "data": {},
  "extra": {}
}
```

Allowed error codes are `BAD_REQUEST`, `UNAUTHORIZED`, `FORBIDDEN`,
`RATE_LIMITED`, and `INTERNAL_ERROR`. Never mix error fields with a success
payload.

## Existing integration compatibility

Existing Partners on v0.2 remain unchanged and must retain an explicit
`credentials.protocol_version: "0.2"` profile. See the
[v0.2 compatibility quickstart](PARTNER-QUICKSTART-v0.2.md). An unconfigured
Partner remains on its existing legacy profile; AON does not probe a v0.3
endpoint and silently retry a different contract.
