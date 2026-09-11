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
[category taxonomy](specs/category-taxonomy.md),
[Offer location and age targeting](specs/location-targeting.md), and
[Location Search API](specs/location-search-api.md). Query v1.0 does not
expose viewer location or age fields.

The taxonomy publishes only evidence-admitted category ids. Commerce
candidate and source-crosswalk artifacts are planning and audit inputs, not a
second set of selectable ids. Integrations must validate against the published
taxonomy tree or resolver for the active release.

Machine-readable schemas, validators, types, and payloads are published in the
matching [Schema](https://github.com/agentoffernetwork/schema/tree/main/v1.0)
and [Examples](https://github.com/agentoffernetwork/examples/tree/main/v1.0)
directories.

Query response Offers may carry one optional, response-scoped
`offer_info.commercial.display_price` for presentation in a target currency.
The original `commercial.price` remains authoritative as the source and
fallback price; Partner and OfferProvider supply carriers reject the derived
field. See the [Query API](specs/query-api.md),
[Offer schema](specs/offer-schema.md), and
[field semantics](specs/offer-field-semantics.md).

## Contract lifecycle

An empty main Query result may include an independent `alternative_offers`
list with 1–3 complete Generic Offers and mandatory request-specific selection
reasons, initially based only on genuine same-country `regional_popularity`.
Main `offers: []` and `empty_reason` remain; alternatives never claim a direct
query match. See [Query API](specs/query-api.md#optional-alternative-offers) for
eligibility, explicit-constraint and special-entry restrictions.

This response extension does not change requests, the exact `1.0` selector,
or existing `force_offer` behavior. Old closed readers may reject the field;
permissive readers may discard it. Consumers must upgrade before producers
enable it. Service, SDK, and Agent adoption is separate from the protocol
assets and is not certified by publication or synthetic examples.

The current contract is **adopted** and **stable** for new integrations.
Contract adoption does not claim that every deployment accepts the contract;
deployment owners publish endpoint, access, rollout, and conformance status
separately. See [Contract lifecycle](governance/contract-lifecycle.md) for the
authority, compatibility, and change rules.

## Provenance

Earlier releases remain available from immutable tags for audit and recovery.
They are not alternate current integration paths.
