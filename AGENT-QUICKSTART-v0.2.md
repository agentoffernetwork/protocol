---
title: Agent quickstart v0.2
audience: agent
canonical: false
compatibility: explicit
---

# Agent quickstart v0.2

Use this document only when an existing integration explicitly needs the
AgentOffer Protocol v0.2 compatibility contract. It describes the public source
contract only; it does not claim that a hosted AON runtime is available.

## Machine-readable contract

```json
{
  "schema": "aon-agent-quickstart-v1",
  "contract_version": "0.2",
  "runtime_availability": "not_available",
  "access_eligibility": "not_publicly_specified",
  "version_selection": {
    "omitted": { "action": "select", "contract_version": "0.2" },
    "explicit_0_2": { "action": "select", "contract_version": "0.2" },
    "explicit_0_1": { "action": "reject", "mode": "fail_closed" },
    "unknown": { "action": "reject", "mode": "fail_closed" }
  },
  "references": {
    "query_api": "https://github.com/agentoffernetwork/protocol/blob/main/specs/query-api.md",
    "offer_schema": "https://github.com/agentoffernetwork/protocol/blob/main/specs/offer-schema-v0.2.md",
    "schema": "https://github.com/agentoffernetwork/schema/blob/main/json-schema/offer-query-schema-v0.2.json",
    "examples": "https://github.com/agentoffernetwork/examples/blob/main/http/offer-response-v0.2.json"
  }
}
```

## Compatibility path

1. Load the Query API and Offer Schema references above before constructing a
   request or parsing an Offer.
2. Send `AON-Protocol-Version: 0.2` explicitly. Treat an explicit `0.1` or any
   unknown version as rejected; do not silently retry with a legacy wire format.
3. Validate the v0.2 response before use. A valid public Offer has
   `version: "2.0"` and non-empty `goals[]`; conversion callbacks identify the
   selected goal with its required `event_name`.
4. Keep this path for existing integrations and explicit compatibility requests;
   new integrations should start from the canonical v0.3 quickstart.
