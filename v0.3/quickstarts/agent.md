---
title: Agent quickstart
audience: agent
canonical: true
contract_version: "0.3"
---

# Agent integration quickstart

Start here when adding AgentOffer query and Offer-consumption support to an
Agent. This guide defines the portable public contract; a deployment provides
its own endpoint and access details.

## Build an Agent integration

1. Read the [Query API](../specs/query-api.md) and
   [Offer schema](../specs/offer-schema.md) before constructing a request or
   parsing an Offer. The matching files in the
   [schema repository](https://github.com/agentoffernetwork/schema) and
   [examples repository](https://github.com/agentoffernetwork/examples) are
   bound by exact path in the machine-readable metadata below. Add
   [MCP tools](../specs/mcp-tools.md) only when your Agent exposes those tools.
   Use the [category taxonomy](../specs/category-taxonomy.md) for category ids.
   When consuming Offer targeting, follow
   [location and age targeting](../specs/location-targeting.md); Query v0.3
   does not accept viewer location or age fields.
2. Obtain a deployed endpoint and access credentials from the relevant
   deployment owner.
3. Send `AON-Protocol-Version: 0.3`. The request carries the current structured intent and required `intent.provenance`; bounded
   session context, `force_offer`, and `thinking_mode` are optional.
4. Validate the response before use. The response echoes
   `protocol_version: "0.3"`, includes `request_id`, `language`, and `offers`,
   and may include bounded `refinements` or `followup_topics` guidance.
5. If the Agent receives conversion callbacks, implement the
   [Postback contract](../specs/postback.md). Verify the signature over the exact
   raw body, keep receiver processing idempotent, and use the published retry
   and signing vectors.

## Existing integration compatibility

If you maintain an existing v0.2 SDK or Partner integration, use the
[compatibility quickstart](https://github.com/agentoffernetwork/protocol/tree/v0.2.0-legacy). A request
that explicitly selects `0.2` must not be silently reinterpreted as the
current contract.

## Need a protocol change?

Use the RFC process for semantic contract changes. The RFC repository governs
changes; this quickstart remains the integration entry point.

## Machine-readable metadata

```json
{
  "schema": "aon-agent-quickstart-v1",
  "contract_version": "0.3",
  "runtime_availability": "not_available",
  "access_eligibility": "not_publicly_specified",
  "version_selection": {
    "omitted": { "action": "follow_deployment_documented_rollout" },
    "explicit_0_3": { "action": "select", "contract_version": "0.3" },
    "explicit_0_2": { "action": "select", "contract_version": "0.2", "mode": "compatibility" },
    "explicit_0_1": { "action": "reject", "mode": "fail_closed" },
    "unknown": { "action": "reject", "mode": "fail_closed" }
  },
  "references": {
    "query_api": {"repository": "agentoffernetwork/protocol", "path": "v0.3/specs/query-api.md"},
    "offer_schema": {"repository": "agentoffernetwork/protocol", "path": "v0.3/specs/offer-schema.md"},
    "mcp_tools": {"repository": "agentoffernetwork/protocol", "path": "v0.3/specs/mcp-tools.md"},
    "category_taxonomy": {"repository": "agentoffernetwork/protocol", "path": "v0.3/specs/category-taxonomy.md"},
    "location_targeting": {"repository": "agentoffernetwork/protocol", "path": "v0.3/specs/location-targeting.md"},
    "query_schema": {"repository": "agentoffernetwork/schema", "path": "v0.3/json-schema/offer-query-schema.json"},
    "query_example": {"repository": "agentoffernetwork/examples", "path": "v0.3/http/offer-query.json"},
    "agent_postback_spec": {"repository": "agentoffernetwork/protocol", "path": "v0.3/specs/postback.md"},
    "agent_postback_schema": {"repository": "agentoffernetwork/schema", "path": "v0.3/json-schema/postback-agent-payload.json"},
    "agent_postback_example": {"repository": "agentoffernetwork/examples", "path": "v0.3/http/postback/agent/basic-conversion.http"},
    "agent_postback_signing_vectors": {"repository": "agentoffernetwork/schema", "path": "v0.3/fixtures/postback-agent-webhook.json"}
  },
  "compatibility_history": "https://github.com/agentoffernetwork/protocol/tree/v0.2.0-legacy"
}
```
