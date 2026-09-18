# AgentOffer Protocol

The public contract for discovering commercial offers, preserving attribution,
and reporting conversion events between AI agents, AON, and Offer Providers.

## Current contract

Start a new integration from the role that matches your system.

## Choose your path

- **Building an Agent:** start with the [Agent quickstart](v1.0/quickstarts/agent.md), then use the [Query API](v1.0/specs/query-api.md), [Offer schema](v1.0/specs/offer-schema.md), [Offer and Query Response field semantics](v1.0/specs/offer-field-semantics.md), and [Postback contract](v1.0/specs/postback.md).
- **Building an Offer Provider:** start with the [Partner quickstart](v1.0/quickstarts/partner.md), then implement the [OfferProvider API](v1.0/specs/offer-provider-api.md) and [Postback contract](v1.0/specs/postback.md).
- **Using MCP:** review [MCP tools](v1.0/specs/mcp-tools.md) and [feedback and watches](v1.0/specs/mcp-feedback-watches.md).
- **Using shared classifications and targeting:** review the [category taxonomy](v1.0/specs/category-taxonomy.md) and [Offer location and age targeting](v1.0/specs/location-targeting.md).

See the [current contract overview](v1.0/README.md) for scope, lifecycle, and
source boundaries.

Machine-readable schemas, validators, and canonical payloads are published in the matching
[schema](https://github.com/agentoffernetwork/schema) and
[examples](https://github.com/agentoffernetwork/examples)
repositories. Individual deployments publish their endpoint, access, and
conformance details separately.

The `v1.0/` directory is the current release surface. Exact repository commits
come from the protected release manifest rather than mutable branch URLs.

The definition-first Taxonomy v1 expansion preserves 515 existing ids and adds
272 definitions for 787 canonical ids, independently of product admission.
Earlier deferred candidate evidence does not invalidate these definitions.
The ordinary protected protocol release publishes committed definitions
independently of downstream product validation. Runtime, classifier, and
warehouse adaptation and product validation follow a separate downstream Plan;
definition publication does not certify deployment support or product eligibility.

## Provenance

Earlier releases remain available from immutable tags and release evidence for
audit and recovery. They are not alternate current integration paths.

## Contributing

Editorial fixes may be submitted directly. Semantic or breaking changes follow
the RFC and [contract lifecycle](v1.0/governance/contract-lifecycle.md). See
[CONTRIBUTING.md](CONTRIBUTING.md).

This specification is licensed under
[Creative Commons Attribution 4.0 International](https://creativecommons.org/licenses/by/4.0/).

## Typed Flight Query source extension

[Flight Query](https://github.com/agentoffernetwork/protocol/blob/main/v1.0/specs/query-api.md#typed-flight-query)
adds explicit itinerary requests, reference versus traveler-total prices and
required Flight details while preserving Generic projections.
[Ctrip mapping](https://github.com/agentoffernetwork/protocol/blob/main/v1.0/specs/flight-query-ctrip-mapping.md)
records supplied tool evidence and unsupported facts.
[RFC-0007](https://github.com/agentoffernetwork/rfcs/blob/main/rfcs/RFC-0007-flight-query-profile.md)
is accepted/implemented in sources; public release and runtime support require
separate evidence. Exact version 1.0 alone does not enable this capability.
