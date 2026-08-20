---
title: Partner quickstart v0.2 compatibility
audience: partner
canonical: false
compatibility: explicit
---

# Partner quickstart v0.2 compatibility

Use this page only for an existing Offer Provider that explicitly remains on
Protocol v0.2. New Provider integrations start from the canonical
[Partner v0.3 quickstart](PARTNER-QUICKSTART.md).

## Machine-readable contract

```json
{
  "schema": "aon-partner-quickstart-v1",
  "contract_version": "0.2",
  "runtime_availability": "deployment_owned",
  "access_eligibility": "not_publicly_specified",
  "version_selection": {
    "configured_0_2": { "action": "select", "contract_version": "0.2" },
    "configured_0_3": { "action": "select", "contract_version": "0.3" },
    "unknown": { "action": "reject", "mode": "fail_closed" }
  },
  "references": {
    "provider_api": "https://github.com/agentoffernetwork/protocol/blob/main/specs/offer-provider-api.md",
    "query_api": "https://github.com/agentoffernetwork/protocol/blob/main/specs/query-api.md",
    "offer_schema": "https://github.com/agentoffernetwork/protocol/blob/main/specs/offer-schema-v0.2.md",
    "request_schema": "https://github.com/agentoffernetwork/schema/blob/main/json-schema/offer-provider-request-v0.2.json",
    "response_schema": "https://github.com/agentoffernetwork/schema/blob/main/json-schema/offer-provider-response-v0.2.json"
  }
}
```

## Compatibility path

Keep `credentials.protocol_version: "0.2"`, accept
`AON-Protocol-Version: 0.2`, and use the v0.2 Provider request/response
schemas. This compatibility page does not redefine the v0.3 default.
