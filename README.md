# AgentOffer Protocol

The open contract for discovering commercial offers, preserving attribution,
and reporting conversion events between AI agents, AON, and Offer Providers.

## Current normative contract: Protocol v0.2

Protocol v0.2 is the only current public contract for new integrations. The
checked-in specification, JSON Schemas, TypeScript types, examples, and
contract vectors define the baseline. This source status does not claim that
the hosted AON runtime has implemented v0.2; runtime adoption and release
gating are tracked separately by BL-034 through BL-039 and WS-22.

## Start here

| Surface | Normative specification | Machine-readable contract |
|---|---|---|
| Agent-facing Query | [Query API v0.2](specs/query-api.md) | `offer-query-schema-v0.2.json` and `offer-query-response-v0.2.json` |
| AON-to-Partner discovery | [OfferProvider API v0.2](specs/offer-provider-api.md) | `offer-provider-request-v0.2.json` and `offer-provider-response-v0.2.json` |
| Returned Offer | [Offer Schema v0.2](specs/offer-schema-v0.2.md) | `offer-schema-v0.2.json` and `offer-v0.2.types.ts` |
| Conversion callbacks | [Postback v0.2](specs/postback.md) | Partner and Agent Postback payload schemas |
| Taxonomy | [Category Taxonomy](specs/category-taxonomy.md) | AON Taxonomy v1 registry |
| Contract lifecycle | [Contract Governance](specs/contract-governance.md) | `protocol/docs/contract-governance/contracts.json` |

Machine-readable files live in the
[`agentoffernetwork/schema`](https://github.com/agentoffernetwork/schema)
repository and canonical payloads live in
[`agentoffernetwork/examples`](https://github.com/agentoffernetwork/examples).

## v0.2 contract boundaries

- `POST /v1/offers/query` remains the HTTP shell. The path major and protocol
  payload version are separate concerns.
- An omitted `AON-Protocol-Version` header or the explicit value `0.2` selects
  v0.2. A successful response echoes `AON-Protocol-Version: 0.2`.
- Explicit `0.1`, unknown, or invalid protocol versions fail closed. The
  current public contract does not define a silent legacy fallback.
- Query and OfferProvider share one request business core. Provider dispatch
  additionally requires `request_id`; transport authentication and error
  handling remain channel-specific.
- Every Offer uses `version: "2.0"` and a non-empty `goals[]`. Each Goal has
  an event identity and closed CPA or CPS pricing. Public v0.2 Offer payloads
  do not contain `bid` or `conversion_rule.accepted_types`.
- Partner-to-AON and AON-to-Agent conversion payloads both carry required
  `event_name`, which exactly identifies a declared `goals[].event`.

## Runtime status

The repository publishes a stable source contract while
`offers.query/public-v0.2.runtime_support` remains `not_available`.
Implementations must not infer runtime availability from the presence of these
files. BL-038 owns the final cross-repository lifecycle and release decision.

## Historical material

v0.1 files are retained only as historical references. They are not the
default for new integrations, are not a compatibility obligation of the v0.2
baseline, and are not a prerequisite for the v0.2 contract gate.

## Contributing

Editorial fixes may be submitted directly. Semantic or breaking changes must
follow the RFC and contract-governance process. See [CONTRIBUTING.md](CONTRIBUTING.md).

This specification is licensed under
[Creative Commons Attribution 4.0 International](https://creativecommons.org/licenses/by/4.0/).
