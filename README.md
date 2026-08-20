# AgentOffer Protocol

The open contract for discovering commercial offers, preserving attribution,
and reporting conversion events between AI agents, AON, and Offer Providers.

## Current normative contract: Protocol v0.3

Protocol v0.3 is the current public contract for new integrations. The
checked-in specification, JSON Schemas, TypeScript types, examples, and
contract vectors define the baseline. Runtime availability is deployment-owned:
an implementation must provide conformance evidence before accepting v0.3
traffic. Protocol v0.2 remains an explicit compatibility path and is unchanged.

## Start here

For a directly loadable integration guide, use the canonical
[Agent v0.3 quickstart](AGENT-QUICKSTART-v0.3.md). Partner integrations begin
with the [OfferProvider API v0.3](specs/offer-provider-api-v0.3.md). These are
the public entry points; the other public repositories link here rather than
copying the steps.

| Surface | Normative specification | Machine-readable contract |
|---|---|---|
| Agent-facing Query | [Query API v0.3](specs/query-api-v0.3.md) | `offer-query-schema-v0.3.json` and `offer-query-response-v0.3.json` |
| AON-to-Partner discovery | [OfferProvider API v0.3](specs/offer-provider-api-v0.3.md) | `offer-provider-request-v0.3.json` and `offer-provider-response-v0.3.json` |
| Returned Offer | [Offer Schema v0.3](specs/offer-schema-v0.3.md) | `offer-schema-v0.3.json` and `offer-v0.3.types.ts` |
| Conversion callbacks | [Postback v0.2](specs/postback.md) | Partner and Agent Postback payload schemas |
| Taxonomy | [Category Taxonomy](specs/category-taxonomy.md) | AON Taxonomy v1 registry |
| Contract lifecycle | [Contract Governance](specs/contract-governance.md) | `protocol/docs/contract-governance/contracts.json` |

Machine-readable files live in the
[`agentoffernetwork/schema`](https://github.com/agentoffernetwork/schema)
repository and canonical payloads live in
[`agentoffernetwork/examples`](https://github.com/agentoffernetwork/examples).

## v0.3 contract boundaries

- `POST /v1/offers/query` remains the HTTP shell. The path major and protocol
  payload version are separate concerns.
- `AON-Protocol-Version: 0.3` selects v0.3. A successful response echoes the
  same version when the deployment supports it.
- Requests with `0.2` keep the existing v0.2 behavior. Unknown or unsupported
  versions fail closed; implementations must not silently reinterpret payloads.
- Query and OfferProvider share one request business core. Provider dispatch
  additionally requires `request_id`; transport authentication and error
  handling remain channel-specific.
- v0.3 carries current-turn intent, optional structured context, guided
  refinements/follow-up topics, optional `listing_source`, and the
  `thinking_mode`/`force_offer` controls described by the v0.3 specs.

## Runtime status

The repository publishes the adopted v0.3 source contract. Implementations must
not infer runtime availability from the presence of these files; API, MCP,
Provider, and Host owners publish deployment-specific conformance evidence.

## v0.2 compatibility status

Protocol v0.2 remains available for existing integrations and explicit
compatibility requests. Its historical source files and release evidence are
unchanged. New integrations should start from the v0.3 quickstart.

## Historical material

v0.1 files are retained only as historical references. They are not the
default for new integrations, are not a compatibility obligation of the v0.2
baseline, and are not a prerequisite for the v0.2 contract gate.

## Contributing

Editorial fixes may be submitted directly. Semantic or breaking changes must
follow the RFC and contract-governance process. See [CONTRIBUTING.md](CONTRIBUTING.md).

This specification is licensed under
[Creative Commons Attribution 4.0 International](https://creativecommons.org/licenses/by/4.0/).
