# AgentOffer v0.3 Quickstart

> This is the canonical quickstart for new v0.3 integrations. Runtime
> availability is deployment-specific; use the deployment's conformance and
> availability evidence before sending production traffic.

The main additions are current-turn structured intent, limited session context,
optional `force_offer`, `response_options.thinking_mode`, guidance via
`refinements` / `followup_topics`, item-level `query_helper`, and optional
`listing_source`. `decision_factors` is intentionally not part of v0.3.

See the v0.3 Query, Offer, Provider and MCP specifications for exact fields and
negative cases. v0.2 remains available as an explicit compatibility path.
