# AgentOffer Protocol

The public contract for discovering commercial offers, preserving attribution,
and reporting conversion events between AI agents, AON, and Offer Providers.

## Current contract

Start a new integration from the role that matches your system.

## Choose your path

- **Building an Agent:** start with the [Agent quickstart](v0.3/quickstarts/agent.md), then use the [Query API](v0.3/specs/query-api.md), [Offer schema](v0.3/specs/offer-schema.md), and [Postback contract](v0.3/specs/postback.md).
- **Building an Offer Provider:** start with the [Partner quickstart](v0.3/quickstarts/partner.md), then implement the [OfferProvider API](v0.3/specs/offer-provider-api.md) and [Postback contract](v0.3/specs/postback.md).
- **Using MCP:** review [MCP tools](v0.3/specs/mcp-tools.md) and [feedback and watches](v0.3/specs/mcp-feedback-watches.md).
- **Using shared classifications and targeting:** review the [category taxonomy](v0.3/specs/category-taxonomy.md) and [Offer location and age targeting](v0.3/specs/location-targeting.md).

See the [current contract overview](v0.3/README.md) for scope, lifecycle, and
source boundaries.

Machine-readable schemas, validators, and canonical payloads are published in the matching
[schema](https://github.com/agentoffernetwork/schema) and
[examples](https://github.com/agentoffernetwork/examples)
repositories. Individual deployments publish their endpoint, access, and
conformance details separately.

The `v0.3/` directory is the current release surface. Exact repository commits
come from the protected release manifest rather than mutable branch URLs.

## Earlier contracts

Maintainers of existing integrations can use the immutable
[`v0.1.0-legacy`](https://github.com/agentoffernetwork/protocol/tree/v0.1.0-legacy)
and
[`v0.2.0-legacy`](https://github.com/agentoffernetwork/protocol/tree/v0.2.0-legacy)
tags.

## Contributing

Editorial fixes may be submitted directly. Semantic or breaking changes follow
the RFC and [contract lifecycle](v0.3/governance/contract-lifecycle.md). See
[CONTRIBUTING.md](CONTRIBUTING.md).

This specification is licensed under
[Creative Commons Attribution 4.0 International](https://creativecommons.org/licenses/by/4.0/).
