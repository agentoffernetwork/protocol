---
title: Partner quickstart
audience: partner
canonical: true
contract_version: "0.3"
---

# Offer Provider integration quickstart

Start here when implementing an Offer Provider. This guide defines the public
request and response contract; deployment approval and endpoint configuration
remain between the Provider and its AON integration.

## Build an Offer Provider

1. Read the [OfferProvider API](../specs/offer-provider-api.md) and
   [Offer schema](../specs/offer-schema.md). Validate payloads against the exact
   Provider request and response paths in the
   [schema repository](https://github.com/agentoffernetwork/schema); the
   machine-readable metadata below binds those paths to the release manifest.
   Use the [category taxonomy](../specs/category-taxonomy.md) for category ids
   and [location and age targeting](../specs/location-targeting.md) for Offer
   eligibility declarations.
2. Register the Partner with `credentials.protocol_version: "0.3"`. AON then
   dispatches the Provider request and sends
   `AON-Protocol-Version: 0.3`.
3. Accept an HTTP `POST` with the shared Query body. Provider dispatch
   additionally requires `request_id`; do not use a request adapter that
   changes the public body shape.
4. Verify transport authentication before processing. HMAC key id, timestamp,
   nonce, signature, and test-mode headers remain transport metadata, not body
   fields. Sign the exact transmitted request body.
5. Return either a raw success response whose `request_id` exactly echoes
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
`response_options.thinking_mode` use the same bounded semantics as the Query API.
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

Each `offers[]` item follows the Offer schema. Keep `entity`, optional
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

## Report conversions

Use the [Postback contract](../specs/postback.md) for Provider-to-AON conversion
reports. Select the current contract with `AON-Protocol-Version: 0.3`, send the
closed JSON payload, and validate success and rejection behavior against the
published Provider examples. This trust boundary is separate from Agent
callback signing.

## Existing integration compatibility

Existing Partners remain on their configured compatibility profile. Use the
[legacy v0.2 quickstart](https://github.com/agentoffernetwork/protocol/tree/v0.2.0-legacy)
when maintaining that integration. AON does not probe a new endpoint and
silently retry a different contract.

## Machine-readable metadata

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
    "provider_api": {"repository": "agentoffernetwork/protocol", "path": "v0.3/specs/offer-provider-api.md"},
    "offer_schema": {"repository": "agentoffernetwork/protocol", "path": "v0.3/specs/offer-schema.md"},
    "category_taxonomy": {"repository": "agentoffernetwork/protocol", "path": "v0.3/specs/category-taxonomy.md"},
    "location_targeting": {"repository": "agentoffernetwork/protocol", "path": "v0.3/specs/location-targeting.md"},
    "request_schema": {"repository": "agentoffernetwork/schema", "path": "v0.3/json-schema/offer-provider-request.json"},
    "response_schema": {"repository": "agentoffernetwork/schema", "path": "v0.3/json-schema/offer-provider-response.json"},
    "provider_postback_spec": {"repository": "agentoffernetwork/protocol", "path": "v0.3/specs/postback.md"},
    "provider_postback_schema": {"repository": "agentoffernetwork/schema", "path": "v0.3/json-schema/postback-partner-payload.json"},
    "provider_postback_success_json": {"repository": "agentoffernetwork/examples", "path": "v0.3/http/postback/partner/basic-conversion.json"},
    "provider_postback_success_http": {"repository": "agentoffernetwork/examples", "path": "v0.3/http/postback/partner/basic-conversion.http"},
    "provider_postback_error_json": {"repository": "agentoffernetwork/examples", "path": "v0.3/http/postback/partner/invalid-unknown-field.json"},
    "provider_postback_error_http": {"repository": "agentoffernetwork/examples", "path": "v0.3/http/postback/partner/invalid-unknown-field.http"}
  },
  "compatibility_history": "https://github.com/agentoffernetwork/protocol/tree/v0.2.0-legacy"
}
```
