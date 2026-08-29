# Current Contract

This directory is the current AgentOffer Protocol contract for new
integrations. It defines portable behavior between Agents, AON deployments,
and Offer Providers; endpoint access and runtime availability remain owned by
each deployment.

## Start by role

- **Agent integration:** follow the [Agent quickstart](quickstarts/agent.md),
  then implement the [Query API](specs/query-api.md),
  [Offer schema](specs/offer-schema.md), and
  [Offer and Query Response field semantics](specs/offer-field-semantics.md), and
  [Postback contract](specs/postback.md) as needed.
- **Offer Provider integration:** follow the
  [Partner quickstart](quickstarts/partner.md), then implement the
  [OfferProvider API](specs/offer-provider-api.md) and
  [Postback contract](specs/postback.md).
- **MCP integration:** use [MCP tools](specs/mcp-tools.md) together with
  [feedback and watches](specs/mcp-feedback-watches.md).

Shared resources define the stable
[category taxonomy](specs/category-taxonomy.md) and
[Offer location and age targeting](specs/location-targeting.md). Query v0.3
does not expose viewer location or age fields.

Machine-readable schemas, validators, types, and payloads are published in the
matching [Schema](https://github.com/agentoffernetwork/schema/tree/main/v0.3)
and [Examples](https://github.com/agentoffernetwork/examples/tree/main/v0.3)
directories.

## Contract lifecycle

The current contract is **adopted** and **stable** for new integrations.
Contract adoption does not claim that every deployment accepts the contract;
deployment owners publish endpoint, access, rollout, and conformance status
separately. See [Contract lifecycle](governance/contract-lifecycle.md) for the
authority, compatibility, and change rules.

## Compatibility

Existing v0.1 and v0.2 integrations remain available through immutable legacy
tags. New integrations should not mix legacy payload assumptions with the
current contract or rely on silent version fallback.
